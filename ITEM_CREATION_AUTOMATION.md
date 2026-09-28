# Item Creation Automation — Template Rules

**Prepared:** 2026-09-28  
**Phase:** Design / documentation only  
**Source repositories:** read-only  
**Purpose:** Canonical rules for future automated item_proto generation.

## Core principle

The user provides semantic intent, for example:
- "tek el kılıç"
- "çift el savaşçı silahı"
- "hançer"
- "yay"
- "zırh"
- "zırh kostümü"
- "saç kostümü"
- "silah kostümü"

The automation must not invent a proto layout from scratch when a compatible existing class exists.

For every new item:
1. identify the requested class;
2. select an existing same-class item as the structural reference;
3. copy only structural/default fields appropriate to that class;
4. replace VNUM/name/effects/limits/visual mapping requested by the user;
5. preserve class-specific restrictions and VALUE semantics;
6. validate that no field is accidentally reused for a different subsystem purpose.

## Canonical proto field groups

Current build/parser supports:
- TYPE
- SUBTYPE
- SIZE
- ANTI_FLAG
- FLAG
- WEAR_FLAG
- IMMUNE
- SHOP_BUY_PRICE / SHOP_SELL_PRICE
- LIMIT_TYPE0 / LIMIT_VALUE0
- LIMIT_TYPE1 / LIMIT_VALUE1
- APPLYTYPE0 / APPLYVALUE0
- APPLYTYPE1 / APPLYVALUE1
- APPLYTYPE2 / APPLYVALUE2
- APPLYTYPE3 / APPLYVALUE3
- VALUE0..VALUE5
- sockets
- refine/refine_set
- magic/specular/socket-related fields
- MASK_TYPE / MASK_SUBTYPE

Current build has `ITEM_APPLY_MAX_NUM = 4`.

## Class map

### Weapon
TYPE = ITEM_WEAPON

Canonical subtypes:
- WEAPON_SWORD
- WEAPON_DAGGER
- WEAPON_BOW
- WEAPON_TWO_HANDED
- WEAPON_BELL
- WEAPON_FAN
- WEAPON_ARROW
- WEAPON_MOUNT_SPEAR
- WEAPON_CLAW
- WEAPON_QUIVER
- WEAPON_BOUQUET

Default wearable slot for normal weapon classes:
- WEAR_FLAG includes WEAR_WEAPON.

Reference selection rule:
- exact subtype first;
- intended allowed jobs/races second;
- comparable level/refine/value layout third.

Example:
- user says "çift el savaşçı silahı" -> use an existing `ITEM_WEAPON / WEAPON_TWO_HANDED` item with Warrior-only restrictions as the reference, not a sword or costume weapon.

### Armor
TYPE = ITEM_ARMOR

Canonical subtypes:
- ARMOR_BODY
- ARMOR_HEAD
- ARMOR_SHIELD
- ARMOR_WRIST
- ARMOR_FOOTS
- ARMOR_NECK
- ARMOR_EAR
- ARMOR_PENDANT
- ARMOR_GLOVE

Reference selection uses exact subtype and intended job restrictions.

### Costume
TYPE = ITEM_COSTUME

Canonical subtypes:
- COSTUME_BODY
- COSTUME_HAIR
- COSTUME_MOUNT
- COSTUME_ACCE
- COSTUME_WEAPON
- COSTUME_AURA

Costume equip slot is selected by subtype in server code; normal costume entries do not require a conventional equipment WEAR_FLAG to determine their costume slot.

#### COSTUME_BODY
Visual chain:
`item VNUM -> client SetArmor(VNUM) -> item_proto VALUE3 -> MSM ShapeIndex -> GR2/DDS`.

Important:
- VALUE3 is not an APPLY/bonus slot.
- APPLY0..3 remain available independently.
- dynamic item attributes remain independent from VALUE3.
- same ShapeIndex should be present in every supported character MSM that needs that costume, each pointing to the race/sex-specific GR2/DDS.

#### COSTUME_HAIR
Server directly uses item proto VALUE3 as the hair shape value when equipped.

#### COSTUME_WEAPON
VALUE3 is already used for weapon subtype compatibility in the current build. Do not treat COSTUME_WEAPON VALUE3 as body-costume ShapeIndex semantics.

## ANTI_FLAG / race-job-sex restriction rules

Current parser supports:
- ANTI_FEMALE
- ANTI_MALE
- ANTI_MUSA
- ANTI_ASSASSIN
- ANTI_SURA
- ANTI_MUDANG
- ANTI_WOLFMAN
- ANTI_GET
- ANTI_DROP
- ANTI_SELL
- ANTI_EMPIRE_A
- ANTI_EMPIRE_B
- ANTI_EMPIRE_C
- ANTI_SAVE
- ANTI_GIVE
- ANTI_PKDROP
- ANTI_STACK
- ANTI_MYSHOP
- ANTI_SAFEBOX
- ANTI_RT_REMOVE
- ANTI_QUICKSLOT
- ANTI_CHANGELOOK
- ANTI_REINFORCE
- ANTI_ENCHANT
- ANTI_ENERGY
- ANTI_PETFEED
- ANTI_APPLY
- ANTI_ACCE
- ANTI_MAIL

