# CURRENT — Canonical Active Checkpoint

**Active subsystem:** Hunting System
**Status:** PARTIAL — static audit nearly complete
**Machine state:** `STATE.json`
**Last updated:** 2026-09-26

## Startup read set
For a normal "ilerleyelim" turn read only:
1. `STATE.json`
2. `CURRENT.md`
3. `systems/hunting.md`

Conditional:
- `bugs/hunting.md` only when validating/recording a bug.
- `tests/hunting.md` only for runtime/fault-injection work.
- source repos: search first, then fetch only exact files/ranges needed.

Do **not** reconstruct state from old chats. GitHub state is canonical.
Do **not** read `archive/`, completed subsystem files, or legacy root maps unless recovery is required.

## Closed this checkpoint
- Hunting static mission/reward table declarations and initializer dimensions validated.
- Levels 1-90 table contents checked for structural zero/missing pairs.
- Money/EXP bands and random reward groups validated.
- Client/server Hunting headers, structs, parser routing and sequence handling validated.
- Quest-flag save lifecycle mapped.
- Item-vs-quest persistence ordering mapped.
- Mission-90 -> level-91 terminal behavior closed end-to-end.
- New verified bug: `BUG-HUNT-005` crash-consistency item duplication window.

Verified bugs: `BUG-HUNT-001..005`.

## Exact next work
1. Validate the 62 unique Hunting reward VNUMs against the actual server item-proto dataset/export.
2. Decide whether the unchecked `CreateItem(nullptr)` path is reachable with current data and should receive a new verified bug ID.
3. Then mark Hunting STATIC COMPLETE and transition to the next subsystem/runtime phase.

## Known blocker
The checked-in `Project_DumpProto/tr/item_proto.txt` export is non-UTF8 and large; the GitHub connector cannot decode/read its content directly. No claim about missing reward VNUMs should be made until this dataset is read successfully.

## End-of-turn write rule
When meaningful progress is made:
1. update `systems/hunting.md`;
2. update `bugs/hunting.md` / `tests/hunting.md` only if affected;
3. overwrite this file with the new short cursor;
4. update `STATE.json`.

Keep this file short and overwrite-only. Git history is the checkpoint history.
