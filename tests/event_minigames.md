# Event MiniGames — Test Plan

## EM-001 — Okey event-active gate
1. Enable `mini_game_okey_event`, give a character the required card-set item and Yang, and verify a normal game can start.
2. End/disable the event through the normal event manager path.
3. Keep at least one Okey card-set item on the character.
4. Send `HEADER_CG_OKEY_CARD / SUBHEADER_CG_RUMI_START` directly.
5. **Expected after fix:** server rejects the request; no Yang/item is consumed and no Okey game is opened.
6. Send DECK/HAND/FIELD/DESTROY Okey subheaders while the event is disabled.
7. **Expected after fix:** no reward-producing state mutation is accepted unless an explicit server policy permits completion of a game that was already running before event shutdown.
8. Re-enable the event and confirm normal gameplay still works.

## Cursor-1 packet validation coverage
- Okey: variable subpacket lengths are checked before payload casts; unknown subheaders do not mutate state.
- Catch King: variable subpacket lengths are checked before payload casts; active handlers gate on `mini_game_catchking_event`.
- FindM: fixed request struct size is checked; active handlers gate on `mini_game_findm_event`.
- BNW: fixed request struct size is checked; active handlers gate on `mini_game_bnw_event`.


## EM-002 — Okey state initialization
1. Construct/login a fresh character without opening Okey previously.
2. Before any Okey request, inspect/verify server-side Okey state.
3. **Expected after fix:** `cards_left == 0`, points/field_points are 0, all hand/field/randomized card entries are zero.
4. Send Okey START while event is active with valid Yang + card set.
5. Verify the server always enters the initialization/payment branch exactly once for a fresh game.
6. Disconnect/relogin and repeat; state must again start from a deterministic zero state unless persistence is explicitly added.


## EM-003 — Catch King no-reward terminal cleanup
1. Start Catch King normally.
2. Complete a run with final score below 10.
3. Send the normal reward/end request.
4. **Expected after fix:** no reward item is granted, but score/bet/field runtime state is cleared and `gameStatus == false`.
5. Immediately start another Catch King round.
6. **Expected after fix:** new round starts normally without requiring relog.
7. Repeat with a score >= 10 and verify reward + cleanup behavior remains unchanged.
