# Move Costume Attr — Test Plan

## MCA-001
1. Submit a valid base/material pair with an unrelated ITEM_MEDIUM subtype.
2. **Expected after fix:** request is rejected and no item/attribute state changes.
3. Repeat with MEDIUM_MOVE_COSTUME_ATTR and, when enabled, MEDIUM_MOVE_ACCE_ATTR.
4. Verify only intended subtypes succeed.
