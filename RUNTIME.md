# RUNTIME — In-Game / Fault-Injection Validation

**Status:** DEFERRED — static mapping complete; runtime execution still locked
**Started:** 2026-09-26

Static mapping is now complete across the subsystem index. This file is the short cursor for the next phase; do not bulk-load every bug registry into one chat.

## Goal
Future-only test inventory. During the current detection/mapping phase, do not execute these tests and do not modify source/game files. Keep this file only as deferred validation notes.

## Read discipline
For each runtime turn:
1. read `STATE.json`, `CURRENT.md`, `RUNTIME.md`;
2. select one subsystem/test cluster;
3. read only that subsystem's `bugs/<name>.md` and `tests/<name>.md`;
4. fetch only the exact source symbols needed;
5. record result before moving on.

## Initial validation order
### Stage A — normal-path, no crafted packets
Start with defects that should reproduce using current checked-in data/UI:
- Dungeon: T10 UI list creation, T09 ranking SQL, T11 GLOBAL flag parsing, T12 cooldown wrap.
- Ticket: normal-user pagination mismatch (T07 in Ticket test plan).
- Hunting: legitimate level-90 terminal boundary and reward persistence behavior where safe.

### Stage B — isolated ASan/debug or modified client
- Dungeon: index/slot/config overflow tests T01-T08.
- Ticket: foreign ID, boundary, non-NUL packet tests.
- Hunting: invalid type/action and CreateItem/ground fault injection.

### Stage C — crash consistency / persistence
- Hunting item-vs-quest reward commit window.
- other subsystem persistence tests already recorded in their individual test files.

### Sung Mahi Tower — prepared cluster
Detailed deferred plan: `tests/sung_mahi_tower.md`.

Prepared validations:
- SMT-T01 — missing tower quest runtime normal path (BUG-SMT-001)
- SMT-T02 — missing monthly reset Lua library (BUG-SMT-002)
- SMT-T03 — monthly mailbox memcpy / unterminated-string ASan check (BUG-SMT-003)
- SMT-T04 — cross-year same-month rollover simulation (BUG-SMT-004)
- SMT-T05 — dark king 7591 resistance comparison (BUG-SMT-005)
- SMT-T06 — tower-only item proto availability (BUG-SMT-006)

Do not run this cluster until runtime execution is explicitly enabled.

## Current next target
**DUNGEON-T09 live ranking reproduction is the gate.**

Preflight result:
- current server config has 9 dungeon entries;
- client receives/stores them before GC OPEN;
- UI `Initialize()` creates list rows only in the zero-count branch;
- therefore T10 has a deterministic normal-path trigger.

Dependency:
`T10 live repro -> fix/bypass T10 -> T09 -> T11 -> T12`.

T09, T11 and T12 are code-path confirmed but remain **LIVE PENDING** because the broken T10 list prevents normal dungeon selection/UI inspection.

### Exact live action
On the current running client/server:
1. log in normally;
2. click the Dungeon Info icon next to the minimap;
3. capture whether the window has dungeon rows.

Reproduction criterion for BUG-DUNGEON-010:
**window opens, but the dungeon list is empty/missing even though the server config contains 9 dungeons.**

No source/config modification is needed for this first test.



## Runtime-readiness consolidation — Dungeon Info — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

The Dungeon Info cluster is now normalized into two execution classes without changing any source or game data:

- **Normal-path chain:** DUNGEON-T09 -> DUNGEON-T11 -> DUNGEON-T12.
- **Isolated ASan/debug chain:** DUNGEON-T01 through DUNGEON-T08.

Bug/test mapping:
- T01 -> BUG-DUNGEON-001
- T02 -> BUG-DUNGEON-002
- T03 -> BUG-DUNGEON-003
- T04 -> BUG-DUNGEON-004
- T05 -> BUG-DUNGEON-005
- T06 -> BUG-DUNGEON-006
- T07 -> BUG-DUNGEON-007
- T08 -> BUG-DUNGEON-008
- T09 -> BUG-DUNGEON-009
- T10 -> BUG-DUNGEON-010
- T11 -> BUG-DUNGEON-011
- T12 -> BUG-DUNGEON-012

The first future live gate is **DUNGEON-T09**. DUNGEON-T10 was retracted after source revalidation. T11/T12 remain behind T09; T01-T08 remain isolated/debug-only and must not be executed during the current detection-only phase.

**Next documentation cluster:** Ticket runtime readiness.


## Runtime-readiness consolidation — Ticket — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Ticket now has a one-to-one bug/test map:
- TICKET-T01 -> BUG-TICKET-001
- TICKET-T02 -> BUG-TICKET-002
- TICKET-T03 -> BUG-TICKET-003
- TICKET-T04 -> BUG-TICKET-004
- TICKET-T05 -> BUG-TICKET-005
- TICKET-T06 -> BUG-TICKET-006
- TICKET-T07 -> BUG-TICKET-007

