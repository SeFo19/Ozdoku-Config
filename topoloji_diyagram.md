# Ozdoku Network Topoloji Diyagramı

Bu diyagram `usak_topoloji.md` ve `config/` altındaki cihaz konfigürasyonlarından, ayrıca `investigation.md`'deki canlı SSH bulgularından derlenmiştir. Sorunlu bağlantılar (CASE-007) kırmızı ile işaretlenmiştir.

**Son güncelleme:** 2026-08-13 — 🚨 **Kök neden Omurga DEĞİL, SFP de DEĞİL.** Bu diyagram `investigation.md` ile birlikte güncel tutulmalıdır; investigation dosyasında yeni case/bulgu eklendikçe burası da güncellenmeli.

## 🔖 TL;DR

- Kabinet3'ün Omurga bağlantısı (aynı SFP + aynı kablo) sırayla 3 farklı Omurga portuna taşındı (tg1/0/5 → tg1/0/9 → tg2/0/12, 2 farklı chassis) — **hepsinde aynı kronik flap sorunu.** → **Omurga ekarte edildi.**
- Sonra hem Omurga hem Kabinet3 tarafında **yeni/sıfır SFP** takıldı — **sorun yine de devam etti.** → **SFP modülleri de ekarte edildi.**
- **Kalan şüpheliler:** fiber patch kablosu (hiç değişmedi) ve/veya Kabinet3'ün kendisi.
- Sıcaklık kontrol edildi, anomali yok (ama sadece anlık veri, gerçek trend yok).
- Kabinet1-2 tarafı (`tg1/0/3`) ayrı, hâlâ çözülmemiş bir sorun — bu testlerin kapsamı dışında.

Detay: `investigation.md` — Bölüm 5, CASE-007.

---

## Genel Topoloji

