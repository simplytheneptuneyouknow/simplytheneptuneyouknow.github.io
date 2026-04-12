---
title: "Hack The Box: Sendai - Write-up"
date: 2026-04-12 14:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, adcs, esc1, esc4, certipy, bloodhound, gmsa, sqlconfig, windows]
---

# ⛩️ Hack The Box: Sendai - İleri Seviye AD CS ve Sertifika Suistimali

**İşletim Sistemi:** Windows  
**Hedef:** SMB Anonim Erişim, GMSA Parola Okuma, AD CS ESC4 -> ESC1 Dönüşümü ve Domain Admin Erişimi.

---

## 1. Keşif ve İlk Bilgi Toplama (Recon)

Operasyona hedef sistemin üzerindeki servisleri ve SMB paylaşımlarını analiz ederek başlıyoruz.

**Adım 1:** Nmap taramasıyla sistemin bir Domain Controller (DC) olduğunu ve klasik AD portlarının açık olduğunu doğruluyoruz.
![Nmap Taraması](/assets/img/sendai/WhatsApp%20Image%202026-04-07%20at%2019.13.08.jpeg)

**Adım 2:** NetExec (nxc) ile `Guest` kullanıcısını kullanarak RID Brute forcing saldırısı gerçekleştiriyor ve domaindeki kullanıcı listesini döküyoruz.
![RID Brute](/assets/img/sendai/rid%20brute.jpeg)

**Adım 3:** Paylaşımları listelediğimizde `config` ve `transfer` dizinlerine okuma erişimimiz olduğunu görüyoruz.
![SMB Paylaşımları](/assets/img/sendai/guest%20sharelar.jpeg)

---

## 2. Bilgi Sızıntısı: SQL Config ve Kimlik Bilgileri

Sistem içerisindeki paylaşımlarda yaptığımız araştırmada kritik konfigürasyon verilerine ulaşıyoruz.

**Adım 1:** `config` paylaşımı altındaki `.sqlconfig` dosyasını çekiyoruz.
![SQL Config](/assets/img/sendai/smb%20ile%20sqlconfig.jpeg)

**Adım 2:** Dosya içeriğini okuduğumuzda `sqlsvc` kullanıcısına ait açık metin parolayı (`SurenessBlob85`) yakalıyoruz.
![SQL Parolası](/assets/img/sendai/sqlconfig%20dosya%20içeriği.jpeg)

---

## 3. Kan Analizi ve Yetki Yolu (BloodHound)

Tespit ettiğimiz `Thomas Powell` kullanıcısı üzerinden BloodHound analizi yapıyoruz.

**Adım 1:** BloodHound üzerinde `Thomas`'ın yetkilerini incelediğimizde, kendisinin `ADMSVC` grubu üzerinden bir GMSA (Group Managed Service Account) parolasını okuma hakkı olduğunu görüyoruz.
![BloodHound Analizi](/assets/img/sendai/bloodhound%20thomas.jpeg)

**Adım 2 (Kritik Hamle):** `impacket-changepassword` kullanarak Thomas'ın parolasını güncelliyoruz.
![Parola Güncelleme](/assets/img/sendai/impacket%20changepassword.jpeg)

---

## 4. GMSA Suistimali ve User Flag

Elde ettiğimiz yetkilerle GMSA parolasını dökerek sistemde ilerliyoruz.

**Adım 1:** `gMSADumper.py` kullanarak `mgtsvc$` hesabının hash değerlerini alıyoruz.
![GMSA Dump](/assets/img/sendai/şifre%20dump.jpeg)

**Adım 2:** Bu hash ile Evil-WinRM üzerinden bağlanarak ilk hedefimiz olan **user.txt** bayrağını elde ediyoruz.
![User Flag](/assets/img/sendai/user%20flag.jpeg)

---

## 5. AD CS Suistimali: ESC4'ten ESC1'e

`Clifford Davey` kullanıcısının kimlik bilgilerini sistemdeki servisleri tarayarak buluyoruz. Clifford'ın AD CS şablonları üzerinde tehlikeli izinleri olduğunu fark ediyoruz.

**Adım 1:** `Certipy` ile ESC4 zafiyetine sahip şablonu tespit ediyoruz.
![ESC4 Keşfi](/assets/img/sendai/esc4%20göründü.jpeg)

**Adım 2:** `SendaiComputer` şablonunu manipüle ederek onu ESC1 (SAN manipülasyonuna açık) haline getiriyoruz.
![Sömürüldü](/assets/img/sendai/sömürüldü.jpeg)

**Adım 3:** Artık savunmasız olan şablonu kullanarak `Administrator` adına bir sertifika (`.pfx`) talep ediyoruz.
![Sertifika Talebi](/assets/img/sendai/adminpfx.jpeg)

---

## 6. Final: Domain Admin ve Root Flag

**Adım 1:** `certipy auth` ile Administrator kullanıcısının NTLM hash'ini alıyoruz.
![Admin Hash](/assets/img/sendai/admin%20hash.jpeg)

**Adım 2:** Evil-WinRM üzerinden Pass-the-Hash saldırısı ile sisteme Administrator olarak dalıyor ve **root.txt** bayrağını elde ediyoruz.
![Root Flag](/assets/img/sendai/admin%20flag.jpeg)

---
