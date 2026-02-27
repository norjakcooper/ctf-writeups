Proceeding with directory enumeration using ffuf:
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.113.185.115/FUZZ -e .php,.txt,.html,.bak -fs 279 -t 100 2>/dev/null
```

Output:
```
assets      [Status: 301]
denied.php  [Status: 302]
index.html  [Status: 200]
login.php   [Status: 200]
portal.php  [Status: 302]
robots.txt  [Status: 200]
```

The most interesting findings are `login.php` and `robots.txt`.