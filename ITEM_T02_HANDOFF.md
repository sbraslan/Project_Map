# ITEM-T02 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-ITEM-001`  
**Canonical test:** `ITEM-T02`  
**Class:** normal-flow reachable / crash-lifetime evidence preferred

> Handoff only. It does not authorize runtime execution.

## Static basis

Current server destroy flow:

`CHARACTER::RemoveItem(const TItemPos&, uint8_t)`
→ validates item/state
→ `ITEM_MANAGER::RemoveItem(item, "DESTROY")` for ordinary items
→ `ITEM_MANAGER::DestroyItem(item)` for Growth Pet upbringing items
→ manager destruction reaches `M2_DELETE(item)`
→ caller then executes:
`ChatPacket(..., item->GetName())`

For ordinary items, `ITEM_MANAGER::RemoveItem` ends with:
`M2_DESTROY_ITEM(item)`

and `ITEM_MANAGER::DestroyItem` removes the item from maps and ends with:
`M2_DELETE(item)`.

Therefore the pointer held by `CHARACTER::RemoveItem` is invalid before the final `item->GetName()` dereference.

## Reachability

This is reachable through the normal destroy-system path; it does not require a malformed packet to enter the vulnerable sequence.

Adjacent behavior:
- `ITEM-T01 / BUG-ITEM-002` covers destroy-count semantics.
- A crash/UAF from ITEM-T02 may preempt visual confirmation of ITEM-T01.
- Keep both findings separate when classifying runtime evidence.

## Preconditions

1. Development/test game core corresponding to mapped source.
2. `ENABLE_DESTROY_SYSTEM` enabled.
3. Disposable ordinary inventory item.
4. Prefer ASan/UBSan/debug allocator or equivalent lifetime instrumentation.
5. Do not patch source before evidence capture.

## Future live action

When runtime is explicitly unlocked and earlier gates are resolved:

1. Log in with a disposable test character.
2. Place a disposable item in inventory.
3. Destroy it through the ordinary client destroy UI.
4. Capture server log and client-visible result.
5. If instrumentation is available, capture UAF report/backtrace at the final `item->GetName()`.
6. Stop after evidence capture; do not patch in the same step.

## Expected result

Mapped source permits a dereference after object destruction.

Possible manifestations:
- sanitizer reports heap-use-after-free;
- game core crash;
- corrupted/garbled destroyed-item name;
- apparently normal message due to freed memory remaining readable.

The last case does not disprove the bug without lifetime instrumentation.

## Result classification

### REPRODUCED
Sanitizer/debug allocator/backtrace proves the post-delete dereference, or the normal destroy action crashes/produces corruption at that path.

### NOT REPRODUCED
Only use if a mapped-equivalent running build is verified to keep the item alive through the ChatPacket or otherwise contains a source-level lifetime fix.

### INCONCLUSIVE
Destroy succeeds without crash under a non-instrumented allocator and no evidence can establish whether freed memory was dereferenced.

## Evidence template

- Test: `ITEM-T02`
- Bug: `BUG-ITEM-001`
- Result:
- Date/time:
- Client/server build:
- Item VNUM / count:
- Instrumentation:
- Server log:
- Sanitizer/backtrace:
- Client message:
- Deployment drift:
- Evidence reference:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