Execution classes:
- **Normal-path UI validation:** TICKET-T07. This can be observed with legitimate ticket creation/UI behavior and no crafted packet.
- **Isolated/adversarial validation:** TICKET-T01 through TICKET-T06. These require one or more of: modified client input, foreign ticket IDs, disposable DB data, controlled collision setup, invalid admin mode, ASan/UBSan, or crafted non-NUL packets.

Overall runtime order now starts with DUNGEON-T09; DUNGEON-T10 is retracted. Ticket's first future normal-path test is **TICKET-T07** after the Dungeon normal-path cluster is cleared.

**Next documentation cluster:** Hunting runtime readiness.


## Runtime-readiness consolidation — Hunting — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified bug/test coverage:
- HUNT-T01 -> BUG-HUNT-001
- HUNT-T02 -> BUG-HUNT-002
- HUNT-T03 -> BUG-HUNT-003
- HUNT-T04 -> BUG-HUNT-004
- HUNT-T05 -> BUG-HUNT-005 (broad crash-boundary exploration)
- HUNT-T06 -> BUG-HUNT-005 (direct item-persisted / quest-flags-stale reproduction)

Robustness-only validations:
- HUNT-T07 -> CreateItem nullptr handling; no verified bug ID in the current data snapshot.
- HUNT-T08 -> ignored AddToGround failure; no verified bug ID until runtime/fault-injection reachability is proven.

Execution classes:
- **Legitimate/controlled normal-path:** HUNT-T04 is the primary Hunting live candidate because mission 90 can be completed through the intended progression path. HUNT-T03 is a controlled-state reward-loss check requiring a near-cap gold setup.
- **Modified-client / isolated:** HUNT-T01 and HUNT-T02.
- **Crash consistency / persistence:** HUNT-T05 and HUNT-T06.
- **Fault-injection robustness:** HUNT-T07 and HUNT-T08.

The global first live gate is **DUNGEON-T09**. Hunting does not preempt that order.

**Next documentation cluster:** Battle Pass runtime readiness.


## Runtime-readiness consolidation — Battle Pass — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

The existing BP-T01..BP-T14 matrix has been classified against the verified Battle Pass bug registry.

Summary:
- several bugs have more than one complementary validation;
- one test remains a broad lifecycle/regression check rather than a unique one-to-one mapping;
- BP-T02 is retained as the primary normal-path observational candidate;
- crash-consistency, leak/instrumentation, and controlled-state cases remain deferred;
- no Battle Pass runtime test has been executed.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Achievement runtime readiness.


## Runtime-readiness consolidation — Achievement — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Achievement now has deferred runtime coverage for all verified BUG-ACH-001..008. Existing ACH-T01..T08 covered all except BUG-ACH-002, so ACH-T09 was added for the non-transactional cache-rebuild persistence case.

Execution classes:
- **Normal-path gameplay candidates:** ACH-T03 and ACH-T04.
- **Modified-client / isolated interaction:** ACH-T01.
- **Crash consistency / persistence:** ACH-T02 and ACH-T07.
- **Privileged/trusted force paths:** ACH-T05 and ACH-T06.
- **Config-evolution / sanitizer:** ACH-T08.
- **DB atomicity / disposable-data fault test:** ACH-T09.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Biolog runtime readiness.


## Runtime-readiness consolidation — Biolog — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Biolog verified bug coverage is complete:
- BIO-T01..BIO-T07 -> BUG-BIO-001..BUG-BIO-007.

Additional readiness:
- BIO-T08 -> actual/live biolog DB proto audit; external data dependency.
- BIO-T09 -> OBS-BIO-003 empty-proto boot behavior.
- BIO-T10 -> OBS-BIO-001 client getter bounds.
- BIO-T11 -> OBS-BIO-002 conditional sequence-system compatibility regression.

Execution classes:
- **Normal-path integration candidate:** BIO-T01.
- **Packet/debug:** BIO-T02.
- **Initialization/config validation:** BIO-T03/BIO-T04.
- **Crash consistency:** BIO-T05.
- **Trusted-script replay:** BIO-T06.
- **Modified-client validation:** BIO-T07.
- **External DB dependency:** BIO-T08.
- **Observation-only / isolated:** BIO-T09..BIO-T11.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Inventory / Item / Special Inventory runtime readiness.


## Runtime-readiness consolidation — Inventory / Item / Special Inventory — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified Item bug coverage:
- ITEM-T02 -> BUG-ITEM-001
- ITEM-T01 -> BUG-ITEM-002
- ITEM-T03 -> BUG-ITEM-003
- ITEM-T06 -> BUG-ITEM-004
- ITEM-T09 + ITEM-T10 -> BUG-ITEM-006
- ITEM-T11 -> BUG-ITEM-007
- ITEM-T12 -> BUG-ITEM-008

