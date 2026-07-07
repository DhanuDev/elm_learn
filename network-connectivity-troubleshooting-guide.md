# Network Connectivity Troubleshooting Guide
### From Basic Checks to Advanced Diagnostics (with an OpenShift/OCP Case Study)

**Audience:** Application Support / L2-L3 engineers
**Goal:** Move from "run telnet, nslookup, curl -v and hope" to a systematic, layer-by-layer method that lets you *prove* where a connection is breaking and hand off to the right team with evidence.

---

## 1. The Core Idea: Troubleshoot by Layer, Not by Habit

Every connectivity problem lives at one (or more) of these layers. If you check them **in order**, you never waste time debugging TLS certs when the real problem is DNS.

| Layer | What it answers | Breaks look like |
|---|---|---|
| L1/L2 – Physical/Link | Is the NIC/cable/Wi-Fi/VPN adapter up? | No IP, "media disconnected", VPN adapter down |
| L3 – IP / Routing | Can packets get from A to B? | `ping` timeout, "no route to host" |
| DNS | Does the name resolve to the right IP? | `nslookup`/`dig` fails or returns wrong/stale IP |
| L4 – TCP/UDP (Port) | Is something listening on that port, and is the path to it open? | `telnet`/`Test-NetConnection` hangs or refuses |
| TLS/SSL | Is the certificate valid, trusted, and does the handshake complete? | `curl` "SSL handshake failed", cert expired |
| L7 – Application (HTTP/API) | Does the app respond correctly once connected? | HTTP 401/403/500, timeouts after connection opens |

**Rule of thumb:** work top-down through this table. Each layer you confirm "good" lets you rule out an entire category of root cause.

---

## 2. Escalation Ladder — Basic → Advanced

For a single symptom like "I can't connect to X:port", here is the same problem checked at increasing depth:

### Level 1 — Basic (what you're doing today)
```bash
ping <host>
nslookup <host>
telnet <host> <port>
curl -vf https://<host>
```
Tells you: reachable/unreachable, resolves/doesn't, port open/closed, HTTP-level success/failure. Good for "is it up or down," bad for "why."

### Level 2 — Intermediate (path & detail)
```bash
tracert <host>              # Windows
traceroute <host>           # Linux/macOS
dig <host> +trace           # full DNS resolution path
Test-NetConnection <host> -Port <port> -InformationLevel Detailed   # PowerShell
nc -vz <host> <port>        # netcat port check with verbose output
openssl s_client -connect <host>:<port> -servername <host>
curl -v --resolve <host>:<port>:<ip> https://<host>   # force curl to use a specific IP
```
Tells you: *where along the path* it breaks, whether it's a routing hop, whether the cert chain is trusted, whether DNS is returning inconsistent results across resolvers.

### Level 3 — Advanced (proof-level evidence)
```bash
mtr -rw <host>                       # continuous traceroute + packet loss % per hop
nmap -Pn -p <port> <host>            # scan without relying on ICMP
tcpdump -i any host <ip> and port <port> -w capture.pcap
ss -tanp | grep <port>               # Linux socket states
netstat -ano | findstr <port>        # Windows socket states
curl -w "@curl-format.txt" -o /dev/null -s https://<host>   # timing breakdown (DNS/connect/TLS/TTFB)
Wireshark (open capture.pcap)        # visual packet analysis
```
Tells you: exact packet loss per hop, whether SYN packets are sent but never ACKed (firewall drop vs. host down), TCP retransmissions, whether the client-side socket even attempts the handshake, precise millisecond breakdown of where time is spent.

---

## 3. Command Reference by Layer

### 3.1 Link/Interface (L1-L2)
| Task | Windows | Linux/macOS |
|---|---|---|
| Show interfaces & IP | `ipconfig /all` | `ip a` / `ifconfig` |
| Show ARP table | `arp -a` | `arp -a` / `ip neigh` |
| Show VPN adapter status | `ipconfig /all` (look for VPN adapter) | `ip a` (look for tun/tap) |

