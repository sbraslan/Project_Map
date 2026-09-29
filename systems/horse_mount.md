# Horse / Mount / Riding — Static Map

**Status:** STATIC COMPLETE  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.

## Entry roots

Server:
- `game/src/horse_rider.cpp/.h` — persistent horse level/health/stamina, ride-state events;
- `game/src/char_horse.cpp` — summon/dismiss/ride/unride/death/revive, horse character ownership;
- `game/src/questlua_horse.cpp` — Lua horse API;
- `game/src/char_item.cpp` — mount item/use/affect integration;
- `game/src/item.cpp` — ride-item classification, mount proto affects, ChangeLook mount expiry helpers;
- `game/src/char.cpp` — mount VNUM, stat recomputation, warp/show/login persistence.

Client/data surfaces to close:
- horse/mount packet and render state;
- mount costume/item proto families;
- horse NPC/race assets;
- horse appearance persistence;
- mount ChangeLook paths.

## Initial lifecycle map

### Classic horse state
`CHorseRider` owns:
- level;
- health;
- stamina;
- riding flag;
- stamina consume/regeneration events;
- health-drop timing.

`StartRiding()` requires positive horse level, health and stamina, then starts stamina consumption.

`StopRiding()` clears the riding flag and returns to stamina regeneration.

The horse-rider destructor cancels both stamina events.

### Character horse actor
`CHARACTER::HorseSummon(true)` spawns a horse actor, binds rider/horse pointers and sends horse-state UI data.

`CHARACTER::StartRiding()`:
1. validates map/state restrictions;
2. derives current horse/mount race;
3. enters CHorseRider riding state;
4. dismisses the standing horse actor;
5. sets character mount VNUM.

`StopRiding()` clears the mount VNUM and normally respawns the previously ridden horse actor.

Dead-horse summon creates a delayed dead-event which later dismisses the horse actor.

### Mount item/costume layer
`CItem::IsRideItem()` covers:
- UNIQUE_SPECIAL_RIDE;
- UNIQUE_SPECIAL_MOUNT_RIDE;
- COSTUME_MOUNT under the active mount-proto-affect system.

Mount item equip/unequip integrates with `POINT_MOUNT`, `MountVnum()` and full point recomputation.

### ChangeLook mount expiry candidate
Mount/horse-summon ChangeLook has a dedicated event using socket2.

`CItem::IsExpireTimeItem()` currently uses:
`if (GetType() != ITEM_COSTUME && GetSubType() != COSTUME_MOUNT) return false;`

That predicate is broader than the apparent intended exact type/subtype check. It remains an **unpromoted candidate** until a live caller/reachable consequence is proven in this subsystem.

## Initial cross-system observations
- Achievement data contains SUMMON_MOUNT tasks, while prior Achievement audit found no mapped horse/mount caller invoking that task family.
- Costume/Appearance audit already identified mount ChangeLook helper gaps but intentionally left them unpromoted pending a closed mount call chain.
- Horse/mount state affects character combat stats and therefore needs explicit recompute/cleanup auditing across dismount, item expiry, death, warp and logout.

## Exact next work
1. close horse persistence/login/logout and level bounds;
2. audit stamina/health events for lifetime and duplicate-event state;
3. map quest horse API authorization and range checks;
4. trace mount item/costume use -> affect -> `MountVnum` lifecycle;
5. trace item expiry/unequip/death/warp cleanup;
6. close ChangeLook mount expiry caller graph;
7. map Achievement SUMMON_MOUNT producer gap in this owning subsystem;
8. close client race/proto/appearance asset coverage;
9. create bugs/tests only for verified reachable paths.

No Horse/Mount runtime test is authorized. Global first future live gate remains `DUNGEON-T09`.


## Checkpoint — persistence/login + stamina/health event lifecycle closed

### Persistence / login
- `TPlayerTable::horse` persists level, riding, stamina, health and health-drop timestamp.
- DB save/load covers `horse_level`, `horse_riding`, `horse_hp`, `horse_hp_droptime` and `horse_stamina`.
- `CHARACTER::SetPlayerProto()` restores the raw `THorseInfo` and applies offline stamina regeneration through `UpdateHorseDataByLogoff()`.
- Enter-game then calls `EnterHorse()`; when persisted `bRiding` is true, `EnterHorse()` normalizes the flag and re-enters `StartRiding()`, recreating the consume-event side of the state machine.
- `SetHorseLevel()` clamps API-driven horse levels to `0..HORSE_MAX_LEVEL` (30).
- DB-loaded `THorseInfo::bLevel` itself is copied without a clamp before horse-stat table access. No tracked in-repository malformed producer was established, so this remains a persistence-integrity candidate rather than a promoted bug.