Observation/regression coverage:
- ITEM-T05 -> OBS-ITEM-001
- ITEM-T07 -> OBS-ITEM-002
- ITEM-T04 -> ground persistence/lifecycle regression
- ITEM-T08 -> Special Inventory type/range regression
- ITEM-T13 -> item_proto dataset audit dependency

No canonical BUG-ITEM-005 exists; the numbering gap is intentionally preserved.

The mixed legacy Guild Storage/Switchbot sections in tests/inventory_items.md are not treated as Inventory ownership. Switchbot will be consolidated from its own canonical bug/test files next.

Primary Inventory normal-path candidate: **ITEM-T01**.
The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Switchbot runtime readiness.


## Runtime-readiness consolidation — Switchbot — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Legacy migration left duplicate `SWITCHBOT-T01..T06` IDs in the test file. They are now explicitly non-canonical.

Canonical deferred IDs:
- SWB-T01 -> BUG-SWITCHBOT-001
- SWB-T02 -> BUG-SWITCHBOT-002
- SWB-T03 -> BUG-SWITCHBOT-003
- SWB-T04 -> BUG-SWITCHBOT-004
- SWB-T05 -> BUG-SWITCHBOT-005
- SWB-T06 -> OBS-SWITCHBOT-001
- SWB-T07 -> OBS-SWITCHBOT-002

Primary legitimate monitored candidates are SWB-T01 and SWB-T03. Modified-client testing is confined to SWB-T04/SWB-T07; observation status remains unchanged for SWB-T06/SWB-T07.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Guild Storage runtime readiness.


## Runtime-readiness consolidation — Guild Storage — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified mapping:
- GS-T10 / GS-T11 / GS-T19 -> BUG-GS-003
- GS-T12 -> BUG-GS-004
- GS-T15 -> BUG-GS-007
- GS-T16 -> BUG-GS-008
- GS-T17 -> BUG-GS-009
- GS-T18 -> BUG-GS-010
- GS-T20 -> BUG-GS-011

Candidate-only mapping:
- GS-T04 / GS-T07 / GS-T08 -> BUG-CANDIDATE-GS-002
- GS-T13 -> BUG-CANDIDATE-GS-005
- GS-T14 -> BUG-CANDIDATE-GS-006
- BUG-CANDIDATE-GS-001 remains an architectural/shared-handler candidate with baseline coverage only.

Legacy candidate GS-003 and GS-004 are superseded by their later verified BUG-GS-003/004 records. Candidate IDs 001/002/005/006 remain candidates and are not promoted by documentation alone.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Exchange / Trade runtime readiness.


## Runtime-readiness consolidation — Exchange / Trade — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Canonical runtime IDs are now EXC-T01..EXC-T10. Legacy EX-Txx and EXCHANGE-Txx entries remain historical only.

Coverage:
- EXC-T01 -> BUG-EXCHANGE-001
- EXC-T02 -> BUG-EXCHANGE-002
- EXC-T03 / EXC-T04 -> BUG-EXCHANGE-003
- EXC-T06 -> currency-atomicity BUG-EXCHANGE-004 + BUG-EXCHANGE-005 specialization
- EXC-T07 -> distinct final-distance finding also historically labeled BUG-EXCHANGE-004
- EXC-T08 -> persistence severity for BUG-EXCHANGE-001/002
- EXC-T05 / EXC-T10 -> observation-only validation
- EXC-T09 -> lifecycle regression/sanity

Important registry condition: BUG-EXCHANGE-004 is duplicated across two different verified findings. The ID collision is documented but not renumbered during the current detection-only phase.

Primary normal-path candidates: **EXC-T01** and **EXC-T02**.
The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Shop / Premium Private Shop runtime readiness.


## Runtime-readiness consolidation — Shop / Premium Private Shop — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Canonical deferred IDs are SHP-T01..SHP-T14.

Verified coverage:
- SHP-T01 -> BUG-SHOP-001
- SHP-T02 -> BUG-SHOP-002
- SHP-T03 -> BUG-SHOP-003
- SHP-T04 -> BUG-SHOP-004
- SHP-T05 -> BUG-SHOP-005
- SHP-T06 -> BUG-SHOP-006
- SHP-T07 -> BUG-SHOP-007
- SHP-T08 -> BUG-SHOP-008
- SHP-T09 -> BUG-SHOP-009
- SHP-T10 -> BUG-SHOP-010

Observation coverage:
- SHP-T11 -> OBS-SHOP-001
- SHP-T12 -> OBS-SHOP-002
- SHP-T13 -> OBS-SHOP-003
- SHP-T14 -> OBS-SHOP-004

Historical provisional SHOP-T01..T03 are retained but are not canonical continuation IDs. In particular, old stash-cap clipping is observation-only after the later static invariant audit.

Primary normal-path candidates: **SHP-T02** and **SHP-T03**.
The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Safebox / Mall runtime readiness.


## Runtime-readiness consolidation — Safebox / Mall — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Canonical deferred IDs are SFB-T01..SFB-T12.

