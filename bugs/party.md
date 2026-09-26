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
