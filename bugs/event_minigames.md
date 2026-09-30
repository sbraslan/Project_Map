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


## EM-002 — Okey/Rumi runtime state is not initialized in CHARACTER::Initialize

**Status:** VERIFIED_STATIC  
**Severity:** High  
**Affected:** ServerSRC / Okey-Rumi lifecycle

### Evidence
- `CHARACTER::CHARACTER()` calls `Initialize()`.
- `CHARACTER::Initialize()` explicitly initializes Catch King, BNW, FindM and YutNori runtime state.
- Under `ENABLE_MINI_GAME_OKEY_NORMAL`, `CHARACTER` owns:
  - `CARDS_INFO character_cards`;
  - `S_CARD randomized_cards[DECK_COUNT_MAX]`.
- `S_CARD` and `CARDS_INFO` are plain structs with no constructors or default member initializers.
- No Okey initialization for these members is present in `CHARACTER::Initialize()`.
- `Cards_open()` immediately reads `character_cards.cards_left` before calling `Cards_clean_list()`.

### Reachable consequence
A newly created `CHARACTER` may enter `Cards_open()` with indeterminate Okey state. If `cards_left` happens to be greater than zero, the normal initialization branch is skipped and the server proceeds using uninitialized hand/field/randomized-card data.

This can produce corrupted game state, invalid card counts/points, and nondeterministic behavior. It also undermines the intended payment/init boundary because the branch that consumes Yang + card set is conditional on an uninitialized value.

### Fix boundary
Initialize both Okey state blocks in `CHARACTER::Initialize()`, preferably by calling the same zeroing logic used by `Cards_clean_list()` or equivalent explicit zero initialization.

### Regression target
See `tests/event_minigames.md#em-002`.


## EM-003 — Catch King losing game can leave character permanently stuck in active game state

**Status:** VERIFIED_STATIC  
**Severity:** Medium-High  
**Affected:** ServerSRC / Catch King lifecycle

### Evidence
- `MiniGameCatchKingStartGame()` refuses to start if `MiniGameCatchKingGetGameStatus() == true`.
- At the end of a run, `MiniGameCatchKingGetReward()` requires hand card = 0 and hand-card-left = 0.
- Reward vnum is assigned only when score >= 10.
- State cleanup (`SetScore(0)`, `SetBetNumber(0)`, `SetGameStatus(false)`, field clear) happens only inside `if (dwRewardVnum)`.
- If the completed run has score < 10, `dwRewardVnum == 0`; the function only sets `bReturnCode = 1` and leaves the active-game state intact.

### Reachable consequence
A player who completes a Catch King round below the minimum reward threshold can remain with `gameStatus == true` after the game is over. Subsequent START requests are rejected by the existing active-game guard, so the character cannot start another Catch King round without a lifecycle reset such as relog/reconstruction.

### Fix boundary
End-of-round cleanup must be independent from whether a reward item exists. Reward delivery may remain conditional, but terminal game state must always be cleared once the run is finished.

### Regression target
See `tests/event_minigames.md#em-003`.


## EM-004 — Attendance login info dereferences an empty reward vector

**Status:** VERIFIED_STATIC  
**Severity:** Medium-High  
**Affected:** ServerSRC / Attendance

### Evidence
- `attendanceRewardVec` is a normal `std::vector<TRewardItem>` and therefore starts empty.
- `ReadRewardItemFile()` can return false when the reward file cannot be opened and does not establish a non-empty invariant.
- `CInputLogin` calls `CMiniGameManager::AttendanceEventInfo(ch)` for every login while `ENABLE_MONSTER_BACK` is compiled, independent of whether the attendance event is active.
- `AttendanceEventInfo()` sends:
  `Packet(&attendanceRewardVec[0], sizeof(TRewardItem) * attendanceRewardVec.size())`
  without checking `empty()`.
- Taking element 0 from an empty vector is invalid even when the transmitted byte count is zero.

### Reachable consequence
If the attendance reward list is missing, fails to load, or is otherwise empty, a normal player login reaches invalid vector element access. Depending on build/runtime behavior this can produce undefined behavior and may crash or destabilize the game process.

### Fix boundary
Never index element 0 when the vector is empty. Send the reward payload only when `!attendanceRewardVec.empty()`; also make startup/reload handling explicitly report and handle a missing/empty reward configuration.

### Regression target
See `tests/event_minigames.md#em-004`.
