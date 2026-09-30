# Yohara Progression / Conqueror + Sungma — Static System Map

**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source policy:** Project_ClientSrc, Project_ServerSRC, Project_Binary, Project_Game and Project_DumpProto are read-only.

## Canonical scope
This node owns the shared Yohara progression layer:
- `ENABLE_YOHARA_SYSTEM`;
- Conqueror level, level-step, EXP and unspent Conqueror points;
- persistent Sungma STR / HP / MOVE / IMMUNE stats;
- client/server point synchronization;
- player-table load/save fields;
- Conqueror EXP generation/distribution;
- Sungma map attribute loading and generic combat/movement/HP penalties;
- Conqueror/Sungma item/stat application and reset logic.

## Boundaries
- Sung Mahi Tower is already CLOSED and is not reopened.
- Snake/Queen Nethis, White Dragon and other feature-specific Yohara dungeons remain separate candidate nodes; this audit only follows their shared Yohara dependency surface when necessary.
- Pure UI/art assets are dependencies unless they alter state or packet semantics.

## Deployment proof
- ServerSRC enables `ENABLE_YOHARA_SYSTEM`.
- Player tables persist conqueror level, step, EXP, Conqueror points and four Sungma stats.
- Character point packets expose Conqueror level/EXP/next EXP and Sungma-derived values.
- Multiple tracked maps contain `sungma_attr.txt`, and server character/combat code applies those map requirements.
- Binary contains live Sungma/Conqueror character UI assets and handlers.

## Audit cursor
1. Audit Conqueror EXP -> quarter-step -> level-up state machine, overflow recursion and point awards.
2. Audit DB load/save and client point synchronization for type/range/order mismatches.
3. Audit Sungma map-attribute loading, missing-map behavior and combat/movement/HP penalties.
4. Audit reset/stat-allocation/item application boundaries and overflow/negative paths.
5. Close deployment/data parity and promote only source-proven defects.
