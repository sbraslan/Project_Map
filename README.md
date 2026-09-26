# Project_Map

Metin2 projesinin salt-okuma kaynak haritası ve kalıcı teknik checkpoint deposu.

## Start here
**Normal çalışma başlangıcı: `CURRENT.md`.**

Then follow its active subsystem pointer. Do not load the legacy map set on every turn.

## Architecture
- `CURRENT.md` — tiny overwrite-only active checkpoint
- `INDEX.md` — subsystem status/navigation
- `WORKFLOW.md` — low-context continuation contract
- `systems/` — one canonical map per subsystem
- `bugs/` — subsystem-scoped bug registries
- `tests/` — subsystem-scoped runtime/fault-injection tests
- `archive/` — full legacy monolithic files, preserved but excluded from normal startup

## Safety rule
Source repos are read-only. Mapping writes go only to `Project_Map`.

## Why this layout exists
The old monolithic progress/bug/test files became large enough that rereading them filled chat context. The pointer-based layout makes each continuation load only the active subsystem.
