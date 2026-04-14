---
title: "Hack The Box: Delegate - Write-up"
date: 2026-04-14 02:00:00 +0300
categories: [Walkthrough, HackTheBox]
tags: [active-directory, kerberoasting, unconstrained-delegation, petitpotam, dcsync, windows, medium]
---

# 🛡️ Hack The Box: Delegate

**Zorluk:** Medium  
**İşletim Sistemi:** Windows  
**Hedef:** Targeted Kerberoasting, Unconstrained Delegation İstismarı ve DCSync ile Domain Ele Geçirme.

---

## 1. Keşif ve İlk Bilgi Toplama (Recon)

Operasyona her zaman olduğu gibi ağ üzerindeki servisleri ve giriş noktalarını analiz ederek başlıyoruz.

**Adım 1: Nmap Taraması** Nmap sonuçları, makinenin bir Domain Controller (DC) olduğunu ve Kerberos, LDAP, SMB gibi kritik servislerin açık olduğunu teyit ediyor.
![Nmap Taraması](/assets/img/delegate/delegate%20nmapp.jpeg)

**Adım 2: SMB Erişimi ve Bilgi Sızıntısı** `Guest` hesabının aktif olduğunu görerek `impacket-smbclient` ile sisteme bağlanıyoruz. `NETLOGON` paylaşımı altında bulduğumuz script içerisinden **A.Briggs** kullanıcısına ait parolayı (`P4ssw0rd1#123`) sızdırıyoruz.
![SMB Erişimi](/assets/img/delegate/impacket%20smb.jpeg)
![A.Briggs Giriş Kontrolü](/assets/img/delegate/abrigss%20nxc%20pass%20kontrolü.jpeg)

---

## 2. Kan Analizi ve Yetki Yükseltme (BloodHound)

**Adım 1:** BloodHound analizi sonucunda **A.Briggs** kullanıcısının **N.Thompson** üzerinde `GenericWrite` yetkisi olduğunu keşfediyoruz.
![BloodHound Analizi](/assets/img/delegate/bloodhound.jpeg)

**Adım 2 (Targeted Kerberoasting):** `bloodyAD` kullanarak N.Thompson hesabına rastgele bir SPN atıyoruz.
![SPN Atama](/assets/img/delegate/spn%20ekleme.jpeg)

**Adım 3:** SPN atandıktan sonra kullanıcının Kerberos hash'ini çekiyoruz ve hashcat ile kırıyoruz. Parola: `KALEB_2341`.
![N.Thompson Hash](/assets/img/delegate/nthompson%20hash.jpeg)
![Hash Kırıldı](/assets/img/delegate/hash%20kırıldı.jpeg)

---

## 3. Unconstrained Delegation İstismarı

N.Thompson kullanıcısı ile sisteme dahil olduğumuzda, elimizdeki yetkileri test ediyoruz.

**Adım 1: Yetki ve Grup Kontrolü** Thompson'ın parolasıyla giriş yapabildiğimizi doğruluyoruz ve sistemdeki yetkilerini inceliyoruz.
![Thompson Giriş Kontrolü](/assets/img/delegate/thompson%20pass%20kontrolü.jpeg)
![Yetki Kontrolü](/assets/img/delegate/yetki%20görünüyor.jpeg)

**Adım 2: Sahte Makine Hesabı ve Delegasyon** Thompson'ın yeni makine hesabı açma yetkisini (`SeMachineAccountPrivilege`) kullanarak **neptun$** hesabını oluşturuyoruz. Ardından bu hesabı **Unconstrained Delegation** için güvenilir hale getiriyoruz.
![Makine Hesabı Ekleme](/assets/img/delegate/neptun%20adlı%20makine%20hesabı%20ekledik.jpeg)
![Trusted Delegation](/assets/img/delegate/trusted%20delegation.jpeg)

---

## 4. Final: PetitPotam ve Domain Admin

**Adım 1: Krbrelayx ve PetitPotam** `krbrelayx` dinleyicisini başlattıktan sonra PetitPotam saldırısı ile Domain Controller'ı bizim sahte makinemize kimlik doğrulamaya zorluyoruz. DC'nin Kerberos biletini kaptıktan sonra hash'leri elde ediyoruz.
![DNS Kaydı](/assets/img/delegate/dns%20record.jpeg)
![DCSync Başlatıldı](/assets/img/delegate/secretsdump%20ile%20hashler%20alındı.jpeg)

**Adım 2: Bayrakların Alınması** Elde edilen Administrator hash'i ile sisteme giriş yapıyor ve her iki bayrağı elde ediyoruz.
![User Flag](/assets/img/delegate/user%20flag.jpeg)
![Root Flag](/assets/img/delegate/rootflag.jpeg)
