# Değişiklik Özeti — Kabinet1-2

## 🔖 TL;DR
Kabinet1-2'nin saati NTP olmadan zaten doğruydu (CASE-011 gizemi); yine de diğer switch'lerle tutarlılık için `ntp server 162.159.200.1` eklendi, kalıcı hale getirildi. Acil bir arıza düzeltmesi değil, önleyici işlem.

- **Tarih/Saat:** 2026-08-13 ~13:11 TRT
- **Cihaz:** Kabinet1-2 (10.64.0.22)
- **İlgili oturum log'u:** `logs/2026-08-13/Kabinet1-2_131000.md`
- **Gerekçe:** Kullanıcı talebi — diğer switch'lerle (Kabinet3, Kabinet1-1) tutarlılık için NTP konfigürasyonu. Not: bu switch'in saati NTP olmadan da zaten doğruydu (CASE-011), bu önleyici bir işlemdir, acil bir arıza düzeltmesi değil.
- **Etki alanı:** Sadece global NTP client konfigürasyonu. Başka hiçbir ayar değişmedi.

## Önce (Before)
```
clock timezone Istanbul
lldp run
```
(NTP satırı yok)

## Sonra (After)
```
clock timezone Istanbul
ntp server 162.159.200.1
lldp run
```

## Fark (Diff)
```diff
 clock timezone Istanbul
+ntp server 162.159.200.1
 lldp run
```

## Doğrulama
- `show clock`: değişiklik öncesi `13:10:17 EEST Thu Aug 13 2026`, sonrası `13:11:13 EEST Thu Aug 13 2026` — ikisi de zaten doğruydu (tutarlı).
- `show startup-config`: `ntp server 162.159.200.1` satırı mevcut — `wr` ile kalıcı hale getirildi.

## Geri Alma Planı (Rollback)
```
configure terminal
no ntp server 162.159.200.1
end
wr
```
(`no` sözdizimi test edilmedi, Kabinet3'teki gibi doğrulanmadan uygulanmamalı.)

## Onay
- **Otomatik mi onaylandı, manuel mi:** Manuel — kullanıcı bu konuşmada açıkça talep etti ("Kabinet1-2 saat düzenlemesini tekrar yapmayı dene"), daha önce aynı işlem için verilmiş genel onayın devamı niteliğinde.
- **Onaylayan:** Kullanıcı
