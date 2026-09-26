# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Party System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/party.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Party checkpoint
Mapped and closed:
- create/join/remove/delete + DB replication;
- CG/GC client bridge;
- invite/accept authority;
- reconnect/offline state;
- role counters;
- dynamic minimap position parser;
- near-member/role-bonus periodic update;
- actual kill EXP distribution path;
- channel-scoped DB setup/rebuild;
- mutating quest party APIs.

Verified bugs:
- `BUG-PARTY-001` — leader `Quit()` use-after-free.
- `BUG-PARTY-002` — Party Heal hard-disabled.
- `BUG-PARTY-003` — mismatched role-off corrupts role counters.
- `BUG-PARTY-004` — malformed dynamic party-position size can underflow parser.
- `BUG-PARTY-005` — leader Quit can preserve party role combat bonuses after party destruction.
- `BUG-PARTY-006` — quest `get_near_member_pids` lacks any near/range check.

## Important closure notes
- party state is intentionally channel-scoped; DB setup rebuilds only the peer's channel parties;
- kill EXP distribution applies its own same-map + 5000 range filters;
- `ComputePoints()` preserves party role bonus points, confirming BUG-PARTY-005 can survive ordinary point recomputation;
- `party.leave_party` adds a second normal reachability path to the leader Quit defects.

## Exact next work
1. Separate/audit Party Match.
2. Audit remaining quest Party APIs.
3. Audit party item ownership/drop rotation and EXP-centralize pointer lifecycle.
4. Check remaining packet field/size consistency.
5. Decide core Party STATIC COMPLETE.

GitHub state is canonical.
