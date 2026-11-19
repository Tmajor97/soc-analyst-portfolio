# soc-analyst-portfolio
week01-networking-cloud
# Week 1 — Networking & Cloud Fundamentals


Week 1 — Networking & Cloud Fundamentals

**Phase 1: Foundations | Day 2 of 7**

# What I'm learning this week
- OSI and TCP/IP models
- Cloud networking (VPC, subnets, security groups, NACLs)
- Where encryption fits in modern traffic (TLS 1.3, QUIC, DoH)
- How a SOC analyst *sees* each layer in logs

## Day 1 — OSI/TCP-IP models -  L5, L6, L7 are all happening at the same time on the same packet. They're not sequential steps
**

**What confused me:, "where TLS sits in the model" -TLS doesn't map cleanly to one OSI layer. It rides on L4 (TCP), manages the session at L5, and carries encrypted application data at L6–L7. Saying 'TLS is L5' or 'TLS is L7' is both wrong and right — it straddles them." That answer signals you actually understand it.


**Resource used:** Professor Messer Network+ — OSI Model + TCP/IP Model videos

## Day 2 — Cloud networking vocabulary + OSI→Cloud→SOC mapping

**Six cloud terms I now understand:**
1. VPC — your own private network in the cloud
2. Subnet (public vs private) — a slice of the VPC(public subnet gas route to the internet, private doesn’t
3. Internet Gateway —  your access to the internet through a private local network and public internet allowing data to flow safely between them
4. NAT Gateway — is a managed cloud service that allows resources in a private subnet to access the public internet or external networks while blocking unsolicited inbound connections from the outside.
5. Security Group (stateful) — it tracks the state of active connections and automatically allows return traffic for any request that has already been permitted
6. NACL (stateless) —  stateless firewall on the subnet (allow + deny rules)

**My VPC diagram:** `vpc-diagram.png` (draw.io → export PNG)

**The mapping table:**

| OSI Layer | What lives here | Cloud component | SOC log source |
|-----------|-----------------|-----------------|----------------|
| L7 Application | HTTP, DNS, app protocols | ALB, WAF, Route 53 | ALB logs, WAF logs, DNS query logs |
| L6 Presentation | Encryption, encoding, certificates | TLS termination, KMS, ACM | Cert metadata, KMS/CloudTrail events |
| L5 Session | Session setup/state, TLS handshake | LB session affinity, SG connection tracking | JA3 fingerprints, ALB logs |
| L4 Transport | TCP/UDP, ports | Security Groups | VPC Flow Logs (ports, flags) |
| L3 Network | IP, routing, packets | Route tables, IGW, NAT, NACLs | VPC Flow Logs |
| L2 Data Link | MAC addresses, switching | ENIs, virtual switch | VPC Flow Logs (partial); mostly abstracted |
| L1 Physical | Cables, hardware, data centers | Regions, AZs | Provider-managed; not customer-visible |
| **Control Plane** | **API calls that change infra** | **IAM, cloud management API** | **CloudTrail (AWS) / Activity Log (Azure)** |



## Day 3 — Subnetting (block-size method)

**Time spent:** 5hrs | **Confidence:** [1–5] 4.5

### Why a SOC analyst needs this
Flow logs give me raw IPs. Subnetting is how I turn them into meaning:
- Is this internal or external? (`10.x.x.x` = private, anything else = egress)
- Are these two hosts on the same subnet? (same = local; different = cross-subnet = lateral movement candidate)
- Does this security group rule actually cover this IP?
- Is `169.254.x.x` an attack or just a misconfigured instance?

Without subnetting I can't triage a single flow log line.

### The method (3 formulas)
For any subnet — block size `B`, network address `N`:
1. **Network address** = the boundary your IP sits above
2. **Broadcast** = `N + B − 1`
3. **Usable hosts** = `B − 2` (1 network + 1 broadcast reserved)

**Block size ladder (halves each step):**
| CIDR | Block | Usable | Subnet boundaries (4th octet) |
|------|-------|--------|-------------------------------|
| /24 | 256 | 254 | 0 |
| /25 | 128 | 126 | 0, 128 |
| /26 | 64 | 62 | 0, 64, 128, 192 |
| /27 | 32 | 30 | 0, 32, 64, 96, 128, 160, 192, 224 |
| /28 | 16 | 14 | 0, 16, 32, 48, ... 240 |
| /29 | 8 | 6 | 0, 8, 16, ... 248 |
| /30 | 4 | 2 | 0, 4, 8, ... 252 |

**Derivation (so I don't just memorize):** `/N` = first N bits are network, rest are host.
Block size = `2^(host bits in that octet)`. `/26` = 6 host bits = 2⁶ = 64.
Which octet is "interesting" = `ceil(N / 8)`. `/22` → ceil(22/8) = 3 → 3rd octet.

**Edge cases:** `/31` = 2 usable (point-to-point, RFC 3021). `/32` = 1 (single host). Everything /30 and below: usable = B − 2.

### Drills I completed

**Drill 1: `10.0.0.100/26`** — *got wrong first attempt (subtracted network/broadcast incorrectly), corrected*
- Block 64 → boundaries 0, 64, 128, 192 → 100 sits between 64 and 127
- Network `10.0.0.64` | Broadcast `10.0.0.127` | Usable `.65–.126` = **62**

**Drill 2: `192.168.5.200/27`** — *got wrong first attempt, corrected*
- Block 32 → boundaries 0, 32, ... 192, 224 → 200 sits between 192 and 223
- Network `192.168.5.192` | Broadcast `192.168.5.223` | Usable `.193–.222` = **30**

**Drill 3: `10.0.4.17/26` vs `10.0.4.50/26`**
- Both fall in 0–63 → **same subnet** (`10.0.4.0/26`)

**Self-test: `10.0.8.45/28`**
- Block 16 → boundaries 0, 16, 32, 48 → 45 sits between 32 and 47
- Network `10.0.8.32` | Broadcast `10.0.8.47` | Usable **14**

### Mistake log (what I got wrong and why)
- **Mistake:** subtracted network/broadcast incorrectly when computing usable range
- **Root cause:** [fill in — e.g., "confused broadcast address with the next subnet's network address"]
- **Fix:** broadcast = `N + B − 1`, not `N + B`. The next subnet starts at `N + B`.
- **Lesson:** usable = block − 2; don't hand-count the range.

### 5 flow log lines — what subnetting tells me

1. `10.0.4.17 → 10.0.9.3:22` — **cross-subnet**, both internal → lateral movement candidate
2. `10.0.4.17 → 203.0.113.45:443, 4.8MB` — internal → **external**, high volume, encrypted → exfil candidate
3. `10.0.4.17 → 198.51.100.20:8443, 412 bytes every 60s` — regular beaconing, unusual port → **C2 candidate**, pull PCAP + JA3 next
4. `169.254.169.254 → 10.0.4.17:80` — link-local / metadata endpoint → **misconfiguration, not attack**
5. `10.0.4.17 → 10.0.5.10:3389` — SG rule says `10.0.4.0/24`; `10.0.5.10` is **outside** that range → policy/log mismatch → check CloudTrail

### AI usage & verification log
- **Asked the LLM:** [e.g., "walk me through /26 subnetting with block size method"]
- **Verified independently:** [e.g., `python3 -c "import ipaddress; ..."` on every drill answer]
- **What I corrected:** [e.g., "I was computing broadcast as N+B; verified against ipaddress module that it's N+B−1"]

### Questions I still have
- [ ] [e.g., "how do I read the 3rd-octet case (/22) quickly in my head?"]

