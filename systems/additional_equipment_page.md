# Additional Equipment Page — Static System Map

**Status:** STATIC MAPPING OPEN
**Mode:** detection / mapping only
**Execution:** LOCKED / NOT RUN

## Canonical scope
- persistent selected/unlocked equipment-page state;
- dedicated `ADDITIONAL_EQUIPMENT_1` item window;
- equip/swap/unequip routing between normal and additional equipment;
- combat-stat selection between equipment pages;
- client/server page state parity.

## Deployment proof
- `ENABLE_ADDITIONAL_EQUIPMENT_PAGE` enabled;
- player table persists `page_equipment` and `unlock_page_equipment`;
- character owns additional equipment item array;
- item routing recognizes `ADDITIONAL_EQUIPMENT_1`;
- combat computation has page-aware branches.

## Audit cursor
1. Swap/move/equip window routing and slot-domain validation.
2. Selected-page combat stat application and equip/unequip lifecycle.
3. persistence/client parity and closeout.