Active-build verified coverage:
- SFB-T01 / SFB-T02 -> BUG-SAFEBOX-003
- SFB-T03 / SFB-T04 / SFB-T05 -> BUG-SAFEBOX-004
- SFB-T06 / SFB-T07 -> BUG-SAFEBOX-005

Observation coverage:
- SFB-T08 -> OBS-SAFEBOX-002
- SFB-T09 -> OBS-SAFEBOX-003
- SFB-T10 -> OBS-SAFEBOX-004

Dormant ENABLE_SAFEBOX_MONEY coverage:
- SFB-T11 -> BUG-SAFEBOX-001
- SFB-T12 -> BUG-SAFEBOX-002

Historical OBS-SAFEBOX-001 is superseded by BUG-SAFEBOX-005. The active build has no clean normal-player verified-bug test; SFB-T09 is an ordinary-flow access-policy observation, not a verified bug.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Mailbox runtime readiness.


## Runtime-readiness consolidation — Mailbox — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified bug coverage:
- MAIL-T01 / MAIL-T02 -> BUG-MAIL-001
- MAIL-T03 / MAIL-T04 -> BUG-MAIL-002
- MAIL-T10 -> BUG-MAIL-003 + BUG-MAIL-004 persistence boundaries
- MAIL-T05 / MAIL-T06 -> BUG-MAIL-005
- MAIL-T07 -> BUG-MAIL-006
- MAIL-T08 / MAIL-T09 -> BUG-MAIL-007
- MAIL-T11 -> BUG-MAIL-008
- MAIL-T12 -> BUG-MAIL-009
- MAIL-T13 -> BUG-MAIL-010
- MAIL-T14 -> BUG-MAIL-011
- MAIL-T15 -> BUG-MAIL-012

Observation coverage:
- MAIL-T16 -> OBS-MAIL-001
- MAIL-T17 -> OBS-MAIL-002
- MAIL-T18 -> OBS-MAIL-003

Primary ordinary-flow candidates: **MAIL-T07** and **MAIL-T11**. Adversarial, crash-consistency and fault-injection cases remain isolated/deferred.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Ranking runtime readiness.


## Runtime-readiness consolidation — Ranking — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- RANK-T01 -> BUG-RANK-001
- RANK-T02 -> BUG-RANK-002
- RANK-T03 -> BUG-RANK-003
- RANK-T04 -> BUG-RANK-004
- RANK-T07 -> BUG-RANK-005
- RANK-T06 -> BUG-RANK-007

**BUG-RANK-006 remains RETRACTED / RESERVED** and is intentionally excluded from the runtime bug-validation matrix.

Conditional/unpromoted:
- RANK-T05 -> dormant generic PARTY ranking integration gap if an active caller appears later.
- Generic SOLO category 2..7 dictionary gap remains documentation-only without an active opener.

Primary normal-path candidates: **RANK-T01, RANK-T02, RANK-T03, RANK-T04**.
RANK-T07 is isolated malformed-server-packet parser validation.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Party runtime readiness.


## Runtime-readiness consolidation — Party — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

A dedicated `tests/party.md` file did not previously exist; the canonical deferred matrix is now PARTY-T01..PARTY-T06.

Verified coverage:
- PARTY-T01 -> BUG-PARTY-001
- PARTY-T02 -> BUG-PARTY-002
- PARTY-T03 -> BUG-PARTY-003
- PARTY-T04 -> BUG-PARTY-004
- PARTY-T05 -> BUG-PARTY-005
- PARTY-T06 -> BUG-PARTY-006

Execution classes:
- **Normal-path:** PARTY-T01, PARTY-T02, PARTY-T05
- **Modified-client/state integrity:** PARTY-T03
- **Malformed-server-packet parser:** PARTY-T04
- **Trusted/dev quest semantic:** PARTY-T06

Party Match remains a separate subsystem and is not folded into the Party matrix.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Party Match runtime readiness.


## Runtime-readiness consolidation — Party Match — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

A dedicated `tests/party_match.md` file did not previously exist; canonical deferred IDs are now PMATCH-T01..PMATCH-T05.

Verified coverage:
- PMATCH-T01 -> BUG-PMATCH-001
- PMATCH-T02 -> BUG-PMATCH-002
- PMATCH-T03 -> BUG-PMATCH-003

Unpromoted robustness:
- PMATCH-T04 -> duplicate SEARCH / HOLD client-state desync
- PMATCH-T05 -> ignored WarpSet failure / non-atomic completion ordering

Primary normal-path candidate: **PMATCH-T03**.
PMATCH-T01 is a controlled multi-core normal-flow architecture test; PMATCH-T02 is isolated exchange/item-lifetime testing.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Dungeon Core runtime readiness.


## Runtime-readiness consolidation — Dungeon Core — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- DCORE-T01 -> BUG-DUNGEON-001
- DCORE-T02 -> BUG-DUNGEON-002
- DCORE-T03 -> BUG-DUNGEON-003
- DCORE-T04 -> BUG-DUNGEON-004

