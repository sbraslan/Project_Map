# Global Runtime-Readiness Integrity Audit

**Date:** 2026-09-28  
**Phase:** Detection / Mapping Only  
**Result:** COMPLETE — PASS WITH NORMALIZATIONS  
**Execution:** LOCKED / NOT RUN

Only `sbraslan/Project_Map` is writable. No source/game/runtime action is authorized by this audit.

## Coverage result

- `INDEX.md` contains **21 STATIC COMPLETE** subsystem rows.
- All 21 have a matching entry in `STATE.json -> runtime_readiness`.
- `Guild lifecycle` is intentionally **MAPPED WITH GUILD STORAGE**, but it contains one independent verified bug and therefore receives folded companion runtime ownership:
  - `GUILD-T01 -> BUG-GUILD-001`.
- No STATIC COMPLETE subsystem is missing deferred runtime-readiness coverage.
- No runtime test has been executed.

## Global bug-ID normalization

### Dungeon namespace collision
Historical files reused `BUG-DUNGEON-001..004` for two different subsystems:
- Dungeon Info: `bugs/dungeon_info.md`
- Dungeon Core: `bugs/dungeon_core.md`

Historical IDs are not renumbered during detection-only phase.

**Global references must therefore use qualified labels:**
- `DINFO::BUG-DUNGEON-001..012`
- `DCORE::BUG-DUNGEON-001..004`

Unqualified `BUG-DUNGEON-001..004` is only valid inside its local subsystem file.

### Existing normalized exceptions
- `BUG-EXCHANGE-004` is historically reused for two distinct findings; global references must include the descriptive qualifier `currency_atomicity` or `final_distance`.
- `BUG-RANK-006` is **RETRACTED / RESERVED**.
- `BUG-BFIELD-004` is **RETRACTED / RESERVED**.
- Historical `BUG-BFIELD-005` is owned canonically by `BUG-RANK-003`.
- Historical `BUG-BFIELD-007` is owned canonically by `BUG-RANK-007`.
- `BUG-ITEM-005` does not exist; the numbering gap is intentional.
- Guild Storage candidate IDs remain candidates unless already superseded by verified `BUG-GS-003/004`.
- Safebox `BUG-SAFEBOX-001/002` remain dormant while `ENABLE_SAFEBOX_MONEY` is off.
- Historical `OBS-SAFEBOX-001` is subsumed by `BUG-SAFEBOX-005`.
- Early provisional Shop records remain historical; the later canonical Shop index controls runtime ownership.
- Sung Mahi deferred architectural/data questions remain unpromoted.

## Canonical test ownership

| Prefix | Canonical owner |
|---|---|
| DUNGEON-T | `tests/dungeon_info.md` |
| DCORE-T | `tests/dungeon_core.md` |
| TICKET-T | `tests/ticket.md` |
| HUNT-T | `tests/hunting.md` |
| BP-T | `tests/battle_pass.md` |
| ACH-T | `tests/achievement.md` |
| BIO-T | `tests/biolog.md` |
| ITEM-T | `tests/inventory_items.md` |
| SWB-T | `tests/switchbot.md` |
| GS-T | `tests/guild_storage.md` |
| GUILD-T | `tests/guild.md` |
| EXC-T | `tests/exchange.md` |
| SHP-T | `tests/shop.md` |
| SFB-T | `tests/safebox_mall.md` |
| MAIL-T | `tests/mailbox.md` |
| RANK-T | `tests/ranking.md` |
| PARTY-T | `tests/party.md` |
| PMATCH-T | `tests/party_match.md` |
| BFIELD-T | `tests/battle_field.md` |
| WLOT-T | `tests/world_lottery.md` |
| WB-T | `tests/world_boss.md` |
| SMT-T | `tests/sung_mahi_tower.md` |

Migrated files contain copied legacy blocks. A test ID found outside its canonical owner file is historical only unless `STATE.json` explicitly says otherwise.

Non-canonical historical forms include:
- `SWITCHBOT-Txx`;
- `EX-Txx` / `EXCHANGE-Txx`;
- `SHOP-Txx`;
- old `TEST-WB-*` headings;
- copied `GUILD-T01`, ITEM, Switchbot, Dungeon Info or Battle Pass blocks in unrelated migrated test files.

## Canonical future execution order

Runtime remains locked. This is an ordering document only.

