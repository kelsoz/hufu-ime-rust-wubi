# 虎符 HuFu 文档导航

## 新接手

- [AI 交接说明](AI_HANDOFF.md)：仓库身份、当前改动、运行态、构建、部署、排障和工作规则。
- [当前状态](status.md)：当前分支、已验证能力、本机安装状态和平台进度。
- [当前待办](TODO-待修复与改进.md)：只列尚未完成的工作。

## 当前参考

- [构建指南](build.md)：引擎、设置页、Windows TSF、Linux 与 macOS 的构建入口。
- [架构设计](architecture.md)：共享 Rust 核心、平台前端、IPC、HTTP、码表、整句、皮肤和设置子系统。
- [功能总览](features.md)：当前功能口径。
- [码表格式](dictionary-formats.md)：HuFu、Rime、多多、QQ 五笔、虎整句及伴随文件格式。
- [资源来源与许可](asset-sources.md)：`assets/` 的来源、模型下载和台账校验。
- [游戏宿主支持](game-host.md)：当前默认惰性策略、显式接管和已知限制。
- `corpus/`：整句基准语料与准备脚本；100 句历史语料保留在 `corpus/test_sentences_100.txt`。

## 历史材料

`docs/archive/` 仅保留在本地工作区，不纳入 Git。旧文档仍按 Git 中的原路径保留；需要追溯时先查 Git 历史。