### 3.2 Routing (L3)
| Task | Windows | Linux/macOS |
|---|---|---|
| Ping | `ping <host>` | `ping <host>` |
| Trace route | `tracert <host>` | `traceroute` / `mtr` |
| Show routing table | `route print` | `ip route` / `netstat -rn` |
| Continuous loss analysis | `pathping <host>` | `mtr -rw <host>` |

> **Note:** Many corporate/cloud firewalls block ICMP (ping/traceroute) entirely. **A failed ping does NOT confirm the host is down** — it only confirms ICMP is blocked or the host is down. Always corroborate with a TCP-level check (telnet/nc/Test-NetConnection) before concluding.

### 3.3 DNS
| Task | Command |
|---|---|
| Basic lookup | `nslookup <host>` |
| Lookup against a specific DNS server | `nslookup <host> <dns-server-ip>` |
| Full resolver trace (Linux) | `dig <host> +trace` |
| Query type-specific record | `dig <host> A` / `dig <host> CNAME` |
| Windows equivalent of dig | `Resolve-DnsName <host> -Server <dns-ip>` |
| Reverse lookup | `nslookup <ip>` |
| Check TTL / caching issues | `dig <host>` and look at the TTL field |

**What to conclude:**
- Resolves to the *wrong* IP → stale DNS cache, split-horizon DNS misconfig, or you're on the wrong network/VPN for that private zone.
- Doesn't resolve at all → DNS server unreachable, wrong DNS server configured, or the record genuinely doesn't exist from your vantage point (common with internal-only zones when off VPN).

### 3.4 TCP/UDP Port Reachability (L4)
| Task | Command |
|---|---|
| Basic TCP port check | `telnet <host> <port>` |
| Windows native (no telnet client needed) | `Test-NetConnection <host> -Port <port>` |
| Netcat (Linux) | `nc -vz <host> <port>` |
| Port scan a range | `nmap -p 1-65535 <host>` |
| Port scan skipping host-alive check | `nmap -Pn -p <port> <host>` |
| Local listening sockets (Linux) | `ss -tulnp` |
| Local listening sockets (Windows) | `netstat -ano \| findstr LISTEN` |

**What to conclude:**
- Connection **refused** immediately → host is up and reachable, but nothing is listening on that port (service down, wrong port).
- Connection **times out / hangs** → packet is being silently dropped somewhere (firewall, security group, ACL, routing black hole) — this is the classic signature you hit with the OCP error below.

### 3.5 TLS/SSL
| Task | Command |
|---|---|
| Full handshake + cert chain | `openssl s_client -connect <host>:<port> -servername <host>` |
| Check cert expiry only | `echo \| openssl s_client -connect <host>:<port> 2>/dev/null \| openssl x509 -noout -dates` |
| Verify against a specific CA bundle | `openssl s_client -connect <host>:<port> -CAfile ca.pem` |
| curl showing cert details | `curl -vI https://<host>` |

### 3.6 HTTP/Application (L7)
| Task | Command |
|---|---|
| Verbose request/response | `curl -v https://<host>/path` |
| Fail on HTTP error codes | `curl -vf https://<host>/path` |
| Timing breakdown | `curl -w "@curl-format.txt" -o /dev/null -s https://<host>` |
| Force a specific resolved IP (bypass DNS) | `curl --resolve host:port:ip https://host` |
| Follow redirects | `curl -vL https://<host>` |

`curl-format.txt` example for timing breakdown:
```
    dns_resolve:  %{time_namelookup}\n
   tcp_connect:  %{time_connect}\n
   tls_handshake: %{time_appconnect}\n
   time_to_first_byte: %{time_starttransfer}\n
   total: %{time_total}\n
```
This single command tells you *which phase* is slow or hanging — DNS, TCP connect, TLS, or the server's actual response — instead of guessing.

