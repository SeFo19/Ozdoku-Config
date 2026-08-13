# Ozdoku Network Sorunu — Investigation Dosyası

**Durum:** Açık / Devam ediyor
**Oluşturulma:** 2026-08-11
**Son güncelleme:** 2026-08-13
**Kural:** Cihazlarda hiçbir konfigürasyon değişikliği yapılmadı/yapılmayacak. Bu doküman sadece gözlem, bulgu, hipotez ve soru takibi için kullanılıyor. Emin olunmayan hiçbir konu "tespit" olarak yazılmıyor, "hipotez" veya "doğrulanması gereken" olarak işaretleniyor.

---

## 🔖 TL;DR (2026-08-13 itibarıyla)

- **Sorun:** Kabinet1-2 ve Kabinet3'ün Omurga'ya olan uplink bağlantıları kronik, tekrarlayan şekilde flap ediyor (saniyeler içinde down/up). Kabinet1-2 tarafı hâlâ çözülmedi, hiç izole edilmedi. Kabinet3 tarafında yoğun izolasyon testleri yapıldı (aşağıda).
- **Ekarte edildi:** Omurga switch donanımı (chassis 1, chassis 2, hiçbir port) — bağlantı 3 farklı Omurga portuna/2 farklı chassis'e taşındı, sorun her seferinde takip etti. **SFP modülleri de ekarte edildi** — hem Omurga hem Kabinet3 tarafında sıfır/yeni SFP takıldı, sorun devam etti. Güç/fan/sıcaklık normal (anomali yok, ama sıcaklık için sadece anlık veri var, gerçek 24 saatlik trend yok).
- **Kalan en güçlü şüpheliler:** (1) Fiber patch kablosu — hiç değiştirilmedi, testler boyunca sabit kaldı; (2) Kabinet3 tarafının kendisi (switch/port).
- **Önerilen sonraki adım:** Fiber patch kablosunun değiştirilmesi/test edilmesi; mümkünse farklı bir fiber rotası ile test.
- **Yan bulgular:** SSH host key paylaşımı (güvenlik, CASE-010), bazı switch'lerin NTP olmadan doğru saat göstermesi (CASE-011), TTL/routing tutarsızlığı (CASE-002/003, hâlâ doğrulanmadı).
- Detaylar için: Bölüm 5 (Case kayıtları), Bölüm 6 (açık sorular), `topoloji_diyagram.md` (görsel özet).

---

## 1. Özet

Müşteri Kabinet1-2 (10.64.0.22) switch'ine erişimi (SSH/ping) kaybettiklerini bildirdi. Kesinti anında ping atılıyordu ve çıktı ekran görüntüsü (`image.png`) paylaşıldı. Switch web konsolu loglarında aynı zaman aralığında `vlan1` arayüzünün çok hızlı şekilde down/up yaptığı görüldü. Kullanıcı benzer sorunun Kabinet3 (10.64.0.28) kenar switch'inde de yaşandığını bildirdi; bu switch'e canlı SSH erişimiyle teşhis yapıldı (bkz. Bölüm 5, CASE-007) ve **çok daha ciddi, kronik bir port flapping** tespit edildi.

Elimizdeki veriler: topoloji dokümanı (`usak_topoloji.md`), 6 switch config dökümü (`config/`), bir log parçası (`prompt.md`) ve bir ping ekran görüntüsü (`image.png`).

**Ön değerlendirme:** Ping çıktısındaki TTL değerleri ve kesinti anında görülen "TTL expired in transit" mesajları, müşterinin tarif ettiği "ağ tamamen L2 çalışıyor" mimarisiyle çelişen bir bulgu ortaya koyuyor (bkz. CASE-002, CASE-003). Bu, henüz **doğrulanmamış bir hipotez** — teyit için ek bilgi gerekiyor (bkz. Bölüm 6).

**🚨 GÜNCEL EN ÖNEMLİ BULGU (2026-08-11, CASE-007):** Omurga'ya canlı bağlanılarak yapılan inceleme, **kenar switch'lerin kendisinde değil, Omurga'nın chassis 1'inde** ortak bir sorun olduğunu kesin olarak doğruladı: Kabinet1-2 (`tg1/0/3`) ve Kabinet3'ün (`tg1/0/5`) Omurga tarafındaki portları, sabah bildirilen olaydan beri **hâlâ aktif olarak, sürekli flap ediyor** (~5.5 saatte ~500'er kez); chassis 2'ye bağlı bir port ise neredeyse hiç etkilenmedi. Güç/fan/sıcaklık normal — kök neden muhtemelen SFP/optik veya donanımsal (line card/ASIC) düzeyde, henüz kesinleşmedi. Detay: Bölüm 5, CASE-007.

