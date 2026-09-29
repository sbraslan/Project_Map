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
- `BUG-DS-001..011` -> `dragon_soul.md`
- Malformed grade bound and change-attr step bound remain candidate/unpromoted.
- Step-refine equipped-first validation is `BUG-DS-008`; relog set wrap, pull-out extractor lifetime, and zero-ID daily-gift authorization are `BUG-DS-009..011`.


## Active subsystem ownership — Acce / Sash — 2026-09-28
- `BUG-ACCE-001..008` -> `acce.md`
- Status: STATIC COMPLETE / VERIFIED STATIC.
- Runtime ownership draft: `../tests/acce.md` (`ACCE-T01..T08`), execution locked.


## Dragon Soul closure — 2026-09-28
- Status: STATIC COMPLETE / VERIFIED STATIC for Dragon Soul.
- `BUG-DS-001..011` -> `dragon_soul.md`.
- Malformed grade/step boundaries remain unpromoted data-dependent candidates.


## Active subsystem ownership — Aura System — 2026-09-28
- `BUG-AURA-001` -> `aura.md`.
- Status: STATIC MAPPING IN PROGRESS.
- First verified finding: post-open Aura transaction distance gate bypass.


## Messenger / Friend / Block ownership — 2026-09-29
- `BUG-MSG-001..021` -> `messenger.md`.
- Status: STATIC COMPLETE / VERIFIED STATIC.
- Deferred runtime ownership: `../tests/messenger.md` (`MSG-T01..MSG-T21`); execution locked.
