# Computer Networking — Senior Engineer Interview Questions & Model Answers

---

## SECTION 1: Fundamentals & The OSI Model

---

**Q1. Walk me through the OSI model. Why does it matter in practice, and where do most engineers misplace their mental models?**

**Model Answer:**

The OSI model has 7 layers: Physical, Data Link, Network, Transport, Session, Presentation, and Application. Most engineers can recite these, but the real value of the model is in *fault isolation* — when something breaks, the OSI model is your debugging ladder.

Where engineers go wrong is conflating the OSI model with the TCP/IP model. TCP/IP collapses OSI into 4 layers: Network Access (OSI 1+2), Internet (OSI 3), Transport (OSI 4), and Application (OSI 5–7). In practice, when you're debugging a production issue, you think in TCP/IP but you reason in OSI. For example:

- If packets aren't arriving at all → Layer 1/2 problem (bad cable, wrong VLAN, NIC issue).
- If routing is broken → Layer 3 (wrong gateway, BGP flap, ECMP misconfiguration).
- If connections are being established but data is garbled → Layer 4 or Layer 6 (TLS misconfiguration, MTU mismatch).
- If your API is returning 502s → Layer 7 (load balancer misconfiguration, upstream timeout).

The most dangerous mental model mistake engineers make is assuming a problem is at Layer 7 because they see it at Layer 7. A flapping NIC at Layer 1 will manifest as intermittent 5xx errors at Layer 7. You always start from the bottom and work up.

---

**Q2. What is the difference between a hub, a switch, and a router? This seems basic — so tell me something non-obvious about each.**

**Model Answer:**

At a surface level: hubs broadcast everything (Layer 1), switches forward based on MAC addresses (Layer 2), and routers forward based on IP addresses (Layer 3).

The non-obvious parts:

**Hubs:** They create a single collision domain. Every device on a hub shares bandwidth. This is why CSMA/CD was essential in early Ethernet — devices had to detect collisions and back off. Hubs are almost entirely gone, but the *concept* matters because WiFi is essentially a hub at the RF level. All devices on a WiFi channel share the medium and must contend for it. Understanding hub behavior helps you reason about WiFi performance degradation in dense environments.

**Switches:** They learn MAC addresses by inspecting incoming frames and building a CAM (Content Addressable Memory) table. The non-obvious detail is that a switch *floods* frames for unknown MACs — it behaves like a hub until it learns. This is relevant for security: a MAC flooding attack (like with `macof`) deliberately overflows the CAM table, forcing the switch into hub mode and enabling sniffing. Also, switches can create separate *broadcast domains* only with VLANs — otherwise all ports share one broadcast domain, which at scale causes serious issues (e.g., ARP storms).

**Routers:** They maintain routing tables and make hop-by-hop forwarding decisions. The non-obvious detail is that routers *decrement TTL* and *recompute checksums* at each hop — they are actively modifying the packet, not just forwarding it. Also, modern "routers" in cloud environments (like AWS VPC route tables) are not physical devices at all — they are software-defined policy engines. Understanding this distinction is crucial for reasoning about latency and failure modes in cloud networking.

---

**Q3. Explain what happens — in precise technical detail — when you type `https://www.google.com` in your browser and press Enter.**

**Model Answer:**

This is the classic question. The key is depth and precision at every step.

1. **URL Parsing:** The browser parses the URL. Scheme is `https`, host is `www.google.com`, path is `/`. It checks its HSTS preload list to confirm HTTPS must be used.

2. **DNS Resolution:**
   - Browser checks its own DNS cache.
   - Checks the OS DNS cache (`/etc/hosts`, then the system resolver).
   - If uncached, sends a recursive DNS query to the configured resolver (e.g., 8.8.8.8).
   - The resolver performs iterative resolution: asks a root nameserver → TLD nameserver (`.com`) → Google's authoritative nameserver → gets back an A or AAAA record for `www.google.com`.
   - Result is cached per its TTL.

3. **TCP Connection:**
   - Browser picks an ephemeral source port (e.g., 52341) and initiates a TCP 3-way handshake to port 443 on Google's IP.
   - SYN → SYN-ACK → ACK.
   - If there's an existing QUIC/H3 connection, this may be skipped entirely.

4. **TLS Handshake (TLS 1.3):**
   - Client sends `ClientHello` with supported cipher suites, TLS version, and a `key_share` (ephemeral Diffie-Hellman public key).
   - Server responds with `ServerHello`, selects cipher, sends its own `key_share`, then immediately sends its certificate and `Finished` message (1-RTT).
   - Client verifies the certificate against trusted CAs, derives session keys, sends its `Finished`.
   - With TLS 1.3, the handshake is 1-RTT (vs 2-RTT in TLS 1.2). 0-RTT is possible for resumption but has replay attack implications.

5. **HTTP Request:**
   - Browser sends an HTTP/2 (or HTTP/3) GET request. With HTTP/2, this is a binary-framed HEADERS frame over the TLS-encrypted TCP connection.
   - With HTTP/3 (QUIC), the entire transport is UDP-based with built-in encryption, eliminating TCP head-of-line blocking.

6. **Server Processing & Response:**
   - Google's servers (likely behind Anycast routing, so you hit the geographically nearest POP) receive the request, process it, and return an HTTP 200 response with HTML.

7. **Rendering:**
   - Browser parses HTML, discovers subresources (CSS, JS, images), issues additional requests (potentially using HTTP/2 multiplexing to parallelize them).
   - JavaScript is parsed and executed, potentially triggering more network requests (AJAX, fetch).

Key depth signals: mentioning HSTS, TLS 1.3 specifics, 1-RTT vs 2-RTT, QUIC, Anycast, and HTTP/2 multiplexing. Most candidates stop at "DNS then TCP then HTTP." The best candidates understand why each step exists and what the failure modes are at each.

---

## SECTION 2: TCP/IP Deep Dive

---

**Q4. Explain TCP's congestion control mechanisms. What are the four algorithms and how do they interact?**

**Model Answer:**

TCP congestion control has four interrelated algorithms defined in RFC 5681:

**1. Slow Start:**
Despite the name, this is aggressive. The congestion window (`cwnd`) starts at 1 MSS (or 10 MSS per RFC 6928) and *doubles* every RTT until it reaches `ssthresh` (slow start threshold) or a loss is detected. So it's exponential growth. The reason it starts "slow" is historical — before this, TCP would blast packets and overwhelm networks.

**2. Congestion Avoidance:**
Once `cwnd >= ssthresh`, growth slows to additive increase: `cwnd` grows by 1 MSS per RTT (linear). This is the AIMD (Additive Increase, Multiplicative Decrease) behavior. You incrementally probe for more bandwidth.

