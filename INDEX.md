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
| Ticket System | STATIC MAPPED | `systems/ticket.md` |
| Dungeon Info | STATIC MAPPED | `systems/dungeon_info.md` |
| Battle Pass | STATIC COMPLETE | `systems/battle_pass.md` |
| Achievement System | STATIC COMPLETE | `systems/achievement.md` |
| Biolog System | STATIC COMPLETE | `systems/biolog.md` |
| Hunting System | **PARTIAL — ACTIVE** | `systems/hunting.md` |

## Supporting files
Each subsystem can have:
- `bugs/<system>.md`
- `tests/<system>.md`

Open those only when needed.

## Legacy archive
Full pre-migration maps/checkpoints are in `archive/`. They are fallback evidence, not startup context.
