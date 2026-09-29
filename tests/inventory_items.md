# Inventory / Item / Special Inventory — Deferred Runtime Tests

> Lifecycle authority: `../MAP_STATE.json`. This file contains Inventory/Item test evidence only.
> Runtime remains locked until the project phase is explicitly changed.

### ITEM-T01 — Destroy count semantics
- Stack count örneğin 50 olan test itemı kullan.
- Destroy packetini count=1 / 10 gibi partial değerlerle gönder.
- Sonuç inventory + DB'de gözlenir.

Beklenen API davranışı:
requested count kadar işlem veya count alanının açıkça ignored/full-delete olarak tasarlanması.

Mevcut statik kod:
tüm item stack objesini siliyor.

### ITEM-T02 — Destroy use-after-free
- Debug/ASan mümkünse aktif test game core.
- Destroy system üzerinden normal item yok et.
- `CHARACTER::RemoveItem` dönüşündeki ChatPacket/GetName yolunu izle.
- crash / sanitizer UAF raporu / bozuk isim kontrol et.

### ITEM-T03 — Drop AddToGround failure rollback
Kontrollü test ortamında `AddToGround` false yolu üret:
- invalid/olmayan sectree veya kontrollü instrumentation.

Full ve partial stack ayrı test edilir.

Beklenen:
item/count source'a geri dönmeli veya işlem false olmalı.

Mevcut statik kod:
rollback yok, fonksiyon true dönüyor.

### ITEM-T04 — Ground DB lifecycle
1. Benzersiz item ID'li itemı inventory'de doğrula.
2. Yere bırak.
3. DB row/cache durumunu kontrol et.
4. Item hâlâ world'de iken tekrar pickup yap.
5. DB row'un player owner/window ile yeniden oluştuğunu doğrula.
6. Ek olarak item yerdeyken game core crash/restart senaryosunu kontrollü test et.

Amaç:
ground itemların DB'den bilinçli olarak çıkarıldığını ve core memory'ye bağımlı olduğunu doğrulamak.

### ITEM-T05 — Destroy sender sequence davranışı
- Client network logging ile ardışık destroy + başka packet senaryoları test edilir.
- `SendItemDestroyPacket` sonrası `SendSequence` eksikliğinin packet dispatch üzerinde etkisi olup olmadığı ölçülür.

### ITEM-T06 — Invalid DB item position / AddToCharacter
**Yalnız kontrollü test DB'de.**

Ayrı testler:
- INVENTORY pos > valid max
- BELT_INVENTORY pos >= BELT_INVENTORY_SLOT_COUNT
- DRAGON_SOUL_INVENTORY pos >= DRAGON_SOUL_INVENTORY_MAX_NUM
- SWITCHBOT / ADDITIONAL invalid pos

Karakter login edilir.

Kontrol:
- core crash / ASan OOB
- item manager VID/ID map durumu
- owner pointer
- character slot arrays
- DB row'un sonraki save davranışı

Beklenen güvenli davranış:
invalid row karantinaya/restore listesine alınmalı; array indexing yapılmamalı.

### ITEM-T07 — Additional Equipment occupied-slot swap
`ENABLE_ADDITIONAL_EQUIPMENT_PAGE` aktif build.

- page 0 ve page 1 ayrı test
- aynı wear slotunda mevcut item varken başka item equip et
- swap sonrası iki itemın:
  - window
  - cell
  - equipped state
  - stat bonus
  - client görünümü
  - DB persistence
değerlerini kontrol et.

Amaç:
`SwapItem` shadowing için statik incelemede doğrudan runtime placement etkisi bulunmadı. Bu test artık sınıflandırma testi değil, page 0/page 1 occupied-slot swap için **regression doğrulaması** olarak tutulur.

### ITEM-T08 — Special Inventory type/range
Kontrollü karakter üzerinde:
- skillbook
- metin stone
- material/resource
- normal item
ile move/autogive testleri.

Doğrula:
- normal item special range'e giremez
- special item yalnız kendi range'ine gider
- special item size > 1 varsa special grid reddeder
- relog sonrası window=INVENTORY ve special cell korunur.

Ek corruption testi:
DB'de special itemı yanlış special subrange pos'a koy.
Relog sonrası restore davranışını ve manuel move ile recovery'yi izle.

