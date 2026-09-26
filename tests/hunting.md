# hunting — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Hunting System — initial runtime tests

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
Covers BUG-HUNT-004.

### HUNT-T05 — claim crash boundaries
Fault inject between each reward grant and its quest-flag clear, and before final reward_cached/level updates.
Relog and compare item/gold/exp state with cached flags to identify duplicate/loss windows.