### Current-build health/stamina semantics
- `ENABLE_INFINITE_HORSE_HEALTH_STAMINA` is enabled in the active common defines.
- In this build, `GetHorseHealth()` and `GetHorseStamina()` always return the level maximum.
- Therefore classic consume/drop exhaustion is intentionally masked at the public accessor layer; the stamina consume event continues scheduling while riding, but cannot force a zero-stamina dismount through the accessor in the current build.
- Regen/consume event switching is single-owner: starting one cancels the opposite event, and `CHorseRider::Destroy()` cancels both.
- No duplicate-event or lifetime bug was verified in the current build.

### Login normalization note
- Later login setup calls `SetHorseLevel(GetHorseLevel())`, which resets underlying horse HP/stamina/drop-time to level maxima/new drop time.
- With infinite horse health/stamina enabled this has no distinct current gameplay consequence; if that feature is disabled later, this path must be reopened because persisted HP/stamina semantics would change materially.

### ChangeLook mount follow-up
- `StartChangeLookExpireEvent()` supports both costume mounts and horse-summon items.
- Current automatic event-start sites found in `ITEM_MANAGER::CreateItem()` and `CItem::OnAfterCreatedItem()` only start it for `IsHorseSummonItem()`.
- This is now the lead ChangeLook-mount candidate. It is not promoted until the socket2 producer/load path and a reachable costume-mount consequence are closed.

## Exact next work
1. audit quest horse API authorization/range handling and current quest producers;
2. trace mount item/costume -> affect -> `MountVnum` lifecycle;
3. trace expiry/unequip/death/warp cleanup;
4. close ChangeLook mount socket2 producer + automatic event-start caller graph;
5. resolve Achievement SUMMON_MOUNT producer gap;
6. close client race/proto/appearance asset coverage;
7. create bugs/tests only for verified reachable paths.

No Horse/Mount runtime test is authorized. Global first future live gate remains `DUNGEON-T09`.


## Checkpoint — active quest API + mount state synchronization

### Active horse quest deployment
The current `quest_list` loads these eight Horse/Mount scripts:
- `horse_exchange_ticket.quest`
- `horse_guard.quest`
- `horse_menu.quest`
- `horse_revive.quest`
- `horse_ride.quest`
- `horse_summon.quest`
- `ride_ticket_change.quest`
- `training_mount.quest`

Verified current producers:
- `horse_menu` -> `horse.ride/unride/summon/unsummon/revive/feed/set_name`;
- `horse_summon` -> `horse.summon()` with no custom VNUM argument;
- `horse_ride` -> `pc.mount(20030, 600)` and item 71241 -> `pc.mount(20030, 10)`;
- `training_mount` consumes horse-level gates but does not change horse level.

No active h_horse producer was found for:
- `horse.set_level`;
- `horse.advance`;
- `horse.set_appearance`;
- `horse.set_stat0`.

The current package therefore depends on an already-existing horse level for grade-gated flows. This is kept as a deployment/progression candidate until all non-h_horse active quest producers are excluded.

### BUG-HORSE-001 — pc.mount affect does not synchronize MountVnum
Current build enables `ENABLE_MOUNT_COSTUME_SYSTEM` and therefore `ENABLE_MOUNT_PROTO_AFFECT_SYSTEM`.

Live path:
`horse_ride.quest`
-> `pc.mount(20030, duration)`
-> `questlua_pc.cpp::pc_mount`
-> `AddAffect(AFFECT_MOUNT, POINT_MOUNT, 20030, ...)`
-> `ComputeAffect`
-> `PointChange(POINT_MOUNT, 20030)`.

But `CHARACTER::PointChange(POINT_MOUNT)` only updates the point value; its `MountVnum(val)` call is commented out.

No later call in `pc_mount` synchronizes `m_dwMountVnum`.

Consequences in the reachable active quest path:
- `GetPoint(POINT_MOUNT)` becomes non-zero;
- `GetMountVnum()` remains unchanged;
- `pc.is_mount()` reads `GetMountVnum()`, so it does not reflect the new affect-backed mount;
- character packet/render riding state also continues to use `GetMountVnum()`;
- the active rental horse path therefore has split authoritative state.

Promoted as `BUG-HORSE-001`.

### Item/costume mount lifecycle closure
Normal equipped ride-item flow is separate from `pc.mount` and does explicitly synchronize:
- `CItem::ModifyPoints(true)` applies `APPLY_MOUNT -> POINT_MOUNT`;
- `CHARACTER::EquipItem()` then calls `MountVnum(GetPoint(POINT_MOUNT))`;
- unequip removes the apply and then calls `MountVnum(GetPoint(POINT_MOUNT))` again.

Death cleanup:
- `Dead()` clears riding state;
- `UnEquipSpecialRideUniqueItem()` covers special-ride UNIQUE items and `WEAR_COSTUME_MOUNT`.