**🚨🚨🚨 2026-08-13 EN GÜNCEL SONUÇ — KÖK NEDEN OMURGA DEĞİL:** Müşteri, sorunu izole etmek için bağlantıyı sırayla taşıdı: `tg1/0/5` (chassis 1) → `tg1/0/9` (chassis 1, hâlâ flap) → **`tg2/0/12` (chassis 2, hâlâ flap!)**. Aynı SFP modülü (SN: ANW22071402084) ve aynı, değişmeyen fiber patch kablosuyla üç farklı Omurga portunda, iki farklı chassis'te aynı kronik flap sorunu görüldü. **Bu, Omurga'nın (herhangi bir chassis/port) kök neden olma ihtimalini çok zayıflatıyor.** Kalan şüpheliler: taşınan SFP modülünün kendisi, değişmeyen fiber kablosu, veya Kabinet3 tarafı (port 26 SFP'si/kendisi) — henüz test edilmedi. Detay: Bölüm 5, CASE-007.

**✅ 2026-08-13 ~13:53 Güncelleme:** Kullanıcı `tg2/0/12`'ye doğru trunk config'i ekledi ve fiziksel bağlantı da düzelmiş görünüyor — LOS yok, RX gücü normal, ~9 dakikadır flap yok. **Henüz kalıcı olduğu teyit edilmedi** (kısa pencere), izleme devam ediyor. ⚠️ **Güncelleme (14:09):** Bu iyileşme kalıcı değilmiş — `tg2/0/12` tekrar down/LOS'a döndü.

**🚨🚨🚨🚨 2026-08-13 ~14:09 EN KRİTİK SONUÇ — SFP DE ELENDİ:** Kullanıcı hem Omurga hem Kabinet3 tarafında **sıfır/yeni SFP modülleri** taktı — **sorun yine de devam etti.** SFP modülleri artık kesin olarak elendi. (Not: canlı kontrolümde `tg2/0/12`'de hâlâ eski SFP seri numarası — ANW22071402084 — görünüyordu; kullanıcı yeni SFP taktığını teyit etti, bu tutarsızlık çözülmedi, muhtemelen zamanlama/önbellek farkı — bkz. CASE-007 notu.) **Artık en güçlü şüpheli: fiber patch kablosu (hiç değişmedi) veya Kabinet3 tarafının kendisi.** Sıcaklık kontrolü de yapıldı: Omurga chassis1/2 (46-47°C) ve Kabinet3 (61°C) hepsi normal aralıkta (alarm eşiklerinin çok altında), 24 saatlik log taramasında hiçbir sıcaklık/alarm kaydı yok — ama platform gerçek anlamda geçmişe dönük sıcaklık trendi tutmuyor, sadece anlık okuma sunuyor, bu yüzden "anomali yok" tespiti sınırlı veriye dayanıyor.

---

## 2. Toplanan Kanıtlar

| Kaynak | İçerik | Not |
|---|---|---|
| `prompt.md` | Kabinet1-2 web konsolu log kaydı (kesinti günü) | Sadece "info" seviyeli NETD/HAL logları, muhtemelen filtrelenmiş/kısmi |
| `usak_topoloji.md` | Topoloji, VLAN planı, switch/port eşlemeleri | Müşteri beyanı — config ile çapraz kontrol edildi |
| `image.png` | Kesinti anında `ping 10.64.0.22` çıktısı | Kaynak host, OS, VLAN bilgisi belirtilmemiş |
| `config/Kabinet1-1.txt` | Running-config | Okundu |
| `config/Kabinet1-2.txt` | Running-config | Okundu |
| `config/Kabinet2.txt` | Running-config | Güncellendi (2026-08-11) — okundu, bkz. CASE-001 (kapatıldı) |
| `config/Kabinet3.txt` | Running-config | Okundu |
| `config/Yonetim_ALT.txt` | Running-config (hostname: `Yonetim_Alt`) | Okundu |
| `config/Omurga config.txt` | Running-config (2 chassis, stack) | Okundu |
| `logs/2026-08-11/Kabinet3_195500.md` | Kabinet3'e canlı SSH oturumu (salt-okuma) | `usak_topoloji.md`'de paylaşılan erişim kurallarına göre loglandı |
| `logs/2026-08-11/Kabinet3_200500.md` + `logs/changes/2026-08-11_Kabinet3_ntp-saat-senkronizasyonu.md` | Kabinet3 NTP config değişikliği | Kullanıcı onayıyla, CASE-008 |
| `logs/2026-08-11/Omurga_203400.md` | Omurga'ya canlı SSH oturumu (salt-okuma) | CASE-007'nin ana kanıtı — chassis 1 flap analizi |

---

## 3. Olay Zaman Çizelgesi (Kabinet1-2 log kaydından, 2026-08-11)

Ham log ters kronolojik (en yeni üstte) paylaşılmıştı; aşağıda kronolojik sıraya (eskiden yeniye) çevrilmiş hali var — **bu benim düzenlemem, yorum değil, sadece sıralama.**

| Saat | Olay |
|---|---|
| 16:39:23 | `vlan1` → down |
| 16:39:24 | `vlan1` → up |
| 16:39:25 | `vlan1` → down |
| 16:39:27 | `vlan1` → up |
| 16:39:28 | `vlan1` → down |
| 16:39:29 | `vlan1` → up |
| 16:39:30 | `vlan1` → down |
| 16:39:31 | `vlan1` → up |
| 16:39:32 | `vlan1` → down |
| **16:39:32 – 16:44:33** | **Log kaydı yok (~5 dakikalık boşluk)** |
| 16:44:33 | `tengigabitEthernet0/49` (omurga uplink portu) → up |
| 16:44:34 | `vlan1` → up |
| 16:44:44 | `vlan1`, DHCP ile router (gateway) IP'sini öğreniyor: 10.64.0.1 |
| 16:44:44 | `vlan1`, DHCP ile IP alıyor: 10.64.0.22/24 |

**Dikkat çeken nokta:** 16:39:32'de vlan1 down olduktan sonra, tg0/49 uplink portunun ne zaman down olduğuna dair **hiçbir log kaydı yok** — sadece 16:44:33'te tekrar "up" olduğu görünüyor. Bu ya (a) paylaşılan logun eksik/filtreli olduğunu, ya da (b) fiziksel port hiç down olmadan farklı bir mekanizmanın (örn. STP, kontrol düzlemi) vlan1 SVI'sini etkilediğini gösterebilir. **Doğrulanmadı — CASE-005.**

---

## 4. Topoloji ↔ Config Karşılaştırması

### 4.1 Doğrulanan / Eşleşen Bilgiler

- Kabinet1-2 `tg0/49` trunk ↔ Omurga `tg1/0/3` trunk (vlan-allowed 1-4) — **eşleşiyor**
- Kabinet1-1 `tg0/49` trunk ↔ Omurga `tg2/0/10` trunk (vlan-allowed 1-4) — **eşleşiyor**
- Kabinet3 `tg0/25` trunk ↔ Omurga `tg1/0/5` trunk (vlan-allowed 1-4) — **eşleşiyor (orijinal config)** — ⚠️ **2026-08-13'te fiziksel olarak değişti:** müşteri SFP değişimi sırasında Kabinet3 tarafını `tg0/26`'ya taşıdı, Omurga tarafı da muhtemelen `tg1/0/9`'a taşındı (kesin teyit edilmedi). Bkz. CASE-007, 2026-08-13 güncellemesi. **Bu artık güncel/canlı topolojiyi yansıtmıyor, sadece orijinal/başlangıç durumunu gösteriyor.**
- Yonetim_ALT `tg0/52` trunk ↔ Omurga `tg2/0/4` trunk (vlan-allowed 1-4) — **eşleşiyor**
- Kabinet2 `tg0/25` trunk ↔ Omurga `tg2/0/8` trunk (vlan-allowed 1-4) — **eşleşiyor**
- Omurga `tg1/0/2` + `tg2/0/2` → LACP aggregator-group 2 → firewall bağlantısı — topoloji notuyla **eşleşiyor**
- Tüm switch'lerde `spanning-tree mode rstp` aktif; Omurga'da `spanning-tree rstp priority 8192` ile root bridge önceliği düşürülmüş (Omurga'nın root olması hedeflenmiş gibi görünüyor) — tutarlı
- **Tüm 5 kenar switch'te (Kabinet1-1, Kabinet1-2, Kabinet2, Kabinet3, Yonetim_ALT) ortak desen:** trunk uplink portlarında (`tg0/49-52` veya `tg0/25-28`) `loop-detect enable` YOK, ama access portlarının tamamında var. Kabinet2 config'i eklenince bu desenin tüm switch'lerde tutarlı olduğu doğrulandı — muhtemelen kasıtlı bir tasarım (uplink portunda loop-detect beklenmez), Kabinet1-2'ye özgü bir eksiklik değil.

### 4.2 Küçük / İkincil Gözlemler (case açılmadı, sadece not)

- Omurga config'inde `tg1/0/4, tg1/0/6, tg1/0/8, tg2/0/1, tg2/0/3, tg2/0/5, tg2/0/7` portları tekli (eşleşmeyen partner'sız) LACP aggregator-group üyesi olarak tanımlı. İşlevsel bir sorun yaratmayabilir ama kullanılmıyorsa temizlik konusu olabilir. Sorunla ilgisi şu an bilinmiyor.
- Omurga `tg1/0/10` portu trunk olarak yapılandırılmamış (sadece fiber-auto/ddm var). Topolojide bu port için bir kullanım belirtilmemiş, muhtemelen ilgisiz.
- Yonetim_ALT config'inde hostname `Yonetim_Alt`, topoloji dokümanında `Yonetim_ALT` — sadece yazım farkı, önemsiz.

---

## 5. Case Kayıtları (Tutarsızlıklar / Açık Bulgular)

### CASE-001 — Kabinet2 config dosyası boş
**Durum:** Kapatıldı (2026-08-11)
`config/Kabinet2.txt` ilk paylaşıldığında içeriği boştu. Kullanıcı config'i güncelledi. Karşılaştırma yapıldı: Kabinet2 `tg0/25` trunk ↔ Omurga `tg2/0/8` trunk eşleşiyor, diğer switch'lerle tutarlı bir yapı (bkz. Bölüm 4.1). Sorunla ilişkili yeni bir anomali bulunmadı.

### CASE-002 — Ping TTL değerleri, saf L2 mimari beklentisiyle çelişiyor
**Durum:** Açık — Hipotez, doğrulama bekliyor
`image.png`'deki başarılı ping cevaplarında `TTL=62` görünüyor. Topoloji dokümanına göre switch yönetimi tamamen VLAN1 üzerinden, düz L2 (aynı broadcast domain) olarak yapılıyor — yani kaynak host ile 10.64.0.22 arasında **routing (L3 hop) olmaması beklenir**, TTL kaynaktaki orijinal değerle (tipik 64) aynı gelmelidir.

TTL=62 (varsayılan TTL=64 kabul edilirse) paketin **2 adet L3 hop'tan** geçtiğini gösteriyor. Bu, ya (a) ping kaynağının VLAN1 dışında bir subnette olduğunu ve trafiğin firewall + başka bir L3 cihazdan geçtiğini, ya da (b) beklenmeyen bir routing davranışı olduğunu düşündürüyor.

**Doğrulanması gereken:**
- Ping'in atıldığı makinenin IP'si/VLAN'ı ve işletim sistemi (varsayılan TTL OS'e göre değişir: Windows genelde 128, Linux/network cihazları genelde 64)
- Aynı hedefe `tracert`/`traceroute` çıktısı

### CASE-003 — Omurga switch'te tüm VLAN'larda SVI + default route var; L3 routing aktif mi belirsiz
**Durum:** Açık — Hipotez, doğrulama bekliyor
Omurga config'inde VLAN1, 2, 3, 4'ün hepsi için IP adresi tanımlı SVI var (10.64.0.6, 10.64.2.6, 10.64.3.6, 10.64.4.6) ve `ip route default 10.64.0.1` tanımlı. Topoloji dokümanı şunu belirtiyor: *"VLANların tamamı layer 2 olarak firewall'a geliyor"* ve *"Her trafik routing firewall'a yönlendiriliyor olmalıdır"* — yani Omurga'nın sadece L2 switching yapması, VLAN'lar arası routing yapmaması bekleniyor.

Elimizdeki config dökümünde `ip routing` komutunun global olarak açık olup olmadığı net görünmüyor (config bazı yerlerde `!` ile boş satırlar içeriyor, muhtemelen dökümün tamamı değil). Eğer IP routing omurga üzerinde aktifse, aynı subnetler için **iki farklı L3 cihaz (Omurga + Firewall)** routing yapıyor olabilir — bu asimetrik routing veya routing loop'a yol açabilir. Bu, **CASE-002'deki "TTL expired in transit" bulgusuyla örtüşüyor** (bkz. CASE-004).

**Doğrulanması gereken:**
- Omurga üzerinde `show ip routing` / config'de `ip routing` satırının varlığı
- Omurga `show ip route` çıktısı
- Firewall tarafındaki VLAN1-4 arayüz/route tanımları (henüz paylaşılmadı)

### CASE-004 — Kesinti anında "TTL expired in transit" mesajları (routing loop belirtisi)
**Durum:** Açık — Hipotez, doğrulama bekliyor
`image.png`'de kesinti anında şu satırlar görülüyor:
```
Reply from 10.64.0.1: TTL expired in transit.
Reply from 10.64.0.1: TTL expired in transit.
Reply from 10.64.0.1: TTL expired in transit.
Request timed out.
```
10.64.0.1, switch loglarında VLAN1'in DHCP ile öğrendiği gateway adresi olarak geçiyor (muhtemelen firewall'un VLAN1 arayüzü). "TTL expired in transit" mesajının 10.64.0.1'den gelmesi, paketin firewall'a TTL=1 ile ulaştığını, yani daha önce bir yerlerde (muhtemelen bir routing loop içinde) TTL bütçesinin tükenmiş olduğunu gösterir. Bu klasik bir **routing loop belirtisidir** ve CASE-003 ile birlikte değerlendirilmeli.

