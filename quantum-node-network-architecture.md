# Classical Networking for Quantum-Secure Node Networks
### A reference for scaling QKD nodes from 2 → 15+ across India

> **Context this was written for:** you're building the classical networking layer that lets QKD nodes exchange the public/classical data QKD needs (sifting, error reconciliation, privacy amplification, KMS-KMS key synchronization) — starting at 2 nodes with sub-second sync, scaling toward 5–15 nodes distributed nationally. This assumes a Debian-family Linux stack (Kali/Ubuntu/Debian), Docker-based nodes, and an SDN control plane (Ryu/OpenFlow-style), since that's the common baseline for this kind of testbed.

---

## Table of Contents
1. [Networking Fundamentals](#1-networking-fundamentals)
2. [Mobile Generations: 3G → 6G](#2-mobile-generations-3g--6g)
3. [Where Your Project Actually Sits](#3-where-your-project-actually-sits-the-classical-channel-in-qkd)
4. [IP Addressing & Subnetting](#4-ip-addressing--subnetting)
5. [DHCP](#5-dhcp-dynamic-host-configuration-protocol)
6. [Linux Network Configuration](#6-linux-network-configuration)
7. [Secure Tunnels for a WAN Node Network](#7-secure-tunnels-for-a-wan-node-network)
8. [Time Synchronization — the "millisecond" question](#8-time-synchronization--the-millisecond-question)
9. [Application-Layer Data Sync Between Nodes](#9-application-layer-data-sync-between-nodes)
10. [Physics Reality Check: Latency Across India](#10-physics-reality-check-latency-budgets-across-india)
11. [Topology for Scaling 2 → 15 Nodes](#11-network-topology-for-scaling-2--15-nodes)
12. [Security Baseline for the Classical Channel](#12-security-baseline-for-the-classical-channel)
13. [Command Cheat Sheet](#13-command-cheat-sheet)
14. [Recommended Stack for Your Setup](#14-recommended-stack-for-your-setup)
15. [References](#15-references)

---

## 1. Networking Fundamentals

### 1.1 The layered models

| OSI (7 layers) | TCP/IP (4 layers) | What lives here | Examples |
|---|---|---|---|
| 7 Application | Application | Your actual protocol/data | HTTP, gRPC, DNS, your KMS API |
| 6 Presentation | Application | Encoding, encryption, serialization | TLS, JSON/Protobuf |
| 5 Session | Application | Session management | TLS session, WebSocket |
| 4 Transport | Transport | End-to-end delivery | TCP, UDP, QUIC |
| 3 Network | Internet | Logical addressing, routing | IP, ICMP |
| 2 Data Link | Link | Framing, local delivery | Ethernet, Wi-Fi, ARP |
| 1 Physical | Link | Bits on the wire/fiber/air | Copper, fiber, radio |

Everything below (DHCP, IP, Linux config, VPNs) operates at layers 2–4. Your QKD post-processing/KMS traffic is layer 7, riding on top.

### 1.2 Core metrics (get the definitions exact — they matter for your latency budget)

- **Bandwidth**: max theoretical data rate of a link (Mbps/Gbps).
- **Throughput**: actual achieved data rate (always ≤ bandwidth).
- **Latency**: one-way time for a bit to travel from A to B.
- **RTT (round-trip time)**: what `ping` shows you — roughly 2× one-way latency plus processing at both ends.
- **Jitter**: variation in latency between packets — worse for you than raw latency, since it breaks "very updated, millisecond" assumptions if it's inconsistent.
- **Packet loss**: % of packets that never arrive — forces retransmission (adds latency) or, on UDP, silent data loss.

### 1.3 LAN vs MAN vs WAN

- **LAN**: one building/campus. Your current QNet5-style setup (Kali controller + TP-Link switch + Docker nodes) is a LAN.
- **MAN**: city-scale (e.g., a metro dark-fiber ring).
- **WAN**: your target — nodes "across the country India" is a WAN by definition, and that single fact changes almost every assumption below (latency floor, who owns the fiber, whether DHCP/broadcast even works between sites, whether you need a VPN overlay).

### 1.4 Topologies (preview — full discussion in §11)

Point-to-point, star (hub-and-spoke), ring, mesh (full/partial), tree/hierarchical. Which one you pick for the **classical control plane** doesn't have to match the topology of your **quantum links** — this distinction matters a lot for you and is unpacked in §11.

---

## 2. Mobile Generations: 3G → 6G

You asked for this "mostly for classical computer networks" context — here's the part that's actually transferable to your project: **every cellular generation is fundamentally solving the same problem you have** — many geographically distributed nodes, strict latency budgets by network segment, and a control layer (increasingly SDN-based) that manages it all. 5G's architecture in particular is close enough to your use case that it's worth understanding in detail, not just as trivia.

### 2.1 Generation-by-generation

| Gen | Standard | Core network | Radio access | Peak/typical rate | Latency | Key architectural idea |
|---|---|---|---|---|---|---|
| **3G** | UMTS (W-CDMA) / CDMA2000, ITU IMT-2000 | Circuit + packet switched (hybrid) | WCDMA | ~384 kbps mobile, ~2 Mbps stationary; HSPA later reached ~14 Mbps down | ~100–500 ms | First "always-on" mobile data |
| **4G** | LTE / LTE-Advanced, ITU IMT-Advanced | **All-IP** (Evolved Packet Core, EPC) | OFDMA (down) / SC-FDMA (up) | ~100 Mbps mobility, up to ~1 Gbps (LTE-A, carrier aggregation) | ~30–50 ms, ~10 ms optimized | Flat, all-IP core; mobile broadband becomes primary use case |
| **5G** | NR, 3GPP Release 15+, ITU IMT-2020 | 5GC — service-based architecture (AMF, SMF, UPF, PCF, UDM, NRF...) | Massive MIMO, mmWave + sub-6 GHz | Target 20 Gbps down / 10 Gbps up (peak); ~100 Mbps user-experienced | Target ≤1 ms (URLLC), ~4 ms (eMBB) | **Split architecture** (fronthaul/midhaul/backhaul), network slicing, SDN/NFV, edge computing |
| **6G** | IMT-2030 (ITU-R), 3GPP Release 20/21 | AI-native, integrated sensing | Sub-THz exploration, extreme MIMO | Target ~terabit-class peak (still being finalized) | Sub-ms targets, extreme reliability | AI-native air interface, non-terrestrial (satellite/HAPS) integration |

**Caveat worth knowing:** independent evaluation of the actual submitted 5G radio technologies found none fully satisfied the strict IMT-2020 URLLC latency/reliability numbers on paper — the *target* numbers above are design goals, not universally achieved guarantees. Keep that in mind when you see "5G = 1ms latency" as a marketing claim.

### 2.2 5G's internal architecture — the genuinely useful part

5G splits its transport network into three segments, each with its own distance and latency budget. This is the single most directly transferable fact from cellular networking to your project, because it's a real-world example of exactly the kind of latency-budget-by-segment planning you'll need to do:

| Segment | What it connects | Distance | Transport | Capacity | One-way delay | Security |
|---|---|---|---|---|---|---|
| **Fronthaul** | Radio Unit (RRH/RRU) ↔ Distributed Unit (DU) | up to ~20 km | eCPRI | ~20 Gbps | ~100 μs | MACsec / IPsec |
| **Midhaul** | DU ↔ Centralized Unit (CU) | 100–200 km | Ethernet, point-to-multipoint | 10/25 GbE | 1–2 ms | MACsec / IPsec |
| **Backhaul** | CU ↔ 5G Core | up to ~2000 km | IP/Ethernet, any-to-any WAN | 10/100 GbE | >10 ms | MACsec / IPsec |

Notice the backhaul row: **"up to 2000 km, one-way delay >10 ms."** That's the same order of magnitude as India's geographic extent, and it's the honest floor for wide-area classical links even in a network designed by people whose entire job is minimizing latency. Section 10 does the same exercise for your specific city pairs.

5G also introduced, and you'll want to borrow the vocabulary/pattern from:
- **SDN** (control/data plane separation, programmable via OpenFlow/NETCONF) — you already use this (Ryu).
- **NFV** (network functions as software instead of dedicated hardware).
- **Network slicing** (multiple logical networks sharing physical infrastructure) — directly applicable if you ever want to run your quantum-node control plane and other lab traffic over the same physical links without interference.
- **MEC** (Multi-access Edge Computing) — conceptually similar to placing KMS/post-processing compute close to each quantum node instead of centralizing it.
- A **hierarchical SDN pattern**: separate controllers per network domain (RAN controller, transport controller, core controller), all reporting up to a central **controller orchestrator**. This is precisely the template to reuse when your single-domain Ryu/OpenFlow controller needs to become a multi-site control plane — see §11.

### 2.3 6G — current status (as of September 2026)

6G is formally designated **IMT-2030** by the ITU. Status right now:

- **Stage 1 (Vision)** — done. ITU-R Recommendation M.2160 (June 2023) defines 6 usage scenarios and 15 target capabilities.
- **Stage 2 (Technical performance requirements)** — ITU-R Working Party 5D reached agreement on the draft requirements in February 2026; formal approval is expected when the parent Study Group 5 meets in December 2026.
- **Stage 3 (Full specifications)** — targeted for completion by 2030. Candidate radio technology submissions are expected to open around February 2027, with a final submission deadline before WP5D's February 2029 meeting.
- **3GPP** is running 5G-Advanced and 6G study items concurrently under **Release 20**, with **Release 21** expected to deliver the first formal 6G specifications for submission to the ITU-R.
- Realistic commercial deployment window: **2028–2030**.

None of this is operationally relevant to your 2026 build, but it tells you 6G's headline features (AI-native networking, integrated sensing and communication, sub-THz spectrum, extreme reliability) are still design targets, not deployed technology — useful to know if anyone asks whether you should be "designing for 6G" (you shouldn't yet; there's nothing to design against).

---

## 3. Where Your Project Actually Sits: The Classical Channel in QKD

This is worth being explicit about, because it reframes your whole ask. Every QKD link is actually **two channels**:

1. **The quantum channel** — single photons, a physical fiber (or free-space) run, fundamentally distance-limited (practical point-to-point QKD tops out around 100–150 km on standard fiber before key rate collapses) and cannot be routed/switched the way IP traffic can without also spending a "trusted node" or a quantum repeater.
2. **The public/classical channel** — ordinary IP networking. This is what carries basis reconciliation (sifting), error correction, privacy amplification, and KMS-to-KMS key synchronization. **This is the part you're actually asking to design**, and it has no photon-related distance limit at all — it's exactly the DHCP/IP/Linux/latency problem this whole document covers.

One security detail worth knowing because it shapes how paranoid you need to be about the classical link: QKD's security proof (BB84 and derivatives) requires the classical channel to be **authenticated**, not necessarily secret. An attacker who can read the classical channel doesn't break the protocol; an attacker who can *forge messages on it* does (classic man-in-the-middle). In practice almost everyone still encrypts it too (defense in depth, and to protect metadata), but the hard requirement is authentication — typically via an information-theoretic MAC (Wegman-Carter authentication, seeded by a short pre-shared key) so the whole system doesn't quietly fall back to computational security through its "classical" side channel.

Standardized APIs you'll likely end up implementing against (and it sounds like you already are, via your ETSI GS QKD 014 work):
- **ETSI GS QKD 004** — stateful, connection-oriented API (`OPEN_CONNECT` reserves keys with QoS parameters, blocks until established).
- **ETSI GS QKD 014** — simpler, REST-based, stateless (`GET_STATUS` then `GET_KEY`), better suited to networks with lighter-weight KMS nodes and occasional key requests — this is the one most new projects (including, per the literature, yours) build against.

**Real-world validation this exact problem is being solved at national scale, right now, in India:** as of 2026, India's National Quantum Mission has demonstrated a **1,000 km QKD network** (target: 2,000 km by 2032) built by chaining ~200-km point-to-point hops, with the quantum and classical channels coexisting on the *same* fiber (10 Gbps classical traffic alongside the quantum signal), sub-4% QBER, and ~8 kbps key generation rate at metro distances. That network uses a trusted-node relay architecture — which is the mainstream approach, and the one your "trusted-node-free" TAHQEECAT design is explicitly trying to avoid (see §11.2 for what that architectural choice implies for your classical network design).

---

## 4. IP Addressing & Subnetting

### 4.1 IPv4 basics
32-bit address, written as 4 decimal octets (`192.168.1.10`). The old class A/B/C system is obsolete — everything today uses **CIDR** (Classless Inter-Domain Routing).

| CIDR | Subnet mask | Usable hosts | Typical use |
|---|---|---|---|
| /32 | 255.255.255.255 | 1 (single host route) | Loopback-style routes |
| /30 | 255.255.255.252 | 2 | Point-to-point link between two nodes |
| /29 | 255.255.255.248 | 6 | Tiny segment |
| /28 | 255.255.255.240 | 14 | **Good fit for a single-site node cluster (your 2-node phase, or one regional hub)** |
| /27 | 255.255.255.224 | 30 | A site with room to grow |
| /24 | 255.255.255.0 | 254 | Classic "one subnet per site" |
| /16 | 255.255.0.0 | 65,534 | Whole-organization private range |

For 2 → 15 nodes, you do not need to overthink this: a **/28 or /27 per physical site**, all pulled from one **/16 private block** for the whole project, gives you years of headroom and keeps routing simple.

### 4.2 Private address space (RFC 1918) — use these, not public IPs, for the node network itself
- `10.0.0.0/8` — largest block, good for a project that might grow (e.g., `10.10.0.0/16` for "quantum node network", `10.10.1.0/24` = site 1, `10.10.2.0/24` = site 2...)
- `172.16.0.0/12`
- `192.168.0.0/16` — smallest, common for LANs/labs (your current QNet5 setup likely already uses this)

Also worth knowing: `127.0.0.0/8` is loopback, `169.254.0.0/16` is link-local (auto-assigned when DHCP fails — if you ever see a `169.254.x.x` address, that node failed to get an address and is not actually on your network).

### 4.3 Public vs. private, and why you want an overlay
Nodes at different institutions across India will each sit behind their own institutional network/firewall with a private LAN address. You will not get clean public routability between them without cooperation from every site's IT department. The practical fix (detailed in §7) is a **VPN overlay**: every node gets a private overlay IP (e.g., from your own `10.10.0.0/16`) that's reachable over the VPN regardless of the messy reality of each site's actual public/NAT'd address underneath.

### 4.4 IPv6 — do you need it?
Honestly, no, not for this. IPv6 solves address *exhaustion* at Internet scale; you have at most a few dozen nodes. Standard IPv4 private space plus a VPN overlay is simpler to operate, debug, and explain to collaborators. Keep IPv6 in mind only if a specific institution's network mandates it.

---

## 5. DHCP (Dynamic Host Configuration Protocol)

### 5.1 How it works — the DORA process
1. **D**iscover — client broadcasts `DHCPDISCOVER` (it has no IP yet, so this goes to `255.255.255.255`).
2. **O**ffer — a DHCP server responds `DHCPOFFER` with a proposed IP + lease terms.
3. **R**equest — client broadcasts `DHCPREQUEST` accepting a specific offer (broadcast so any other offering servers know they weren't picked).
4. **A**cknowledge — server sends `DHCPACK`, finalizing the lease.

Leases are renewed at **T1** (50% of lease time elapsed, client asks original server directly) and, if that fails, rebound at **T2** (~87.5%, client broadcasts to any server).

### 5.2 DHCP relay
DHCP is broadcast-based, so it doesn't cross subnet/router boundaries on its own. A **relay agent** (or a router with `ip helper-address` / equivalent) forwards discover/request packets as unicast to a central DHCP server. You'd need this if you ever wanted one central DHCP server to serve multiple sites — which leads to the actual recommendation below.

### 5.3 The actual recommendation for your project: don't use DHCP for the nodes themselves
DHCP is the right tool for client devices (laptops, phones) where you don't care which exact address they get. For infrastructure nodes — your QKD nodes, KMS instances, SDN controllers — you want **static IPs**, because:
- TLS certificates, firewall rules, WireGuard peer configs, and SDN flow rules all end up keyed on IP address.
- A node silently getting a new address on lease renewal is a debugging nightmare in a security-sensitive system.
- Across a WAN with relay complexity, static config is simply more robust.

Use DHCP (if at all) only for genuinely dynamic things on your lab LAN — a laptop joining the bench network, a phone for testing. For the nodes: static IP, covered in §6.

### 5.4 Setting up a DHCP server anyway (for completeness / lab use)
Two common options on Debian-family Linux:
- **`dnsmasq`** — lightweight, does DHCP + DNS in one process, good for small lab networks.
  ```bash
  sudo apt install dnsmasq
  # /etc/dnsmasq.conf
  interface=eth0
  dhcp-range=192.168.1.100,192.168.1.200,12h
  ```
- **ISC `dhcpd` (isc-dhcp-server)** — older, more configuration options, historically the Debian/Ubuntu default.
  ```bash
  sudo apt install isc-dhcp-server
  # /etc/dhcp/dhcpd.conf
  subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 8.8.8.8;
  }
  ```
- **Kea** — ISC's modern replacement for `isc-dhcp-server` if you're starting fresh and want active upstream development.

---

## 6. Linux Network Configuration

### 6.1 The modern toolset: `ip` (iproute2), not `ifconfig`
`ifconfig`, `route`, and `netstat` (net-tools) have been effectively deprecated in favor of `iproute2` for well over a decade. Kali ships both, but use `ip`:

```bash
ip addr show                          # view all interfaces + addresses (replaces ifconfig)
ip addr add 192.168.1.10/24 dev eth0  # add an IP — TEMPORARY, lost on reboot
ip addr del 192.168.1.10/24 dev eth0  # remove it
ip link set eth0 up                   # bring interface up (replaces ifconfig eth0 up)
ip link set eth0 down
ip route show                         # view routing table (replaces route -n)
ip route add default via 192.168.1.1  # set default gateway
ip route add 10.10.2.0/24 via 192.168.1.254  # static route to a remote site's subnet
ip -s link show eth0                  # interface stats (errors, drops — useful for latency debugging)
```

### 6.2 Making it persistent — pick one per distro/setup

**NetworkManager (`nmcli`)** — what Kali Linux uses by default, and probably your simplest option since it's already there:
```bash
nmcli con show                                          # list connections
nmcli con mod "Wired connection 1" ipv4.addresses 192.168.1.10/24 \
    ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8" ipv4.method manual
nmcli con up "Wired connection 1"                       # apply
nmcli device status                                     # quick overview
```

**netplan** (Ubuntu Server/Desktop default since 17.10+) — YAML-based, good if you deploy additional nodes on Ubuntu:
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.10/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```
```bash
sudo netplan apply
```

**systemd-networkd** — lighter-weight, common on minimal/server images and containers:
```ini
# /etc/systemd/network/20-wired.network
[Match]
Name=eth0
[Network]
Address=192.168.1.10/24
Gateway=192.168.1.1
DNS=8.8.8.8
```
```bash
sudo systemctl restart systemd-networkd
```

**`/etc/network/interfaces`** (Debian ifupdown, legacy but still functional if NetworkManager is disabled):
```
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
```

### 6.3 DNS
Modern systems route DNS through `systemd-resolved`; `/etc/resolv.conf` is often a symlink to `/run/systemd/resolve/stub-resolv.conf`.
```bash
resolvectl status          # see active DNS servers per interface
resolvectl query <host>    # test resolution
```
For a small fixed set of nodes, you can skip DNS entirely and use `/etc/hosts` entries or static IPs directly — one less moving part across a WAN.

### 6.4 Basic firewalling
```bash
# ufw (Uncomplicated Firewall) — simplest
sudo ufw allow 51820/udp        # e.g. WireGuard
sudo ufw allow from 10.10.0.0/16 to any port 8000  # KMS API, only from your overlay net
sudo ufw enable

# nftables (modern successor to iptables)
sudo nft list ruleset
```
Rule of thumb for a security-sensitive network like this: default-deny, then explicitly allow only the ports your KMS/SDN controller/WireGuard actually need, scoped to your overlay subnet where possible — not `0.0.0.0/0`.

### 6.5 Diagnostics — how you'll actually verify "millisecond" claims
```bash
ping -c 20 <host>              # basic RTT, min/avg/max/mdev (mdev ≈ jitter)
mtr <host>                     # ping + traceroute combined, live, per-hop — best first tool for a WAN link
traceroute <host>              # one-shot hop-by-hop path
ss -tulnp                      # what's listening where (replaces netstat -tulnp)
iperf3 -s   /   iperf3 -c <server>   # actual achievable throughput between two nodes
tcpdump -i eth0 port 8000      # packet capture for debugging your KMS traffic
```
`mtr` in particular is what you want running during initial WAN link testing — it'll show you exactly which hop is adding latency or dropping packets, which matters a lot when you're trying to hold a tight sync budget.

---

## 7. Secure Tunnels for a WAN Node Network

Once nodes leave the LAN, you're crossing institutional networks and (likely) the public internet between sites. You need an encrypted, authenticated tunnel — both for security and to give yourself a clean, stable private address space regardless of what each site's local network looks like.

### 7.1 WireGuard — the recommended default
Modern, kernel-integrated, minimal attack surface, very low overhead (meaningfully lower CPU/latency cost than OpenVPN's userspace design) — a good fit when you're trying to protect a millisecond-scale latency budget.

```ini
# /etc/wireguard/wg0.conf on Node A
[Interface]
PrivateKey = <nodeA-private-key>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]   # Node B
PublicKey = <nodeB-public-key>
Endpoint = <nodeB-public-ip-or-ddns>:51820
AllowedIPs = 10.10.0.2/32
PersistentKeepalive = 25
```
```bash
wg genkey | tee privatekey | wg pubkey > publickey   # generate a keypair
sudo wg-quick up wg0
sudo wg show                                          # verify handshake/traffic
```
Every node gets a fixed `10.10.0.x` overlay address regardless of its real network location — this is what makes your KMS/SDN configs stable as you scale.

### 7.2 Alternatives
- **OpenVPN** — more mature/ubiquitous, higher overhead, still fine if an institution already standardizes on it.
- **IPsec** (strongSwan etc.) — the enterprise-standard choice, more complex to configure, same one used in the 5G fronthaul/backhaul security model discussed in §2.2.

### 7.3 Overlay topology options
- **Full mesh** — every node has a WireGuard peer entry for every other node. Fine up to ~15 nodes (105 peer pairs at 15 nodes, which is annoying but scriptable, not infeasible).
- **Hub-and-spoke** — every node peers only with a central relay node, which forwards. Simpler to add nodes to, but the hub becomes a bottleneck/SPOF for the classical channel (though not necessarily for the quantum channel — see §11).

For your growth path (2 → 5 → 15), starting full-mesh and only moving to hub-and-spoke if peer management becomes painful is the more flexible order — collapsing a mesh into a hub later is easy; the reverse is more disruptive.

---

## 8. Time Synchronization — the "millisecond" question

This is the section that actually answers your stated requirement ("very updated, like millisecond updates"), because the tool you pick here matters more than raw network speed.

### 8.1 NTP — good enough for most of what you'll do
Hierarchical: stratum 0 (reference clocks — GPS, atomic) → stratum 1 (servers directly attached to stratum 0) → stratum 2, etc. Typical achievable accuracy: low milliseconds on a good LAN, tens of milliseconds over a congested/long-haul WAN path.

**Use `chrony`, not the old `ntpd`** — it's the modern default on most distros now, converges faster, and handles intermittent connectivity (exactly your WAN scenario) better.
```bash
sudo apt install chrony
# /etc/chrony/chrony.conf
server 0.in.pool.ntp.org iburst
server 1.in.pool.ntp.org iburst
```
```bash
chronyc tracking    # current offset, drift
chronyc sources -v  # which reference sources it's using and how good they are
```

### 8.2 PTP (Precision Time Protocol, IEEE 1588) — sub-microsecond, but LAN-scoped
Designed for LAN-scale, hardware-timestamped sync — sub-microsecond accuracy is achievable, but it needs NIC hardware timestamping support and, for multi-hop, switches that understand PTP (boundary/transparent clocks). Over an ordinary WAN path with unaware switches/routers in between, PTP's accuracy advantage largely evaporates.
```bash
sudo apt install linuxptp
sudo ptp4l -i eth0 -m          # runs the PTP protocol against the NIC's hardware clock
sudo phc2sys -s eth0 -c CLOCK_REALTIME -m   # syncs system clock to the NIC's PTP hardware clock
```
Use this **within a site** (your current LAN-based node pair) if your NICs support hardware timestamping — you'll get far tighter sync than NTP there. Don't expect it to solve the cross-India problem by itself.

### 8.3 The actually-scalable answer for WAN-scale sync: give every site its own absolute clock
This is the point telecom and finance both learned the hard way: instead of trying to synchronize node-to-node *over the network* (where your accuracy is capped by path latency and its variability), give **each site an independent, GNSS-disciplined time reference**. A cheap GPS/GNSS receiver with a PPS (pulse-per-second) output, feeding `chrony`'s refclock support, gives every site sub-microsecond alignment to the same external time standard — without depending on network latency between sites at all. This is exactly how cell towers hit tight sync at a distance, and it's the more robust path than chasing PTP precision across a national WAN.

**Practical target for your build:** LAN-phase (your current 2 nodes) — PTP if your NICs support it, chrony otherwise, both giving you sub-millisecond-to-low-millisecond sync. WAN-phase (5–15 nodes across India) — chrony synced to multiple sources plus, once budget allows, a GPS-disciplined reference at each site. Treat "millisecond" as the realistic target once you're multi-site, not sub-millisecond — see §10 for why.

---

## 9. Application-Layer Data Sync Between Nodes

Your phrasing ("synchronous data file... millisecond updates") suggests file-sync tools — worth flagging directly: **`rsync` and similar file-sync tools are the wrong layer for this.** They're built for periodic/on-demand batch sync (seconds to minutes), not millisecond cadence, and they weren't designed with the request/response or push semantics a KMS handshake needs.

Better fits, roughly in order of how naturally they'd sit on top of what you already have (FastAPI + ETSI GS QKD 014):

- **WebSockets inside your existing FastAPI app** — simplest integration path if you want push-based updates without adding a new stack. Good first move for the 2-node phase.
- **gRPC** — HTTP/2-based, Protocol Buffers, built-in bidirectional streaming, lower overhead than plain HTTP/JSON. Worth adopting once you're coordinating several nodes and want a typed, efficient contract between them.
- **ZeroMQ** — brokerless messaging library, very low latency, PUB/SUB or REQ/REP patterns, easy to embed directly in Python/C without running a separate broker process. Good if you want to decouple this entirely from the web-framework layer.

Whichever you choose, this is functionally the layer that implements ETSI GS QKD 014's `GET_STATUS`/`GET_KEY` exchange and KMS-KMS signaling — i.e., it's very likely the layer your post-processing refactor is already building. The advice above is about *what carries it*, not about redesigning the protocol you're already targeting.

---

## 10. Physics Reality Check: Latency Budgets Across India

Light in optical fiber travels at roughly **200,000 km/s** (about 2/3 c, due to the fiber's refractive index) — call it **~5 μs per km, one-way**, before any router/switch/processing overhead. Real fiber routes also aren't straight lines (they follow physical infrastructure corridors), so actual path length typically runs 1.3–1.5× the straight-line distance.

| Route (approx.) | Straight-line | Est. fiber path | Propagation-only one-way | Propagation-only RTT | Realistic observed RTT* |
|---|---|---|---|---|---|
| Delhi ↔ Mumbai | ~1150 km | ~1500 km | ~7.5 ms | ~15 ms | ~25–40 ms |
| Delhi ↔ Kolkata | ~1300 km | ~1700 km | ~8.5 ms | ~17 ms | ~25–40 ms |
| Delhi ↔ Bengaluru | ~1740 km | ~2300 km | ~11.5 ms | ~23 ms | ~35–60 ms |
| Delhi ↔ Chennai | ~1760 km | ~2300 km | ~11.5 ms | ~23 ms | ~35–60 ms |

*"Realistic observed" accounts for routing hops, switching/queuing, and the fact that traffic rarely takes the theoretical shortest path — treat these as planning-grade estimates, not measurements. Once real links exist, `mtr`/`ping` will give you the actual numbers, and you should use those, not this table.

This lines up with 5G's own backhaul design target from §2.2 — "up to 2000 km, one-way delay >10 ms" — because it's the same physics, not a coincidence. **The honest takeaway: sub-millisecond node-to-node classical-channel latency is achievable within a building or a metro dark-fiber ring (your current phase). Once nodes are genuinely distributed across India, tens of milliseconds RTT is the physics-imposed floor, and no amount of protocol optimization on your end changes that** — the design target for your 5–15-node phase should be "consistently low tens of milliseconds with low jitter," not "milliseconds," and your post-processing protocol (block sizes, timeout values, retry logic) should be designed around that number rather than fighting it.

---

## 11. Network Topology for Scaling 2 → 15 Nodes

### 11.1 Classical-network topology options

| Topology | Links needed | Pros | Cons |
|---|---|---|---|
| Point-to-point | 1 (for 2 nodes) | Simplest, what you likely have now | Doesn't scale |
| Star / hub-and-spoke | N−1 | Easy to add nodes; simple control plane | Hub = single point of failure |
| Full mesh | N(N−1)/2 | No single point of failure; every pair has a direct path | 105 links at 15 nodes — heavy to manage without automation |
| Partial mesh / hierarchical | Varies | Balances the above; matches how real large networks (including 5G's controller/orchestrator model from §2.2) actually scale | More design work up front |

For the **classical/VPN overlay**, full mesh is genuinely fine up to 15 nodes if you script the WireGuard peer config generation (it's just config, not physical infrastructure). For the **SDN control plane**, the hierarchical pattern is the better long-term fit — see 11.3.

### 11.2 Trusted-node relay vs. trusted-node-free — this changes what your classical network's job actually is
Most operational multi-hop QKD networks today (including India's current 1,000 km NQM chain, mentioned in §3) use **trusted-node relay**: since point-to-point QKD tops out around 100–150 km, longer distances are covered by chaining hops, with each intermediate node briefly holding the key in the clear to re-encrypt it for the next hop. That's a real trust assumption on every intermediate site — which is exactly what a **trusted-node-free** design (your TAHQEECAT project) is built to avoid.

The practical consequence for your classical network design: in a trusted-node relay network, the classical control plane's job is largely "coordinate key regeneration at each hop." In a trusted-node-free design, the classical channel's job shifts toward **coordinating genuinely end-to-end links** — synchronizing detector gating, timing, and basis reconciliation between the two true endpoints — even though the SDN signaling/management traffic may still physically hop through intermediate infrastructure. In other words: intermediate sites can still exist as *classical network relay points* (routers, VPN hubs) without ever becoming *trusted nodes* in the QKD sense, as long as no intermediate site ever holds key material. Keep that distinction sharp as you design your topology — "the classical packet passes through site C" and "site C is a trusted node" are not the same thing, and conflating them is the easiest way to accidentally reintroduce the trust assumption you're trying to remove.

### 11.3 A phased scaling plan
- **Phase 1 (2 nodes, now):** direct point-to-point link (LAN or dedicated line), PTP or chrony for sync, single SDN controller (what you already run).
- **Phase 2 (~5 nodes):** star or partial mesh overlay via WireGuard; still one SDN controller is probably fine at this scale, but start scripting peer/config generation now so it isn't manual work later.
- **Phase 3 (10–15 nodes, multi-site):** adopt the hierarchical SDN pattern from §2.2 — per-region controllers (or per-institution, if nodes span multiple universities/labs) reporting to a central orchestrator, mirroring how 5G separates RAN/transport/core controllers under one orchestrator. Your existing Ryu/OpenFlow controller becomes one regional domain controller rather than needing to be rebuilt.

---

## 12. Security Baseline for the Classical Channel

- **Authentication is non-negotiable** (§3) — at minimum, mutual TLS with certificates issued per-node; ideally an information-theoretic MAC (Wegman-Carter) for the specific messages that carry key material, so the security of your "quantum-secure" key doesn't quietly rest on a computational assumption at the classical layer.
- **Default-deny firewalling**, scoped to your VPN overlay subnet, not the public internet (§6.4).
- **No plaintext KMS/SDN traffic on any shared segment** — this is the same logic as the paper's MACsec/IPsec recommendation for 5G fronthaul/backhaul (§2.2); your situation is architecturally the same problem (sensitive control traffic over infrastructure you don't fully control end-to-end).
- **Certificate/key rotation policy** for the classical channel's own TLS material, independent of QKD key rotation — don't let the "quantum-secure" framing distract from the mundane classical PKI hygiene the system also depends on.

---

## 13. Command Cheat Sheet

```bash
# --- Viewing state ---
ip addr show                    # interfaces + IPs
ip route show                   # routing table
resolvectl status                # DNS config
nmcli device status             # NetworkManager overview
wg show                          # WireGuard peers/handshakes
chronyc tracking                 # time sync status

# --- Changing IP (temporary, testing) ---
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.1.1

# --- Changing IP (persistent, NetworkManager/Kali) ---
sudo nmcli con mod "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8
sudo nmcli con up "Wired connection 1"

# --- Diagnostics ---
ping -c 20 <host>
mtr <host>
iperf3 -c <host>
ss -tulnp
tcpdump -i eth0 port 8000

# --- WireGuard ---
wg genkey | tee privatekey | wg pubkey > publickey
sudo wg-quick up wg0
sudo wg-quick down wg0

# --- Firewall ---
sudo ufw allow 51820/udp
sudo ufw allow from 10.10.0.0/16 to any port 8000
sudo ufw status verbose
```

---

## 14. Recommended Stack for Your Setup

Given what you're already running (Kali Linux nodes/controller, Docker-based deployment, TP-Link switch, Ryu/OpenFlow SDN, FastAPI dashboard, ETSI GS QKD 014 KMS):

1. **Addressing:** one `/16` private block for the whole project (e.g. `10.10.0.0/16`), `/28` per site, static IPs on every node — no DHCP on infrastructure.
2. **WAN transport:** WireGuard overlay, full mesh initially, scripted config generation from day one so it scales without manual toil.
3. **Time sync:** chrony everywhere now; add PTP within any single site where NICs support hardware timestamping; budget for GPS-disciplined chrony refclocks per site once you're genuinely multi-city.
4. **Node-to-node data channel:** start with WebSockets in your existing FastAPI app for the 2-node phase; migrate to gRPC once you're coordinating several nodes and want typed, lower-overhead messaging.
5. **Control plane:** keep Ryu/OpenFlow as your first regional domain controller; plan the orchestrator layer now (even as a stub) so adding domain controllers later is additive, not a rearchitecture.
6. **Design constant:** treat tens-of-milliseconds RTT as your real-world target for the WAN phase (§10), and make sure your post-processing timeouts/retry logic are designed around that number rather than an LAN-scale millisecond assumption that won't survive contact with India-scale geography.

---

## 15. References

- Mehic, M. et al., "Quantum Cryptography in 5G Networks: A Comprehensive Overview," *IEEE Communications Surveys & Tutorials*, vol. 26, no. 1, First Quarter 2024. DOI: 10.1109/COMST.2023.3309051. (Source for the fronthaul/midhaul/backhaul figures in §2.2, the SDN controller/orchestrator pattern in §2.2/§11.3, ETSI 004/014 details in §3, and the trusted-node relay/key-relay concept in §11.2.)
- ITU-R, "IMT-2030: Technical requirements for the 6G future," March 2026 — [itu.int/hub](https://www.itu.int/hub/2026/03/imt-2030-technical-requirements-for-the-6g-future/)
- ITU-R Working Party 5D / IEEE ComSoc Technology Blog coverage of the IMT-2030 timeline and 3GPP Release 20/21 — [techblog.comsoc.org](https://techblog.comsoc.org/2026/01/02/roles-of-3gpp-and-itu-r-wp-5d-in-the-imt-2030-6g-standards-process/)
- India National Quantum Mission / QNu Labs 1,000 km QKD milestone coverage, April 2026 — [quantumcomputingreport.com](https://quantumcomputingreport.com/indias-national-quantum-mission-achieves-1000-km-milestone-via-qnu-labs-and-viavi-validation/), [nationalquantummission.com](https://nationalquantummission.com/)

---

*Written to sit alongside your ETSI GS QKD 014 KMS/post-processing work and your existing Ryu/OpenFlow QNet5 testbed — treat §3, §11, and §14 as the sections most specific to your actual build, and §1–§10 as the general reference underneath them.*
