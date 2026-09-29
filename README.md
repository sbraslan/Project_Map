# Project_Map

Bu depo Metin2 kaynak kodu için **salt-okuma statik haritalama ve bug kanıt deposudur**.

## Tek kanonik ilerleme sistemi
**İlerleme durumunu belirleyen tek dosya: `MAP_STATE.json`.**

Yeni bir sohbet veya devam turu her zaman:
1. yalnız `MAP_STATE.json` dosyasını okur;
2. `active.status == OPEN` ise yalnız o sistemin cursor'undan devam eder;
3. `CLOSED` sistemleri aynı source snapshot'ında tekrar taramaz;
4. yeni bir aday sistem açmadan önce `alias_index` ile mevcut CLOSED/OPEN node'lara eşleştirir.

`systems/`, `bugs/` ve `tests/` yalnız teknik **kanıt** dosyalarıdır. İçlerindeki eski tarihsel ifadeler ilerleme durumu belirlemez.

## Kalıcı kilit
Bir subsystem `CLOSED` olduğunda tekrar açılması yalnız iki durumda mümkündür:
- kullanıcı açıkça yeniden denetim ister;
- ilgili source repo SHA değişir ve impact check sistemi `INVALIDATED` olarak işaretler.

Bunun dışında CLOSED node tekrar okunmaz/taranmaz.

## Lookup indexleri
`index/files.json` ve `index/repositories.json` yalnız ham source envanteri/metadata için kullanılır. State'ten türetilmiş feature/symbol/packet/callgraph indeksleri tutulmaz. **Status/cursor/queue yalnız MAP_STATE.json içindedir.**

## Repo politikası
Yalnız `sbraslan/Project_Map` yazılabilir. Client/Server/Binary/Game/DumpProto kaynak repoları salt-okumadır.

## Şu anki devam noktası
Bunu README'den değil, her zaman `MAP_STATE.json -> active` alanından oku.