**3. Fast Retransmit:**
Instead of waiting for a retransmit timeout (RTO), when the sender receives 3 duplicate ACKs, it infers a packet was lost and immediately retransmits the missing segment without waiting for the timer. This is much faster than waiting for RTO, which can be hundreds of milliseconds.

**4. Fast Recovery:**
After fast retransmit, instead of dropping back to slow start (which was the original behavior in Tahoe), Reno halves `cwnd` and sets `ssthresh = cwnd/2`, then enters congestion avoidance directly. This avoids the brutal throughput collapse that Tahoe caused.

**Modern variants:**

- **CUBIC** (default in Linux): Uses a cubic function of time since last congestion event to set `cwnd`. More aggressive on high-bandwidth/high-latency (BDP) links. It's the default in Linux.
- **BBR (Bottleneck Bandwidth and RTT):** Google's algorithm. Instead of inferring congestion from loss, BBR models the network path's bandwidth and RTT directly. It's particularly effective on lossy links (like mobile or satellite) where packet loss doesn't necessarily indicate congestion. BBR is now used extensively in Google's infrastructure.

**The interaction matters in practice:** On a link with 1% random loss (like LTE), CUBIC interprets every loss as congestion and backs off, severely reducing throughput. BBR ignores loss-as-a-signal and maintains near-wire-speed throughput. This is why configuring the correct congestion control algorithm for your deployment environment is non-trivial.

---

**Q5. What is the TCP 3-way handshake, and what are the security and performance implications of each step?**

**Model Answer:**

