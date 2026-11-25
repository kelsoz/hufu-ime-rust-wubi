# 虎符 HuFu · Wubi86 衍生版 AI 交接说明

> 写作日期：2026-11-25。本文是当前仓库的接手入口，不是上游官方文档。
> 执行前先运行 `git status`、`git log -5 --oneline` 和 `git remote -v`，以仓库实际状态为准。

## 0. 先读这十条

1. **唯一工作目录**：`F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi`。不要在其他目录维护第二份仓库。
2. **项目身份**：这是基于 `LeafHW/hufu-ime-rust` 的衍生仓库，不是原创项目，也不是上游官方发行版。
3. **上游基线**：`v1.9.9.9`，commit `d438807`。本仓库只增加了五笔短码补全、候选编码注释和 Windows 自动编译等少量改动。
4. **不要自动推送 GitHub**。用户会先在本机验证，明确说“上传”后才提交并推送。
5. **不要做深度或磁盘级扫描**。用户明确要求注意性能；检索限定到具体目录、文件或符号。
6. **本机已安装并验证的是服务端**，不是完整重打包。`hufu_tsf.dll` 仍沿用本机原版，不要随便覆盖。
7. **构建服务端前必须先构建设置页**。`hufu-server` 在编译时嵌入 `settings-app/dist`；该目录有意纳入版本控制，干净 clone 不自带 Node 也能构建服务端。
8. **Windows CI 只构建服务端、渲染助手和设置页**，不构建 TSF DLL，也不生成安装包。
9. **应用内更新器仍指向上游仓库**。不要用它把衍生版覆盖回上游版；需要先改更新源或手动部署。
10. **个人 Wubi86 码表和用户调整只在本机**。没有确认许可前不要提交进 Git，更不要把用户词内容写进公开文档。

## 1. 仓库身份与关系

| 项目 | 值 |
| --- | --- |
| canonical repository | `F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi` |
| 当前分支 | `main` |
| `origin` | `https://github.com/kelsoz/hufu-ime-rust-wubi.git` |
| `upstream` | `https://github.com/LeafHW/hufu-ime-rust.git` |
| 上游基线 | `v1.9.9.9` / `d438807` |
| 许可 | GPL-3.0-or-later，沿用上游 |
| GitHub 关系 | 独立衍生仓库，不显示 `Forked from`；来源由 README 和 `upstream` 远程保留 |

写作时本地与 `origin/main` 同步于 `590377d`。后续先看 Git，不要把这个提交号当成永久事实。

### 本仓库提交脉络

- `1c3b7ee`：增加 Wubi86 前缀补全开关。
- `90e612e`：明确五笔适配版与上游项目关系。
- `6f11005`：增加互斥的拼音与五笔候选注释。
- `590377d`：添加 Windows 自动编译与构建产物上传。

## 2. 本衍生版到底改了什么

### 2.1 前缀补全

**目的**：输入 `tj` 时，精确候选“得”仍排第一，但后面能出现 `tj` 开头的长码字词，解决短码只有精确候选、无法直接看到高频长码候选的问题。

**配置**：

