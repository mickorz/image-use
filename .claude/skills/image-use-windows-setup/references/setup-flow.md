# image-use Windows 环境配置流程

按阶段执行，每阶段末尾的验证点通过后才继续。`坑N` 是实机踩过并验证过修法的坑。

## 阶段 0：前置检查

```bash
uname -s                                     # 确认是 MINGW64*（Git Bash），不是 WSL
python --version                             # 3.10+
ls "/c/Program Files/Google/Chrome/Application/chrome.exe"   # Chrome 已安装
ls E:/DesignProjects/image-use/image-use     # image-use 仓库存在
```

任一失败：先补齐（Python、克隆仓库），Chrome 未装则让用户安装后再继续。

## 阶段 1：代理探测

GitHub 直连在境内网络通常被重置（`Recv failure: Connection was reset`）。
系统代理（IE 设置）一般是关闭的，但 Clash 类工具会监听本地端口：

```bash
netstat -ano | grep -E "127.0.0.1:(7890|7897|1080|10809|8888)" | grep ESTABLISHED | head -3
```

有 ESTABLISHED 连接的端口即为代理端口（本机为 7897）。找不到就问用户代理端口；
用户确认无代理则本流程阻塞（GitHub 资源下不来）。

验证代理可用：

```bash
export https_proxy=http://127.0.0.1:7897 http_proxy=http://127.0.0.1:7897
curl -fsSL --connect-timeout 15 https://raw.githubusercontent.com/leeguooooo/chrome-use/main/install.ps1 -o "$USERPROFILE/chrome-use-install.ps1" && echo OK
```

## 阶段 2：安装 chrome-use CLI

**坑1：`install.sh` 不支持 Windows。** `uname -s` 返回 `MINGW64_NT-*`，sh 脚本直接报
`unsupported OS`。Windows 必须用 `install.ps1`。

**坑2：PowerShell 下载不走代理。** 系统代理没开时，`Invoke-WebRequest` / `HttpWebRequest`
默认直连失败。必须在 PowerShell 会话里显式设置 `WebRequest.DefaultWebProxy`。
注意：.NET 不读 `http_proxy` 环境变量，设了也没用。

