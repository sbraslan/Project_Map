# hunting

**Status:** PARTIAL — ACTIVE (static audit nearly complete)

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Hunting System audit

**Tarih:** 2026-09-26

Biolog STATIC COMPLETE sonrasında Hunting System açıldı.

### Roots mapped
- Server runtime: `game/src/char_hunting.cpp`
- CG action dispatch: `input_main.cpp::ReciveHuntingAction`
- kill progress hook: `char_battle.cpp::UpdateHuntingMission`
- login/level-up bootstrap: `input_login.cpp::CheckHunting` + `char.cpp::CheckHunting`
- static tables: `game/src/constants.h` + `game/src/constants.cpp`
- client packet structs: `UserInterface/Packet.h`
- client packet registration/parser: `PythonNetworkStream.cpp` + `PythonNetworkStreamPhaseGame.cpp`
- client UI: `root/uihunting.py`
- client send: `m2netm2g.SendHuntingAction`
- state/persistence: `hunting_system.*` quest flags via `quest::PC`.

### Action protocol
- action 1: open current/main window
- action 2: select mission type using client-controlled `dValue`
- action 3: open cached reward window
- action 4: receive cached rewards.

Official UI sends mission type 0/1.

### State machine
First `CheckHunting`:
- is_active 0 -> -1
- level -> 1
- type -> -1
- count -> 0.

Selection:
is_active -1 + player level >= hunting level
-> is_active 1
-> type = client dValue
-> count 0
-> main window.

Kill:
`char_battle -> UpdateHuntingMission(monsterVnum)`
-> target/count table match
-> increment quest flag
-> completion -> `SetCachedRewards`.

Cached reward flags:
- reward_race / count
- reward_rand / count
- reward_money
- reward_exp
- reward_cached.

Claim grants rewards, clears flags, sets inactive and increments hunting level by one.

## Static table audit — COMPLETE
Active constants:
- `HUNTING_MISSION_COUNT = 90`
- `THuntingMissions[91][2][2]`
- `THuntingRewardItem[91][2][4][2]`
- `THuntingRewardMoney[9]`
- `THuntingRewardEXP[9]`
- random reward arrays: `6 / 13 / 13 / 13 / 13` entries for level bands 1-20 / 21-40 / 41-60 / 61-80 / 81-90.

Initializer counts match declarations exactly:
- missions: 91 rows
- race rewards: 91 rows
- money: 9 rows
- EXP: 9 rows
- random item groups: 6, 13, 13, 13, 13.

Content checks:
- mission levels 1-90 have two nonzero target/count pairs each;
- reward rows 1-90 have structurally valid VNUM/count pairs;
- money/EXP bands cover 1-90 continuously;
- random reward entries have nonzero VNUM/count values.
- Final `THuntingMissions` source comment says `// Lv80`, but physical position is index 90; this is a comment typo, not runtime logic.

The remaining data-compatibility check is to compare the 62 unique Hunting reward VNUMs against the actual server item-proto dataset. The repository export is non-UTF8 and the GitHub connector cannot currently decode that large file, so no missing-proto claim is recorded.

## Packet/parser/sequence audit — COMPLETE
Client/server values agree:
- CG Hunting action = 220
- GC open-main/select/reward/update/random-items = 198..202.

Client packet header map registers all five GC packets as `STATIC_SIZE_PACKET` using the correct struct sizes.
`PythonNetworkStreamPhaseGame.cpp` handles all five packet cases and forwards them to the Python BINARY handlers.
Client `SendHuntingAction` sends the fixed struct and calls `SendSequence()`.

Server `CPacketInfoCG` registers:
`HEADER_CG_SEND_HUNTING_ACTION` with `sizeof(TPacketGCHuntingAction)` and sequence flag `true`.

No client/server packet-size, header, parser-routing, or sequence mismatch was found.

## Persistence / crash consistency
`CHARACTER::SetQuestFlag` -> `quest::PC::SetFlag` updates the in-memory flag map and queues DB persistence in `m_FlagSaveMap`.
The flag DB write only happens later in `PC::Save()` via `HEADER_GD_QUEST_SAVE`.

Normal character save event:
- configured at `passes_per_sec * 120` = 120 seconds;
- `save_event` queues `CHARACTER::Save()` and flushes delayed items;
- delayed character saves are processed about every 1.16 seconds;
- `SaveReal()` sends player state first, then calls `PC::Save()`.

Item persistence is separate:
- `CItem::Save()` -> `ITEM_MANAGER::DelayedSave`;
- `ITEM_MANAGER::Update()` runs about every 5.08 seconds and writes `HEADER_GD_ITEM_SAVE`.

Therefore Hunting reward grant and reward-flag clearing are not one DB transaction. A GAME crash can persist a newly granted item while the old nonzero reward flags remain in DB. This is recorded as BUG-HUNT-005.

## Reward delivery robustness
`ReciveHuntingRewards()` creates both item rewards with `ITEM_MANAGER::CreateItem` and never checks for `nullptr`.
`CreateItem` can return `nullptr` for an invalid/missing proto or creation failure. In that case the current code can dereference the null pointer during inventory/ground handling.

This is a confirmed null-safety defect in the code path, but current-table reachability is not promoted to a separate verified bug until the actual server item-proto dataset can be checked.

The full-inventory fallback calls `AddToGround()`, but ignores its boolean result and clears the reward flags afterward. A failed ground insertion is therefore also a reward-loss risk; normal connected-player reachability still needs runtime/fault-injection validation.

## Level-90 terminal audit — COMPLETE
Legitimate mission-90 claim always increments `hunting_system.level` to 91.
There is no server terminal state.

At level 91:
- a player below hunting level receives the zero-data main packet path;
- a player whose character level is >=91 is routed to `OpenHuntingWindowSelect()`;
- that server path indexes `THuntingMissions[91]` / `THuntingRewardItem[91]`, outside the defined 0..90 domain.

Client-side completion display does not protect the server from this index, because the server must build/send the packet first.
This confirms BUG-HUNT-004 end-to-end.

## Verified bugs
- BUG-HUNT-001: action 2 mission type is unvalidated; arbitrary client value becomes fixed-table index.
- BUG-HUNT-002: action 4 has no reward/completion authorization and always advances mission level.
- BUG-HUNT-003: gold-cap rejection still clears cached gold reward.
- BUG-HUNT-004: final mission 90 claim advances level to 91 with no server terminal guard.
- BUG-HUNT-005: reward item persistence can commit before reward quest flags, allowing duplicate claim after a GAME crash.

## Remaining before static close
1. Verify the 62 unique Hunting reward VNUMs against the actual server item-proto dataset/export.
2. If all VNUMs exist, classify the `CreateItem(nullptr)` path as defensive robustness only; otherwise promote it to a verified crash bug.
3. Then mark Hunting STATIC COMPLETE and move to the next subsystem/runtime phase.

## Related
- Bugs: `../bugs/hunting.md`
- Runtime tests: `../tests/hunting.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
