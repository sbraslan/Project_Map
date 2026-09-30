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


## AUTO-003 — MOVE anti-cheat remains authoritative during Auto Hunt
1. Activate valid Auto Hunt.
2. Send crafted MOVE packets exceeding normal teleport-distance thresholds.
3. Send abnormal timing/speed packets and, where applicable, dead/combo movement cases.
4. **Expected after fix:** the same authoritative rejection/logging rules still apply while Auto Hunt is active; only explicitly allowed automation tolerance differs.
5. Confirm normal Auto Hunt movement continues to work.

## AUTO-004 — SyncPosition distance enforcement during Auto Hunt
1. Activate valid Auto Hunt.
2. Establish normal sync ownership over a nearby attackable target.
3. Send a crafted SyncPosition element moving that target by more than the normal allowed delta.
4. **Expected after fix:** request is rejected/ignored and the target position remains authoritative.
5. Repeat after the target has moved beyond the initial owner-distance threshold.
6. Verify existing sync ownership cannot be used to chain large displacements.
