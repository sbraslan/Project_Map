# Dragon Soul / Alchemy — Bug Registry

**Status:** STATIC COMPLETE EVIDENCE  
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


## BUG-DS-004 — Server Change Attribute accepts strength-refine materials

**Status:** VERIFIED STATIC

Official client:
`DragonSoulRefineWindow.__CanRefineChangeAttr()`
accepts only:
`ITEM_TYPE_MATERIAL && MATERIAL_DS_CHANGE_ATTR`.

Server:
`DSManager::DoChangeAttr()`
classifies material with `IsDragonSoulRefineMaterial()`.

That helper accepts:
- MATERIAL_DS_REFINE_NORMAL;
- MATERIAL_DS_REFINE_BLESSED;
- MATERIAL_DS_REFINE_HOLLY;
- MATERIAL_DS_CHANGE_ATTR.

No later subtype equality check requires MATERIAL_DS_CHANGE_ATTR.

The server then uses:
- `pMaterial->GetValue(0)` as fee;
- `pMaterial->GetCount()` for the hard-coded 1/3/9/27/81 requirement;
- and consumes that material on success.

**Impact:** modified client can use unintended Dragon Soul strength-refine materials for Myth attribute reroll; exact economic advantage depends on the item_proto VALUE0/count availability of those materials.

**Runtime:** Stage B isolated/modified-client only.

## BUG-DS-005 — RefineStep table validation checks the wrong node

**Status:** VERIFIED STATIC / DORMANT IN CURRENT DATA

`DragonSoulTable::CheckRefineStepTables()` tests:
`m_pRefineStrengthTableNode == nullptr`
instead of:
`m_pRefineStepTableNode == nullptr`.

`GetRefineStepValues()` later directly dereferences:
`m_pRefineStepTableNode->GetGroupRow(...)`.

Current tracked server/client table contains both RefineStepTables and RefineStrengthTables, so the defect is dormant.

With RefineStepTables missing but RefineStrengthTables present, the initial guard passes and the code can dereference a null step-table pointer during startup validation.

The inverse configuration can also falsely report RefineStepTables missing merely because RefineStrengthTables is missing.

**Impact:** malformed/incomplete Dragon Soul data can turn a clean configuration error into startup null-deref behavior or misleading validation.

**Runtime:** configuration-validation only; do not alter production data.

## BUG-DS-006 — Any open Dragon Soul refine window authorizes Change Attribute packets

**Status:** VERIFIED STATIC

The character stores one authorization pointer:
`m_pDragonSoulRefineWindowOpener`.

Both normal refine and Change Attribute openers write that same pointer.
No server-side window mode is retained.

All refine handlers, including `DoChangeAttr()`, authorize only through:
`DragonSoul_RefineWindow_CanRefine()`
which returns true whenever the pointer is non-null.

Current build also exposes GM_PLAYER command:
`/refine_open`
-> `DragonSoul_RefineWindow_Open(ch)`.

That command:
- uses the player itself as opener;
- does not check qualification;
- opens normal refine mode.

A modified client can then send `DS_SUB_HEADER_DO_CHANGE_ATTR`; the server cannot distinguish it from a request originating from a real Change Attribute window.

**Impact:** server-side mode/qualification boundary for Change Attribute can be bypassed. Item/grade/count checks inside DoChangeAttr still apply.

**Runtime:** Stage B modified-client authorization test only.

## BUG-DS-007 — Dragon Soul refine opener can survive warp and keep item handling locked

**Status:** VERIFIED STATIC

`CanHandleItem()` returns false while:
`DragonSoul_RefineWindow_GetOpener() != nullptr`.

But:
- `CanWarp()` does not reject the Dragon Soul refine opener;
- `WarpSet()` does not close/clear it;
- the mapped clear path is `DS_SUB_HEADER_CLOSE -> DragonSoul_RefineWindow_Close()`.

Therefore a warp can occur while the server still considers the refine window open.

After arrival, if the client transition discarded the UI without successfully sending CLOSE, the non-null opener continues to make `CanHandleItem()` reject normal inventory/item operations.

With self-opener `/refine_open`, the pointer itself remains valid across the warp, making this a state-lifetime problem rather than a dangling-pointer requirement.

**Impact:** player can arrive after a warp with server-side item handling stuck behind stale Dragon Soul refine state until that state is explicitly cleared.

**Runtime:** controlled normal-flow warp/state observation; no packet crafting required if a normal warp path is available while refine UI is open.

## Candidate / unpromoted refinements

- `GetBasePosition()` accepts grade == DRAGON_SOUL_GRADE_MAX due `>` rather than `>=`; malformed VNUM/proto required.
- `DoChangeAttr()` indexes a five-element need-count array by VNUM-derived step without explicit bounds validation; malformed VNUM/proto required.
- ChangeAttrStepTables / ChangeStone* groups are present in data but not consumed by the mapped current server/client implementation; treated as legacy/dead data, not a bug by itself.


## BUG-DS-008 — Step refine skips equipped-state validation for the first pointer-sorted Dragon Soul

**Status:** VERIFIED STATIC / ORDER-DEPENDENT REACHABILITY

`DoRefineGrade()` explicitly rejects every packet grid position whose `TItemPos::IsEquipPosition()` is true before resolving the item.

`DoRefineStep()` does not perform that packet-position check.

Instead it:
1. resolves all non-null grid positions;
2. inserts the resulting item pointers into `std::set<LPITEM>`;
3. uses `set_items.begin()` as the reference Dragon Soul;
4. advances the iterator first with `while (++it != set_items.end())`;
5. only inside that loop checks `pItem->IsEquipped()`.

