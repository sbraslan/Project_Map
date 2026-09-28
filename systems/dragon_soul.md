# Dragon Soul / Alchemy — Static Map

**Status:** MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Execution:** LOCKED / NOT RUN  
**Opened:** 2026-09-28

## Scope

Primary server roots:
- `game/src/DragonSoul.cpp`
- `game/src/DragonSoul.h`
- `game/src/char_dragonsoul.cpp`
- `game/src/dragon_soul_table.cpp/.h`
- `game/src/questlua_dragonsoul.cpp`
- `game/src/char_item.cpp`
- `game/src/item.cpp`
- `game/src/item_manager.cpp`
- `game/src/input_main.cpp`
- `game/src/cmd_general.cpp`
- `game/src/cmd.cpp`

Client/data roots:
- `Project_Binary/root/uidragonsoul.py`
- `Project_Binary/root/dragon_soul_refine_settings.py`
- `Project_Binary/locale/locale/common/dragon_soul_table.txt`
- `Project_ClientSrc/GameLib/DragonSoulTable.cpp/.h`
- Dragon Soul quest package under `Project_Game/share/locale/europe/quest/d_dragon_soul/`

Enabled build features include:
- `ENABLE_DS_GRADE_MYTH`
- `ENABLE_DS_CHANGE_ATTR`
- `ENABLE_DS_REFINE_WINDOW`
- `ENABLE_DS_SET`

## Main flows

### Deck activation
Client sends:
`/dragon_soul activate <deck>`

Server:
`CHARACTER::DragonSoul_ActivateDeck()`
-> validate deck index / qualification
-> deactivate previous deck
-> add deck affect
-> set `iDragonSoulActiveDeck`
-> activate each equipped stone
-> `DragonSoul_HandleSetBonus()`.

Deactivation:
`DragonSoul_DeactivateAll()`
-> deactivate all DS equipment
-> remove set bonus contribution through `DragonSoul_HandleSetBonus()`
-> active deck = -1
-> remove deck/set affects.

### Equip / pull-out
Normal move/use path for `ITEM_DS` routes equipped stones to:
`DSManager::PullOut()`.

`PullOut()`:
1. resolves a valid destination in Dragon Soul inventory;
2. calls `RemoveFromCharacter()` on the equipped stone;
3. evaluates extraction probability/by-product;
4. either re-adds the stone or destroys it.

### Refine packet path
`HEADER_CG_DRAGON_SOUL_REFINE`
-> `CInputMain`
-> subtype dispatch:
- grade;
- step;
- strength;
- change attr;
- close.

Refine authorization is currently:
`DragonSoul_RefineWindow_CanRefine() == (m_pDragonSoulRefineWindowOpener != nullptr)`.

With `ENABLE_DS_REFINE_WINDOW`, the GM_PLAYER command:
`/refine_open`
-> `do_open_refine_ds`
-> `DragonSoul_RefineWindow_Open(ch)`.

This means the current build intentionally supports a self-opener refine window from the Dragon Soul UI, separate from NPC quest opening.

## Verified findings

### DS set bonus removal
A full active DS set receives extra stat contribution via:
`DragonSoul_HandleSetBonus()`
-> per equipped stone attribute
-> `ApplyPoint(..., +setValue)`.

When one active stone is pulled out:
- `Unequip()` deactivates that stone;
- other active stones keep the deck active;
- the removed slot is cleared;
- `PullOut()` calls `DragonSoul_HandleSetBonus()`.

The removal branch detects the old `NEW_AFFECT_DS_SET` and switches to subtract mode, but the subsequent equipment loop executes:
`if (!pkItem) return;`.

Therefore the first missing slot terminates cleanup before all previously-added set contributions are subtracted.
The set marker affect is then explicitly removed by `PullOut()`.

Result: stale DS set stat contributions can remain on the character after the set is broken.

See `BUG-DS-001`.

### Dragon Heart extraction item lifetime
`ExtractDragonHeart()` consumes the source DS with:
`pItem->SetCount(pItem->GetCount() - 1)`.

For a normal count-1 stone, `CItem::SetCount(0)` removes and destroys the item.

Both zero-charge failure and success branches subsequently call:
`LogManager::ItemLog(ch, pItem, ...)`.

`ItemLog(LPCHARACTER, LPITEM, ...)` dereferences:
- `item->GetID()`;
- `item->GetOriginalVnum()`.

Thus the consumed pointer is used after destruction.

See `BUG-DS-002`.

### Strength refine success orphan
On strength-refine success:
1. result item is created;
2. source `pDragonSoul->RemoveFromCharacter()`;
3. attributes are copied/refreshed;
4. source `SetCount(count - 1)`;
5. result is given to character.

For a normal count-1 source, step 2 has already set the source owner to nullptr.
Therefore `SetCount(0)` does not execute the owner-backed zero-count destruction branch.

The source object remains registered in item-manager ID/VID maps.
Its delayed save sees no owner and sends DB ITEM_DESTROY, but `SaveSingleItem()` does not destroy/remove the in-memory CItem.

Result: each successful strength refine can leave an ownerless, zero-count in-memory item object until later global cleanup/restart.

See `BUG-DS-003`.

## Candidate / not yet promoted

### Step refine equipped-first validation asymmetry
`DoRefineGrade()` rejects equip positions before collection.
`DoRefineStrength()` and `DoChangeAttr()` check `IsEquipped()` for every collected item.

`DoRefineStep()` initializes type/grade/step from the first element of a `std::set<LPITEM>`, then checks `IsEquipped()` only inside `while (++it != end)`.

Therefore the first pointer in set ordering is never checked for equipped state.
A crafted item grid containing an equipped DS can bypass this check only when that equipped pointer is the first sorted element and the material-count constraints also pass.

Because pointer-order dependency affects reproducibility, this remains candidate-only for now.

### Data-boundary candidates
- `GetBasePosition()` uses `row_type > DRAGON_SOUL_GRADE_MAX` rather than `>=`; malformed grade == max can cross the intended row bound.
- `DoChangeAttr()` indexes `needFireCountList[dwDSStep]` without an explicit step bounds check.
These require malformed/inconsistent DS proto/VNUM state and are not yet promoted against the tracked dataset.

## Next static work
1. close grade/step/strength material-count and stack semantics;
2. audit DS table file vs server/client enum dimensions;
3. inspect deck/set bonus reactivation and relog persistence behavior;
4. audit refine-window overlap/warp/window-state interactions;
5. audit extraction tools and source/extractor aliasing;
6. inspect Dragon Soul quest qualification/daily lifecycle.

No runtime test has been executed.
