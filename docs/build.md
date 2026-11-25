# 虎符 HuFu · 源码构建指南

> 本页面向 Wubi86 衍生仓库。项目身份、本机部署与回滚见 [`AI_HANDOFF.md`](AI_HANDOFF.md)。

## 1. 环境

- Windows 10/11，PowerShell 7。
- Node.js 22（与 GitHub Actions 一致）。
- Rust stable，至少安装 `x86_64-pc-windows-msvc`。
- 如需重建 TSF，另需 `x86_64-pc-windows-gnu`；服务端 CI 与 TSF 生产构建是两条不同工具链。

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi'
node --version
cargo --version
rustup target list --installed
```

## 2. 核心引擎测试

```powershell
Set-Location engine
cargo test --locked --target x86_64-pc-windows-msvc `
  -p hufu-config -p hufu-dict -p hufu-engine
```

2026-11-25 本地结果：206 通过、0 失败（含 doc tests）。修改代码后必须重新运行，不要沿用旧数量。

## 3. 设置页与服务端

**顺序不能颠倒**：`hufu-server` 的 `build.rs` 会把 `settings-app/dist` 嵌入二进制；先构建设置页，才能让 `/next` 包含最新 UI。

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi\settings-app'
npm ci
npm run build

Set-Location '..\engine'
cargo build --locked --release --target x86_64-pc-windows-msvc `
  -p hufu-server -p hufu-render-helper
```

`settings-app/dist` 有意纳入版本控制：干净 clone 可以直接构建服务端，发布机不需要 Node。修改 React 源码后仍必须先运行 `npm run build`，并把重新生成的 `dist/index.html` 与哈希资源一起提交；不要只提交源码而留下旧 dist。

产物：

- `engine\target\x86_64-pc-windows-msvc\release\hufu-server.exe`
- `engine\target\x86_64-pc-windows-msvc\release\hufu-render-helper.exe`

设置页开发服务器：

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi\settings-app'
npm run dev
```

默认地址 `http://127.0.0.1:5173`。修改 UI 后必须在浏览器验证，不能只依赖 TypeScript 构建。

### 检查设置页是否嵌入

```powershell
$index = [IO.File]::ReadAllText(
  (Resolve-Path 'settings-app/dist/index.html'),
  [Text.Encoding]::UTF8
)
$server = [Text.Encoding]::UTF8.GetString(
  [IO.File]::ReadAllBytes(
    (Resolve-Path 'engine/target/x86_64-pc-windows-msvc/release/hufu-server.exe')
  )
)
if (-not $server.Contains($index)) {
  throw 'hufu-server.exe 未嵌入当前 settings-app/dist/index.html'
}
```

## 4. Windows TSF 前端

生产 TSF 使用 GNU 工具链。只有需要改前端、重新注册或验证完整输入法时才构建。

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi\platform\windows'
cargo build --release --target x86_64-pc-windows-gnu -p hufu-tsf
./target/x86_64-pc-windows-gnu/release/hufu-tsf-smoke.exe
```

注册脚本：

```powershell
powershell -ExecutionPolicy Bypass -File `
  'platform\windows\install\install.ps1'
```

注意：

- 启动早于注册的应用需要重启后才能看到 HuFu。
- `ctfmon` 重启或注销后，当前输入法可能重置为系统默认。
- `platform/windows/.cargo/config.toml` 的 i686 配置写死了上游开发者路径；不要在其他机器直接构建 32 位 DLL。
- 本衍生版没有修改 TSF 源码，五笔短码补全和注释都在服务端/引擎/设置页。不要为了验证服务端改动而随意覆盖 TSF。

## 5. 本机替换服务端

不要覆盖正在运行的 exe。完整流程见 [`AI_HANDOFF.md` §6](AI_HANDOFF.md#6-本机部署与回滚)：

1. 构建并检查设置页内嵌。
2. 计算新 exe SHA-256。
3. 优雅关闭旧服务端。
4. 将旧 exe 改名到新备份目录。
5. 复制新 exe、核对哈希、启动。
6. 验证 `/api/config`、`/next?shell=1` 和五笔 `tj` 输入。
7. 失败时停止服务端并恢复备份。

## 6. GitHub Actions

- Windows 工作流：[`.github/workflows/windows-build.yml`](../.github/workflows/windows-build.yml)
- Linux 最小集：[`.github/workflows/ci.yml`](../.github/workflows/ci.yml)

Windows 工作流覆盖核心测试、设置页、服务端和渲染助手，并上传含 SHA256 清单的产物。它**不覆盖 TSF DLL、32 位前端、完整安装包或真实输入法交互**。

## 7. Linux fcitx5 前端

```bash
cmake -S platform/linux/hufu-addon -B platform/linux/build \
    -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
cmake --build platform/linux/build -j
platform/linux/install.sh
fcitx5 -r -d && fcitx5-configtool
```

安装、卸载、功能与作者声明见 [`platform/linux/README.md`](../platform/linux/README.md)。

## 8. macOS 前端

开发中的 IMK 骨架，尚未完成 Mac 实机验证，也没有安装包。结构见 [`platform/macos/README.md`](../platform/macos/README.md)。

## 9. 发行脚本边界

`发行脚本/release.ps1`、`pack.ps1` 等保留上游写作时的个人路径、tag/push 与发布流程，**不要直接照跑**。当前可信的自动构建入口是 GitHub Actions；完整衍生版安装包与版本规则仍待补齐。
