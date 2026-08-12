# Değişiklik Özeti — Kabinet3
- **Tarih/Saat:** 2026-08-11 ~20:09 TRT
- **Cihaz:** Kabinet3 (10.64.0.28)
- **İlgili oturum log'u:** `logs/2026-08-11/Kabinet3_200500.md`
- **Gerekçe:** `investigation.md` CASE-008 — cihaz saati senkron değildi (NTP yapılandırılmamıştı, `show clock` 1970 epoch gösteriyordu), bu da log zaman damgalarının gerçek olaylarla (CASE-007 flap analizleri, diğer switch logları) karşılaştırılmasını engelliyordu. Kullanıcı bu değişikliğe açıkça onay verdi ve NTP kaynağı olarak dış/genel bir NTP sunucusu seçilmesini istedi.
- **Etki alanı:** Sadece global NTP client konfigürasyonu. VLAN, interface, trunk, spanning-tree gibi trafik/erişimle ilgili hiçbir ayar değişmedi.

## Önce (Before)
```
hostname Kabinet3
!
logging console 6
logging monitor 6
logging trap 6
!
spanning-tree mode rstp
telnet-server enable
ssh-server enable
web-server enable http
username admin privilege 15 password 5 $1$nVoAW3w/$LS3YuOqUe6WvETFKbsIcH0
clock timezone Istanbul
lldp run
!
...
```
(NTP satırı yok)

## Sonra (After)
```
hostname Kabinet3
!
logging console 6
logging monitor 6
logging trap 6
!
spanning-tree mode rstp
telnet-server enable
ssh-server enable
web-server enable http
username admin privilege 15 password 5 $1$nVoAW3w/$LS3YuOqUe6WvETFKbsIcH0
clock timezone Istanbul
ntp server 162.159.200.1
lldp run
!
...
```

## Fark (Diff)
```diff
 clock timezone Istanbul
+ntp server 162.159.200.1
 lldp run
```

## Doğrulama
`show clock` çıktısı değişiklikten ~75 saniye sonra:
- Önce: `07:22:57  EET Fri Jan 02 1970`
- Sonra: `20:13:17  +03 Tue Aug 11 2026` ✅ Doğru gerçek tarih/saat, doğru timezone offset (+03, Istanbul yaz saati)

## Geri Alma Planı (Rollback)
```
configure terminal
no ntp server 162.159.200.1
end
```
(Bu komut test edilmedi — `no` sözdizimi bu OS'te standart olarak bekleniyor ama doğrulanmadı. Gerekirse önce `no ntp server ?` ile teyit edilmeli.) Geri alma sonrası cihaz saati tekrar senkronsuz duruma (1970 epoch) döner; işlevsel bir bağımlılık yoktur, risksiz geri alınabilir.

## Onay
- **Otomatik mi onaylandı, manuel mi:** Manuel — kullanıcı bu konuşmada açıkça onay verdi ("Evet, bu değişikliğe onay veriyorum") ve NTP kaynağı seçimini yaptı ("Dışarıdan genel bir NTP sunucusu").
- **Onaylayan:** Kullanıcı (bu oturumdaki talep sahibi)

## Açık Konu — Kalıcılık
Değişiklik henüz **startup-config'e kaydedilmedi** (sadece running-config'te). Reboot sonrası kaybolur. Kullanıcıya kalıcı hale getirmenin istenip istenmediği ayrıca soruldu (bkz. `investigation.md`).
