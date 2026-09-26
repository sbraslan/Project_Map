# biolog — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-BIO-001 — new Biolog manager completion quest is missing from Project_Game
- Statik durum: **doğrulandı**
- Sınıf: integration break / progression dead-end

When the C++ Biolog manager reaches the required collected count it sets quest flags named under `biolog_manager.*`.
The client completion button explicitly calls:
`SendRequestEventQuest("biolog_manager")`.

Current `Project_Game/share/locale/europe/quest/quest_list` registers only legacy biolog quests:
`collect_herb_lv4` and `collect_quest_lv30...94`.

No source/object named `biolog_manager` exists in the current Project_Game tree, and the legacy quests do not call the new `pc.biolog_*` Lua API.

Thus the mapped current repository has no consumer that performs the new system's sub-mission/reward/mission-advance step after C++ collection reaches completion.

### BUG-BIO-002 — TIMER packet extra-length validation uses the wrong byte count
- Statik durum: **doğrulandı**
- Sınıf: packet framing / out-of-bounds read

Server packet-info defines `HEADER_CG_BIOLOG_MANAGER` base length as `sizeof(TPacketCGBiologManagerAction)` (2 bytes).

`CInputMain::BiologManager` advances `c_pData` by those 2 bytes, but passes the original `uiBytes` unchanged to `RecvClientPacket`.

TIMER then checks:
`if (uiBytes < sizeof(bool)) return -1;`
and reads `*(bool*)c_pData`.

If exactly the 2-byte base packet is currently available, `uiBytes` is still 2, so the check passes even though zero payload bytes remain. The bool read occurs beyond the currently received packet bytes and the handler returns an extra consumed byte.

This is reachable through malformed input and can also be exposed by TCP fragmentation of the official two-write TIMER send path.

### BUG-BIO-003 — reminder event-state boolean is uninitialized
- Statik durum: **doğrulandı**
- Sınıf: uninitialized state / nondeterministic reminder

CHARACTER initialization sets:
- `m_pkBiologManager = nullptr`
- `s_pkReminderEvent = nullptr`

but does not initialize `m_BiologReminderEventState`.

On player load, after creating the manager, code calls:
`SetBiologCooldownReminder(t->m_BiologCooldownReminder)`.

For a persisted nonzero reminder preference this setter eventually calls `IsBiologRemiderEvent()`, reading the uninitialized boolean. A random true value prevents creation of the reminder event.

### BUG-BIO-004 — Biolog reward apply_type can underflow/out-of-bounds aApplyInfo
- Statik durum: **doğrulandı as DB/config trust bug**
- Sınıf: config validation / memory safety

Reward rows are loaded from `biolog_rewards` into uint16 `wApplyType[]`.

Both info/reward code use:
`pReward->wApplyType[i] - 1`.

`pc_biolog_set_reward_bonus` then indexes:
`aApplyInfo[wApply].wPointType`
without validating the derived apply index.

A zero apply_type with nonzero value underflows to 65535; any out-of-range DB value can likewise index outside `aApplyInfo`.

### BUG-BIO-005 — item consumption and Biolog progress durability are non-atomic
- Statik durum: **doğrulandı**
- Sınıf: crash consistency

Submission removes/mutates the required item through normal item persistence and separately changes:
- biolog_collected
- biolog_cooldown
- reminder state
in CHARACTER memory.

Biolog setters do not synchronously save the player row.
Normal CHARACTER save event defaults to 120 seconds.

Therefore item durability and successful-progress durability are separate. Crash timing can produce:
- consumed item durable but successful collected increment rolled back, or
- player progress saved while item mutation is not yet durable.

Runtime fault injection is required to characterize exact timing windows.

### BUG-BIO-006 — repeated reward helper stacks permanent Biolog affects
- Statik durum: **doğrulandı helper behavior; current caller missing**
- Sınıf: trusted quest replay / reward duplication

`pc.biolog_set_reward_bonus()` has no already-claimed guard and loops every configured bonus.

It calls:
`AddAffect(AFFECT_COLLECT, ..., INFINITE_AFFECT_DURATION, ..., false)`.

In `CHARACTER::AddAffect`, when `bOverride=false`, an existing affect does not prevent creation of another affect; a new `CAffect` is appended.

Any quest path that invokes the helper repeatedly can therefore stack the same permanent Biolog bonus. Current Project_Game does not contain the intended new quest caller, so this is presently a trusted-script/ future-integration hazard rather than a mapped player packet exploit.

### BUG-BIO-007 — crafted SEND can consume Researcher Elixir after mission collection is already complete
- Statik durum: **doğrulandı**
- Sınıf: server validation / resource loss

`SendBiologItem` processes the Researcher Elixir affect before checking:
`if (iCount < wRequiredItemCount)`.

If the mission is already at required count and a modified client sends CG SEND:
- required item presence and cooldown are checked;
- Researcher Elixir affect is found and removed;
- then the count condition is false and no item submission occurs.

Official UI hides/disables the normal submit button at required count, so this is a modified-client/server-validation resource-loss path.

### OBS-BIO-001 — client reward bonus getter lacks index bounds
The Python binding accepts arbitrary integer `index` and directly calls fixed-array getters. Official UI uses 0..MAX_BONUSES_LENGTH-1; modified Python can read out-of-range client memory.

### OBS-BIO-002 — sequence mismatch is dormant in current build
Client `SendBiologManagerAction` calls `SendSequence()`, while server packet-info marks Biolog packet as non-sequenced.
Both current source flags comment out ENABLE_SEQUENCE_SYSTEM, so SendSequence is a no-op today. Re-enabling sequence without fixing this pair would break Biolog packet framing.

### OBS-BIO-003 — empty proto boot vector uses vector[0]
If `biolog_missions` or `biolog_rewards` is empty, DB initialization still returns true.
Boot encoding uses `&m_vec_BiologMissions[0]` / `&m_vec_BiologRewards[0]` even for zero-size vectors. The encoded byte count is zero, but indexing an empty vector is undefined behavior.


## Hunting System — first-pass bugs
