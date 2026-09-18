# DNS-01 API 验证与证书管理优化实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 在 `sing-box.sh` 脚本中重构 Let's Encrypt 证书申请与续签机制，引入 Cloudflare DNS-01 验证、优化 HTTP-01 端口挂钩 (hooks)、使用系统根证书库检测商用证书、清理证书物理残留。

**架构：**
1. 重构证书选择交互菜单，增加验证类型与 Cloudflare API Credentials 输入选择。
2. 升级 HTTP-01 申请参数，绑定 `--pre-hook` 和 `--post-hook` 自动化服务启停。
3. 升级 `IS_COMMERCIAL_CERT` 变量判断，利用 `openssl verify` 进行通用权威证书库校验。
4. 增强 Revert 逻辑，物理清理 `/root/.acme.sh` 下的废弃证书文件夹。

**技术栈：** Bash Shell, openssl, acme.sh, systemctl

---

### 任务 1：重构商业公信证书智能检测逻辑

**文件：**
- 修改：`D:/Work/scripts/sing-box/sing-box.sh:4565-4569`

- [ ] **步骤 1：分析并替换原有硬编码匹配**
  在 `sing-box.sh` 的 4565-4569 行：
  ```bash
  local IS_COMMERCIAL_CERT=false
  local ISSUER=$(openssl x509 -in ${WORK_DIR}/cert/cert.pem -issuer -noout 2>/dev/null)
  if grep -q -i "Let's Encrypt" <<< "$ISSUER"; then
    IS_COMMERCIAL_CERT=true
  fi
  ```
  修改为系统受信链校验：
  ```bash
  local IS_COMMERCIAL_CERT=false
  if [ -s "${WORK_DIR}/cert/cert.pem" ]; then
    if openssl verify "${WORK_DIR}/cert/cert.pem" >/dev/null 2>&1; then
      IS_COMMERCIAL_CERT=true
    fi
  fi
  ```

- [ ] **步骤 2：使用 bash -n 进行语法验证**
  运行：`bash -n D:/Work/scripts/sing-box/sing-box.sh`
  预期：没有输出任何语法错误。

- [ ] **步骤 3：编写本地临时模拟测试验证逻辑**
  创建临时测试脚本 `D:/Work/scripts/sing-box/scratch/test_cert_verify.sh`：
  ```bash
  #!/bin/bash
  # 模拟检测逻辑
  WORK_DIR="D:/Work/scripts/sing-box"
  # 这里可以用一个已有的自签证书或空文件进行验证测试
  IS_COMMERCIAL_CERT=false
  if [ -s "${WORK_DIR}/cert/cert.pem" ]; then
    if openssl verify "${WORK_DIR}/cert/cert.pem" >/dev/null 2>&1; then
      IS_COMMERCIAL_CERT=true
    fi
  fi
  echo "IS_COMMERCIAL_CERT=${IS_COMMERCIAL_CERT}"
  ```
  运行：`bash D:/Work/scripts/sing-box/scratch/test_cert_verify.sh`
  预期：若不存在 `cert.pem` 或其为自签证书，输出 `IS_COMMERCIAL_CERT=false`。

- [ ] **步骤 4：Commit**
  ```bash
  git add sing-box.sh
  git commit -m "feat: replace hardcoded certificate detection with openssl verify"
  ```

---

### 任务 2：重构证书申请菜单与 Standalone Hooks / DNS-01 API 实现

**文件：**
- 修改：`D:/Work/scripts/sing-box/sing-box.sh:5765-5850`