Execution classes:
- **Registered Lua / lifecycle:** DCORE-T01
- **Controlled dungeon-script behavior:** DCORE-T02, DCORE-T04
- **ASan/debug raw-pointer lifetime:** DCORE-T03

Primary Dungeon Core candidates: **DCORE-T01** and **DCORE-T02**.

Dungeon Core is separate from the already-consolidated Dungeon Info subsystem. The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Battle Field runtime readiness.


## Runtime-readiness consolidation — Battle Field — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Unique verified Battle Field coverage:
- BFIELD-T01 -> BUG-BFIELD-001
- BFIELD-T02 -> BUG-BFIELD-002
- BFIELD-T03 -> BUG-BFIELD-003
- BFIELD-T06 -> BUG-BFIELD-006
- BFIELD-T08 -> BUG-BFIELD-008
- BFIELD-T09 -> BUG-BFIELD-009
- BFIELD-T10 -> BUG-BFIELD-010
- BFIELD-T11 -> BUG-BFIELD-011

Registry corrections:
- **BUG-BFIELD-004 is RETRACTED / RESERVED**; the LoadRanking symbol is resolvable through the already-verified declaration/include chain.
- BUG-BFIELD-005 is a historical cross-system duplicate of canonical BUG-RANK-003; use RANK-T03.
- BUG-BFIELD-007 is a historical cross-system duplicate of canonical BUG-RANK-007; use RANK-T06.
- BFIELD-T04/T05/T07 are therefore not active Battle Field validation targets.

Primary ordinary-flow candidates: **BFIELD-T01, T02, T03, T09, T10, T11**.
BFIELD-T06 is privileged/admin routing; BFIELD-T08 is deterministic schedule arithmetic.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** World Lottery runtime readiness.


## Runtime-readiness consolidation — World Lottery — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- WLOT-T01 -> BUG-WLOT-001
- WLOT-T02 -> BUG-WLOT-002
- WLOT-T03 / WLOT-T15 -> BUG-WLOT-003
- WLOT-T04 -> BUG-WLOT-004
- WLOT-T05 -> BUG-WLOT-005
- WLOT-T06 -> BUG-WLOT-006
- WLOT-T07 -> BUG-WLOT-007
- WLOT-T08 -> BUG-WLOT-008
- WLOT-T09 -> BUG-WLOT-009
- WLOT-T10 -> BUG-WLOT-010
- WLOT-T11 -> BUG-WLOT-011
- WLOT-T12 -> BUG-WLOT-012
- WLOT-T13 -> BUG-WLOT-013
- WLOT-T14 -> BUG-WLOT-014

Execution classes:
- **Modified-client/adversarial:** WLOT-T01, T02, T05, T06
- **Numeric/high-value boundary:** WLOT-T03, T04, T15
- **DB/state-shaping:** WLOT-T07, T08, T09, T10
- **Dormant ranking endpoint:** WLOT-T11, T12
- **Crash/fault-injection:** WLOT-T13, T14

Primary ordinary/current-flow candidates: **WLOT-T04** and **WLOT-T10**.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** World Boss runtime readiness.


## Runtime-readiness consolidation — World Boss — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Canonical deferred IDs are WB-T01..WB-T16.

Verified coverage:
- WB-T01 -> BUG-WB-001
- WB-T02 -> BUG-WB-002
- WB-T03 -> BUG-WB-003
- WB-T04 -> BUG-WB-004
- WB-T05 -> BUG-WB-005
- WB-T06 -> BUG-WB-006
- WB-T07 -> BUG-WB-007
- WB-T08 -> BUG-WB-008
- WB-T09 -> BUG-WB-009
- WB-T10 -> BUG-WB-010
- WB-T11 -> BUG-WB-011
- WB-T12 -> BUG-WB-012
- WB-T13 -> BUG-WB-013
- WB-T14 -> BUG-WB-014
- WB-T15 -> BUG-WB-015
- WB-T16 -> BUG-WB-016

The old TEST-WB-MULTICORE-OWNERSHIP, TEST-WB-TITLEBAR-PARENT-STATE and TEST-WB-REWARD-TIER-PROVENANCE headings remain historical aliases for WB-T14/T15/T16.

Dependency note:
- WB-T08 is not a clean normal-path test because BUG-WB-016 leaves normal players at tier 0; it needs a controlled nonzero-tier setup.

Primary ordinary/current-flow candidates: **WB-T09, WB-T13, WB-T15, WB-T16**.
Scheduler/multicore/lifecycle, client-parser and reward-atomicity cases remain controlled/deferred.

The global first live gate is **DUNGEON-T09**.

**Next documentation cluster:** Sung Mahi Tower runtime-readiness final consolidation.


## Runtime-readiness consolidation — Sung Mahi Tower — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- SMT-T01 -> BUG-SMT-001
- SMT-T02 -> BUG-SMT-002
- SMT-T03 -> BUG-SMT-003
- SMT-T04 -> BUG-SMT-004
- SMT-T05 -> BUG-SMT-005
- SMT-T06 -> BUG-SMT-006

