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
