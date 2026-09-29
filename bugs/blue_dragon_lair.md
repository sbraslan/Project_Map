# Blue Dragon / Beran Setaou — Bug Registry

**Status:** STATIC MAPPING OPEN / 1 VERIFIED BUG
**Execution:** LOCKED / NOT RUN

## BUG-BDL-001 — access items are consumed before a disconnect-cancellable personal entry timer

**Class:** item transaction / disconnect rollback

### Proof
- Entry first calls `pc.remove_item(access_item, access_item_amount)`.
- The first entrant then schedules the personal timer `dragon_lair_warptimer` with a delay of `pc.get_channel_id() * 2`.
- Successful start/warp and the competing-start refund are both implemented only inside that timer callback.
- Character disconnect eventually calls `CQuestManager::DisconnectPC`, which erases the player's quest `PC` object.
- `PC::~PC -> Destroy -> ClearTimer` cancels all personal quest timers.
- The Blue Dragon quest has no logout/disconnect compensation for the already removed entry items.

### Consequence
Disconnecting or losing the connection during the delayed entry window can permanently consume the Blue Dragon access items without starting/joining the run and without refund.

### Deferred validation
`BDL-T01`.

