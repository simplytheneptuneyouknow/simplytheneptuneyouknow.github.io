---
title: "Hack The Box: Silentium - Write-up"
date: 2026-05-12 12:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [api-exploitation, cve-2025-58434, logic-flaw, flowise, rce, password-reuse, gogs, cve-2025-8110, pentest, linux]
---

# 🐾 Hack The Box: Silentium

**Zorluk:** Medium / Hard
**İşletim Sistemi:** Linux
**Hedef:** VHOST Keşfi, CVE-2025-58434 (API Logic Flaw) ile Account Takeover, Flowise üzerinden RCE, Ortam Değişkeni (ENV) Sızıntısı ile SSH Erişimi ve İç Ağ Pivotlama üzerinden Gogs RCE (CVE-2025-8110) ile Root yetkisi elde etme.

---

## 1. Keşif ve Bilgi Toplama (Reconnaissance)

Operasyona hedef sistemin dış ağ yüzeyini haritalandırarak başlıyoruz.

**Adım 1: Nmap Taraması**
Hedefe yönelik Nmap taramamızda 22 (SSH) ve 80 (HTTP) portlarının açık olduğunu tespit ettik. 80 portuna gelen isteklerin `silentium.htb` adresine yönlendirildiğini (redirect) gördüğümüz için makinenin IP adresini `/etc/hosts` dosyamıza ekliyoruz.
![Nmap Taraması ve Hosts Dosyası Düzenleme]
(/assets/img/silentium/nmap ve etchosts.png)

**Adım 2: VHOST (Sanal Sunucu) Keşfi**
Ana web sayfasında ilerleyecek bir yol bulamayınca, alt alan adlarını bulmak için `gobuster` ile VHOST taraması gerçekleştiriyoruz. Bu tarama sonucunda HTTP 200 yanıtı veren **`staging.silentium.htb`** adresini keşfediyoruz ve bunu da hemen `/etc/hosts` dosyamıza ekliyoruz.
![Gobuster VHOST Keşfi](/assets/img/silentium/gobuster arama.png)
![Staging Hosts Dosyasına Ekleme](/assets/img/silentium/stagingi tekrardan etchosta ekledik.png)

---

## 2. Zafiyet Tespiti: API Üzerinden Hesap Ele Geçirme (Account Takeover)

`staging.silentium.htb` adresi üzerinde çalışan API uç noktalarını incelerken, mimarideki devasa bir güvenlik açığını tespit ediyoruz. Flowise AI uygulamasının kimlik doğrulamasız bir API mantıksal hatası (CVE-2025-58434) barındırdığını ve bu zafiyetin parola sıfırlama token'larını API yanıtında doğrudan sızdırdığını biliyoruz.

**Adım 1: Parola Sıfırlama İsteği ve Token Sızıntısı**
`/api/v1/account/forgot-password` uç noktasına `ben@silentium.htb` kullanıcısı için bir JSON isteği gönderdiğimizde, sunucu bize sadece "E-posta gönderildi" demek yerine, veritabanındaki kullanıcı objesini olduğu gibi geri dönüyor! Bu objenin içinde zayıf bir Bcrypt hash'i ve parolayı sıfırlamak için gereken **`tempToken`** değeri bulunuyor.
![Forgot Password ve Token Sızıntısı](/assets/img/silentium/ilk curl.png)

**Adım 2: Parolayı Değiştirme**
Elde ettiğimiz `tempToken` değerini kullanarak `/api/v1/account/reset-password` uç noktasına ikinci bir istek atıyoruz ve `ben` kullanıcısının parolasını kendi belirlediğimiz bir şifreyle (`Yavuz123!`) değiştirerek hesabı tamamen ele geçiriyoruz.
![Reset Password ve Hesap Ele Geçirme](/assets/img/silentium/ikinci curl.png)

---

## 3. Sömürü (Exploitation): Flowise RCE ve Shell Erişimi

Yeni şifremizle `staging.silentium.htb` adresine giriş yaptığımızda karşımıza **Flowise** (LLM uygulamaları oluşturmak için kullanılan bir arayüz) çıkıyor. 

**Adım 1: API Key Tespiti**
Flowise ortamında komut çalıştırabilmek için API Keys bölümüne girerek `DefaultKey` değerini (`hWp_...`) kopyalıyoruz.
![Flowise API Key Tespiti](/assets/img/silentium/giriş yapılıp api key alındı ve komuta basıldı.png)