```mermaid
graph TB
    FW["🔥 Firewall<br/>VLAN1 GW: 10.64.0.1<br/>VLAN2/3/4 GW'leri de burada<br/>Tüm inter-VLAN routing burada olmalı"]

    subgraph OMURGA["OMURGA Switch (Stack) — 10.64.0.6 — ✅ artık şüpheli değil"]
        direction LR
        C1["Chassis 1<br/>🔴 tg1/0/3 (Kabinet1-2) kronik flap<br/>⚪ tg1/0/5, tg1/0/9 artık boş"]
        C2["Chassis 2<br/>✅ Kabinet1-1/2/YA temiz<br/>🔴 tg2/0/12 de flap ediyor (SFP'den bağımsız)"]
    end

    FW <-->|"LACP agg2<br/>tg1/0/2 + tg2/0/2"| OMURGA

    C1 -->|"tg1/0/3<br/>🔴 SÜREKLİ FLAP<br/>(~470 döngü / 5.5 saat)"| K12["Kabinet1-2 — 10.64.0.22<br/>port 49 trunk uplink"]
    C1 -.->|"tg1/0/5, tg1/0/9 ⚪ ARTIK BOŞ<br/>(sırayla denendi, ikisi de flap etti)"| K3OLD["(eski) denemeler"]
    C2 -->|"tg2/0/12 🔴 SÜREKLİ FLAP<br/>(aynı SFP+kablo, chassis 2'de de!)<br/>(2026-08-13, doğrulandı)"| K3["Kabinet3 — 10.64.0.28<br/>port 26 trunk uplink"]

    C2 -->|"tg2/0/10 ✅<br/>bugün 1 izole flap<br/>(switch reboot'u nedeniyle)"| K11["Kabinet1-1 — 10.64.0.24<br/>port 49 trunk uplink"]
    C2 -->|"tg2/0/8 ✅<br/>geçmişte flap oldu,<br/>~12 gündür temiz"| K2["Kabinet2 — 10.64.0.55<br/>port 25 trunk uplink"]
    C2 -->|"tg2/0/4 ✅<br/>13+ gündür neredeyse<br/>tamamen temiz"| YA["Yonetim_ALT — 10.64.0.75<br/>port 52 trunk uplink"]

    K12 --> D1["Dokuma 1-32<br/>(48 port, VLAN3)"]
    K3 --> D2["VLAN4 cihazları<br/>(24 port, çoğu shutdown)<br/>⚠️ ŞÜPHELİ: fiber kablosu/Kabinet3 tarafı<br/>(SFP ekarte edildi)"]
    K11 --> D3["VLAN4 + VLAN2 trunk<br/>downlink'ler (48 port)"]
    K2 --> D4["VLAN4 cihazları<br/>(24 port)"]
    YA --> D5["NVR, Kamera, PC<br/>VLAN2/3/4 karışık<br/>(48 port)"]

    classDef problem fill:#ff6b6b,stroke:#a30000,color:#fff,stroke-width:2px
    classDef ok fill:#d3f9d8,stroke:#2b8a3e,color:#000
    classDef fw fill:#ffd43b,stroke:#e67700,color:#000
    classDef omurga fill:#e7f5ff,stroke:#1971c2,color:#000
    classDef unused fill:#e9ecef,stroke:#868e96,color:#666
    classDef suspect fill:#ffd8a8,stroke:#e8590c,color:#000,stroke-width:2px

    class C1,C2 omurga
    class FW fw
    class K3OLD unused
    class K3 suspect
```
Not: Chassis kutuları artık nötr (mavi) — Omurga'nın kendisi şüpheli değil. `tg1/0/3` (Kabinet1-2) hâlâ ayrı, çözülmemiş bir sorun (bu SFP taşıma testinin konusu değil).

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
| Kabinet3 | 10.64.0.28 | ~~25~~ **26 (2026-08-13'te taşındı)** | ~~tg1/0/5~~ ~~tg1/0/9~~ **tg2/0/12 (chassis 2!)** | ~~1~~ **2** | 🔴 Sorun 3 farklı Omurga portunda/2 chassis'te, hatta yeni SFP'lerle bile devam etti — **Omurga ve SFP ekarte edildi**, fiber kablosu/Kabinet3 tarafı şüpheli |
| Yonetim_ALT | 10.64.0.75 | 52 | tg2/0/4 | 2 | ✅ Temiz (canlı doğrulandı) — 30 Temmuz'da 1 izole flap, sonrasında temiz |
| Omurga (stack) | 10.64.0.6 | tg1/0/2 + tg2/0/2 (LACP) | Firewall | 1+2 | ✅ Normal |

---

## Sorun Örüntüsü (CASE-007 özeti)

```mermaid
graph LR
    subgraph "Chassis 1"
        direction TB
        A["tg1/0/3<br/>~470 flap / 5.5 saat<br/>(hâlâ aktif, Kabinet1-2 — AYRI sorun)"]
        B["tg1/0/5, tg1/0/9<br/>ARTIK BOŞ<br/>(Kabinet3 testinde sırayla<br/>kullanıldı, ikisi de flap etti)"]
    end
    subgraph "Chassis 2"
        direction TB
        C["tg2/0/10<br/>1 izole flap (bugün, reboot)"]
        D["tg2/0/8<br/>geçmişte flap, ~12 gündür temiz"]
        E["tg2/0/4<br/>1 izole flap (30 Tem), sonra temiz"]
        G["tg2/0/12 🚨<br/>Kabinet3 testi burada<br/>(eski SFP: 62 flap/9.5dk)<br/>(yeni SFP: sorun devam etti)"]
    end
    subgraph "Elenen / Kalan Şüpheliler"
        direction TB
        H["✅ Omurga (chassis 1, chassis 2,<br/>3 farklı port) — EKARTE"]
        I["✅ SFP modülleri<br/>(eski + yeni, iki uçta) — EKARTE"]
        J["❓ Fiber patch kablosu<br/>— HİÇ DEĞİŞMEDİ, ŞÜPHELİ"]
        K["❓ Kabinet3 tarafı<br/>(switch/port) — İZOLE EDİLMEDİ"]
    end

    classDef problem fill:#ff6b6b,stroke:#a30000,color:#fff
    classDef ok fill:#d3f9d8,stroke:#2b8a3e,color:#000
    classDef unused fill:#e9ecef,stroke:#868e96,color:#666
    classDef cleared fill:#d3f9d8,stroke:#2b8a3e,color:#000
    classDef suspect fill:#ffd8a8,stroke:#e8590c,color:#000,stroke-width:2px

    class A,G problem
    class B unused
    class C,D,E ok
    class H,I cleared
    class J,K suspect
```

**🚨 Kesin sonuç (2026-08-13):** Kabinet3 bağlantısı (aynı SFP + aynı kablo) sırayla `tg1/0/5` → `tg1/0/9` (chassis 1) → `tg2/0/12` (**chassis 2**) taşındı, **üçünde de aynı flap sorunu devam etti** → Omurga ekarte edildi. Sonra hem Omurga hem Kabinet3 tarafında **yeni/sıfır SFP takıldı, sorun yine devam etti** → SFP modülleri de ekarte edildi. **Kalan şüpheliler: fiber patch kablosu (hiç değişmedi) ve Kabinet3 tarafı (hiç izole edilmedi).** (Not: `tg1/0/3`/Kabinet1-2 sorunu bu testten bağımsız, hâlâ çözülmemiş, ayrı bir konu.)

**CASE-010 (güvenlik, ayrı konu):** Kabinet2↔Kabinet3 ve Kabinet1-1↔Yonetim_ALT ikişerli gruplar halinde birebir aynı SSH host key'i paylaşıyor — muhtemelen fabrika varsayılanı anahtar, MITM riski.

Detaylı bulgular, hipotezler ve doğrulama adımları için bkz. `investigation.md` — CASE-007, CASE-010, CASE-011.
