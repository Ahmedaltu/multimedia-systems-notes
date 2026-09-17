# M0 Networking Primer — Study Notes


---

## 1. Network architecture

- A network = physical devices + their interconnections + the protocols they use. Sometimes called **network architecture**.
- The **Internet** is a network of many different networks, built on a **layered architecture**: the TCP/IP protocol stack.

**The stack (top to bottom), with OSI numbers:**

| Layer | OSI | Job |
|---|---|---|
| Application | L7 | Implements services (HTTP, DNS, SMTP, SIP, RTSP) |
| Transport | L4 | Deliver to correct app process, reliability if needed (TCP, UDP, SCTP) |
| Network (IP) | L3 | Get packets to the right destination machine (IP, ICMP) |
| Link / Physical | L2-L1 | Send bits over a medium, handle shared access (Ethernet, WiFi, 3GPP) |

- App layer lives in user space; transport / network / link live in the OS (kernel + device drivers).

### Hourglass model
- The stack is wide at top (many apps), wide at bottom (many media), **narrow at the waist = IP**.
- **Why an IP layer?** Global addressing and routing; isolates end-to-end protocols from network details.
- **Why a single protocol (IP)?** Maximize interoperability, minimize service interfaces. Two versions: v4, v6.
- **Why narrow?** Least-common-denominator functionality so IP runs over the maximum number of networks.

---

## 2. Application layer

### HTTP
- Application-layer protocol, runs **on top of TCP**. HTTP/1.1 dates from the late 90s.
- Original job: transmit web pages. Now also carries most video.
- **HTTP is stateless**: server keeps no memory of past client requests. Client requests each object separately. **Cookies** let a server "remember" a client.

**Methods:** GET (request object), POST (upload form data in body), HEAD (headers only, no body), PUT (upload file to URL path), DELETE (delete file at URL).

**Connections:**
- **Non-persistent**: at most one object per TCP connection. Multiple objects = multiple connections.
- **Persistent**: multiple objects over one TCP connection.

**Performance:**
- Non-persistent response time = **2 × RTT + file transmission time** (1 RTT to set up TCP, 1 RTT to request/get file).
- Persistent saves 1 RTT by leaving the connection open. ⚠️ Request **pipelining is not supported** even though HTTP/1.1 specifies it.
- Browsers allow only a few parallel connections. Hacks to speed up: **image spriting** (combine images), **domain sharding** (spread across domains to bypass the browser limit), **content inlining**.

**HTTP/2 (2015):** multiplexed prioritized streams in **one** connection (cuts TCP overhead, latency, head-of-line blocking), better header compression, server push, TLS tweaks. Google's SPDY was the predecessor.

**HTTP/3 (2021):** formerly HTTP-over-QUIC. Main change: **uses QUIC instead of TCP**. Core HTTP semantics unchanged.

### DNS
- Primary job: translate domain names to IP addresses ("phone book" of the Internet). Secondary: load balancing, failover.
- **Distributed, hierarchical database** of many name servers (centralized would not scale).
- App-layer request-response protocol, **runs over UDP** (lightweight beats reliability here).

**Server types:**
- **Root** servers: 13 logical worldwide, each replicated many times.
- **TLD** servers: per top-level domain (.com, .edu, .fi). E.g. Verisign runs .com.
- **Authoritative** servers: an org's own servers holding its hostname-to-IP mappings.
- **Local** ("default name server"): not strictly in the hierarchy, acts as a proxy forwarding queries; each ISP/company/uni has one; also caches.

**Query types:**
- **Iterated**: contacted server replies with the *name of the next server to ask* ("I don't know, but ask this one"). The local server does the legwork.
- **Recursive**: puts the burden on the contacted server to resolve fully. Heavy load at upper levels.

**Caching:** servers cache learned mappings; entries expire after a **TTL**. TLD servers usually cached locally, so root servers rarely hit. ⚠️ Cached entries can be **out of date**: if a host changes IP, it may not be known everywhere until TTLs expire. Best-effort translation.

**Load balancing:** DNS can return different IPs for one domain (e.g. by client geolocation). Central to CDNs.

---

## 3. Transport protocols

- Provide **logical communication between application processes** on different hosts. Sender breaks messages into segments; receiver reassembles.
- **TCP** (RFC 793): reliable, connection-oriented.
- **UDP** (RFC 768): best-effort, connectionless.
- Others: **QUIC** (taking over from TCP and UDP), **SCTP** (WebRTC, 4G/5G signaling).

**Transport-layer tasks:**
- Multiplexing/demultiplexing (and optionally managing connections)
- Reliable data transfer (error control over a best-effort network)
- **Flow control**: don't overload the *receiving application*
- **Congestion control**: don't overload the *network itself*

⚠️ Flow control vs congestion control is a classic trap: flow = protect the receiver, congestion = protect the network.

