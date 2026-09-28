# Sung Mahi Tower — Deferred Runtime Validation

**Status:** DEFERRED — STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Date:** 2026-09-27
**Canonical bugs:** `bugs/sung_mahi_tower.md`

> This file documents later validation only. Do not execute tests or modify source/game/config/data until the user explicitly changes phase.

## Test order

### SMT-T01 — Missing tower quest runtime normal-path check
**Covers:** BUG-SMT-001  
**Class:** Stage A / normal path

When runtime testing is explicitly enabled:
1. enter map 386 (`metin2_map_smhdungeon_01`) normally;
2. interact with tower NPC vnum 4020;
3. observe whether a Sung Mahi quest/button entry exists and whether the client receives the expected tower UI commands;
4. if the entry window opens, record `sungMahiQuest` behavior and whether entry reaches a private map-387 instance.

**Repository expectation:** no tracked quest source/list/object/4020 handler exists, so the checked-in runtime package alone should not provide the mapped quest flow.

**Pass/fail evidence to capture:**
- NPC interaction response;
- tower entry UI presence;
- server syserr/quest logs;
- resulting map index if entry succeeds.

### SMT-T02 — Monthly reset missing Lua library
**Covers:** BUG-SMT-002  
**Class:** Stage B / isolated time-event validation

Requires a disposable test environment or controlled event invocation. Do not alter production clock/data.

Trigger the monthly rollover path and capture the `lua_dofile` failure for:
`Questlibs/dungeonInfoLibrary.lua`.

Also verify whether the SQL ranking table is truncated despite the Lua hook failure.

### SMT-T03 — Monthly mailbox packet memory-safety validation
**Covers:** BUG-SMT-003  
**Class:** Stage B / ASan-debug only

Use an isolated ASan/debug build. Trigger a monthly reward with at least one ranking winner.

Observe:
- invalid read originating from fixed-width `memcpy` of title/from/message/recipient fields;
- malformed or unterminated mailbox title;
- subsequent DB/mailbox serialization behavior.

Do not run this test on a production server.

### SMT-T04 — Cross-year same-month rollover
**Covers:** BUG-SMT-004  
**Class:** Stage B / isolated state-time simulation

In a disposable DB/server copy:
1. set persistent `sungMahiLastMonth` to month N;
2. leave ranking rows present;
3. start the game core with a date in month N of a later year;
4. wait for the reward timer check.

**Expected from static analysis:** month-only equality suppresses rollover because year is not part of the season identity.

### SMT-T05 — Dark king elemental resistance comparison
**Covers:** BUG-SMT-005  
**Class:** Stage A/B / data validation

Compare vnum 7591 (dark king) with the five corresponding elemental kings and higher dark king 7614.

Static values:
- 7591: `AttDark=55`, `ResistDark=-1`
- expected family pattern: matching element resistance `-30`

A runtime comparison is optional because the data defect is already deterministic statically. If tested, use controlled damage/stat inspection rather than changing proto data.

### SMT-T06 — Tower-only consumable proto availability
**Covers:** BUG-SMT-006  
**Class:** Stage A / proto-load validation

Check whether vnums 70390–70395 and 70405 exist in the actual loaded item proto.

Tracked repository state:
- names exist;
- client icons exist;
- server gating code expects the vnums;
- EN/DE/TR source `item_proto.txt` rows are absent.

If a live/prebuilt binary proto does contain them, record that as deployment drift from the tracked proto sources rather than retracting the source-integration bug.

## Deferred candidates — no test yet
Do not promote or test these until the missing producer/schema is recovered:
- `pc.mailbox_reward` null-mailbox pointer precondition;
- `smhgate_flower` 9100–9107 missing server monster folder;
- ranking `player_login` versus character-name mailbox recipient semantics;
- malformed/out-of-range `dungeonLevel` values;
- `m_bDungeon_Difficulty` / `dungeonLevel` synchronization.

## Phase lock
No runtime action has been performed. Source/game repositories remain read-only.


## Sung Mahi Tower readiness consolidation — 2026-09-28
- No Sung Mahi Tower runtime test was executed.
- SMT-T01 -> BUG-SMT-001.
- SMT-T02 -> BUG-SMT-002.
- SMT-T03 -> BUG-SMT-003.
- SMT-T04 -> BUG-SMT-004.
- SMT-T05 -> BUG-SMT-005.
- SMT-T06 -> BUG-SMT-006.
- Primary normal/current-repository candidate: SMT-T01.
- SMT-T05 is deterministic data validation and may not need runtime execution if static evidence is sufficient.
- SMT-T06 is deployment/proto-load validation and must distinguish tracked-source state from any stale/external binary proto.
- SMT-T02/T03/T04 require isolated time/state or ASan/debug environments.
- The following remain **deferred findings, not verified bug IDs**:
  - nullable `pc.mailbox_reward` mailbox pointer precondition;
  - missing `smhgate_flower` 9100–9107 server monster folder without a tracked producer;
  - ranking `player_login` versus character-name mailbox recipient semantics;
  - malformed/out-of-range quest-produced `dungeonLevel`;
  - `m_bDungeon_Difficulty` versus `dungeonLevel` synchronization dependency.
- Overall first live runtime gate remains DUNGEON-T10.
