# IP Command

The command follows this pattern:

```
ip <object> <command>
```

The objects you'll use most are:

```
ip addr       → IP addresses/interfaces
ip link       → network interfaces
ip route      → routing table
ip neigh      → ARP/neighbor table
ip vlan       → VLAN information
ip rule       → advanced routing rules
```

The four you should master first are:

```
ip addr
ip link
ip route
ip neigh
```

---

# `ip addr` — IP addresses

Start with:

```
ip addr
```

or the shorter:

```
ip a
```

You'll see something like:

```
2: eno1:
    inet 192.168.1.20/24
    inet6 fe80::1234/64
    link/ether aa:bb:cc:dd:ee:ff
    state UP
```

There are several pieces of information here.

### Interface

```
eno1
```

This is the network interface.

### IPv4

```
192.168.1.20/24
```

This means:

```
IP:      192.168.1.20
Network: 192.168.1.0/24
```

### IPv6

```
fe80::1234/64
```

### MAC address

```
aa:bb:cc:dd:ee:ff
```

# Understanding the interface

You might see:

```
1: lo:
```

That's the **loopback** interface.

```
127.0.0.1
```

means:

```
this machine itself
```

Then you might see:

```
2: eno1:
```

Your physical Ethernet interface.

And perhaps:

```
3: wlp2s0:
```

Your Wi-Fi interface.

Or:

```
docker0
```

A Docker bridge.

This is particularly useful for you because your Hyperia/Docker experiments will create additional interfaces.

# `ip link`

Now:

```
ip link
```

or:

```
ip l
```

This focuses on the **network interfaces themselves**, rather than their IP addresses.

Example:

```
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

Important states:

```
UP
```

means the interface is administratively enabled.

```
DOWN
```

means it is disabled.

You can also see the MAC address:

```
link/ether aa:bb:cc:dd:ee:ff
```

# `ip link` vs `ip addr`

This distinction is worth remembering:

```
ip link
   ↓
"What network interfaces do I have?"

ip addr
   ↓
"What IP addresses are assigned to them?"
```

For example:

```
eno1
 │
 ├── MAC: aa:bb:cc:dd:ee:ff
 └── IP: 192.168.1.20/24
```

# Turning an interface on/off

You can administratively enable an interface:

```
sudo ip link set eno1 up
```

Disable it:

```
sudo ip link set eno1 down
```

Be careful with this on a remote machine: taking down the interface you're connected through can disconnect you.

# Assigning an IP address

You can manually assign an address:

```
sudo ip addr add 192.168.10.10/24 dev eno1
```

Now:

```
ip addr show eno1
```

should show the address.

To remove it:

```
sudo ip addr del 192.168.10.10/24 dev eno1
```

This is useful for labs because you can configure networking without immediately editing configuration files.

# `ip route` — routing table

This is probably the **second most important `ip` command**.

Run:

```
ip route
```

You might see:

```
default via 192.168.1.1 dev eno1
192.168.1.0/24 dev eno1 proto kernel scope link src 192.168.1.20
```

Let's understand it.

### Default route

```
default via 192.168.1.1 dev eno1
```

means:

> If Linux doesn't have a more specific route, send the packet to `192.168.1.1` through `eno1`.

That's your **default gateway**.

# The routing decision

Suppose you execute:

```
ping 8.8.8.8
```

Linux asks:

> Which route should I use?

It sees:

```
8.8.8.8
```

doesn't belong to:

```
192.168.1.0/24
```

So Linux uses:

```
default via 192.168.1.1
```

The path becomes:

```
Your PC
192.168.1.20
     │
     ▼
192.168.1.1
Gateway
     │
     ▼
Internet
     │
     ▼
8.8.8.8
```

This connects directly to the routing concepts we studied.

#  Ask Linux which route it would use

This is an extremely useful command:

```
ip route get 8.8.8.8
```

You might get:

```
8.8.8.8 via 192.168.1.1 dev eno1 src 192.168.1.20
```

Now Linux is basically telling you:

```
Destination → 8.8.8.8
Gateway     → 192.168.1.1
Interface   → eno1
Source IP   → 192.168.1.20
```

This is an excellent troubleshooting command.

# `ip neigh` — ARP table

Now we connect `ip` to **ARP**.

Run:

```
ip neigh
```

You might see:

```
192.168.1.1 dev eno1 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

This means:

```
IP
192.168.1.1
   ↓
MAC
aa:bb:cc:dd:ee:ff
```

Remember:

> ARP resolves an IPv4 address to a MAC address on the local network.

So:

```
ip route
```

tells you:

> Where should the packet go?

while:

```
ip neigh
```

tells you:

> What MAC address should I use for the local next hop?

# A very important distinction

Suppose you're accessing:

```
8.8.8.8
```

Your computer does **not** need the MAC address of `8.8.8.8`.

It needs the MAC address of the next local hop:

```
192.168.1.1
```

So:

```
8.8.8.8
   │
   │ routing
   ▼
192.168.1.1
   │
   │ ARP
   ▼
MAC of router
```

This is one of the most important concepts in Linux networking.

---

# `ip` and Docker

Now things get especially interesting for you.

Run:

```
ip link
```

on a machine running Docker.

You may see:

```
docker0
vethxxxx
vethyyyy
```

Docker creates networking components such as:

```
Linux network namespace
        │
        ▼
       veth
        │
        ▼
     docker0
        │
        ▼
      NAT
        │
        ▼
    physical NIC
```

So when you understand:

```
ip link
ip addr
ip route
```

you are also starting to understand what Docker is doing underneath.

---

# VLANs with `ip`

You can also create VLAN interfaces using `ip`.

For example:

```
sudo ip link add link eno1 name eno1.10 type vlan id 10
```

Then:

```
ip link
```

may show:

```
eno1
eno1.10
```

And:

```
ip -d link show eno1.10
```

will show VLAN-specific information.

You can then assign an address:

```
sudo ip addr add 192.168.10.10/24 dev eno1.10
```

This connects directly with the VLAN lesson we had earlier.