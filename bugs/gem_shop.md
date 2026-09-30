# Gem Shop — Verified Bugs

## GEM-001 — Empty configured row causes invalid random selection / vector access

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Gem Shop table generation

### Evidence
- `CShopManager::GemShopGetRandomId(dwRow)` builds a vector of table ids whose `dwRow` equals the requested row.
- There is no guard for an empty result.
- It immediately executes `number(0, dwItemId.size() - 1)`.
- It then returns `dwItemId[randomNumber]`.
- `OpenGemShopFirstTime()` and refresh paths request rows `1..GEM_SHOP_HOST_ITEM_MAX_NUM`.

### Reachable consequence
If any required Gem Shop row has no valid configured item (including because all entries for that row were rejected during initialization due to invalid item vnums), opening or refreshing Gem Shop reaches random selection on an empty vector and then invalid vector indexing. This is a server crash/undefined-behavior configuration path.

### Fix boundary
Validate required row coverage during `InitializeGemShop()` and fail initialization, or make `GemShopGetRandomId()` return an explicit invalid sentinel when no candidates exist and have every caller handle it before storing/using the id.

### Regression target
See `tests/gem_shop.md#gem-001`.


## GEM-002 — Gem Shop leaks a created item when inventory is full

**Status:** VERIFIED_STATIC  
**Severity:** Medium  
**Affected:** ServerSRC / Gem Shop BUY

### Evidence
- `GemShopBuy()` calls `ITEM_MANAGER::CreateItem(dwVnum, bCount)` before checking for an empty destination slot.
- A newly created item is assigned an ID/VID and inserted into ITEM_MANAGER tracking maps.
- If `GetEmptyDragonSoulInventory()` / `GetEmptyInventory()` returns < 0, `GemShopBuy()` only sends the inventory-full message and returns.
- There is no `M2_DESTROY_ITEM(item)` on that branch.

### Reachable consequence
Repeated BUY attempts while inventory is full can create ownerless tracked item objects that are never placed into inventory and never explicitly destroyed on this path. This leaks server-side item objects/IDs and can accumulate unnecessary ITEM_MANAGER state over time.

### Fix boundary
Either determine destination space before creating the item, or explicitly destroy the freshly created item before returning on placement failure.

### Regression target
See `tests/gem_shop.md#gem-002`.


## GEM-003 — BUY persistence ordering can preserve the purchased item while reverting Gem debit / offer consumption after a crash

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Gem Shop BUY persistence

### Evidence
- `GemShopBuy()` debits `POINT_GEM` and marks `m_gemItems[bPos].bSlotStatus = 2` only in CHARACTER runtime state.
- Gem balance, `gem_items`, and `gem_next_refresh` are persisted through the normal player/CHARACTER save path.
- The purchased item is added to the character and immediately persisted with `ITEM_MANAGER::FlushDelayedSave(item)`.
- No atomic DB transaction or immediate CHARACTER save couples the item save with the Gem debit and consumed-offer state.

### Reachable consequence
If the game process fails after the purchased item is flushed but before the character state is saved, the item can remain persisted while the previous Gem balance and previous offer-slot state are restored from DB on next login. The same offer/currency can therefore become usable again despite the purchased item already existing.

### Fix boundary
Persist the currency debit and Gem Shop state together with the granted item in one durable transaction/acknowledged commit boundary, or order the operation so recovery cannot retain the item without the corresponding debit/offer consumption.

### Regression target
See `tests/gem_shop.md#gem-003`.

## GEM-004 — REFRESH / ADD consumables can be lost if the process fails before Gem Shop state is saved

**Status:** VERIFIED_STATIC  
**Severity:** Medium  
**Affected:** ServerSRC / Gem Shop REFRESH + slot unlock persistence

### Evidence
- `RefreshGemShopWithItem()` removes `GEM_REFRESH_ITEM_VNUM` before regenerating offers and updating `gem_next_refresh`.
- `GemShopAdd()` removes `GEM_UNLOCK_ITEM_VNUM` before setting `bSlotStatus/bSlotUnlocked`.
- Item count mutations use normal item persistence, while the resulting Gem Shop state is stored in CHARACTER player data and saved separately/later.
- Neither path forces an atomic player-state save after consuming the item.

### Reachable consequence
A process failure after the consumable item mutation is persisted but before the Gem Shop player state is saved can consume the refresh/unlock item while reverting the refreshed offers, refresh timer, or unlocked slot on next login.

### Fix boundary
Commit consumable removal and Gem Shop state mutation under one durable transaction/recovery boundary, or add an acknowledged persistence sequence that can safely resume/reconcile after failure.

### Regression target
See `tests/gem_shop.md#gem-004`.
