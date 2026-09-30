# Skill Color — Verified Bugs

## SKCLR-001 — Ninth-skill UI exposes slot 9 but server rejects it

**Status:** VERIFIED_STATIC  
**Severity:** Medium  
**Affected:** Client/Server Skill Color slot-domain parity

### Evidence
- With `ENABLE_NINETH_SKILL`, client `MAX_SKILL_COUNT = 9`.
- Character UI iterates `skillSlot < 10` and creates a Skill Color button for slot 9 (while only excluding slots 6 and 7).
- The Python binding sends that slot directly as `uint8_t`.
- Server `SetSkillColor()` rejects any `p->skill >= MAX_SKILL_COUNT`.
- Therefore slot 9 is rejected because `9 >= 9`.

### Reachable consequence
The visible ninth-skill color control can send a valid-looking request that the server silently discards, so that skill's color cannot be persisted/applied through the exposed UI.

### Fix boundary
Use one canonical slot-domain definition across UI/client/server. Either valid slots are 0..8 and the UI must not expose 9, or the matrix/count must include slot 9 and all packet/storage bounds must be updated consistently.

### Regression target
See `tests/skill_color.md#skclr-001`.
