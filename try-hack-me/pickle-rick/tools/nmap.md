
Starting with a port scan using Nmap:
```bash
nmap -sC -sV -A 10.113.185.115
```

Output:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
|_http-title: Rick is sup4r cool
```

Only ports **22 (SSH)** and **80 (HTTP)** are open. The web server is running **Apache 2.4.41** on Ubuntu.