Restricted-map cleanup:
- `WarpSet()` itself does not unmount;
- post-warp EnterGame checks `IS_MOUNTABLE_ZONE()` and calls `Unmount()` when required.

No separate death/warp stale-mount bug was verified in these paths.

### ChangeLook mount candidate strengthened
`CTransmutation::Accept()` writes only:
`left->SetChangeLookVnum(right->GetVnum())`
and then destroys the right material.

It does not:
- call `right->IsExpireTimeItem()`;
- copy `right->GetRealExpireTime()` into the target's socket2;
- call `StartChangeLookExpireEvent()`.

Additionally, automatic ChangeLook expire-event start sites currently found in `ITEM_MANAGER::CreateItem()` and `CItem::OnAfterCreatedItem()` gate on `IsHorseSummonItem()`, not `COSTUME_MOUNT`.

This is a strong lifetime-laundering candidate, but promotion is deferred until a current deployed time-limited `COSTUME_MOUNT` proto is verified.

## Exact next work
1. verify current time-limited COSTUME_MOUNT proto/data reachability and close the ChangeLook lifetime candidate;
2. exclude/locate non-h_horse active producers for horse level progression;
3. close timer-based/real-time mount item expiry paths;
4. resolve Achievement SUMMON_MOUNT producer gap;
5. close client race/proto/horse-appearance asset coverage;
6. promote only verified reachable additional Horse/Mount bugs/tests.

No runtime execution is authorized. Global first future live gate remains `DUNGEON-T09`.


## Checkpoint — horse progression + expiry + Achievement ownership

### BUG-HORSE-002 — tracked deployment has no normal-player horse-level progression producer
Current deployment evidence:
- `quest_list` loads exactly the tracked h_horse package already mapped.
- recursive tracked quest/object state contains only the eight deployed horse/mount scripts; no horse level-up/mission/advance quest state exists.
- no compiled `object/50050/use` handler exists.
- `horse_exchange_ticket.quest` converts item 50005 into item 50050, but the tracked quest package contains no consumer that turns 50050 into horse level progression.
- active horse scripts call `horse.get_level/get_grade` for gates but never call `horse.advance` or `horse.set_level`.
- server Lua exposes `horse.advance` and `horse.set_level`, proving the engine-side capability exists.
- the direct command `/horse_level` is registered at `GM_HIGH_WIZARD`, so it is not a normal-player progression path.
- no hardcoded 50050/horse-level setter path was found in the mapped item-use surfaces.

Result: within the tracked deployment, normal gameplay has horse-grade/level consumers but no producer. Existing characters with pre-populated DB horse levels can still use the system, but new/zero-level progression is not implemented by the tracked content.

Promoted as `BUG-HORSE-002`.

### Mount expiry lifecycle closed
For equipped ride items / costume mounts:
`real_time_expire_event` or `timer_based_on_wear_expire_event`
-> `ITEM_MANAGER::RemoveItem`
-> `CItem::RemoveFromCharacter`
-> `CItem::Unequip`
-> `ModifyPoints(false)`
-> active mount-proto-affect branch re-evaluates `MountVnum(GetPoint(POINT_MOUNT))`.

Therefore normal item expiry removes the mount point and synchronizes the visible mount VNUM. No separate stale-mount expiry bug was verified.

### Achievement SUMMON_MOUNT ownership resolved
Current `achievements.xml` contains configured `TYPE_SUMMON_MOUNT` tasks (including achievements 61, 62 and 63).

Horse/mount summon/equip/ride paths contain no `CAchievementSystem::OnSummon(...TYPE_SUMMON_MOUNT...)` producer. PetSystem does call `OnSummon(...TYPE_SUMMON_PET...)`.

This is already canonically owned by `BUG-ACH-006` in the completed Achievement subsystem. Horse/Mount mapping therefore cross-references that bug and does not create a duplicate Horse bug ID.

### ChangeLook lifetime candidate status
Static code still shows:
- ChangeLook accept stores donor VNUM then destroys donor;
- donor expiry metadata is not transferred;
- costume-mount ChangeLook expiry event restart is not wired like horse-summon items.

However, the tracked `Project_DumpProto/*/item_proto.txt` snapshot is not exposed by the current connector as decodable line text, so a deployed time-limited `COSTUME_MOUNT` row cannot be proven from repository data in this pass. The candidate remains unpromoted rather than inferred.

## Exact next work
1. close Additional Equipment Page interaction with UNIQUE ride items;
2. close client mount packet/race/asset and horse-appearance coverage;
3. audit horse name/appearance persistence + ChangeLook interaction;
4. revisit ChangeLook lifetime only if a concrete time-limited COSTUME_MOUNT row becomes readable;
5. decide Horse/Mount STATIC COMPLETE and prepare deferred runtime tests.

No runtime execution is authorized. Global first future live gate remains `DUNGEON-T09`.


## Final checkpoint — client race/combat coverage + subsystem closure

