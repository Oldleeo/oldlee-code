# Oldlee Code：Windows 桌面版构建与改造指南

这份文档给要在 Windows 上改 Oldlee Code 的人，或者接手的 AI。只讲 Windows 桌面版，不涉及 Mac、Linux、手机端。

仓库是 pnpm 工作区（monorepo），桌面版是 Electron 应用，里面自带一个 Node 服务器。

## 1. 要装什么

| 东西 | 版本 | 说明 |
| --- | --- | --- |
| Windows | 10 / 11，x64 | 当前只构建 x64 |
| Git for Windows | 任意较新版本 | 克隆时要开长路径，见下 |
| Vite+（`vp`） | 1.8.0 | 它会按 `package.json` 自动装好 Node 和 pnpm，不用单独装 |
| Node.js | 26.13.1 | 由 `vp` 管理（`package.json` 的 `engines.node`） |
| pnpm | 12.14.0 | 由 `vp` 管理（`packageManager`） |
| Rust（rustup） | stable | 打包时要编译 `native/resource-monitor` |
| Python | 3.x | node-gyp 用 |
| Visual Studio Build Tools 2022 | 含「使用 C++ 的桌面开发」、Windows SDK、MSVC 生成工具、**带 Spectre 缓解的库** | 少了 Spectre 库，打包前检查会报错 |

不需要 bun。不需要 Mac 或 Linux 的任何工具。

安装 `vp`（PowerShell）：

```powershell
irm https://vite.plus/ps1 | iex
```

安装 Rust 并加目标：

```powershell
winget install Rustlang.Rustup
rustup target add x86_64-pc-windows-msvc
```

Spectre 库：打开 Visual Studio Installer，选 Build Tools，修改，在「单个组件」里勾选 `MSVC v143 - VS 2022 C++ x64/x86 Spectre 缓解库（最新）`。

## 2. 取代码

```powershell
git clone -c core.longpaths=true https://github.com/Oldleeo/oldlee-code.git
cd oldlee-code
```

如果拿到的是 zip（已含 `.git`），解压后在目录里执行 `git config core.longpaths true` 即可。

`.repos/` 是随仓库带的参考代码（effect、alchemy-effect），构建和运行都用不到，很大。想省空间可以跳过它：

```powershell
git sparse-checkout set --no-cone "/*" "!/.repos/"
```

## 3. 装依赖并本地运行

```powershell
vp i
vp run dev:desktop      # 开发模式启动 Electron 桌面版
```

其他常用命令：

```powershell
vp run dev              # 服务器 + 网页，浏览器里看（要用开发运行器打印的配对链接打开）
vp run typecheck        # 类型检查
vp run lint
vp run fmt
vp run test             # 全部测试
```

注意：

- 开发数据默认放在 `%USERPROFILE%\.t3\dev`。不要让开发服务器指向 `%USERPROFILE%\.t3\userdata`，那是正式使用的数据库。
- 开发时不要设 `VITE_HTTP_URL`、`VITE_WS_URL`。
- 命令行名字仍叫 `t3`，环境变量仍是 `T3CODE_*`，数据目录仍是 `.t3`。这是有意保留的内部标识，改名会破坏兼容。
- 第一次启动会用 Cargo 编译 `native/resource-monitor`。没装 Rust 也能启动，只是没有进程监控数据。

## 4. 打 Windows 安装包（.exe）

在 Windows 上，仓库根目录执行：

```powershell
vp run dist:desktop:artifact --platform win --target nsis --arch x64 --build-version 0.1.12 --verbose
```

