# Party System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-26

> Source repositories are strictly read-only. This file is the canonical static map for the active Party subsystem.

## Initial scope
Core party lifecycle, DB/P2P replication, client packet bridge, and the normal in-game party UI.

Party Match is adjacent but will be separated if its packet/data lifecycle is independent.

## Confirmed source roots

### Server game
- `game/src/party.h`
- `game/src/party.cpp`
- `game/src/input_main.cpp` — CG party handlers
- `game/src/packet.h`
- `game/src/questlua_party.cpp`
- character party linkage in `char.h/.cpp`

### Server DB
- `db/src/ClientManagerParty.cpp`

### Client C++
- `UserInterface/PythonNetworkStreamPhaseGame.cpp`
- `UserInterface/PythonNetworkStreamModule.cpp`
- `UserInterface/PythonPlayer.cpp`
- `UserInterface/PythonPlayer.h`
- `UserInterface/Packet.h`

### Client Python/UI
- `root/uiparty.py`
- `root/interfacemodule.py`
- `root/game.py`

## Initial server model
`CPartyManager` maintains:
- PID -> party mapping for PC parties;
- separate mob-party mapping;
- a set of all PC parties;
- an enable gate for PC-party changes.

Mapped manager operations include:
- `CreateParty`
- `DeleteParty`
- `SetParty`
- `SetPartyMember`
- `P2PCreateParty`
- `P2PDeleteParty`
- `P2PJoinParty`
- `P2PQuitParty`
- P2P login/logout linkage.

`CParty` supports up to `PARTY_MAX_MEMBER = 8` and declares leader/normal plus attacker, tanker, buffer, skill-master, haste and defender roles.

## Initial DB replication model
`db/src/ClientManagerParty.cpp` keeps party state per game channel through `m_map_pkChannelParty[peer->GetChannel()]`.

Mapped DB operations:
- create -> `HEADER_DG_PARTY_CREATE`;
- delete -> `HEADER_DG_PARTY_DELETE`;
- add -> `HEADER_DG_PARTY_ADD`;
- remove -> `HEADER_DG_PARTY_REMOVE`;
- state change -> `HEADER_DG_PARTY_STATE_CHANGE`;
- member level -> `HEADER_DG_PARTY_SET_MEMBER_LEVEL`.

The DB manager forwards these updates to game peers on the relevant channel.

## Initial game request surface
`CInputMain` contains live handlers for:
- party invite;
- invite answer;
- member state/role change;
- member removal / leave;
- party skill use;
- party parameter / EXP-distribution changes.

The exact CG/GC packet chain and validation boundaries are the next mapping target.

## Initial UI surface
`root/uiparty.py` exposes:
- member boards;
- role/state controls;
- warp/heal party skill controls;
- kick/leave/disband controls;
- EXP distribution controls;
- optional minimap party information.

Python UI sends party actions through the client network module, including state changes, skill use, member removal and party exit.

## Exact next audit
1. Map Create / Join / Leave / Delete lifecycle across game -> DB -> game peers.
2. Map every CG party packet and server-side authority/validation check.
3. Map every GC party packet into client C++ caches and `uiparty.py`.
4. Audit leader-only operations, role bounds, PID/VID trust boundaries and cross-channel behavior.
5. Separate Party Match if its lifecycle is independent.


## Core lifecycle mapping — create/join/remove/delete

### Create
`CPartyManager::CreateParty(leader)`:
1. returns the existing party if the leader already has one;
2. allocates a `CParty`;
3. for PC leaders sends `HEADER_GD_PARTY_CREATE` to DB;
4. marks the party as PC;
5. locally `Join(leaderPID)`;
6. inserts into the manager PC-party set;
7. links the live leader character.

`Join(pid)` first applies local `P2PJoin(pid)`, then sends `HEADER_GD_PARTY_ADD` to DB for PC parties.

### DB/channel replication
DB tracks parties in `m_map_pkChannelParty[peer->GetChannel()]`.

- GD CREATE -> DB creates channel party -> DG CREATE to other peers on same channel.
- GD ADD -> DB inserts member -> DG ADD to other peers on same channel.
- GD REMOVE -> DB removes member -> DG REMOVE to other peers on same channel.
- GD DELETE -> DB erases party -> DG DELETE to other peers on same channel.
- GD STATE_CHANGE -> DB validates party/member, updates role, forwards DG STATE_CHANGE.
- GD SET_MEMBER_LEVEL -> DB validates and forwards DG SET_MEMBER_LEVEL.

Game `CInputDB` consumes these through:
- `P2PCreateParty`
- `P2PJoinParty`
- `P2PQuitParty`
- `P2PDeleteParty`
- `SetRole`
- `P2PSetMemberLevel`.