Interpretation rule:
ANTI_* means **disallowed**.

Examples:
- Warrior-only item: block Assassin + Sura + Shaman + Wolfman; do not set ANTI_MUSA.
- Wolfman-only item: block Warrior + Assassin + Sura + Shaman; do not set ANTI_WOLFMAN.
- male-only item: ANTI_FEMALE.
- female-only item: ANTI_MALE.

Do not hardcode job restrictions solely from the spoken item name if an exact same-subtype reference exists. Prefer copying the intended reference item's anti-flag model and then applying explicit user overrides.

## FLAG rules

Available parser flags include:
- ITEM_TUNABLE
- ITEM_SAVE
- ITEM_STACKABLE
- COUNT_PER_1GOLD
- ITEM_SLOW_QUERY
- ITEM_UNIQUE
- ITEM_MAKECOUNT
- ITEM_IRREMOVABLE
- CONFIRM_WHEN_USE
- QUEST_USE
- QUEST_USE_MULTIPLE
- QUEST_GIVE
- LOG
- ITEM_APPLICABLE
- ITEM_GROUP_DMG_WEAPON

Automation rule:
- copy class/reference defaults unless the user explicitly requests special behavior;
- never add stackability/refineability/irremovable behavior merely by guess.

## WEAR_FLAG rules

Parser supports:
- WEAR_BODY
- WEAR_HEAD
- WEAR_FOOTS
- WEAR_WRIST
- WEAR_WEAPON
- WEAR_NECK
- WEAR_EAR
- WEAR_SHIELD
- WEAR_UNIQUE
- WEAR_ARROW
- WEAR_HAIR
- WEAR_ABILITY
- WEAR_PENDANT
- WEAR_GLOVE
- WEAR_PET (when enabled)

Normal equipment uses these flags.
Costume slots are primarily selected from COSTUME subtype in server code.

## LIMIT rules

Two independent limit slots are available:
- LIMIT_TYPE0 / LIMIT_VALUE0
- LIMIT_TYPE1 / LIMIT_VALUE1

Supported types include:
- LIMIT_NONE
- LEVEL
- STR
- DEX
- INT
- CON
- REAL_TIME
- REAL_TIME_FIRST_USE
- TIMER_BASED_ON_WEAR
- NEWWORLD_LEVEL
- DURATION

Automation rule:
- copy a same-class reference limit layout unless user specifies level/duration behavior;
- time limits must not be guessed because REAL_TIME, FIRST_USE and TIMER_BASED_ON_WEAR have different semantics.

## Fixed proto APPLY rules

Current build supports 4 fixed proto APPLY slots:
- APPLY0
- APPLY1
- APPLY2
- APPLY3

These are independent from:
- VALUE0..5;
- MSM ShapeIndex;
- dynamic item attributes.

For COSTUME_BODY, VALUE3 remains available for ShapeIndex even when all 4 fixed APPLY slots are used.

## Dynamic attribute capacity reminder

Current item storage:
- 5 normal attribute slots;
- 2 rare slots;
- total 7 physical attribute slots.

Current standard costume attribute flow normally generates up to 3 dynamic costume attributes through `AlterToMagicItem()` probabilities; normal generic add/rare flows reject ITEM_COSTUME.

Do not confuse dynamic attributes with fixed proto APPLY0..3.

## Reference-template strategy

When asked to create an item, future automation should resolve a template in this order:

1. exact TYPE + SUBTYPE;
2. same intended job/race restriction pattern;
3. same gender restriction pattern where applicable;
4. same timer/limit model;
5. same refineability/equipment behavior;
6. closest current production item.

Examples:
- "kılıç" -> ITEM_WEAPON / WEAPON_SWORD reference.
- "çift el savaşçı silahı" -> ITEM_WEAPON / WEAPON_TWO_HANDED Warrior-usable reference.
- "zırh" -> ITEM_ARMOR / ARMOR_BODY reference matching job.
- "zırh kostümü" -> ITEM_COSTUME / COSTUME_BODY reference, plus VALUE3/MSM ShapeIndex workflow.
- "saç kostümü" -> ITEM_COSTUME / COSTUME_HAIR reference, VALUE3 hair-index workflow.
- "silah kostümü" -> ITEM_COSTUME / COSTUME_WEAPON reference; preserve subtype-compatibility VALUE3 semantics.

## Future automation safety checks

Before emitting a proto row:
- VNUM must be unused in the intended namespace.
- TYPE/SUBTYPE combination must be valid.
- ANTI_FLAG must match intended allowed jobs/sex.
- WEAR_FLAG must match class semantics.
- LIMIT slots must match requested lifetime/level behavior.
- APPLY count must be <= 4.
- VALUE fields must preserve subtype-specific semantics.
- COSTUME_BODY VALUE3 must match a valid ShapeIndex in required MSM files.
- COSTUME_WEAPON VALUE3 must not be overwritten with body ShapeIndex semantics.
- MASK_TYPE/MASK_SUBTYPE must match the visible item category.
- reference item provenance should be recorded so generated rows are auditable.

## Current state

This is a design rule set only. No item_proto, MSM, source, locale or game file has been modified.
