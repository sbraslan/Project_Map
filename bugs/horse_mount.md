# Horse / Mount / Riding — Static Bug Registry

**Status:** ACTIVE STATIC MAPPING

## BUG-HORSE-001 — `pc.mount()` updates POINT_MOUNT without synchronizing `MountVnum`

**Class:** state desynchronization / active quest functionality  
**Reachability:** VERIFIED — current `horse_ride.quest` calls `pc.mount(20030, 600)` and item 71241 calls `pc.mount(20030, 10)`.

### Static proof
1. `questlua_pc.cpp::pc_mount` removes old mount affects and adds:
   `AFFECT_MOUNT -> POINT_MOUNT -> mount_vnum`.
2. `AddAffect()` applies it through `ComputeAffect()`.
3. `ComputeAffect()` calls `PointChange(POINT_MOUNT, mount_vnum)`.
4. `PointChange(POINT_MOUNT)` updates the point but the historical `MountVnum(val)` call is commented out.
5. `pc_mount` contains no explicit `MountVnum()` call afterward.
6. `pc.is_mount()`, character insert/update packets, riding checks and rendering ownership use `GetMountVnum()`, not merely `GetPoint(POINT_MOUNT)`.

### Consequence
The active quest can create a non-zero mount point while the character's actual mount VNUM remains unchanged. The rental horse path can therefore fail to enter the same riding/render state used by equipped mounts and classic horse riding.

### Ownership
Owned by Horse / Mount / Riding because the live producer is the active horse rental quest and the broken invariant is `POINT_MOUNT <-> MountVnum` synchronization.

### Deferred validation
Canonical runtime test: `HORSE-T01`.

---

## Open candidates

### ChangeLook mount lifetime transfer
`CTransmutation::Accept()` consumes a mount appearance material after storing only its VNUM. The dedicated ChangeLook mount expiry helpers/socket2 lifetime are not wired into that transaction, and costume-mount automatic event restart is not currently mapped.

Promotion is deferred until a deployed time-limited `COSTUME_MOUNT` proto is verified.

### Horse-level progression producer gap
The active h_horse quest package uses horse-level/grade gates but contains no `horse.advance` or `horse.set_level` producer. Keep as a deployment/content candidate until all other active quest producers are excluded.

### Raw persisted horse-level bounds
DB-loaded `THorseInfo::bLevel` is copied without a clamp before horse-stat table use. No tracked malformed producer is currently established.


---

## BUG-HORSE-002 — tracked quest deployment has no normal-player horse-level progression producer

**Class:** missing gameplay/content integration / unreachable progression  
**Reachability:** VERIFIED for the tracked quest deployment

### Static proof
1. Current `quest_list` loads the mapped h_horse package, but no horse level-up/mission quest.
2. Recursive tracked compiled quest state contains no horse level-up/advance state and no `object/50050/use` handler.
3. `horse_exchange_ticket.quest` gives item 50050, but no tracked quest consumes 50050 to change horse level.
4. Deployed horse scripts read `horse.get_level()` / `horse.get_grade()` and gate content on levels 11/20+, but contain no `horse.advance()` or `horse.set_level()` call.
5. Engine Lua exposes those setters, so the missing boundary is content/producer integration rather than missing server capability.
6. `/horse_level` exists only at `GM_HIGH_WIZARD` and is not normal-player progression.

### Consequence
Within the tracked deployment, zero/new horse state has no normal gameplay path to become level 1+ or progress through armed/military horse levels. Grade-gated summon/training/mount content can only work for characters whose horse level was populated externally or historically.

### Scope note
The live DB schema/default values are not versioned in the tracked repositories. If an external untracked service or DB bootstrap pre-populates horse levels, that external dependency should be documented; it does not restore a tracked gameplay progression producer.

### Deferred validation
Canonical runtime test: `HORSE-T02`.
