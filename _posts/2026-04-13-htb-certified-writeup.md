---
title: "Hack The Box: Certified - Write-up"
date: 2026-04-12 23:30:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, adcs, certipy, bloodhound, acl-abuse, shadow-credentials, windows]
---

# 📜 Hack The Box: Certified Write Up

**İşletim Sistemi:** Windows  
**Hedef:** AD CS (Active Directory Certificate Services) Suistimali, ACL Manipülasyonu, Shadow Credentials Saldırısı ve Domain Admin Erişimi.

---

## 1. Keşif ve BloodHound Analizi

Ağdaki ilişkileri ve yetki yollarını görselleştirmek için BloodHound ile başlıyoruz.

**Adım 1:** `bloodhound-python` aracını kullanarak `judith.mader` kullanıcısı ile domain verilerini topluyoruz.
![BloodHound Veri Toplama](/assets/img/certified/bloodhound%20dosya%20oluşturulması.jpeg)

**Adım 2:** BloodHound analizinde, elimizdeki kullanıcının ve bazı servis hesaplarının kritik gruplar üzerinde sahiplik (Owner) ve yazma hakları olduğunu tespit ediyoruz.
![BloodHound Analizi 1](/assets/img/certified/bloodhound%20iç%201.jpeg)
![BloodHound Analizi 2](/assets/img/certified/bloodhound%20iç%202.jpeg)
![Yetki Zinciri](/assets/img/certified/bloodhound%20iç%203.jpeg)

---

## 2. ACL Manipülasyonu ve İlk Yetki Artırımı

BloodHound'un gösterdiği yolu takip ederek, grup üyeliklerini manipüle ediyoruz.

**Adım 1:** `impacket-owneredit` ile hedef nesnenin sahipliğini üzerimize alıyoruz.
![Owner Edit](/assets/img/certified/impacketowneredit.jpeg)

**Adım 2:** `net rpc` komutunu kullanarak kendimizi (veya hedeflediğimiz hesabı) **Management** grubuna dahil ediyoruz.
![Net RPC Grup Ekleme](/assets/img/certified/net%20rpc%20komutu.jpeg)

**Adım 3:** Ardından `impacket-dacledit` ile gerekli izinleri (WriteMembers vb.) tanımlıyoruz.
![DACL Edit](/assets/img/certified/impacketdacledit.jpeg)

---

## 3. Certipy ile Shadow Credentials ve Hash Dump

Sistemdeki en kritik aşama, servis hesaplarının yetkilerini kullanarak üst düzey hash'leri ele geçirmek.

**Adım 1:** `Certipy` kullanarak `management_svc` hesabının NT hash'ini Shadow Credentials saldırısı veya benzeri bir sertifika suistimali ile elde ediyoruz.
![Management SVC Hash](/assets/img/certified/managament%20svcnin%20hashi%20certipy%20ile.jpeg)

**Adım 2:** Elde ettiğimiz bu yetkili hesapla sisteme giriş yaparak ilk hedefimiz olan **user.txt** bayrağını okuyoruz.
![User Flag](/assets/img/certified/user%20flag.jpeg)

---

## 4. Sertifika Manipülasyonu (AD CS ESC Saldırısı)

Şimdi Domain Admin yetkileri için Sertifika Servisleri'ni (AD CS) hedef alıyoruz.

**Adım 1:** `ca_operator` kullanıcısını Certipy ile güncelleyerek Kerberos sunucusuna kendimizi `Administrator` gibi tanıtıyoruz.
![UPN Güncelleme](/assets/img/certified/ca%20operatoru%20kerberos%20sunucusuna%20admin%20gibi%20gösterdik%20certipy%20ile.jpeg)

**Adım 2:** Manipüle edilmiş bu kimlikle Administrator adına bir `.pfx` sertifikası alıyoruz.
![PFX Sertifikası](/assets/img/certified/ca%20operaor%20adına%20administartor.pfx%20aldık.jpeg)

**Adım 3:** İşlem tamamlandıktan sonra çakışmaları önlemek için hesabı eski haline döndürüyoruz.
![Temizlik](/assets/img/certified/ca%20operatoru%20eski%20haline%20getirdik%20yoksa%20çakışma%20oluyor.jpeg)

---

## 5. Final: Domain Admin ve Root Bayrağı

**Adım 1:** Aldığımız `.pfx` sertifikasını `certipy auth` ile kullanarak Administrator kullanıcısının NTLM hash'ini alıyoruz.
![Admin Hash](/assets/img/certified/eldeettigimadministratorileadminhashi.jpeg)

**Adım 2:** Bu hash'in geçerliliğini NetExec (nxc) ile doğruluyoruz (Pwn3d!).
![NetExec Doğrulama](/assets/img/certified/smb%20ile%20kontrol%20edildi.jpeg)

**Adım 3:** Evil-WinRM ile Administrator olarak bağlanıp **root.txt** bayrağını okuyarak makineyi tamamen ele geçiriyoruz.
![Root Flag](/assets/img/certified/ve%20root%20flag.jpeg)
