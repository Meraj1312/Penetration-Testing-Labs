# Penetration Test Write-up: Bolt CMS 3.7.1 Authenticated RCE

## Executive Summary  
A remote code execution vulnerability was successfully exploited in a Bolt CMS instance version 3.7.1, leading to full system compromise and retrieval of a sensitive flag. The attack chain involved service enumeration, credential discovery from an exposed internal message, and exploitation of a known authenticated RCE vulnerability in the CMS.

## Target Information  
- **IP Address:** 10.81.144.228  
- **Operating System:** Linux (Ubuntu)  
- **Services Identified:**  
  - SSH (22/tcp)  
  - HTTP Apache (80/tcp)  
  - HTTP Bolt CMS (8000/tcp)  

## Reconnaissance

### Port Scanning  
Initial reconnaissance was performed using Nmap to identify open ports and services.  

![Nmap Scan](Screenshot%202026-01-09%20171724.png)  

**Command:**  
```bash
nmap -sS -sV 10.81.144.228
```

**Findings:**  
- Port 22: OpenSSH 7.6p1  
- Port 80: Apache httpd 2.4.29 (Ubuntu) – Default Ubuntu page  
- **Port 8000: HTTP service running Bolt CMS (PHP 7.2.32-1)**  

The service on port 8000 was identified as Bolt CMS through the HTTP response headers and page content.

### Web Enumeration  
The web application on port 8000 was confirmed to be **Bolt CMS version 3.7.1**. The main page contained a welcome message from the user "bolt" (Jake), indicating he was new to the CMS.  

![Welcome Message](Screenshot%202026-01-09%20171701.png)  

Further inspection of the site revealed an internal IT message exposed publicly, which contained administrative credentials.  

![Internal Message with Credentials](Screenshot%202026-01-09%20173716.png)  

**Credentials Discovered:**  
- **Username:** bolt  
- **Password:** boltadmin123  

### Login Page Discovery  
The Bolt CMS login page was located at:  
```
http://10.81.144.228:8000/bolt/login
```

## Vulnerability Identification  

Using Metasploit, a search for Bolt-related exploits was conducted.  

![Metasploit Search](Screenshot%202026-01-09%20182923.png)  

An authenticated remote code execution exploit was identified:  
- **Module:** `exploit/unix/webapp/bolt_authenticated_rce`  
- **Description:** Bolt CMS 3.7.0 – Authenticated Remote Code Execution  
- **Disclosure:** 2020-05-07  

## Exploitation  

### Metasploit Configuration  
The exploit was configured with the gathered credentials and target information.  

![Metasploit Configuration](Screenshot%202026-01-09%20182953.png)  

**Parameters Set:**  
- `RHOST`: 10.81.144.228  
- `LHOST`: 10.81.125.132 (attacker IP)  
- `USERNAME`: bolt  
- `PASSWORD`: boltadmin123  

### Execution  
The exploit was executed, resulting in a reverse shell with **root privileges**.

## Post-Exploitation  

Once access was gained, the system was enumerated and the flag was located and retrieved.  

![Post-Exploitation and Flag Retrieval](Screenshot%202026-01-09%20183015.png)  

**Commands Executed:**  
```bash
whoami
# root

ls -la
cd /
ls -la
cat flag.txt
```

## Summary of Attack Path  
1. **Service Discovery:** Nmap scan revealed Bolt CMS on port 8000.  
2. **Information Leakage:** Exposed internal forum message provided admin credentials.  
3. **Vulnerability Mapping:** Identified Bolt CMS 3.7.1 as vulnerable to authenticated RCE.  
4. **Exploitation:** Used Metasploit module with found credentials to gain root access.  
5. **Flag Retrieval:** Located and read `flag.txt` in the root directory.  

## Recommendations  
- **Update Bolt CMS** to the latest version to patch known RCE vulnerabilities.  
- **Restrict access** to internal messages and administrative interfaces.  
- **Implement strong password policies** and avoid storing credentials in plaintext.  
- **Use network segmentation** to limit exposure of management interfaces.  
- **Conduct regular security audits** and penetration tests to identify misconfigurations.