- [ ] **步骤 1：修改 `manage_acme_certificate_menu` 的 case 1)**
  精确定位并替换 `sing-box.sh` 约 5765-5850 行的证书申请及安装逻辑。
  替换为以下核心代码，处理 HTTP-01（带 pre/post hooks）以及 DNS-01（Cloudflare API 验证）：
  ```bash
      # 验证解析
      [ "$L" = "C" ] && info "开始检查域名解析..." || info "Checking domain DNS resolution..."
      local RESOLVED_IP=$(ping -c 1 -4 "$CERT_DOMAIN" 2>/dev/null | awk -F '[()]' '/PING/{print $2}')
      [ -z "$RESOLVED_IP" ] && RESOLVED_IP=$(nslookup "$CERT_DOMAIN" 2>/dev/null | awk '/Address: / {print $2}' | tail -n 1)

      # 选择申请验证方式
      local ACME_MODE=""
      if [ "$L" = "C" ]; then
        echo -e "请选择证书申请验证方式 / Choose validation method:"
        echo -e "1. HTTP-01 验证 (独立 80 端口模式，需临时停用 80 端口服务)"
        echo -e "2. DNS-01 验证 (Cloudflare DNS API 模式，免占用 80 端口/免停机)"
        reading " 请选择 [1-2] (默认 1): " ACME_MODE
      else
        echo -e "Please choose certificate validation method:"
        echo -e "1. HTTP-01 Validation (Standalone Port 80, requires temporary downtime)"
        echo -e "2. DNS-01 Validation (Cloudflare DNS API, zero-downtime)"
        reading " Please choose [1-2] (default 1): " ACME_MODE
      fi
      ACME_MODE=${ACME_MODE:-1}

      if [ "$ACME_MODE" = "1" ]; then
        # HTTP-01 的域名解析检查（不可省）
        if [ -z "$RESOLVED_IP" ]; then
          [ "$L" = "C" ] && warning "域名 $CERT_DOMAIN 无法解析，请确保已添加 DNS 解析！" || warning "Domain $CERT_DOMAIN cannot be resolved. Please make sure DNS is configured!"
          sleep 3
          manage_acme_certificate_menu
          return
        fi
        [ "$L" = "C" ] && info "检测到域名解析到 IP: $RESOLVED_IP" || info "Detected domain resolves to IP: $RESOLVED_IP"
      fi

      # 安装依赖
      if command -v apt-get >/dev/null 2>&1; then
        apt-get update -y && apt-get install -y socat curl >/dev/null 2>&1
      elif command -v yum >/dev/null 2>&1; then
        yum install -y socat curl >/dev/null 2>&1
      fi

      # 安装 acme.sh
      if [ ! -d "/root/.acme.sh" ]; then
        curl https://get.acme.sh | sh -s email=my_singbox_acme@gmail.com >/dev/null 2>&1
      fi

      if [ -x "/root/.acme.sh/acme.sh" ]; then
        # 注册 ACME 账户，设置默认 CA 为 Let's Encrypt
        /root/.acme.sh/acme.sh --register-account -m my_singbox_acme@gmail.com --server letsencrypt >/dev/null 2>&1
        /root/.acme.sh/acme.sh --set-default-ca --server letsencrypt >/dev/null 2>&1

        if [ "$ACME_MODE" = "1" ]; then
          # 临时停止占用 80 端口的服务
          systemctl stop nginx >/dev/null 2>&1 || true
          systemctl stop caddy >/dev/null 2>&1 || true
          systemctl stop sing-box >/dev/null 2>&1 || true

          # 申请证书 (HTTP-01, standalone 且配置自动续签 pre-hook / post-hook)
          /root/.acme.sh/acme.sh --issue -d "$CERT_DOMAIN" --standalone --keylength ec-256 --force \
            --pre-hook "systemctl stop nginx caddy sing-box" \
            --post-hook "systemctl start nginx caddy sing-box"
        else
          # DNS-01 Mode, 收集 credentials
          local CF_CRED_CHOOSE=""
          local CF_TOKEN_INPUT=""
          local CF_KEY_INPUT=""
          local CF_EMAIL_INPUT=""
          if [ "$L" = "C" ]; then
            echo -e "请选择 Cloudflare 凭据类型:"
            echo -e "1. 使用 API Token (推荐)"
            echo -e "2. 使用 Global API Key"
            reading " 请选择 [1-2] (默认 1): " CF_CRED_CHOOSE
          else
            echo -e "Choose Cloudflare credentials type:"
            echo -e "1. Use API Token (Recommended)"
            echo -e "2. Use Global API Key"
            reading " Please choose [1-2] (default 1): " CF_CRED_CHOOSE
          fi
          CF_CRED_CHOOSE=${CF_CRED_CHOOSE:-1}

          if [ "$CF_CRED_CHOOSE" = "1" ]; then
            if [ "$L" = "C" ]; then
              reading " 请输入 Cloudflare API Token: " CF_TOKEN_INPUT
            else
              reading " Please enter Cloudflare API Token: " CF_TOKEN_INPUT
            fi
            if [ -z "$CF_TOKEN_INPUT" ]; then
              [ "$L" = "C" ] && warning "Token 不能为空！" || warning "Token cannot be empty!"
              sleep 2
              manage_acme_certificate_menu
              return
            fi
            export CF_Token="$CF_TOKEN_INPUT"
            unset CF_Key
            unset CF_Email
          else
            if [ "$L" = "C" ]; then
              reading " 请输入 Cloudflare 账户 Email: " CF_EMAIL_INPUT
              reading " 请输入 Cloudflare Global API Key: " CF_KEY_INPUT
            else
              reading " Please enter Cloudflare Email: " CF_EMAIL_INPUT
              reading " Please enter Cloudflare Global API Key: " CF_KEY_INPUT
            fi
            if [ -z "$CF_EMAIL_INPUT" ] || [ -z "$CF_KEY_INPUT" ]; then
              [ "$L" = "C" ] && warning "Email 或 Key 不能为空！" || warning "Email or Key cannot be empty!"
              sleep 2
              manage_acme_certificate_menu
              return
            fi
            export CF_Email="$CF_EMAIL_INPUT"
            export CF_Key="$CF_KEY_INPUT"
            unset CF_Token
          fi

          # 申请证书 (DNS-01 CF 验证)
          /root/.acme.sh/acme.sh --issue --dns dns_cf -d "$CERT_DOMAIN" --keylength ec-256 --force
        fi

        # 检查并安装证书
        if [ -s "/root/.acme.sh/${CERT_DOMAIN}_ecc/${CERT_DOMAIN}.key" ] && [ -s "/root/.acme.sh/${CERT_DOMAIN}_ecc/fullchain.cer" ]; then
          [ ! -d ${WORK_DIR}/cert ] && mkdir -p ${WORK_DIR}/cert

          # 通过 acme.sh 官方命令进行安装，设置自动续期 reloadcmd，续期后自动重启 sing-box
          /root/.acme.sh/acme.sh --install-cert -d "$CERT_DOMAIN" --ecc \
            --key-file "${WORK_DIR}/cert/private.key" \
            --fullchain-file "${WORK_DIR}/cert/cert.pem" \
            --reloadcmd "cp ${WORK_DIR}/cert/cert.pem ${WORK_DIR}/cert/cert_200.pem && systemctl restart sing-box"

          # 拷贝一份给 NaiveProxy 使用
          cp "${WORK_DIR}/cert/cert.pem" "${WORK_DIR}/cert/cert_200.pem"

          # 自动修正现有 inbound 配置文件中的域名
          for FILE in ${WORK_DIR}/conf/*_inbounds.json; do
            if [ -s "$FILE" ] && grep -q 'certificate_path' "$FILE"; then
              sed -i "s/\"server_name\":.*/\"server_name\":\"$CERT_DOMAIN\",/g" "$FILE"
            fi
          done

          [ "$L" = "C" ] && info "Let's Encrypt 证书申请与安装成功！已配置每日自动检查与续期。" || info "Let's Encrypt certificate applied & installed successfully! Auto-renew and restart scheduled."
        else
          if [ "$ACME_MODE" = "1" ]; then
            [ "$L" = "C" ] && error "证书申请失败，请检查服务器 80 端口是否放行，且未被占用！" || error "Certificate application failed. Check if port 80 is open and not in use!"
          else
            [ "$L" = "C" ] && error "证书申请失败，请检查 Cloudflare Token/Key 是否正确，且域名解析是否在 CF 托管！" || error "Certificate application failed. Check if Cloudflare Token/Key is correct and domain is managed by CF!"
          fi
        fi

        # 恢复服务 (仅 HTTP-01 模式需要手动拉起，DNS 模式下无需拉起，因为没有停止过)
        if [ "$ACME_MODE" = "1" ]; then
          systemctl start nginx >/dev/null 2>&1 || true
          systemctl start caddy >/dev/null 2>&1 || true
          systemctl start sing-box >/dev/null 2>&1 || true
        fi
      else
        [ "$L" = "C" ] && error "未找到 acme.sh，安装失败！" || error "acme.sh not found, installation failed!"
      fi
  ```