Therefore the first pointer-sorted item is never checked for `IsEquipped()`.

The C2S refine packet carries full `TItemPos` entries, so a modified client can include an equipped Dragon Soul position. If that equipped item becomes `set_items.begin()`, it can pass the missing validation while later items satisfy matching type/grade/step and material-count requirements.

The consumption phase later calls either:
- `RemoveFromCharacter(); M2_DESTROY_ITEM(...)`; or
- `SetCount(...)`.

For an equipped Dragon Soul, `RemoveFromCharacter()` explicitly enters `Unequip()` before removal, so this is not merely a harmless reference to an equipped object: a successful/failed Step refine can consume an equipped Dragon Soul when the pointer ordering places it first.

**Impact:** server-side equipped-item invariant for Step refine is order-dependent; a crafted refine grid can reach destructive refinement of an equipped Dragon Soul.

**Reachability note:** which item is first depends on process pointer ordering, so a single attempt is not deterministic. The missing validation and destructive path are static facts.

**Runtime:** Stage B/C isolated modified-client/state test only; never production.


## BUG-DS-009 — Relog set cleanup wraps active deck -1 into ordinary wear slots

**Status:** VERIFIED STATIC / NORMAL RELOG PATH / CONDITIONAL ON LATE-WEAR CONTENT

Dragon Soul deck/set affects are persisted by the generic affect layer.

Login order:
1. `LoadAffect()` restores `AFFECT_DRAGON_SOUL_DECK_0/1` and `NEW_AFFECT_DS_SET`;
2. `ComputePoints()` runs while `iDragonSoulActiveDeck == -1`;
3. `DragonSoul_Initialize()` resets DS active sockets;
4. the persisted deck affect causes `DragonSoul_ActivateDeck(deck)`;
5. `DragonSoul_ActivateDeck()` first calls `DragonSoul_DeactivateAll()`;
6. `DragonSoul_DeactivateAll()` calls `DragonSoul_HandleSetBonus()` before assigning active deck `-1` again/removing affects.

At this first cleanup the active deck is still the initialization value `-1`.

`DragonSoul_HandleSetBonus()` converts it to `uint8_t`. In the current build:
- `WEAR_MAX_NUM = 33`;
- `DS_SLOT_MAX = 6`;
- `uint8_t(-1) = 255`;
- `33 + 255 * 6` wraps to `27`.

Because the persisted `NEW_AFFECT_DS_SET` exists, the function enters subtraction mode and loops wear slots 27..32. Those are ordinary late equipment slots, not Dragon Soul equipment.

For each non-null item, its normal attributes are passed to `GetDSSetValue()`. That helper derives a DS type from the item's VNUM and does not first require `IsDragonSoul()`.

The loop returns at the first null slot, so the exact effect depends on occupancy and attribute/VNUM matches.

**Impact:** a normal relog with a previously active complete DS set can introduce erroneous negative point deltas from ordinary late equipment before the correct deck/set is reactivated. Any non-zero erroneous delta survives the subsequent normal DS reactivation until a later full point recomputation corrects it.

**Runtime:** controlled normal relog/state comparison only; see `DS-T09`.

## BUG-DS-010 — PullOut logs a count-1 extractor after destroying it

**Status:** VERIFIED STATIC

Server:
`DSManager::PullOut()`.

When an extractor is supplied:
```
iBonus = pExtractor->GetValue(...);
pExtractor->SetCount(pExtractor->GetCount() - 1);
```

For a normal owned count-1 extractor, `SetCount(0)` removes/destroys the item.

After that consumption, both outcome branches still dereference the old pointer while formatting the item log:
```
pExtractor->GetVnum()
```

Affected branches:
- successful pull-out;
- failed pull-out.

The ordinary ITEM_EXTRACT caller reaches this path for `EXTRACT_DRAGON_SOUL` when used on an equipped Dragon Soul.

This is independent of `BUG-DS-002`, which concerns the consumed Dragon Soul source in Dragon Heart extraction.

**Impact:** use-after-free / core-crash candidate on an ordinary Dragon Soul pull-out using a count-1 extractor.

**Runtime:** isolated debug/ASan only; see `DS-T10`. Never reproduce on production.

## BUG-DS-011 — Daily-gift event ID 0 can bypass level and qualification checks

**Status:** VERIFIED STATIC / CONFIGURATION-DEPENDENT

Quest:
`dragon_soul_daily_gift.quest`.

The chat handler first reads:
`event_id = game.get_event_flag("ds_dg_id")`.

Level >= 50 and `ds.is_qualified()` are checked only inside:
```
if pc.getqf("event_id") != event_id then
    ...
end
```

A character that has never participated normally has quest flag `event_id == 0`.

If an administrator/external event controller activates the time window with `ds_dg_st/ds_dg_et` but leaves `ds_dg_id == 0`, that new character also has `0 == 0`. The entire eligibility block is skipped and execution continues directly to the daily-count gift branch.

The tracked repository contains the compiled daily-gift quest object, while no tracked `dragon_soul_daily_gift_mgr.quest` or other manager was found that enforces a non-zero ID before event activation. An external controller may still do so; therefore reachability is configuration-dependent.

**Impact:** during a misconfigured/default-ID active event, a below-level or unqualified never-participated character can reach the configured once-per-day gift.

**Runtime:** isolated quest/event-flag validation only; see `DS-T11`. Do not alter production event flags for this test.

## Closure

Dragon Soul / Alchemy static mapping is complete with canonical verified findings `BUG-DS-001..BUG-DS-011`.

The malformed grade/step data-boundary observations remain candidates because current tracked data does not establish a malformed producer.
