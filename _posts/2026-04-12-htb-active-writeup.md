---
title: "Hack The Box: Active - Makine Çözümü (Write-up)"
2026-04-12 12:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, smb, gpp, cpassword, kerberoasting, pentest, windows, easy]
---

# 🐾 Hack The Box: Active - Sızma Testi Raporu

**Zorluk:** Easy
**İşletim Sistemi:** Windows
**Hedef:** Active Directory Keşfi, SMB Anonim Erişim Suistimali, GPP (Group Policy Preferences) cpassword Deşifresi ve Kerberoasting ile Yetki Yükseltme.

---

## 1. Keşif ve Bilgi Toplama (Reconnaissance)

Operasyona her zaman olduğu gibi ağdaki açık kapıları ve servisleri tespit ederek başlıyoruz.

**Adım 1: Nmap Taraması**
Hedef sisteme yönelik kapsamlı bir port taraması gerçekleştirdiğimizde, tam teşekküllü bir Active Directory ortamı (Domain Controller) ile karşı karşıya olduğumuzu anlıyoruz. DNS (53), Kerberos (88), RPC (135), LDAP (389, 3268) ve SMB (445) portları açık durumda.
![Nmap Taraması](/assets/img/active/2.jpeg)

*Önemli Not:* Nmap çıktısında domain adının **`active.htb`** olduğunu tespit ettik. Sistemlerin isim çözünürlüğünü düzgün yapabilmesi için bu adresi host dosyamıza ekliyoruz:
![Hosts Dosyası Düzenleme](/assets/img/active/3.jpeg)

---

## 2. Zafiyet Tespiti: SMB Paylaşımları

Active Directory ortamlarında ilk bakılması gereken yerlerden biri SMB paylaşımlarıdır. Bazen yapılandırma hataları nedeniyle yetkisiz kullanıcılara fazla hak verilebilir.

**Adım 1: SMBMap ile İnceleme**
Hedef makinedeki paylaşımları ve izinleri listelemek için `smbmap` aracını çalıştırıyoruz.
![SMBMap Çıktısı](/assets/img/active/4.jpg)

> **Sonuç:** Anonim (kimlik doğrulamasız) erişimle **`Replication`** paylaşımına **`READ ONLY`** (Sadece Okuma) yetkimiz olduğunu görüyoruz. Bu harika bir haber!

**Adım 2: Paylaşıma Bağlanma**
Hemen `smbclient` kullanarak anonim olarak sisteme giriş yapıyor ve klasörleri inceliyoruz.
![SMBClient Girişi](/assets/img/active/1.jpeg)

---

## 3. Sömürü (Exploitation): GPP cpassword Zafiyeti

`Replication` paylaşımı genellikle Domain Controller'ların SYSVOL klasörünün bir parçasıdır ve domain üzerindeki Group Policy (Grup İlkesi) ayarlarını barındırır. 

**Adım 1: Hedef Dosyayı Bulma**
`smbclient` içindeyken `active.htb\Policies` dizinlerinde derinlemesine bir gezintiye çıkıyoruz. Bu dizinlerde eski Windows Server sürümlerinde sıkça karşılaşılan bir zafiyet arıyoruz.
![Policies Dizinleri](/assets/img/active/5.jpg)

**Adım 2: Groups.xml Keşfi**
`MACHINE\Preferences\Groups` dizinine ulaştığımızda altın madenini buluyoruz: **`Groups.xml`**. Bu dosyayı okuduğumuzda, **`active.htb\SVC_TGS`** kullanıcısı için oluşturulmuş bir parola ilkesi görüyoruz. Ancak parola açık metin değil, `cpassword` parametresi altında AES-256 ile şifrelenmiş olarak tutuluyor. (Microsoft bu şifreleme anahtarını yıllar önce yanlışlıkla internette sızdırmıştı!)
![Groups XML](/assets/img/active/6.jpeg)

**Adım 3: Parolayı Kırma**
Elde ettiğimiz o uzun şifreli metni, Kali Linux'ta yerleşik olarak bulunan `gpp-decrypt` aracına veriyoruz.
![GPP Decrypt](/assets/img/active/8.jpeg)

> **Sonuç:** Parola başarıyla çözüldü!
> **Kullanıcı Adı:** `SVC_TGS`
> **Parola:** `GPPstillStandingStrong2k18`

---

## 4. Sistem Erişimi ve User Bayrağı (Foothold)

Elde ettiğimiz geçerli kimlik bilgileriyle artık sıradan bir anonim kullanıcı değil, domain'de yetkili bir kullanıcıyız.

**Adım 1: Kullanıcı Dosyalarına Erişim**
`smbclient` ile hedef sistemin **`Users`** paylaşımına `SVC_TGS` kullanıcısı ile giriş yapıyoruz.
![Users Paylaşımı](/assets/img/active/9.jpeg)

**Adım 2: Bayrağı Alma**
Doğrudan kullanıcının masaüstü (`Desktop`) dizinine giderek ilk hedefimiz olan **`user.txt`** dosyasını kendi makinemize indiriyoruz.
![User Flag İndirme](/assets/img/active/10.jpeg)

---

## 5. Yetki Yükseltme (Privilege Escalation): Kerberoasting

Elde ettiğimiz `SVC_TGS` kullanıcısı, domain içerisinde SPN (Service Principal Name) kaydına sahip diğer hesapları sorgulamak için yeterli yetkiye sahip. Bu durumu klasik bir **Kerberoasting** saldırısıyla suistimal edeceğiz.

**Adım 1: TGS Biletini Çekme (GetUserSPNs)**
Impacket araç setinden `GetUserSPNs.py` betiğini kullanarak domain'deki SPN'leri sorguluyoruz.
![Kerberoasting](/assets/img/active/11.jpeg)

> **Sonuç:** `Administrator` hesabı için bir SPN kaydı bulundu ve bu hesaba ait Kerberos biletinin (TGS) hash'ini başarıyla çektik.

**Adım 2: Hash Kırma (Hashcat)**
Elde ettiğimiz bu hash'i kırmak için Hashcat ve `rockyou.txt` sözlüğünü kullanıyoruz. Hash türü Kerberos 5 TGS-REP olduğu için mod numarasını `13100` olarak belirliyoruz.
![Hashcat](/assets/img/active/12.jpg)

> **Sonuç:** Administrator parolası kırıldı!
> **Parola:** `Ticketmaster1968`

---

## 6. Domain Admin ve Root Bayrağı

Artık Domain Admin yetkilerine sahibiz. Hedef sisteme tam yetkili olarak bağlanıp son bayrağı alma vakti geldi.

**Adım 1: Sisteme Bağlantı (Wmiexec)**
Yine Impacket araç setinden `wmiexec.py` kullanarak Administrator kimlik bilgileriyle Domain Controller üzerinde yetkili bir shell (kabuk) elde ediyoruz.
![Wmiexec Shell](/assets/img/active/winrm.jpeg)

**Adım 2: Root Bayrağı**
Doğrudan `C:\Users\Administrator\Desktop` dizinine giderek **root.txt** dosyasını okuyoruz ve makineyi tamamen ele geçiriyoruz!
![Root Flag](/assets/img/active/root.jpeg)
---
