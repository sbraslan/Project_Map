# World Boss System — Deferred Test Notes

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only

Do not execute runtime/fault-injection tests unless the user explicitly changes project phase.

- WB-T01 spawn a boss and leave it alive through the configured battle-window end; observe `wb_Spawned`, boss lifetime, and next scheduled spawn. Covers BUG-WB-001.
- WB-T02 compare state-command receipt on the game process originating spawn/kill versus a P2P peer; also test a single-process topology. Covers BUG-WB-002.
- WB-T03 kill a World Boss with ranking enabled and verify the ranking `ChatPacket` executes on the monster with null descriptor. Covers BUG-WB-003.
- WB-T04 deliver a valid `worldboss update|state|timer|cooldown` command to the shipped client and observe the unloaded temporary `MainBoard` path. Covers BUG-WB-004.
- WB-T05 after isolating BUG-WB-004, inspect state text values from the slice-based timer/cooldown parser. Covers BUG-WB-005.
- WB-T06 deliver `worldboss_ranking update|NormalName|65000` and verify player-name `int()` conversion fails. Covers BUG-WB-006.
- WB-T07 bypass the name conversion in a controlled test build and exercise the subsequent unloaded headers, list-as-function calls, and `MakeText` argument overwrite. Covers BUG-WB-007.
- WB-T08 with a valid nonzero tier, arrange inventory so an early reward fits and a later reward does not; repeat `get_wb_reward` after freeing space and verify partial-item duplication. Covers BUG-WB-008.
- WB-T09 open the official World Boss window and click `reward_button`; verify no handler/command is bound. Covers BUG-WB-009.

- WB-T10 vary World Boss drop count across 0, 1 and 2+ items while keeping the same damage participants; compare ranking construction and verify player/damage iterator misalignment for multiple qualifying players. Covers BUG-WB-010.
- WB-T11 force the scheduled timeout branch, then inspect `m_dwWBVID`, `pkWB`, and the next spawn attempt after `DestroyCharacter`. Covers BUG-WB-011.
- WB-T12 disable `world_boss_event` while a boss is alive, then test both leaving it alive and killing it while disabled; re-enable the event and inspect stale manager state/spawn behavior. Covers BUG-WB-012.

- WB-T13 log in/reconnect after a World Boss has already spawned and before any later transition; open the World Boss window and verify no current-state request or sync packet/command occurs. Covers BUG-WB-013.
