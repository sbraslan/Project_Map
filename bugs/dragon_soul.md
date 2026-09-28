# Dragon Soul / Alchemy — Bug Registry

**Status:** ACTIVE STATIC MAPPING  
**Execution:** NOT RUN

## BUG-DS-001 — Breaking an active full DS set can leave stale set-bonus stats

**Status:** VERIFIED STATIC

Affected build:
- `ENABLE_DS_SET`

Set-bonus application:
`CHARACTER::DragonSoul_HandleSetBonus()`
adds per-stone set values through `ApplyPoint()` while a complete same-grade active deck is present.

Normal stone removal:
`MoveItem/UseItem`
-> `DSManager::PullOut()`
-> `pItem->RemoveFromCharacter()`
-> `CItem::Unequip()`
-> `DeactivateDragonSoul()`.

If other stones remain active, `RefreshDragonSoulState()` keeps the deck active.

After the removed slot is cleared, `PullOut()` calls:
`DragonSoul_HandleSetBonus()`.

Removal mode is selected from the existing `NEW_AFFECT_DS_SET`, but the slot loop does:
`if (!pkItem) return;`.

Consequences:
- cleanup stops at the first now-empty DS slot;
- set bonus values from that slot and later slots are not subtracted;
- `PullOut()` then removes `NEW_AFFECT_DS_SET`, so the marker no longer represents the stale stat contribution.

**Impact:** character points can retain part or all of a Dragon Soul set bonus after the set is broken. Amount depends on the removed slot position.

**Reachability:** ordinary equipped Dragon Soul pull-out / move path.

**Runtime:** deferred normal-path observation. No execution under current phase.

## BUG-DS-002 — Dragon Heart extraction logs a Dragon Soul after SetCount(0) destroys it

**Status:** VERIFIED STATIC

Server:
`DSManager::ExtractDragonHeart()`.

Both result branches consume the source DS before logging:
`pItem->SetCount(pItem->GetCount() - 1)`.

For count=1:
`CItem::SetCount(0)`
-> `RemoveFromCharacter()`
-> `M2_DESTROY_ITEM(this)`.

After this, extraction calls:
`LogManager::ItemLog(ch, pItem, ...)`.

The overload dereferences:
- `item->GetID()`
- `item->GetOriginalVnum()`.

Therefore `pItem` is a dangling pointer at the log call for a normal count-1 DS.

Affected branches:
- zero-charge extraction failure;
- successful Dragon Heart extraction.

**Impact:** use-after-free / core-crash candidate on ordinary Dragon Heart extraction.

**Runtime:** isolated debug/ASan only. Do not reproduce on production.

## BUG-DS-003 — Successful strength refine leaves the consumed source CItem alive in memory

**Status:** VERIFIED STATIC

Server:
`DSManager::DoRefineStrength()` success branch.

Order:
1. create upgraded result;
2. `pDragonSoul->RemoveFromCharacter()`;
3. copy attributes / refresh result;
4. `pDragonSoul->SetCount(pDragonSoul->GetCount() - 1)`;
5. consume refine stone and give result.

For a normal count-1 DS:
- after `RemoveFromCharacter()`, source owner is nullptr;
- `SetCount(0)` therefore skips its owner-backed `M2_DESTROY_ITEM(RemoveFromCharacter())` branch;
- the source remains in `ITEM_MANAGER` ID/VID maps.

The delayed-save path detects owner==nullptr and sends `HEADER_GD_ITEM_DESTROY` to DB, but `SaveSingleItem()` does not call `DestroyItem()` or erase the in-memory object.

**Impact:** one ownerless zero-count CItem can be leaked for each successful DS strength refinement, producing unbounded process-memory/item-registry growth over time.

**Runtime:** normal operation can trigger it, but validation should use controlled observation; no execution now.

## Candidates / observations

- Step refine skips `IsEquipped()` for the first pointer in its `std::set<LPITEM>`; pointer ordering makes reachability nondeterministic, so not promoted yet.
- `GetBasePosition()` row bound uses `>` instead of `>=`; malformed grade state required.
- Change-attr fire-count table has no explicit `dwDSStep` bounds guard; malformed step state required.
- `/refine_open` is GM_PLAYER and opens with the character itself as opener. This appears intentional for `ENABLE_DS_REFINE_WINDOW` and is not classified as a bug by itself.
