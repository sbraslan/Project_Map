# Horse / Mount / Riding — Static Bug Registry

**Status:** STATIC COMPLETE EVIDENCE

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


---

## BUG-HORSE-003 — current mount races 20276..20283 are absent from client mount-level classification

**Class:** client/server data-code mismatch / mounted combat disabled  
**Reachability:** VERIFIED — current packed item proto contains deployed COSTUME_MOUNT items using all eight affected races.

### Current deployment proof
Project_Binary and Project_DumpProto use the same packed item-proto blob:
`24ed504beb93a38aff772b8c24d9b6e1bcd4c2d0`.

Decoded rows:
- 71259 -> 20276
- 71260 -> 20277
- 71261 -> 20278
- 71262 -> 20279
- 71263 -> 20280
- 71264 -> 20281
- 71265 -> 20282
- 71266 -> 20283

Each row is `ITEM_COSTUME / COSTUME_MOUNT` with first apply `APPLY_MOUNT`.

Client `npclist.txt` also contains the corresponding 2021/2022 race entries.

### Broken client invariant
`ENABLE_NO_MOUNT_CHECK` is disabled.

`InstanceBase.cpp::GetMountLevelByVnum()` has no case for 20276..20283, so each falls through to `MOUNT_TYPE_NONE`.

That result is consumed by:
- `SHORSE::CanAttack()`;
- `SHORSE::CanUseSkill()`;
- `CInstanceBase::CanAttackHorseLevel()`;
- `CInstanceBase::CanAttack()`;
- `CPythonPlayer` auto-attack handling.

### Consequence
These current mount items can render through the normal mount path, while the client rejects mounted combat and horse-skill eligibility for their race IDs.

### Deferred validation
Canonical runtime test: `HORSE-T03`.

---

## Closure

Horse / Mount / Riding static bug set:
- BUG-HORSE-001
- BUG-HORSE-002
- BUG-HORSE-003

Remaining observations intentionally unpromoted:
- time-limited mount ChangeLook donor lifetime is not transferred, but a tracked normal-player transmutation opener is not proven;
- raw DB horse level lacks a load-time clamp, but no malformed producer is tracked;
- login horse-level normalization resets underlying HP/stamina/drop-time, currently masked by infinite horse health/stamina;
- Additional Equipment UNIQUE ride remove helper is suspicious, but no tracked live remove-side producer was established.
