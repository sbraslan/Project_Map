# Auto Hunt — Test Plan

## AUTO-001 — Premium expiry while online
1. Give a character a short remaining `PREMIUM_AUTO_USE` duration.
2. Activate Auto Hunt and verify `AFF_AUTO_USE` is present.
3. Stay online until the premium expiry time passes without relogging.
4. **Expected after fix:** server removes Auto Hunt state at/after entitlement expiry and movement/sync exceptions stop applying.
5. Verify manual deactivation still works before expiry.
6. Verify Auto Hunt can be activated again only after entitlement becomes valid again.
