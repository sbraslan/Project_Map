# switchbot

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Switchbot item lifecycle / event / P2P haritası tamamlandı

### Bu tur doğrulanan zincirler
- Inventory ↔ SWITCHBOT item move
- `SetItem(SWITCHBOT)` → Register/Unregister
- DB persistence: `owner_id=playerID, window=SWITCHBOT, pos=slot`
- login ItemLoad → AddToCharacter → RegisterItem
- START/STOP variable packet parser
- active-slot movement/use protections
- 0.2s switch event
- attribute completion / resource consumption
- inter-core warp transfer
- EnterGame resume

### Yeni bulgular
- **BUG-SWITCHBOT-001:** inter-core warp source tarafında `CSwitchbot*` map'ten erase ediliyor fakat delete edilmiyor.
- **BUG-SWITCHBOT-002:** manager `Initialize()` yalnız raw-pointer map'i `clear()` ediyor; destructor da aynı yolu kullanıyor, owned objects delete edilmiyor.
- **BUG-SWITCHBOT-003:** normal logout/disconnect'te manager cleanup yok. PID'ye ait Switchbot object/event runtime'da kalabiliyor.
- **BUG-SWITCHBOT-004:** START server-side slot item varlığını doğrulamıyor. Empty/stale slot active yapılıp 0.2s event sonsuza kadar dönmeye bırakılabiliyor.
- `TSwitchbotUpdateItem.vnum` server/client'ta `uint8_t`; ancak mevcut client receiver bu alanı kullanmıyor. Şimdilik observation.

### Sıradaki
- Special Inventory type/range modeli
- sonra Item subsystem için genel completion checkpoint.

## Checkpoint — Special Inventory + Switchbot lifecycle

### Special Inventory statik haritası kapatıldı
Special Inventory ayrı bir item window değildir; `INVENTORY` window içinde üç hücre aralığıdır:
- Skillbook
- Stone
- Material

`TItemPos::IsSpecialInventoryPosition()` yalnız INVENTORY + special range kontrolü yapar.
`TItemPos::GetSpecialInventoryType()` hücre aralığından tipi türetir.

Item tarafı:
- `ITEM_SKILLBOOK` → Skillbook
- `ITEM_METIN` → Stone
- `ITEM_MATERIAL` / `ITEM_RESOURCE` → Material
- VNUM 27987 ayrıca Material olarak zorlanır.

`IsEmptySpecialItemGrid` size > 1 itemları reddeder.
`GetEmptyInventory(item)` special item için yalnız kendi special range'ini tarar.
`MoveItem` source INVENTORY olduğunda item special type ile destination special type eşleşmesini zorlar.

### Switchbot move/save lifecycle kapatıldı
Slot sayısı: 7.

Normal UI:
`root/uiswitchbot.py`
→ item INVENTORY ↔ SWITCHBOT için normal `SendItemMovePacket`
→ boş slot veya attribute konfigürasyonu yoksa Start düğmesi disable.

Server:
`MoveItem`
→ active Switchbot source item hareketini reddeder
→ destination SWITCHBOT ise yalnız weapon/armor ve destek açıksa uygun costume tipleri kabul edilir
→ `SetItem(SWITCHBOT,...)`
→ `CSwitchbotManager::RegisterItem/UnregisterItem`.

Persistence:
SWITCHBOT itemları normal player item row'u olarak `window='SWITCHBOT'` ile kaydedilir.
Login item query SWITCHBOT window'u yükler.
`ItemLoad` → `AddToCharacter(SWITCHBOT,pos)` → manager yeniden register edilir.

### Switchbot runtime
Python/C++:
`switchbot.Start(slot)`
→ `HEADER_CG_SWITCHBOT` / START
→ server `CInputMain::Switchbot`
→ `CSwitchbotManager::Start`
→ event
→ `SwitchItems`
→ attribute change
→ özel Switchbot update packet.

Cross-core warp:
`WarpSet`
→ manager warping=true
→ `P2PSendSwitchbot`
→ event Pause
→ table `HEADER_GG_SWITCHBOT` ile target core'a taşınır
→ target `P2PReceiveSwitchbot`
→ EnterGame sırasında event gerekiyorsa yeniden başlatılır.

### Yeni bug / adaylar
- **BUG-SWITCHBOT-001:** `P2PSendSwitchbot` raw `CSwitchbot*` pointer'ını map'ten erase ediyor fakat delete etmiyor → her cross-core transferde memory leak.
- **BUG-SWITCHBOT-004:** server START slotta gerçek item bulunduğunu doğrulamıyor. Normal UI boş slotu engelliyor; custom/malformed client yolu runtime resource testine açık.
- **OBS-SWITCHBOT-002:** client Python binding slot kontrolü `bSlot > SWITCHBOT_SLOT_COUNT`; eşit değer client katmanından geçse de server `ValidPosition` tarafından reddediliyor.

### Sıradaki
- Switchbot cross-core runtime testi
- Additional Equipment SwapItem runtime etkisi
- Item subsystem genel checkpoint ve kalan internal AddToCharacter caller taraması.

## Related
- Bugs: `../bugs/switchbot.md`
- Runtime tests: `../tests/switchbot.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
