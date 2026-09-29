# shop — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Shop / Premium Private Shop runtime tests

### SHOP-T01 — stash cap clipping
Use disposable local characters/shop. Set seller shop gold stash just below GOLD_MAX and/or cheque stash just below CHEQUE_MAX; list item whose price exceeds remaining stash capacity; buy it.

Track buyer debit/item receipt, seller stash before-after, DB shop cache and sale notification.

Bug indicator: sale succeeds but stash increase is less than sale proceeds because it clamps at max.

### SHOP-T02 — premium personal_shop tax
Set personal_shop event flag to a visible nonzero tax. Buy a premium private shop item.

Compare buyer debit, seller stash delta, logged local net dwPrice and listed sold.price.

Bug indicator: seller stash delta equals full listed price instead of post-tax value.

### SHOP-T03 — sale persistence ordering
Dev/fault-injection only: instrument boundaries after item FlushDelayedSave and before/after SHOP_SUBHEADER_GD_BUY and buyer Save. Verify reconnect/restart consistency. Do not run on production data.


## Canonical Shop / Premium Private Shop runtime matrix — 2026-09-28

The older SHOP-T01..T03 entries above are retained as historical notes. From this point forward, **SHP-Txx** identifiers are canonical.

### SHP-T01 — premium sale cross-process acknowledgement gap
Disposable premium-shop sale with fault injection around the GAME -> DB sale notification boundary after buyer debit/item transfer but before confirmed seller-stash processing.
Covers BUG-SHOP-001.

### SHP-T02 — empty Private Shop Search result
Run an isolated/debug search that returns zero rows and observe the response path under ASan/debug.
Covers BUG-SHOP-002.

### SHP-T03 — premium personal_shop tax accounting
Use a nonzero personal_shop tax in a controlled environment and compare buyer debit, listed price and seller-stash delta.
Covers BUG-SHOP-003.

### SHP-T04 — dependent TransferItemAway boundary
Only after creating or loading an isolated shop state with an item in the 80..89 display-position domain, request removal at the vector boundary and observe bounds behavior under ASan/debug.
Covers BUG-SHOP-004 and depends on the size-domain inconsistency represented by BUG-SHOP-008/corrupt state.

### SHP-T05 — oversized initial shop item count
Craft an isolated initial MyShop request above SHOP_HOST_ITEM_MAX and verify that no item transfer is committed before count validation.
Covers BUG-SHOP-005.

### SHP-T06 — duplicate initial display position
Use two distinct source items with the same display_pos in a disposable initial shop setup and inspect runtime ownership/listing consistency.
Covers BUG-SHOP-006.

### SHP-T07 — stash withdraw TOCTOU
Issue a valid withdraw request, change player currency toward the cap before DB success response is applied, then verify stash debit and player credit remain atomic.
Covers BUG-SHOP-007.

### SHP-T08 — display_pos 80..89 server bound
Isolated modified-client add/create request targeting display_pos 80..89 while the server vector domain is 0..79. Run under ASan/debug and verify rejection before any item transfer.
Covers BUG-SHOP-008.

### SHP-T09 — ClosePlayerShop Special Inventory mismatch
Controlled shop close with enough regular inventory capacity but no compatible special-inventory capacity for a listed special item. Verify close is atomic and no earlier item is partially recovered.
Covers BUG-SHOP-009.

### SHP-T10 — shop-cache rebuild atomicity
Disposable DB only. Fault-inject between private_shop_items DELETE and replacement INSERT during shop-cache flush, then restart/reload and compare item rows versus shop metadata.
Covers BUG-SHOP-010.

### SHP-T11 — stash clamp observation
Only in an isolated deliberately desynchronized/corrupt state, test whether stash-clamp behavior can truncate proceeds. This validates OBS-SHOP-001 only and does not restore the older provisional bug classification.

### SHP-T12 — cross-empire 3x price observation
Conditional isolated test only when SHOP_PRICE_3X_TAX / cross-empire price multiplication is enabled. Exercise a high listed price and compare effective charged price for uint32 wrap.
Validates OBS-SHOP-002 only.

### SHP-T13 — MyShopInfoLoad index robustness
Isolated corrupted/stale shop metadata with display position at or above SHOP_HOST_ITEM_MAX; load under ASan/debug and inspect array/index handling.
Validates OBS-SHOP-003 only.

### SHP-T14 — official-client Won withdraw width
Using the official/debug client, request cheque/Won withdraw values around 255/256 and compare Python/C++ argument value with the uint32 packet field received by server.
Validates OBS-SHOP-004 only.

### Readiness consolidation — 2026-09-28
- No Shop runtime test was executed.
- SHP-T01..SHP-T10 cover canonical BUG-SHOP-001..010 one-to-one.
- SHP-T11..SHP-T14 cover OBS-SHOP-001..004 without promoting observations.
- Historical SHOP-T01 is observation-only because the old stash-cap clipping claim was downgraded by the later invariant audit.
- Historical SHOP-T02 aligns with canonical SHP-T03; historical SHOP-T03 aligns with canonical SHP-T01.
- Primary normal-path candidates: SHP-T02 (empty search) and SHP-T03 (premium tax accounting where the feature/tax is enabled).
- SHP-T01/SHP-T07/SHP-T10 are persistence/fault-injection tests.
- SHP-T04/SHP-T05/SHP-T06/SHP-T08/SHP-T13 require isolated crafted/corrupt state.
- Overall first live runtime gate remains DUNGEON-T09.
