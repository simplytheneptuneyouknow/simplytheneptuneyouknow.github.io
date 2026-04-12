---
title: "Hack The Box: Baby - Makine Çözümü (Write-up)"
date: 2026-04-12 21:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, ldap, diskshadow, ntds, password-spraying, windows, medium]
---

# 🍼 Hack The Box: Baby - Sızma Testi Raporu

**Zorluk:** Medium  
**İşletim Sistemi:** Windows  
**Hedef:** LDAP Anonim Erişim, Password Spraying, Diskshadow ile NTDS Sızıntısı ve Domain Admin Hakları.

---

## 1. Keşif ve LDAP Üzerinden Bilgi Toplama

Operasyona hedef sistemin LDAP servisini anonim olarak sorgulayarak başlıyoruz.

**Adım 1:** Nmap ile sistemin bir Domain Controller olduğunu ve LDAP servisinin açık olduğunu doğruluyoruz.
![Nmap Taraması](/assets/img/baby/nmap.jpeg)

**Adım 2:** `ldapsearch` kullanarak dizindeki kullanıcıları anonim olarak listeliyoruz. Elde ettiğimiz kullanıcı adlarıyla bir `users.txt` listesi oluşturuyoruz.
![LDAP Sorgusu](/assets/img/baby/ldap%20search.jpeg)
![Geniş LDAP Taraması](/assets/img/baby/daha%20geniş%20bir%20ldap%20search.jpeg)

---

## 2. İlk Erişim: Password Spraying

**Adım 1:** **Teresa Bell** kullanıcısının açıklama (description) kısmında sızan `BabyStart123!` parolasını buluyoruz.
![Parola Sızıntısı](/assets/img/baby/description%20da%20şifre%20bulunuyor.jpeg)

**Adım 2 (Kritik Nokta):** Bu parolanın diğer kullanıcılar için geçerli olup olmadığını anlamak için NetExec (nxc) ile `Password Spraying` saldırısı gerçekleştiriyoruz. Listenin en sonunda **Caroline Robinson** kullanıcısı için `STATUS_PASSWORD_MUST_CHANGE` yanıtını alıyoruz. Bu, parolanın doğru olduğunu ancak değiştirilmesi gerektiğini gösteriyor!
![Password Spray Başarılı](/assets/img/baby/yeni%20liste%20ile%20password%20sprey.jpeg)

---

## 3. Caroline Olarak Giriş ve Yetki Yükseltme

**Adım 1:** `smbpasswd` aracı ile Caroline'ın parolasını güncelliyoruz ve `evil-winrm` kullanarak sisteme dahil oluyoruz.
![WinRM Girişi](/assets/img/baby/elde%20edilen%20credentials%20ile%20winrm%20alınıyor.jpeg)

---

## 4. Yetki Yükseltme: Diskshadow ve NTDS.dit Sızıntısı

Caroline üzerinden aldığımız yetkilerle Domain'in tüm parolalarını barındıran NTDS.dit dosyasını çalmak için Diskshadow servisini kullanacağız.

**Adım 1:** Hazırladığımız `script.txt` dosyasını sisteme yükleyip Volume Shadow Copy (VSS) oluşturuyoruz.
![Script Upload](/assets/img/baby/makineye%20upload%20ediyoruz.jpeg)
![Diskshadow Expose](/assets/img/baby/upload%202.jpeg)

**Adım 2:** Shadow Copy üzerinden kilitli olmayan dosyaları `robocopy /b` ile çekiyoruz ve kayıt defterini (registry) yedekliyoruz.
![Robocopy NTDS](/assets/img/baby/ntds%20kopyalama.jpeg)
![Reg Save](/assets/img/baby/reg%20save.jpeg)

---

## 5. Domain Ele Geçirme (DCSync & Hash Dump)

**Adım 1:** Sızdırdığımız `ntds.dit`, `SAM` ve `SYSTEM` dosyalarını kendi makinemize indirip Impacket `secretsdump.py` ile Administrator hash'ini söküp alıyoruz.
![NTDS Download](/assets/img/baby/download%20ntds.jpeg)
![NTDS Secretdump](/assets/img/baby/ntds%20secretdump%20son.jpeg)

**Final:** Elde edilen hash ile sisteme Administrator olarak bağlanıyor ve tam kontrolü ele geçiriyoruz.
![NetExec Pwned](/assets/img/baby/netecex%20ile%20son.jpeg)
![Root Flag](/assets/img/baby/flag%20alındı.jpeg)
