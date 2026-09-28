# Acce / Sash — Deferred Runtime Tests

**Status:** DOCUMENTATION DRAFT / NOT RUN  
**Execution:** LOCKED  
**Global first future live gate:** DUNGEON-T10 remains unchanged.

These tests exist only to preserve ownership of the current static findings. They are not authorized for execution.

## ACCE-T01 — Final request without an opened Acce window
Owner: BUG-ACCE-001.

Future isolated modified-client check:
- do not open Acce through quest/UI;
- submit a syntactically valid final Acce request using disposable items;
- observe whether server transaction logic is reached.

Static prediction: server does not require Acce open-state/mode.

Safety: Stage B modified-client / state validation.

## ACCE-T02 — Wrong costume subtype as sash input
Owner: BUG-ACCE-002.

Requires controlled disposable proto/data satisfying the later Acce apply preconditions.

Static prediction: ITEM_COSTUME with non-ACCE subtype is not rejected by the server's current AND predicate.

Safety: Stage B controlled-data / modified-client.

## ACCE-T03 — Non-body armor absorption material
Owner: BUG-ACCE-003.

Use a disposable non-body ITEM_ARMOR item through an isolated modified-client request.

Static prediction: server accepts the item because it tests ITEM_ARMOR type without enforcing ARMOR_BODY subtype.

Safety: Stage B destructive material-consumption test.

## ACCE-T04 — Same-slot combine alias
Owner: BUG-ACCE-004.

Only under disposable database state plus ASan/debug instrumentation:
- submit one inventory cell as both primary and material;
- capture failure and success branches separately.

Static prediction:
- failure can consume the primary item;
- success reaches second removal through a stale alias.

Safety: Stage C crash/lifetime/sanitizer. Do not run on production.

No test executed.
