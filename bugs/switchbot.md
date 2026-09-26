# switchbot — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-SWITCHBOT-001 — Inter-core warp CSwitchbot memory leak
- Statik durum: **çok yüksek güven / doğrudan ownership leak**

`P2PSendSwitchbot`:
1. raw pointer `pkSwitchbot = FindSwitchbot(pid)`
2. `Pause()`
3. `m_map_Switchbots.erase(pid)`
4. table kopyalanıp P2P gönderilir
5. fonksiyon biter

`delete pkSwitchbot` yok.
Map raw pointer taşıdığı için erase ownership'i serbest bırakmıyor.

Her cross-core transfer source core'da bir `CSwitchbot` allocation sızdırabilir.

### BUG-SWITCHBOT-002 — Manager Initialize/destructor raw-pointer leak
- Statik durum: **doğrulandı**

Manager objectleri `new CSwitchbot()` ile allocate ediyor.

`CSwitchbotManager::Initialize()` yalnız:
`m_map_Switchbots.clear()`

Destructor:
`~CSwitchbotManager() { Initialize(); }`

Map içindeki raw pointerlar delete edilmiyor.
Process kapanışında OS memory reclaim olsa da graceful reinitialize/test/reload yollarında destructor ownership modeli hatalı; ayrıca CSwitchbot destructor event cancellation'ı da çalışmıyor.

### BUG-SWITCHBOT-003 — Normal logout sonrası Switchbot manager entry/event cleanup yok
- Statik durum: **yüksek güven**

`CHARACTER::Disconnect` içinde Switchbot:
- Stop
- Pause
- Remove
- erase
- delete

çağrılarından hiçbiri yok.

Manager entry PID bazında runtime map'te kalıyor.

Eğer active state stale kalırsa:
`switchbot_event`
→ her 0.2s `SwitchItems`
→ item ID artık ITEM_MANAGER'da yok
→ `continue`
→ event tekrar schedule edilir.

Sonuç:
Uzun uptime'da Switchbot kullanmış/offline olmuş PID'ler için memory ve potansiyel recurring-event birikimi.

### BUG-SWITCHBOT-004 — START empty/stale slotu active event'e çevirebiliyor
- Statik durum: **doğrulandı**
- Trigger: normal UI engelliyor; custom/malformed client packet gerekir.

Server START:
- slot range var
- manager var
- already-active check var
- fakat registered item existence check yok.

Daha önce Switchbot manager oluşturmuş oyuncu boş slot için START gönderebilir:
`items[slot] == 0`
→ active=true
→ event starts
→ `ITEM_MANAGER::Find(0)` null
→ continue
→ active true kalır
→ event her 0.2s tekrar çalışır.

Aynı durum stale/nonexistent item ID için de geçerli.

Logout cleanup eksikliğiyle birleştiğinde recurring background event oyuncu çıktıktan sonra da kalabilir.

### OBS-SWITCHBOT-001 — UPDATE_ITEM vnum 8-bit
Server/client `TSwitchbotUpdateItem.vnum` alanı uint8_t iken item VNUM 32-bit.
Server assignment truncation yapıyor.

Ancak mevcut client receiver bu alanı kullanmıyor; item index normal ITEM_SET state'inden geliyor.
Şu an için kullanıcı-visible bug olarak sınıflandırılmadı, ileride bu alan kullanılmaya başlanırsa protokol hatasına dönüşür.

### BUG-SWITCHBOT-005 — UnregisterItem running event'i sonlandırmıyor
- Statik durum: **doğrulandı**
- BUG-SWITCHBOT-003 logout/event leak için kesin mekanizmalardan biri

`CSwitchbot::UnregisterItem` slotun:
- item ID
- active
- finished
- alternatives
state'ini temizler.

Fakat `CSwitchbotManager::UnregisterItem`:
`!HasActiveSlots() && IsSwitching()`
durumunda Stop çağırmıyor.

Son active item MoveItem dışı bir yolla kaldırılırsa running event kalır.

`switchbot_event`:
→ `SwitchItems()`
→ active slot yoksa iş yapmadan döner
→ yine `PASSES_PER_SEC(0.2f)` döndürerek schedule olur.

Normal logout sırasında CHARACTER destructor → `ClearItem()` → SWITCHBOT item `RemoveFromCharacter` → UnregisterItem zinciri bulunduğundan bu kusur custom packet'e bağımlı değildir.

### OBS-SWITCHBOT-002 — Client slot upper-bound off-by-one
`PythonSwitchbot.cpp` Start/Stop bindingleri slot guard olarak:
`if (bSlot > SWITCHBOT_SLOT_COUNT)`
kullanıyor.

Bu nedenle `slot == SWITCHBOT_SLOT_COUNT` client binding katmanından geçebilir.
Server tarafındaki `ValidPosition(slot) -> slot < SWITCHBOT_SLOT_COUNT` bunu reddettiği için mevcut server state korunuyor.

Doğru client-local sınır semantiği `>=` olmalı; şimdilik server tarafından absorbe edilen boundary observation.
