# 虎符 HuFu · Wubi86 衍生版当前状态

> 更新日期：2026-11-25。本页只记录当前事实；历史评测、旧版本验收和方案稿保留在 Git 原路径或本地 `docs/archive/`，后者不纳入 Git。

## 仓库与基线

- 唯一工作目录：`F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi`
- 当前分支：`main`
- 上游基线：`v1.9.9.9` / `d438807`
- 本仓库远端：`origin` → `kelsoz/hufu-ime-rust-wubi`
- 上游远端：`upstream` → `LeafHW/hufu-ime-rust`
- 身份与许可：衍生仓库，沿用上游 GPL-3.0-or-later；不是上游官方发行版。

## 本衍生版已落地的改动

1. **Wubi86 前缀补全**
   - 配置：`input.prefix_completion`，默认关闭。
   - 精确候选保持原序，随后追加主码表中编码严格更长且同前缀的候选。
   - 追加候选按权重降序、原序升序；同词保留最优编码，过滤软删并限量。
   - 不混入用户词、个人存储表；反查、快符、通配符和整句锁路径不启用。

2. **候选注释三选一**
   - 配置：`candidates.show_pinyin_comment` 与 `candidates.show_code_comment`。
   - UI 为“关闭 / 拼音 / 五笔”，两者互斥；旧配置只开拼音时保持拼音。
   - 五笔模式显示单字和词语的完整候选编码。
   - 整句模式暂不显示拼音或五笔编码注释。

3. **Windows 自动编译**
   - 工作流：`.github/workflows/windows-build.yml`。
   - 构建核心测试、React 设置页、`hufu-server.exe` 与 `hufu-render-helper.exe`。
   - 校验设置页已嵌入服务端，并上传含 `SHA256SUMS.txt` 的构建产物。
   - 该工作流**不构建 TSF DLL，也不生成完整安装包**。

## 最近验证结果

以下命令已在本机执行通过：

```powershell
cd engine
cargo test --locked --target x86_64-pc-windows-msvc -p hufu-config -p hufu-dict -p hufu-engine

cd ..\settings-app
npm ci
npm run build

cd ..\engine
cargo build --locked --release --target x86_64-pc-windows-msvc -p hufu-server -p hufu-render-helper
```

- 核心测试：206 通过、0 失败（含 doc tests）。
- 设置页：TypeScript 与 Vite 构建通过；重新生成的 `settings-app/dist` 已核对。
- `settings-app/dist` 有意纳入版本控制；修改 React 源码时须同时提交新 dist 与哈希资源。
- 发布构建：MSVC x86_64 通过；本次有 3 个 dead-code 警告，不影响构建。
- 产物检查：服务端包含当前 `settings-app/dist` 的 index、CSS、JS 与图标，并包含文档导航链接标记。
- 浏览器验证：设置页“概览 → 帮助”已显示 `hufu/docs/README.md`，无 console error/warn。
- 行为验证：`tj` 精确候选在前，长码补全候选可选；五笔注释显示完整编码。
- GitHub Actions：Windows 工作流已添加并触发；最新结论以仓库 Actions 页面为准。

## 本机运行态（2026-11-25 检查）

- 安装目录：`C:\Users\kelso\AppData\Local\Programs\HuFu\`
- 正在运行：`hufu-server.exe`
- 已验证安装版 SHA-256：`D4602D8D5AFD7170F4B4C95027FA11ACE3719714FEE121E008D5545305BBDB8B`
- 当前方案：`五笔`
- 当前码表：`码表\五笔\wubi86_jidian.txt`（由用户提供的 Wubi86 极点码表转换，未纳入 Git）
- 当前设置：`prefix_completion=true`、`show_code_comment=true`、`show_pinyin_comment=false`、每页 6 个候选。
- TSF DLL 仍使用本机原有前端；本衍生版没有重新打包或替换 `hufu_tsf.dll`。
- 当前 `engine\target` 产物是后续重构建结果，哈希可能与已安装文件不同；重新部署前必须重新备份、校验并实机验证。

## 平台状态

| 平台 | 状态 |
| --- | --- |
| Windows | 核心引擎、服务端与设置页已在本机验证；TSF 前端沿用上游实现。 |
| Linux | 上游 fcitx5 最小集 CI 保留；本衍生版没有新增 Linux 专属改动。 |
| macOS | 上游 IMK 开发骨架，未完成实机验证，也未发布安装包。 |

## 当前主要未完成事项

- 游戏宿主默认已改为“完全惰性，不挂 sink、不拆链、不子类化”，但游戏内打字与进一步自愈仍见 [`TODO-待修复与改进.md`](TODO-待修复与改进.md)。
- Windows CI 尚未覆盖 TSF DLL 与完整安装包。
- 仓库版本标记仍沿用上游 `version.txt` / `BUILD_TAG`，尚未建立衍生版独立发行号。
- Wubi86 个人码表只存在于本机，是否随仓库分发需先确认许可与用户意愿。

## 文档入口

- 新接手 AI 必读：[`AI_HANDOFF.md`](AI_HANDOFF.md)
- 构建与部署：[`build.md`](build.md)
- 架构：[`architecture.md`](architecture.md)
- 功能：[`features.md`](features.md)
- 资源与许可：[`asset-sources.md`](asset-sources.md)
- 码表格式：[`dictionary-formats.md`](dictionary-formats.md)
- 游戏宿主当前策略：[`game-host.md`](game-host.md)
- 历史材料：旧路径保留在 Git；本地 `docs/archive/` 不纳入提交。
