# Party System — Bug Registry

**Phase:** Detection / Mapping Only

### BUG-PARTY-001 — leader Quit path uses a deleted CParty object
- Statik durum: **doğrulandı**
- Sınıf: C++ use-after-free / lifecycle

`CParty::Quit(dwPID)` calls `P2PQuit(dwPID)` first.

Inside `P2PQuit`, if the removed member is the leader:
`CPartyManager::Instance().DeleteParty(this)`
is executed.

`DeleteParty` removes the party from the manager and `M2_DELETE`s it. `P2PQuit` explicitly contains the warning that no code may use `this` after that delete.

Nevertheless, execution returns to `Quit`, which reads `m_bPCParty` and calls `GetLeaderPID()` on the destroyed object.

Normal reachability:
`CBattleField::Connect`
-> existing `pChar->GetParty()`
-> `party->Quit(pChar->GetPlayerID())`.

If the BattleField-entering character is the party leader, this reaches the UAF path.

No source modification is authorized in the current phase.

### BUG-PARTY-002 — Party Heal is a deterministic dead feature
- Statik durum: **doğrulandı**
- Sınıf: hard-disabled feature / client-server integration

The client UI has:
- a Party Heal button;
- `OnPartyUseSkill`;
- `SendPartyUseSkillPacket(PARTY_SKILL_HEAL, 0)`;
- a `PartyHealReady` command handler that reveals the heal button.

The server computes party-heal readiness, but delivery is disabled by:
`if (0) // XXX DELETEME until client completes`.

The actual server execution function `CParty::HealParty()` also begins with an unconditional `return` block carrying the same unfinished-client comment.

Thus:
- the normal server never tells the client that heal is ready;
- even a manually sent heal request reaches a no-op function;
- party heal cannot restore HP/SP in the current source snapshot.

## Audited non-bugs / guarded paths
- Party invite acceptance requires a live server-side invite event and revalidates mutable join conditions.
- Role changes require leader authority, member membership, and a whitelist of role values.
- Party warp resolves the client VID but `SummonToLeader` verifies the resulting PID is a party member.
- EXP distribution mode is bounds-checked in `CParty::SetParameter`.
- Normal GC party removal clears the C++ player party cache through the Python UI callback.


### BUG-PARTY-003 — mismatched role-off packet corrupts role counters
- Statik durum: **doğrulandı**
- Sınıf: server authority / state-counter corruption

`PartySetState` restricts callers to the party leader and whitelists special role IDs, but it does not require the role supplied for a remove operation to match the member's current role.

For `flag=false`, `CParty::SetRole`:
- checks only that the member's current role is neither LEADER nor NORMAL;
- changes the member to NORMAL;
- decrements `m_anRoleCount[bRole]` using the packet-supplied role.

A leader can therefore remove an ATTACKER state while supplying DEFENDER as the role ID, for example. The member becomes NORMAL, ATTACKER remains falsely counted as occupied, and DEFENDER can be decremented below zero.

Role assignment limits subsequently rely on these counters, so state/capacity becomes inconsistent.

The stock Python UI normally sends the member's current role; the defect is missing server enforcement of that assumption.

### BUG-PARTY-004 — malformed party-position dynamic packet can underflow parser length
- Statik durum: **doğrulandı**
- Sınıf: client packet parser / dynamic-size boundary validation

`HEADER_GC_PARTY_POSITION_INFO` is dynamic.

`RecvPartyPositionInfo()` computes the payload length from `Packet.wSize - sizeof(Packet)` and loops by subtracting `sizeof(SPartyPosition)`, but never validates:
- `wSize >= sizeof(TPacketGCPartyPosition)`;
- payload size is an exact multiple of `sizeof(SPartyPosition)`.

A too-small or non-aligned declared size can underflow/wrap the loop counter and lead to reads beyond the packet's declared boundary / receive-stream desynchronization.

The normal server sender builds well-formed packets from whole position records. This bug is recorded as malformed-server-packet robustness, not a client-to-server exploit.


### BUG-PARTY-005 — leader Quit can preserve party role bonuses after party destruction
- Statik durum: **doğrulandı**
- Sınıf: stale combat stats / lifecycle
- Build koşulu: `ENABLE_PASSIVE_ATTR` aktif

Leader `P2PQuit` flow:
1. leader entry is erased from `m_memberMap`;
2. cleanup calls `ComputeRolePoint(ch, GetLeaderCharacter(), role, false)`;
3. `GetLeaderCharacter()` uses `m_memberMap[leaderPID]`, recreating an empty leader entry;
4. the returned leader pointer is null;
5. passive-attr `ComputeRolePoint` immediately returns when `pkLeader == nullptr`.

Then leader removal triggers `DeleteParty(this)`.

During destructor `RemoveBonus()`, remaining party members also pass the same null leader pointer to `ComputeRolePoint`, so their party bonus cleanup can be skipped as well.

The stale points include the `POINT_PARTY_*_BONUS` family. They are not automatically erased by `ComputePoints()`; that function explicitly snapshots and restores the party bonus values.

Verified normal reachability:
- BattleField leader entry;
- active quest API `party.leave_party` when a leader leaves a party with more than two members.

This is distinct from BUG-PARTY-001: BUG-PARTY-001 is the freed-`this` access after return from `P2PQuit`; BUG-PARTY-005 is stale combat-stat state created during the self-delete path itself.

### BUG-PARTY-006 — `party.get_near_member_pids` returns same-map members without near filtering
- Statik durum: **doğrulandı**
- Sınıf: quest Lua API semantic defect
- Build koşulu: `ENABLE_DUNGEON_RENEWAL` aktif

The registered API is named `get_near_member_pids`, but its implementation only calls:
`ForEachOnMapMember(..., currentMapIndex)`.

There is no range check and no `IsNearLeader/bNear` check.

The implementation itself carries:
`// Near Check missing!`

As a result, same-map party members farther than the normal 5000 party range are returned as "near".

No concrete caller was found in the current Project_Game search, so current gameplay impact depends on quest usage; the exposed Lua API implementation itself is nevertheless incorrect.


## Static closure
Core Party static audit closed on 2026-09-26.

Verified bugs:
- BUG-PARTY-001
- BUG-PARTY-002
- BUG-PARTY-003
- BUG-PARTY-004
- BUG-PARTY-005
- BUG-PARTY-006

No additional bug was promoted from:
- item ownership/drop rotation;
- dormant EXP-centralize storage;
- remaining packet field/layout comparison;
- remaining quest helpers without a verified normal gameplay failure path.

Party Match continues independently under `systems/party_match.md`.


## Additional reachability — Devil Catacomb item-group flow
The active Devil Catacomb quest calls `d.exit_all_by_item_group("reapers_credit")`.

Its server functor removes a party member without the required item by calling `pParty->Quit(pid)` when party size is greater than 2.

If the affected member is the party leader, this reaches the already verified BUG-PARTY-001 leader self-delete/use-after-free path.

This is additional normal gameplay reachability evidence for BUG-PARTY-001; no new bug ID is created.
