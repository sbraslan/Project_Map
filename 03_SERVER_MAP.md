# 03 — Server Map

Server tarafındaki handler, karakter, item, guild ve servis akışları burada tutulur.

## Kayıt formatı

### Sistem
- Packet:
- Entry handler:
- Doğrulamalar:
- Business logic:
- Veri değişimi:
- DB çağrısı:
- Response:
- Edge cases:

## Öncelikli alan
- Guild Storage checkin
- Guild Storage checkout
- Slot / item validation
- Ownership / permission kontrolleri
- Persistence
- Logout / reconnect davranışı

## Guild Storage — doğrulanan server zinciri

### Entry
`CInputMain::Analyze` içinde:
- `HEADER_CG_GUILDSTORAGE_CHECKIN` → `SafeboxCheckin(ch, c_pData, 2)`
- `HEADER_CG_GUILDSTORAGE_CHECKOUT` → `SafeboxCheckout(ch, c_pData, 2)`

Bu sistem ayrı bir storage handler yerine safebox altyapısını `bMall == 2` modu ile yeniden kullanıyor.

### Checkin doğrulamaları
`CInputMain::SafeboxCheckin`:
- character null kontrolü
- GM block kontrolü
- aktif quest kontrolü
- `CanHandleItem()`
- `GetGuildstorage()` varlığı
- kaynak item varlığı
- irremovable / equipped kontrolleri
- hedef slot boşluğu: `pkSafebox->IsEmpty(...)`
- safebox-expand item engeli
- `ITEM_ANTIFLAG_SAFEBOX`
- item lock
- soulbind/seal
- basic item engeli
- belt inventory bağımlılığı

Başarılı işlem:
`RemoveFromCharacter()`
→ quickslot sync
→ `pkSafebox->Add(...)`
→ item log
→ guild log (Guild Renewal aktifse)

### Checkout doğrulamaları
`CInputMain::SafeboxCheckout`:
- character null
- GM block
- `CanHandleItem()`
- `GetGuildstorage()`
- storage slotunda item varlığı
- hedef inventory grid boşluğu
- Dragon Soul hedef alan doğrulaması
- belt inventory doğrulaması
- special inventory type doğrulaması

Başarılı işlem:
`pkSafebox->Remove(...)`
→ `pkItem->AddToCharacter(...)`
→ `ITEM_MANAGER::FlushDelayedSave(pkItem)`
→ `HEADER_GD_ITEM_FLUSH`
→ log / guild log

Guild Renewal aktifse ayrıca son checkout bilgisi güncellenip P2P refresh gönderiliyor.

## Guild Storage — open / lock / close

### Open command
`cmd.cpp`
- `click_guildstorage` → `do_click_guildstorage`

`cmd_general.cpp::do_click_guildstorage`
→ `SetGuildstorageOpenPosition()`
→ `ReqGuildstorageLoad()`

### `CHARACTER::ReqGuildstorageLoad()`
Doğrulamalar:
- guild pointer
- guild storage satın alınmış/seviye > 0
- exchange/shop/mailbox/change-look/safebox/cube çakışması
- `ENABLE_GUILDRENEWAL_SYSTEM` aktifse:
  - guild member alınır
  - `GUILD_AUTH_BANK` yetkisi kontrol edilir
- `pGuild->IsStorageOpen()`
- growth-pet window
- mevcut `GetGuildstorage()`
- 1 saniyelik pulse/rate limit
- açılış noktasına mesafe ≤ 1000
- `m_bOpeningGuildstorage` overlap kontrolü

Başarılı request:
`HEADER_GD_GUILDSTORAGE_LOAD`
→ DB
→ **hemen ardından**
`pGuild->SetStorageState(true, GetPlayerID())`

### Load response
`CInputDB::GuildstorageLoad`
- response guild ID, karakterin güncel guild ID'si ile eşleştiriliyor
- çakışan pencere kontrolü tekrar yapılıyor
- guild storage boyutu guild objesinden alınıyor
- `LoadGuildstorage(...)` çağrılıyor

`LoadGuildstorage`:
- `SetOpenGuildstorage(true)`
- `CSafebox` oluştur/değiştir
- window mode = `GUILDBANK`
- `HEADER_GC_GUILDSTORAGE_OPEN`
- DB'den gelen item'lar local storage'a ekleniyor

### Close
Client:
`/guildstorage_close`

Server:
`do_guildstorage_close`
→ `CloseGuildstorage()`
→ `ch->Save()`
→ tekrar `SetStorageState(false,0)`

`CloseGuildstorage()` zaten kendi içinde de:
- `SetOpenGuildstorage(false)`
- `GetGuild()->SetStorageState(false,0)`
- storage save (SAFEBOX_MONEY build'inde)
- object delete
- client `CloseGuildstorage` command
- load-time reset

Not: close komutunda state reset iki kez çağrılıyor.