**坑3：MSYS tar 抢占导致解压失败。** 从 Git Bash 启动的 PowerShell 继承其 PATH，
`tar.exe` 解析到 MSYS 的 tar，遇到 `C:\` 路径报 `Cannot connect to C: resolve failed`。
修法：把 `C:\Windows\System32` 提到该 PowerShell 会话 PATH 最前面，用系统自带 bsdtar。

一条完整命令（从 Git Bash 执行，注意 `\$` 转义）：

```bash
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "\$env:Path = 'C:\Windows\System32;C:\Windows;' + \$env:Path; [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; [System.Net.WebRequest]::DefaultWebProxy = New-Object System.Net.WebProxy('http://127.0.0.1:7897'); & 'C:\Users\<用户名>\chrome-use-install.ps1'"
```

安装脚本会自动：装到 `%LOCALAPPDATA%\Programs\chrome-use`、写入用户 PATH（注册表）、
安装 chrome-use agent skill 到各 agent 目录、跑 `doctor --quick --offline` 自检。

验证点：输出末尾 `CLI installation complete`，且 `chrome-use.exe` 存在于
`%LOCALAPPDATA%\Programs\chrome-use\`。

## 阶段 3：Native Host + Chrome 扩展

```bash
"$LOCALAPPDATA/Programs/chrome-use/chrome-use.exe" extension install < /dev/null
```

输出 `[1/2] ✓ registered (8 browser registry keys)` 即 Native Host 注册成功。
第二步会打印 Web Store 扩展页地址。打开它给用户（坑4：Git Bash 里 `cmd /c start` 会
把 URL 拆碎，改用 PowerShell）：

```bash
powershell.exe -NoProfile -Command "Start-Process 'https://chromewebstore.google.com/detail/knfcmbamhjmaonkfnjhldjedeobeafmk'"
```

**用户手动步骤（Windows 无静默安装策略，只能点一次）：**

1. 在打开的页面点「添加至 Chrome」→「添加扩展程序」
2. 确认该 Chrome 已登录 chatgpt.com（没登录先登录）

用 ask_user_question 等用户确认两步都完成后再验证。

验证点：

```bash
"$LOCALAPPDATA/Programs/chrome-use/chrome-use.exe" extension status
```

必须看到 `✓ live extension version: 0.5.x` 和 `✓ driving Chrome profile: <账号>`。

## 阶段 4：编码修复（必做）

**坑5：GBK 解码崩溃。** 中文 Windows 上 Python 子进程按 GBK 解码 chrome-use 的
UTF-8 输出，reader 线程直接 `UnicodeDecodeError` 崩掉，表现为
`could not confirm ChatGPT received the prompt`（此时提示词可能已发出）。
必须持久化 UTF-8 模式：

```bash
setx PYTHONUTF8 1
```

对新终端生效；当前已开的终端需要重开，或临时 `export PYTHONUTF8=1`。

## 阶段 5：doctor 验证

```bash
export PATH="/c/Users/<用户名>/AppData/Local/Programs/chrome-use:$PATH" PYTHONUTF8=1
cd E:/DesignProjects/image-use && python image-use doctor
```

**坑6：Git Bash 的 PATH 前插必须用 POSIX 风格**（`/c/Users/...`），写 Windows 风格
（`C:\Users\...`）会被 Git Bash 丢弃，子进程里 `shutil.which` 找不到 chrome-use。

预期结果：`chrome-use [ok]`、`relay connected [ok]`。

**坑7：`profiles none detected` 可忽略。** image-use 0.34.5 的离线 cookie 扫描只查
macOS/Linux 路径，不含 Windows 的 `%LOCALAPPDATA%\Google\Chrome\User Data`，所以
Windows 上永远报这一行 fail。实际生图走 relay 驱动已打开的 Chrome，不受影响。

## 阶段 6：真实出图验证

```bash
python image-use "a simple watercolor cat sitting on a windowsill" -o /tmp/test-web.png --no-style
```

成功标志：`✓ saved → <路径> (<字节数>)`。全程约 100-170 秒。

**运行时坑A：输入框残留草稿。** 上次失败运行会把提示词留在 ChatGPT 输入框，下次运行报
`ChatGPT composer is not empty; prompt not submitted`（工具为防重复发送拒绝覆盖）。
修法（chrome-use 三步清空）：

```bash
chrome-use open "https://chatgpt.com/"
chrome-use snapshot -i -c          # 找输入框 textbox 的 @ref（形如 @e114）
chrome-use click @e114 && chrome-use press Control+a && chrome-use press Delete
```

清空后重跑即可。

**非致命警告（不处理）：** 结束时偶见 `could not delete conversation (CDP error ...
relay timeout)`——只是删除对话失败，图片已保存，对话留在 imagegen 项目历史里。

## 常见坑速查表

| 症状 | 根因 | 修法 |
| --- | --- | --- |
| `Connection was reset` 下载失败 | GitHub 直连被重置 | 挂本地代理（阶段1），且 .NET 要显式设 DefaultWebProxy（坑2） |
| `unsupported OS: MINGW64*` | install.sh 不支持 Windows | 改用 install.ps1（坑1） |
| `Cannot connect to C: resolve failed` | MSYS tar 抢占 tar.exe | PowerShell 会话内 System32 提到 PATH 最前（坑3） |
| `UnicodeDecodeError: 'gbk' codec` | Python 按 GBK 解码 UTF-8 输出 | `setx PYTHONUTF8 1`（坑5） |
| `shutil.which` 找不到 chrome-use | Git Bash PATH 前插了 Windows 风格路径 | 用 `/c/...` POSIX 风格（坑6） |
| `composer is not empty` | 上次失败残留草稿 | chrome-use 清空输入框（坑A） |
| doctor 报 `profiles none detected` | 离线扫描缺 Windows 路径 | 可忽略，relay 不受影响（坑7） |
| `cmd start` 打开 URL 失败 | cmd 解析 URL 拆参 | 改用 `powershell Start-Process`（坑4） |
