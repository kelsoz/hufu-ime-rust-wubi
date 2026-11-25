# 游戏宿主支持（当前策略）

> 当前默认策略定稿于 2026-11。旧版子类化流派仅保留在本地 `docs/archive/research/game-host-subclass-2026-10-06.md`，不纳入 Git；长期策略稿见 [`game-input-compat-strategy.md`](game-input-compat-strategy.md)。

## 核心底线

**游戏里虎符打不了字可以接受，但必须能切到其他输入法，不能把键盘搞死。**

因此默认不再通过“识别游戏 → 拆链 → 子类化”来强接管，而是先判断该进程是否具备真实可编辑文档焦点。

## 默认行为

`platform/windows/hufu-tsf/src/tsf.rs` 的 `Activate` 当前行为：

- 命中硬否决特征（全屏弹出窗、进程名单、反作弊模块等）时**完全惰性**：不挂键 sink、不拆链、不子类化。
- 命中软否决特征（游戏输入栈模块且无文本焦点）时先惰性；`OnSetFocus` 出现可编辑焦点后才补挂。
- 补挂前用 `caret_evidence` / `GetSelection` 取证，失败时拒绝接管，并再次检查 `game_hard_veto`。
- 即使不是游戏，`Activate` 时拿不到文档焦点也不挂 sink，等真正的文档焦点再补挂。
- trace 中 `class=hard` / `class=soft` 可用于日志归类。

相关源码入口：

- 激活与惰性判定：`platform/windows/hufu-tsf/src/tsf.rs:1010`
- 硬/软否决：`platform/windows/hufu-tsf/src/tsf.rs:10772`
- UI-less 候选刷新：`platform/windows/hufu-tsf/src/ipc.rs:950`

## UI-less 候选

宿主在 `BeginUIElement` 的 `pbShow` 返回 FALSE、声明自己绘制候选时：

- `candidate_ui::host_draws()` 识别该状态。
- `candwin2::show` 收掉 HuFu 自绘候选窗，避免双窗。
- `state_request()` 轮询响应仍会无条件 publish 最新候选，保证后台重排或云候选到达后约 140ms 内刷新。
- cs2 / dota2 / deltaforce 等“假自绘”游戏继续由 HuFu 自绘，因为它们只替内置白名单输入法名画候选。

## 显式接管与豁免

- `league of legends.exe` 仍保留旧适配路径。
- 其他游戏如需游戏内打字，可创建 `C:\ProgramData\HuFu\game-takeover`，恢复高风险子类化流派，风险由用户承担。
- `C:\ProgramData\HuFu\not-game.txt`：每行一个 exe 名，`#` 为注释；命中的进程永不按游戏处理。
- `C:\ProgramData\HuFu\caps-no-hook`：Caps 虚拟化急停旗标。

## 验证

无需真实游戏即可验证硬否决：

1. 给目标宿主进程设置 `HUFU_FORCE_GAME=1`。
2. 创建 `C:\ProgramData\HuFu\trace\trace-on`。
3. 观察 `%TEMP%\hufu-tsf-trace.log`。
4. 预期出现 `activate: 硬否决(class=hard) → 完全惰性`，窗口内字母直接进入控件。

真实游戏还需验证：切到微软拼音可用、退出游戏后无需重启、管理员权限进程不出现全局死键。

## 安全红线

- 不在无 TSF 文档的进程内挂前景键 sink。
- 不依赖 `UnadviseKeyEventSink` 撤销已经武装的 CUAS 吞键管线。
- 不在游戏进程内创建可见窗口。
- 不通过 `ImmAssociateContextEx(hwnd, NULL, 0)` 拆链来“兼容”游戏。
- 任何异常、超时或管道不健康路径都必须直通按键。

## 已知限制

- 独占全屏下叠加候选窗物理不可见，需游戏切无边框窗口。
- 显式 `game-takeover` 子类化路径仍有聊天状态、Enter 发送和还原风险，不属于默认安全路径。
- 全局钩子生命周期、TTL 自愈、自动黑名单和进程外输入条仍待进一步加固，见 [`TODO-待修复与改进.md`](TODO-待修复与改进.md)。
