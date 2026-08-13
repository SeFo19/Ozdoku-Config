# Ozdoku Network Topoloji Diyagramı

Bu diyagram `usak_topoloji.md` ve `config/` altındaki cihaz konfigürasyonlarından, ayrıca `investigation.md`'deki canlı SSH bulgularından derlenmiştir. Sorunlu bağlantılar (CASE-007) kırmızı ile işaretlenmiştir.

**Son güncelleme:** 2026-08-13 — Kabinet3'ün fiziksel bağlantısı SFP değişimi sırasında taşındı (aşağıda işaretli). Bu diyagram `investigation.md` ile birlikte güncel tutulmalıdır; investigation dosyasında yeni case/bulgu eklendikçe burası da güncellenmeli.

⚠️ **2026-08-13 fiziksel değişiklik:** Müşteri, Omurga ve Kabinet3 taraflarında SFP değiştirdi. Kabinet3 tarafı artık **port 26** (port 25 değil) kullanıyor. Omurga tarafında bağlantının **tg1/0/9**'a taşınmış olması güçlü ihtimal (trafik deseniyle destekleniyor) ama fiziksel olarak %100 teyit edilmedi. **Sorun taşındı, çözülmedi** — tg1/0/9 şu anda tg1/0/3/tg1/0/5 ile aynı kronik flap'i gösteriyor. Detay: `investigation.md` CASE-007.

---

## Genel Topoloji

```mermaid
graph TB
    FW["🔥 Firewall<br/>VLAN1 GW: 10.64.0.1<br/>VLAN2/3/4 GW'leri de burada<br/>Tüm inter-VLAN routing burada olmalı"]

    subgraph OMURGA["OMURGA Switch (Stack) — 10.64.0.6"]
        direction LR
        C1["Chassis 1<br/>🔴 tg1/0/3 ve tg1/0/5<br/>kronik flap ediyor (aktif)"]
        C2["Chassis 2<br/>✅ 3 switch de canlı doğrulandı<br/>şu anda tamamen temiz"]
    end

    FW <-->|"LACP agg2<br/>tg1/0/2 + tg2/0/2"| OMURGA

    C1 -->|"tg1/0/3<br/>🔴 SÜREKLİ FLAP<br/>(~470 döngü / 5.5 saat)"| K12["Kabinet1-2 — 10.64.0.22<br/>port 49 trunk uplink"]
    C1 -.->|"tg1/0/5 ⚪ ARTIK KULLANILMIYOR<br/>(2026-08-13'te taşındı, şimdi down)"| K3OLD["(eski) Kabinet3 port 25"]
    C1 -->|"tg1/0/9 ⚠️ muhtemel yeni bağlantı<br/>🔴 SÜREKLİ FLAP (aynı sorun!)<br/>(2026-08-13, kesin teyit edilmedi)"| K3["Kabinet3 — 10.64.0.28<br/>port 26 trunk uplink (YENİ)"]

    C2 -->|"tg2/0/10 ✅<br/>bugün 1 izole flap<br/>(switch reboot'u nedeniyle)"| K11["Kabinet1-1 — 10.64.0.24<br/>port 49 trunk uplink"]
    C2 -->|"tg2/0/8 ✅<br/>geçmişte flap oldu,<br/>~12 gündür temiz"| K2["Kabinet2 — 10.64.0.55<br/>port 25 trunk uplink"]
    C2 -->|"tg2/0/4 ✅<br/>13+ gündür neredeyse<br/>tamamen temiz"| YA["Yonetim_ALT — 10.64.0.75<br/>port 52 trunk uplink"]

    K12 --> D1["Dokuma 1-32<br/>(48 port, VLAN3)"]
    K3 --> D2["VLAN4 cihazları<br/>(24 port, çoğu shutdown)"]
    K11 --> D3["VLAN4 + VLAN2 trunk<br/>downlink'ler (48 port)"]
    K2 --> D4["VLAN4 cihazları<br/>(24 port)"]
    YA --> D5["NVR, Kamera, PC<br/>VLAN2/3/4 karışık<br/>(48 port)"]

    classDef problem fill:#ff6b6b,stroke:#a30000,color:#fff,stroke-width:2px
    classDef ok fill:#d3f9d8,stroke:#2b8a3e,color:#000
    classDef fw fill:#ffd43b,stroke:#e67700,color:#000
    classDef omurga fill:#e7f5ff,stroke:#1971c2,color:#000

    classDef unused fill:#e9ecef,stroke:#868e96,color:#666

    class C1 problem
    class C2 ok
    class FW fw
    class K3OLD unused
```

