# Additional Equipment Page — Test Plan

## AEP-001
1. Select an Additional Equipment target slot.
2. Swap a valid inventory item into that slot.
3. **Expected after fix:** server uses `ADDITIONAL_EQUIPMENT_1` for validation and placement.
4. Repeat with a normal Equipment target and verify normal routing remains unchanged.
5. Test invalid slot/window combinations and verify rejection.
