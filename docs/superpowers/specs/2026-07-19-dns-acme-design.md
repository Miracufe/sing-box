# Sing-box 部署脚本证书管理与续签机制优化设计规约

## 1. 目的与背景
当前 `sing-box.sh` 一键部署脚本在证书管理上存在以下优化空间：
- **自动续签失败隐患**：当前 Let's Encrypt 证书使用 standalone (80 端口) 申请，若后台 crontab 自动续签时 nginx / caddy 等服务处于开启状态，续签会因端口冲突而失败。
- **验证方式单一**：目前仅支持占用 80 端口的 HTTP-01 验证，对于 80 端口被封禁或不希望中断反代网站服务的用户，无法使用。
- **证书类型错判**：在生成客户端配置时，强行以 Issuer 字段是否含有 `Let's Encrypt` 判断证书是否为公信商业证书，容易误判 ZeroSSL、BuyPass 或自定义受信任证书为“自签证书”。
- **文件残留**：用户回退到自签证书模式时，acme.sh 物理证书文件仍残留在本地。

为了解决上述问题，本设计提出：
- 增加 Cloudflare DNS-01 API 验证支持。
- 对 HTTP-01 验证追加 `pre-hook` 和 `post-hook` 服务自动关启逻辑。
- 采用 `openssl verify` 通用证书链信任验证。
- 强化证书回退的物理清理。

---

## 2. 详细设计与实现方案

### 2.1 交互菜单重构
在 `manage_acme_certificate_menu` 的选项 `1` 中，输入域名并通过 DNS 解析校验后，引入子菜单供用户选择验证模式：

```text
请选择证书申请验证方式 / Please choose certificate validation method:
1. HTTP-01 验证 (独立 80 端口模式，适合 80 端口开放且未被占用的服务器)
   HTTP-01 Validation (Standalone Port 80, requires port 80 open & unused)
2. DNS-01 验证 (Cloudflare DNS API 模式，适合有 Cloudflare 托管域名，免停机/免占 80 端口)
   DNS-01 Validation (Cloudflare DNS API, zero-downtime, no port 80 required)
```

如果用户选择 `2` (DNS-01)，则进一步引导凭证输入：
```text
请选择 Cloudflare 凭据类型 / Choose Cloudflare Credentials:
1. 使用 API Token (推荐) / Use API Token (Recommended)
2. 使用 Global API Key (需要输入 Email) / Use Global API Key (Requires Email)
```
- 对于选项 `1` (API Token)，读取并设置环境变量 `CF_Token`。
- 对于选项 `2` (Global Key)，读取并设置环境变量 `CF_Email` 与 `CF_Key`。

---

### 2.2 验证与自动续签优化

#### 2.2.1 HTTP-01 模式优化
在 `--issue` 申请证书时，显式配置 `pre-hook` 和 `post-hook` 自动管理可能占用 80 端口的服务：
```bash
/root/.acme.sh/acme.sh --issue -d "$CERT_DOMAIN" --standalone --keylength ec-256 --force \
  --pre-hook "systemctl stop nginx caddy sing-box" \
  --post-hook "systemctl start nginx caddy sing-box"
```
通过此参数，acme.sh 会将 Hook 命令持久化在 `/${HOME}/.acme.sh/${CERT_DOMAIN}_ecc/${CERT_DOMAIN}.conf` 中。后续后台 cron 任务静默自动续签此证书时，会自动在验证前关停服务，验证后拉起服务，解决端口冲突导致的后台续期失败问题。

#### 2.2.2 DNS-01 模式实现
在导出环境变量后，调用 acme.sh 官方 dns_cf 插件完成挑战：
```bash
/root/.acme.sh/acme.sh --issue --dns dns_cf -d "$CERT_DOMAIN" --keylength ec-256 --force
```
验证通过后，使用 `--install-cert` 安装证书：
```bash
/root/.acme.sh/acme.sh --install-cert -d "$CERT_DOMAIN" --ecc \
  --key-file "${WORK_DIR}/cert/private.key" \
  --fullchain-file "${WORK_DIR}/cert/cert.pem" \
  --reloadcmd "cp ${WORK_DIR}/cert/cert.pem ${WORK_DIR}/cert/cert_200.pem && systemctl restart sing-box"
```
由于 DNS-01 不需要监听 80 端口，因此不需要 any pre/post 关停服务 Hook。

---

### 2.3 商业公信证书智能检测
将订阅配置及客户端输出逻辑中原有的 `Let's Encrypt` 字符串强匹配：
```bash
local ISSUER=$(openssl x509 -in ${WORK_DIR}/cert/cert.pem -issuer -noout 2>/dev/null)
if grep -q -i "Let's Encrypt" <<< "$ISSUER"; then
  IS_COMMERCIAL_CERT=true
fi
```
重构为通用系统证书库判定：
```bash
local IS_COMMERCIAL_CERT=false
if [ -s "${WORK_DIR}/cert/cert.pem" ]; then
  # 利用系统内置的默认根证书库验证该证书链是否受信任
  if openssl verify "${WORK_DIR}/cert/cert.pem" >/dev/null 2>&1; then
    IS_COMMERCIAL_CERT=true
  fi
fi
```
系统包含 `ca-certificates`（常见 Linux VPS 系统均默认安装）时，若为公信 CA 签发的有效证书（如 Let's Encrypt, ZeroSSL, BuyPass, Cloudflare 边缘证书），`openssl verify` 将成功返回状态码 `0`，由此判定 `IS_COMMERCIAL_CERT=true`，客户端不再输出证书指纹。

---

### 2.4 回退证书物理残留清理
在证书管理菜单中选择 `4. 回退并恢复使用系统自签证书` 时，在原有的 `acme.sh --remove` 执行完毕后，清空物理残留目录，避免硬盘垃圾和证书泄露：
```bash
if [ -d "/root/.acme.sh/${CURRENT_DOMAIN}_ecc" ]; then
  rm -rf "/root/.acme.sh/${CURRENT_DOMAIN}_ecc"
fi
```

---

## 3. 测试与验证策略
1. **单元验证**：
   - 验证 `openssl verify` 针对本地已申请的 Let's Encrypt 证书和原版自签证书的表现。
   - 检查在 HTTP-01 验证下申请证书时，`/root/.acme.sh` 下对应域名的 `.conf` 配置文件中是否成功包含 `SAVED_PRE_HOOK` 和 `SAVED_POST_HOOK`。
2. **逻辑流程验证**：
   - 模拟用户进入证书申请菜单，选择 HTTP-01 路径，测试正常申请。
   - 模拟用户选择 DNS-01 路径，输入测试 Cloudflare 凭据（Token），测试申请逻辑。
   - 校验回退证书功能后，`/root/.acme.sh/${CURRENT_DOMAIN}_ecc` 目录被成功物理清除。
