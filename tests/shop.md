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
