# SS Command

> The ss command is a modern replacement for netstat. It shows information about network sockets.

## What is `ss`?

`ss` means **socket statistics**.

It shows information about network sockets:

- listening ports
- active TCP connections
- UDP sockets
- local/remote IPs
- ports
- connection states
- processes using sockets

Think:

```
ip       → interfaces, IPs and routes
ss       → sockets and connections
netstat  → older tool for similar information
nmap     → scans hosts/ports
```

Run:

```
ss
```

You'll probably see TCP connections.

For a much more useful overview:

```
ss -tuln
```

The options mean:

```
-t  TCP
-u  UDP
-l  listening
-n  numeric
```

So:

```
ss -tuln
```

means:

> Show TCP and UDP sockets that are listening, without resolving names.

You might see:

```
Netid  State   Local Address:Port    Peer Address:Port
tcp    LISTEN  0.0.0.0:22            0.0.0.0:*
tcp    LISTEN  0.0.0.0:80            0.0.0.0:*
udp    UNCONN  0.0.0.0:53            0.0.0.0:*
```

## Common `ss` commands

Use:

```
sudo ss -tulnp
```

The `-p` shows the process.

Example:

```
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=1234))
tcp   LISTEN 0.0.0.0:80          users:(("nginx",pid=5678))
```

Now you can answer:

> **What program is listening on this port?**

For example:

```
22  → sshd
80  → nginx
443 → web server
5432 → PostgreSQL
```

That's extremely useful when you're hardening a Linux server.

## `ss` and TCP

Show TCP connections:

```
ss -t
```

More useful:

```
ss -tn
```

All TCP connections, including listening:

```
ss -ant
```

Where:

```
-a → all
-n → numeric
-t → TCP
```

---

## Understanding `ESTAB`

Suppose you see:

```
tcp ESTAB 192.168.1.20:52134 142.250.x.x:443
```

`ESTAB` means:

```
ESTABLISHED
```

There is an active TCP connection.

Conceptually:

```
Your machine
192.168.1.20:52134
       │
       │ TCP
       ↓
142.250.x.x:443
       │
       └── HTTPS
```

Notice something important:

### The ports don't have to be the same.

Your machine might use:

```
52134
```

while the server uses:

```
443
```

This is normal.

## Listening sockets

To see services waiting for connections:

```
ss -ltn
```

TCP:
``
-l → listening
-t → TCP
-n → numeric
```

For UDP:

```
ss -lun
```

Both:

```
ss -tuln
```

This is probably the command you'll use most often when checking a Linux server.

---

# 7. `0.0.0.0` vs `127.0.0.1`

This is extremely important.

Suppose you see:

```
tcp LISTEN 0.0.0.0:8080
```

The application is listening on **all IPv4 interfaces**.

But:

```
tcp LISTEN 127.0.0.1:8080
```

means the application is listening only on localhost.

Conceptually:

```
0.0.0.0:8080
      │
      ├── eno1
      ├── docker0
      └── other IPv4 interfaces
```

while:

```
127.0.0.1:8080
      │
      └── only this machine
```

This distinction becomes very important when you're securing servers.

---

# 8. Find who is using a specific port

Suppose you want to investigate port `8080`.

```
sudo ss -ltnp 'sport = :8080'
```

Or port 22:

```
sudo ss -ltnp 'sport = :22'
```

This is very useful when you think:

> "Something is using this port. What is it?"

---

# 9. Find connections to a specific port

For example, connections involving HTTPS:

```
ss -tn 'dport = :443'
```

You can also look for a source port:

```
ss -tn 'sport = :443'
```

The distinction:

```
sport → source port
dport → destination port
```

---

# 10. UDP

Show UDP sockets:

```
ss -un
```

Listening UDP:

```
ss -lun
```

TCP + UDP:

```
ss -tun
```

TCP + UDP + listening:

```
ss -tuln
```

---

# 11. IPv4 vs IPv6

You can filter by address family.

IPv4:

```
ss -4
```

IPv6:

```
ss -6
```

For example:

```
ss -4tuln
```

means:

```
IPv4
+ TCP
+ UDP
+ listening
+ numeric
```

---

# 12. See the process

This is another important command:

```
sudo ss -tulpn
```

Example:

```
Netid State  Local Address:Port
tcp   LISTEN 0.0.0.0:22
```

With `-p`:

```
users:(("sshd",pid=1024,fd=3))
```

Now you have:

```
Port
 ↓
22
 ↓
Protocol
 ↓
TCP
 ↓
Process
 ↓
sshd
```

That's the beginning of **service enumeration on your own server**.

---

# 13. `ss` vs `netstat`

You can directly compare them.

### Old

```
sudo netstat -tulnp
```

### Modern

```
sudo ss -tulnp
```

For routing:

### Old

```
netstat -rn
```

### Modern

```
ip route
```

So your modern Linux toolkit should look like:

```
ip
 │
 ├── ip addr
 ├── ip link
 ├── ip route
 └── ip neigh

ss
 │
 ├── TCP
 ├── UDP
 ├── listening ports
 └── active connections
```

# 14. `ss` + `ip` = very powerful combination

Imagine you have:

```
ip -br addr
```

and get:

```
eno1    UP    192.168.1.20/24
docker0 UP    172.17.0.1/16
```

Then:

```
sudo ss -tulnp
```

shows:

```
tcp LISTEN 0.0.0.0:22
tcp LISTEN 127.0.0.1:5432
tcp LISTEN 172.17.0.1:8080
```

You can start reasoning:

```
22
└── SSH
    └── exposed on all IPv4 interfaces

5432
└── PostgreSQL
    └── localhost only

8080
└── application
    └── listening on Docker interface
```

That is exactly the type of reasoning you'll need for **Linux hardening and cybersecurity**.

---

# 15. Commands I recommend memorizing

Don't try to memorize everything.

Start with these:

```
ss -tuln
```

**What ports are listening?**

```
sudo ss -tulnp
```

**What ports are listening and which processes own them?**

```
ss -ant
```

**What TCP connections exist?**

```
ss -uan
```

**What UDP sockets exist?**

```
ss -4tuln
```

**What IPv4 ports are listening?**

```
ss -6tuln
```

**What IPv6 ports are listening?**

```
sudo ss -ltnp 'sport = :22'
```

**What is using TCP port 22?**