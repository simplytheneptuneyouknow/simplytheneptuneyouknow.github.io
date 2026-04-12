---
title: "Hack The Box: Baby2 - Write-up"
date: 2026-04-13 10:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, gpo-abuse, bloodhound, powerview, acl, windows, hard]
---

# 🍼 Hack The Box: Baby2 Write Up

**İşletim Sistemi:** Windows
**Hedef:** GPO (Group Policy Object) Suistimali, ACL Manipülasyonu ve Domain Controller Tam Denetimi.

---

## 1. Keşif ve Veri Toplama (Recon)

Operasyona `Amelia Griffiths` kullanıcısı üzerinden elde ettiğimiz verileri BloodHound ile analiz ederek başlıyoruz.

**Adım 1:** `bloodhound-python` aracını kullanarak AD ortamının tüm haritasını çıkartıyoruz.
![BloodHound Veri Toplama](/assets/img/baby2/bloodhound%20ile%20dosya%20oluşturma.jpeg)

**Adım 2:** BloodHound analizinde `Amelia`'nın, `LEGACY` grubu üzerinden `GPOADM` ve `GPO-MANAGEMENT` objeleri üzerinde kritik yetkilere (WriteDacl, WriteOwner) sahip olduğunu keşfediyoruz.
![BloodHound Analizi](/assets/img/baby2/amelia%20bloodhound%20ekranı.jpeg)

---

## 2. İlk Erişim ve ACL Manipülasyonu

Tespit ettiğimiz bu yetki yollarını kullanarak sistemde daha ileriye gitmeyi hedefliyoruz.

**Adım 1:** `Amelia` olarak sisteme ilk adımımızı atıyoruz.
![Initial Foothold](/assets/img/baby2/amelia%20ilk%20shell%20ekranı.jpeg)

**Adım 2:** `PowerView.ps1` scriptini sisteme aktararak ACL manipülasyonuna başlıyoruz. `Add-DomainObjectAcl` komutu ile `gpoadm` kullanıcısı üzerinde tam denetim (All Rights) elde ediyoruz.
![PowerView ACL Abuse](/assets/img/baby2/amelia%20shelinde%20gpoadmin%20powerview%20eklenip%20kod%20çalıştırılıyor.jpeg)

---

## 3. GPO Suistimali ve Reverse Shell (Lateral Movement)

BloodHound verilerine göre `gpoadm` kullanıcısı Domain Controller ve Domain Policy üzerinde `GenericAll` yetkisine sahip. Bu, tüm domaini ele geçirmek için altın anahtarımız.

**Adım 1:** SYSVOL paylaşımı altındaki GPO scriptlerini inceliyoruz. `login.vbs` dosyasının her dakika çalıştığını fark ediyoruz.
**Adım 2:** Kendi makinemizde bir Python HTTP sunucusu açarak hazırladığımız `script.ps1` dosyasını paylaşıma sunuyoruz.
![Payload Hazırlığı](/assets/img/baby2/İçeriden%20reverse%20shell%20alabilmek%20ardından%20shellimizi%20güçlendirmek%20için%20yazılıyor%20link%20kısmı%20ise%20kendi%20makinemizde%20python%20sunucusu%20açarak%20dosyayı%20kendimizden%20makineye%20atıyoruz.jpeg)

**Adım 3:** `login.vbs` dosyasını indirip, içine bizim zararlı scriptimizi çalıştıracak kodu ekliyor ve `put` komutu ile tekrar makineye yüklüyoruz. 
![GPO Script Manipülasyonu](/assets/img/baby2/değiştirilip%20içerisine%20reverse%20shell%20yazılan%20login.vbs%20put%20ile%20makineye%20yükleniyor%20ardından%20bu%20shellin%20aktif%20olması%20için%201%20dakika%20geçmesi%20gerekiyor%20çünkü%20login.vbs%20dakikada%201%20çalışıyor.jpeg)

---

## 4. Yetki Yükseltme ve Domain Admin

GPO tetiklendiğinde hazırladığımız script çalışıyor ve bize `gpoadm` yetkilerinde bir shell veriyor.

**Adım 1:** Scriptin başarıyla çekildiğini HTTP sunucumuzdan doğruluyoruz.
![Payload Delivery](/assets/img/baby2/evet%20script%20shellimizde%20çekildiğine%20göre%20shellimizi%20elde%20etitk.jpeg)

**Adım 2:** `gpoadm` kullanıcısının Domain Controller üzerindeki tam yetkisini kullanarak administrator parolasını resetliyoruz veya doğrudan sistem hashlerini elde ediyoruz.
![GPOADM Yetkileri](/assets/img/baby2/gpoadm%20inin%20ise%20domain%20controller%20ve%20domain%20policy%20üzerinde%20GenericAll%20yetkisini%20görüyoruz%20bu%20yetkiyi%20kendimize%20almak%20için.jpeg)

---

## 5. Final: Root Flag

Elde ettiğimiz Administrator yetkileriyle devam ediyoruz.

**Adım 1:** `evil-winrm` ile Administrator olarak bağlanıyor ve son bayrağımızı okuyoruz.
![Root Flag](/assets/img/baby2/değiştirilen%20gpoadmin%20şifresi%20ile%20evil%20winrm%20üzerinden%20giriş%20yapılıp%20admin%20flagi%20alınıyor.jpeg)

> **Makine Pwned!** > GPO Abuse ve ACL manipulasyonu başarıyla tamamlandı.
