# Project_Map

Bu repo, Metin2 projesinin salt-okuma haritalama ve teknik dokümantasyon deposudur.

## Amaç
- Project_ClientSrc, Project_ServerSRC, Project_Binary ve Project_Game repolarındaki yapıyı belgelemek
- Sistem → dosya → fonksiyon → packet → server → DB akışlarını kalıcı olarak kaydetmek
- Potansiyel bug adaylarını ve oyun içi test planlarını takip etmek
- Sohbet tekrarını azaltmak ve kaldığımız yeri güvenilir şekilde saklamak

## Kural
Bu repo haritalama/dokümantasyon içindir. Oyun kodu burada tutulmaz ve kaynak repolara yazma işlemi yapılmaz.

## Dosyalar
- 00_PROGRESS.md — güncel checkpoint ve ilerleme
- 01_REPO_MAP.md — dört ana reponun genel haritası
- 02_CLIENT_MAP.md — client/python katmanı
- 03_SERVER_MAP.md — server çekirdeği
- 04_GAME_MAP.md — game/quest/data katmanı
- 05_PACKET_MAP.md — packet ve network eşleşmeleri
- 06_DATABASE_MAP.md — DB/persistence akışları
- 07_SYSTEM_FLOWS.md — uçtan uca sistem akışları
- 08_BUG_CANDIDATES.md — potansiyel bug ve riskler
- 09_TEST_PLAN.md — oyun içi doğrulama senaryoları
- 10_UNKNOWN_AREAS.md — henüz çözülmemiş bölgeler
