# Mining / Pickaxe — Runtime Tests

> Documentation only. Execution remains locked.

## MINE-T01 — current pickaxe refine threshold
Using disposable data after runtime unlock:
1. bring a 29101..29109 pickaxe to exactly `socket0 == value2`;
2. give it to NPC 20015 through the normal mining quest;
3. observe `__refine_pick` / return code and item state;
4. separately test a controlled `socket0 > value2` state.

Bug indicator:
- equality reaches quest refine UI but C++ returns failure;
- greater-than state is rejected by quest before calling refine.

Covers `BUG-MINE-001`.

## MINE-T02 — mining death/warp lifecycle
After runtime unlock in isolated/dev conditions:

Death branch:
- begin mining beside a valid vein;
- die without ordinary movement before delayed completion;
- verify whether ore roll/drop and pick practice still resolve.

Warp branch:
- begin mining beside a valid vein;
- trigger a same-character warp before delayed completion;
- verify event survival and whether ore drops at destination coordinates.

Covers `BUG-MINE-002`.

Global first future live gate remains `DUNGEON-T09`; do not run these now.
