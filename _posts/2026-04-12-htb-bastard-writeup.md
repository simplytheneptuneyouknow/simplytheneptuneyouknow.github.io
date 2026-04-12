---
title: "Hack The Box: Bastard - Write-up"
date: 2026-04-12 22:30:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [drupal, drupalgeddon, rce, juicy-potato, privilege-escalation, windows, medium]
---

# 🗿 Hack The Box: Bastard Write Up

**Zorluk:** Medium
**İşletim Sistemi:** Windows
**Hedef:** Drupal 7 Uzaktan Kod Çalıştırma (RCE), SeImpersonatePrivilege İstismarı ve Juicy Potato ile Root Erişimi.

---

## 1. Keşif ve Bilgi Toplama (Reconnaissance)

Hedef sistem üzerindeki servisleri ve versiyonları analiz ederek zayıf halkayı tespit ediyoruz.

**Adım 1: Nmap Taraması**
Nmap sonuçları, hedefte bir Microsoft IIS 7.5 web sunucusu olduğunu gösteriyor.
![Nmap Taraması](/assets/img/bastard/nmap.jpeg)

**Adım 2: Web Keşfi**
Web sayfasına gittiğimizde sistemin **Drupal** CMS kullandığı açıkça görülüyor.
![Web Giriş](/assets/img/bastard/drupal%20ile%20yapılmış.jpeg)

`robots.txt` dosyası da Drupal'ın standart dizin yapısını onaylıyor.
![Robots.txt](/assets/img/bastard/robots.txt.jpeg)

---

## 2. Zafiyet Analizi ve Sömürü (Exploitation)

**Adım 1: Versiyon Tespiti**
`changelog.txt` dosyasından sistemin **Drupal 7.54** kullandığını öğreniyoruz.
![Drupal Versiyon](/assets/img/bastard/drupal%20versiyon.jpeg)

**Adım 2: Exploit Seçimi**
Bu sürüm, kritik bir RCE zafiyeti olan **Drupalgeddon 2 (CVE-2018-7600)** saldırısına karşı savunmasızdır.
![Exploit Bulundu](/assets/img/bastard/exploit%20bulundu.jpeg)

**Adım 3: İlk Erişim (Foothold)**
Exploit aracılığıyla sistemde komut çalıştırıyor (RCE) ve kendi payload'umuzu (`payload.exe`) sisteme aktararak bir reverse shell elde ediyoruz.
![RCE Başarılı](/assets/img/bastard/rce%20yedi.jpeg)
![Payload Aktarımı](/assets/img/bastard/payload%20aktarımı.jpeg)
![Shell Geldi](/assets/img/bastard/shell%20geldi.jpeg)

Düşük yetkili kullanıcı ile ilk bayrağı alıyoruz.
![User Flag](/assets/img/bastard/user%20flag.jpeg)

---

## 3. Yetki Yükseltme (Privilege Escalation)

Sistemde `whoami /priv` komutunu çalıştırdığımızda, IIS servis hesabının **`SeImpersonatePrivilege`** yetkisine sahip olduğunu görüyoruz. Bu, Juicy Potato saldırısı için yeşil ışıktır.
![Yetki Kontrolü](/assets/img/bastard/yetkilerde%20seimpersonate%20göründü.jpeg)

**Adım 1: Juicy Potato İstismarı**
Doğru CLSID değerini kullanarak Juicy Potato üzerinden `NT AUTHORITY\SYSTEM` yetkilerine yükseliyoruz.
![Juicy Potato Saldırısı](/assets/img/bastard/juicy%20kod.jpeg)

---

## 4. Final: Root Bayrağı

Sistemin en üst yetki seviyesine ulaştıktan sonra `Administrator` masaüstündeki son bayrağı okuyarak operasyonu başarıyla tamamlıyoruz.
![Root Flag](/assets/img/bastard/WhatsApp%20Image%202026-04-04%20at%2021.54.52.jpeg)
