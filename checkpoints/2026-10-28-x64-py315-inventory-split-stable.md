# Checkpoint — x64 / Python 3.15 / Inventory Split Stable

Date: 2026-10-28

Checkpoint ref name:

`checkpoint/x64-py315-inventory-split-stable-2026-10-28`

## Purpose

This checkpoint freezes the verified state immediately before the next complex change set.

The previous gameplay-stable checkpoint remains untouched. This checkpoint adds the later Python 3.15 ConnectingDialog timing fix and the verified split inventory/equipment runtime state.

## Repository snapshot

| Repository | Branch / source ref | Commit |
|---|---|---|
| Project_ClientSrc | `migration/client-x64` | `75ad11e25e1e82f8f6bb918537d64e6525df20c3` |
| Project_ServerSRC | `modernization/x64-wiretime-compat` | `1742dbdc073324bb7154128218954b9555d6551f` |
| Project_Binary | `main` | `7f945a414f02e8a17adcccacab6316b6110bc497` |
| Project_Game | `main` | `24bfe96db414abd070f77f215bece36aaa52e1d0` |
| Project_DumpProto | `main` | `1066f4620af451c5f728260eb5f8a086672e6384` |

## Binary change included in this checkpoint

Commit:

`7f945a414f02e8a17adcccacab6316b6110bc497`

Message:

`fix: migrate ConnectingDialog timing to Python 3.15`

File:

`root/intrologin.py`

Verified migration:

- `time.clock()` count before: 3
- `time.clock()` count after: 0
- `time.perf_counter()` count after: 3
- Python 3.15 syntax validation: PASS
- Runtime copy and Project_Binary copy were tested with the same logical change.

The three converted call sites are the ConnectingDialog timer start and the two OnUpdate timer reads.

## Inventory / equipment runtime state

Verified desired mode:

`SHOW_EQUIPMENT_VIEW 1`

Expected behavior:

- Inventory window and equipment window are shown as separate/split sections.
- `InventoryWindowSlots.py` path is used for the split view.
- EquipmentDialog is constructed as the separate equipment window.
- Previous half-rendered inventory failure is no longer reproduced.

Root cause of the earlier half-rendered inventory state:

1. Equipment View setting changed.
2. Restart dialog attempted to run.
3. Python 3.15 no longer provides `time.clock()`.
4. Restart failed while the setting had already changed.
5. Existing EquipmentDialog object remained alive under the new setting.
6. Refresh code then reached the wrong object shape and raised the historical `EquipmentDialog ... SetItemSlot` AttributeError.

The timer migration and a clean client restart removed this inconsistent state.

## Runtime validation

Latest verified client session:

- Architecture: x64
- Python: 3.15.1 x64
- Client build shown by central syserr: `Oct 28 2026 00:59:48`
- Client startup: PASS
- Login / game entry: PASS
- Inventory open/close: PASS
- Split inventory/equipment view: PASS
- `time.clock` runtime error: NOT PRESENT
- `EquipmentDialog / SetItemSlot` traceback: NOT PRESENT
- Python Traceback: NOT PRESENT
- AttributeError: NOT PRESENT
- RuntimeError/SystemError: NOT PRESENT
- ACCESS_VIOLATION: NOT PRESENT
- FATAL CLIENT CRASH: NOT PRESENT
- `PYTHON STARTUP: RunMainScript returned true`: PRESENT

## Runtime cleanup state

The x64 runtime cleanup remains valid.

Critical local runtime dependencies retained:

- `metin2release.exe`
- `python315.dll`
- `granny2_x64.dll`
- `d3dx9_43.dll`

Obsolete legacy/CEF/Miles runtime files were quarantined and gameplay testing passed without them.

## Known open issue — do not confuse with checkpoint regression

Turkish quest/LC text encoding is still unresolved.

Observed examples remain mojibake after `[LC;id]` expansion, including strings equivalent to:

- Biolog request text
- Dragon stone-piece collection text
- Blacksmith dialogue/questions

This is the next known diagnostic area, but it is NOT a regression of the inventory/runtime checkpoint.

## Rollback rule

If the next complex work causes regressions:

1. Return each repository to the checkpoint ref of the same name.
2. Restore runtime `SHOW_EQUIPMENT_VIEW 1`.
3. Ensure the three ConnectingDialog timing calls remain `time.perf_counter()`.
4. Re-test login -> character -> game -> inventory -> equipment -> relog/exit before proceeding.

## Status

`CHECKPOINT_STABLE_X64_PY315_INVENTORY_SPLIT_PASS`
