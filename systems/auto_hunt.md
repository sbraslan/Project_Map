# Auto Hunt — Static System Map

**Status:** STATIC MAPPING CLOSED  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source snapshot:** pinned by `MAP_STATE.json`

## Canonical scope
Auto Hunt owns an independent runtime lifecycle:
- account entitlement / premium time (`PREMIUM_AUTO_USE`);
- `/autohunt` command activation/deactivation;
- `AFFECT_AUTO` + `AFF_AUTO_USE` runtime state;
- logout cleanup;
- AutoHunt restart state and `/restart_auto` client command path;
- movement/sync anti-hack exceptions while Auto Hunt is active;
- client affect/UI restart integration.

## Deployment proof
- `ENABLE_AUTO_SYSTEM` and `ENABLE_AUTO_RESTART_EVENT` are enabled on the pinned snapshot.
- `do_autohunt` is registered as player command.
- activation checks `GetPremiumRemainSeconds(PREMIUM_AUTO_USE) > 0` and installs `AFFECT_AUTO/AFF_AUTO_USE`.
- logout removes `AFFECT_AUTO`.
- movement/sync validation changes behavior when `AFF_AUTO_USE` is active.
- client schedules `/restart_auto` flows and reacts to Auto Hunt affects.

## Boundary
Owned here:
- entitlement-to-active-state transition;
- active Auto Hunt affect lifecycle;
- restart state;
- Auto Hunt-specific movement/sync exceptions.

Not owned here:
- generic premium account storage;
- generic movement/combat subsystem;
- Switchbot (already CLOSED);
- VIP entitlement itself.

## VIP classification
`ENABLE_VIP_SYSTEM` is active, but `IsVIP()` is currently a thin GM-authority check (`GM_VIP`) plus affect marker. No independent user-owned persistence/packet/lifecycle was proven in this discovery pass. VIP remains integrated/noncanonical unless new source evidence shows a separate lifecycle.

## Audit cursor
1. Entitlement activation/deactivation and premium expiry handling.
2. Logout/death/map/warp/restart lifecycle.
3. Movement/sync anti-hack bypass boundaries.
4. Client/server command and affect parity.
5. Close subsystem and return to discovery queue.


## Closeout
- Entitlement activation/expiry, logout/death/restart lifecycle, movement/sync trust boundaries and client/server command/affect parity were audited.
- Final verified bugs: 4.
- Final test plans: 4.
- No additional parity defect was promoted.
- Lifecycle is CLOSED and locked on the pinned source snapshot.
