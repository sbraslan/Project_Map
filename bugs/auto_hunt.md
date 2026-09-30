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
