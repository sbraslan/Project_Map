# Auto Hunt — Verified Bugs

## AUTO-001 — Active Auto Hunt survives premium entitlement expiry until logout/manual stop

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Auto Hunt entitlement lifecycle

### Evidence
- `do_autohunt b` checks `GetPremiumRemainSeconds(PREMIUM_AUTO_USE) > 0` only when activation is requested.
- Successful activation installs `AFFECT_AUTO` with `AFF_AUTO_USE` and `INFINITE_AFFECT_DURATION`.
- The affect-processing loop does not re-check `PREMIUM_AUTO_USE` expiry for `AFFECT_AUTO`.
- Logout explicitly removes `AFFECT_AUTO`, and manual `/autohunt d` removes it, but no source-proven online expiry path removes it when premium time reaches zero.

### Reachable consequence
A player can activate Auto Hunt shortly before premium entitlement expires and keep `AFF_AUTO_USE` active while remaining online after the entitlement deadline. Auto Hunt-specific movement/sync exemptions therefore continue without a valid premium entitlement until logout or manual deactivation.

### Fix boundary
Tie active Auto Hunt state to entitlement validity continuously: on premium expiry (or periodic affect processing), remove `AFFECT_AUTO` when `GetPremiumRemainSeconds(PREMIUM_AUTO_USE) <= 0`. Do not rely only on activation-time checks.

### Regression target
See `tests/auto_hunt.md#auto-001`.


## AUTO-002 — `/restart_auto` bypasses Auto Hunt entitlement and special-map revive-cost rules

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Auto Hunt restart lifecycle

### Evidence
- `restart_auto` is registered for `GM_PLAYER` and routes to `SCMD_RESTART_AUTOHUNT`.
- The intended Auto Hunt restart eligibility guard is commented out in the restart branch.
- The Zodiac revive/prism validation is executed only for `SCMD_RESTART_HERE`.
- `SCMD_RESTART_AUTOHUNT` is a distinct subcommand, so it bypasses that Zodiac prism dialog/cost branch.
- Later, the Auto Hunt restart branch directly calls `RestartAtSamePos()`, restores HP to 50, applies death penalty handling and revive invisibility.
- No `AFF_AUTO_USE` or `PREMIUM_AUTO_USE` check is performed before that branch.

### Reachable consequence
A dead player can invoke `/restart_auto` without an active Auto Hunt entitlement/state. On maps whose special revive restrictions are keyed specifically to normal restart subcommands—proven here for Zodiac prism handling—the player can reach same-position revival without paying/processing the intended special revive requirement.

The generic death wait guard still applies, so this is not classified as an instant-revive cooldown bypass.

### Fix boundary
Authorize `SCMD_RESTART_AUTOHUNT` server-side using active Auto Hunt + valid premium entitlement, and route it through the same map-specific revive validation/cost rules as the equivalent normal restart operation.

### Regression target
See `tests/auto_hunt.md#auto-002`.


## AUTO-003 — Auto Hunt disables the entire MOVE anti-cheat validation block

**Status:** VERIFIED_STATIC  
**Severity:** Critical  
**Affected:** ServerSRC / movement trust boundary

### Evidence
- In `CInputMain::Move()`, the teleport-distance check, dead-move guard, speedhack timing checks and combohack check are all wrapped by:
  `if (!ch->IsAffectFlag(AFF_AUTO_USE)) { ... }`.
- Therefore every one of those validations is skipped while `AFF_AUTO_USE` is active.
- After the skipped block, the server still accepts the client-provided movement coordinates and calls `Goto(pinfo->lX, pinfo->lY)` / `Move(pinfo->lX, pinfo->lY)`.

### Reachable consequence
A modified client with valid Auto Hunt state can send movement packets that would normally be rejected as teleport/speed/dead/combo movement abuse. Auto Hunt therefore acts as a broad server-side anti-cheat bypass rather than a narrowly scoped automation exception.

### Fix boundary
Do not wrap the full movement validation block with Auto Hunt state. Keep authoritative teleport/dead/speed/combo validation active and add only the minimum, explicitly bounded tolerance needed by legitimate Auto Hunt-generated movement.

### Regression target
See `tests/auto_hunt.md#auto-003`.

## AUTO-004 — Auto Hunt bypasses SyncPosition distance limits and permits extreme displacement of a sync-owned attackable target

**Status:** VERIFIED_STATIC  
**Severity:** Critical  
**Affected:** ServerSRC / SyncPosition trust boundary

### Evidence
- `SyncPosition()` normally rejects/logs sync deltas where `fDist > 25.0f`.
- Under `ENABLE_AUTO_SYSTEM`, that distance condition is guarded by `&& !ch->IsAffectFlag(AFF_AUTO_USE)`; active Auto Hunt therefore falls through to `victim->Sync(e->lX, e->lY)`.
- `SetSyncOwner(ch)` requires the target to be attackable and initially within the sync-owner distance boundary.
- Once the same character already owns sync, the >250 distance branch returns true when `m_pkChrSyncOwner == ch`, allowing ownership to persist after displacement.

### Reachable consequence
A modified client can first establish sync ownership over a nearby attackable target and, while Auto Hunt is active, submit a large coordinate delta that bypasses the normal SyncPosition distance guard. The server then applies the requested target position, enabling extreme server-authoritative displacement within valid map coordinates.

### Fix boundary
Never disable the authoritative sync-distance bound based solely on Auto Hunt. Preserve the same victim displacement limit for automated and manual play, or introduce a tightly bounded server-generated Auto Hunt movement path that does not trust arbitrary client SyncPosition coordinates.

### Regression target
See `tests/auto_hunt.md#auto-004`.
