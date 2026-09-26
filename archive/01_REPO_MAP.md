# 01 — Repo Map

Bu dosya dört ana Metin2 reposunun yüksek seviyeli haritasını tutar.

## Project_ClientSrc
- Client tarafı C++ / Python entegrasyonu
- UI → binding → network zincirleri
- Packet gönderim/alım noktaları

## Project_ServerSRC
- Game/server çekirdeği
- Packet handler'lar
- Character / item / guild / quest / DB çağrıları

## Project_Binary
- Client binary tarafı
- Network, UI binding, packet işleme ve engine bağlantıları

## Project_Game
- Questler, game dosyaları ve runtime data içerikleri

## Haritalama ilkesi
Her sistem mümkün olduğunda şu sırayla belgelenir:

UI / Script
→ Python binding
→ C++ client function
→ Packet
→ Server handler
→ Business logic
→ DB / persistence
→ Response / client refresh
