---
title: "Hack The Box: Puppy - Makine Çözümü (Write-up)"
date: 2026-04-11 12:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, dpapi, pentest, windows, medium]
---

# 🐾 Hack The Box: Puppy - Tam Kapsamlı Sızma Testi Raporu

**Zorluk:** Medium / Hard 
**İşletim Sistemi:** Windows
**Hedef:** Active Directory ACL Suistimali (GenericWrite & GenericAll), DPAPI Şifre Çözme ve Yanal Hareket.

---

## 1. İlk Sızıntı ve Paylaşım Keşfi (GenericWrite Suistimali)

Operasyon, elimizdeki `levi.james` kullanıcısına ait kimlik bilgileriyle (`KingofAkron2025!`) başladı. 

**Adım 1:** NetExec ile SMB paylaşımlarını kontrol ettiğimizde, Levi'nin yalnızca varsayılan okuma yetkilerine sahip olduğunu ve özel bir paylaşıma erişimi olmadığını gördük.
![Paylaşımlar Boş](/assets/img/puppy/sharelar%20boş.jpeg)

**Adım 2:** BloodHound analizinde, Levi'nin üyesi olduğu `HR` grubunun, **`DEVELOPERS`** grubu üzerinde **GenericWrite** yetkisine sahip olduğu ortaya çıktı. Bu, kendimizi o gruba ekleyebileceğimiz anlamına geliyordu.
![Bloodhound Analizi](/assets/img/puppy/levi%20bloodhound%20devepoler.jpeg)

**Adım 3:** Kendimizi gruba ekledikten sonra paylaşımları tekrar taradık. Bingo! Artık **`DEV`** (DEV-SHARE for PUPPY-DEVS) paylaşımına `READ` yetkisiyle erişebiliyorduk.
![Genişletilmiş Paylaşımlar](/assets/img/puppy/developera%20eklendikten%20sonra%20sharelar.jpeg)

**Adım 4:** `smbclient` ile paylaşıma bağlanıp, `Projects` klasörünün içindeki `recovery.kdbx` KeePass veritabanını Kali makinemize çektik.
![KeePass İndirme](/assets/img/puppy/recovery%20indirildi.jpeg)

---

## 2. Kasa Kırımı ve Parola Çıkarımı (Credential Harvesting)

İndirdiğimiz veritabanını açmaya çalıştığımızda bizden bir ana parola istedi.
![Kasa Şifre Ekranı](/assets/img/puppy/şifre%20lazım.jpeg)

**Adım 1:** Parolayı bulmak için KeePass dosyasının hash'ini elde edip John the Ripper ve `rockyou.txt` ile kırmaya koyulduk. Kasa parolası çok geçmeden **`liverpool`** olarak kırıldı.
![John The Ripper](/assets/img/puppy/johntheripper.jpeg)

**Adım 2:** Kasayı KeePassXC grafik arayüzüyle açtık ve içeride Jamie, Adam, Antony, Steve ve Samuel gibi kullanıcılara ait parolaların olduğunu gördük.
![KeePass İçi](/assets/img/puppy/keepas%20uygulama%20içi.jpeg)

**Adım 3:** İşlemleri otomatize etmek için kasayı bir CSV dosyasına (`output.csv`) döktük. Buradan şifreleri temiz bir şekilde alıp `passwords.txt` listemize ekledik.
![CSV Çıktısı](/assets/img/puppy/output%20svc.jpeg)

---

## 3. Parola Püskürtme ve Yeni Hedef (Password Spraying)

Elde edilen parolaları Domain'deki kullanıcılara karşı test etme vakti gelmişti. NetExec kullanarak `users.txt` ve `passwords.txt` listeleriyle bir "Password Spray" saldırısı başlattık. Sistemden yeşil onay geldi: **`ant.edwards`** kullanıcısının parolasının **`Antman2025!`** olduğunu doğruladık.
![Password Spraying](/assets/img/puppy/ant%20edwards%20şifre%20bulundu.jpg)

---

## 4. Active Directory Sömürüsü: GenericAll ve Hesap Aktifleştirme

`ant.edwards` ile BloodHound'a tekrar başvurduk. Ant'in üyesi olduğu `SENIOR DEVS` grubunun, **`adam.silver`** hesabı üzerinde **GenericAll** (Tam Yetki) hakkına sahip olduğunu gördük.
![GenericAll Yetkisi](/assets/img/puppy/ant%20edwards%20genericall%20var.jpeg)

**Adım 1 (Parola Sıfırlama):** `net rpc` aracı ile bu yetkiyi kullanıp Adam Silver'ın parolasını `Yavuz123!` olarak değiştirdik.
![Parola Değişimi](/assets/img/puppy/password%20değiştim.jpeg)

