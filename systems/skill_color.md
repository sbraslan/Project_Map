# Skill Color — Static System Map

**Status:** STATIC MAPPING CLOSED  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source snapshot:** pinned by `MAP_STATE.json`

## Canonical scope
Skill Color owns a distinct networked/persistent customization lifecycle:
- dedicated `HEADER_CG_SKILL_COLOR`;
- per-character skill-color matrix;
- server validation and mutation;
- immediate DB save via `HEADER_GD_SKILL_COLOR_SAVE`;
- character spawn/update propagation of color data;
- client/Python edit UI and packet path.

## Deployment proof
- `ENABLE_SKILL_COLOR_SYSTEM` is enabled.
- server exposes `CInputMain::SetSkillColor()`;
- character packets include `dwSkillColor`;
- client has dedicated send packet and Python binding;
- DB save packet exists for the matrix.

## Boundary
Owned here:
- skill-color packet validation;
- skill/buff slot index domain;
- color matrix persistence;
- client/server matrix parity.

Not owned here:
- generic skill mechanics;
- ChangeLook / costume appearance (already CLOSED under `costume_appearance`);
- passive-skill progression unless it independently owns lifecycle/state.

## Audit cursor
1. Skill/buff slot index domain and packet validation parity.
2. DB save/load persistence and matrix sizing.
3. Character add/update propagation and client bounds.
4. Close subsystem and resume remaining appearance discovery.


## Closeout
- Slot-domain parity, DB save/load sizing and character packet propagation were audited.
- Final verified bugs: 1.
- Final test plans: 1.
- No additional persistence/propagation defect was promoted.
- Lifecycle is CLOSED and locked on the pinned source snapshot.
