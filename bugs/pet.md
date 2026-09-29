# Classic Pet System — Static Bug Registry

**Status:** ACTIVE STATIC MAPPING

## BUG-PET-001 — Bruce auto-pickup range ignores Y-axis distance

**Class:** logic/range validation defect  
**Reachability:** VERIFIED — current item 53233 is PET_PAY Bruce with race 34055.

### Static proof
1. Current packed item proto maps item 53233 to `ITEM_PET / PET_PAY`, VALUE0=34055.
2. Current item description identifies Bruce as the pet that automatically gathers items.
3. `CPetActor::Update()` enables `CheckPetPickup()` for race 34055.
4. `PetPickUpItemStruct::operator()` intends to reject owned ground items outside the configured range.
5. Its distance input is:
   `itemX - playerX, playerY - playerY`.
6. The Y delta is therefore always zero.

### Consequence
The 900-unit pickup eligibility check is not radial/two-dimensional. Items with acceptable X displacement can be selected while much farther away on Y, subject to the broader sectree iteration area.

### Deferred validation
Canonical runtime test: `PET-T01`.

## BUG-PET-002 — cached Bruce pickup target can become a dangling LPITEM

**Class:** lifetime / stale-pointer / potential use-after-free  
**Reachability:** VERIFIED — Bruce stores a selected owned ground item while moving toward it, and the owner can manually pick it up before Bruce arrives.

### Static proof
1. `PetPickUpItemStruct` selects an owned ground item and stores its raw `LPITEM` in the pet actor.
2. `BringItem()` retains that pointer whenever the target is farther than 250 units and moves the pet toward it.
3. The pointer is reused on later pet update ticks without resolving the item again through `ITEM_MANAGER`.
4. The owner can manually pick up the same item while the pet is travelling.
5. Manual pickup destroys some ground-item objects outright:
   - gold/ELK uses `M2_DESTROY_ITEM(item)`;
   - a stackable item that fully merges into an existing stack also uses `M2_DESTROY_ITEM(item)`.
6. The next `BringItem()` dereferences the cached pointer via `GetX()/GetY()`.

### Consequence
Bruce can dereference a freed item object after a concurrent manual pickup, producing a crash/corruption boundary depending on allocator reuse.

### Deferred validation
Canonical runtime test: `PET-T02`.

## BUG-PET-003 — REAL_TIME PET_PAY expiry can leave the pet summoned after its item is destroyed

**Class:** lifecycle / stale actor / forced-removal cleanup defect  
**Reachability:** VERIFIED — current deployed classic PET_PAY families, including Bruce 53233, use `REAL_TIME` limits.

### Static proof
1. Current decoded item proto contains many `ITEM_PET / PET_PAY` rows with `REAL_TIME`; item 53233 is `WEAR_PET`, `REAL_TIME 2592000`, race 34055.
2. `CItem::StartRealTimeExpireEvent()` starts the real-time event for `LIMIT_REAL_TIME`.
3. At expiry, `real_time_expire_event` calls `ITEM_MANAGER::RemoveItem(item, "REAL_TIME_EXPIRE")`; classic PET_PAY is not exempt.
4. Forced lower-level removal unequips/destroys the item without calling `CHARACTER::PetUnsummon()`.
5. `CPetActor::Update()` detects the missing summon-item VID but immediately returns `false` when lookup fails, before `Unsummon()`.
6. `CPetSystem::Update()` does not delete/unsummon that actor on the false return.
7. `petsystem_update_event` ignores the `CPetSystem::Update()` return value and continues scheduling at the normal interval.
8. Owner teardown eventually closes the state: `CHARACTER::Destroy()` destroys the pet system, and actor destruction calls `Unsummon()`.

### Consequence
A classic pet can remain spawned/summoned after its real-time summon item has expired and been destroyed, until a later pet-system/owner cleanup path removes it. The actor/update event can therefore continue in a state whose authoritative summon item no longer exists.

### Deferred validation
Canonical runtime test: `PET-T03`.

## Open candidates
- legacy Lua `pet.summon` argument/signature mismatch; needs tracked active quest producer;
- Achievement `pet.summon_time` is reset only when `Unsummon()` can still resolve the summon item; exact visible accounting consequence after abnormal cleanup still needs closure.
