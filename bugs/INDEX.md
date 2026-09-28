# Bug registries

Bug records are split by subsystem.

Normal rule: read only `<active-system>.md` from this folder when bug analysis is needed.

Global status/navigation: `../INDEX.md`.


## Global integrity rules — 2026-09-28
See `../READINESS_AUDIT.md`.

Important global ID rules:
- Dungeon Info and Dungeon Core both historically use `BUG-DUNGEON-001..004`. Global references must use `DINFO::BUG-DUNGEON-xxx` or `DCORE::BUG-DUNGEON-xxx`; historical local IDs are not renumbered.
- `BUG-EXCHANGE-004` requires a descriptive qualifier because two verified findings share that historical ID.
- `BUG-RANK-006` and `BUG-BFIELD-004` are RETRACTED / RESERVED.
- historical `BUG-BFIELD-005` and `BUG-BFIELD-007` are owned canonically by `BUG-RANK-003` and `BUG-RANK-007`.
- `BUG-ITEM-005` is intentionally absent.
- Candidate, observation and dormant labels must never be promoted by documentation alone.


## Post-audit subsystem ownership — 2026-09-28
- `BUG-LOOK-001..007` -> `costume_appearance.md`
- Mount-expiry helper gaps, Aura overlap, omitted GuildStorage/Roulette/Switchbot guards and the conditional free-ticket alias remain unpromoted observations/candidates.


## Dragon Soul ownership — 2026-09-28
- `BUG-DS-001..003` -> `dragon_soul.md`
- Step-refine equipped-first, malformed grade bound and change-attr step bound remain candidate/unpromoted.
