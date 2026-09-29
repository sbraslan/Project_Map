# Zodiac Temple / 12ZI — Bug Registry

**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

No bug is promoted solely from a suspicious line. Promotion requires a complete reachable source path or a deterministic data/lifetime proof.

## Active candidates under verification
- `CZodiacManager::Initialize()` has a non-channel-99 control path with no explicit bool return; caller/use semantics must be checked before promotion.
- Zodiac reward/check-box helpers create temporary item objects; ownership/destruction must be proven before classifying a leak.
- Player commands `/cz_check_box`, `/cz_reward`, `/jumpfloor`, `/nextfloor` need server-side state/authorization review.
- event structs store raw `LPCHARACTER` in Zodiac combat paths; destruction/cancellation lifecycle must be mapped.

## Verified bugs
None yet.
