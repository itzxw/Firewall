# Userspace C Firewall

A custom firewall running in userspace, written entirely in C. The project intercepts, analyzes, and filters network traffic captured through a virtual `tun0` interface, using advanced routing rules on Linux.

## Overview
Unlike a traditional firewall based on `iptables`/`nftables` (which runs inside the kernel), this project implements all of the filtering logic in a userspace process (`fwall`), written in C. The operating system hands packets to this process through a virtual network interface (TUN), the process decides what to do with each packet (block it, let it through, or "trap" the connection via tarpit), and returns the result — all without requiring any kernel modification.

### How a TUN interface works

A TUN interface ("network TUNnel") is a virtual network interface created in software, with no physical hardware behind it. It operates at layer 3 (network) of the OSI model — meaning the program controlling it reads and writes "raw" IP packets (without an Ethernet header), unlike a TAP interface, which operates at layer 2.

**The basic flow is:**

+ The program opens `/dev/net/tun` and uses `ioctl(TUNSETIFF)` to register a new interface (e.g. tun0) in the kernel, associated with its file descriptor.
+ The kernel starts treating `tun0` like any other network interface: it can be assigned an IP, have routes associated with it, show up in `ip link`, etc.
+ Whenever the kernel decides (based on the routing table) that a packet should be sent through `tun0`, instead of transmitting it over a cable/radio, it delivers that packet to the process that opened the file descriptor, as if it were a `read()` from a regular file.
+ The process can inspect, modify, drop, or return that packet with a `write()` on the same file descriptor — and the kernel treats that `write()` as if the packet had "arrived" through the interface.

## Scope and important limitations

This firewall only filters traffic that the kernel decides to route through the `tun0` interface — not all of the machine's traffic.
This is a relevant scope decision for understanding the project correctly:

+ The `firewall.sh` script creates the `tun0` interface with the IP `10.0.0.1/24` and adds a static route only for the `10.0.0.0/24` network.
+ This means that **only packets destined for that specific subnet** are routed into the program and, therefore, evaluated against the rules in `conf.rc`.
+ Traffic to any other destination (for example, a request to a server on the internet) keeps following the system's default route, leaving through the physical interface (`eth0`, Wi-Fi, etc.) — **without ever passing through the firewall**.

In other words: the "default deny" implemented in the code (packets with no matching rule are dropped) is valid **within the universe of packets the program actually receives**, and does not represent a security policy for the whole machine. For the firewall to intercept all outbound traffic from a host, it would be necessary to replace the system's default route (`0.0.0.0/0`) to point to `tun0` and implement real forwarding of the allowed packets back to the network — which is outside the current scope of the project.

## Project architecture

The code is divided into three modules:

| File | Responsibility |
|---|---|
| `main.c` | Main loop: TUN interface creation, packet reading, rule application, forwarding/dropping, logging. |
| `rules.c` / `rules.h` | Parsing of the configuration file (`conf.rc`) and building the in-memory rule list. |
| `checksum.c` / `checksum.h` | Manual calculation of IP, TCP, and ICMP checksums, required whenever a packet is modified before being sent again. |
| `firewall.sh` | Orchestration of the network infrastructure (creating/removing the TUN interface, IP, routes) and management of the process lifecycle (`start`/`stop`/`restart`/`status`), including permission checks and error handling. |

### Installation
Make sure you have the necessary dependencies; otherwise, run the commands below:
```bash
# Debian/Ubuntu
sudo apt install build-essential iproute2

# Fedora
sudo dnf groupinstall "Development Tools"
sudo dnf install iproute

# Arch
sudo pacman -S base-devel iproute2

# optional dependencies for testing (Debian/Ubuntu)
sudo apt install netcat-openbsd tcpdump python3-scapy
```
With the dependencies installed, clone the repository and start the firewall
```bash
git clone https://github.com/itzxw/psel
cd psel
git checkout my-firewall
```
```bash
chmod +x firewall.sh
make
sudo ./firewall.sh start
```
## Testing the firewall

Since the firewall only sees traffic routed to the `10.0.0.0/24` network (see [Scope and important limitations](#scope-and-important-limitations)), test traffic must be generated explicitly against the `tun0` interface or the IP `10.0.0.1`. Some ways to do that:

**Ping (tests the `ALLOW` action with an ICMP reply):**
```bash
ping -c 3 10.0.0.1
```

**TCP connection (tests `DENY`/`TARPIT`, depending on the rule for the source IP):**
```bash
nc 10.0.0.1 80
```

**Watching the raw traffic on the interface:**
```bash
sudo tcpdump -i tun0 -vv
```

**Generating custom packets** (spoofed source IP, specific payload, arbitrary TCP flags), useful for testing rules and the malicious payload filter in a controlled way, using [Scapy](https://scapy.net/):
```python
from scapy.all import *
send(IP(src="10.0.0.99", dst="10.0.0.1")/TCP(dport=80, flags="S"), iface="tun0")
```

## References
- [RFC 1071 - Computing the Internet checksum](https://datatracker.ietf.org/doc/html/rfc1071)
- https://www.geeksforgeeks.org/computer-networks/tcp-ip-packet-format/
- https://stackoverflow.com/questions/75261549/setting-an-ip-address-to-a-tun-in-c
- https://www.linkedin.com/pulse/how-i-built-private-ip-network-using-tun-interface-linux-alan-guldc/
- [Tarpitting (LaBrea Tarpit)](https://en.wikipedia.org/wiki/Tarpit_(networking))