### Destroy
`CParty::~CParty -> Destroy()`:
- removes all member PID -> party mappings through `CPartyManager::SetPartyMember(pid, nullptr)`;
- cancels party update/minimap events;
- removes party bonuses;
- sends GC remove to connected local members;
- clears each character's party pointer;
- clears member map and dungeon/zodiac ownership links.

## Client packet bridge
Mapped GC receive handlers:
- `HEADER_GC_PARTY_INVITE -> RecvPartyInvite`
- `HEADER_GC_PARTY_ADD -> RecvPartyAdd`
- `HEADER_GC_PARTY_UPDATE -> RecvPartyUpdate`
- `HEADER_GC_PARTY_REMOVE -> RecvPartyRemove`
- `HEADER_GC_PARTY_LINK -> RecvPartyLink`
- `HEADER_GC_PARTY_UNLINK -> RecvPartyUnlink`
- `HEADER_GC_PARTY_PARAMETER -> RecvPartyParameter`
- optional `HEADER_GC_PARTY_POSITION_INFO -> RecvPartyPositionInfo`.

Client C++ caches party members in `CPythonPlayer::m_PartyMemberMap`.

Normal remove flow is complete:
`RecvPartyRemove`
-> Python `RemovePartyMember(pid)`
-> `uiparty.py::RemovePartyMember`
-> `playerm2g2.RemovePartyMember(pid)`
-> `CPythonPlayer::RemovePartyMember`.

If the removed PID is the local character, Python calls `playerm2g2.ExitParty()`, which clears the entire C++ party cache.

## Invite/accept authority
Two invitation/request mechanisms exist, but both use server-side pending-event state.

Packet invite path:
- inviter stores invitee PID in `m_PartyInviteEventMap`;
- accept checks that an invitation event actually exists;
- accept cancels/removes that event;
- accept rechecks mutable party conditions before joining: server enabled, dungeon restriction, observer, level boundary, already joined, party full;
- if inviter is already in a party, inviter must still be its leader.

The invite answer therefore does not trust the client-provided leader VID alone.

## Role/remove/skill authority
`PartySetState`:
- requires an active party;
- caller must be leader;
- target PID must be a member;
- role is switch-whitelisted before `SetRole`.

`PartyRemove`:
- leader may remove party members or disband;
- non-leader may only remove itself;
- dungeon/Zodiac/Meley restrictions are checked before the operation.

`PartyUseSkill`:
- caller must be party leader;
- warp target VID is resolved to a live character, then `SummonToLeader(pid)` verifies the PID belongs to the party before summoning.

`PartyParameter`:
- caller must be leader;
- `CParty::SetParameter` rejects modes >= `PARTY_EXP_DISTRIBUTION_MAX_NUM`.

## Verified bugs

### BUG-PARTY-001 — leader Quit path uses CParty after it deletes itself
`CParty::Quit(pid)` first calls `P2PQuit(pid)`.

When the removed PID has `PARTY_ROLE_LEADER`, `P2PQuit` calls:
`CPartyManager::DeleteParty(this)`.

`DeleteParty` sends DB delete, removes the party from the manager set and `M2_DELETE`s the party. The source itself explicitly warns that no code may use `this` after that call.

However control returns to `CParty::Quit`, which then evaluates:
`m_bPCParty && dwPID != GetLeaderPID()`.

That dereferences the destroyed party object and is a use-after-free / undefined-behavior path.

Normal reachability is verified: `CBattleField::Connect` removes any party member by calling `party->Quit(pChar->GetPlayerID())`; if the entering character is the party leader, the leader self-delete path is taken.

### BUG-PARTY-002 — Party Heal is exposed in the client but hard-disabled on the server
The Python UI implements a heal button and sends `PARTY_SKILL_HEAL` through `SendPartyUseSkillPacket`.

The server also computes heal readiness from Leadership/cooldown state, but the ready notification is wrapped in:
`if (0) // XXX DELETEME until client completes`.

Even if a heal packet is sent manually or through a visible stale button, `CParty::HealParty()` begins with an unconditional block that immediately returns:
`// XXX DELETEME until client completes { return; }`.

Therefore the mapped Party Heal feature cannot become usable through its normal UI/server lifecycle and the heal operation can never execute.

## Findings not promoted
- `RecvPartyRemove` does not directly erase the C++ cache, but the normal Python remove callback does erase it through `playerm2g2.RemovePartyMember` / `ExitParty`; this is not a bug on the mapped normal path.
- invite/accept mutable conditions are revalidated at accept time; no forged-invite acceptance bug was found in the mapped path.

## Exact next audit
1. Audit reconnect/offline member ADD/LINK/UNLINK behavior and duplicate PID cache handling.
2. Audit `CParty::SetRole` internal bounds on DB/P2P call paths.
3. Map EXP/bonus/near-member update lifecycle.
4. Audit Party Match separately and decide whether it belongs to Party or its own subsystem.
5. Continue client packet/state consistency audit.