**Adım 2 (Engeli Görme):** NetExec ile test ettiğimizde, şifre doğru olmasına rağmen hesabın `STATUS_ACCOUNT_DISABLED` olduğunu gördük.
![Hesap Disabled](/assets/img/puppy/account%20disabled.jpeg)

**Adım 3 (Hesabı Açma):** `bloodyAD` aracıyla Adam'ın `userAccountControl` bayrağına müdahale edip hesabı aktif hale getirdik.
![Hesap Aktifleştirme](/assets/img/puppy/acoount%20disabled%20kaldırıldı.jpeg)

**Adım 4 (Doğrulama ve Giriş):** NetExec ile tekrar denediğimizde başarılı girişi gördük. 
![Giriş Başarılı](/assets/img/puppy/kaldırılma%20kanıt.jpeg)

BloodHound raporunda Adam'ın **`Remote Management Users`** grubunda olduğunu bildiğimiz için Evil-WinRM kapılarının bize açık olduğunu teyit ettik.
![AD Group Info](/assets/img/puppy/adam%20silver%20grup%20bloodhound.jpeg)

Evil-WinRM ile sisteme bağlandık ve doğrudan `C:\Users\adam.silver\Desktop` dizinine giderek **user.txt** bayrağını aldık!
![User Flag](/assets/img/puppy/user%20flag.jpeg)

---

## 5. Yedek Dosyalarından Gelen Sürpriz

Adam Silver olarak sistemde gezinirken `C:\backups` dizini dikkatimizi çekti. İçinde `site-backup-2024-12-30.zip` adında devasa bir yedek dosyası vardı.
![Yedek Dosyası](/assets/img/puppy/site%20backup%20görüldü.jpeg)

**Adım 1:** Bu dosyayı Evil-WinRM'in `download` komutuyla kendi makinemize indirdik.
![Dosya İndirme](/assets/img/puppy/indirildi.jpeg)

**Adım 2:** Zip dosyasını açıp içindeki konfigürasyon dosyalarını okuduk. `nms-auth-config.xml.bak` dosyasının içinde **`steph.cooper`** kullanıcısının LDAP bind parolası olan **`ChefSteph2025!`** kabak gibi karşımızdaydı.
![XML Analizi](/assets/img/puppy/okundu%20ve%20password%20geldi.jpeg)

---

## 6. Yetki Yükseltme: DPAPI Hazinesi ve Root

`steph.cooper` kimlik bilgileriyle Evil-WinRM oturumu açtık ve sisteme yüklediğimiz `winPEAS` aracını çalıştırdık. Tarama sonucunda DPAPI Master Key ve Credential dosyalarının yollarını tespit ettik.
![WinPEAS Çıktısı](/assets/img/puppy/winpeas%20indirildi%20çalıştırıldı%20sonuc.jpg)

**Adım 1 (Korumaları Aşma):** DPAPI dosyaları "Gizli" ve "Sistem" korumalı olduğu için doğrudan inmiyordu. Önce dosyaları masaüstüne kopyaladık.
![Dosya Kopyalama](/assets/img/puppy/masaüstüne%20kopyaladım%20indirebilmek%20için.jpeg)

Ardından `attrib -h -s` komutuyla gizliliklerini kaldırıp Kali'ye indirdik.
![Öznitelik Sıfırlama](/assets/img/puppy/gizliliği%20kaldırıp%20indirdim.jpeg)

**Adım 2 (Master Key Çözümü):** İndirdiğimiz `masterkey.bak` dosyasını, Steph'in SID numarası ve bulduğumuz `ChefSteph2025!` parolası ile `impacket-dpapi` kullanarak kırdık.
![MasterKey Çözümü](/assets/img/puppy/masterkey%20kırıldı.jpeg)

**Adım 3 (Kasa Patlatma):** Elde ettiğimiz açık anahtarla `realcred.bak` kasa dosyasını deşifre ettik. Karşımıza çıkan sonuç muazzamdı; hedefin Domain Admin hesabı olan `steph.cooper_adm`'in parolası olan **`FivethChipOnItsWay2025!`** elimizdeydi!
![Admin Parolası](/assets/img/puppy/impacket%20ile%20crendetial%20alındı.jpeg)

**Final:** Tam yetkili Admin parolasını aldıktan sonra son kez Evil-WinRM ile sisteme bağlandık. Administrator masaüstüne gidip **root.txt** bayrağını dalgalandırdık. Sistem tamamen ele geçirilmiştir!
![Root Flag](/assets/img/puppy/VE%20ADMİN%20FLAG.jpeg)
---
