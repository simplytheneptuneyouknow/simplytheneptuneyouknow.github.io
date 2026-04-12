---
title: "Hack The Box: Administrator - Makine Çözümü (Write-up)"
date: 2026-04-12 15:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, ftp, psafe3, hashcat, targeted-kerberoasting, dcsync, windows]
---

# 👑 Hack The Box: Administrator - Tam Kapsamlı Sızma Testi Raporu

**İşletim Sistemi:** Windows
**Hedef:** AD Yetki Zinciri (GenericAll & ForceChangePassword), Password Safe (psafe3) Kırımı, Targeted Kerberoasting ve DCSync ile Domain'i Ele Geçirme.

---

## 1. Keşif ve İlk Sızıntı (Recon)

Hedef sisteme yönelik gerçekleştirdiğimiz Nmap taramasında klasik Active Directory portlarının (DNS, Kerberos, LDAP, SMB) ve FTP servisinin (21) açık olduğunu tespit ediyoruz.
![Nmap Taraması](/assets/img/administrator/nmap.jpeg)

Operasyona elimizdeki `olivia` kullanıcısının kimlik bilgileriyle (`ichliebedich`) başlıyoruz.

---

## 2. Yetki Yükseltme Zinciri: Olivia -> Michael -> Benjamin

BloodHound ile Active Directory ortamının haritasını çıkardığımızda, karşımıza sömürülmeyi bekleyen harika bir yetki zinciri çıkıyor!
![Yetki Zinciri](/assets/img/administrator/oliviabloodhound.jpeg)

**Adım 1:** `net rpc` aracı ile Michael'ın parolasını zorla `Yavuz123` olarak değiştiriyoruz.
![Michael Pass Değişimi](/assets/img/administrator/netrpcpassword.jpeg)

*NetExec kontrolü:*
![Michael Pass Kontrol](/assets/img/administrator/michaelpasskontrolü.jpeg)

**Adım 2:** Benjamin'in parolasını da `Yavuz123` olarak güncelliyoruz.
![Benjamin Pass Değişimi](/assets/img/administrator/benjaminpassdeğişim.jpeg)

---

## 3. FTP Erişimi ve Ganimetin Bulunması

**Adım 1:** FTP üzerinden **`Backup.psafe3`** dosyasını indiriyoruz.
![FTP İndirme](/assets/img/administrator/ftpdosyacekme.jpeg)

**Adım 2 (Kasa Kırımı):** Hashcat ile ana parolayı kırıyoruz: **`tequieromucho`**
![Hashcat Kırma](/assets/img/administrator/backupkırma.jpeg)
![Şifre Bulundu](/assets/img/administrator/backupşifrebulundu.jpeg)

**Adım 3 (Kasa İçeriği):** `pwsafe` aracıyla Emily'nin parolasını alıyoruz.
![Pwsafe Çalıştırma](/assets/img/administrator/pwsafeuygulaması.jpeg)
![Kasa İçi](/assets/img/administrator/uygulamaiçi.jpeg)

**Adım 4 (User Bayrağı):** Emily ile bağlanıp bayrağı alıyoruz.
![Emily Parola Kontrolü](/assets/img/administrator/emilypasswordkontrolü.jpeg)
![User Flag](/assets/img/administrator/userflag.jpeg)

---

## 4. Targeted Kerberoasting: Emily -> Ethan

Emily'nin **`ethan`** üzerindeki yetkisini keşfediyoruz.
![Emily Bloodhound](/assets/img/administrator/emilybloodhound.jpeg)

**Adım 1:** Ethan için Kerberos hash'ini çekiyoruz.
![Targeted Kerberoast](/assets/img/administrator/ethanhash.jpeg)

**Adım 2:** Hash'i kırıyoruz: **`limpbizkit`**
![Ethan Hash Kırma](/assets/img/administrator/ethanhashkirma.jpeg)
![Ethan Şifre Kırıldı](/assets/img/administrator/ethanhashkırıldı.jpeg)

---

## 5. Domain'in Düşüşü: DCSync ve Root

Ethan'ın Domain üzerindeki `GetChangesAll` hakkını suistimal ediyoruz.
![Ethan Yetkileri](/assets/img/administrator/ethanbloodhoundyetkileri.jpeg)

**Adım 1 (DCSync):** Tüm NTLM hash'lerini sızdırıyoruz.
![Secretsdump](/assets/img/administrator/Userhashleri.jpeg)

**Adım 2 (Root Bayrağı):** Administrator olarak bağlanıp makineyi bitiriyoruz.
![Root Flag](/assets/img/administrator/rootflag.jpeg)
