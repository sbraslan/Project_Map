# Move Costume Attr — Verified Bugs

## MCA-001 — Any ITEM_MEDIUM subtype can pass the transfer-medium validation

**Status:** VERIFIED_STATIC
**Severity:** Medium-High

### Evidence
The server rejects the medium only when:
`itemMedium->GetType() != ITEM_MEDIUM && subtype != MEDIUM_MOVE_COSTUME_ATTR ...`

Because the type test is joined with `&&`, any item whose type is `ITEM_MEDIUM` makes the first condition false and bypasses the whole rejection expression even when its subtype is unrelated to costume/acce attribute transfer.

### Consequence
A crafted request can use and consume an unrelated ITEM_MEDIUM item as the transfer catalyst, bypassing the intended required-medium subtype.

### Fix boundary
Require `ITEM_MEDIUM` AND an explicit allowed subtype. Reject when type is wrong OR subtype is not in the allowlist.
