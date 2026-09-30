# Event MiniGames — Verified Bugs

## EM-001 — Okey/Rumi can be started and played while the event flag is disabled

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Okey-Rumi

### Evidence
- `CEventManager::SetMiniGameOkeyEvent()` controls the event through `mini_game_okey_event`.
- `CInputMain::MiniGameOkeyCard()` dispatches START/EXIT/DECK/HAND/FIELD/DESTROY requests without checking that flag.
- `CHARACTER::Cards_open()` validates trade/shop/safebox state, Yang and card-set inventory, but does not validate `mini_game_okey_event`.
- The other active minigames (Catch King, FindM, BNW) explicitly reject gameplay actions when their corresponding event flag is 0.

### Reachable path
```
client HEADER_CG_OKEY_CARD
  -> CInputMain::MiniGameOkeyCard()
  -> SUBHEADER_CG_RUMI_START
  -> CHARACTER::Cards_open()
  -> consume RUMI_PLAY_YANG + RUMI_PLAY_ITEM
  -> create/open a new Okey game
```

There is no server-side event-active gate on this path.

### Impact
A modified client can continue to start and operate Okey/Rumi games after the scheduled Okey event has been disabled, as long as the character still owns the required play item and Yang. This bypasses event scheduling and can keep reward-generation gameplay reachable outside the intended event window.

### Fix boundary
Add authoritative server-side `mini_game_okey_event` checks before starting or mutating Okey gameplay. The START path must be blocked at minimum; mutating subheaders should also be gated or an already-running game should have an explicitly documented end-of-event policy.

### Regression target
See `tests/event_minigames.md#em-001`.
