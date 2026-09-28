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
