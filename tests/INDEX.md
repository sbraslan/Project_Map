# Runtime tests

Runtime/fault-injection plans are split by subsystem.

Normal rule: read only the active subsystem's test file when runtime work begins. Static mapping does not require loading test files every turn.


## Canonical ownership — 2026-09-28
See `../READINESS_AUDIT.md` for the full global order.

Canonical test prefixes:
- `DUNGEON-T` -> `dungeon_info.md`
- `DCORE-T` -> `dungeon_core.md`
- `TICKET-T` -> `ticket.md`
- `HUNT-T` -> `hunting.md`
- `BP-T` -> `battle_pass.md`
- `ACH-T` -> `achievement.md`
- `BIO-T` -> `biolog.md`
- `ITEM-T` -> `inventory_items.md`
- `SWB-T` -> `switchbot.md`
- `GS-T` -> `guild_storage.md`
- `GUILD-T` -> `guild.md`
- `EXC-T` -> `exchange.md`
- `SHP-T` -> `shop.md`
- `SFB-T` -> `safebox_mall.md`
- `MAIL-T` -> `mailbox.md`
- `RANK-T` -> `ranking.md`
- `PARTY-T` -> `party.md`
- `PMATCH-T` -> `party_match.md`
- `BFIELD-T` -> `battle_field.md`
- `WLOT-T` -> `world_lottery.md`
- `WB-T` -> `world_boss.md`
- `SMT-T` -> `sung_mahi_tower.md`
- `LOOK-T` -> `costume_appearance.md`

Migrated files contain copied foreign blocks. A test ID outside its canonical owner file is historical only unless `STATE.json` explicitly says otherwise.
Legacy `SWITCHBOT-Txx`, `EX-Txx`, `EXCHANGE-Txx`, `SHOP-Txx`, and `TEST-WB-*` forms are non-canonical aliases/history.


## Post-audit prefix extension — 2026-09-28
- `LOOK-T` -> `costume_appearance.md`

This subsystem was added after the original global readiness audit. Its test ownership is canonical here; global first execution gate remains `DUNGEON-T09`.


## Dragon Soul prefix ownership — 2026-09-28
- `DS-T` -> `dragon_soul.md`

Dragon Soul is STATIC COMPLETE. `DS-T01..DS-T11` are canonical deferred plans; none has been executed. Global first execution gate remains `DUNGEON-T09`.


## Acce / Sash prefix ownership — 2026-09-28
- `ACCE-T` -> `acce.md`

Acce is STATIC COMPLETE. `ACCE-T01..ACCE-T08` are canonical deferred tests; none has been executed. Global first execution gate remains `DUNGEON-T09`.


## Aura System prefix ownership — 2026-09-28
- `AURA-T` -> `aura.md`.

Aura is mapping-in-progress. `AURA-T01` is deferred and has not been executed. Global first execution gate remains `DUNGEON-T09`.


## Messenger / Friend / Block prefix ownership — 2026-09-29
- `MSG-T` -> `messenger.md`.

Messenger is mapping-in-progress. `MSG-T01..MSG-T08` are canonical deferred tests and have not been executed. Global first execution gate remains `DUNGEON-T09`.
