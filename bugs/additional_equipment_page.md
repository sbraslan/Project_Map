# Additional Equipment Page — Verified Bugs

## AEP-001 — SwapItem shadow variables prevent destination window from switching to ADDITIONAL_EQUIPMENT_1

**Status:** VERIFIED_STATIC
**Severity:** High

### Evidence
`SwapItem()` first creates outer:
`TItemPos srcCell(INVENTORY, wCell), destCell(EQUIPMENT, wDestCell);`

Inside the Additional Equipment branch it then declares new local variables with the same names:
`TItemPos srcCell(...), destCell(ADDITIONAL_EQUIPMENT_1, ...);`

Those inner declarations shadow the outer variables and are destroyed immediately. Subsequent logic therefore still uses the original outer `destCell(EQUIPMENT,...)`.

### Consequence
Swap/equip routing intended for the additional equipment window is evaluated/committed against the normal equipment destination domain, creating incorrect placement/validation behavior for Additional Equipment swaps.

### Fix boundary
Assign to the existing `srcCell/destCell` variables rather than redeclaring them, or construct them once from a correctly selected window type.

### Regression target
See `tests/additional_equipment_page.md#aep-001`.
