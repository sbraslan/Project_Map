# Low-Context Continuation Protocol

Goal: continue long Metin2 static mapping without filling chat history.

## On every "ilerleyelim"
1. Read `CURRENT.md`.
2. Read only the active `systems/<name>.md`.
3. Read `bugs/<name>.md` only when checking/adding a bug.
4. Read `tests/<name>.md` only when planning/running runtime tests.
5. Fetch only exact source files/ranges needed from read-only source repos.
6. Save meaningful findings to the active subsystem files.
7. **Overwrite** `CURRENT.md` with the new short checkpoint.

## Never do by default
- Do not read `archive/`.
- Do not read every subsystem.
- Do not reread completed systems.
- Do not append historical prose to `CURRENT.md`.
- Do not repeat the full project recap in chat.
- Do not modify source repos during mapping.

## Static-complete transition
When a subsystem closes:
1. mark it STATIC COMPLETE in its system file;
2. update its bug/test files;
3. change its row in `INDEX.md`;
4. choose the next unmapped subsystem;
5. overwrite `CURRENT.md` to point to that subsystem.

## Source-change invalidation
If a source repo commit changes in a mapped area, reopen only the affected subsystem, not the whole project.

## Chat output rule
Progress messages should be short: what closed, what new bug IDs appeared, what checkpoint was written, and the next exact target. Full detail lives in GitHub.
