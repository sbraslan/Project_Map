# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Growth Pet System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 11 VERIFIED STATIC BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Growth Pet System  
**System:** `systems/growth_pet.md`  
**Bugs:** `bugs/growth_pet.md`  
**Tests:** `tests/growth_pet.md`  
**Last completed subsystem:** Refine / Cube / Crafting  
**Effective completed/readiness-covered subsystems:** 26  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Refine / Cube / Crafting closure

Refine / Cube / Crafting is now **STATIC COMPLETE**.

Canonical verified findings:
`BUG-REFCUBE-001..BUG-REFCUBE-016`.

Canonical deferred tests:
`REFCUBE-T01..REFCUBE-T16`, none executed.

Closure includes:
- Cube multiplier/accounting/open-state/NPC/improve-item/control-field boundaries;
- classic normal/scroll/Serpent refine lifetime and authorization;
- Soul scroll routing;
- refine-skill and scroll preview/execution probability drift;
- DB `refine_proto` deployment-source mapping;
- current 3327-section Cube data domain audit;
- MONEY_ONLY Devil Tower/Serpent execution-time authorization;
- dormant Over9 exposure;
- client refine UI/session lifecycle;
- transform metadata loss for ChangeLook, Set Item identity, Serpent random-default values and Basic starter flag.

Unpromoted:
- live DB `refine_proto` semantic values are outside tracked repository data;
- result-item creation failure after pre-consumption requires fault injection / abnormal creation failure;
- Over9 metadata loss has no tracked current producer.

Runtime-readiness ownership is complete. Current effective coverage is **26/26 STATIC COMPLETE** subsystem rows plus folded Guild lifecycle.

## Active Growth Pet state

Canonical verified findings:
`BUG-GPET-001..BUG-GPET-011`.

Current findings cover:
- 55713 hatch-info table out-of-bounds;
- feed packet count out-of-bounds;
- skill-slot array out-of-bounds;
- arbitrary/non-skill item accepted as skill book;
- premium revive type/count trust and stack inflation;
- duplicate evolution-material cell alias;
- revive birthday/duration changes lost in local copy;
- specialist skill table grouping/lookup defect;
- HEAL absolute-value-as-delta over-heal;
- unbounded packet-name strlen;
- hatch socket1 duration interpreted as birth timestamp for final evolution age.

Deferred tests:
`GPET-T01..GPET-T11`, none executed.

## Exact next work
1. finish summon/dismiss/death/real-time expiry/rewarp lifetime;
2. audit EXP table and item/mob EXP arithmetic through level 105;
3. audit active/passive skill execution beyond HEAL;
4. audit name-change normal unsummoned branch;
5. audit attribute determine/change state and material consumption beyond the 55713 OOB;
6. close DB pet-table save/load/delete ownership and orphan lifecycle;
7. close current 55701..55713 proto/race/client asset coverage;
8. consolidate Growth Pet runtime ownership and decide STATIC COMPLETE.

Do not execute `GPET-T01..GPET-T11` or any other runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
