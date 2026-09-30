# Runtime Validation Queue

**Static mapping:** COMPLETE  
**Execution:** LOCKED / NOT RUN  
**Purpose:** single canonical live-test order after the pinned static mapping snapshot.

## Gate 1 — Dungeon Info normal-path validation

### 1. DUNGEON-T09 — Ranking SQL whitespace failure
Trigger:
1. Log in normally.
2. Open Dungeon Info.
3. Select a valid dungeon.
4. Open ranking / request a ranking type.

Capture:
- GAME SQL error log.
- Whether ranking rows are returned.

Expected current-snapshot defect:
- SQL contains missing whitespace before `LEFT JOIN`.
- Ranking request fails and returns no valid rows.

Stop rule:
- Record `REPRODUCED | NOT_REPRODUCED | INCONCLUSIVE`.
- Do not modify source before evidence is captured.

### 2. DUNGEON-T11 — Global-vs-PC cooldown flag mismatch
After T09 evidence is recorded:
- use Blue Dragon / map 208;
- compare the actual global `dragon_lair_time` event flag with Dungeon Info displayed cooldown;
- verify whether the UI reads per-character `dragon_lair_access.dragon_lair_time` instead.

### 3. DUNGEON-T12 — Expired cooldown uint32 wrap
After T11:
- use a QUEST-backed dungeon with unset/expired flag;
- open Dungeon Info;
- check for an abnormally huge cooldown instead of zero/available.

## Gate 2 — Critical Auto Hunt trust-boundary validation

### 4. AUTO-003 — MOVE anti-cheat bypass
Run only in isolated/dev environment.
- activate valid Auto Hunt;
- inject crafted movement beyond normal teleport/speed bounds;
- verify whether server accepts movement that normal state rejects.

### 5. AUTO-004 — SyncPosition displacement bypass
Run only in isolated/dev environment.
- establish legitimate sync ownership over a nearby attackable target;
- submit a large sync delta;
- verify whether target position is accepted while Auto Hunt is active.

## Gate 3 — High-impact follow-up
After the two critical Auto Hunt tests:
- AUTO-001 premium-expiry persistence;
- AUTO-002 restart entitlement/special-map revive rules;
- Party Match exchange-lifetime defect;
- Messenger long-name / cross-core block/presence defects;
- remaining High findings by subsystem.

## Operating rule
One test at a time:
1. reproduce,
2. save evidence,
3. classify result,
4. only then prepare a fix,
5. retest the exact same case,
6. mark resolved only after regression passes.

No static CLOSED subsystem is reopened unless runtime evidence contradicts the pinned map or the source snapshot changes.