产物在 `release\`：`Oldlee-Code-0.1.12-x64.exe`（NSIS 安装程序，约 206 MB）。装好后主程序叫 `Oldlee Code (Beta).exe`。版本号取自 `apps/desktop/package.json`（当前 0.1.12）。

打包脚本在 Windows 主机上还会真的启动一次打好的程序做自检。有一道「Windows 包文件数不超过 80」的检查，在当前依赖下实际是 104 到 106 个文件，会报错。所以打包时要设环境变量放宽（工作流里已经设了）：

```powershell
$env:T3CODE_WINDOWS_PAYLOAD_FILE_LIMIT = "300"
```

再出一个免安装 zip（要先成功跑过上一步，用 `--skip-build` 复用构建结果）：

```powershell
vp run dist:desktop:artifact --platform win --target zip --arch x64 --build-version 0.1.12 --skip-build
```

打包脚本会在构建时检查 Windows 包的结构、原生模块能否加载等。缺 Rust、Python、MSVC 或 Spectre 库时，它会列出缺哪一项。

### 不想装 Windows 工具链：用 GitHub Actions

仓库里有手动工作流 `.github/workflows/windows-build.yml`，在 GitHub 提供的 `windows-latest` 机器上构建，不需要签名密钥：

```powershell
gh workflow run windows-build.yml -f version=0.1.12
gh run watch
gh run download <run-id> -n oldlee-code-windows-x64
```

原仓库其余工作流（`release.yml`、`release-desktop.yml` 等）依赖上游的 Blacksmith 付费跑机和一堆签名、部署密钥，在这个仓库里不会正常运行，不要拿它们当 Windows 构建入口。

## 5. 代码签名与 SmartScreen

当前没有代码签名证书，所以安装包和 exe 都是未签名的。Windows 会弹 SmartScreen 蓝色警告「Windows 已保护你的电脑」，点「更多信息」再点「仍要运行」即可。

要签名，要自己申请证书。脚本支持 Azure Trusted Signing（`--signed` 加 `AZURE_*` 环境变量，见 `docs/operations/03_release.md`）。没有就保持未签名。

## 6. 和 Windows 桌面版有关的目录

| 路径 | 作用 |
| --- | --- |
| `apps/desktop/` | Electron 桌面应用本体 |
| `apps/desktop/src/boot.ts` | 入口（`package.json` 的 `main` 指向打包后的 `dist-electron/boot.cjs`） |
| `apps/desktop/src/main.ts` | 主进程 |
| `apps/desktop/src/preload.ts` | 预加载脚本 |
| `apps/desktop/src/window/`、`app/`、`ipc/`、`settings/`、`updates/` | 窗口、应用生命周期、IPC、设置、自动更新 |
| `apps/desktop/src/wsl/` | Windows 上经 WSL 运行代理的支持 |
| `apps/desktop/resources/installer.nsh` | NSIS 安装程序脚本 |
| `apps/server/` | 内置的 Node WebSocket 服务器（包名 `t3`），包装各个代理命令行 |
| `apps/web/` | 界面（React + Vite），桌面版和浏览器都用它。笔记本风格的界面主要改这里 |
| `packages/*` | 共享代码：`client-runtime`、`contracts`、`shared`、各代理的 `provider-*` 适配等 |
| `native/resource-monitor/` | Rust 写的进程监控，打包时一起编译成 `t3-resource-monitor.exe` |
| `scripts/build-desktop-artifact.ts` | 打包总入口：暂存应用、跑 electron-builder、做 Windows 检查 |
| `scripts/verify-windows-desktop-artifact.ts` | 校验已打好的 Windows 包 |
| `scripts/lib/windows-jsign.ts`、`windows-tar.ts` | Windows 签名、打包辅助 |
| `assets/prod`、`assets/dev`、`assets/nightly` | 应用图标（Windows 用 `.ico`） |
| `.github/readme/` | README 用的标志和头图 |
| `.github/workflows/windows-build.yml` | 手动 Windows 构建工作流 |

## 7. 给改代码的 AI 的提示

- 先读根目录 `AGENTS.md` 和 `CLAUDE.md`，里面有这个项目的约定。
- 改界面主要在 `apps/web/src/`。改完要同时考虑网页和桌面两种用法。
- 提交前跑 `vp run typecheck`、`vp run lint`、`vp run fmt`、相关的 `vp run test`。
- 不要改 `LICENSE`。MIT 要求保留原版权声明。
- 图标的 Icon Composer 工程里还是旧标志，不要用 `icons:export` 覆盖新图标。

## 8. 已知限制

- 未签名，会有 SmartScreen 警告。
- Oldlee Connect（账号登录、中继）依赖上游托管服务，本项目没有运营这些服务。局域网配对和 Tailscale 不受影响。要用 Connect，得自己配 Clerk 和中继，见 `.env.example` 和 `infra/relay`。
- 自动更新用的是 electron-builder 的 GitHub 发布源，这个仓库还没有 Releases，更新检查找不到新版本。
- 打包脚本里写死了作者 `T3 Tools`（`scripts/build-desktop-artifact.ts` 里 `author: "T3 Tools"`），会出现在包内 `package.json` 里。要改成自己的名字，改这一处再重新打包。
- 软件名目前是 `Oldlee Code (Beta)`，安装包文件名是 `Oldlee-Code-<版本>-x64.exe`，由 `artifactName` 和 `productName` 决定。
- 这个 Windows 包没有内置 WSL 运行时（没传 `--wsl-runtime`）。在 Windows 本机直接用代理没问题，要走 WSL 后端，需要另外提供 Linux 命令行压缩包，见 `docs/operations/03_release.md`。
- 代理本身（Claude Code、Codex 等）要自己装好并登录，软件只负责驱动它们。
- 系统里的 `.env` 不要放服务器密钥。仓库里没有任何密钥或令牌。
