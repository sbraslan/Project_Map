# biolog — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Biolog System — static-complete runtime matrix

### BIO-T01 — completion bridge
Use disposable character/data and finish a C++ Biolog mission until collected == required.
Click the completion/sub-mission button.

Check:
- emitted `biolog_manager.*` quest flag,
- event-quest request result,
- mission advance,
- reward,
- UI refresh.

Expected current repo result: no registered `biolog_manager` quest handles the request.
Covers BUG-BIO-001.

### BIO-T02 — TIMER fragmented packet
Debug/ASan server:
send only the 2-byte Biolog TIMER base packet, withholding the bool byte.
Also test a deliberately fragmented official TIMER send.

Observe handler read/packet accounting and connection state.
Covers BUG-BIO-002.

### BIO-T03 — reminder initialization
Persist `biolog_cooldown_reminder=1`, repeatedly login fresh characters/processes and instrument `m_BiologReminderEventState` before `SetBiologCooldownReminder`.
Covers BUG-BIO-003.

### BIO-T04 — invalid reward apply types
Isolated DB only:
test reward rows with apply_type 0 + nonzero value and an apply type beyond valid aApplyInfo range.
Invoke reward helper under ASan.
Covers BUG-BIO-004.

### BIO-T05 — submission crash consistency
Submit an item successfully, then fault-inject GAME crashes at boundaries:
- after item SetCount/Destroy persistence,
- after collected count mutation,
- before/after normal CHARACTER save.

Relog and compare item count + biolog_collected/cooldown.
Covers BUG-BIO-005.

### BIO-T06 — reward helper replay
Temporary trusted dev quest:
call `pc.biolog_set_reward_bonus()` twice for same mission.
Inspect persisted AFFECT_COLLECT rows/character points after relog.
Covers BUG-BIO-006.

### BIO-T07 — completed-count Researcher Elixir
Reach collected == required, activate Researcher Elixir, then use modified client to send CG SEND despite hidden button.
Verify affect disappears without submission.
Covers BUG-BIO-007.

### BIO-T08 — actual DB proto audit
Dump/read live `biolog_missions` and `biolog_rewards` rows.
Validate:
- mission keys/contiguity and final mission sentinel assumption,
- chance 0..100,
- required counts/items/sub-items,
- cooldown ranges,
- reward item counts,
- every apply_type range and value,
- matching mission/reward key sets.

This is required because current Git repositories do not contain these table rows.

### BIO-T09 — empty proto boot
On isolated DB, temporarily empty one biolog proto table and boot DB/GAME under ASan/debug.
Check `vector[0]` zero-size encode behavior.
Covers OBS-BIO-003.


### BIO-T10 — client reward-bonus getter bounds
Isolated/debug client only. Call the Biolog reward bonus getter with valid indices and deliberately out-of-range indices while running under ASan/debug instrumentation.

Purpose: validate OBS-BIO-001 without promoting it to a verified bug before runtime evidence exists.

### BIO-T11 — sequence compatibility regression
Only if ENABLE_SEQUENCE_SYSTEM is enabled in an isolated future build, exercise the Biolog action send/receive path and verify packet framing remains synchronized.

Purpose: validate the dormant compatibility risk in OBS-BIO-002. This test stays conditional and must not be run in the current build merely to force the feature on.


### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no Biolog runtime test executed.
- BIO-T01..BIO-T07 cover BUG-BIO-001..007 one-to-one.
- BIO-T08 is the external/live DB proto audit required because biolog proto rows are not present in the tracked repositories.
- BIO-T09 covers OBS-BIO-003.
- BIO-T10 added for OBS-BIO-001; observation status is unchanged.
- BIO-T11 added as a conditional future regression for OBS-BIO-002; observation status is unchanged.
- Primary legitimate normal-path candidate: BIO-T01.
- BIO-T08 remains dependency-gated on access to actual biolog DB rows.
- Overall first live runtime gate remains DUNGEON-T09.
