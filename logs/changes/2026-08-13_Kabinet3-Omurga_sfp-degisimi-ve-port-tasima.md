# Değişiklik Özeti — Kabinet3 ↔ Omurga (fiziksel müdahale)

- **Tarih/Saat:** 2026-08-13, ~09:22 - 09:29 TRT (Kabinet3 tarafı), saat aralığı Omurga tarafı için netleşmedi
- **Cihazlar:** Kabinet3 (10.64.0.28), Omurga (10.64.0.6)
- **Kim yaptı:** **Müşteri/kullanıcı — sahada fiziksel müdahale** (SFP modül değişimi, fiber/port taşıma). Bu, AI ajanı (Claude Code) tarafından SSH üzerinden yapılan bir config değişikliği DEĞİLDİR — ajan sadece sonucu salt-okuma komutlarla doğrulayıp bu kaydı oluşturdu.
- **İlgili oturum/doğrulama logları:** Bu dosyanın hazırlanması sırasında çalıştırılan salt-okuma doğrulama komutları ayrı bir oturum logu olarak kaydedilmedi (kullanıcının sözlü bildirimi + canlı doğrulama tek mesajda birleştirildi) — çıktı özetleri aşağıda.
- **Gerekçe:** CASE-007 kapsamında önerilen "SFP çapraz test" adımının müşteri tarafından sahada uygulanması — sorunun port mu, SFP mi, yoksa başka bir şey mi olduğunu anlamak için.
- **Etki alanı:** Kabinet3'ün Omurga'ya trunk uplink bağlantısı (VLAN 1-4, tüm downstream Dokuma/IOT trafiği).

## Kullanıcının Bildirdiği Olay Sırası (ham bildirim)
1. Omurga'da, Kabinet3'ün bağlı olduğu SFP değiştirildi (port değişmedi, sadece modül) → **ağ sorunu tekrar nüksetti** (düzelmedi).
2. Kabinet3'te, port 25'e bağlı SFP değiştirildi → değişiklik sırasında **1-2 dakika switch erişimi kesildi**, sonrasında ağ ve switch erişimi geri geldi.
3. Kabinet3'te SFP şu anda **port 26'da** takılı (port 25'ten taşınmış).

## Canlı Doğrulama (2026-08-13, ~09:52 - 12:56 TRT arası, salt-okuma)

### Kabinet3 tarafı — `show interface brief` (09:52 civarı)
```
TeE0/25    ETH   down    PD      --      --      --           --      --
TeE0/26    ETH   up      none    10000M  FULL    OFF          ON      --
```
✅ Doğrulandı: aktif uplink artık **port 26**, port 25 down/boşta.

### Kabinet3 tarafı — `show logging` (son olaylar)
- `09:22:53` - `09:28:38` arası (~6 dakika): **port 25'te çok yoğun flap** (yüzlerce down/up, SFP değişimi sırasında beklenen fiziksel bozulma + muhtemelen ilk SFP/port denemesi de sorunlu çıktı) — kullanıcının bildirdiği "1-2 dakika" kesintiden biraz daha uzun görünüyor, kesin değil.
- `09:28:38`: port 25 son kez down.
- `09:28:47`: **port 26 up** — yeni bağlantı burada başladı.
- `09:28:47`'den `09:52:04`'e (bu doğrulama anı) kadar **port 26'da hiç flap yok** — ~23 dakika temiz.

### Omurga tarafı — `show interface tg1/0/5` (12:56 civarı)
```
tg1/0/5 is down, line protocol is down
```
⚠️ Kabinet3'ün eski Omurga bağlantı noktası (tg1/0/5) **tamamen down, trafik yok** — beklenen, çünkü fiber/SFP başka bir porta taşınmış.

### Omurga tarafı — `show logging 1 300` (en güncel 300 kayıt, 11:42 - 12:29 TRT)
**300/300 kaydın tamamı `tg1/0/9` için** — bu portun **şu anda, aktif olarak, tg1/0/3 ve tg1/0/5'teki ile birebir aynı türde kronik flap yaptığını** gösteriyor.

### Omurga tarafı — `show interface tg1/0/9` (12:56 civarı, "up" anı yakalandı)
```
tg1/0/9 is up, line protocol is up, 10000Mb/s Full-duplex
Received 8342874 packets — 239 error, 239 FCS (oran: ~%0.003, ihmal edilebilir)
DDM: TX -2.63dBm, RX -4.96dBm, Temp 47.0°C, Voltage 3.27V, Bias 33.39mA (hepsi normal eşik aralığında)
Trafik hacmi/deseni (~20 Mbps, yoğun 1024-1518B paket) Kabinet3'ün downstream trafiğiyle tutarlı.
```

## Sonuç / Yorum
- **Kullanıcı, Omurga tarafında Kabinet3 fiber'ini `tg1/0/5`'ten farklı bir porta (muhtemelen `tg1/0/9`) taşımış** — kullanıcı bunu SSH oturumunda net teyit edemedi ("emin değilim" dedi), ama trafik hacmi/deseni ve zamanlama `tg1/0/9` olduğunu güçlü şekilde destekliyor. **Kesin teyit için fiziksel port etiketleri kontrol edilmeli.**
- `tg1/0/9` config'te önceden trunk (VLAN 1-4) olarak hazır bulunuyordu — bu yüzden yazılım tarafında ek bir config değişikliği gerekmedi (ne Kabinet3 ne Omurga tarafında).
- **Kritik bulgu: port/SFP değişikliği sorunu ÇÖZMEDİ, sadece taşıdı.** Kabinet3 kendi tarafında (port 26) 23 dakika temiz görünse de, Omurga tarafında yeni kullanılan port (tg1/0/9) aynı kronik flap desenini gösteriyor. Bu, sorunun tek bir SFP veya tek bir Omurga portuna özgü olmadığını, **daha genel bir chassis 1 sorunu** (veya her iki SFP değişiminde de kullanılan ortak bir fiber/patch kablosu) olabileceğini güçlendiriyor.

## Rollback / Sonraki Adım
Bu bir "geri alınacak" config değişikliği değil, fiziksel bir müdahale — rollback planı yok. **Önerilen sonraki adım:** Kullanıcının hangi Omurga portuna fiber'i taktığını kesin olarak teyit etmesi (fiziksel etiket/port kontrolü), ve mümkünse farklı bir fiber/patch kablosu ile test edilmesi (kabloyu da değiştirerek SFP+port+kablo üçlüsünden hangisinin ortak faktör olduğu netleştirilebilir).

## Onay
- **Otomatik mi onaylandı, manuel mi:** Fiziksel müdahale kullanıcı tarafından bağımsız yapıldı (AI ajanı onayı gerektirmedi, çünkü ajan bu işlemi yapmadı). Ajanın yaptığı tek şey salt-okuma doğrulama.
- **Onaylayan:** Kullanıcı (fiziksel işlemi kendisi yaptı)
