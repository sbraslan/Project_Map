# ACH-T03 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-ACH-006`  
**Canonical test:** `ACH-T03`

> Handoff only. It does not authorize runtime execution.

## Static basis

Current Achievement task enum defines:
- `TYPE_SUMMON_MOUNT = 5`
- `TYPE_SPEND_SEARCH_SHOP = 19`
- `TYPE_SPEND_SHOP = 20`
- `TYPE_WITHDRAW = 22`

Current `achievements.xml` contains live tasks for:
- TYPE_SUMMON_MOUNT: **16**
- TYPE_SPEND_SEARCH_SHOP: **5**
- TYPE_SPEND_SHOP: **3**
- TYPE_WITHDRAW: **5**

Mapped gameplay callers exist for many other task families, but no mapped caller invokes Achievement progression for these four families.

Examples of nearby working families:
- PetSystem -> `OnSummon(...TYPE_SUMMON_PET...)`
- refine fee -> `OnGoldChange(...TYPE_SPEND_UPGRADE...)`
- gold collect -> `Collect(...TYPE_COLLECT_GOLD...)`

The mapped horse/mount, normal shop, private-shop-search spending and safebox-withdraw flows contain no equivalent calls for the four missing task types.

## Future live action

When runtime is explicitly unlocked:
1. choose one unfinished configured achievement from each available family;
2. record its current progress;
3. perform the corresponding legitimate gameplay action through normal UI/gameplay;
4. reopen/refresh Achievement UI;
5. record whether task progress changed;
6. stop after observation.

Prefer one family at a time. Do not modify packets, quest flags or XML.

## Expected result

Legitimate gameplay action succeeds, but the corresponding Achievement task progress remains unchanged because no gameplay hook forwards the event into the Achievement system.

## Result classification

**REPRODUCED:** legitimate action occurs and matching configured task remains unchanged.  
**NOT REPRODUCED:** mapped-equivalent deployment contains a real caller and progress updates.  
**INCONCLUSIVE:** task is already complete, feature disabled, action unavailable, or deployment differs.

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
