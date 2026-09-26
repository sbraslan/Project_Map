# Battle Field System — Deferred Validation Notes

**Phase:** Detection / Mapping Only
**Execution status:** NOT RUN — documentation only

These are future validation ideas only. No runtime/fault-injection execution is allowed in the current phase.

## BFIELD-T01 — out-of-map delayed exit
Covers BUG-BFIELD-001:
- from a normal non-Battle-Field map where `CanWarp()` is true, issue `/exit_battle_field`;
- expected static behavior: Battle Field exit timer is created and character is warped to empire start.

## BFIELD-T02 — out-of-map immediate dead-exit command
Covers BUG-BFIELD-002:
- from outside Battle Field, including while alive, issue `/exit_battle_field_on_dead 1`;
- expected static behavior: direct `ExitCharacter` path, immediate empire-start warp, Battle Field cooldown set.

## BFIELD-T03 — repeat-kill cooldown renewal
Covers BUG-BFIELD-003:
- kill the same victim once;
- verify kills inside 60 seconds are rejected;
- wait until the stored expiry passes;
- kill once and immediately repeat;
- expected static defect signature: subsequent immediate repeats are accepted because the existing map timestamp was never overwritten.

Do not execute these tests unless the user explicitly changes the project phase.