### 3.7 Packet Capture (deepest level of proof)
```bash
# Linux
tcpdump -i any host <target-ip> and port <port> -w capture.pcap

# Windows (needs Npcap) — usually easier via Wireshark GUI directly
```
Open the `.pcap` in Wireshark. What you're looking for:
- **SYN sent, no SYN-ACK ever comes back** → packet is being dropped in transit (firewall/ACL/routing) — this is proof it's a network path issue, not the client.
- **SYN sent, RST comes back immediately** → something (host or firewall) is actively rejecting — port closed or explicitly blocked.
- **SYN-ACK comes back but hangs after** → likely proxy/inspection device interfering, or asymmetric routing.

This distinction (no response vs. active reject) is the single most useful fact for deciding whether to escalate to network team vs. application/server team.

---

## 4. A Simple Decision Flow

```
START: "Can't connect to host:port"
   │
   ├─ 1. Does nslookup/dig resolve to the EXPECTED IP?
   │      NO  → DNS issue (wrong resolver, off-VPN, stale cache) → fix DNS/VPN, retest
   │      YES → continue
   │
   ├─ 2. Is the IP reachable at L3? (ping — but treat blocked ICMP as inconclusive)
   │      Use ping AND tracert/mtr together
   │      Total timeout at hop 1 or 2 → local network/VPN/gateway issue
   │      Reaches partway then dies → issue at that hop (ISP, firewall, LB) → note the last responding hop
   │
   ├─ 3. Is the TCP port open? (telnet / Test-NetConnection / nc)
   │      REFUSED  → host up, service down/wrong port → escalate to app/infra owner
   │      TIMEOUT  → packet silently dropped → firewall/security-group/ACL issue → escalate to network team with mtr+tcpdump evidence
   │      OPEN     → continue
   │
   ├─ 4. Does the TLS handshake complete? (openssl s_client)
   │      NO  → cert expired/untrusted/SNI mismatch → fix cert or client trust store
   │      YES → continue
   │
   └─ 5. Does the application respond correctly? (curl -v, check status code/body)
          4xx/5xx → app-layer issue (auth, config, backend down) → escalate to app owner with response body/logs
          Hangs after connect → app is accepted the connection but not responding → check app/server-side logs, thread pool exhaustion, etc.
```

---

## 5. Case Study: Your `oc login` Error

### The error
```
oc login --token=... --server=https://api.prod2ocp4.elm.sa:6443
error: dial tcp 192.168.59.25:6443: connectex: A connection attempt failed because the
connected party did not properly respond after a period of time...
```