---

## VLAN Planı

| VLAN | Subnet | Ad | Not |
|---|---|---|---|
| VLAN1 | 10.64.0.0/24 | ManagementVlan | Switch yönetimi, gateway firewall (10.64.0.1) |
| VLAN2 | 10.64.2.0/24 | ClientsVlan | |
| VLAN3 | 10.64.3.0/24 | IOTVLAN | Dokuma makineleri (Kabinet1-2 üzerinden) |
| VLAN4 | 10.64.4.0/24 | KameraVlan | Kameralar, NVR |

Tüm VLAN'lar Omurga üzerinden L2 trunk ile firewall'a taşınıyor; DHCP gateway'leri firewall'da tanımlı. (CASE-002/003: bu varsayımla çelişen TTL bulgusu `investigation.md`'de ayrıca takip ediliyor.)

---

## Uplink Port Eşleme Tablosu

| Kenar Switch | IP | Uplink Portu | Omurga Portu | Chassis | Durum (2026-08-11) |
|---|---|---|---|---|---|
| Kabinet1-1 | 10.64.0.24 | 49 | tg2/0/10 | 2 | ✅ Temiz (canlı doğrulandı) — bugün ~18:43 açıklanmamış coldstart, 1 izole flap'e sebep oldu |
| Kabinet1-2 | 10.64.0.22 | 49 | tg1/0/3 | **1** | 🔴 Kronik flap — CASE-007, hâlâ aktif (2026-08-13'te tekrar kontrol edilmedi) |
| Kabinet2 | 10.64.0.55 | 25 | tg2/0/8 | 2 | ✅ Temiz (canlı doğrulandı) — geçmişte (~12 gün önce) 387 flap olayı, artık aktif değil |
| Kabinet3 | 10.64.0.28 | ~~25~~ **26 (2026-08-13'te taşındı)** | ~~tg1/0/5~~ **tg1/0/9 (kesin teyit edilmedi)** | **1** | 🔴 Sorun taşındı, çözülmedi — yeni port da kronik flap ediyor |
| Yonetim_ALT | 10.64.0.75 | 52 | tg2/0/4 | 2 | ✅ Temiz (canlı doğrulandı) — 30 Temmuz'da 1 izole flap, sonrasında temiz |
| Omurga (stack) | 10.64.0.6 | tg1/0/2 + tg2/0/2 (LACP) | Firewall | 1+2 | ✅ Normal |

---

## Sorun Örüntüsü (CASE-007 özeti)

```mermaid
graph LR
    subgraph "Chassis 1 — SORUNLU"
        direction TB
        A["tg1/0/3<br/>~470 flap / 5.5 saat<br/>(hâlâ aktif, Kabinet1-2)"]
        B["tg1/0/5<br/>ARTIK KULLANILMIYOR<br/>(taşındı, şimdi down)"]
        F["tg1/0/9<br/>YENİ — 08-13'te başladı<br/>300/300 flap (46 dk pencere)<br/>muhtemelen yeni Kabinet3 portu"]
    end
    subgraph "Chassis 2 — TEMİZ (3/3 canlı doğrulandı)"
        direction TB
        C["tg2/0/10<br/>1 izole flap (bugün, reboot)"]
        D["tg2/0/8<br/>geçmişte flap, ~12 gündür temiz"]
        E["tg2/0/4<br/>1 izole flap (30 Tem), sonra temiz"]
    end

    classDef problem fill:#ff6b6b,stroke:#a30000,color:#fff
    classDef ok fill:#d3f9d8,stroke:#2b8a3e,color:#000
    classDef unused fill:#e9ecef,stroke:#868e96,color:#666

    class A,F problem
    class B unused
    class C,D,E ok
```

**Önemli çıkarım (2026-08-13):** SFP/port değişimi sorunu çözmedi — chassis 1 içinde `tg1/0/5`'ten `tg1/0/9`'a taşıdı. Bu, kök nedenin belirli bir SFP/porta değil, **Omurga chassis 1'in kendisine** (veya değişimlerde ortak kullanılan bir fiber/patch kablosuna) işaret ettiğini güçlendiriyor.

**CASE-010 (güvenlik, ayrı konu):** Kabinet2↔Kabinet3 ve Kabinet1-1↔Yonetim_ALT ikişerli gruplar halinde birebir aynı SSH host key'i paylaşıyor — muhtemelen fabrika varsayılanı anahtar, MITM riski.

Detaylı bulgular, hipotezler ve doğrulama adımları için bkz. `investigation.md` — CASE-007, CASE-010, CASE-011.
