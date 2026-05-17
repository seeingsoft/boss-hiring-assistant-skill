# Windows 环境约束

## ACP 模式约束

Windows ACP（App Control Policy）模式下：
- PowerShell 无法通过 Start-Process 启动 GUI 程序（如 Chrome）
- 解决方案：生成 bat 脚本由用户手动执行，或在 WorkBuddy 聊天窗口中用自然语言指令让 WorkBuddy 代为启动

## Chrome 远程调试启动

Windows 下以远程调试模式启动 Chrome 的推荐方式：

1. 在 WorkBuddy 聊天窗口输入"用远程调试模式启动 Chrome，端口 9222，用户数据目录 C:\tmp\chrome-debug-profile"
2. 如果 WorkBuddy 无法自动执行，生成 bat 脚本：
   ```bat
   start "" "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=C:\tmp\chrome-debug-profile
   ```
3. 用户双击执行 bat 脚本

## Node.js PATH 配置

Windows 安装 Node.js 后必须配置 PATH（详见操作手册步骤 5.3-5.4）。