### Step 1 — Read the error precisely
`dial tcp` means the Go HTTP client (used by `oc`) got as far as **attempting a TCP handshake** and it **timed out** — not refused, not a TLS error, not an auth error. This immediately tells you two things:
1. DNS resolution *already succeeded* — `api.prod2ocp4.elm.sa` resolved to `192.168.59.25`. If DNS had failed you'd see "no such host" instead.
2. The failure is at **Layer 3/4** — the SYN packet either never reached the API server/load balancer, or the response never made it back. It is **not** a token/auth/certificate problem (those errors only occur after a successful TCP+TLS handshake, so don't waste time re-checking the token yet).

### Step 2 — Confirm DNS is resolving where you expect
```powershell
nslookup api.prod2ocp4.elm.sa
```
Confirm it returns `192.168.59.25` (a private/RFC1918 address). This tells you the cluster's API endpoint is only reachable over an internal network — **you almost certainly need to be on VPN or on-prem network to reach it.**

### Step 3 — Basic reachability, done properly
```powershell
Test-NetConnection 192.168.59.25 -Port 6443 -InformationLevel Detailed
ping 192.168.59.25
tracert 192.168.59.25
```
- If `tracert` dies at your first or second hop (your gateway or VPN concentrator) → you're not on the required network/VPN, or split-tunnel VPN doesn't route `192.168.59.0/24`.
- If it reaches several hops then stalls at a known internal firewall/hop → flag that hop to network team.

### Step 4 — Isolate client vs. network vs. server
This is the key "how do I conclude" step — test from more than one vantage point:

| Test from | Result | Conclusion |
|---|---|---|
| Your laptop, VPN connected | Times out | Could still be firewall/LB — go to next row |
| Your laptop, VPN connected, different network (mobile hotspot + VPN) | Times out | Points away from your local Wi-Fi/LAN, toward VPN routing or the server side |
| A jump host / bastion already inside the cluster network | Succeeds | **Confirms it's a client-side network/VPN/firewall issue**, not the API server |
| Same jump host | Also times out | **Confirms it's server-side** — API server/HAProxy/keepalived VIP is down, or a firewall change was made — escalate to OCP cluster admin |
| Ask OCP admin to check from inside cluster (`oc get co`, check API server pods, check LB/haproxy/keepalived VIP status) | — | Definitive server-side confirmation |

This table is the actual deliverable you hand to whichever team you escalate to — it proves you isolated the layer before asking them to look.

### Step 5 — Conclusion & typical root causes for this exact signature
For a `dial tcp ... connectex: connection attempt failed` to an internal (`192.168.x.x`) OCP API VIP, the most common root causes, roughly in order of frequency:
1. **Not connected to VPN**, or VPN is connected but **split-tunnel routes don't include** the `192.168.59.0/24` subnet.
2. **Client-side corporate firewall/proxy** blocking outbound 6443.
3. **The API load-balancer VIP (keepalived) has failed over or is down** — common right after a master node reboot/patching if keepalived didn't re-elect properly. Only the OCP admin can confirm this via `oc get co`, checking HAProxy/keepalived status on the bootstrap/LB nodes, or `crictl`/`journalctl` on masters.
4. **A firewall/ACL change** was made on the network path between your location and the cluster (common after a network team maintenance window).
5. Less likely given this specific error, but possible: **API server pods crash-looping** — this usually gives a different, faster "connection refused" rather than a timeout, since something would still be listening on the node even if the app isn't healthy. A pure timeout leans toward network path or LB VIP, not the pods themselves.

### Step 6 — What to send in your escalation ticket
Always attach:
- The exact `oc login` command + error (already precise).
- `nslookup` output showing the resolved IP.
- `Test-NetConnection`/`tracert` output showing where it dies.
- VPN connection status/screenshot (connected, which profile, split-tunnel or full-tunnel).
- Timestamp, and whether it worked before (if so, when did it last work, and what changed — patching window, VPN client update, etc.).
- If you have access to any bastion/jump host on the cluster network, the result of testing from there.

This turns "OCP login is broken" into "TCP handshake to 192.168.59.25:6443 times out from client network X, succeeds/fails from jump host Y, VPN routes are/aren't present for that subnet" — which is instantly actionable for network or OCP admin teams.

---

## 6. Other Common OCP/Kubernetes API Connectivity Scenarios

| Symptom | Likely layer | Typical cause | Quick check |
|---|---|---|---|
| `dial tcp: connectex ... timed out` | L3/L4 | VPN/firewall/LB VIP down (this doc's case study) | tracert + Test-NetConnection from multiple vantage points |
| `dial tcp: connectex ... actively refused` | L4 | Nothing listening on 6443 — API server down or wrong port/IP | Test-NetConnection; ask admin to check `oc get co`/API pods |
| `no such host` | DNS | DNS record missing, wrong DNS server, off-VPN for internal zone | `nslookup`/`dig` against the zone's actual DNS server |
| `x509: certificate signed by unknown authority` | TLS | Cluster's CA not trusted by client, or `--insecure-skip-tls-verify` needed for a self-signed cluster | `openssl s_client -connect host:6443` and inspect the chain |
| `x509: certificate has expired or is not yet valid` | TLS | API server cert or ingress cert expired — common cause of *sudden* outages right after a cert's 1-year/90-day expiry | `openssl s_client ... \| openssl x509 -noout -dates` |
| Login succeeds but `oc get pods` etc. hang/timeout | L7, different endpoint | Console/API works but a specific service route/ingress is down, or RBAC issue | `curl -v` directly against the specific route |
| `Unauthorized` / 401 after successful TCP+TLS | L7 (auth) | Token expired/revoked, wrong cluster context, clock skew | Re-issue token; check `oc whoami --show-server` |
| Intermittent timeouts (works sometimes) | L3, often LB | One of several HAProxy/API server replicas unhealthy, DNS round-robin hitting a bad node | Repeat `Test-NetConnection` several times; ask admin to check health of all LB backends |

---

## 7. Building Your Own Habit/Checklist

A good personal SOP to internalize (print this mentally before every "can't connect" ticket):

1. **Read the exact error text first** — Go/Java/Python error messages almost always tell you which layer failed (DNS vs. TCP vs. TLS vs. HTTP). Don't skip straight to tools.
2. DNS check (`nslookup`/`dig`) — confirm it resolves, and to the *expected* IP.
3. Path check (`ping` + `tracert`/`mtr`) — treat ping-only failure as inconclusive (ICMP is often blocked).
4. Port check (`telnet`/`Test-NetConnection`/`nc`) — distinguish **timeout** (dropped somewhere) from **refused** (nothing listening).
5. TLS check (`openssl s_client`) if the port is open but the app-level connection fails.
6. HTTP/app check (`curl -v`) — read status code and body, not just "it failed."
7. **Test from a second vantage point** whenever possible (different network, VPN on/off, jump host) — this single step does more to isolate client-vs-network-vs-server than any single tool.
8. Only after 1-7, escalate — with the evidence from each step attached.

---

## 8. Learning Resources (Basic → Advanced)

**Foundational / free, video-based**
- **Professor Messer — Network+ course** (professormesser.com / YouTube) — the best free, structured start for OSI layers, DNS, TCP/IP fundamentals.
- **PracticalNetworking.net** (Ed Harmoush) — excellent deep-dive articles/videos on subnetting, DNS, VPNs, routing, explained visually and intuitively.
- **Cisco Networking Academy (NetAcad)** — free self-paced "Networking Basics" and CCNA-track courses, good for structured fundamentals.

**Reference & cheat sheets**
- **PacketLife.net cheat sheets** — concise printable references for subnetting, common protocols, IOS commands.
- **Julia Evans' zines** (wizardzines.com) — short, very approachable illustrated guides specifically on DNS, `curl`, `tcpdump`, and debugging networks; great for building intuition fast.

**Deeper / protocol-level**
- **"High Performance Browser Networking" by Ilya Grigorik** — free online (hpbn.co) — outstanding for understanding TCP, TLS handshakes, and HTTP behavior in depth.
- **Wireshark's official documentation & "Wireshark University" materials** (wireshark.org/docs) — once you're comfortable with the CLI tools, this is the natural next step for packet-level proof.
- **RFCs** for the protocols themselves (RFC 793 TCP, RFC 1035 DNS, RFC 8446 TLS 1.3) — dense, but the ultimate ground truth when tool behavior seems to disagree with documentation.

**Certification-oriented (structured curriculum, useful even if not pursuing the cert)**
- **CompTIA Network+** study guides/objectives — a solid checklist of "everything an app support engineer should know" about networking fundamentals.
- **Red Hat's official OpenShift networking & troubleshooting documentation** (docs.redhat.com / docs.openshift.com) — essential once you're past generic networking and into cluster-specific behavior (SDN/OVN-Kubernetes, routes, ingress, HAProxy).

**Hands-on practice**
- **TryHackMe** — has free, guided "networking fundamentals" rooms that let you practice `nmap`, `tcpdump`, DNS enumeration in a safe lab.
- Set up a local lab (VMs or containers) and deliberately break DNS/firewall/routing to practice diagnosing your own induced failures — this builds the fastest intuition.