Deferred architectural/data questions remain separate and unpromoted:
- nullable pc.mailbox_reward mailbox pointer precondition;
- smhgate_flower 9100–9107 missing server monster folder without tracked producer;
- ranking player_login versus character-name mailbox recipient semantics;
- malformed/out-of-range quest-produced dungeonLevel;
- m_bDungeon_Difficulty versus dungeonLevel synchronization dependency.

Primary current-repository candidate: **SMT-T01**.
SMT-T02/T03/T04 require isolated time/state or ASan/debug execution; SMT-T05 is deterministic data validation; SMT-T06 is deployment/proto drift validation.

The global first live gate is **DUNGEON-T09**.

## Runtime-readiness documentation milestone — COMPLETE — 2026-09-28
All currently indexed static-complete subsystem clusters now have deferred runtime-readiness coverage or an explicitly documented folded ownership path.

Execution remains locked. No runtime, crafted-packet, sanitizer, crash-consistency, or fault-injection test has been run.

The next allowed documentation-only step is a **global readiness integrity audit**:
1. cross-check every STATIC COMPLETE subsystem in INDEX.md against STATE.json runtime_readiness;
2. cross-check verified/retracted/candidate/observation ownership for numbering gaps and aliases;
3. produce one canonical global execution order with DUNGEON-T09 as the first live gate;
4. do not execute anything until the user explicitly changes phase.


## Global runtime-readiness integrity audit — 2026-09-28
**Status:** COMPLETE — PASS WITH NORMALIZATIONS
**Execution:** LOCKED / NOT RUN
**Canonical detail:** `READINESS_AUDIT.md`

Audit result:
- all **21/21 STATIC COMPLETE** subsystem rows in `INDEX.md` have runtime-readiness ownership;
- Guild lifecycle is folded with Guild Storage but its independent `BUG-GUILD-001` now has explicit `GUILD-T01` runtime ownership;
- historical Dungeon Info/Core reuse of `BUG-DUNGEON-001..004` is globally qualified as `DINFO::...` versus `DCORE::...` without renumbering local history;
- `BUG-RANK-006` and `BUG-BFIELD-004` remain retracted/reserved;
- historical `BUG-BFIELD-005/007` remain owned by `BUG-RANK-003/007`;
- candidate/observation/dormant statuses remain non-promoted;
- migrated foreign test blocks are non-canonical outside the prefix owner declared in `tests/INDEX.md`;
- `STATE.json completed_subsystems` is normalized to the 21 STATIC COMPLETE rows.

Canonical future order is recorded in `READINESS_AUDIT.md` and `STATE.json`.
The first future live gate is **DUNGEON-T09**.

**Next documentation-only target:** use the prepared DUNGEON-T09 handoff when runtime is explicitly unlocked.


## DUNGEON-T10 runtime gate handoff — RETRACTED — corrected 2026-09-29
**Execution:** LOCKED / NOT RUN  
**Canonical checklist:** `DUNGEON_T10_HANDOFF.md`

The first future live gate is now fully prepared.

Gate contract:
- normal player login only;
- click the Dungeon Info button next to the minimap;
- no packet crafting, Python injection, GM command, DB/config mutation, sanitizer or source edit;
- capture window state and relevant client/server errors;
- classify only as `REPRODUCED`, `NOT REPRODUCED`, or `INCONCLUSIVE`;
- stop immediately after evidence capture.

Static prediction remains:
- tracked config has 9 dungeon entries;
- client count is nonzero before UI initialization;
- list creation is located in the zero-count branch;
- expected reproduction is an opened Dungeon Info window with no visible dungeon rows.

T09/T11/T12 remain blocked behind a recorded T10 outcome. Applying `FIX-DUNGEON-010` is a separate future phase action and is not authorized by this handoff.

## Patch readiness
First-cluster fix plan is ready at `fixes/dungeon_info.md`.

Prepared without modifying source repos:
- FIX-DUNGEON-010: correct UI list-construction control flow.
- FIX-DUNGEON-009: repair missing whitespace before ranking LEFT JOIN.
- FIX-DUNGEON-011: accept documented numeric `1 = GLOBAL` while retaining literal `GLOBAL` compatibility.
- FIX-DUNGEON-012A: use signed/clamped cooldown arithmetic to eliminate uint32 wrap.
- FIX-DUNGEON-012B: intentionally deferred data-semantics decision; current quest flags represent different timer meanings, so no guessed COOLDOWN values will be written.

Important: Flame/Snow `exit_time` and Dragon `dragon_lair_time` are timestamp-style flags, but they do not all represent the same gameplay window. Arithmetic safety can be fixed generically; displayed cooldown policy must be chosen per dungeon.


## Current phase lock
This file is **not the active execution cursor**.

