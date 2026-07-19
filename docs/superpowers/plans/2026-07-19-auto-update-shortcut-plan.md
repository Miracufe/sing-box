# 启动时自动静默更新快捷方式实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 在 `sing-box.sh` 的启动分支中，针对直接运行脚本查看菜单的用户，自动且静默地执行一次 `create_shortcut`。由此保证老用户的快捷方式在脚本更新后能无缝自动同步更新，杜绝网络 404 地址遗留。

**架构：**
1. 定位 `sing-box.sh` 脚本最尾部的启动判断逻辑。
2. 在 `else` 分支的第一行插入 `create_shortcut >/dev/null 2>&1`。
3. 确保此操作是静默的，不打印“快捷方式创建成功”的多余回显干扰菜单展现。

**技术栈：** Bash Shell

---

### 任务 1：重构脚本启动逻辑，注入静默更新函数

**文件：**
- 修改：`D:/Work/scripts/sing-box/sing-box.sh` 结尾部分启动分支（约 6328-6331 行）

- [ ] **步骤 1：分析定位并修改启动逻辑**
  定位脚本尾部（约 6328-6331 行）：
  ```bash
  else
    menu_setting
    menu
  fi
  ```
  修改为在进入菜单前自动静默更新快捷链接：
  ```bash
  else
    create_shortcut >/dev/null 2>&1
    menu_setting
    menu
  fi
  ```

- [ ] **步骤 2：进行语法静态检查**
  运行：`bash -n D:/Work/scripts/sing-box/sing-box.sh`
  预期：无语法错误。

- [ ] **步骤 3：编写本地测试脚本，验证在普通启动（else 分支）时会成功重新写入 sb.sh**
  在本地创建一个模拟的主启动逻辑测试文件 `D:/Work/scripts/sing-box/scratch/test_auto_update.sh`：
  ```bash
  #!/bin/bash
  WORK_DIR="D:/Work/scripts/sing-box/scratch"
  mkdir -p "$WORK_DIR"
  
  # 模拟 create_shortcut 函数
  create_shortcut() {
    echo "writing to $WORK_DIR/sb.sh"
    echo "#!/bin/bash" > "$WORK_DIR/sb.sh"
  }
  
  # 模拟启动分支
  # 故意清空 sb.sh 模拟文件丢失或旧状态
  rm -f "$WORK_DIR/sb.sh"
  
  # else 分支
  create_shortcut >/dev/null 2>&1
  
  # 验证是否成功重新生成了 sb.sh
  if [ -s "$WORK_DIR/sb.sh" ]; then
    echo "Test PASS: sb.sh was automatically updated on startup!"
  else
    echo "Test FAIL: sb.sh was not generated!"
  fi
  ```
  在本地运行它：`bash D:/Work/scripts/sing-box/scratch/test_auto_update.sh`
  预期：输出 `Test PASS: sb.sh was automatically updated on startup!`。
  测试通过后，删除该临时测试脚本。

- [ ] **步骤 4：Commit**
  ```bash
  git add sing-box.sh
  git commit -m "feat: automatically update the shortcut link on script startup to prevent stale repo URL link hangs"
  ```
