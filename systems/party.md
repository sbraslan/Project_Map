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