**Not:** Bu sadece paylaşılan ekran görüntüsündeki sınırlı veriye dayanıyor, kesin değil.

**Doğrulanması gereken:**
- Kesinti anında firewall loglarında ilgili trafiğe dair kayıt var mı
- Omurga `show mac address-table` (VLAN1) — MAC adresi iki port arasında flap ediyor mu (loop belirtisi)
- Omurga `show ip route` ve firewall route tablosu karşılaştırması

### CASE-005 — Log kaydında ~5 dakikalık boşluk, uplink portunun ne zaman düştüğü belirsiz
**Durum:** Açık
Bölüm 3'te detaylandırıldığı gibi, 16:39:32'de vlan1 down olduktan sonra 16:44:33'e kadar hiçbir log kaydı paylaşılmadı; bu sürede fiziksel uplink portunun (tg0/49) durumu bilinmiyor.
**Gereken:** O gün için kesintisiz, tüm severity seviyelerini içeren tam log kaydı (Kabinet1-2 ve mümkünse Omurga tg1/0/3 tarafı).

### CASE-007 — Omurga chassis 1'e bağlı iki uplink portu (tg1/0/3, tg1/0/5) şu anda, gerçek zamanlı olarak sürekli flap ediyor
**Durum:** Açık — 🚨 **DOĞRULANDI, EN KRİTİK BULGU — kenar switch'lerden değil Omurga chassis 1'den kaynaklanıyor**

**2026-08-11 ~20:34-20:46 arası Omurga'ya (10.64.0.6) canlı SSH ile bağlanılıp incelendi** (oturum kaydı: `logs/2026-08-11/Omurga_203400.md`). Sonuç kesin ve çok net:

- Omurga'nın kendi logları (`show logging 1 300`), **`tg1/0/3` (Kabinet1-2'nin bağlı olduğu port) portunun 18:31'den sorgu anına (20:39) kadar kesintisiz, saniyeler içinde tekrarlayan şekilde flap ettiğini** gösterdi — yani sabah bildirilen olay (16:39-16:44) tek seferlik değilmiş, sorun **hâlâ aktif ve devam ediyor**.
- Son 2000 log satırı (2026-08-11 15:07–20:42 aralığı, ~5.5 saat) yerel olarak analiz edildi, interface bazlı flap sayısı:

  | Port | Bağlı olduğu switch | Chassis | Flap olay sayısı (~5.5 saatte) |
  |---|---|---|---|
  | **tg1/0/5** | Kabinet3 | **1** | **1056** (~528 down/up döngüsü) |
  | **tg1/0/3** | Kabinet1-2 | **1** | **940** (~470 döngü) |
  | tg2/0/10 | Kabinet1-1 | 2 | 4 (~2 döngü, tek seferlik) |

- **Sonuç: Omurga chassis 1'e bağlı HER İKİ port (tg1/0/3 ve tg1/0/5) da eşit yoğunlukta ve sürekli flap ediyor; chassis 2'ye bağlı port (tg2/0/10) neredeyse hiç etkilenmiyor.** Bu, sorunun Kabinet1-2 veya Kabinet3'ün kendisinden değil, **Omurga'nın chassis 1 tarafında ortak bir sebepten** kaynaklandığını çok güçlü şekilde doğruluyor (CASE-007'nin ilk hipotezi doğrulandı).

- **Temel donanım sağlığı kontrol edildi, sorunu açıklamıyor:** `show power` (4 PSU, ikisi de chassis, hepsi OK), `show fan` (8 fan, hepsi OK), `show temperature` (chassis1: 48°C, chassis2: 45°C, ikisi de normal eşik aralığında). Yani genel güç/soğutma arızası değil.

- **Anlık interface durumu (sorgu anında "up" yakalandı) normal görünüyor:** `show interface tg1/0/3` ve `tg1/0/5` DDM/optik değerleri (TX/RX power, sıcaklık, voltaj, bias akımı) normal eşik aralığında, hata sayaçları (FCS) toplam trafiğe oranla ihmal edilebilir düzeyde (tg1/0/3: 6/36.4M, tg1/0/5: 693/1.09 milyar). **Bu, anlık/kümülatif sayaçların flap olaylarını yakalamadığını gösteriyor** — sorun kısa süreli link-down anlarında, sürekli hata üretmeden gerçekleşiyor.

**Destekleyici kanıt (Kabinet3 tarafı, daha önce toplandı):** 2026-08-11'de Kabinet3'e bağlanıp `show logging` çalıştırıldığında (`logs/2026-08-11/Kabinet3_195500.md`), 1024 satırlık log buffer'ının tamamının `tengigabitEthernet0/25` (Kabinet3'ün bu uplink'i) için tekrarlayan up/down kaydıyla dolu olduğu, hatta iki kez açık `%HAL-3: ... link-up-down error` uyarısı görüldüğü tespit edilmişti — bu, Omurga tarafındaki `tg1/0/5` bulgusuyla tam örtüşüyor.

**Kalan hipotez alanı (henüz teyit edilmedi):**
- Chassis 1'deki bu iki port için ortak bir SFP/optik modül partisi sorunu (üretim hatası, kirlenme, marjinal sinyal)
- Chassis 1'in bu portları besleyen line card/port-grubu/ASIC'inde donanımsal bir arıza
- Chassis 1 ile chassis 2 arasındaki stack senkronizasyonunda chassis 1'e özgü bir sorun
- (Daha az olası ama dışlanmadı) Kabinet1-2 ve Kabinet3 tarafındaki fiber/SFP'lerde bağımsız ama eşzamanlı bir sorun — iki farklı switch'te aynı anda, aynı yoğunlukta, aynı desende olması bu ihtimali zayıflatıyor

**Chassis 1 / chassis 2 versiyon & senkronizasyon kontrolü (2026-08-11 ~21:05-21:09):** Kullanıcı sorusu üzerine kontrol edildi. Bu platformda (Angora Networks ANW-5733-24X2Q) `show switch`, `show stack`, `show module`, `show inventory`, `show unit`, `show slot` komutlarının **hiçbiri yok** ("Unknown command") — chassis'ler arası versiyon karşılaştırması yapacak dedike bir komut bulunamadı. `show version` ve `show kernel-info` tek, birleşik bir sistem versiyonu veriyor (513A Build 152847), chassis ayrımı yapmıyor. `show oir-information` her iki chassis'in de aynı donanım modeli olduğunu doğruluyor (`Slot 1/0` ve `Slot 2/0`, ikisi de `ANW-5733-24X2Q`, `present`). Daha önce çekilen ~5.5 saatlik log (15:07-20:42) yerelde `mismatch/sync/master/standby/incompatib/stack` anahtar kelimeleriyle tarandı — **hiçbir eşleşme yok**. **Sonuç: versiyon uyuşmazlığı/senkronizasyon sorununa dair kanıt bulunamadı, ama CLI'ın bunu kesin doğrulayacak bir aracı olmadığı için %100 dışlanamıyor.** Flapping'in chassis 1'in tamamını değil, spesifik olarak sadece iki portu (tg1/0/3, tg1/0/5) etkilemesi, sorunun chassis-geneli bir stack/sync problemi olmaktan çok o iki porta özgü (SFP/optik/line-card) bir donanım sorunu olma ihtimalini güçlendiriyor — ama kesinleşmedi.

**Chassis 2 tarafı doğrulandı (2026-08-11, Kabinet1-1/Kabinet2/Yonetim_ALT canlı kontrolü):**

| Switch | Uplink portu | Toplam flap (log geçmişinde) | Güncel/aktif flap var mı? |
|---|---|---|---|
| Kabinet1-1 | tg0/49 | 0 (sadece coldstart anındaki normal açılış geçişleri) | ❌ Hayır — coldstart'tan (tahminen bugün ~18:43) beri tamamen stabil |
| Kabinet2 | tg0/25 | 387 (**hepsi geçmiş/tarihsel**, son flap tahminen ~12 gün önce) | ❌ Hayır — en az 12 gündür (bugün dahil) hiç flap yok |
| Yonetim_ALT | tg0/52 | 2 (1 flap döngüsü, 2026-07-30 tarihli, ~5 dakikalık izole kesinti) | ❌ Hayır — o tek olay dışında 13+ günlük log boyunca temiz |

**Sonuç: chassis 2 tarafındaki 3 switch de ŞU ANDA temiz/stabil — CASE-007'nin ana hipotezini (chassis 1'e özgü, güncel/aktif sorun) güçlendiriyor.** Not: Kabinet2'de geçmişte (haftalar önce) benzer yoğun flapping görülmüş olması, bu tür bir sorunun geçmişte chassis 2'de de yaşanabildiğini gösteriyor — yani "chassis 1 kalıcı/fiziksel olarak bozuk" hipotezi yanında "belirli SFP'ler veya bağlantılar zaman zaman, dönüşümlü olarak sorun çıkarıyor olabilir" hipotezi de tamamen dışlanamıyor. Ama **şu anda aktif olan** sorun kesinlikle sadece chassis 1'de (tg1/0/3, tg1/0/5).