- 字段：`input.prefix_completion`
- 默认值：`false`
- 旧配置没有字段时：保持关闭
- 定义：[`engine/crates/hufu-config/src/lib.rs:211`](../engine/crates/hufu-config/src/lib.rs#L211)
- 默认值：[`engine/crates/hufu-config/src/lib.rs:265`](../engine/crates/hufu-config/src/lib.rs#L265)

**行为约束**：

- 只使用主码表，不混入用户词、个人存储表。
- 只追加编码严格更长且以当前编码开头的条目。
- 精确候选保持原序，补全候选追加在后。
- 补全候选按权重降序、原码表顺序升序。
- 同词保留排名最优的编码。
- 已软删条目不参与。
- 至少保留 1 个名额；数量受每页候选数限制。
- 不作用于反查、快符、通配符、反引号超集、整句锁和超长码接管路径。

**实现位置**：

- 主码表筛选、去重、排序：[`engine/crates/hufu-dict/src/schema.rs:704`](../engine/crates/hufu-dict/src/schema.rs#L704)
- 候选生成与合并：[`engine/crates/hufu-engine/src/lib.rs:5780`](../engine/crates/hufu-engine/src/lib.rs#L5780)
- 回归测试：[`engine/crates/hufu-engine/tests/engine_test.rs:111`](../engine/crates/hufu-engine/tests/engine_test.rs#L111)

**性能注意**：当前 `prefix_completions` 会遍历主码表条目。若实测候选延迟升高，优先增加前缀索引或桶，保持现有排序、去重和软删语义；不要改成扫描文件系统。

### 2.2 候选注释三选一

**UI**：关闭 / 拼音 / 五笔。

**配置**：

- `candidates.show_pinyin_comment`
- `candidates.show_code_comment`
- 两者默认都为 `false`。
- 两者同时为 `true` 时，拼音优先，避免旧配置或外部配置出现双重注释。
- 五笔模式显示当前候选的完整编码，单字和词语都显示。
- 整句模式暂不显示拼音或五笔编码注释。

**实现位置**：

- 配置字段与默认值：[`engine/crates/hufu-config/src/lib.rs:306`](../engine/crates/hufu-config/src/lib.rs#L306)
- 注释生成：[`engine/crates/hufu-engine/src/lib.rs:6159`](../engine/crates/hufu-engine/src/lib.rs#L6159)
- React 三选一状态：[`settings-app/src/pages/Input.tsx:561`](../settings-app/src/pages/Input.tsx#L561)
- React 候选注释控件：[`settings-app/src/pages/Input.tsx:1172`](../settings-app/src/pages/Input.tsx#L1172)
- 旧设置页：`settings-ui/index.html`
- 设置搜索：`settings-app/src/pages/searchIndex.ts`
- 回归测试：[`engine/crates/hufu-engine/tests/engine_test.rs:51`](../engine/crates/hufu-engine/tests/engine_test.rs#L51)

**不要改错的语义**：

- 拼音注释只用于单字；五笔编码注释用于单字和词语。
- 前缀补全候选必须保留完整长码，不能退化成短码。
- 整句抑制注释是当前设计，不是漏实现。

### 2.3 Windows 自动编译

工作流：[`.github/workflows/windows-build.yml`](../.github/workflows/windows-build.yml)

它做这些事：

1. 在 `windows-latest` 上安装 Node 22 与 Rust stable MSVC。
2. 运行 `hufu-config`、`hufu-dict`、`hufu-engine` 测试。
3. 运行 `npm ci` 和 `npm run build`。
4. 构建 `hufu-server` 与 `hufu-render-helper`。
5. 检查服务端二进制包含当前 `settings-app/dist/index.html`。
6. 生成 `SHA256SUMS.txt` 并上传构建产物，保留 14 天。

它**不做**这些事：

- 不构建 `hufu-tsf` DLL。
- 不构建 32 位 TSF。
- 不生成 `HuFuSetup-*.exe` 或 ZIP 安装包。
- 不发布 GitHub Release。
- 不修改用户本机配置。

触发路径只包含 `engine/**`、`settings-app/**` 和该工作流本身；纯文档改动不会触发 Windows 构建。上游 Linux 最小集工作流仍在 [`.github/workflows/ci.yml`](../.github/workflows/ci.yml)。

## 3. 运行时架构速览

完整设计见 [`architecture.md`](architecture.md)。接手时至少掌握以下链路：

```text
Windows TSF DLL
  │ 4-byte little-endian length + UTF-8 JSON
  ▼
\\.\pipe\hufu-ime
  ▼
hufu-server.exe / Host / Engine
  ├─ hufu-config
  ├─ hufu-dict
  ├─ hufu-engine
  ├─ hufu-sentence
  ├─ hufu-rerank
  └─ hufu-server HTTP 127.0.0.1:<port>
       ├─ /next?shell=1  当前 React 设置页
       ├─ /              旧设置页
       └─ /api/*         配置、状态、皮肤、码表、同步等
```

关键事实：

- Windows 管道名：`\\.\pipe\hufu-ime`，见 [`platform/windows/hufu-tsf/src/ipc.rs:50`](../platform/windows/hufu-tsf/src/ipc.rs#L50) 与 [`engine/crates/hufu-server/src/pipe.rs:14`](../engine/crates/hufu-server/src/pipe.rs#L14)。
- server 当前是进程级共享 `Host` 和共享 `Session`，不是每个 TSF 上下文独立会话。
- HTTP 首选端口 `4390`，占用时按固定列表回退；实际端口写入 `%LOCALAPPDATA%\HuFu\http_port.txt`。
- Windows 数据目录默认是 exe 同目录下的 `数据`，见 [`engine/crates/hufu-server/src/main.rs:277`](../engine/crates/hufu-server/src/main.rs#L277)。
- 可用 `HUFU_DATA`、`--data`、`--port` 覆盖，但 server 有单实例互斥，不要随手再起一个实例。
- 设置页配置使用 `rev` 做 CAS；不要手写一个不含最新 `rev` 的旧快照回写。

## 4. 本机当前运行态

以下是 2026-11-25 的实机检查结果，不是仓库通用默认值。

| 项目 | 当前值 |
| --- | --- |
| 安装目录 | `C:\Users\kelso\AppData\Local\Programs\HuFu\` |
| 服务端 | `hufu-server.exe` 正在运行 |
| 已验证安装版 SHA-256 | `D4602D8D5AFD7170F4B4C95027FA11ACE3719714FEE121E008D5545305BBDB8B` |
| 当前方案 | `五笔` |
| 当前主码表 | `码表\五笔\wubi86_jidian.txt` |
| 前缀补全 | 开启 |
| 候选注释 | 五笔 |
| 拼音注释 | 关闭 |
| 每页候选数 | 6 |
| TSF DLL | 本机原版，未由本衍生版重新构建 |
| `version.txt` | `1.9.9.9` |
| 服务端 `BUILD_TAG` | `2026-11-10-v1.9.9.4-r2` |

`wubi86_jidian.txt` 是 HuFu 原生 TSV，首行类似：

```text
#hufu-dict v1 name=wubi86_jidian version=4.3
```

它由用户提供的 `wubi86_jidian.dict.yaml` 转换而来，只存在于本机。`用户调整.txt`、`个人存储表.txt` 等也是本机私有数据，不要复制进仓库或公开文档。

### 已安装版与当前 target 的区别

- 已安装、已实机验证的服务端哈希是 `D4602D...`。
- 写作时 `engine/target/x86_64-pc-windows-msvc/release/hufu-server.exe` 是后续重构建产物，哈希为 `423C032E22079E9E46776B067E092A79A35B609C366011ABC29B3129CE3CD489`，大小 16,166,400 字节。
- 两者大小相同，都包含当前设置页和五笔改动标记，但哈希不同。
- 因此不要假设 target 产物等于已安装文件；重新部署前仍要备份、校验和实机验证。

## 5. 构建与测试

### 5.1 核心引擎与设置页

优先使用 PowerShell 7：

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi'
git status --short --branch

Set-Location engine
cargo test --locked --target x86_64-pc-windows-msvc `
  -p hufu-config -p hufu-dict -p hufu-engine

Set-Location ..\settings-app
npm ci
npm run build

Set-Location ..\engine
cargo build --locked --release --target x86_64-pc-windows-msvc `
  -p hufu-server -p hufu-render-helper
```

最近一次本地验证：

- `hufu-config`：8 个测试通过。
- `hufu-dict`：58 个测试通过。
- `hufu-engine`：115 个测试通过。
- doc tests：25 个通过。
- 合计 206 通过、0 失败。
- TypeScript / Vite 构建通过。
- MSVC release 构建通过；本次有 3 个 dead-code 警告，分别位于 `host.rs` 与 `trainer.rs`。
- 已再次确认 target 服务端内嵌当前 `settings-app/dist` 的 index、CSS、JS 与图标，且 JS 包含 `hufu/docs/README.md` 新链接标记。
- 浏览器验证：通过 Vite 开发页打开“概览 → 帮助”，确认显示 `hufu/docs/README.md`；未发现 console error/warn。

这些数字是 2026-11-25 的快照。修改代码后必须重新运行，不要沿用旧数字。

### 5.2 为什么必须先构建设置页

[`engine/crates/hufu-server/build.rs`](../engine/crates/hufu-server/build.rs#L73) 在编译时读取 `settings-app/dist` 并嵌入 `/next` 页面与静态资源。

- `settings-app/dist` 有意纳入版本控制，干净 clone 可以直接构建服务端，不会因为缺少 Node 构建产物而必然 404。
- 只运行 `cargo build` 可能嵌入旧 dist；修改 React 设置页后，顺序必须是：`npm run build` → `cargo build` → 检查二进制内嵌资源。
- 提交 UI 源码改动时，要同时提交重新生成的 `dist/index.html` 和哈希资源文件；Vite 会删除不再引用的旧哈希资源，这是正常现象。

### 5.3 Windows TSF 前端

生产 TSF 使用 GNU 工具链，与 Windows CI 的 MSVC 服务端构建不是同一条链。

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi\platform\windows'
rustup target list --installed
cargo build --release --target x86_64-pc-windows-gnu -p hufu-tsf
./target/x86_64-pc-windows-gnu/release/hufu-tsf-smoke.exe
```

注意：

- 本机不一定安装了 GNU target；先检查，不要擅自改 CI 的 MSVC 目标。
- `platform/windows/.cargo/config.toml` 的 i686 配置写死了上游开发者路径，见该文件第 20 行起；不要在其他机器直接构建 32 位 DLL。
- 本衍生版没有改 TSF 源码，五笔改动都在服务端/引擎/设置页，因此通常不需要重建 TSF。

### 5.4 Linux / macOS

- Linux 现有最小集 CI 保留，详见 [`build.md`](build.md) 和 `platform/linux/README.md`。
- macOS 仍是上游 IMK 骨架，没有完成实机验证，也没有安装包。
- 不要因为仓库里有前端目录就声称对应平台已经发布。

## 6. 本机部署与回滚

### 6.1 只替换服务端的安全顺序

1. 完成设置页构建、服务端 release 构建和测试。
2. 计算新 exe 的 SHA-256。
3. 检查新 exe 是否嵌入当前 `settings-app/dist/index.html`。
4. 通过设置页 `/api/shutdown` 请求优雅关闭；该请求可能直接断开连接，属于正常现象。
5. 等待进程退出；必要时再结束残留进程。
6. 把旧 `hufu-server.exe` 改名到新的备份目录，不要直接覆盖。
7. 复制新 exe 到安装目录，核对 SHA-256。
8. 启动服务端，检查 `/api/config`、`/api/state`、`/next?shell=1`。
9. 切到五笔方案，输入 `tj`，确认精确候选在前、长码补全可选、五笔注释显示完整编码。
10. 若失败，停止服务端，恢复备份 exe，再启动。

历史备份位于 `C:\Users\kelso\.qoder\tmp\hufu-install-backup-*`，但这是临时目录，不保证长期存在。下次部署创建新备份，不要依赖旧备份。

### 6.2 不要碰的东西

除非任务明确要求，不要替换或修改：

- `hufu_tsf.dll` / `hufu_tsf32.dll`
- `数据/config.json`
- `码表/` 下的用户数据
- `模型/`
- 皮肤缓存和用户词

本衍生版的核心改动不需要动 TSF。覆盖 TSF 会扩大回归面，并可能引入与本机工具链、系统注册和游戏宿主策略相关的问题。

## 7. 排障入口

### 7.1 服务端

- 启动/退出日志：`%LOCALAPPDATA%\HuFu\server-start.log`
- 实际 HTTP 端口：`%LOCALAPPDATA%\HuFu\http_port.txt`
- 当前配置：`GET http://127.0.0.1:<port>/api/config`
- 当前会话：`GET http://127.0.0.1:<port>/api/state`
- 关于/构建信息：`GET http://127.0.0.1:<port>/api/about`
- 当前设置页：`GET http://127.0.0.1:<port>/next?shell=1`

写配置时先读最新 `rev`，保留其他字段，再通过 UI 或 `POST /api/config` 提交。不要直接把旧配置整包回写。

### 7.2 TSF 与游戏宿主

- trace 日志：`%TEMP%\hufu-tsf-trace.log`
- trace 开关：`C:\ProgramData\HuFu\trace\trace-on`
- 游戏默认策略：[`game-host.md`](game-host.md)
- 当前代码入口：[`platform/windows/hufu-tsf/src/tsf.rs:1010`](../platform/windows/hufu-tsf/src/tsf.rs#L1010)
- 硬/软否决：[`platform/windows/hufu-tsf/src/tsf.rs:10772`](../platform/windows/hufu-tsf/src/tsf.rs#L10772)
- 合成游戏形态探针：环境变量 `HUFU_FORCE_GAME=1`

当前底线是：**游戏里虎符可以打不了字，但不能吞键，也不能让用户切不到其他输入法。**

### 7.3 设置页

- 开发服务器：`npm run dev`，默认 `127.0.0.1:5173`。
- 生产页面由 `hufu-server` 提供，不依赖 Node。
- 修改 UI 后必须启动 dev server 并在浏览器里验证；类型检查和单测不能替代真实交互。
- 已验证页面包含“关闭 / 拼音 / 五笔”三选一和前缀补全开关。

## 8. 已知限制与陷阱

### 8.1 应用内更新器仍指向上游

[`engine/crates/hufu-server/src/main.rs:2118`](../engine/crates/hufu-server/src/main.rs#L2118) 附近仍请求 `LeafHW/hufu-ime-rust` 的 release。

风险：用户可能通过应用内更新覆盖衍生版。接手时不要把它当作“检查本 fork 更新”的入口。若要改，必须同时考虑更新、下载、校验和安装路径，并单独测试。

### 8.2 上游发行脚本不可直接照跑

`发行脚本/release.ps1` 等脚本保留上游写作时的 `E:\DSH-KF` 路径、tag/push 和发布流程。不要直接在本仓库执行，否则可能构建错误路径、推送错误仓库或发布到错误位置。

本仓库当前可信的自动构建入口是 `.github/workflows/windows-build.yml`，不是上游完整发行脚本。

### 8.3 版本号无法区分衍生版

- `version.txt` 仍是 `1.9.9.9`。
- `BUILD_TAG` 仍是 `2026-11-10-v1.9.9.4-r2`。
- 不能只看“关于”页判断是否包含五笔改动。

版本规则需要用户确认后再改，不要自行升号。

### 8.4 CI 覆盖不完整

Windows CI 不覆盖 TSF、安装包和真实输入法交互。通过 CI 只代表服务端/设置页构建成功，不代表完整输入法已经发布。

### 8.5 个人码表不在仓库

本机 Wubi86 码表是用户资源。若新环境缺表，需要用户提供原表或确认合法分发来源；不要从用户目录偷偷复制进 Git。

### 8.6 游戏默认策略已变化

旧 TODO 中“LOL 游戏丢键”仍列为待办，但 2026-11 代码已改为默认完全惰性。当前工作不是继续加游戏白名单，而是加固接管生命周期、提供可撤销通道和进程外输入方案。

### 8.7 `sentence-bench` 仍含上游硬编码路径

[`engine/crates/hufu-rerank/src/bin/sentence_bench.rs:182`](../engine/crates/hufu-rerank/src/bin/sentence_bench.rs#L182) 附近的默认语料、默认报告路径，以及 [`engine/crates/hufu-rerank/src/bin/sentence_bench.rs:206`](../engine/crates/hufu-rerank/src/bin/sentence_bench.rs#L206) 的数据目录仍指向 `E:\DSH-KF`。

- 在本机直接裸跑会失败或写到错误位置。
- 必须显式传入语料和 `--out`；但数据目录当前没有命令行参数，仍需先改代码或提供对应目录。
- 若后续要复现历史 Qwen/ngram 基准，先补 `--data-dir` 和仓库相对默认路径，再跑小样本验证；不要直接套用报告中的旧数字。

### 8.8 5 万句语料脚本依赖外部原始数据

[`docs/corpus/prepare_corpus_50k.py`](corpus/prepare_corpus_50k.py) 所在目录已保留准备脚本、`test_sentences_50k.txt` 等成品和统计；但 LCSTS、THUCNews、外卖评论、酒店评论、网购评论等原始数据没有全部纳入仓库。成品可用于基准，重新生成则不一定具备完整输入。

复现前先确认原始数据的来源与许可，并按脚本文件名放入 `docs/corpus/`；不要把来源不明或含个人信息的语料提交进 Git。

## 9. 当前未完成工作

详见 [`TODO-待修复与改进.md`](TODO-待修复与改进.md)：

1. 游戏宿主进一步加固：钩子生命周期、TTL、自愈、自动黑名单、进程外输入条、合成宿主回归。
2. Windows 完整发行 CI：TSF GNU 构建、冒烟、安装包和双产物校验。
3. 衍生版版本与发布标识。
4. Wubi86 码表是否合法分发的决策。
5. `sentence-bench` 可移植性与 5 万句语料的可复现性。

## 10. 文档结构

- [`README.md`](README.md)：文档导航。
- [`AI_HANDOFF.md`](AI_HANDOFF.md)：本文件。
- [`status.md`](status.md)：当前状态快照。
- [`TODO-待修复与改进.md`](TODO-待修复与改进.md)：当前待办。
- [`architecture.md`](architecture.md)：架构。
- [`build.md`](build.md)：构建。
- [`features.md`](features.md)：功能。
- [`dictionary-formats.md`](dictionary-formats.md)：码表格式。
- [`asset-sources.md`](asset-sources.md)：资源与许可。
- [`game-host.md`](game-host.md)：当前游戏宿主策略。
- `docs/archive/`：本地历史归档，不纳入 Git；旧文档原路径仍保留在 Git 中。

本地归档原则：过时文档移动，不删除；dated 报告、旧基准、旧状态、设计稿和发行证据都放在 `docs/archive/` 对应分类。归档中的行号和结论不是当前事实，Git 提交不包含该目录。

## 11. 工作规则

- 先读真实源码和运行态，再修改；不要只依赖旧文档或记忆。
- 修改限定在任务范围，不做无关重构。
- 文件检索限定目录；避免深度扫描整个磁盘。
- 优先可逆操作；替换运行中的 exe 前先备份。
- 不自动提交、不自动推送；用户明确授权上传后再操作 Git。
- 不提交 `.env`、凭据、个人码表、用户词、用户调整日志或临时构建产物。
- 不维护重复仓库副本。
- 用户明确批准某一步后，直接执行，不要反复声明。
- UI 改动必须启动 dev server 并在浏览器验证主路径和边界情况。
- 文档更新后检查相对链接、移动后的路径和命令是否仍可复现。

## 12. 接手后的第一次检查清单

```powershell
Set-Location 'F:\Agent_Workspace\02_projects\hufu-ime-rust-wubi'
git status --short --branch
git log -5 --oneline
git remote -v

Get-CimInstance Win32_Process |
  Where-Object { $_.Name -eq 'hufu-server.exe' } |
  Select-Object ProcessId, ExecutablePath, CommandLine

Get-Content "$env:LOCALAPPDATA\HuFu\http_port.txt"
curl.exe -sS "http://127.0.0.1:$(Get-Content "$env:LOCALAPPDATA\HuFu\http_port.txt")/api/config"
```

然后：

1. 读根 [`README.md`](../README.md) 和本文件。
2. 确认当前方案仍为“五笔”，`prefix_completion` 与注释模式符合用户预期。
3. 若要改引擎，先跑核心测试；若要改设置页，先 `npm run build` 再构建服务端。
4. 若要改 TSF，先确认 GNU 工具链，并准备单独的前端回归；不要假设 Windows CI 已覆盖。
5. 改完后给出 diff、验证结果和是否需要本机替换的说明，等待用户明确授权上传。
