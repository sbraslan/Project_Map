# Horse / Mount / Riding — Deferred Runtime Tests

**Execution:** LOCKED / NOT RUN

## HORSE-T01 — pc.mount POINT_MOUNT / MountVnum parity

**Owner bug:** `BUG-HORSE-001`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. use the active `horse_ride.quest` rental path or item 71241;
2. record `POINT_MOUNT`, server `GetMountVnum()`, client visible mount VNUM/state and `pc.is_mount()`;
3. confirm whether the affect-backed value becomes non-zero while `MountVnum` remains zero/old;
4. wait for affect expiry and verify cleanup symmetry;
5. relog during the affect and verify persisted affect reconstruction.

Expected static result: `POINT_MOUNT` changes without the corresponding `MountVnum` synchronization.

No Horse/Mount runtime test has been executed.
The global first future live gate remains `DUNGEON-T10`.


## HORSE-T02 — zero-level horse progression reachability

**Owner bug:** `BUG-HORSE-002`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. use a clean/test character with horse level 0;
2. obtain/use the tracked horse exchange flow that grants item 50050;
3. verify whether any normal player interaction can raise horse level to 1;
4. attempt the tracked summon/training menus and record their grade/level gates;
5. verify that no normal-player command or item-use path changes horse level unless an external/untracked dependency is present.

Expected static result: no tracked normal-player progression from horse level 0.

## Cross-system note
Configured mount-summon Achievement tasks are already covered by `BUG-ACH-006`; no duplicate Horse test ID is created for that defect.


## HORSE-T03 — newer mount race combat classification

**Owner bug:** `BUG-HORSE-003`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. use one current mount from 71259..71266;
2. confirm rendered race 20276..20283;
3. attempt normal mounted auto-attack and manual attack;
4. attempt horse/mount skill use where the character otherwise qualifies;
5. compare against a known classified modern mount from the existing supported set.

Expected static result:
- affected race renders as a mount;
- client mount-level lookup returns NONE;
- mounted combat/skill eligibility is rejected client-side.

No Horse/Mount runtime test has been executed.
