# Skill Color — Test Plan

## SKCLR-001 — Ninth-skill slot parity
1. Build with `ENABLE_NINETH_SKILL`.
2. Open the Skill Color UI and identify every generated color button.
3. Change the color for the ninth-skill-visible slot.
4. **Expected after fix:** every UI-exposed slot is accepted by the server and persists/reloads correctly.
5. Verify no slot outside the canonical matrix is accepted.
