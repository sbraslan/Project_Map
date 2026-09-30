# Meley / Red Dragon Lair — Dormant Source Classification

**Status:** DORMANT / COMPILED FEATURE WITHOUT TRACKED DEPLOY CALLER  
**Mode:** discovery classification only  
**Execution:** NOT REACHABLE FROM TRACKED QUEST DEPLOYMENT  
**Source policy:** source repositories remain read-only.

## Why this is not an OPEN canonical node
The pinned ServerSRC snapshot compiles `ENABLE_GUILD_DRAGONLAIR_SYSTEM` and `ENABLE_GUILD_DRAGONLAIR_PARTY_SYSTEM`, including:
- `MeleyLair.cpp/.h`;
- `questlua_meley_lair.cpp`;
- login/reconnect, party/guild, combat and client time-result hooks.

However the pinned Game deployment contains:
- no Meley/Red Dragon/Guild Dragon quest in `quest_list`;
- no Meley quest source under the quest tree;
- no Meley state object under `quest/object/state`;
- no primary quest/object handlers for Meley NPC 20419, reward NPC 20420, party chest 20501, guild/party stones 20422/20500.

The only tracked object under 20421 is an unrelated `cube_opener_list` take handler.

Therefore the manager APIs that actually create/register runs — `RegisterGuild`, `EnterGuild`, `EnterParty` — are exposed through the Lua binding but have no tracked deployed quest caller in this source snapshot.

## Runtime hooks that remain compiled
Generic server code still contains Meley guards for:
- reconnect into a pre-existing private 356xxxx map;
- party leave restrictions while `GetMeleyLair()` is set;
- guild-member removal;
- statue/boss combat callbacks;
- Meley time/result packets.

These hooks do not create a run on their own.

## Latent source defect — not promoted as an active bug
`CMeleyLair::EndDungeonWarp()` calls `ClearGuild(GetGuild())` or `ClearParty(GetParty())`.

Both manager cleanup functions use:
- `m_RegGuilds.erase(iter, m_RegGuilds.end())`;
- `m_RegPartys.erase(iter, m_RegPartys.end())`.

That erases the matched registration **and every registration after it**, not only the target entry. If Meley deployment is re-enabled with concurrent runs, ending one run can remove unrelated later guild/party registrations from the manager.

This is retained as a dormant-source defect only because no tracked deployment caller can start Meley in the pinned Game snapshot.

## Reopen rule
Reopen Meley as a canonical audit only if:
1. a Meley quest/object entry caller is added/restored;
2. a non-quest caller to `RegisterGuild` / `EnterGuild` / `EnterParty` is proven; or
3. relevant source SHAs change and impact discovery invalidates this classification.
