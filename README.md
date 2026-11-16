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

**Note to self:** TLS straddles L4–L7 — it doesn't map cleanly to one layer. Good interview answer.
