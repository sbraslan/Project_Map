# Party System — Deferred Runtime Tests

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only
**Created:** 2026-09-28

Do not execute runtime, sanitizer, malformed-packet, or fault-injection tests unless the user explicitly changes project phase.

### PARTY-T01 — leader Quit lifecycle / UAF
Use a disposable party with more than two members and exercise a normal gameplay path that makes the leader leave through `CParty::Quit` (BattleField entry is the primary path; Devil Catacomb item-group removal is a second normal path).

Observe under debug/ASan:
- party destruction,
- return from `P2PQuit`,
- any access to the deleted `CParty`,
- process stability.

Covers BUG-PARTY-001.

### PARTY-T02 — Party Heal end-to-end
Create a valid party/leadership state where heal should become ready, then observe:
- whether the normal client receives PartyHealReady,
- whether the heal control becomes available,
- whether a legitimate or manually issued PARTY_SKILL_HEAL changes member HP/SP.

Expected current static behavior: readiness notification is disabled and `HealParty()` returns without healing.

Covers BUG-PARTY-002.

### PARTY-T03 — mismatched role-off counter integrity
Isolated modified-client test with a leader:
1. assign a member one special role;
2. send role removal with a different whitelisted role ID;
3. inspect actual member role, internal role counters, and subsequent role-cap behavior.

Covers BUG-PARTY-003.

### PARTY-T04 — malformed party-position dynamic packet
Isolated/debug client only. Feed malformed `HEADER_GC_PARTY_POSITION_INFO` packets with:
- declared size smaller than the base packet;
- payload size not divisible by `sizeof(SPartyPosition)`.

Observe parser boundary handling and receive-stream synchronization under ASan/debug.

Covers BUG-PARTY-004. This is malformed-server-packet robustness testing, not a client-to-server exploit test.

### PARTY-T05 — stale role bonuses after leader Quit
Give party members active role bonuses, then trigger a normal leader-quit/destruction path with more than two members.

After party destruction inspect:
- `POINT_PARTY_*_BONUS` values on the former leader and remaining members;
- recomputation behavior;
- persistence across ordinary stat recalculation.

Covers BUG-PARTY-005.

### PARTY-T06 — near-member Lua semantic check
Temporary isolated dev quest only:
- form a party with members on the same map;
- keep at least one member farther than PARTY_DEFAULT_RANGE;
- call `party.get_near_member_pids`;
- compare returned PIDs with actual near/far positions.

Covers BUG-PARTY-006.

## Readiness consolidation — 2026-09-28
- No Party runtime test was executed.
- PARTY-T01..PARTY-T06 cover BUG-PARTY-001..006 one-to-one.
- Normal-path candidates: PARTY-T01, PARTY-T02, PARTY-T05.
- Modified-client/state-integrity: PARTY-T03.
- Isolated client parser robustness: PARTY-T04.
- Trusted/dev quest semantic validation: PARTY-T06.
- BUG-PARTY-001 and BUG-PARTY-005 intentionally share the same leader-quit family but validate different failure effects.
- Party Match is excluded from this file and remains a separate subsystem.
- Overall first live runtime gate remains DUNGEON-T09.
