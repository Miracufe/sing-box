# 快捷方式容错防无反应优化实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 在 `sing-box.sh` 中重构快捷方式生成函数 `create_shortcut`，使在 `wget` 失败或拉取内容为空（如 GitHub 404 或解析超时）时，能够显式报错，杜绝“敲 sb 没有任何反应直接换行”的体验。

**架构：**
1. 重构 `/usr/bin/sb` 生成的 Bash 内容，通过本地临时变量保存 `wget` 返回的内容。
2. 校验此内容，非空时再通过 `bash <(echo "$SCRIPT_CONTENT")` 启动。
3. 否则，输出带红字的高亮提示：`Error: Failed to fetch the sing-box.sh script from GitHub...`。

**技术栈：** Bash Shell, wget

---

### 任务 1：优化快捷运行文件 sb.sh 生成逻辑

**文件：**
- 修改：`D:/Work/scripts/sing-box/sing-box.sh` 中的 `create_shortcut()` 函数

- [ ] **步骤 1：分析定位并修改快捷方式生成代码**
  修改 `sing-box.sh` 约 5271-5280 行：
  ```bash
  create_shortcut() {
    cat > ${WORK_DIR}/sb.sh << EOF
  #!/usr/bin/env bash

  bash <(wget --no-check-certificate -qO- https://raw.githubusercontent.com/Miracufe/sing-box/main/sing-box.sh) \$@
  EOF
    chmod +x ${WORK_DIR}/sb.sh
    ln -sf ${WORK_DIR}/sb.sh /usr/bin/sb
    [ -s /usr/bin/sb ] && info "\n \$(text 71) "
  }
  ```
  替换为以下包含安全验证的逻辑：
  ```bash
  create_shortcut() {
    cat > ${WORK_DIR}/sb.sh << EOF
  #!/usr/bin/env bash
  SCRIPT_CONTENT=\$(wget -T 10 -t 2 --no-check-certificate -qO- https://raw.githubusercontent.com/Miracufe/sing-box/main/sing-box.sh)
  if [ -n "\$SCRIPT_CONTENT" ]; then
    bash <(echo "\$SCRIPT_CONTENT") "\$@"
  else
    echo -e "\033[31mError: Failed to fetch the sing-box.sh script from GitHub.\033[0m"
    echo -e "Please check if the URL 'https://raw.githubusercontent.com/Miracufe/sing-box/main/sing-box.sh' is accessible."
  fi
  EOF
    chmod +x ${WORK_DIR}/sb.sh
    ln -sf ${WORK_DIR}/sb.sh /usr/bin/sb
    [ -s /usr/bin/sb ] && info "\n \$(text 71) "
  }
  ```

- [ ] **步骤 2：进行 Bash 语法静态检查**
  运行：`bash -n D:/Work/scripts/sing-box/sing-box.sh`
  预期：无语法错误。

- [ ] **步骤 3：编写本地测试脚本测试快捷方式在异常情况下的报错回显**
  在本地创建一个模拟快捷方式的测试文件：`D:/Work/scripts/sing-box/scratch/test_sb_error.sh`：
  ```bash
  #!/usr/bin/env bash
  # 故意使用一个 404 地址
  SCRIPT_CONTENT=$(wget -T 5 -t 1 --no-check-certificate -qO- https://raw.githubusercontent.com/Miracufe/sing-box/main/non_exist_file.sh)
  if [ -n "$SCRIPT_CONTENT" ]; then
    bash <(echo "$SCRIPT_CONTENT") "$@"
  else
    echo -e "\033[31mError: Failed to fetch the sing-box.sh script from GitHub.\033[0m"
    echo -e "Please check if the URL 'https://raw.githubusercontent.com/Miracufe/sing-box/main/non_exist_file.sh' is accessible."
  fi
  ```
  在本地运行它：`bash D:/Work/scripts/sing-box/scratch/test_sb_error.sh`
  预期：秒级回显红字 `Error: Failed to fetch the sing-box.sh script from GitHub...`，而非静默卡死。
  测试通过后，删除该临时脚本。

- [ ] **步骤 4：Commit**
  ```bash
  git add sing-box.sh
  git commit -m "feat: improve sb shortcut to exit gracefully with red error message upon remote script download failure"
  ```
