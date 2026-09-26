# Dungeon Info — Fix Plan

**Status:** PATCH PLAN READY — source repos unchanged
**Date:** 2026-09-26

This file records the minimal fixes for the first runtime cluster. Source repositories remain read-only until live reproduction/explicit fix execution.

## FIX-DUNGEON-010 — restore normal dungeon-list creation
Source: `Project_Binary/root/uidungeoninfo.py::DungeonInfoWindow.Initialize`.

Current defect: the `for key in xrange(...)` list-construction loop is indented under the `GetCount() == 0` `else` branch.

Minimal correction:
1. keep the empty-data popup only in the zero-count branch;
2. return after showing the zero-data popup, or otherwise keep list construction outside that branch;
3. run the list-construction loop only when `GetCount() > 0`.

Target control flow:
```python
count = dungeonInfo.GetCount()
if count <= 0:
	self.OnLockButtons(True)
	# show DUNGEON_INFO_NOT_FOUND popup
	return

self.OnLockButtons(False)
for key in xrange(min(dungeonInfo.MAX_DUNGEON_SCROLL, count)):
	# create ListToggleButton and append
```

Expected effect: with the current 9-entry server config, the first visible list page is populated and T09/T11/T12 become reachable through normal UI.

## FIX-DUNGEON-009 — repair ranking SQL literal boundary
Source: `Project_ServerSRC/game/src/DungeonInfo.cpp::Ranking`.

Minimal correction: add whitespace between the table identifier and `LEFT JOIN`.

Current:
```cpp
"... FROM `player`.`dungeon_ranking`"
"LEFT JOIN `player`.`player` ..."
```

Target:
```cpp
"... FROM `player`.`dungeon_ranking` "
"LEFT JOIN `player`.`player` ..."
```

Do not combine this fix with the separate unchecked `byIndex` bug; BUG-DUNGEON-001 should be fixed independently with an index guard.

## FIX-DUNGEON-011 — honor documented numeric quest-flag type
Source: `Project_ServerSRC/game/src/DungeonInfo.cpp::LoadFile`.

Config contract in `dungeon_info.txt`:
- `0` = PC quest flag
- `1` = GLOBAL event flag.

Current parser recognizes only literal `GLOBAL` and therefore misclassifies numeric `1` as PC.

Compatibility-safe target logic:
```cpp
if (!strcmp(szValue3, "GLOBAL") || iValue3 == 1)
	sDungeonQuest.byType = QUEST_FLAG_GLOBAL;
else
	sDungeonQuest.byType = QUEST_FLAG_PC;
```

This preserves old literal-GLOBAL configs while making the checked-in numeric format work.

## FIX-DUNGEON-012A — prevent unsigned cooldown underflow
Source: `Project_ServerSRC/game/src/DungeonInfo.cpp::SendInfo`.

Current code subtracts `get_global_time()` into `uint32_t`; expired timestamps wrap to a huge positive value.

Use signed/time_t arithmetic and clamp before conversion:
```cpp
const time_t now = get_global_time();
time_t remain = 0;

if (dwFlagValue > now)
	remain = dwFlagValue - now;
else if (dwFlagValue > 0 && pSDungeonData->dwCooldown > 0)
{
	const time_t endTime = dwFlagValue + pSDungeonData->dwCooldown;
	if (endTime > now)
		remain = endTime - now;
}

packet.dwCooldown = remain > 0 ? static_cast<uint32_t>(remain) : 0;
```

This fixes the safety/correctness bug even when config data is incomplete.

## FIX-DUNGEON-012B — supply intended cooldown semantics
The checked-in QUEST-backed dungeon entries currently have no `COOLDOWN` lines.

Observed quest semantics:
- Flame Dungeon `exit_time`: stores `get_global_time()` on logout; rejoin logic uses 5 minutes in one path and `ENTER_LIMIT_TIME` 30 minutes for fresh-entry gating.
- Snow Dungeon `exit_time`: stores `get_global_time()` on logout; rejoin logic uses 5 minutes, while fresh-entry gate uses 240 minutes.
- Dragon Lair global `dragon_lair_time`: stores start time; live quest cooldown is 1200 seconds, test-server cooldown is 900 seconds.

Therefore a single generic `COOLDOWN` value does not automatically represent every quest rule. Before adding config `COOLDOWN` lines, choose which gameplay timer the Dungeon Info UI is intended to display (rejoin window, fresh-entry lockout, or dungeon-global cooldown).

Recommended first fix sequence:
1. apply 012A to eliminate wrap/garbage immediately;
2. validate the desired UI meaning per dungeon;
3. then add/derive correct cooldown duration data instead of guessing.

## Runtime order after patch readiness
1. live reproduce T10 on current build;
2. apply FIX-DUNGEON-010;
3. verify 9 rows appear;
4. live reproduce T09, then apply FIX-DUNGEON-009;
5. verify T11 with map 208/global dragon flag, then apply FIX-DUNGEON-011;
6. reproduce T12 with zero/expired flag, then apply FIX-DUNGEON-012A;
7. decide per-dungeon cooldown semantics before any 012B data changes.

## Scope rule
No source repository was modified while producing this plan.