### Stage A — normal/current-flow first
1. `DUNGEON-T10 -> DUNGEON-T09 -> DUNGEON-T11 -> DUNGEON-T12`
2. `TICKET-T07`
3. `HUNT-T04`
4. `BP-T02`
5. `ACH-T03 -> ACH-T04`
6. `BIO-T01`
7. `ITEM-T01`
8. `SWB-T01 -> SWB-T03`
9. `GS-T01 -> GS-T02 -> GS-T17`
10. `GUILD-T01`
11. `EXC-T01 -> EXC-T02`
12. `SHP-T02 -> SHP-T03`
13. `SFB-T09` — observation-only ordinary-flow check
14. `MAIL-T07 -> MAIL-T11`
15. `RANK-T01 -> RANK-T02 -> RANK-T03 -> RANK-T04`
16. `PARTY-T01 -> PARTY-T02 -> PARTY-T05`
17. `PMATCH-T03`
18. `DCORE-T01 -> DCORE-T02`
19. `BFIELD-T01 -> T02 -> T03 -> T09 -> T10 -> T11`
20. `WLOT-T04 -> WLOT-T10`
21. `WB-T09 -> WB-T13 -> WB-T15 -> WB-T16`
22. `SMT-T01`

**Global first live gate remains `DUNGEON-T10`.**

### Stage B — isolated / modified-client / admin / parser / controlled-state
Run only after Stage A and only in an isolated environment:
- Dungeon Info `DUNGEON-T01..T08`;
- Ticket `TICKET-T01..T06`;
- Hunting `T01/T02/T03/T07/T08`;
- remaining non-persistence Battle Pass tests after BP-T02;
- Achievement `T01/T05/T06/T08`;
- Biolog `T02/T03/T04/T06/T07/T08/T09/T10/T11`;
- Inventory `T02/T03/T04/T05/T06/T07/T08/T09/T10/T11/T12/T13`;
- Switchbot `T02/T04/T05/T06/T07`;
- Guild Storage controlled/candidate validation not reserved for Stage C;
- Exchange `T03/T04/T05/T06/T07/T09/T10`;
- Shop isolated/adversarial observation tests not reserved for Stage C;
- Safebox/Mall isolated, dormant and parser/state tests;
- Mailbox adversarial/observation tests;
- Ranking `T05/T07`;
- Party `T03/T04/T06`;
- Party Match `T01/T02/T04/T05`;
- Dungeon Core `T03/T04`;
- Battle Field `T06/T08`;
- World Lottery non-crash controlled tests;
- World Boss scheduler/multicore/client/ranking controlled tests;
- Sung Mahi `T02/T03/T04/T05/T06`.

### Stage C — crash consistency / persistence / fault injection
Examples with explicit current ownership:
- `HUNT-T05/T06`;
- Battle Pass persistence/crash cases from `tests/battle_pass.md`;
- `ACH-T02/T07/T09`;
- `BIO-T05`;
- Guild Storage pending-load/restart/disband/persistence cases;
- `EXC-T08`;
- `SHP-T01/T07/T10`;
- Safebox DB-load failure state validation where fault injection is required;
- `MAIL-T10/T12/T15`;
- `WLOT-T13/T14`.

Do not move a test between stages merely because it is easy to trigger; preserve the safety/isolation class documented in its canonical subsystem file.

## Integrity outcome

Resolved by documentation normalization:
1. all 21 STATIC COMPLETE subsystem rows have runtime-readiness ownership;
2. Guild lifecycle `BUG-GUILD-001` is no longer omitted from global runtime ownership;
3. Dungeon Info/Core bug-ID collision is globally qualified without renumbering history;
4. retracted Ranking/Battle Field IDs remain excluded from active validation;
5. cross-system Battle Field -> Ranking ownership is explicit;
6. candidate/observation/dormant statuses remain non-promoted;
7. migrated duplicate test blocks are non-canonical outside their owner file;
8. one future execution order is established with `DUNGEON-T10` first.

## Next documentation-only handoff

Prepare the `DUNGEON-T10` runtime gate handoff/checklist without running it. Runtime execution requires an explicit user phase change.


## Post-audit extension — Costume / Appearance / ChangeLook — 2026-09-28

The original audit result above remains historically scoped to the 21 STATIC COMPLETE subsystem rows that existed at audit time.

A subsequent static-mapping extension added:
- `Costume / Appearance / ChangeLook`
- verified bugs `BUG-LOOK-001..007`
- canonical tests `LOOK-T01..LOOK-T07`
- readiness ownership in `RUNTIME.md`

Current effective static-complete/readiness coverage is therefore **22/22**, plus the folded Guild lifecycle companion.

This extension does not change the previously recorded execution order's first gate. **DUNGEON-T10 remains the first future live gate.**


## Post-audit extension — Acce / Sash — 2026-09-28

The original audit remains historically scoped to the 21 STATIC COMPLETE rows that existed at audit time.

Subsequent extensions now include:
- Costume / Appearance / ChangeLook — `BUG-LOOK-001..007`, `LOOK-T01..LOOK-T07`;
- Acce / Sash — `BUG-ACCE-001..008`, `ACCE-T01..ACCE-T08`.

Current effective static-complete/readiness coverage is therefore **23/23**, plus the folded Guild lifecycle companion.

Acce execution classes are documented in `RUNTIME.md`. No Acce runtime test has been executed.

This extension does not alter the canonical future order's first gate. **DUNGEON-T10 remains the first future live gate.**
