# hunting — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Hunting System — runtime tests

### HUNT-T01 — invalid mission type
Modified test client: while inactive and level-qualified send action 2 with dValue 2, 255 and 0xffffffff.
Run GAME under ASan/debug and inspect table access / quest flags.
Covers BUG-HUNT-001.

### HUNT-T02 — reward action progression skip
Fresh disposable player:
send action 4 repeatedly without selecting/completing a mission.
Verify `hunting_system.level` increments.
Covers BUG-HUNT-002.

### HUNT-T03 — gold cap reward loss
Complete a mission and cache nonzero gold.
Raise player gold so reward credit reaches/exceeds GOLD_MAX, then claim.
Verify gold unchanged and reward_money cleared.
Covers BUG-HUNT-003.

### HUNT-T04 — level 90 terminal boundary
Complete/prepare mission level 90, claim normally, verify stored level becomes 91.
On a character level >=91, open Hunting window under ASan/debug.
Expected server path: `OpenHuntingWindowSelect()` attempts level-91 table access.
Covers BUG-HUNT-004.

### HUNT-T05 — claim crash boundaries
Fault inject between each reward grant and its quest-flag clear, and before final reward_cached/level updates.
Relog and compare item/gold/exp state with cached flags to identify duplicate/loss windows.

### HUNT-T06 — item persisted, quest flags stale
Use a completed mission with a nonzero item reward.

1. Claim reward.
2. Allow/force `ITEM_MANAGER::Update()` or `FlushDelayedSaveItem()` so the granted item reaches DB.
3. Prevent `PC::Save()` / `HEADER_GD_QUEST_SAVE` from committing the cleared Hunting reward flags.
4. Crash/kill the GAME process.
5. Relog the same disposable character.
6. Verify the granted item still exists while Hunting reward flags reload from the pre-claim DB state.
7. Claim again and verify a second item can be produced.

Covers BUG-HUNT-005.

### HUNT-T07 — CreateItem failure safety
In isolated test data only, make one cached Hunting reward reference an unavailable/invalid item proto or inject `CreateItem == nullptr`.
Claim the reward under ASan/debug.
Expected current code: null dereference in inventory/ground handling.
This validates the robustness finding; do not run against production data.

### HUNT-T08 — ground fallback failure
Fill inventory, then fault-inject `AddToGround == false` for a Hunting item reward.
Verify whether reward flags are still cleared and whether the created item survives anywhere.
This determines whether the ignored ground-insertion result is a reachable reward-loss bug.


### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no Hunting runtime test executed.
- HUNT-T01..T04 map to BUG-HUNT-001..004.
- HUNT-T05 and HUNT-T06 both exercise BUG-HUNT-005 at different crash-consistency depths.
- HUNT-T07 and HUNT-T08 remain robustness validations without promoted bug IDs.
- Primary legitimate Hunting live candidate: HUNT-T04.
- Crash/persistence validation remains deferred until runtime execution is explicitly enabled.
- Overall first live runtime gate remains DUNGEON-T09.


### HUNT-T04 preflight — 2026-09-29
Current source reverified: HUNTING_MISSION_COUNT=90, mission/reward tables are [91] (valid indices 0..90), and ReciveHuntingRewards unconditionally increments hunting_system.level. A legitimate level-90 claim stores 91. The follow-up OOB path requires character level >=91; otherwise OpenHuntingWindowMain takes the lower-level zero-data branch. Canonical handoff: `../HUNT_T04_HANDOFF.md`.
