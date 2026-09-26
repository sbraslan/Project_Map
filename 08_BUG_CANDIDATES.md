# 08 — Bug Candidates

Bu dosyada sadece **aday** problemler tutulur. Kaynak kod + davranış testi doğrulanmadan “kesin bug” denmez.

## Kayıt formatı

### BUG-CANDIDATE-XXX
- Sistem:
- Repo / dosya:
- Belirti:
- Teknik neden:
- Etki:
- Exploit ihtimali:
- Reprodüksiyon:
- Kaynak kanıt:
- Oyun içi test:
- Durum: açık / doğrulandı / yanlış alarm / düzeltildi

## Öncelikli test alanları
- Guild Storage slot sınırları
- Invalid window_type / cell
- Eşzamanlı checkin / checkout
- Logout sırasında işlem
- Reconnect sonrası item/state tutarlılığı
- Permission bypass ihtimali
