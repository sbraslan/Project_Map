# Project_Map

Metin2 projesinin salt-okuma kaynak haritası ve kalıcı teknik checkpoint deposu.

## Start here
**Normal çalışma başlangıcı: `STATE.json` -> `CURRENT.md` -> aktif `systems/<name>.md`.**

Yeni sohbette geçmiş konuşmaları taşımaya gerek yok. Şu kısa komut yeterlidir:

> `sbraslan/Project_Map STATE.json dosyasını oku ve aktif checkpointten WORKFLOW.md kurallarına göre ilerle.`

Sonrasında aynı sohbet içinde yalnızca **"ilerleyelim"** denebilir.

## Architecture
- `STATE.json` — machine-readable project memory, active cursor and source snapshot
- `CURRENT.md` — tiny overwrite-only human checkpoint
- `INDEX.md` — subsystem status/navigation
- `WORKFLOW.md` — low-context continuation contract
- `systems/` — one canonical map per subsystem
- `bugs/` — subsystem-scoped bug registries
- `tests/` — subsystem-scoped runtime/fault-injection tests
- `history/` — rare architecture/migration checkpoints; never normal startup context
- `archive/` — legacy monolithic files; recovery only

## Safety rule
Source repos are read-only. Mapping writes go only to `Project_Map`.

## Core principle
**Do not rebuild project state from chat history. Persist it to GitHub, then resume from GitHub.**