### ITEM-T09 — Storage TItemPos destination allowlist
Kontrollü test clientı/Python console ile Safebox, Mall ve Guild Storage checkout için 3-arg window-aware binding kullan.

Destination matris:
- INVENTORY
- special inventory doğru/yanlış subrange
- BELT
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1
- DRAGON_SOUL_INVENTORY.

SWITCHBOT:
- geçerli weapon
- normal potion/material
- costume/non-supported type
ayrı denenir.

Kontrol:
- server reject/accept
- item window/cell
- manager registration
- DB persistence
- relog state.

Beklenen güvenli tasarım:
yalnız storage özelliğinin açıkça desteklediği destination windowları kabul edilmeli ve hedef window'un semantic validator'ı yeniden çalışmalı.

### ITEM-T10 — Additional Equipment direct checkout
`ENABLE_ADDITIONAL_EQUIPMENT_PAGE` build.

Safebox/Mall/Guild Storage checkout destination:
`ADDITIONAL_EQUIPMENT_1`.

Uygun ve uygunsuz item ile:
- slot pointer
- item window/cell
- IsEquipped
- stat bonus
- client visibility
- page unlock state
- relog persistence
kontrol edilir.

### ITEM-T11 — Special Inventory bWindow bounds
Yalnız izole development server'da test et.

Extend Inventory request ve upgrade için special-state açıkken sınır değerleri:
- valid: 0, 1, 2
- invalid: 3 ve 255

İzle:
- server log/crash
- ASan/UBSan varsa array OOB
- `bSpecialInventoryStage[3]` komşu state
- key consumption
- stage mutation
- player save/load sonucu.

Beklenti: invalid window server tarafında hiçbir state'e dokunmadan reddedilmeli.

### ITEM-T12 — Locked special slot persistence restore
Kontrollü test karakteri ve yedek DB ile:
1. special stage=0 bırak.
2. aynı special type'ın ilk 45 açık slotunun üstünde fakat static type range içinde bir persisted item position hazırla.
3. login/item load çalıştır.
4. item owner/window/cell, grid pointer, client visibility ve relog persistence kontrol et.

Beklenti: locked position restore edilmemeli; item güvenli recovery inventory path'ine alınmalı veya explicit reject/restore queue uygulanmalı.

### ITEM-T13 — Special type size > 1 dataset audit
Runtime server `item_proto` tablosu veya unpacked proto export üzerinde:
- ITEM_SKILLBOOK
- ITEM_METIN
- ITEM_MATERIAL
- ITEM_RESOURCE
- vnum 27987

için `size > 1` satırları ara.

Kodun mevcut invariant'ı: `IsEmptySpecialItemGrid(..., bSize > 1) -> false`. Dataset'te böyle item varsa special auto-placement/movement uyumsuzluğu ayrıca sınıflandırılmalı.

### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no Inventory/Item runtime test executed.
- ITEM-T02 -> BUG-ITEM-001.
- ITEM-T01 -> BUG-ITEM-002.
- ITEM-T03 -> BUG-ITEM-003.
- ITEM-T06 -> BUG-ITEM-004.
- ITEM-T09 and ITEM-T10 -> BUG-ITEM-006 from generic storage-window and Additional Equipment angles.
- ITEM-T11 -> BUG-ITEM-007.
- ITEM-T12 -> BUG-ITEM-008.
- ITEM-T05 validates OBS-ITEM-001 only; observation status is unchanged.
- ITEM-T07 validates OBS-ITEM-002 as regression/behavioral evidence only; observation status is unchanged.
- ITEM-T04 is a ground-item persistence/lifecycle regression test without a unique verified bug mapping.
- ITEM-T08 is a Special Inventory type/range regression test.
- ITEM-T13 is a runtime item_proto dataset audit dependency.
- There is no canonical BUG-ITEM-005 entry in bugs/inventory_items.md; the numbering gap is preserved and no bug is invented.
- Legacy Guild Storage and duplicate SWITCHBOT test blocks remain in this file for history, but their readiness ownership belongs to their canonical subsystem clusters.
- Primary normal-path candidate: ITEM-T01. ITEM-T02 is also reachable through the normal destroy flow but sanitizer/debug evidence is preferred for the use-after-free.
- Overall first live runtime gate remains DUNGEON-T09.
