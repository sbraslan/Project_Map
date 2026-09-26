# Battle Field System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source repositories are read-only. This map records static architecture/findings only.

## Initial source roots
Server:
- `game/src/battle_field.h`
- `game/src/battle_field.cpp`
- `game/src/ranking_system.*` only where Battle Field ranking integration crosses subsystem boundaries
- `game/src/cmd_general.cpp` Battle Field commands
- `game/src/input_p2p.cpp` Battle Field command propagation
- `game/src/packet.h` Battle Field packets
- quest/event flags and DB tables used by Battle Field

Client:
- `root/uibattlefield.py`
- `root/interfacemodule.py::OpenRankingBoardWindow`
- `root/game.py` Battle Field command callbacks
- `UserInterface/PythonNetworkStreamPhaseGame.cpp::RecvBattleZoneInfo`
- player/network Battle Field bindings

## Known architecture from completed Ranking cross-audit
- Battle Field ranking cache is loaded by `CRankingSystem`.
- `CBattleField::OpenBattleUI` sends field state/timing and ranking data.
- ranking GC transport uses dynamic `HEADER_GC_BATTLE_ZONE_INFO`.
- client clears/rebuilds ranking cache then opens Battle Field UI.
- Battle Field UI has current/accumulated ranking tabs.

Ranking-specific defects remain canonical under `bugs/ranking.md`; they are not duplicated here unless a distinct Battle Field lifecycle bug is found.

## Battle Field state surface
Already observed server concepts:
- open/closed/disabled status via quest event flag;
- configured open/close schedule loaded from SQL;
- ranking update schedule loaded from SQL;
- enter/exit cooldown;
- event-mode score multiplier;
- temporary in-session Battle Field points;
- persistent `POINT_BATTLE_FIELD`;
- player kill/death handling;
- random respawn positions;
- weekly ranking rollover and winner affects;
- P2P broadcast commands for UI/event state.

## Exact next audit
1. Map entry/exit command authorization and map/channel restrictions.
2. Map kill/death/score accounting and persistence.
3. Audit schedule calculations for day/time boundary errors.
4. Audit cooldown and reconnect/disconnect behavior.
5. Audit P2P state synchronization across cores.
6. Start Battle Field-specific bug registry; do not duplicate Ranking bugs.