Ayrıca üç switch'in de config'i (`show running-config`) repo'daki kopyalarla `diff` ile karşılaştırıldı — **hiçbirinde drift/fark bulunamadı** (Kabinet1-1'e kullanıcının eklediği NTP satırı hariç, bilinen bir değişiklik).

**Kalan doğrulama adımları:**
- Kabinet1-2'nin kendi tarafından (henüz canlı bağlanılmadı — sadece web konsol log ekran görüntüsü var) `show logging`/`show interface` ile teyit
- ~~Fiziksel SFP/kablo değişimi~~ — **2026-08-13'te müşteri tarafından yapıldı, bkz. aşağıdaki alt bölüm**
- Angora Networks (üretici) ile chassis 1 line card / port grubu için destek talebi açılması düşünülebilir — yazılım/uzaktan teşhisle ulaşılabilecek noktaya gelindi

#### 2026-08-13 Güncellemesi — Fiziksel SFP değişimi ve port taşıma (sorun ÇÖZÜLMEDİ, TAŞINDI)

**Değişiklik kaydı:** `logs/changes/2026-08-13_Kabinet3-Omurga_sfp-degisimi-ve-port-tasima.md` (fiziksel müdahale — müşteri tarafından yapıldı, AI ajanı sadece salt-okuma doğrulama yaptı)

Müşteri iki adım uyguladı:
1. **Omurga tarafında**, Kabinet3'ün bağlı olduğu SFP değiştirildi (port aynı kaldı) → **sorun düzelmedi, tekrar nüksetti.**
2. **Kabinet3 tarafında**, port 25'teki SFP değiştirildi ve bağlantı **port 26'ya taşındı** → ~6 dakikalık yoğun geçiş/kesinti sonrası (kullanıcının tarif ettiği "1-2 dakika" ile aynı olay, süre tam örtüşmüyor) **switch erişimi ve ağ geri geldi.**

**Canlı doğrulama sonucu (2026-08-13, ~09:52-12:56 TRT):**
- ✅ Kabinet3 tarafı: port 25 down/boşta, **port 26 aktif ve 23+ dakikadır tamamen temiz.**
- ⚠️ Omurga tarafı: `tg1/0/5` (Kabinet3'ün eski bağlantı noktası) **tamamen down**, trafik yok — bağlantı başka bir porta taşınmış.
- 🚨 **Omurga'da `tg1/0/9` portu (muhtemelen yeni Kabinet3 bağlantı noktası — kesin teyit edilmedi, kullanıcı "emin değilim" dedi) ŞU ANDA, tg1/0/3 ve tg1/0/5 ile BİREBİR AYNI türde kronik/aktif flap yapıyor** (son 300 log kaydının tamamı bu porta ait, 11:42-12:29 TRT aralığında ve devam ediyor). `tg1/0/9`, config'te zaten önceden trunk (VLAN 1-4) olarak hazır bulunuyordu, ek config değişikliği gerekmedi. Trafik hacmi/deseni ve DDM/optik değerleri (normal aralıkta) Kabinet3'ün burada olduğunu destekliyor ama fiziksel port etiketiyle kesin teyit edilmedi.

**Bu, araştırmanın yönünü önemli ölçüde değiştiriyor:** SFP/port değişimi sorunu **çözmedi, sadece Omurga chassis 1 içinde başka bir porta taşıdı**. Bu, "belirli bir SFP veya belirli bir port bozuk" hipotezini zayıflatıp, **"Omurga chassis 1'in kendisinde (güç/ASIC/line card seviyesinde) veya her iki değişimde de ortak kullanılan bir fiber/patch kablosunda" daha genel bir sorun** hipotezini güçlendiriyor.

**✅ Kullanıcı tarafından teyit edildi (2026-08-13):**
- `tg1/0/5`'teki bağlantı **kesin olarak `tg1/0/9`'a taşınmış** (varsayım değil, doğrulandı).
- **Fiber patch kablosu değişmedi** — aynı fiziksel kablo, sadece Omurga ucundaki port değişti. Bu, kablonun kendisinin (bükülme/kirlenme gibi) tek başına ortak faktör olma ihtimalini zayıflatmıyor ama "port değişince sorun da değişir" beklentisini çürüttü — **aynı kablo, farklı port, aynı sorun.**
- **Planlanan sonraki adım (kullanıcı tarafından):** `tg1/0/9`'daki bağlantı **chassis 2'ye taşınacak**. Bu, chassis-1-özgü hipotezi için kritik bir test: sorun chassis 2'de de devam ederse kök neden kabloda/Kabinet3 tarafında/SFP'de demektir; chassis 2'de düzelirse Omurga chassis 1'in kendisinde (donanım) demektir. Kullanıcı taşıma sonrası hangi porta taşındığını bildirecek.

**Kabinet1-2 tarafı (`tg1/0/3`):** Kullanıcı talebiyle bu turda tekrar kontrol edilmedi.

**✅ Kullanıcı teyidi (2026-08-13, sonraki mesaj):** `tg1/0/5`'in `tg1/0/9`'a taşındığı kesinleşti (varsayım değil). Fiber patch kablosu **değişmedi** — aynı fiziksel kablo, sadece Omurga ucundaki port değişti; sorun yine de devam etti. Kullanıcının planı: `tg1/0/9`'daki bağlantıyı **chassis 2'ye taşımak** ve sonucu bildirmek — bu, chassis-1-özgü donanım arızası hipotezini kesin olarak test edecek.

### 🚨🚨🚨 2026-08-13 KRİTİK SONUÇ — Sorun chassis 2'de de devam ediyor, kök neden Omurga chassis 1 DEĞİL

Kullanıcı bağlantıyı `tg1/0/9`'dan **`tg2/0/12`'ye** (chassis 2) taşıdı. Canlı doğrulama (~13:37 TRT):

- **`tg2/0/12` (chassis 2) da flap ediyor** — 9.5 dakikalık pencerede (13:27:55-13:37:26) **62 flap olayı**, aynı yoğun/hızlı desen (bazen saniyede birden fazla geçiş).
- İlk bağlantı denememde `show interface tg2/0/12` **LOS: yes, RX Power: -40.00dBm** (sinyal yok) gösterdi; birkaç dakika sonra port "up" durumuna geçmişti — bu tutarsızlık, bağlantının o an aktif olarak flap ettiğini doğruluyor.
- Kabinet3'e bağlanmayı ilk denediğimde bağlantı zaman aşımına uğradı, ikinci denemede başarılı oldu (port 26 "up" görünüyordu) — bu da bağlantı kararsızlığıyla tutarlı.
- **Önemli teknik detay:** `tg2/0/12`, Omurga config'inde **trunk olarak yapılandırılmamış** (sadece `fiber-auto`/`ddm`, `switchport mode trunk` yok) — bu nedenle VLAN2/3/4 trafiği muhtemelen düzgün geçmiyor olabilir (sadece native/access VLAN1 geçebilir). Bu, "flap" bulgusunu geçersiz kılmıyor (LINE/LINEPROTO seviyesinde fiziksel/link katmanı olayı, VLAN config'inden bağımsız) ama VLAN2-4 üzerindeki cihazların bu test sırasında tam çalışmayabileceği anlamına geliyor — henüz trunk config eklenmedi (kullanıcı onayı gerekir).
- **Ek gözlem:** Taşınan SFP modülü, hem `tg1/0/9`'da hem şimdi `tg2/0/9`'da **aynı seri numarayı** taşıyor (`ANW22071402084`) — yani SFP modülünün kendisi (sadece fiber değil) taşınmış.