### BUG-HORSE-003 — deployed 2021/2022 mount races fall through client mount classification

Current packed item proto was decoded from the exact blob shared by:
- `Project_Binary/locale/locale/common/item_proto`;
- `Project_DumpProto/tr/item_proto`;
- blob SHA: `24ed504beb93a38aff772b8c24d9b6e1bcd4c2d0`.

Verified current mount items:
- 71259 -> `APPLY_MOUNT 20276`
- 71260 -> `APPLY_MOUNT 20277`
- 71261 -> `APPLY_MOUNT 20278`
- 71262 -> `APPLY_MOUNT 20279`
- 71263 -> `APPLY_MOUNT 20280`
- 71264 -> `APPLY_MOUNT 20281`
- 71265 -> `APPLY_MOUNT 20282`
- 71266 -> `APPLY_MOUNT 20283`

All eight rows are current `ITEM_COSTUME / COSTUME_MOUNT` records and all have matching client `npclist.txt` race names.

Client feature state:
- `ENABLE_NO_MOUNT_CHECK` is disabled.
- `InstanceBase.cpp::GetMountLevelByVnum()` contains none of 20276..20283.
- unmatched races return `MOUNT_TYPE_NONE`.

Combat consequence:
- `SHORSE::CanAttack()` requires at least `MOUNT_TYPE_COMBAT`;
- `SHORSE::CanUseSkill()` requires `MOUNT_TYPE_MILITARY`;
- `CInstanceBase::CanAttackHorseLevel()` forwards `m_kHorse.CanAttack()`;
- `CPythonPlayer` clears auto-attack when a mounted actor fails that check;
- normal `CInstanceBase::CanAttack()` is also rejected by the horse check.

The server can therefore equip and render these current mount races, but the client classifies them as no mount level and blocks mounted combat/horse skills.

Promoted as `BUG-HORSE-003`.

### Client mount packet/render lifecycle closed
Normal server `CHARACTER::MountVnum()`:
1. updates `m_dwMountVnum`;
2. sends an actor re-insert;
3. `TPacketGCCharacterAdditionalInfo.dwMountVnum` carries the race;
4. client `RecvCharacterAdditionalInfo` stores it in `SNetworkActorData`;
5. `NetworkActorManager` recreates the instance when mount status changes;
6. `CInstanceBase::Create` calls `MountHorse(race)`.

The commented direct mount/dismount block inside `NetworkActorManager::UpdateActor()` is therefore not a separate defect for the normal `MountVnum()` path.

### Horse name / appearance closure
Horse names:
- quest setter validates name then writes a 30-day quest flag/affect;
- `CHorseNameManager` broadcasts through DB;
- DB persists in `horse_name` via `REPLACE`;
- login requests missing cached name;
- affect processing calls `CHorseNameManager::Validate()` to remove expired names.

Horse appearance:
- `horse_appearance` is persisted in the player table and restored at login;
- `GetMyHorseVnum()` prefers the persisted appearance when non-zero;
- Lua `horse.set_appearance` accepts an arbitrary numeric VNUM, but no tracked active quest producer calls it, so no reachable validation bug is promoted.

### ChangeLook mount lifetime candidate — data side proven, reachability not promoted
The decoded current item proto contains 131 time-limited `COSTUME_MOUNT` rows, including real-time family 52001+.

Transmutation:
- accepts compatible mount donor material;
- stores only donor VNUM;
- destroys donor;
- does not transfer donor lifetime into target socket2;
- does not start target ChangeLook expiry as part of Accept.

Engine `game.open_transmutation` exists, but the tracked current `quest_list` / compiled quest object tree does not expose a normal-player opener in this snapshot. Under the verified-reachability rule this remains an unpromoted cross-system candidate owned by Costume/Appearance.

### Additional Equipment Page closure
- `WEAR_COSTUME_MOUNT` is excluded from alternate equipment page by `GetWearNotChange()`.
- UNIQUE ride slots can theoretically participate; the remove-side helper has a suspicious `IsEquipped()` check after clearing `m_bEquipped`, but no tracked live `RefreshAdditionalEquipmentItems(..., false)` producer was established.
- no Horse bug is promoted from this boundary.

### Asset coverage note
`npclist.txt` contains 20276..20283 mappings. Raw newer MSM/GR2 client pack assets are not versioned in the tracked repos, so physical pack presence remains a runtime/deployment check rather than a static missing-asset bug.

## Horse / Mount / Riding — STATIC COMPLETE

Verified Horse bug set:
- `BUG-HORSE-001`
- `BUG-HORSE-002`
- `BUG-HORSE-003`

Deferred runtime tests:
- `HORSE-T01`
- `HORSE-T02`
- `HORSE-T03`

No Horse/Mount runtime test has been executed. Global first future live gate remains `DUNGEON-T09`.
