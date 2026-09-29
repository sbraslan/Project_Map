# DUNGEON-T09 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Phase:** Detection / Mapping Only  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `DINFO::BUG-DUNGEON-009`  
**Canonical test:** `DUNGEON-T09`  
**Purpose:** First future live runtime gate after retraction of DUNGEON-T10.

> This file is a handoff/checklist only. It does not authorize execution. Source/game repositories remain read-only and no runtime action is permitted until the user explicitly changes phase.

## Static basis

Current `game/src/DungeonInfo.cpp::CDungeonInfoManager::Ranking` builds:

```cpp
"SELECT ... FROM `player`.`dungeon_ranking`"
"LEFT JOIN `player`.`player` ..."
```

Adjacent C++ string literals concatenate without adding whitespace, producing:

```
... `dungeon_ranking`LEFT JOIN ...
```

This is syntactically invalid SQL. The function returns when `uiSQLErrno` is set.

The current UI list construction was reverified on 2026-09-29 and is not blocked by the former BUG-DUNGEON-010 finding.

## Preconditions

1. Running client/server should correspond to the mapped snapshot, or deployment drift must be recorded.
2. Dungeon Info must be enabled.
3. Current dungeon config should load normally; tracked snapshot contains 9 entries.
4. Ranking DB/table path must be present in the running environment.
5. Do not patch the missing SQL whitespace before reproduction.
6. Use normal UI flow only; no packet crafting or Python injection.

## Exact future live action

When runtime execution is explicitly unlocked:

1. Start/login normally.
2. Open Dungeon Info from the normal minimap/UI button.
3. Select any visible dungeon row.
4. Open one of the ranking views: completed / fastest time / highest damage.
5. Capture client and GAME/server output immediately.
6. Stop after evidence capture; do not apply a fix in the same step.

## Expected evidence

Primary evidence:
- Dungeon Info list visibly populated;
- ranking request is made through normal UI;
- GAME/server log shows SQL syntax/query error for the ranking query, ideally exposing the `dungeon_ranking`LEFT JOIN` boundary;
- ranking window receives no valid ranking rows.

Useful supporting evidence:
- screenshot of selected dungeon + ranking window;
- client syserr;
- server/game syserr;
- exact deployed build/commit identifier if it differs from mapped snapshot.

## Result classification

### REPRODUCED
Use when the normal ranking request reaches the server and the malformed SQL fails.

### NOT REPRODUCED
Use when the mapped-equivalent build successfully executes ranking SQL and returns valid rows. Before retracting the bug, compare deployed source for an already-applied whitespace fix or other query change.

### INCONCLUSIVE
Use when:
- Dungeon Info does not load;
- ranking UI cannot be opened for an unrelated reason;
- DB schema/table is absent for deployment reasons;
- another failure occurs before the query;
- deployment differs materially from the mapped snapshot.

## Stop conditions

Stop after first clean evidence capture, or immediately if:
- client crashes/disconnects;
- server/core errors affect unrelated state;
- deployment drift prevents comparison;
- continuing would require editing source/config/DB.

## Downstream normal-path order

After T09 is resolved:
1. `DUNGEON-T11`
2. `DUNGEON-T12`

`DUNGEON-T10` is retracted and must not be run.

## Evidence record template

- Test: `DUNGEON-T09`
- Bug: `DINFO::BUG-DUNGEON-009`
- Result: `REPRODUCED | NOT REPRODUCED | INCONCLUSIVE`
- Date/time:
- Client build/snapshot:
- Server build/snapshot:
- Dungeon config entry count:
- Selected dungeon/index:
- Ranking type:
- Client errors:
- Server/GAME SQL error:
- Evidence artifact/reference:
- Deployment drift:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**

No runtime test or source/config/database modification was performed while preparing this handoff.
