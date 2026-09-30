# Snake Lair / Queen Nethis — Deferred Runtime Tests

**Status:** STATIC MAPPING OPEN / 1 TEST DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.


## SNK-T01 — construct Snake instance under memory/UB instrumentation
Create repeated Snake private instances under an iterator/memory-debug build and preferably ASan/UBSan, with allocator patterns that do not zero freshly allocated object storage.

Expected signature in the current code: construction reads the three event pointer members before initialization; a non-zero indeterminate value can reach `event_cancel`. A corrected constructor must initialize all event pointers before any read/cancel operation.

Covers `BUG-SNK-001`.