**SONUÇ VE HİPOTEZ DEĞİŞİKLİĞİ:** Aynı bağlantı (aynı SFP modülü + aynı, değişmeyen fiber patch kablosu + aynı Kabinet3 ucu) artık **üç farklı Omurga portunda** (tg1/0/5 → tg1/0/9 → tg2/0/12), **hem chassis 1 hem chassis 2'de**, aynı kronik flap sorununu gösterdi. Bu, **Omurga'nın (chassis 1 veya 2, herhangi bir port) kök neden olma ihtimalini çok büyük ölçüde zayıflatıyor.** Kalan en olası şüpheliler:
1. **Taşınan SFP modülünün kendisi** (ANW22071402084) — arızalı/marjinal olabilir, her nereye takılırsa orada soruna yol açıyor olabilir.
2. **Değişmeyen fiber patch kablosu** — kullanıcı "değişmedi" dedi, yani hâlâ şüpheli.
3. **Kabinet3 tarafı (port 26'daki SFP veya Kabinet3'ün kendisi)** — henüz bu taraf hiç değiştirilmedi/izole edilmedi.

**Önerilen sonraki adım:** Omurga tarafında **farklı, bilinen-sağlam bir SFP modülü** ile test edilmesi (aynı port, farklı SFP) — eğer sorun düzelirse SFP suçlu; düzelmezse fiber kablosu veya Kabinet3 tarafı şüpheli hale gelir.

#### 2026-08-13 ~13:53 TRT Güncellemesi — `tg2/0/12` şu anda stabil ve düzgün yapılandırılmış

Kullanıcı `tg2/0/12`'ye trunk config ekledi (`switchport trunk vlan-allowed 1-4` + `switchport mode trunk`, kendi tarafından, `wr` ile kaydedilmiş — 13:49'da "Saving current configuration" logda görüldü). Canlı doğrulama:

- `show interface tg2/0/12`: **up, line protocol up, LOS: no, RX Power: -5.25dBm** (önceki kontrolde LOS: yes, RX -40.00dBm idi — önemli iyileşme, muhtemelen fiziksel olarak yeniden düzgün oturtuldu).
- Sağlıklı trafik akışı: ~34 Mbps real-time input, sadece 3 hata / 1.25M paket.
- Log analizi: son flap `13:44:22`'de, sorgu anına (~13:53) kadar **~9 dakikadır kesintisiz temiz**. (Bu pencerede toplam 114 flap olayı var ama hepsi 13:44'ten önce, geçmişte.)
- Kabinet3 tarafı da (port 26) sorunsuz erişilebilir durumda, iki bağlantı denemesi de (Omurga ve Kabinet3) takılmadan başarılı oldu — önceki oturumlardaki bağlantı kopmaları/timeout'lar bu kontrolde yaşanmadı.

**Değerlendirme:** Umut verici ama **~9 dakika, önceki saatler süren kronik desenleri ekarte etmek için yeterince uzun bir pencere değil.** Bu iyileşmenin config eklenmesinden mi (VLAN/trunk — teorik olarak layer-1 flap'i etkilememeli) yoksa eşzamanlı bir fiziksel müdahaleden (kablonun/SFP'nin yeniden düzgün oturtulması) mi kaynaklandığı netleşmedi — kullanıcıya sorulmalı. **Önerilen: en az 30-60 dakika daha izleyip tekrar kontrol etmek**, kalıcı olduğundan emin olmak için.

**⚠️ Düzeltme (~14:09):** Yukarıdaki 9 dakikalık "temiz" gözlem kalıcı değilmiş — sonraki kontrolde `tg2/0/12` tekrar down/LOS bulundu (bkz. aşağıdaki alt bölüm). Yani iyileşme geçiciydi.

#### 2026-08-13 ~14:09 Güncellemesi — SFP modülleri de ekarte edildi, en güçlü şüpheli artık fiber kablosu

