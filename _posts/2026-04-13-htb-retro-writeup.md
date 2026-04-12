---
title: "Hack The Box: Retro - Makine Çözümü (Write-up)"
date: 2026-04-12 18:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, smb, rpc, impacket, privilege-escalation, windows]
---

# 🕹️ Hack The Box: Retro - Sızma Testi Raporu

**İşletim Sistemi:** Windows
**Hedef:** SMB Null Session Keşfi, Bilgi Sızıntısı (Information Disclosure), Machine Account Parola Sıfırlama ve Domain Admin Erişimi.

---

## 1. Keşif ve Bilgi Toplama (Recon)

Hedef sistem üzerinde gerçekleştirdiğimiz Nmap taraması, makinenin bir Domain Controller (DC) olduğunu ve Kerberos, LDAP, SMB gibi kritik servislerin açık olduğunu gösteriyor.
![Nmap Taraması](/assets/img/retro/nmapsorgu.jpeg)

**Adım 1:** Domain adının `retro.vl` olduğunu tespit ettikten sonra `/etc/hosts` dosyamızı güncelliyoruz.
![Hosts Güncelleme](/assets/img/retro/nxc%20guest%20sorgu.jpeg)

---

## 2. SMB Null Session ve Ganimet Keşfi

Active Directory ortamlarında düşük yetkili veya anonim erişimler bazen çok kritik bilgileri sızdırabilir.

**Adım 1:** NetExec kullanarak `Guest` kullanıcısı ile SMB paylaşımlarını tarıyoruz. `Trainees` paylaşımına okuma yetkimiz olduğunu fark ediyoruz.
![SMB Paylaşımları](/assets/img/retro/asilnxcguest.jpg)

**Adım 2:** `smbclient` ile paylaşıma bağlanıyoruz. Klasör içinde `ToDo.txt` ve ilk hedefimiz olan `user.txt` bayrağını buluyoruz.
![User Flag](/assets/img/retro/todo.jpeg)

> **ToDo.txt Notu:** James'ten Thomas'a bir not: *"Eski bankacılık yazılımından kurtulmanın vakti geldi... makine hesabıyla (computer account) başlamalıyız. O benden bile yaşlı."* > Bu not, bize yetki yükseltme yolu için bir ipucu veriyor.

---

## 3. Yetki Yükseltme (Privilege Escalation)

James'in notundaki "eski bankacılık yazılımı" ve "makine hesabı" vurgusu, bizi RPC üzerinden yapılabilecek bir parola sıfırlama saldırısına yönlendiriyor.

**Adım 1 (Parola Sıfırlama):** Impacket araç setinden `impacket-changepasswd` betiğini kullanarak, `banking$` makine hesabının parolasını `deneme123!` olarak güncelliyoruz.
![Parola Sıfırlama](/assets/img/retro/impacket.jpeg)

---

## 4. Domain Admin ve Root Bayrağı

Elde ettiğimiz makine hesabı yetkileriyle sistem üzerinde en üst yetkilere sahip olmayı deniyoruz.

**Adım 1 (WinRM Erişimi):** Administrator hash'ini veya doğrudan yetkili bir oturumu kullanarak Evil-WinRM üzerinden sisteme giriş yapıyoruz.
![WinRM Giriş](/assets/img/retro/winrm%20giriş.jpeg)

**Adım 2 (Final):** Administrator masaüstüne giderek **root.txt** bayrağını okuyoruz ve operasyonu başarıyla tamamlıyoruz!
![Root Flag](/assets/img/retro/asilsonucwinrm.jpeg)