**Adım 2: Reverse Shell ve Ortam Değişkeni (ENV) Avı**
Aldığımız API Key ile `/api/v1/node-load-method/customMCP` uç noktasına bir `cURL` isteği atıyoruz. İstek içerisindeki `mcpServerConfig` parametresine Node.js `child_process.exec` fonksiyonunu kullanarak hazırladığımız Reverse Shell payload'umuzu gömüyoruz.
![Flowise Reverse Shell Payload ve Bağlantı](/assets/img/silentium/shell alındı.png)

Bağlantıyı 443 portumuzda yakaladıktan hemen sonra `env` komutunu çalıştırıyoruz. Ortam değişkenleri adeta bir altın madeni:
* `FLOWISE_USERNAME=ben`
* `FLOWISE_PASSWORD=F1l3_d0ck3r`
* **`SMTP_PASSWORD=r04D!!_R4ge`**
![Reverse Shell ve ENV Sızıntısı](/assets/img/silentium/bilgiler alındı.png)

---

## 4. Sistem Erişimi ve User Bayrağı

Geliştiricilerin sıklıkla düştüğü "Parola Tekrar Kullanımı" (Password Reuse) hatasını test etmek için, ortam değişkenlerinde bulduğumuz SMTP parolasını hedef kullanıcının SSH erişimi için deniyoruz.

**Adım 1: SSH Bağlantısı**
`ben` kullanıcısı ve `r04D!!_R4ge` parolası ile hedef sisteme SSH üzerinden başarılı bir şekilde giriş yapıyoruz.
![SSH Bağlantısı ve User Bayrağı](/assets/img/silentium/bilgiler ile ssha girildi ve flag alındı.png)

> **Sonuç:** İlk hedefimiz tamamlandı!
> **Kullanıcı:** `ben`
> **User Flag:** `130194f2196f3e13c2257f7a7c293199`

---

## 5. İç Ağ Pivotlama (Port Forwarding) ve Gogs Keşfi

Sistemde `root` olabilmek için iç ağdaki servisleri inceliyoruz. Hedef makinede sadece lokalden (127.0.0.1) erişilebilen servisler olduğunu tespit ediyoruz ve SSH port yönlendirmesi (Local Port Forwarding) ile bu servisleri kendi makinemize çekiyoruz.

**Adım 1: Pivot İşlemi**
`ssh -L 8080:127.0.0.1:3001 ben@silentium.htb` komutu ile hedef makinenin lokalindeki servisi kendi makinemizin 8080 portuna taşıyoruz.
![SSH Pivot İşlemi](pivot.png)

**Adım 2: Gogs Keşfi ve Kayıt**
Tarayıcımızdan `http://127.0.0.1:8080` adresine giderek **Gogs** (Git servisi) arayüzüne ulaşıyoruz. Sisteme sızabilmek için `neptun` adında yeni bir kullanıcı oluşturuyor ve giriş yapıyoruz.
![Gogs Arayüzü](gogs.png)
![Gogs Kayıt ve Giriş](kayıt olup giriş yaptık.png)

**Adım 3: Personal Access Token (PAT) Üretimi**
Gogs API'sini istismar etmek için hesap ayarlarından `deneme` adında tam yetkili bir Personal Access Token (Kişisel Erişim Belirteci) oluşturuyoruz.
![Gogs Token Üretimi](/assets/img/silentium/bu sayfadan token oluşturuyoruz.png)

---

## 6. Yetki Yükseltme (Privilege Escalation): Gogs RCE ve Root Bayrağı

Elde ettiğimiz bilgiler ışığında, Gogs üzerinde bir zafiyet (CVE-2025-8110) olduğunu biliyoruz. Bu açık, Git hook'ları veya repository yapılandırmaları üzerinden RCE (Uzaktan Kod Çalıştırma) elde etmemize olanak tanıyor.

**Adım 1: Exploit İnfazı ve Root Shell**
Oluşturduğumuz `neptun` kullanıcısı, parolası ve ürettiğimiz PAT token ile `CVE-2025-8110-Gogs-RCE-Exploit` scriptimizi çalıştırıyoruz.
Script arka planda zararlı bir repository oluşturarak `.git/config` dosyasını manipüle ediyor (symlink API üzerinden) ve bize doğrudan `root` yetkilerinde bir shell gönderiyor.

**Adım 2: Root Bayrağını Alma**
Gelen bağlantıyı `nc -lvnp 5555` ile yakaladıktan sonra doğrudan `/root/root.txt` dosyasını okuyoruz ve makineyi tam yetkiyle (Pwned) tamamlıyoruz!
![Gogs RCE Exploit ve Root Bayrağı](ve admin flag exploit ile beraber.png)

> **Sonuç:** Makine başarıyla tamamlandı!
> **Kullanıcı:** `root`
> **Root Flag:** `7d43fa52dc1345f14286d24402d0459a`
