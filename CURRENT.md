# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Sung Mahi Tower
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/sung_mahi_tower.md`
**Verified bug registry:** `bugs/sung_mahi_tower.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just mapped
- Map 386/387 have no monster regen files; tower population is quest/runtime-spawned.
- Direct Sung Mahi groups 6077..6088 are present and every referenced member exists in both mob_proto and client npclist.
- Primary `smhtower_*` / `smhgate_boss` server motion folders have no missing motlist -> MSA references.
- Special tower proto ranges 7592–7600, 7609–7614 and 7615–7620 are present; room placement is blocked by the missing quest runtime.
- `smhgate_flower` folder for proto 9100–9107 is absent, but no tracked producer/reference proves runtime use; deferred only.
- Verified `BUG-SMT-005`: dark king 7591 has `ResistDark=-1` while the symmetric elemental design, its own dark family, and higher dark king use `-30`.
- Verified bugs: `BUG-SMT-001` through `BUG-SMT-005`.

## Exact next work
1. Run one final independent Sung Mahi static sweep.
2. If no unexamined independent path remains, mark Sung Mahi Tower STATIC COMPLETE while explicitly retaining BUG-SMT-001 as the reason runtime quest behavior cannot be reconstructed.
3. Preserve deferred candidates without promotion unless a producer/schema is recovered.
4. No production source changes.

GitHub state is canonical.
