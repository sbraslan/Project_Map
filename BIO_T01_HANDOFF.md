# BIO-T01 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-BIO-001`  
**Canonical test:** `BIO-T01`

> Handoff only. It does not authorize runtime execution.

## Static basis

Current C++ Biolog flow:
- successful submissions increment `biolog_collected`;
- when collected reaches required count:
  - sub-mission path sets `biolog_manager.can_do_sub_mission_<mission> = 1`;
  - Seon-Pyeong reward-selection path sets `biolog_manager.can_select_reward_<mission> = 1`;
- UI refreshes into the completion/sub-mission state.

Current client completion button:
```python
m2netm2g.SendRequestEventQuest("biolog_manager")
```

Current server exposes the required Lua helpers:
- `pc.biolog_set_mission`
- `pc.biolog_get/set_cooldown`
- `pc.biolog_get_mission_item`
- `pc.biolog_get_sub_mission_item`
- `pc.biolog_get_reward_item`
- `pc.biolog_set_reward_bonus`

But the current tracked Project_Game quest package contains no registered quest/object named `biolog_manager`.
The current `quest_list` registers only the legacy biolog quests:
`collect_herb_lv4` and `collect_quest_lv30...94`.

A fresh repository search also returns no `biolog_manager` quest implementation.

Therefore the new C++ manager can reach its completion flags and UI button, but the event-quest bridge requested by the UI has no mapped current quest consumer to:
- handle sub-mission;
- grant/choose reward;
- call reward bonus helper;
- advance the Biolog mission;
- reset/update the UI state.

## Preconditions

1. Biolog system enabled.
2. Running source/game package should correspond to the mapped snapshot or drift must be recorded.
3. Actual DB `biolog_missions` / `biolog_rewards` rows must load successfully.
4. Use a disposable character on a mission that can be completed normally.
5. Prefer a mission whose completion path exposes either sub-mission or Seon-Pyeong reward selection.
6. No quest/source edits before evidence capture.

## Future live action

When runtime is explicitly unlocked and earlier gates are resolved:

1. Open the Biolog UI normally.
2. Complete the current mission's required collection through normal submissions.
3. Confirm the client shows the completion/sub-mission button.
4. Record the relevant generated quest flag if safely observable.
5. Click the completion/sub-mission button normally.
6. Observe the event-quest request/result.
7. Check whether:
   - a quest dialog starts;
   - reward/sub-mission is processed;
   - mission advances;
   - reward bonus/item is applied;
   - UI refreshes to the next mission.
8. Capture client/GAME quest errors or no-op behavior.
9. Stop; do not install/fix a quest in the same evidence step.

## Expected result

Current tracked deployment model predicts:
- C++ collection reaches required count;
- the `biolog_manager.*` completion flag is set;
- client requests event quest `biolog_manager`;
- no registered quest handles that request;
- mission/reward/sub-mission progression does not complete through the intended new-system bridge.

## Result classification

### REPRODUCED
Use when normal collection reaches completion, the button requests `biolog_manager`, and no corresponding quest/reward/mission advance occurs.

### NOT REPRODUCED
Use if the deployed game package contains a working `biolog_manager` quest/object not present in the mapped repository and the full bridge completes normally. Record deployment drift rather than retracting from repo evidence alone.

### INCONCLUSIVE
Use when:
- actual DB biolog proto data prevents reaching completion;
- chosen mission does not expose the expected completion path;
- quest package/deployment differs materially;
- another Biolog bug blocks the flow first.

## Evidence template

- Test: `BIO-T01`
- Bug: `BUG-BIO-001`
- Result:
- Date/time:
- Client/server/game-package build:
- Biolog mission:
- Required / collected before final submit:
- Completion flag:
- Completion button visible:
- Event quest requested:
- Quest dialog/result:
- Mission before/after:
- Reward before/after:
- UI before/after:
- GAME/client errors:
- Deployment drift:
- Evidence reference:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
