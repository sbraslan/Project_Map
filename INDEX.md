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
| Acce / Sash | STATIC COMPLETE | `systems/acce.md` |
| Dragon Soul / Alchemy | STATIC COMPLETE | `systems/dragon_soul.md` |
| Aura System | STATIC COMPLETE | `systems/aura.md` |
| Refine / Cube / Crafting | STATIC COMPLETE | `systems/refine_cube.md` |
| Growth Pet System | STATIC COMPLETE | `systems/growth_pet.md` |
| Horse / Mount / Riding | MAPPING IN PROGRESS | `systems/horse_mount.md` |

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


## Post-audit extension — Acce / Sash — 2026-09-28
Acce / Sash is now **STATIC COMPLETE** with verified `BUG-ACCE-001..008` and canonical deferred tests `ACCE-T01..ACCE-T08`.

Current effective coverage is **23/23 STATIC COMPLETE subsystem rows** plus the folded Guild lifecycle companion. Dragon Soul / Alchemy remains separately **MAPPING IN PROGRESS**.

Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.



## Post-audit extension — Dragon Soul / Alchemy — 2026-09-28
Dragon Soul / Alchemy is now **STATIC COMPLETE** with verified `BUG-DS-001..011` and canonical deferred tests `DS-T01..DS-T11`.

Current effective coverage is **24/24 STATIC COMPLETE subsystem rows** plus the folded Guild lifecycle companion. Aura System is the next active mapping cursor.

Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.


## Post-audit extension — Aura System — 2026-09-28
Aura System is now **STATIC COMPLETE** with verified `BUG-AURA-001..006` and canonical deferred tests `AURA-T01..AURA-T06`.

Current effective coverage is **25/25 STATIC COMPLETE subsystem rows** plus the folded Guild lifecycle companion.

Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.


## Active continuation — Growth Pet System — 2026-09-28
Aura closure raises effective static-complete/readiness coverage to **25/25** plus folded Guild lifecycle.

Growth Pet System is the next active mapping cursor:
- system: `systems/growth_pet.md`;
- bugs: `bugs/growth_pet.md`;
- tests: `tests/growth_pet.md`;
- first verified finding: `BUG-GPET-001`.

Growth Pet is not yet counted as STATIC COMPLETE. Runtime remains locked.


## Active extension — Refine / Cube / Crafting — 2026-09-28
Aura System is **STATIC COMPLETE** with `BUG-AURA-001..006` and `AURA-T01..AURA-T06`.

Current effective completed/readiness coverage is **25/25 STATIC COMPLETE subsystem rows** plus the folded Guild lifecycle companion.

The active mapping cursor is now **Refine / Cube / Crafting**. Initial Cube Renewal mapping has already verified `BUG-REFCUBE-001..002`.

Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.


## Post-audit extension — Refine / Cube / Crafting — 2026-09-28
Refine / Cube / Crafting is now **STATIC COMPLETE** with verified `BUG-REFCUBE-001..016` and canonical deferred tests `REFCUBE-T01..REFCUBE-T16`.

Current effective static-complete/readiness coverage is **26/26** subsystem rows plus the folded Guild lifecycle companion.

Growth Pet System remains the active mapping cursor. Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.


## Post-audit extension — Growth Pet System — 2026-09-29
Growth Pet System is now **STATIC COMPLETE** with verified `BUG-GPET-001..020` and canonical deferred tests `GPET-T01..GPET-T20`.

Current effective static-complete/readiness coverage is **27/27** subsystem rows plus the folded Guild lifecycle companion.

The active mapping cursor is now **Horse / Mount / Riding**:
- `systems/horse_mount.md`;
- `bugs/horse_mount.md`;
- `tests/horse_mount.md`.

Runtime execution remains locked and `DUNGEON-T10` remains the first future live gate.
