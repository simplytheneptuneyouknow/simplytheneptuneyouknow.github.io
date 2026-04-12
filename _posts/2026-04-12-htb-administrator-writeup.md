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

BloodHound ile Active Directory ortamının haritasını çıkardığımızda, karşımıza sömürülmeyi bekleyen harika bir yetki zinciri çıkıyor! BloodHound grafiğine göre; **`OLIVIA`** kullanıcısı **`MICHAEL`** üzerinde `GenericAll` (Tam Yetki) hakkına, **`MICHAEL`** ise **`BENJAMIN`** üzerinde `ForceChangePassword` (Parola Sıfırlama) hakkına sahip.
![Yetki Zinciri](/assets/img/administrator/olivia%20bloodhound.jpeg)

**Adım 1 (Olivia -> Michael):** `GenericAll` yetkimizi kullanarak, `net rpc` aracı ile Michael'ın parolasını zorla `Yavuz123` olarak değiştiriyoruz.
![Michael Pass Değişimi](/assets/img/administrator/net%20rpc%20password.jpeg)

*NetExec ile kontrol ettiğimizde Michael'ın yeni parolasıyla sisteme başarıyla giriş yaptığımızı doğruluyoruz:*
![Michael Pass Kontrol](/assets/img/administrator/michael%20pass%20kontrolü.jpeg)

**Adım 2 (Michael -> Benjamin):** Şimdi Michael'ın kimliğine bürünerek onun `ForceChangePassword` yetkisini suistimal ediyoruz ve Benjamin'in parolasını da `Yavuz123` olarak güncelliyoruz.
![Benjamin Pass Değişimi](/assets/img/administrator/benjamin%20pass%20değişim.jpeg)

---

## 3. FTP Erişimi ve Kasanın Bulunması

Parolasını ele geçirdiğimiz `benjamin` kullanıcısıyla hedef sistemdeki FTP servisine bağlanıyoruz.

**Adım 1:** FTP dizinlerini listelediğimizde karşımıza **`Backup.psafe3`** adında bir parola kasası dosyası çıkıyor ve bunu kendi makinemize indiriyoruz.
![FTP İndirme](/assets/img/administrator/ftp%20dosya%20çekme%20.jpeg)

**Adım 2 (Kasa Kırımı):** Bu kasayı açabilmek için ana parolasına ihtiyacımız var. Hashcat (`-m 5200` modu) ve `rockyou.txt` sözlüğünü kullanarak kasaya kaba kuvvet saldırısı başlatıyoruz.
![Hashcat Kırma](/assets/img/administrator/backup%20kırma.jpeg)

Kısa süre içinde Hashcat bize ana parolayı veriyor: **`tequieromucho`**!
![Şifre Bulundu](/assets/img/administrator/backup%20şifre%20bulundu.jpeg)

**Adım 3 (Kasa İçeriği):** Bulduğumuz parolayla kasayı `pwsafe` aracı üzerinden açıyoruz.
![Pwsafe Çalıştırma](/assets/img/administrator/pwsafe%20uygulaması.jpeg)

Kasanın grafik arayüzünde Alexander, Emily ve Emma'nın kayıtlı olduğunu görüyoruz. Buradan **`emily`** kullanıcısına ait parolayı alıyoruz.
![Kasa İçi](/assets/img/administrator/uygulama%20içi%20.jpeg)

NetExec ile test ettiğimizde Emily'nin parolasının geçerli olduğunu doğruluyoruz.
![Emily Parola Kontrolü](/assets/img/administrator/emily%20password%20kontrolü.jpeg)

**Adım 4 (User Bayrağı):** Emily kimlik bilgileriyle Evil-WinRM üzerinden sisteme bağlanıp **user.txt** bayrağını cebe indiriyoruz!
![User Flag](/assets/img/administrator/user%20flag.jpeg)

---

## 4. Targeted Kerberoasting: Emily -> Ethan

İçerideyiz. Emily ile BloodHound haritasına tekrar baktığımızda, Emily'nin **`ethan`** kullanıcısı üzerinde `GenericWrite` yetkisi olduğunu keşfediyoruz.
![Emily Bloodhound](/assets/img/administrator/emily%20bloodhound.jpeg)

**Adım 1:** `GenericWrite` yetkisini kullanarak `targetedKerberoast.py` betiği ile Ethan'a geçici bir SPN (Service Principal Name) atıyor ve Kerberos hash'ini çekiyoruz!
![Targeted Kerberoast](/assets/img/administrator/ethan%20hash.jpeg)

**Adım 2:** Elde ettiğimiz bu hash'i Hashcat (`-m 13100` modu) ile kırıyoruz.
![Ethan Hash Kırma](/assets/img/administrator/ethan%20hash%20kırma.jpeg)

Ethan'ın parolası açığa çıkıyor: **`limpbizkit`**
![Ethan Şifre Kırıldı](/assets/img/administrator/ethan%20hash%20kırıldı.jpeg)

---

## 5. Domain'in Düşüşü: DCSync ve Root

Neden bu kadar uğraşıp Ethan hesabını aldık? Çünkü BloodHound'a göre `ethan` hesabı Domain üzerinde **`GetChangesAll`** hakkına sahip! Bu, Domain'deki herkesin hash'ini çekebileceğimiz anlamına geliyor.
![Ethan Yetkileri](/assets/img/administrator/ethan%20bloodhound%20yetkileri.jpeg)

**Adım 1 (DCSync):** Impacket'in `secretsdump.py` aracını kullanarak Domain Controller'dan **Administrator** dahi olmak üzere herkesin NTLM hash'lerini (NTDS.dit) başarıyla sızdırıyoruz.
![Secretsdump](/assets/img/administrator/User%20hashleri.jpeg)

**Adım 2 (Root Bayrağı):** Çaldığımız Administrator hash'i ile Pass-the-Hash (PtH) saldırısı gerçekleştirerek Evil-WinRM üzerinden en yetkili oturumumuzu elde ediyor ve **root.txt** bayrağını okuyarak makineyi tamamen ele geçiriyoruz! Sistem "Pwned"!
![Root Flag](/assets/img/administrator/root%20flag.jpeg)
---