Current user rule:
- inspect only;
- detect/map only;
- write findings only to `Project_Map`;
- no source, Python, C++, quest, config or game-data changes;
- no runtime/fault-injection execution until the user explicitly changes phase.


## Runtime-readiness consolidation — Costume / Appearance / ChangeLook — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- LOOK-T01 -> BUG-LOOK-001
- LOOK-T02 -> BUG-LOOK-002
- LOOK-T03 -> BUG-LOOK-003
- LOOK-T04 -> BUG-LOOK-004
- LOOK-T05 -> BUG-LOOK-005
- LOOK-T06 -> BUG-LOOK-006
- LOOK-T07 -> BUG-LOOK-007

Execution classes:
- **Ordinary/non-destructive observation:** LOOK-T02 eligibility-only.
- **Controlled normal-flow/state:** LOOK-T03.
- **Modified-client isolated:** LOOK-T01, LOOK-T04, LOOK-T05.
- **Crash/lifetime/sanitizer:** LOOK-T06, LOOK-T07.
- LOOK-T02 commit/consumption portion is isolated/disposable only.

Deferred, unpromoted:
- mount ChangeLook expiry helper disconnect;
- broad `IsExpireTimeItem()` predicate;
- Aura overlap without separate proven corruption;
- GuildStorage/Roulette/Switchbot omissions without ChangeLook-specific impact;
- conditional free-ticket RIGHT/FREE pointer alias.

This is a **post-audit extension**: the original global audit covered 21/21 STATIC COMPLETE subsystems. Costume / Appearance / ChangeLook raises current effective static/readiness coverage to **22/22**.

The global first future live gate is **DUNGEON-T09**. LOOK tests do not execute before it unless the project phase and order are explicitly changed.


## Runtime-readiness consolidation — Acce / Sash — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- ACCE-T01 -> BUG-ACCE-001
- ACCE-T02 -> BUG-ACCE-002
- ACCE-T03 -> BUG-ACCE-003
- ACCE-T04 -> BUG-ACCE-004
- ACCE-T05 -> BUG-ACCE-005
- ACCE-T06 -> BUG-ACCE-006
- ACCE-T07 -> BUG-ACCE-007
- ACCE-T08 -> BUG-ACCE-008

Execution classes:
- **Ordinary/non-crash state observation:** ACCE-T05.
- **Ordinary/controlled warp state:** ACCE-T07.
- **UI/persistence observation:** ACCE-T08 when suitable metadata-bearing disposable data already exists; otherwise controlled Stage B setup.
- **Modified-client / controlled item-state:** ACCE-T01, T02, T03, T06.
- **Crash/lifetime/sanitizer:** ACCE-T04.

Cross-system note:
- ChangeLook/Acce raw-pointer overlap remains canonically owned by `BUG-LOOK-007 / LOOK-T07`; it is not duplicated under the Acce namespace.

Acce is a **post-audit extension**. Current effective static-complete/readiness coverage is **23/23**, plus the folded Guild lifecycle companion.

The global first future live gate is **DUNGEON-T09**. No Acce test has been executed.


## Runtime-readiness consolidation — Dragon Soul / Alchemy — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- DS-T01 -> BUG-DS-001
- DS-T02 -> BUG-DS-002
- DS-T03 -> BUG-DS-003
- DS-T04 -> BUG-DS-004
- DS-T05 -> BUG-DS-005
- DS-T06 -> BUG-DS-006
- DS-T07 -> BUG-DS-007
- DS-T08 -> BUG-DS-008
- DS-T09 -> BUG-DS-009
- DS-T10 -> BUG-DS-010
- DS-T11 -> BUG-DS-011

Execution classes:
- **Normal/controlled state:** DS-T01, DS-T07, DS-T09.
- **Isolated/modified-client/configuration:** DS-T04, DS-T05, DS-T06, DS-T08, DS-T11.
- **Crash/lifetime/sanitizer:** DS-T02, DS-T03, DS-T10.

Closure notes:
- source/extractor self-aliasing is rejected by the mapped normal ITEM_EXTRACT caller type constraints;
- ChangeLook/Aura/Acce overlap did not produce a separate proven DS destructive alias path;
- malformed grade/step boundaries remain unpromoted because tracked deployment data does not establish malformed reachability;
- the daily-gift zero-ID issue is verified code behavior but configuration-dependent.

Dragon Soul is a **post-audit extension**. Current effective static-complete/readiness coverage is **24/24**, plus the folded Guild lifecycle companion.

The global first future live gate is **DUNGEON-T09**. No Dragon Soul test has been executed.


## Runtime-readiness consolidation — Aura System — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified coverage:
- AURA-T01 -> BUG-AURA-001
- AURA-T02 -> BUG-AURA-002
- AURA-T03 -> BUG-AURA-003
- AURA-T04 -> BUG-AURA-004
- AURA-T05 -> BUG-AURA-005
- AURA-T06 -> BUG-AURA-006

