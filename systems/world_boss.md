# World Boss System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source/game repositories are read-only. Only Project_Map is writable.

## Initial client roots
- `root/uiworldboss.py`
- `root/uiworldbossranking.py`
- `root/uiscript/worldbosswindow.py`
- `root/uiscript/worldbossrankingwindow.py`
- `root/interfacemodule.py` World Boss window/ranking bridge
- `root/game.py` World Boss callbacks

## Initial server/client-C++ direction
Dedicated World Boss filenames were not found in the server tree. Continue by exact symbol/feature-flag tracing:
- `ENABLE_WORLD_BOSS`
- packet headers / packet structs
- game-server input/character/manager handlers
- DB/ranking storage paths
- spawn/despawn lifecycle
- reward/damage/ranking lifecycle
- client receive callbacks and UI refresh.

## Current status
Only initial roots are established. No World Boss bug is verified yet.

## Exact next work
1. Trace `ENABLE_WORLD_BOSS` and all World Boss packet symbols through ClientSrc and ServerSRC.
2. Map server spawn/state/reward/ranking ownership.
3. Map Python UI callbacks and ranking cache/reset behavior.
4. Record verified findings only.
