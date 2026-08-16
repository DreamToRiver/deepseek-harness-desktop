# DeepSeek Harness Desktop

DeepSeek Harness（DSH）的 Windows x64 桌面版。它使用 Electron 提供桌面窗口和系统托盘，并在安装包内置 DSH 服务与 Node.js 运行时，安装后即可直接使用。

## 下载

普通用户无需克隆源码或安装 Node.js，直接前往 [Releases](https://github.com/DreamToRiver/deepseek-harness-desktop/releases) 下载最新的 `DeepSeek-Harness-Setup-*.exe`。

当前版本：[v0.1.1](https://github.com/DreamToRiver/deepseek-harness-desktop/releases/tag/v0.1.1)

## 首次使用与 API Key

首次启动时，应用会引导用户配置自己的 DeepSeek API Key。项目源码和安装包不包含开发者的 API Key；每位用户的配置保存在自己的 `%USERPROFILE%\.dsh` 目录中，并与 DSH 命令行版本共享。

应用按以下顺序查找凭据：环境变量 `DEEPSEEK_API_KEY`、`%USERPROFILE%\.dsh\.credentials.yaml`、项目 `.env`、`%DSH_HOME%\.env`。仓库通过 `.gitignore` 排除凭据和环境文件。

## 架构

- **Electron 外壳**：负责「窗口 + 生命周期 + 首次引导」，启动后拉起 DSH 后端并把窗口指向本地地址。
- **DSH 后端**：作为子进程，用内置的 `node.exe` 运行 `@deepseek-ai/dsh`（`node server/runtime/node_modules/@deepseek-ai/dsh/lib/bin.js web --port 0`）。
  - `--port 0` 让系统分配空闲端口，外壳从 stdout 解析 `dsh web: http://127.0.0.1:<port>` 后加载，彻底避免端口冲突。
  - DSH 运行在真正的 Node 上，node-pty / sharp / koffi 等原生模块 ABI 完全匹配。
- **数据目录**：沿用 DSH 默认 `~/.dsh`（即 `%USERPROFILE%\.dsh`），与命令行版共享配置、API Key 与历史会话。
- **应用内页面 = 网站**：窗口加载的就是 DSH 网页本体（同一个服务端与前端），样式、设置内容与浏览器打开网站完全一致；自带的启动页与首次引导页按 DSH 深色主题令牌（#0F1115 画布 / #5686FE 品牌蓝）绘制。
- **首次引导**：启动时按 DSH 的凭据优先级检测 API Key（环境变量 `DEEPSEEK_API_KEY` > `$DSH_HOME/.credentials.yaml` > 项目 `.env` > `$DSH_HOME/.env`）。无 Key 时显示引导页，保存后写入 `$DSH_HOME/.credentials.yaml`（与网页版「设置 → 模型」共用同一凭据存储），之后打开自动跳过；也可「暂时跳过」后在应用内设置里配置。
- **图标**：应用 exe、setup.exe、快捷方式统一使用 `build/source-icon.png`（用户提供的 DSH.png）生成的多尺寸 `build/icon.ico`。
- **系统托盘**：点击窗口右上角关闭 → 最小化到系统托盘（任务栏图标消失、后台服务继续运行）；点击托盘图标重新打开，右键可「退出」。首次关闭会弹气泡提示。
- **服务中断恢复**：DSH 后端意外停止时弹出中文提示框，可选「重新启动」（原地拉起新服务，无需退出应用）或「退出」；退出后再次打开桌面版即可正常使用（随机端口无冲突，配置与历史会话保留在 ~/.dsh）。
- **安装向导**：仿 WorkBuddy 风格 —— 品牌蓝侧栏（164×314，白色 logo + 产品名）+ 白色页眉 + 中文欢迎/完成页文案（`build/installerSidebar.bmp`、`build/installerHeader.bmp`、`build/installer.nsh`）。安装与卸载向导均已重做：安装欢迎页「欢迎使用 DeepSeek Harness」，卸载欢迎页「卸载 DeepSeek Harness」（注明不删除 ~/.dsh 数据）；卸载进度页同样使用品牌页眉。
  - 实测：通过枚举真实向导窗口控件文本验证 —— 安装向导「欢迎使用 DeepSeek Harness」✓、卸载向导「卸载 DeepSeek Harness」/「本向导将从您的电脑中卸载」/「卸载不会删除您的个人配置」✓、两侧栏像素 #5686FE 品牌蓝 ✓。

## 目录结构

```
├── main.js                Electron 主进程
├── preload.js             安全的渲染进程桥接
├── package.json           应用清单与 electron-builder 配置
├── build/                 应用图标与 NSIS 安装向导资源
├── scripts/               图标、构建和端到端测试脚本
├── server/runtime/        内置 DSH 运行时（node.exe + node_modules）
└── dist/                  本地构建产物（不会提交到仓库）
```

## 构建

构建安装包需要 Node.js、npm 和网络连接；最终用户不需要预装这些环境。

```powershell
# 安装项目依赖
npm ci

# 生成图标
npm run icon

# 打包为 Windows x64 安装包，产物在 dist/
npm run dist
```

仓库中的 `server/runtime` 已包含可独立运行的 DSH 后端及其依赖，不需要从开发者电脑复制文件。

## 系统要求（目标机器）

- Windows 10 / 11，64 位
- **无需任何预装环境**：Node.js、npm、Electron、VC++ 运行库、PowerShell 7 均不需要（安装包内置 Node 24 + 全部依赖；Shell 工具自动回退到 Windows 自带的 PowerShell 5.1）
- 磁盘约 500 MB（安装后）
- 联网：应用启动不依赖网络，但使用 AI 功能需要能访问 DeepSeek API
- 实测：PATH 仅保留系统目录（模拟全新机器）时，内置 node 正常运行、应用完整启动（DSH 服务 + 托盘）✓

## 产物

- `dist/DeepSeek-Harness-Setup-0.1.1.exe` —— NSIS 安装向导（每用户安装，无需管理员权限，可选择安装目录，自动创建开始菜单/桌面快捷方式，简体中文界面）。
- 卸载方式（任选其一）：开始菜单 →「卸载 DeepSeek Harness」；Windows 设置 → 应用 → 已安装的应用 → DeepSeek Harness → 卸载；或运行安装目录中的 `Uninstall DeepSeekHarness.exe`。卸载不会删除 `~/.dsh` 中的会话与配置数据。
- 已实测：静默安装 → 启动（内置 DSH 服务正常拉起、窗口指向解析出的本地地址）→ 静默卸载（目录、开始菜单/桌面快捷方式、注册表项全部清理）。

## 注意事项

- 安装包未做代码签名（需要购买代码签名证书），首次运行 Windows SmartScreen 可能提示「未知发布者」，选择「仍要运行」即可。
- 应用关闭时后端进程一并结束；DSH 会话持久化是逐批 fsync 的，最多丢失最后一批未落盘事件。
- 修改 `main.js` 后重新执行 `npm run dist` 即可产出新安装包。
