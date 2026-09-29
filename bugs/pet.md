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

## Open candidates
- legacy Lua `pet.summon` argument/signature mismatch; needs tracked active quest producer;
- pickup target is stored as raw LPITEM; lifetime under concurrent/manual pickup still open;
- missing summon-item VID branch in `CPetActor::Update` returns false before Unsummon; item-destruction callers must be closed before promotion.
