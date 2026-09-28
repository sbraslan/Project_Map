# Project_Map — Subsystem Index

This is the navigation index. Normal continuation starts at `STATE.json`, then `CURRENT.md`.

## Core rule
- Source repos are **read-only**: `Project_ClientSrc`, `Project_ServerSRC`, `Project_Binary`, `Project_Game`, `Project_DumpProto`.
- Only `Project_Map` is writable for mapping/checkpoints.
- Never load all map files into one chat.
- Never reconstruct the active state from old chats when `STATE.json` is available.
- Legacy monolithic files are preserved under `archive/` and are not part of normal continuation.

## Subsystems

| Subsystem | Status | Canonical file |
|---|---|---|
| Guild Storage | STATIC COMPLETE | `systems/guild_storage.md` |
| Guild lifecycle | MAPPED WITH GUILD STORAGE | `systems/guild.md` |
| Inventory / Item / Special Inventory | STATIC COMPLETE | `systems/inventory_items.md` |
| Switchbot | STATIC COMPLETE | `systems/switchbot.md` |
| Exchange / Trade | STATIC COMPLETE | `systems/exchange.md` |
| Shop / Premium Private Shop | STATIC COMPLETE | `systems/shop.md` |
| Safebox / Mall | STATIC COMPLETE | `systems/safebox_mall.md` |
| Mailbox | STATIC COMPLETE | `systems/mailbox.md` |
| Ticket System | STATIC COMPLETE | `systems/ticket.md` |
| Dungeon Info | STATIC COMPLETE | `systems/dungeon_info.md` |
| Battle Pass | STATIC COMPLETE | `systems/battle_pass.md` |
| Achievement System | STATIC COMPLETE | `systems/achievement.md` |
| Biolog System | STATIC COMPLETE | `systems/biolog.md` |
| Hunting System | STATIC COMPLETE | `systems/hunting.md` |
| Ranking System | STATIC COMPLETE | `systems/ranking.md` |
| Party System | STATIC COMPLETE | `systems/party.md` |
| Party Match | STATIC COMPLETE | `systems/party_match.md` |
| Dungeon Core | STATIC COMPLETE | `systems/dungeon_core.md` |
| Battle Field System | STATIC COMPLETE | `systems/battle_field.md` |
| World Lottery System | STATIC COMPLETE | `systems/world_lottery.md` |
| World Boss System | STATIC COMPLETE | `systems/world_boss.md` |
| Sung Mahi Tower | STATIC COMPLETE | `systems/sung_mahi_tower.md` |
| Costume / Appearance / ChangeLook | STATIC COMPLETE | `systems/costume_appearance.md` |

## Current phase
- **Detection / mapping only.** Source and game repositories remain read-only.
- Deferred runtime/fault-injection inventory: `RUNTIME.md` (documentation only; not active execution).

## Supporting files
Each subsystem can have:
- `bugs/<system>.md`
- `tests/<system>.md`

Open those only when needed.

## Legacy archive
Full pre-migration maps/checkpoints are in `archive/`. They are fallback evidence, not startup context.


## Runtime-readiness milestone — 2026-09-28
All STATIC COMPLETE subsystem rows currently represented by the canonical mapping have deferred runtime-readiness coverage or an explicitly folded ownership path. Runtime execution remains locked. The active documentation cursor is now the global readiness integrity audit; first future live gate remains DUNGEON-T10.


## Global readiness integrity audit — 2026-09-28
Canonical audit: `READINESS_AUDIT.md`.

Result: **COMPLETE — PASS WITH NORMALIZATIONS**.
- 21/21 STATIC COMPLETE subsystem rows have runtime-readiness ownership.
- Guild lifecycle is folded with Guild Storage but `BUG-GUILD-001` has explicit `GUILD-T01` companion coverage.
- Dungeon Info/Core historical bug-ID collision is globally namespace-qualified.
- Retracted/cross-system IDs and legacy test aliases are normalized.
- First future live gate remains `DUNGEON-T10`.


## First runtime gate handoff — 2026-09-28
`DUNGEON_T10_HANDOFF.md` is the canonical checklist for the first future live test.

Status: **PREFLIGHT COMPLETE / EXECUTION LOCKED / NOT RUN**.

This does not change the project phase or authorize source/runtime changes.


## Post-audit extension — Costume / Appearance — 2026-09-28
Costume / Appearance / ChangeLook was mapped after the original 21/21 readiness audit and is now **STATIC COMPLETE** with explicit runtime-readiness ownership.

Current effective coverage is **22/22 STATIC COMPLETE subsystem rows** plus the folded Guild lifecycle companion.

The historical `READINESS_AUDIT.md` 21/21 result remains valid for its original scope; a post-audit extension records this additional subsystem. The first future live gate remains `DUNGEON-T10`.