The 3-way handshake:
1. Client sends **SYN** (with ISN — Initial Sequence Number, randomly chosen to prevent hijacking).
2. Server responds with **SYN-ACK** (with its own ISN, acknowledges client's ISN+1).
3. Client sends **ACK**.

**Security implications:**

- **SYN Flood Attack:** An attacker sends a massive volume of SYNs without completing the handshake. The server allocates state for each half-open connection in its SYN backlog (limited queue), exhausting resources. The mitigation is **SYN Cookies**: instead of allocating state on SYN, the server encodes connection state in the ISN of the SYN-ACK. If the final ACK arrives with the correct cookie, the connection is established. No state is held until the ACK arrives, making the server stateless during the handshake.

- **ISN Prediction:** Early TCP stacks used predictable ISNs, enabling TCP session hijacking. Modern stacks use cryptographically random ISNs.

- **TCP Reset Attack:** Since RST only requires a valid sequence number (within the window), an off-path attacker who can guess the sequence number can tear down a connection. This is why BGP sessions between routers use MD5 authentication (RFC 2385) or TCP-AO (RFC 5925).

**Performance implications:**

- The handshake costs 1 RTT before any application data can be sent. On a 100ms RTT path, that's 100ms wasted before the first byte.
- **TCP Fast Open (TFO)** addresses this: on a repeat connection, the client can include data in the SYN itself using a TFO cookie. The server can process the data immediately, achieving 0-RTT for repeat connections. However, TFO has replay attack concerns (the data in SYN can be replayed by the network), which limits its use to idempotent operations.
- QUIC eliminates the TCP handshake entirely since it's UDP-based with built-in TLS, achieving 0-RTT or 1-RTT establishment.

---

**Q6. Explain TCP's receive window, the bandwidth-delay product, and why they matter for throughput optimization.**

**Model Answer:**

**Receive Window (`rwnd`):** The amount of unacknowledged data the receiver is willing to hold in its buffer. The sender cannot have more than `min(cwnd, rwnd)` unacknowledged bytes in flight at any time.

**Bandwidth-Delay Product (BDP):** `BDP = bandwidth × RTT`. This is the number of bytes "in flight" in the network at any given time to fully utilize the pipe.

Example: A 1 Gbps link with 100ms RTT:
`BDP = 1,000,000,000 bits/s × 0.1s / 8 = 12.5 MB`

This means you need 12.5 MB of data in flight simultaneously to saturate that link. If `rwnd` is the default TCP receive buffer of, say, 4 MB, you can only achieve: `4 MB / 0.1s = 40 MB/s = 320 Mbps` — you've wasted 68% of your available bandwidth purely due to buffer sizing.

**The fix:**
- **TCP Window Scaling (RFC 1323):** Extends the window size field from 16 bits (max 65535 bytes) to 30 bits (max ~1 GB) using a scaling factor negotiated during the handshake.
- Increase `net.core.rmem_max` and `net.ipv4.tcp_rmem` in Linux.
- Modern Linux uses **auto-tuning** — the kernel dynamically adjusts socket buffer sizes based on the BDP it measures. This is why `net.ipv4.tcp_moderate_rcvbuf = 1` is the default.

**In practice:** This matters enormously for any high-throughput, high-latency link — cross-continental data transfers, satellite links, cloud storage sync. A misconfigured TCP buffer can cap your throughput at a fraction of available bandwidth despite having no congestion. When benchmarking network throughput with `iperf3`, one of the first things to check is whether buffer sizes are the bottleneck.

---

**Q7. What is TIME_WAIT and why does it exist? What happens when it becomes a problem in production?**

**Model Answer:**

**TIME_WAIT** is the state an active closer (the side that sent the first FIN) enters after completing the 4-way FIN handshake. It lasts `2 × MSL` (Maximum Segment Lifetime) — typically 60 seconds on Linux (MSL = 30s).

**Why it exists — two reasons:**

1. **Reliable last ACK delivery:** The last ACK the active closer sends could be lost. If the passive closer retransmits its FIN, the active closer needs to still be alive to re-send the ACK. If it had already closed, it would send a RST, which could confuse the remote side.

2. **Preventing old segments from contaminating new connections:** TCP identifies connections by a 4-tuple (src IP, src port, dst IP, dst port). If a new connection reuses the same 4-tuple immediately, a delayed packet from the old connection could be misinterpreted as data for the new connection. TIME_WAIT's 2×MSL wait ensures all such packets have expired from the network.

**Production problems:**

In high-throughput services (e.g., an HTTP/1.1 service with frequent short-lived connections), the server can accumulate hundreds of thousands of TIME_WAIT sockets, exhausting the ephemeral port range (16,384 to 60,999 by default on Linux). You can see this with `ss -s`.

**Mitigations:**

- **`SO_REUSEADDR`:** Allows binding to a port with a socket in TIME_WAIT. Safe for servers.
- **`tcp_tw_reuse`:** Allows reusing TIME_WAIT sockets for *outbound* connections if the new connection's timestamp is newer (requires TCP timestamps to be enabled). This is generally safe and recommended.
- **`tcp_tw_recycle`:** Was available in older kernels but caused issues with NAT (multiple clients behind the same IP with different timestamps) and was removed in Linux 4.12.
- **The real fix:** Use persistent connections (HTTP keep-alive, connection pooling) to avoid the constant creation and teardown of TCP connections. This is why connection pools in databases, HTTP/2, and gRPC are valuable — not just for latency but for reducing ephemeral port exhaustion.

---

## SECTION 3: DNS

---

**Q8. Explain the full DNS resolution process from a recursive resolver's perspective, including all the actors involved.**

**Model Answer:**

A full iterative resolution from a recursive resolver:

1. Client queries its configured recursive resolver (e.g., `8.8.8.8`) for `api.example.com` type A.
2. The resolver checks its cache. Cache miss.
3. The resolver queries one of the 13 root nameserver clusters (e.g., `a.root-servers.net`). It asks: "Who is authoritative for `api.example.com`?" The root server doesn't know, but it responds with the NS records for `.com` (the TLD).
4. The resolver queries a `.com` TLD nameserver (e.g., `a.gtld-servers.net`). It asks the same question. The TLD server responds with the NS records for `example.com` (the authoritative nameserver for the domain).
5. The resolver queries `example.com`'s authoritative nameserver (e.g., `ns1.example.com`). It returns the A record: `api.example.com → 93.184.216.34`.
6. The resolver caches the result per its TTL and returns it to the client.

**Key nuances:**

- **Glue records:** If `ns1.example.com` is the nameserver for `example.com`, and you need to resolve `ns1.example.com` to query it, you have a chicken-and-egg problem. This is solved by "glue records" — the TLD nameserver includes the A record for `ns1.example.com` in the additional section of its response, so the resolver doesn't need a separate lookup.

- **Negative caching (RFC 2308):** NXDOMAIN responses are also cached for the SOA record's `minimum` TTL. This prevents repeated lookups for non-existent domains.

- **DNSSEC:** The authoritative server signs its records with a private key. The resolver validates the signature chain up to a trusted anchor (the root DNSKEY). This prevents DNS cache poisoning (Kaminsky attack). The Kaminsky attack exploited the fact that DNS uses UDP with predictable transaction IDs — an attacker could flood forged responses to poison a resolver's cache.

- **DNS over HTTPS (DoH) / DNS over TLS (DoT):** These encrypt DNS queries to prevent eavesdropping and manipulation by ISPs or on-path attackers. DoH sends DNS queries inside HTTP/2 to port 443, making it nearly indistinguishable from regular web traffic.

---

**Q9. What is Anycast and how does DNS use it? What are the failure modes?**

**Model Answer:**

**Anycast** is a routing technique where the same IP address is announced from multiple geographic locations simultaneously. BGP routers on the internet route packets to the *topologically nearest* instance of that IP based on BGP path selection (typically shortest AS path).

**DNS use case:** Root nameservers use Anycast extensively. The 13 logical root server addresses (e.g., `192.5.5.241` for `f.root-servers.net`) are actually served from hundreds of physical nodes worldwide. When your resolver queries `192.5.5.241`, BGP routes that packet to the nearest Anycast node — not necessarily a specific machine. This provides:
- Low latency (geographically close node)
- DDoS resilience (attack traffic is distributed across all nodes)
- Automatic failover (if a node goes down, BGP withdraws its route and traffic flows to the next nearest)

**Failure modes:**

1. **BGP route oscillation:** If a node is flapping (repeatedly going up and down), BGP might oscillate, causing DNS resolution instability. BGP route damping can mitigate this.

2. **State is not shared across Anycast nodes:** If you have any stateful protocol or session-based behavior, Anycast breaks it. A connection that starts at Node A might suddenly route to Node B mid-session if routing changes. For DNS (stateless UDP), this is fine. For TCP-based Anycast (used in some CDNs), you need careful session affinity management.

3. **Asymmetric routing:** The path from client to Anycast node and back might traverse different physical paths, causing issues for stateful firewalls and NAT.

4. **BGP convergence time:** When a node fails, BGP reconvergence can take seconds to minutes depending on timer configurations. During that window, some clients may experience DNS failures.

5. **Anycast doesn't guarantee the "nearest" node is the best:** BGP is policy-driven and shortest AS path doesn't mean lowest latency. A resolver in Asia might sometimes route to a European node due to AS topology.

---

## SECTION 4: HTTP & Application Layer

---

**Q10. Compare HTTP/1.1, HTTP/2, and HTTP/3. What problems does each version solve and what new problems does it introduce?**

**Model Answer:**

**HTTP/1.1:**

Solved: Persistent connections (`Connection: keep-alive`), chunked transfer encoding, host headers (enabling virtual hosting).

Problems:
- **Head-of-line (HOL) blocking at the application layer:** Requests on a single connection are processed in order (pipelining is unreliable and disabled by default in most clients). A slow response blocks all subsequent requests.
- **Workaround:** Browsers open 6–8 parallel TCP connections per origin to parallelize requests — which is wasteful (6–8× TLS handshake overhead, multiple slow starts).
- **Header bloat:** HTTP headers are sent as plain text on every request, with no compression. A cookie-heavy request might have 2KB of headers per request.

**HTTP/2:**

Solved:
- **Multiplexing:** Multiple streams are interleaved on a *single* TCP connection using binary framing. Each stream has an ID; frames are tagged with stream IDs and reassembled at the receiver. This eliminates application-layer HOL blocking.
- **Header compression (HPACK):** Uses static and dynamic tables to compress headers. Reduces header overhead from KBs to tens of bytes for repeat requests.
- **Server push:** Server can proactively send resources the client will need (though this has largely been abandoned in practice due to poor cache interaction).
- **Stream prioritization:** Clients can indicate which streams are more important.

Problems:
- **TCP HOL blocking remains:** HTTP/2 runs over TCP. TCP ensures in-order delivery of the entire byte stream. If one packet is lost, *all* HTTP/2 streams stall waiting for retransmission — even streams that don't contain the lost packet. You've solved the application HOL problem but the transport HOL problem remains.
- **TLS 1.2 still requires 2 RTT, HTTP/2 requires another:** Slower connection establishment compared to what's theoretically possible.

**HTTP/3 (QUIC):**

Solved:
- **Transport HOL blocking eliminated:** QUIC is built on UDP and handles streams natively at the transport layer. Each stream has independent loss recovery — a lost packet only blocks the stream it belongs to, not others.
- **0-RTT / 1-RTT connection establishment:** QUIC combines the transport and TLS handshakes. First connection: 1 RTT. Repeat connections: 0-RTT (sending data with the first packet, using cached session credentials).
- **Connection migration:** QUIC connections are identified by a Connection ID, not by the 4-tuple (src IP, src port, dst IP, dst port). When a mobile client switches from WiFi to LTE (changing its IP), the QUIC connection persists seamlessly. TCP would break.

Problems:
- **UDP is often rate-limited:** Many middleboxes, firewalls, and ISPs rate-limit or throttle UDP traffic because it was historically associated with gaming/streaming abuse. Google reports QUIC is blocked for ~5% of connections in the wild.
- **More CPU intensive:** QUIC implements congestion control, reliability, and encryption entirely in userspace (vs. TCP in kernel). This means more CPU overhead per connection, which matters at scale.
- **Head-of-line blocking at the IP layer:** Packet reordering still exists — QUIC just handles it per-stream instead of globally.
- **Immature ecosystem:** Debugging tools (Wireshark support, etc.) are still catching up.

---

**Q11. Explain TLS 1.3. What changed from TLS 1.2 and why do the changes matter?**

**Model Answer:**

TLS 1.3 (RFC 8446) was a major redesign, not an incremental update.

**Key changes:**

**1. Handshake reduced from 2-RTT to 1-RTT:**
In TLS 1.2:
- Round 1: ClientHello → ServerHello + Certificate + ServerHelloDone
- Round 2: ClientKeyExchange + ChangeCipherSpec + Finished → Server Finished
- Data can flow after round 2 — 2 RTTs minimum.

In TLS 1.3:
- The client sends its `key_share` in the very first message (ClientHello). The server can immediately compute the shared secret, send its certificate, and start sending encrypted application data — all in one round trip. The client verifies and sends its Finished. **1-RTT.**

**2. Removed legacy, broken cryptography:**
TLS 1.3 dropped support for: RSA key exchange (not forward-secret), static DH, RC4, 3DES, MD5, SHA-1, and CBC mode cipher suites. These were the source of vulnerabilities like POODLE, BEAST, FREAK, and DROWN. TLS 1.3 mandates only:
- Key exchange: ECDHE (ephemeral) — ensures forward secrecy
- Cipher suites: AES-GCM, ChaCha20-Poly1305 — AEAD only
- Signatures: Ed25519, RSA-PSS

**3. Forward Secrecy is mandatory:**
In TLS 1.2, RSA key exchange meant: if the server's private key was ever compromised (even years later), all past recorded sessions could be decrypted. TLS 1.3 mandates ephemeral key exchange (ECDHE), so each session uses a fresh key pair. Compromising the server key doesn't help decrypt past sessions.

**4. 0-RTT Resumption:**
For repeat connections, the client can include application data in its first flight using a pre-shared key (PSK) from the previous session. This is 0-RTT — data arrives before the handshake completes. **Trade-off:** 0-RTT data has no forward secrecy and is vulnerable to replay attacks. It should only be used for idempotent operations (e.g., HTTP GETs, not POST payments).

**5. Encrypted ServerHello extensions:**
In TLS 1.2, the certificate (and thus the server's identity) was sent in the clear. In TLS 1.3, all handshake messages after `ServerHello` are encrypted. Combined with ECH (Encrypted Client Hello), the entire handshake can be encrypted, preventing passive eavesdroppers from learning which domain you're connecting to.

---

## SECTION 5: Routing & BGP

---

**Q12. Explain how BGP works and why it's considered the "protocol that runs the internet."**

**Model Answer:**

**BGP (Border Gateway Protocol, RFC 4271)** is the exterior gateway protocol used between Autonomous Systems (ASes) — the distinct networks operated by ISPs, enterprises, cloud providers, etc. Every major network on the internet has an ASN (Autonomous System Number) and uses BGP to advertise its IP prefixes to neighbors.

**How it works:**

BGP is a path-vector protocol. Each BGP router maintains a RIB (Routing Information Base) and advertises prefixes along with a full AS_PATH — the sequence of ASNs the prefix has traversed. This path vector serves two purposes:
1. **Loop prevention:** If a router sees its own ASN in the AS_PATH of an incoming advertisement, it discards it.
2. **Policy:** Operators can make routing decisions based on the AS_PATH and other attributes.

**BGP sessions (TCP port 179):**
- **iBGP (internal BGP):** Between routers within the same AS. Used to distribute external routing information internally.
- **eBGP (external BGP):** Between routers in different ASes. Peers must typically be directly connected (TTL = 1 by default, called EBGP multihop if extended).

**Path selection (simplified, in order):**
1. Highest Local Preference (locally configured; controls outbound traffic)
2. Shortest AS_PATH
3. Lowest Origin (IGP < EGP < incomplete)
4. Lowest MED (Multi-Exit Discriminator — hints from remote AS about preferred entry point)
5. eBGP over iBGP
6. Lowest IGP metric to next hop
7. Oldest route (for stability)
8. Lowest Router ID

**Why it "runs the internet":** There is no central routing authority on the internet. BGP is the mechanism by which thousands of independently operated networks agree on how to reach each other. It's essentially a distributed, policy-driven reachability negotiation protocol.

**Famous failure modes:**
- **BGP hijacking (Pakistan Telecom, 2008):** Pakistan Telecom accidentally advertised a more-specific route for YouTube's IP prefix globally, blackholing YouTube traffic for ~2 hours. More-specific routes (longer prefix) always win.
- **BGP leak (Facebook, 2021):** Facebook accidentally withdrew its own BGP routes during a maintenance operation, making its entire infrastructure unreachable — including the tools needed to fix it (engineers had to physically go to data centers).
- **Route reflectors:** iBGP requires full-mesh between all iBGP peers (O(n²) sessions). Route reflectors solve this by acting as a hub, but introduce a single point of failure and additional complexity.

---

**Q13. What is ECMP and what are its implications for TCP flows?**

**Model Answer:**

**ECMP (Equal-Cost Multi-Path)** is a routing strategy where multiple paths of equal cost to a destination are available simultaneously. Instead of choosing one, the router can distribute traffic across all of them.

**How ECMP hashes flows:** Routers use a hash function over the 5-tuple (src IP, src port, dst IP, dst port, protocol) to consistently map a flow to a single path. This ensures all packets of the same TCP connection traverse the same path — which is critical for TCP because out-of-order delivery across paths would trigger false congestion signals.

**Implications for TCP:**

1. **Flow polarization:** If all your servers use the same few destination IPs and ports (e.g., a fleet of app servers all talking to the same database cluster), the hash space may not distribute evenly. You get some paths heavily loaded while others are idle. Randomizing source ports (which databases often do with connection pools) helps.

2. **ECMP and MLAG:** In data center designs with multi-chassis link aggregation, ECMP interacts with LAG hashing in complex ways. Poor hash design can result in asymmetric load.

3. **ECMP doesn't do load balancing in the general sense:** It balances *flows*, not bytes. A single elephant flow (e.g., a large file transfer) will saturate one path regardless of how many paths are available. Flowlet-based switching (breaking flows into short bursts and rerouting each burst) is a research-and-production-level solution to this.

4. **QUIC and ECMP:** QUIC's Connection ID enables better ECMP behavior because routers can use it for hashing, decoupling from the 5-tuple. This is important because NAT traversal can change source ports, disrupting TCP-based ECMP consistency.

---

## SECTION 6: Network Security

---

**Q14. Explain the difference between a stateful and stateless firewall. When would you choose each?**

**Model Answer:**

**Stateless firewall:** Evaluates each packet in isolation based on static rules (src/dst IP, src/dst port, protocol, flags). It has no memory of previous packets. 

Characteristics:
- Very fast — no state to look up.
- Cannot understand bidirectional flows. To allow TCP connections, you must write symmetric rules (allow inbound ACKs explicitly, not just outbound SYNs).
- Can't detect anomalies like SYN flooding, fragmentation attacks, or sequence number anomalies.
- Works at line rate for very high-throughput environments.

**Stateful firewall:** Tracks connection state in a state table. When a new TCP SYN arrives and matches a policy, an entry is created. Subsequent packets are validated against the state table (correct sequence numbers, matching 4-tuple). Return traffic is automatically permitted without explicit rules.

Characteristics:
- Understands the semantics of TCP/UDP sessions.
- Can detect and block anomalous packets (e.g., TCP RSTs from random IPs, out-of-state packets used in some DoS attacks).
- State table has finite size — a state exhaustion attack (sending millions of half-open connections) can fill the table and deny service.
- Higher latency per packet due to state lookups.

**When to choose each:**

- **Stateless:** Core internet routers doing basic ACLs, rate limiting, or filtering at 100 Gbps+. Also for simple inbound rules where you fully control both sides (e.g., internal cluster networks). AWS Security Groups are technically stateful; AWS NACLs are stateless.
- **Stateful:** Edge of enterprise networks, cloud security groups, anywhere you need fine-grained session tracking or don't want to write symmetric rules. Almost all modern "firewalls" are stateful. 

The nuance: in cloud environments (AWS, GCP), "stateful" security groups have a hidden cost — every connection consumes tracking state in a distributed firewall that has per-resource limits. High-connection-rate services (Lambda, microservices) can exhaust connection tracking tables, leading to packet drops that are invisible without inspecting VPC flow logs.

---

**Q15. What is a DDoS attack? Describe at least three distinct attack types and their specific mitigations.**

**Model Answer:**

A DDoS (Distributed Denial of Service) attack aims to exhaust a target's resources — bandwidth, CPU, memory, or connection state — using traffic from a distributed set of sources (often botnets) to make it impractical to block by IP.

**Attack type 1: Volumetric (bandwidth exhaustion)**
Example: UDP flood, DNS amplification.

DNS amplification is particularly nasty: the attacker spoofs the victim's IP as the source and sends small DNS queries (`ANY` type) to open resolvers. The resolver responds to the victim's IP with responses 50–100× larger. A 1 Gbps attacker can generate 50–100 Gbps of attack traffic at the victim.

*Mitigation:*
- **BCP38 (ingress filtering):** ISPs should drop packets with spoofed source IPs. Many don't implement this.
- **Anycast scrubbing:** Route attack traffic to scrubbing centers that absorb and filter it before forwarding legitimate traffic.
- **Rate limiting at resolver:** Limit response rate with RRL (Response Rate Limiting) on DNS servers.
- **Disable `ANY` queries.**

**Attack type 2: Protocol (state exhaustion)**
Example: SYN flood (fills server's SYN backlog), SSDP reflection, ACK flood.

*Mitigation:*
- **SYN cookies** (as described in Q5).
- **Stateless DDoS protection appliances** that absorb SYN packets and proxy verified connections to the backend.
- **SYN proxy at load balancer:** The load balancer completes the handshake on behalf of the backend and only forwards fully established connections.

**Attack type 3: Application Layer (resource exhaustion)**
Example: HTTP Slowloris (holds open thousands of slow HTTP connections), HTTP flood (legitimate-looking GET/POST requests), low-and-slow attacks.

Slowloris sends partial HTTP headers very slowly, holding server threads open indefinitely without sending the terminating `\r\n\r\n`. A single attacker can exhaust all connections on an Apache server with a few hundred sockets.

*Mitigation:*
- **Connection timeouts** for slow clients.
- **Nginx/async servers:** Event-driven servers handle thousands of slow connections without per-thread overhead.
- **WAF (Web Application Firewall):** Rate limiting per IP, behavioral analysis, CAPTCHA challenges for suspicious traffic.
- **Challenge-response:** Cloudflare's JS challenge forces clients to prove they can execute JavaScript before accessing the origin.

**Attack type 4: Algorithmic complexity**
Example: Hash collision attacks (sending keys that all hash to the same bucket, turning O(1) hashtable lookups into O(n)). ReDoS (crafted inputs that trigger catastrophic backtracking in regex engines).

*Mitigation:*
- Hash randomization (per-process seed in Python, Java).
- Regex complexity audits, timeout limits on regex execution.
- Input validation and size limits before expensive processing.

---

## SECTION 7: Load Balancing & CDN

---

**Q16. Explain the difference between L4 and L7 load balancing. When would you use each?**

**Model Answer:**

**L4 Load Balancing (Transport Layer):**
Makes routing decisions based on the TCP/UDP 4-tuple (src IP, src port, dst IP, dst port) and potentially TCP flags. It does not inspect the payload.

How it works:
- Uses NAT (DNAT) to rewrite the destination IP/port and forward packets to a backend.
- Or uses Direct Server Return (DSR): the LB rewrites only the destination MAC address; the backend responds directly to the client (the LB is not in the return path), which is extremely high throughput.

Characteristics:
- Very fast (often done in kernel with iptables/ipvs or hardware ASICs).
- No TLS termination — the backend handles TLS end-to-end.
- Cannot route based on URL path, hostname, headers, or cookies.
- Cannot do HTTP-aware health checks.
- Suitable for TCP connection pinning (all packets of a flow go to the same backend — the 5-tuple guarantees this).

**L7 Load Balancing (Application Layer):**
Terminates the TCP connection at the LB, inspects the HTTP/gRPC payload, and makes routing decisions based on URL path, hostname, headers, cookies, or request body.

Characteristics:
- Can do content-based routing: `/api/v1/*` → backend A, `/api/v2/*` → backend B.
- Can do A/B testing, canary deployments (route 5% of traffic to new version).
- TLS is terminated at the LB; certificate management is centralized.
- Can do request-level retries (not just connection-level).
- Higher latency than L4 (must parse HTTP, maintain two TCP connections: client↔LB, LB↔backend).
- Cannot handle raw TCP/UDP protocols (non-HTTP).

**When to use each:**

- **L4:** High-throughput, low-latency non-HTTP workloads (database traffic, gaming servers, video streaming). Also as a first tier in a two-tier LB architecture where L4 handles raw distribution and L7 handles application routing.
- **L7:** Web applications, microservices, anything requiring routing granularity, rate limiting, authentication, or observability at the request level. Envoy, NGINX, HAProxy, AWS ALB are all L7.

**Two-tier design:** Many large systems use L4 in front of L7. The L4 LB (e.g., Maglev, IPVS) provides horizontal scaling and fault tolerance for the L7 LB tier. The L7 LBs then handle application logic. This is how Google, Facebook, and most hyperscalers operate.

---

**Q17. What is a CDN and how does it work technically? What are the edge cases where a CDN makes things worse?**

**Model Answer:**

A CDN (Content Delivery Network) is a globally distributed network of proxy servers (PoPs — Points of Presence) that cache and serve content from locations physically close to end users, reducing latency and offloading origin infrastructure.

**Technical operation:**

1. **DNS-based routing:** The CDN operates authoritative DNS for the customer's domain. When a user queries `static.example.com`, the CDN's DNS returns the IP of the nearest PoP (using Anycast or geoDNS).

2. **Cache hit path:** The user's request arrives at the PoP. If the requested object is in the PoP's cache and not expired (TTL not exceeded), the PoP serves it directly. No request to origin.

3. **Cache miss path:** PoP doesn't have the object. It fetches from origin (or from a parent/shield node in a hierarchical CDN), caches it, and serves it.

4. **Cache key:** Typically URL + (optionally) headers like `Accept-Encoding`, `Accept-Language`. Query strings may or may not be included depending on configuration. Getting cache keys wrong is a major source of both cache poisoning and poor hit rates.

5. **Origin shield:** A single "shield" PoP acts as an intermediate cache between all edge PoPs and origin. Instead of 200 edge PoPs all hitting origin on a cache miss, only the shield hits origin. This dramatically reduces origin load for low-traffic content.

**Where CDNs make things worse:**

1. **Dynamic, uncacheable content:** If your content has `Cache-Control: no-cache` or is personalized (user-specific), the CDN adds latency because every request still goes to origin, plus you now have an extra hop through the CDN PoP.

2. **Cache poisoning via query strings or headers:** If your CDN doesn't normalize cache keys properly, an attacker can poison your cache by sending a request with a crafted `Host` header or query parameter that results in malicious content being cached and served to legitimate users.

3. **Stale content during deploys:** If your CDN cache TTLs are long (e.g., 1 hour), deploying a new version of your application doesn't immediately propagate to users. You need to either use short TTLs (hurts hit rate), use cache busting (hashed filenames), or issue a cache purge API call.

4. **Latency for cache misses with long origin RTT:** If your origin is in `us-east-1` and your CDN PoP is in Tokyo, a cache miss from a Tokyo user routes to the PoP (5ms) then all the way to `us-east-1` (~150ms) and back. Total RTT: ~310ms + origin processing. Without the CDN, the user might have gone directly to a regional API endpoint.

5. **TLS termination at the PoP:** The CDN decrypts and re-encrypts traffic, acting as a man-in-the-middle. For security-sensitive applications, this means you must trust your CDN provider completely, as they can see all unencrypted traffic.

---

## SECTION 8: Network Programming

---

**Q18. Explain the difference between `select`, `poll`, `epoll`, and `io_uring`. Why does this matter for writing high-performance network servers?**

**Model Answer:**

All four are mechanisms for monitoring multiple file descriptors (sockets) for I/O readiness — the core of any event-driven network server.

**`select` (POSIX):**
- Takes three `fd_set` bitmasks (read, write, error) and a timeout.
- Scans linearly through all FDs up to `FD_SETSIZE` (typically 1024) on every call.
- **O(n) per call** — scales terribly with many connections.
- Modifies the fd_set in place, so you must reset it on every iteration.
- Limited to 1024 FDs by default.

**`poll` (POSIX):**
- Takes an array of `pollfd` structs.
- No FD_SETSIZE limit (theoretically unlimited).
- Still **O(n) per call** — scans the entire array every time.
- Doesn't modify the input array, which is slightly cleaner.

**`epoll` (Linux):**
- Uses the kernel to maintain an interest list. You register FDs with `epoll_ctl` (O(1)). When you call `epoll_wait`, the kernel returns only the FDs that are ready — no scanning.
- **O(1) for event detection** (O(k) where k is the number of ready events, not total monitored FDs).
- Supports **edge-triggered (ET)** and **level-triggered (LT)** modes:
  - Level-triggered: epoll_wait returns as long as the FD is readable. Simpler but may cause busy-looping.
  - Edge-triggered: epoll_wait returns only when the state *changes* (new data arrives). More efficient but requires draining the FD completely before returning (use non-blocking sockets + loop until EAGAIN).
- `epoll` is the foundation of Node.js (via libuv), Nginx, Redis, and most modern high-performance servers.

**`io_uring` (Linux 5.1+):**
- Fundamentally different paradigm: **asynchronous I/O submission and completion**, not just readiness notification.
- Uses two ring buffers in shared memory between userspace and kernel: the submission queue (SQ) and completion queue (CQ).
- You submit I/O operations (read, write, accept, send, recv) to the SQ. The kernel processes them asynchronously. Results appear in the CQ.
- **Zero syscall overhead for batches:** You can submit 100 I/O operations and check completions with a single syscall (or even zero syscalls with `IORING_SETUP_SQPOLL` where a kernel thread polls the SQ).
- Supports `IOSQE_FIXED_FILE` and `IOSQE_FIXED_BUFFER` to pre-register FDs and buffers, eliminating per-operation overhead.
- Enables **true async I/O** for disk I/O, not just network I/O — epoll doesn't work with regular files.
- Still maturing; security hardening is ongoing (several privilege escalation CVEs in early versions).

**Why it matters:**

The C10K problem (handling 10,000 concurrent connections) was solved by moving from select/poll to epoll. Modern servers routinely handle 1M+ concurrent connections with epoll. io_uring is solving the C10M problem and eliminating the last syscall overhead bottlenecks in ultra-high-throughput applications. Understanding these primitives helps you reason about why Nginx is faster than Apache (event-driven vs. process-per-connection), why Node.js uses libuv, and where to invest in optimization.

---

**Q19. What is a socket, technically? Walk me through the kernel data structures involved in a TCP connection.**

**Model Answer:**

A socket is a kernel abstraction representing one endpoint of a communication channel. In Linux, it's a file descriptor backed by kernel structures.

**Key kernel structures:**

**`struct socket`:** The high-level VFS-facing structure. It contains:
- `socket_state state`: SS_UNCONNECTED, SS_CONNECTED, etc.
- `struct sock *sk`: Pointer to the protocol-specific socket.
- `struct proto_ops *ops`: Function pointers for socket operations (connect, accept, sendmsg, etc.).

**`struct sock` (the real thing):**
This is the massive (~2KB) protocol-independent socket structure. Key fields:
- `sk_rcvbuf` / `sk_sndbuf`: Receive and send buffer sizes.
- `sk_receive_queue`: The receive buffer — a linked list of `sk_buff` (skb) structures containing received data.
- `sk_write_queue`: Data waiting to be sent.
- `sk_sleep`: Wait queue for blocking I/O.
- `sk_state`: TCP state machine (TCP_ESTABLISHED, TCP_LISTEN, TCP_TIME_WAIT, etc.).

**`struct tcp_sock`:** Extends `struct sock` with TCP-specific state:
- `snd_una`, `snd_nxt`: Oldest unacknowledged byte, next byte to send.
- `rcv_nxt`: Next expected byte from the receiver.
- `snd_cwnd`: Congestion window.
- `rcv_wnd`: Receive window advertised to the peer.
- `write_seq`: Next sequence number for new data.

**`struct sk_buff` (skb):** The fundamental unit of network data in the Linux kernel. Represents a single packet or fragment. Contains:
- Pointer to data buffer.
- Pointers for network/transport headers within the buffer.
- Metadata (timestamps, protocol, interface).

SKBs travel up and down the network stack. Each layer adds/removes header information without copying data — the header pointer is just adjusted.

**Connection lifecycle in the kernel:**

1. `socket()` syscall creates `struct socket` + `struct sock` + binds to VFS (returns fd).
2. `connect()` initiates TCP handshake. SYN is placed in a `sk_buff`, sent via `tcp_connect()`.
3. On SYN-ACK receipt, `tcp_rcv_synsent_state_process()` handles it, sends ACK, transitions to ESTABLISHED.
4. `read()`/`recv()` copies data from `sk_receive_queue` to userspace buffer.
5. `write()`/`send()` copies data to `sk_write_queue`, TCP decides when to actually send based on Nagle algorithm and cwnd.

This depth of understanding matters when you're doing things like: tuning SO_RCVBUF/SO_SNDBUF sizes, implementing zero-copy networking with `sendfile()` or `splice()`, or debugging kernel network stack performance with tools like `perf` and `ftrace`.

---

## SECTION 9: Advanced & System Design Scenarios

---

**Q20. You're building a globally distributed messaging service that needs sub-100ms latency for 99th percentile. Walk me through the networking architecture.**

**Model Answer:**

This is a system design + networking hybrid question. The key is justifying every networking decision with latency and reliability reasoning.

**Global topology:**

Deploy in 5–8 regions covering major latency zones (US East, US West, EU West, EU Central, Asia Pacific, South Asia, South America). Use a cloud provider's backbone (AWS Global Accelerator, GCP Premium Tier) rather than the public internet for inter-region traffic — this avoids unpredictable BGP routing and typically cuts latency by 30–50% for cross-region paths.

**Edge acceleration:**

Use Anycast to route clients to the nearest PoP. The initial TCP+TLS handshake happens at the edge PoP, eliminating round-trips to the origin region. With TLS 1.3 at the edge and QUIC for mobile clients, we get 1-RTT connection establishment.

**Protocol choices:**

- **Client ↔ Edge:** WebSocket over QUIC (HTTP/3 upgrade). QUIC eliminates TCP HOL blocking and handles mobile network switching gracefully via Connection IDs. Persistent WebSocket means no polling overhead.
- **Edge ↔ Origin:** gRPC over HTTP/2 with persistent connection pools. gRPC gives us bidirectional streaming, multiplexing, and protobuf serialization efficiency.
- **Inter-service:** gRPC or raw TCP with connection reuse.

**Latency budget breakdown (targeting P99 < 100ms):**

- DNS resolution: ~5ms (cached after first lookup)
- TCP + TLS handshake: 1 RTT to nearest PoP (~5–15ms for most users), 1-RTT TLS 1.3
- Request transmission: ~1ms (small message)
- Edge PoP processing: ~1ms
- Edge → origin (if needed): backbone, ~20–60ms depending on region pair
- Origin processing: ~5–10ms
- Response path: mirrors request path

For same-region communication, this is easily sub-100ms. For cross-region, we need to architect around minimizing origin involvement. The key technique: **fan-out at the edge**. When Alice in Tokyo sends a message to Bob in London, the message is written to a globally distributed data store (e.g., DynamoDB Global Tables, Spanner) and Alice's edge PoP immediately sends a delivery notification to Bob's edge PoP via the backbone. Bob's client, already connected via WebSocket to the EU PoP, receives the push delivery with minimal delay.

**Reliability considerations:**

- Implement TCP keepalives and WebSocket pings to detect dead connections quickly.
- Exponential backoff with jitter for reconnection (avoid thundering herd after a PoP outage).
- Circuit breakers between edge and origin to prevent cascade failures.
- BFD (Bidirectional Forwarding Detection) for sub-second link failure detection on backbone paths.

---

**Q21. A service that was working fine suddenly shows massively increased p99 latency with no obvious increase in traffic. How do you diagnose this from a networking perspective?**

**Model Answer:**

This is a systematic debugging question. The goal is to demonstrate a methodical, layer-by-layer approach.

**Step 1: Characterize the problem**
- Is the latency increase network-bound or compute-bound? `top`, `iostat`, CPU profiler. If CPU/IO is saturated, this isn't a pure networking problem.
- What endpoints are affected? All or a subset? Localized to a specific region/AZ?
- What does the latency distribution look like? Is it bimodal (some requests fast, some very slow) or uniformly elevated? Bimodal often suggests a specific failure — timeouts, retries, or a slow backend instance.

**Step 2: Check connection-level metrics**
- `ss -s`: Look for unusual TIME_WAIT accumulation, retransmit counters.
- `netstat -s | grep -i retransmit`: Retransmit rate. Even 1% retransmit rate causes significant p99 spikes because retransmits trigger RTO waits (default initial RTO is 200ms in Linux).
- `cat /proc/net/sockstat`: Connection state counts.

**Step 3: Look for packet loss**
- `ping -f` (flood ping) to backend instances. Any loss?
- `traceroute` or `mtr` to identify which hop is showing loss or increased RTT.
- Check cloud provider's network metrics for packet loss or congestion at the VPC/subnet level.
- Check NIC statistics: `ethtool -S eth0 | grep -i drop` — RX drops at the NIC level often indicate RX buffer overflow (the ring buffer is full because the kernel isn't consuming packets fast enough — a CPU problem, or IRQ affinity issue).

**Step 4: DNS**
- Is DNS resolution timing contributing? Some libraries (especially Go pre-1.12 and Java) have issues with DNS caching or concurrent resolution. Check DNS query latency with `dig +stats`.
- Check if TTLs expired and DNS is re-resolving more frequently (e.g., after a deploy that changed IPs).

**Step 5: TCP-level issues**
- Are you hitting the `SYN` backlog (`net.ipv4.tcp_max_syn_backlog`)? Check `ss -lnt` for accept queue overflow: `Recv-Q` on a listening socket shows backlog depth.
- Is Nagle's algorithm batching small writes? For low-latency RPC services, `TCP_NODELAY` should be set. Check that this wasn't accidentally removed.
- Check RTO: `ss -ti` shows `rto` for individual connections. Unexpectedly high RTOs indicate the kernel's path RTT estimate is wrong (maybe after a network path change).

**Step 6: Middle-box / infrastructure issues**
- Did a load balancer change health check parameters, causing more aggressive connection cycling?
- Did a deploy change the number of backend instances, concentrating load and increasing backend processing time (not a networking issue per se, but manifests as latency)?
- Are connection pools saturated? If your service uses connection pools to a database and pool size is exhausted, requests queue waiting for a connection. This looks like network latency but is actually queue time.
- Check cloud security group or iptables for any recently added rules that might be doing stateful tracking on a high-connection-rate service (state table exhaustion).

**Step 7: Tools**
- **Wireshark/tcpdump:** Capture a few seconds of traffic on the affected instance. Look for retransmissions, duplicate ACKs, RSTs, and RTT samples between SYN and SYN-ACK.
- **`tc` (traffic control):** Check if any traffic shaping or `qdisc` (queueing discipline) was accidentally applied that could be adding queuing delay.
- **eBPF/BCC tools:** `tcplife`, `tcpretrans`, `tcptop` from the bcc-tools package give real-time, low-overhead TCP event tracing without a full packet capture.

---

**Q22. Explain NAT, its types, and specifically why it breaks certain protocols. How does STUN/TURN/ICE solve this for WebRTC?**

**Model Answer:**

**NAT (Network Address Translation):**

NAT maps private IP:port pairs to public IP:port pairs, allowing multiple devices to share a single public IP. The NAT device maintains a translation table.

**Types:**

1. **Full-cone NAT:** Once a mapping is created (private IP:port → public IP:port), any external host can send packets to the public IP:port and they'll be forwarded to the private host. Most permissive.

2. **Restricted-cone NAT:** External hosts can send packets to the public IP:port only if the internal host previously sent a packet to that external IP (any port).

3. **Port-restricted cone NAT:** Same as restricted-cone, but also requires the external port to match.

4. **Symmetric NAT:** A new mapping (different public port) is created for each unique destination (IP + port). The mapping is not reusable for other external endpoints. Most restrictive. This breaks many NAT traversal techniques.

**Why NAT breaks protocols:**

- **FTP (Active mode):** The FTP server initiates a data connection back to the client. The client's private IP is embedded in the `PORT` command payload. The NAT device doesn't know to rewrite the embedded IP in the application payload. The server tries to connect to the private IP, which is unreachable. *Fix: FTP in Passive mode (client initiates both connections), or ALG (Application Layer Gateway) that rewrites FTP payloads.*

- **SIP/VoIP:** SIP messages embed the client's private IP/port in `Contact` and `SDP` headers. The remote party tries to send media to the private IP — NAT breaks it. *Fix: STUN/TURN (see below), or SIP-aware ALG.*

- **IPsec (AH mode):** AH (Authentication Header) authenticates the entire IP header including the source IP. NAT changes the source IP, breaking the authentication. *Fix: Use ESP (Encapsulating Security Payload) with NAT-T (NAT Traversal, which encapsulates ESP in UDP).*

**WebRTC NAT traversal with STUN/TURN/ICE:**

**STUN (Session Traversal Utilities for NAT):**
A client sends a STUN binding request to an external STUN server. The server echoes back the source IP:port as seen from the public internet — the client's *server-reflexive address* (its NATted public IP:port). The client now knows its external address and can share it with peers via signaling.

Limitation: STUN works for full-cone, restricted-cone, and port-restricted-cone NATs. For symmetric NAT, each new destination gets a different mapping, so the STUN-discovered address doesn't work for direct peer connection.

**TURN (Traversal Using Relays around NAT):**
When direct NAT traversal fails, TURN provides a relay server. Both peers allocate relay addresses on the TURN server. All media flows through the relay. This always works but adds latency (RTT through relay) and relay bandwidth cost.

**ICE (Interactive Connectivity Establishment):**
ICE is the framework that orchestrates STUN and TURN. It collects *candidates* for each peer:
- **Host candidates:** Local IP:port addresses.
- **Server-reflexive candidates:** STUN-discovered public IP:port.
- **Relay candidates:** TURN server addresses.

ICE then performs *connectivity checks* — both peers exchange candidates via signaling (SDP), then systematically try to connect to each other's candidates in priority order (direct > server-reflexive > relay). The first working candidate pair is used. ICE supports *aggressive nomination* (use the first working pair immediately) or *regular nomination* (collect all working pairs, then pick the best).

In production WebRTC, STUN succeeds ~80% of the time, and TURN is used as fallback for the remaining ~20% (mostly symmetric NAT behind corporate firewalls). This is why running a TURN server is non-optional for production WebRTC.

---

These questions span the full depth expected of a principal or staff-level software engineer. A truly exceptional candidate won't just recite these answers — they'll draw on production war stories, cite RFCs from memory, and connect the concepts laterally (e.g., "this is why Cloudflare uses io_uring in their Pingora proxy" or "this is exactly the failure mode behind the 2021 Facebook outage"). The model answers above represent the *floor* of what would impress at that level — real depth comes from the war stories and the connections.