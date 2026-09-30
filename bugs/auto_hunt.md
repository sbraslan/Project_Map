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
