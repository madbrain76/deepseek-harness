# Agent Note: Busy Enter interrupt-and-submit delivery

Status: implemented

English | [中文](2026-09-10-busy-enter-interrupt-submit.zh.md)

## Problem

Queue preserves a running turn and Steer offers a message at the nearest step boundary. Neither behavior matches an editor workflow where submitting a replacement instruction must stop the current activity and start a fresh turn automatically. Composing prompt and cancel as separate browser calls creates an unsafe order: cancel-first can destroy useful work before prompt admission succeeds, while prompt-first can let the Agent claim the message into the activity that should be replaced.

The operation must also preserve older queued messages, attachments, message identities, and repeated-submit FIFO order. It must degrade safely when the active turn ends during admission and must work through the continuable-child authorization and residency machinery rather than bypassing it.

## Decision

The conversation setting accepts a third opt-in busy-Enter behavior, `interrupt`; `queue` remains the default. Plain Enter and the running Send button use the selected behavior. The button label identifies Interrupt and send. Cmd/Ctrl+Enter uses Steer when Queue is selected and Queue when Steer or Interrupt is selected, retaining one non-destructive accelerated alternative for every preference.

`SessionPromptRequest.mode` and the continuable-child `SubagentPromptRequest.delivery` carry `interrupt` as a first-class value. Both Host paths finish content, attachment, model-capability, identity, and ownership admission before the final live-Agent cutoff. At that cutoff a running target receives `Agent.cancel({ kind: 'user' }, { keepInbox: true })` followed synchronously by `Agent.followup(message)`. An idle target receives only the follow-up. AgentLoop classifies waking input submitted after abort as `nextTurn`, so the replacement starts after cancellation quiescence rather than joining the aborted activity.

Ordinary Session delivery performs the cutoff after rechecking that the resolved Agent is still the registered instance. Continuable-child delivery performs it inside the existing per-child lock after final parent authorization and before the Activation wake. Repeated Interrupt submissions append distinct `nextTurn` messages in admission order; repeated cancellation calls retain the first abort cause and do not discard the inbox.

## Alternatives considered

**Compose prompt and cancel in the browser.** Rejected because two Remote operations expose an await boundary where the Agent can claim the new message or the prompt can fail after cancellation. The browser also cannot own child locking or final Agent identity checks.

**Cancel first and restore the draft when prompt admission fails.** Rejected because restoring editor content cannot restore an interrupted model stream, claimed inbox work, tool effects, or attachments already consumed by another operation.

**Use Steer and cancel afterward.** Rejected because Steer intentionally targets the current turn's nearest step and may be claimed synchronously. Once claimed, it cannot be converted reliably into a fresh turn.

**Make Stop flush or promote every pending message.** Rejected because Stop and submission are separate user intents. Bulk promotion changes older queue scheduling and cannot associate admission failure with the message that initiated cancellation.

## Consequences

Users can opt into the same interaction shape as Codex auto-submit: Enter while busy stops the active work, retains all unclaimed queue state, and runs the submitted message as a new FIFO turn without a second gesture. Multiple rapid submissions remain separate turns. The default and existing Queue and Steer behavior do not change.

Interrupt is a delivery mode on existing prompt operations, not a new cancellation endpoint or Agent primitive. Its correctness relies on Agent cancellation being synchronous, first-cause-wins, and on `followup()` after abort being wake-latched to `nextTurn`. Failures before the cutoff leave the active turn untouched; the only remaining synchronous failure window is an invariant violation while inserting a fully admitted, freshly identified message into the exact live Agent.