### Congestion control
- Goal: prevent and recover from network overload. Implemented at the **transport layer**.
- Why not elsewhere? App layer has no common method; IP layer can't fix it because overload requires reducing traffic **at the source, not at routers**.
- Not part of original TCP: introduced **1988**, 4 years after TCP's birth, after the first congestion collapse in 1986.
- **Throughput collapse**: a dropped packet wasted all the upstream capacity used to carry it, and TCP retransmits it, making things worse.

**Two approaches:**
- **End-to-end**: no explicit network feedback; congestion *inferred* from loss/delay. Traditional.
- **Network-assisted**: routers signal end systems. E.g. **ECN** packet marking (single bit).
- Principle: sender continuously watches for congestion signs and throttles (congestion → decrease rate, no congestion → increase).

### QUIC
- Introduced by Google in 2013. Hard to deploy new core protocols, so QUIC **uses UDP as a substrate** (gets through middleboxes) and is built in **user space** (fast updates) instead of the kernel.
- Replaces most of the traditional HTTPS stack: faster connection setup, **no head-of-line blocking**, modular congestion control, connection migration, better loss recovery, stream/connection-level flow control, FEC.
- **0-RTT handshake** best case (using cached info from a previous connection). Worst case combines TCP+TLS handshakes.

---

## 4. IP and routing

**Two network-layer functions:**
- **Forwarding** = move a packet from a router's input to the right output. Local, fast. This is the **data plane**.
- **Routing** = determine the route from source to destination. This is the **control plane**.

⚠️ Forwarding = data plane (local, per-router, nanoseconds). Routing = control plane (network-wide, milliseconds). Don't mix them up.

**Two ways to structure the control plane:**
- **Per-router control** (traditional): every router runs its own routing algorithm; they interact. Fully distributed.
- **SDN (Software-Defined Networking)**: a logically centralized **remote controller** computes and installs forwarding tables into routers.

**SDN benefits:** easier management (fewer misconfigs), centralized "programming" of routers, open implementation fosters innovation. Both approaches can combine (run OSPF and configure via SDN).

**Routing algorithms (two axes):**
- Global vs decentralized info: **link-state** (all routers know full topology) vs **distance-vector** (iterative, exchange info with neighbors).
- Static vs dynamic (how fast routes change).

**Intra- vs inter-domain:**
- Internet = many **autonomous systems (AS)** / "domains".
- **Intra**-domain: routers inside one AS (e.g. **OSPF**, shortest path).
- **Inter**-domain: gateways between ASes (**BGP**, policies matter).

**Data plane detail:** router looks up the output port in a forwarding table using header fields; goal is line-speed processing. **Input queuing** if packets arrive faster than the fabric handles. **Output queuing** if packets arrive from the fabric faster than the link sends. Buffering can drop packets (drop policy decides what to drop; scheduling decides transmission order).

**IP protocol:** core of the network layer. Defines datagram format, addressing, packet-handling. **ICMP** (error reporting, router signaling) is also network-layer, sometimes seen as part of IP.
- **IPv4** first major version; **IPv6 introduced 1995** to solve address exhaustion. Slow adoption (core protocol, NATs).

---

## 5. MAC and wireless networks (FYI, less relevant per slides)

**Wireless network scales (small to large):** WPAN (802.15, Bluetooth/ZigBee) < WLAN (802.11, Wi-Fi) < WMAN (802.16, WiMax) < WWAN (802.20, 4G/5G). Differ by range, data rate, power, frequency band, licensing (Wi-Fi unlicensed; cellular licensed).

**MAC = sub-layer of the link layer.** Multiple-access protocols coordinate who uses the shared channel and when. Ideal: 1 node gets all bandwidth, M nodes each get 1/M.

**Three MAC approaches:**
- **Channel partitioning**: split into time slots / frequency sub-bands / CDMA codes. E.g. cellular, Wi-Fi 6+.
- **Random access**: don't partition, allow collisions and recover. E.g. Wi-Fi's CSMA/CA (before Wi-Fi 6).
- **Taking turns**: pass a token. Not really used in wireless.

**Wi-Fi:** 802.11 is the WLAN standard (maintained by IEEE, defines PHY + MAC). **Wi-Fi is a certification mark** by the Wi-Fi Alliance ensuring products based on 802.11 interoperate.

**Cellular (4G/5G):** radio access network (RAN) + packet core. 5G core can run cloud-native (virtualized functions, network slicing). Link layer provides reliable transmission (so transport doesn't worry about link errors). Radio scheduling is centrally controlled by eNB/gNB per subframe (~1 ms); scheduler not standardized (proprietary, often proportional-fair). Wireless performance = latency + throughput, depends on load, link quality, cross traffic.

---

## Quick self-check before the test
- Name the 4 TCP/IP layers and their OSI numbers. What sits at the hourglass waist and why?
- Why is HTTP stateless, and how do cookies work around it?
- Non-persistent HTTP response time formula?
- HTTP/1.1 vs /2 vs /3 in one line each?
- DNS: iterated vs recursive; what is a TTL?
- Flow control vs congestion control?
- Forwarding vs routing / data plane vs control plane?
- Link-state vs distance-vector; OSPF vs BGP (intra vs inter)?
- What does QUIC run over and why?