- [ ] **步骤 2：运行语法语法检查**
  运行：`bash -n D:/Work/scripts/sing-box/sing-box.sh`
  预期：无语法错误。

- [ ] **步骤 3：Commit**
  ```bash
  git add sing-box.sh
  git commit -m "feat: add DNS-01 API challenge support and optimize standalone HTTP-01 hooks"
  ```

---

### 任务 3：优化证书回退逻辑，清理残留

**文件：**
- 修改：`D:/Work/scripts/sing-box/sing-box.sh:5882-5888`

- [ ] **步骤 1：在 `manage_acme_certificate_menu` 的 case 4) 中加入物理删除逻辑**
  在 `[[ "${REVERT_CONFIRM,,}" = "y" ]]` 逻辑中，原有的 `acme.sh --remove` 执行后加入清理残留文件夹的代码：
  ```bash
        [ "$L" = "C" ] && info "正在清理 acme.sh 证书续期计划..." || info "Cleaning up acme.sh cert schedule..."
        if [ -n "$CURRENT_DOMAIN" ] && [ -x "/root/.acme.sh/acme.sh" ]; then
          /root/.acme.sh/acme.sh --remove -d "$CURRENT_DOMAIN" --ecc >/dev/null 2>&1 || true
          # 物理清除残留证书文件
          if [ -d "/root/.acme.sh/${CURRENT_DOMAIN}_ecc" ]; then
            rm -rf "/root/.acme.sh/${CURRENT_DOMAIN}_ecc"
          fi
        fi
  ```

- [ ] **步骤 2：运行语法检查**
  运行：`bash -n D:/Work/scripts/sing-box/sing-box.sh`
  预期：无语法错误。

- [ ] **步骤 3：Commit**
  ```bash
  git add sing-box.sh
  git commit -m "feat: clean up physical certificate files from acme.sh when reverting to self-signed"
  ```
