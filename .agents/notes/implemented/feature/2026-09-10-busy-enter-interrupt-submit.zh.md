# Agent Note: 繁忙 Enter 中断并提交投递

Status: implemented

[English](2026-09-10-busy-enter-interrupt-submit.md) | 中文

## 问题

Queue 会保留正在运行的 turn，Steer 会在最近的 step 边界提供消息。这两种行为都不符合这种编辑器工作流：提交替代指令时必须停止当前 activity，并自动启动新 turn。在浏览器中把 prompt 与 cancel 组合成两次调用会引入不安全顺序：cancel-first 可能在 prompt 准入失败前破坏有用工作，prompt-first 则可能让 Agent 把消息领取到本应被替换的 activity 中。

该操作还必须保留更早的排队消息、附件、消息标识和重复提交的 FIFO 顺序。活跃 turn 在准入期间结束时必须安全退化，并且必须经过可继续 child 的鉴权与驻留机制，不能绕过它们。

## 决策

Conversation 设置接受第三个可选繁忙 Enter 行为 `interrupt`；默认值仍是 `queue`。Plain Enter 与运行中的 Send 按钮使用所选行为，按钮标签显示「中断并发送」。选中 Queue 时 Cmd/Ctrl+Enter 使用 Steer，选中 Steer 或 Interrupt 时使用 Queue，因此每种偏好都保留一个非破坏性加速选项。

`SessionPromptRequest.mode` 与可继续 child 的 `SubagentPromptRequest.delivery` 都把 `interrupt` 作为一等值传递。两条 Host 路径都在最后的在线 Agent cutoff 之前完成 content、附件、模型能力、身份与所有权准入。在该 cutoff，运行中的目标先接收 `Agent.cancel({ kind: 'user' }, { keepInbox: true })`，然后同步接收 `Agent.followup(message)`；空闲目标只接收 follow-up。AgentLoop 会把 abort 后提交的唤醒输入归类到 `nextTurn`，因此替代消息会在取消达到静止后启动，不会加入已 abort 的 activity。

普通 Session 投递在重新确认已解析 Agent 仍是注册实例后执行 cutoff。可继续 child 投递则在既有的逐 child lock 内执行，位于最终 parent 鉴权之后、Activation 唤醒之前。重复的 Interrupt 提交按准入顺序追加不同的 `nextTurn` 消息；重复取消调用保留第一个 abort cause，且不丢弃 inbox。

## 考虑过的替代方案

**在浏览器中组合 prompt 与 cancel。** 拒绝，因为两个 Remote 操作之间的 await 边界会让 Agent 领取新消息，或让 prompt 在取消后失败。浏览器也无法拥有 child lock 或最终 Agent 身份检查。

**先取消，prompt 准入失败时恢复草稿。** 拒绝，因为恢复编辑器内容无法恢复已中断的模型流、已领取 inbox 工作、工具副作用，也无法恢复已被其他操作消耗的附件。

**先使用 Steer，再取消。** 拒绝，因为 Steer 有意定位当前 turn 的最近 step，且可能被同步领取。一旦领取，就无法可靠地把它转换为新 turn。

**让 Stop flush 或提升所有待处理消息。** 拒绝，因为 Stop 与提交是两种不同的用户意图。批量提升会改变更早 queue 的调度，且无法把准入失败与发起取消的那条消息关联。

## 结果

用户可以选择与 Codex auto-submit 相同的交互形态：繁忙时按 Enter 会停止活跃工作、保留全部未领取 queue 状态，并无需第二次操作就把已提交消息作为新 FIFO turn 运行。快速多次提交仍是独立 turn。默认行为与既有 Queue、Steer 行为均不变。

Interrupt 是既有 prompt 操作上的投递模式，不是新的取消 endpoint 或 Agent 原语。它的正确性依赖 Agent 取消的同步性、first-cause-wins 语义，以及 abort 后的 `followup()` 被 wake-latch 到 `nextTurn`。cutoff 前的失败保持活跃 turn 不变；剩余的同步失败窗口只是在确切在线 Agent 中插入已完全准入且拥有新标识的消息时发生不变式违反。