**Kullanıcı teyidi:** Hem Omurga hem Kabinet3 tarafında **sıfır/yeni SFP modülleri** takıldı (`tg2/0/12`'ye ve Kabinet3 port 26'ya). **Sorun yine de devam etti** — kullanıcının kendi ifadesiyle "sorun orada değil" (SFP'de değil).

**Canlı doğrulama sırasında bir tutarsızlık bulundu:** Bu güncellemeden hemen önce `show interface tg2/0/12` çalıştırıldığında port **tekrar down, LOS: yes, RX Power: -40.00dBm** durumundaydı ve **hâlâ eski SFP seri numarasını (ANW22071402084) gösteriyordu** — kullanıcının "yeni SFP taktık" ifadesiyle bire bir örtüşmüyor. Bu net şekilde çözülmedi; olası açıklamalar: (a) yeni SFP henüz tam olarak bu kontrol anında takılı değildi (zamanlama), (b) DDM bilgisi cihaz tarafında anlık güncellenmiyor olabilir, (c) başka bir açıklama. **Bu bir "case" olarak not ediliyor, kesin tespit değil — sonraki oturumda tekrar kontrol edilip netleştirilmeli.**

**Güncel şüpheli sıralaması:**
1. **Fiber patch kablosu** — testler boyunca (3 Omurga portu, 2 chassis, 2 farklı SFP seti) hiç değişmedi. Artık en güçlü şüpheli.
2. **Kabinet3 tarafının kendisi** (switch donanımı veya port 26) — Omurga ucu defalarca değiştirildi/taşındı ama Kabinet3 ucu hiç izole edilmedi (yani Kabinet3'ün farklı bir Omurga bağlantısı/switch'i ile test edilmesi yapılmadı).
3. ~~SFP modülleri~~ — ekarte edildi (yeni modüllerle de sorun sürdü).
4. ~~Omurga switch (chassis/port)~~ — ekarte edildi (3 farklı port/2 chassis'te sorun takip etti).

**Sıcaklık kontrolü (kullanıcı talebiyle):** Anomali bulunamadı, ama veri sınırlı:
- Omurga: chassis1 47°C, chassis2 46°C (alarm eşiği: 100°C)
- Kabinet3: 61°C (alarm eşiği: 120°C)
- Omurga'da ~19 saatlik log (2026-08-12 19:25 – 2026-08-13 14:05) `temp`/`thermal`/`environment` anahtar kelimeleriyle tarandı — **hiçbir kayıt yok** (ne periyodik log ne alarm).
- **Önemli kısıtlama:** Bu platformlarda (`show temperature`) sadece **anlık** okuma var, gerçek bir "24 saatlik sıcaklık trendi/geçmişi" komutu **yok**. Yani "anomali yok" sonucu, sürekli izlenen bir veri setine değil, birkaç anlık ölçüme ve alarm-log yokluğuna dayanıyor — sıcaklığın belirli saatlerde (örn. öğlen sıcak saatlerde) yükselip flap'i tetikleyip tetiklemediğini bu şekilde kesin olarak dışlayamıyoruz.

### CASE-008 — Kabinet3 cihaz saati senkron değildi (NTP yok/çalışmıyordu)
**Durum:** ✅ Kapatıldı (2026-08-11, Kabinet3 için) — **Bu, kural olarak "cihazlarda değişiklik yapma" ilkesine kullanıcı onayıyla yapılan bir istisnadır.**

`show clock` çıktısı `07:10:25 EET Fri Jan 02 1970` gösteriyordu — cihaz gerçek tarihi bilmiyordu. Bu yüzden Kabinet3 loglarındaki zaman damgaları, Kabinet1-2'nin gerçek tarih/saat gösteren loglarıyla (CASE-007'deki olayların ne zaman olduğu) doğrudan karşılaştırılamıyordu.

**Yapılan değişiklik (kullanıcı onayıyla):** Kabinet3'e `ntp server 162.159.200.1` (Cloudflare public NTP) konfigüre edildi. ~75 saniye içinde cihaz saati doğru tarih/saate senkronize oldu: `20:13:17 +03 Tue Aug 11 2026`. Detaylı önce/sonra, doğrulama ve rollback planı: `logs/changes/2026-08-11_Kabinet3_ntp-saat-senkronizasyonu.md`. Oturum kaydı: `logs/2026-08-11/Kabinet3_200500.md`.

✅ **Kalıcılık teyit edildi:** Kullanıcı kendi tarafından (vty 1, 192.168.2.18 üzerinden) `wr` komutunu çalıştırmış (log'da görüldü, 17:19:29 UTC). `show startup-config` kontrol edildi, `ntp server 162.159.200.1` satırı startup-config'te de mevcut — değişiklik reboot'tan sonra da kalıcı.

**Not:** Diğer switch'lerde (Kabinet1-2, Kabinet1-1, Kabinet2, Yonetim_ALT, Omurga) NTP durumu henüz kontrol edilmedi — aynı sorun onlarda da olabilir.

### CASE-009 — `show logging` zaman damgaları UTC gösteriyor, `show clock` doğru yerel saati gösteriyor (cihaz davranışı, config ile düzeltilemedi)
**Durum:** Açık — bilgi amaçlı, kullanıcı yanlışlıkla "NTP sunucusu Türkiye'de değil" sandı, gerçek sebep bu
Kullanıcı NTP düzeltmesinden sonra `show logging` çıktısını kontrol edip zaman damgalarının (`17:24`, `17:26` gibi) gerçek saatten (`20:2x`) farklı olduğunu fark etti ve bunu "timezone yanlış" olarak yorumladı. Doğrulama yapıldı: aynı anda `show clock` = `20:26:35 +03`, `show logging`'deki en yeni kayıt = `17:26:33` — **tam olarak 3 saat fark**, yani `clock timezone Istanbul` ofseti.

**Sonuç:** `show clock` doğru çalışıyor (yerel Türkiye saati, +03). Ama bu switch OS'unun `show logging` alt sistemi zaman damgalarını **UTC** olarak yazıyor, timezone ofsetini uygulamıyor. Bu, NTP sunucusunun coğrafi konumuyla **ilgisi olmayan**, cihazın loglama alt sisteminin bir davranışı/kısıtı. `clock ?` komutunda sadece `timezone` alt komutu var; `service timestamps ...` (Cisco'da log zaman damgası formatını ayarlayan komut) bu OS'te tanınmıyor (`% Unrecognized command`) — yani bunu değiştirecek bir config seçeneği bulunamadı.

**Öneri:** NTP sunucusunu Türkiye merkezli bir adrese değiştirmenin bu davranışı düzeltmeyeceği için **162.159.200.1 (Cloudflare) korunması öneriliyor** — zaten doğrulanmış düşük gecikmeli (~20ms) ve çalışan bir kaynak. `show logging` okurken **+3 saat** eklenmesi gerektiği unutulmamalı (gerçek Türkiye saatine çevirmek için).

**CASE-007 ile ilişkisi:** CASE-007'deki tüm flap kayıtları NTP düzeltmesinden ÖNCEYDİ (1970 epoch damgalı), bu yüzden bu UTC/yerel farkı o kayıtları etkilemiyor. Ama **bundan sonra** Kabinet3'te (veya aynı firmware'i kullanan başka switch'lerde) toplanacak yeni log kayıtlarını Kabinet1-2 gibi diğer cihazların loglarıyla karşılaştırırken bu +3 saatlik farkın akılda tutulması gerekiyor.

### CASE-006 — (İkincil / Güvenlik) Omurga'da parola düz metin olarak saklanıyor
**Durum:** Açık — bilgi amaçlı, muhtemelen ana sorunla ilgisiz
Diğer switch'lerde (Kabinet1-1, Kabinet1-2, Kabinet3, Yonetim_ALT) kullanıcı parolası hashlenmiş (`password 5 $1$...`) saklanırken, Omurga config'inde `username admin password 0 ...` şeklinde **tip 0 (düz metin/plaintext)** olarak saklanıyor. Bu dokümanda parola değeri paylaşılmıyor/tekrar yazılmıyor.
**Öneri (aksiyon değil, öneri):** Parolanın rotasyonu ve mümkünse encrypted secret tipine geçirilmesi. Bu doküman kapsamında bir değişiklik yapılmayacak.

### CASE-010 — (Güvenlik) Kenar switch'ler arasında SSH host key paylaşımı
**Durum:** Açık — güvenlik bulgusu, ana sorunla muhtemelen ilgisiz ama önemli
2026-08-11'de Kabinet1-1, Kabinet2, Yonetim_ALT'a bağlanılırken host key fingerprint'leri karşılaştırıldı:

| Switch | SSH Host Key Fingerprint |
|---|---|
| Kabinet3 | `ssh-ed25519 SHA256:QzHtfKQnjI3RXufdaryIa0wxiCZJU/fPrwQDtQXgP/g` |
| Kabinet2 | `ssh-ed25519 SHA256:QzHtfKQnjI3RXufdaryIa0wxiCZJU/fPrwQDtQXgP/g` ← **Kabinet3 ile birebir aynı** |
| Kabinet1-1 | `ecdsa-sha2-nistp256 SHA256:GcjQPuCFm6xwSpLlPT5SvMtLBHAVCOuA7LhJjwW1j/0` |
| Yonetim_ALT | `ecdsa-sha2-nistp256 SHA256:GcjQPuCFm6xwSpLlPT5SvMtLBHAVCOuA7LhJjwW1j/0` ← **Kabinet1-1 ile birebir aynı** |
| Kabinet1-2 | `ecdsa-sha2-nistp256 SHA256:GcjQPuCFm6xwSpLlPT5SvMtLBHAVCOuA7LhJjwW1j/0` ← **Kabinet1-1 ve Yonetim_ALT ile de birebir aynı** (2026-08-13'te doğrulandı) |
| Omurga | `ssh-ed25519 SHA256:aY95NpQ8NOmsglHdU6IeGRTs83rfnIgmz9hUjWIJXAw` (farklı) |

Her SSH sunucusunun **benzersiz** bir host key üretmesi beklenir (genelde ilk açılışta cihaz üzerinde otomatik üretilir). Farklı cihazların **aynı** host key'e sahip olması normal değil — büyük olasılıkla bu switch modelinin firmware imajında **fabrika varsayılanı/sabit kodlanmış bir SSH anahtarı** var ve cihazlar ilk kurulumda kendi benzersiz anahtarlarını üretmiyor. Not: aynı anahtarı paylaşan 48-port modeller (Kabinet1-1, Kabinet1-2, Yonetim_ALT — hepsi `ANW-2712-48TP4X`) ile 24-port modeller (Kabinet2, Kabinet3 — hepsi `ANW-2712-24TP4X`) kendi aralarında iki ayrı grup oluşturuyor; bu, anahtarın donanım modeline/üretim partisine bağlı olarak sabit kodlanmış olabileceğini düşündürüyor. Bu bir **güvenlik zafiyeti**: host key doğrulaması cihaz kimliğini gerçek anlamda doğrulamıyor, aynı modelden bir cihaz (veya bu anahtarı ele geçiren biri) diğerinin yerine geçebilir (MITM riski).
**Öneri (aksiyon değil, öneri):** Üreticiye (Angora Networks) bu konu sorulmalı; mümkünse her cihazda host key'in yeniden/benzersiz üretilmesi (genelde `crypto key generate` benzeri bir komutla) sağlanmalı. Bu doküman kapsamında bir değişiklik yapılmadı.

### CASE-011 — Bazı switch'lerde saat NTP olmadan doğru, sebebi belirsiz
**Durum:** Açık — bilgi amaçlı
Kabinet2 ve Yonetim_ALT'ın saatleri **doğru** (`show clock` gerçek tarih/saat gösteriyor), ama ikisinde de config'te veya log geçmişinde **hiçbir `ntp`/`sntp` referansı yok** (grep ile doğrulandı). Yonetim_ALT'ın log buffer'ı (1024/1024, dolu) baştan sona (13+ gün) tamamen gerçek tarihli — hiç "1970" damgalı kayıt yok. Bu, Kabinet3/Kabinet1-1'de gördüğümüz "coldstart sonrası 1970'e sıfırlanma, NTP ile düzeltilene kadar" davranışından farklı.
**Güncelleme (2026-08-13):** Kabinet1-2'de de aynı desen görüldü — NTP config'i yokken saat zaten doğruydu (`13:10:17 EEST Thu Aug 13 2026`). Kullanıcı talebiyle tutarlılık amacıyla yine de `ntp server 162.159.200.1` eklendi (bkz. `logs/changes/2026-08-13_Kabinet1-2_ntp-konfigurasyonu.md`). Yani şu ana kadar **saati NTP olmadan doğru olan switch'ler:** Kabinet2, Yonetim_ALT, Kabinet1-2 (üçü de artık NTP config'ine sahip); **saati bozuk olup NTP ile düzeltilen switch'ler:** Kabinet3, Kabinet1-1.
**Hipotez (doğrulanmadı):** Bu ünitelerin donanım saati (RTC + pil) çalışıyor olabilir, Kabinet3/Kabinet1-1'inki ise (pil bitmiş/arızalı olabilir) her soğuk açılışta sıfırlanıyor olabilir. Fiziksel/donanımsal bir fark — yazılımla teşhis edilemez, üretici desteği veya fiziksel inceleme gerekir.

---

## 6. Açık Sorular / İhtiyaç Duyulan Ek Bilgiler

- [x] Kabinet2 config dosyasının tekrar paylaşılması (CASE-001) — 2026-08-11 tamamlandı
- [ ] `image.png`'deki ping'in atıldığı makinenin IP/VLAN'ı ve işletim sistemi (CASE-002)
- [ ] 10.64.0.22'ye `tracert`/`traceroute` çıktısı (CASE-002)
- [ ] Omurga'da `ip routing` durumu ve `show ip route` çıktısı (CASE-003)
- [ ] Firewall'daki VLAN1-4 arayüz ve route/NAT konfigürasyonu (henüz hiç paylaşılmadı)
- [ ] Kesinti günü (2026-08-11) için kesintisiz, tam log kaydı — Kabinet1-2 ve Omurga (CASE-005)
- [ ] "Benzer şeyler diğer kenar switch'lerde de oluyor" denildi — hangi switch'ler, ne zaman, log var mı?
- [ ] Omurga iki chassis arasındaki stack durumu sağlıklı mı (`show stack` / eşdeğeri)?
- [ ] Kabinet1-2 `tg0/49` ve Omurga `tg1/0/3` için interface sayaçları (CRC/error/flap) ve SFP/DDM diagnostik (sıcaklık, tx/rx power)
- [x] SSH erişim bilgileri — 2026-08-11 paylaşıldı (`usak_topoloji.md`), Kabinet3'e canlı erişimle teşhis yapıldı
- [x] Omurga'ya SSH ile bağlanıp `tg1/0/3` ve `tg1/0/5` portlarının log/counter/optik durumu — 2026-08-11 tamamlandı, **her ikisi de kronik flap ediyor** (CASE-007)
- [x] Omurga chassis 1 genel sağlık durumu (güç, fan, sıcaklık) — 2026-08-11 kontrol edildi, hepsi normal, temel donanım arızası değil (CASE-007)
- [x] Kabinet1-1, Kabinet2, Yonetim_ALT'ta `show logging` taraması — 2026-08-11 tamamlandı, **üçü de şu anda temiz** (CASE-007)
- [ ] Kabinet1-2'ye canlı SSH ile bağlanıp kendi tarafından `show logging`/`show interface` ile teyit (henüz sadece web konsol ekran görüntüsü var)
- [ ] Fiziksel: tg1/0/3 ve tg1/0/5 SFP'lerinin çapraz değişimi (sorun port mu SFP mi takip ediyor?) — müşteri onayı + saha erişimi gerekir
- [ ] Angora Networks üretici desteğine chassis 1 için ticket açılması değerlendirilmeli
- [ ] Kabinet1-1'in bugün ~18:43 civarında (tahmini) yeniden başlamış (coldstart) olması — planlı mıydı, sebebi biliniyor mu? (kullanıcıya soruldu)
- [ ] Kabinet2 ve Yonetim_ALT'ın saati NTP olmadan nasıl doğru — RTC pili farkı mı? (CASE-011, kullanıcıya soruldu)
- [ ] SSH host key paylaşımı (CASE-010) — üreticiye sorulmalı, güvenlik riski olarak değerlendirilmeli
- [x] Kabinet3'ün Omurga tarafındaki yeni bağlantı noktası — kullanıcı `tg1/0/9` olduğunu teyit etti
- [x] Fiber/patch kablosu değişti mi — kullanıcı **değişmedi** dedi (aynı kablo, farklı port)
- [ ] Kabinet1-2 (`tg1/0/3`) şu anki durumu tekrar kontrol edilmeli — bu turda bakılmadı
- [ ] **YENİ (2026-08-13, öncelikli):** Omurga tarafında bilinen-sağlam farklı bir SFP ile test — SFP mi yoksa fiber kablosu/Kabinet3 tarafı mı sorunlu, izole edilmeli
- [ ] **YENİ:** Kabinet3 tarafındaki port 26 SFP'sinin DDM/optik değerleri kontrol edilmeli (henüz bakılmadı, sadece Omurga tarafı incelendi)
- [x] `tg2/0/12`'ye trunk config eklendi — kullanıcı tarafından yapıldı
- [ ] **YENİ (öncelikli):** Fiber patch kablosunun değiştirilmesi/farklı bir kabloyla test edilmesi
- [ ] **YENİ:** Kabinet3 tarafının (switch/port) izole edilmesi — örn. farklı bir Omurga bağlantısı yerine geçici olarak başka bir test cihazına bağlanıp bağlanmadığının denenmesi
- [ ] **YENİ:** "Yeni SFP takıldı" ile "hâlâ eski seri numarası görünüyor" tutarsızlığının netleştirilmesi — sonraki oturumda tekrar kontrol edilmeli
- [ ] Kabinet3 (ve diğer switch'ler) için doğru "show interface counters/status/transceiver" komut sözdizimi — bu OS'e özgü, bir sonraki oturumda keşfedilecek (CASE-007)
- [x] Kabinet3 NTP senkronizasyonu — 2026-08-11 uygulandı ve doğrulandı (CASE-008)
- [x] Kabinet3'teki NTP değişikliğinin kalıcılığı — kullanıcı kendi `wr` komutuyla startup-config'e kaydetmiş, teyit edildi
- [ ] Diğer cihazlarda (Kabinet1-2, Kabinet1-1, Kabinet2, Yonetim_ALT, Omurga) NTP / saat senkronizasyonu durumu kontrol edilmeli
- [ ] `show logging` UTC gösterimi diğer switch'lerde de var mı kontrol edilmeli (CASE-009) — log analizlerinde +3 saat düzeltmesi unutulmamalı

---

## 7. Sonraki Adımlar (Salt-Okunur, Onay Sonrası)

Aşağıdaki adımların hiçbiri config değişikliği içermiyor; SSH erişimi sağlandığında ve kullanıcı onayıyla çalıştırılabilir:

1. Omurga: `show ip route`, `show running-config | include ip routing`
2. Omurga: `show mac address-table vlan 1` (loop/flap kontrolü)
3. Omurga: `show spanning-tree` (detay, TCN sayaçları, root port geçmişi)
4. Kabinet1-2 ve Omurga tg1/0/3: `show interface` sayaçları + DDM/transceiver diagnostik
5. Kabinet1-2: tam log (tüm seviyeler, 2026-08-11 16:30–16:50 aralığı)
6. Kaynak host'tan `tracert 10.64.0.22`

---

## 8. Değişiklik Geçmişi

| Tarih | Değişiklik |
|---|---|
| 2026-08-11 | Doküman oluşturuldu. İlk config/topoloji/log/ping analizi yapıldı. CASE-001 – CASE-006 açıldı. |
| 2026-08-11 | Kabinet2 config'i güncellendi ve incelendi. CASE-001 kapatıldı — uplink (tg0/25 ↔ omurga tg2/0/8) diğer switch'lerle tutarlı, yeni anomali yok. Tüm kenar switch'lerde uplink portlarında loop-detect olmaması genel/tutarlı bir desen olarak doğrulandı. |
| 2026-08-11 | Kullanıcı SSH erişim bilgilerini paylaştı. Kabinet3'e canlı SSH ile bağlanılıp salt-okuma teşhis komutları çalıştırıldı (oturum kaydı: `logs/2026-08-11/Kabinet3_195500.md`). **CASE-007 açıldı:** tg0/25 uplink portu kronik/yoğun şekilde flap ediyor (1024 satırlık log buffer'ının tamamı bu olaylarla dolu) — Kabinet1-2'dekinden çok daha ciddi. Kabinet1-2 ve Kabinet3'ün ikisinin de Omurga chassis 1'e bağlı olduğu fark edildi — chassis 1 kaynaklı ortak bir sebep hipotezi ortaya çıktı, teyit gerekiyor. **CASE-008 açıldı:** Kabinet3 saati senkron değil (NTP yok), log zaman damgaları gerçek tarihi yansıtmıyor. |
| 2026-08-11 | Kullanıcı, "cihazlarda değişiklik yapma" kuralına **açık istisna onayı** vererek Kabinet3'e NTP konfigürasyonu yapılmasını istedi. `ntp server 162.159.200.1` uygulandı, doğrulandı (saat gerçek tarihe senkron oldu). **CASE-008 kapatıldı** (Kabinet3 için). Değişiklik kaydı: `logs/changes/2026-08-11_Kabinet3_ntp-saat-senkronizasyonu.md`, oturum kaydı: `logs/2026-08-11/Kabinet3_200500.md`. |
| 2026-08-11 | Kullanıcı "timezone yanlış" dedi ve Türkiye merkezli NTP sunucusu istedi. İnceleme: aslında `show clock` doğruydu, ama `show logging`'in zaman damgalarını UTC (timezone ofsetsiz) yazdığı ortaya çıktı — NTP sunucusunun ülkesiyle ilgisiz bir cihaz/firmware davranışı, düzeltecek bir config seçeneği bulunamadı. **CASE-009 açıldı.** Ayrıca kullanıcının kendi `wr` komutuyla NTP değişikliğini zaten startup-config'e kaydettiği teyit edildi (açık soru kapandı). |
| 2026-08-11 | Kullanıcı, `usak_topoloji.md`'deki Omurga IP'sinin yanlış yazıldığını (Kabinet3 ile aynı, 10.64.0.28) fark edip düzeltti (doğrusu: 10.64.0.6). Omurga'ya canlı SSH ile bağlanıldı (salt-okuma, oturum kaydı: `logs/2026-08-11/Omurga_203400.md`). **🚨 CASE-007 kesin olarak doğrulandı:** Omurga'nın kendi logları, chassis 1'e bağlı **hem tg1/0/3 (Kabinet1-2) hem tg1/0/5 (Kabinet3)** portlarının son ~5.5 saattir (15:07'den itibaren, sorgu anı 20:42'de hâlâ devam ediyor) sürekli ve yoğun şekilde flap ettiğini gösterdi (tg1/0/5: ~528, tg1/0/3: ~470 flap döngüsü), buna karşın chassis 2'ye bağlı tg2/0/10 (Kabinet1-1) sadece 1 kez flap etti. Chassis 1/2 güç, fan ve sıcaklık durumu kontrol edildi — hepsi normal, temel donanım arızası değil. Sorun artık kenar switch'lerden değil, **Omurga chassis 1'in kendisinden (SFP/optik, port/ASIC veya line card düzeyinde)** kaynaklanıyor gibi görünüyor — kesin kök neden hâlâ teyit edilmedi. |
| 2026-08-11 | Kullanıcı sorusu üzerine chassis 1/chassis 2 arası yazılım versiyon uyuşmazlığı ve stack senkronizasyon sorunu ihtimali kontrol edildi (oturum kaydı: `logs/2026-08-11/Omurga_205600.md`). Bu platformda chassis-bazlı versiyon karşılaştırma komutu yok (`show switch/stack/module/inventory/unit/slot` mevcut değil); `show version`/`show kernel-info` tek/birleşik versiyon veriyor. `show oir-information` iki chassis'in de aynı donanım modeli olduğunu doğruladı. ~5.5 saatlik logda versiyon/senkronizasyon anahtar kelimeleri için arama yapıldı, **hiçbir eşleşme yok**. Sonuç: bulgu yok, ama CLI kısıtı nedeniyle %100 dışlanamıyor. |
| 2026-08-11 | Kullanıcı talebiyle Kabinet1-1, Kabinet2, Yonetim_ALT'a canlı SSH ile bağlanılıp config/saat/log kontrolü yapıldı (oturum kayıtları: `logs/2026-08-11/Kabinet1-1_215000.md`, `Kabinet2_220400.md`, `Yonetim_ALT_224100.md`). **Sonuç: üçünde de config drift yok, üçü de ŞU ANDA temiz/stabil (chassis 2 doğrulandı, CASE-007 hipotezini güçlendiriyor).** Kabinet1-1'de kullanıcının kendi başına NTP (`ntp server 162.159.200.1`) uyguladığı görüldü; ayrıca bugün ~18:43 civarında açıklanmamış bir coldstart tespit edildi (Omurga'daki tg2/0/10 tek flap olayıyla zaman olarak örtüşüyor). Kabinet2'de geçmişte (tahminen ~12 gün önce) yoğun ama artık aktif olmayan tg0/25 flapping'i bulundu. **Yeni bulgular:** CASE-010 (güvenlik) — Kabinet2/Kabinet3 ve Kabinet1-1/Yonetim_ALT ikişerli gruplar halinde birebir aynı SSH host key'i paylaşıyor, muhtemelen fabrika varsayılanı anahtar; CASE-011 — Kabinet2 ve Yonetim_ALT'ın saati NTP config'i olmadan doğru, sebebi belirsiz (muhtemelen RTC pili farkı). Kullanıcıya üç açık soru soruldu: coldstart sebebi, saat gizemi, host key paylaşımı. |
| 2026-08-11/12 | Repo git ile versiyonlandı ve GitHub'a (`SeFo19/Ozdoku-Config`, private) push edildi. Push öncesi `usak_topoloji.md` ve `config/Omurga config.txt`'teki düz metin SSH parolaları redakte edildi (repo genelinde grep ile doğrulandı, başka yerde parola kalmadı). |
| 2026-08-13 | Kabinet1-2'ye ilk kez canlı SSH bağlantısı denendi (saat düzeltmesi talebi üzerine) — bağlantı 3 kez koptu/zaman aşımına uğradı (muhtemelen tg1/0/3'ün o anki aktif flap'i nedeniyle), NTP değişikliği tamamlanamadı. Host key'i Kabinet1-1/Yonetim_ALT ile aynı çıktı (CASE-010'a üçüncü örnek eklendi). |
| 2026-08-13 | **🚨 Müşteri fiziksel SFP değişimi yaptı:** Omurga tarafında Kabinet3'ün SFP'si değiştirildi (sorun düzelmedi), sonra Kabinet3 tarafında port 25'in SFP'si değiştirilip bağlantı port 26'ya taşındı (~6 dk kesinti sonrası düzeldi). Canlı doğrulama: Kabinet3 tarafı (port 26) artık temiz; ama Omurga tarafında `tg1/0/5` tamamen boşaldı ve bağlantının taşındığı düşünülen `tg1/0/9` portu **aynı kronik flap sorununu göstermeye devam ediyor**. **Sonuç: SFP/port değişimi sorunu çözmedi, Omurga chassis 1 içinde başka bir porta taşıdı** — kök nedenin tek bir SFP/porttan çok, chassis 1'in genelinde veya ortak kullanılan fiber/patch kablosunda olabileceği hipotezi güçlendi. Değişiklik kaydı: `logs/changes/2026-08-13_Kabinet3-Omurga_sfp-degisimi-ve-port-tasima.md`. |
| 2026-08-13 | Kullanıcı, `tg1/0/9`'un kesinlikle `tg1/0/5`'in taşınmış hali olduğunu ve **fiber patch kablosunun değişmediğini** teyit etti (aynı kablo, farklı port, aynı sorun). Kullanıcının planı: bağlantıyı **chassis 2'ye taşıyıp** sonucu bildirmek — chassis-1-özgü hipotezi için kesin test. Kabinet1-2'ye tekrar bağlanıldı (bu kez sorunsuz) ve saat/NTP tutarlılığı için `ntp server 162.159.200.1` uygulandı ve `wr` ile kalıcı hale getirildi (oturum: `logs/2026-08-13/Kabinet1-2_131000.md`, değişiklik: `logs/changes/2026-08-13_Kabinet1-2_ntp-konfigurasyonu.md`). Kabinet1-2'nin saati de (Kabinet2/Yonetim_ALT gibi) NTP olmadan zaten doğruydu — CASE-011'e üçüncü örnek eklendi. |
| 2026-08-13 | **🚨🚨🚨 KRİTİK SONUÇ:** Kullanıcı bağlantıyı `tg2/0/12`'ye (chassis 2) taşıdı. Canlı doğrulama: **`tg2/0/12` de flap ediyor** (9.5 dakikada 62 olay, LOS anları dahil) — aynı SFP modülü (SN ANW22071402084) ve değişmeyen fiber kablosu artık üç farklı Omurga portunda, iki farklı chassis'te aynı sorunu gösterdi. **Omurga'nın (herhangi bir chassis/port) kök neden olma ihtimali büyük ölçüde ekarte edildi.** Kalan şüpheliler: taşınan SFP modülünün kendisi, fiber kablosu, veya Kabinet3 tarafı (port 26). Önerilen sonraki adım: Omurga tarafında farklı/sağlam bir SFP ile test. |
| 2026-08-13 | Kullanıcı `tg2/0/12`'ye doğru trunk config'i ekledi (`switchport trunk vlan-allowed 1-4 mode trunk`, kendi tarafından, `wr` ile kaydedildi). Canlı doğrulama: **LOS artık yok, RX Power normal (-5.25dBm)** (önceki -40dBm'den iyileşme), ~9 dakikadır flap yok, sağlıklı trafik akışı. Kabinet3 tarafı da (port 26) sorunsuz erişilebilir. **Henüz kalıcı olduğu teyit edilmedi (kısa pencere), izleme öneriliyor.** |
| 2026-08-13 | Kullanıcı sordu: "sorunun nerede olduğunu düşünüyorsun?" — mevcut kanıtlara dayanarak sıralı hipotez sunuldu (en olası: SFP modülü veya kirli/gevşek konnektör; Omurga donanımı ekarte). |
| 2026-08-13 | **🚨🚨🚨🚨 SFP DE ELENDİ:** Kullanıcı hem Omurga hem Kabinet3 tarafında sıfır/yeni SFP taktı — sorun devam etti. Canlı kontrolde `tg2/0/12` tekrar down/LOS bulundu (önceki "9 dakikalık temiz" gözlem kalıcı değilmiş) ve hâlâ eski SFP seri numarası görünüyordu (tutarsızlık, çözülmedi). **Güncel en güçlü şüpheli: fiber patch kablosu (hiç değişmedi) veya Kabinet3 tarafı.** Sıcaklık kontrolü yapıldı (kullanıcı talebi): Omurga 46-47°C, Kabinet3 61°C, ikisi de normal; ~19 saatlik logda sıcaklık/alarm kaydı yok — ama platform gerçek sıcaklık geçmişi/trendi tutmuyor, sadece anlık okuma var, bu yüzden "anomali yok" sonucu sınırlı veriye dayanıyor. **TL;DR bölümü investigation.md'nin başına eklendi**, aynısı `topoloji_diyagram.md` ve değişiklik kayıtlarına da eklendi. |