Execution classes:
- **Controlled normal/state observation:** AURA-T01.
- **Modified-client / disposable-data:** AURA-T02, AURA-T03.
- **Debug / packet-initialization / sanitizer:** AURA-T04.
- **Destructive data-integrity / disposable:** AURA-T05.
- **Modified-client arithmetic instrumentation:** AURA-T06.

Unpromoted closure boundaries:
- direct forced `WarpSet()` can preserve Aura state, but no tracked normal producer combining it with an open Aura transaction was established;
- Lua Aura open functions lack a null opener guard, but no tracked current null-opener producer was established;
- current eligible Aura/ABSORB/GROWTH data does not establish timer-driven checked-in slot replacement;
- cross-window state coexistence did not produce an additional distinct same-item alias under current eligibility.

Aura is a **post-audit extension**. Current effective static-complete/readiness coverage is **25/25**, plus the folded Guild lifecycle companion.

The global first future live gate is **DUNGEON-T09**. No Aura test has been executed.


## Runtime-readiness consolidation — Refine / Cube / Crafting — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Canonical verified coverage:
- REFCUBE-T01 -> BUG-REFCUBE-001
- REFCUBE-T02 -> BUG-REFCUBE-002
- REFCUBE-T03 -> BUG-REFCUBE-003
- REFCUBE-T04 -> BUG-REFCUBE-004
- REFCUBE-T05 -> BUG-REFCUBE-005
- REFCUBE-T06 -> BUG-REFCUBE-006
- REFCUBE-T07 -> BUG-REFCUBE-007
- REFCUBE-T08 -> BUG-REFCUBE-008
- REFCUBE-T09 -> BUG-REFCUBE-009
- REFCUBE-T10 -> BUG-REFCUBE-010
- REFCUBE-T11 -> BUG-REFCUBE-011
- REFCUBE-T12 -> BUG-REFCUBE-012
- REFCUBE-T13 -> BUG-REFCUBE-013
- REFCUBE-T14 -> BUG-REFCUBE-014
- REFCUBE-T15 -> BUG-REFCUBE-015
- REFCUBE-T16 -> BUG-REFCUBE-016

Execution classes:
- **Stage A / normal observation:** T10, T11, T12 where no destructive commit is required.
- **Stage B / disposable state or modified client:** T01, T02, T03, T05, T06, T08, T09, T13, T14, T15, T16.
- **Stage C / crash, lifetime, sanitizer or fault-injection:** T04, T07 and any explicit CreateItem-failure experiment.

Closure candidates left unpromoted:
- live DB `refine_proto` semantic bounds are not versioned in tracked repositories;
- scroll/result CreateItem failure after pre-consumption has no tracked normal-data producer;
- Over9 transform metadata loss is dormant because no tracked current quest/config populates/enables its mapping.

Refine / Cube / Crafting is a **post-audit extension**. Current effective static-complete/readiness coverage is **26/26**, plus the folded Guild lifecycle companion.

The global first future live gate is **DUNGEON-T09**. No REFCUBE test has been executed.


## Classic Pet runtime readiness extension
**Static status:** STATIC COMPLETE  
**Execution:** LOCKED / NOT RUN

Canonical tests:
- `PET-T01` — Bruce pickup range Y-axis defect.
- `PET-T02` — Bruce cached raw ground-item pointer lifetime/UAF.
- `PET-T03` — REAL_TIME PET_PAY expiry while summoned.
- `PET-T04` — lost TYPE_SUMMON_PET elapsed achievement time after forced expiry.

Readiness notes:
- current PET_PAY item/race registration coverage is closed at 143 rows / 135 races;
- 135/135 races exist in server mob_proto and client npclist;
- runtime asset presence remains a deployment check because raw client packs are not fully versioned;
- execution order is unchanged: do not run these before the global first live gate `DUNGEON-T09`.


## Dungeon Info gate correction — 2026-09-29
Fresh source verification retracted `DINFO::BUG-DUNGEON-010` and `DUNGEON-T10`.
The `uidungeoninfo.py` list-button loop is outside the zero-count branch.

Updated normal-path order:
1. `DUNGEON-T09` — first future live gate.
2. `DUNGEON-T11`.
3. `DUNGEON-T12`.

`DUNGEON-T10` is historical/retracted and must not be run.
Execution remains locked.


## Dungeon Info normal-path handoffs complete — 2026-09-29
Preflight reverified against current source/data snapshot:
- `DUNGEON-T09` -> malformed Ranking SQL; first future live gate.
- `DUNGEON-T11` -> numeric config flag type `1` is treated as PC while Blue Dragon uses global `dragon_lair_time`.
- `DUNGEON-T12` -> no explicit COOLDOWN in current QUEST-backed blocks; expired/zero arithmetic can wrap to a huge unsigned cooldown.

Canonical handoff files:
- `DUNGEON_T09_HANDOFF.md`
- `DUNGEON_T11_HANDOFF.md`
- `DUNGEON_T12_HANDOFF.md`

Execution remains locked.
