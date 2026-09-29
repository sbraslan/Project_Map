# BP-T02 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-BPASS-001`  
**Canonical test:** `BP-T02`  
**Order:** Battle Pass primary normal-path observation after Dungeon Info, Ticket and Hunting normal-path gates.

> Handoff only. It does not authorize runtime execution.

## Static basis

Packet definition:
```cpp
typedef struct SPacketGCExtBattlePassMissionUpdate
{
    uint8_t bHeader;
    uint8_t bBattlePassType;
    uint8_t bMissionIndex;
    uint8_t bMissionType;
    uint32_t dwNewProgress;
} TPacketGCExtBattlePassMissionUpdate;
```

Current server creates this packet without zero-initialization and assigns:
- `bHeader`
- `bBattlePassType`
- `bMissionIndex`
- `dwNewProgress`

but does **not** assign `bMissionType`.

This omission exists in:
- existing-mission completion update;
- existing-mission non-complete progress update;
- newly-created mission update;
- `SetExtBattlePassMissionProgress` completion update;
- `SetExtBattlePassMissionProgress` non-complete update;
- newly-created mission path in the setter.

The client receives the byte verbatim and forwards:
```
BINARY_ExtBattlePassUpdate(
    bBattlePassType,
    bMissionIndex,
    bMissionType,
    dwNewProgress)
```

## Impact refinement

Fresh UI audit shows:
- `HaveMission(battlePassType, missionIndex, mission_type)` currently only compares `missionIndex`;
- therefore an invalid/uninitialized mission type does not necessarily prevent the progress row itself from updating;
- if the updated mission is currently selected, `UpdateMission` calls:
  `SetMissionInfo(missionIndex, mission_type)`;
- this can feed the undefined mission-type byte into mission-detail rendering/lookup.

Independently, the packet sends one byte of uninitialized server stack data over the network.

Therefore BP-T02 should capture both:
1. packet-level missionType mismatch/nondeterminism;
2. any selected-mission detail/UI mismatch, without assuming every progress update must disappear.

## Preconditions

1. Battle Pass system enabled and active pass loaded.
2. Use a normal mission with an ordinary gameplay producer.
3. Prefer a mission whose progress can be advanced repeatedly without modifying source/config.
4. Open/select that mission in Battle Pass UI before one update, then repeat with it unselected for comparison.
5. No packet injection is required.

## Future live action

When runtime is explicitly unlocked and prior gates are resolved:

1. Log in normally with an active Battle Pass.
2. Open Battle Pass and select a mission that can receive normal gameplay progress.
3. Trigger one legitimate progress update.
4. Capture the corresponding MISSION_UPDATE packet using existing safe packet/debug instrumentation.
5. Record:
   - battlePassType
   - missionIndex
   - received missionType
   - expected missionType from initial mission-info data
   - newProgress.
6. Repeat several legitimate updates.
7. Observe selected mission detail/title/info refresh behavior.
8. Repeat once with the same mission not selected.
9. Stop after evidence capture.

## Expected result

The packet's `bMissionType` is not guaranteed to equal the mission's real type because the server never initializes or assigns it.

Possible runtime observations:
- varying missionType values across updates;
- a stable but wrong value due to stack reuse;
- correct-looking value by coincidence;
- selected mission detail using the wrong mission type;
- progress itself still updating because HaveMission ignores mission_type.

A coincidentally correct byte in one packet does not disprove the static bug.

## Result classification

### REPRODUCED
Use when at least one normal update packet contains missionType that differs from the mission's known type, or instrumentation proves the byte is uninitialized at send time.

### NOT REPRODUCED
Use only if the deployed mapped-equivalent server explicitly initializes/assigns `bMissionType` before every mission-update send.

### INCONCLUSIVE
Use when packet fields cannot be observed, no mission update can be triggered normally, or deployment differs materially.

## Stop conditions

Stop if:
- observing packets would require modifying source;
- unrelated Battle Pass crash/state corruption occurs;
- deployment/source differs materially;
- next step would alter pass data manually.

Do not continue directly into BP-T01/BP-T03+.

## Evidence template

- Test: `BP-T02`
- Bug: `BUG-BPASS-001`
- Result:
- Date/time:
- Client/server build:
- Battle Pass type:
- Mission index:
- Expected mission type:
- Received mission type(s):
- New progress:
- Mission selected during update: yes/no
- UI/detail effect:
- Client errors:
- Server errors:
- Packet/debug evidence:
- Deployment drift:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
