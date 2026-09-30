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
