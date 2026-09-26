# Low-Context Continuation Protocol

Goal: continue the Metin2 mapping project indefinitely without depending on chat history or filling context.

## Source of truth
1. `STATE.json` — machine-readable active state and source snapshot.
2. `CURRENT.md` — short human-readable cursor.
3. `INDEX.md` — subsystem status/navigation.
4. `systems/<active>.md` — canonical technical map for the active subsystem.

If an old chat summary disagrees with GitHub, **GitHub wins**.

## On every "ilerleyelim"
1. Read `STATE.json`.
2. Read `CURRENT.md`.
3. Read only the active `systems/<name>.md`.
4. Open `bugs/<name>.md` only when a bug is being checked/added.
5. Open `tests/<name>.md` only for runtime/fault-injection work.
6. In source repos, search for exact symbols first; then read only exact files/ranges needed.
7. Save meaningful findings to the active subsystem file.
8. Overwrite `CURRENT.md` with the new short cursor.
9. Update `STATE.json` so the next chat can resume without prior conversation.

## Context budget
- Startup target: 3 files only.
- Never bulk-read `systems/`, `bugs/`, `tests/`, or `archive/`.
- Never reread completed subsystems unless a source change invalidates them.
- Prefer symbol search + bounded line-range reads over loading whole large source files.
- Chat replies should report only: what changed, new bug IDs if any, checkpoint saved, next exact target.

## Repository safety
Read-only source repositories:
- `Project_ClientSrc`
- `Project_ServerSRC`
- `Project_Binary`
- `Project_Game`
- `Project_DumpProto`

Writable mapping/checkpoint repository:
- `Project_Map`

At all times in the current detection/mapping phase, never modify source repositories.

## End-of-turn transaction
Treat checkpoint writing as one logical transaction:
- active `systems/<name>.md`
- affected `bugs/<name>.md`
- affected `tests/<name>.md`
- `CURRENT.md`
- `STATE.json`
- `INDEX.md` only if subsystem status changed

Do not append historical prose to `CURRENT.md`. Git commits already preserve history.

## Static-complete transition
When a subsystem closes:
1. mark it STATIC COMPLETE in its system file;
2. update bug/test files;
3. update its row in `INDEX.md`;
4. choose the next subsystem;
5. overwrite `CURRENT.md`;
6. update `STATE.json`.

## Source-change invalidation
`STATE.json` records source repository HEAD snapshots. If a source repo later changes, compare the new commit only against affected mapped areas and reopen only impacted subsystems.

## Recovery
Use `archive/` only if the canonical active files are missing/corrupt or an old decision must be recovered. It is never normal startup context.


## Detection-only lock
Current project phase is **detection / mapping only**.

Hard constraints:
- only `Project_Map` may be changed;
- all source/game repositories are immutable/read-only;
- do not patch C++ or Python;
- do not edit quest files;
- do not edit game/config/data files;
- do not commit/push to any source repository;
- runtime/fault-injection tests may be documented but not executed unless the user explicitly changes phase.

Every finding must be persisted as documentation/checkpoint data inside `Project_Map`.
