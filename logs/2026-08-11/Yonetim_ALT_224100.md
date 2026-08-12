# SSH Oturum Kaydı
- **Cihaz:** Yonetim_Alt (10.64.0.75)
- **Platform:** ANW-2712-48TP4X, Software hotfix/7.3.2 (r1275 382d3e1), HW V1.1
- **Ajan/Operator:** Claude Code (AI ajan) — RPM Teknoloji, network investigation
- **Oturum başlangıç:** 2026-08-11 ~22:41 TRT
- **Oturum bitiş:** 2026-08-11 ~22:42 TRT
- **Gerekçe / Talep:** Kullanıcı talebi kapsamında (Kabinet2, Kabinet1-1, Yonetim_ALT kontrolü) — config, saat, CASE-007 chassis 2 temizliği kontrolü.
- **Sonuç:** Başarılı, salt-okuma. Hiçbir config değişikliği yapılmadı.

## Bağlantı Notu
⚠️ **Host key Kabinet1-1 ile birebir aynı:** `ecdsa-sha2-nistp256 SHA256:GcjQPuCFm6xwSpLlPT5SvMtLBHAVCOuA7LhJjwW1j/0` — bkz. `investigation.md` CASE-010.

## Eylem Kaydı
| # | Zaman | Komut | Tür | Sonuç | Notlar |
|---|-------|-------|-----|-------|--------|
| 1 | 22:41 | (host key onayı — Kabinet1-1 ile aynı fingerprint) | bağlantı | OK | |
| 2 | 22:41 | `terminal length 0`, `show version`, `show clock`, `show running-config`, `show logging`, `exit` | salt-okuma | OK | |

## Komut Çıktıları

### `show version`
```
Software Version: hotfix/7.3.2 (r1275 382d3e1)
System MAC Address: 98:06:37:a5:00:11
PID: ANW-2712-48TP4X, Hardware Version: V1.1, Serial Number: 25052248B0203
```

### `show clock`
```
22:41:57  EEST Tue Aug 11 2026
```
✅ Saat doğru. `ntp`/`sntp` yine config'te ve loglarda hiç geçmiyor (grep, 0 sonuç). **Ancak bu switch'te log buffer'ı (1024/1024, dolu/wrap olmuş) baştan sona (en eski kayıt: 2026-07-29 14:38) tamamen gerçek tarihli** — hiçbir "1970" damgalı kayıt yok. Bu, diğer switch'lerden (Kabinet3, Kabinet1-1, Kabinet2 — hepsinde coldstart sonrası 1970'e sıfırlanma vardı) farklı: **bu cihazın saati muhtemelen en az 13 gündür (belki daha uzun süredir, buffer'ın kapsadığı süre) sürekli doğru** — muhtemel açıklama: bu ünitenin donanım saati (RTC/pil) çalışıyor, diğerlerininki (özellikle Kabinet3, Kabinet1-1) çalışmıyor/pili bitmiş olabilir. **Doğrulanmadı, fiziksel/üretici bilgisi gerekir.**

### `show running-config` — repo'daki `config/Yonetim_ALT.txt` ile karşılaştırma
`diff` ile karşılaştırıldı: **tam eşleşme, fark yok.**

### `show logging` (1024/1024 — buffer dolu/wrap olmuş, en eski kayıt 2026-07-29 14:38)
- **`tengigabitEthernet0/52` (Omurga `tg2/0/4`'e giden uplink) için sadece 2 kayıt var (1 down + 1 up = 1 flap):**
  - `2026 Jul 30 15:15:38` down
  - `2026 Jul 30 15:20:33` up (~5 dakika kesinti)
- Bu tek olay dışında **tg0/52'de hiç flap yok** — ~13 günlük kapsamda tamamen stabil, bugün dahil hiçbir olay yok.
- İlginç not: Bu tek flap (30 Temmuz) Kabinet2'nin son tarihsel flap'iyle (tahmini 30 Temmuz civarı) zamansal olarak yakın — tesadüf olabilir, teyit edilmedi.

## Sonuç Değerlendirmesi
- ✅ Config drift yok
- ✅ Saat doğru, muhtemelen sürekli çalışan bir RTC sayesinde (NTP yok)
- ✅ Uplink (`tg0/52`) neredeyse tamamen temiz — sadece 30 Temmuz'da tek, izole bir 5 dakikalık kesinti, güncel/kronik bir flap yok
- 🔴 **Güvenlik: SSH host key Kabinet1-1 ile aynı** (CASE-010)

## Hassas Veri Notu
Bu oturumda `usak_topoloji.md`'de paylaşılan admin kullanıcı adı/parola kullanıldı. Parola bu log dosyasında `[REDACTED]`.
