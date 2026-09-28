# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Post-Audit Readiness Extension Complete
**Status:** STATIC MAPPING COMPLETE / 22-OF-22 READINESS COVERED / EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last completed subsystem:** Costume / Appearance / ChangeLook
**Item workflow:** `ITEM_CREATION_AUTOMATION.md`
**First future live gate:** `DUNGEON-T10`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Costume / Appearance closure
Status: **STATIC COMPLETE**

Verified:
- `BUG-LOOK-001..007`

Canonical deferred tests:
- `LOOK-T01..LOOK-T07`

Readiness:
- documentation READY;
- execution NOT RUN;
- LOOK-T02 eligibility-only and LOOK-T03 are the least invasive future observations;
- they do not supersede `DUNGEON-T10`.

Deferred / not promoted:
- mount expiry helper disconnect;
- broad IsExpireTimeItem predicate;
- Aura overlap without separate proven corruption;
- GuildStorage/Roulette/Switchbot omissions without extra proven ChangeLook impact;
- conditional free-ticket RIGHT/FREE pointer alias without closed ticket-proto eligibility.

## Global state
The original readiness audit covered 21/21 subsystem rows.
This post-audit extension adds Costume / Appearance / ChangeLook.

**Current effective coverage: 22/22 STATIC COMPLETE subsystems with runtime-readiness ownership**, plus folded Guild lifecycle coverage.

## Item workflow
When the user says `item ekleyeceğiz`, use `ITEM_CREATION_AUTOMATION.md`:
reference VNUM -> compatible template -> unused VNUM/ShapeIndex proposal -> item_proto + companion item/MSM records.

## Next
No runtime execution is authorized.
If static mapping continues, open the next unmapped subsystem deliberately and save it in Project_Map.
Otherwise the prepared first runtime handoff remains `DUNGEON-T10`.

GitHub state is canonical.
