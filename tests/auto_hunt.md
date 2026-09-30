# Auto Hunt — Test Plan

## AUTO-001 — Premium expiry while online
1. Give a character a short remaining `PREMIUM_AUTO_USE` duration.
2. Activate Auto Hunt and verify `AFF_AUTO_USE` is present.
3. Stay online until the premium expiry time passes without relogging.
4. **Expected after fix:** server removes Auto Hunt state at/after entitlement expiry and movement/sync exceptions stop applying.
5. Verify manual deactivation still works before expiry.
6. Verify Auto Hunt can be activated again only after entitlement becomes valid again.


## AUTO-002 — Auto restart entitlement + special-map revive rules
1. Use a character with no valid Auto Hunt premium and no `AFF_AUTO_USE`.
2. Die and send `/restart_auto` after the normal death wait threshold.
3. **Expected after fix:** server rejects Auto Hunt restart.
4. Activate valid Auto Hunt, enter Zodiac, create a state requiring a Prism of Revival, then die.
5. Send the Auto Hunt restart path.
6. **Expected after fix:** the same Zodiac prism requirement/dialog/payment rules used by the normal same-position revive path are enforced.
7. Verify valid Auto Hunt restart still functions on ordinary maps after the standard death wait.
