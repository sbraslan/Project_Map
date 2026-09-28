# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Party runtime-readiness is consolidated.
- `tests/party.md` did not previously exist; a canonical PARTY-T01..PARTY-T06 deferred matrix was created.
- PARTY-T01..T06 cover BUG-PARTY-001..006 one-to-one.
- PARTY-T01, PARTY-T02 and PARTY-T05 are the primary normal-path Party candidates.
- PARTY-T03 is modified-client/state-integrity; PARTY-T04 is malformed-server-packet parser robustness; PARTY-T06 is trusted/dev quest validation.
- Party Match remains a separate subsystem.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Party Match runtime readiness from `bugs/party_match.md` + `tests/party_match.md`.
2. Keep Party and Party Match ownership separate.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
