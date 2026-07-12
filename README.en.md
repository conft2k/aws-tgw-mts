# Zero-Downtime Hybrid Multicast Architecture for MTS Services

**Sub-second failover with Amazon VPC Route Server, multicast traffic replication with AWS Transit Gateway**

> 한국어 문서: [README.md](README.md)

This is a hybrid multicast HA design that delivers on-premises multicast sources
to AWS receivers via **GRE (anycast anchor) + TGW multicast domain (IGMPv2)**.
Failover is performed by **VPC Route Server (BGP+BFD) + AS-path prepending**
switching VPC routing — the on-premises side (CGW) fails over with **zero
configuration changes**.

## Background

### The problem: two gaps in hybrid multicast

When you try to extend multicast-dependent workloads — market data feeds,
broadcast contribution, command-and-control systems — into AWS, you hit two gaps.

1. **The AWS network does not natively forward multicast.** The TGW multicast
   domain is the only native mechanism, but it works only inside VPCs, and
   Direct Connect/VPN do not carry multicast, so there is no direct
   interconnect with an on-premises PIM domain. The way through is a
   **GRE tunnel**: pull multicast to an EC2 router (C8000V) inside the VPC
   and have it emit into the TGW domain.
2. That instantly creates a new problem — **the EC2 router terminating GRE
   becomes a single point of failure**. Even with two routers, VPC route
   tables are static, so you need "something" that detects failure and
   repoints routes at the standby router.

### Limits of prior approaches

Before Route Server, you had to build that "something" yourself: CloudWatch
alarms + Lambda calling route-table APIs to swap next-hops, ENI reattachment,
or inserting a GWLB. The common limitations: detection takes tens of seconds
to minutes, the failure-detection and switchover logic is all custom code
that must be tested and maintained, and there is no standard way to express
routing policy such as active/standby preference.

### The value of VPC Route Server — the heart of this design

**Amazon VPC Route Server brings standard routing-protocol semantics to VPC
route tables.** It peers eBGP with EC2 appliances, dynamically injects the
routes they advertise into VPC route tables, and withdraws them when a peer
fails. In this design, its value is:

| Value | What it means here | Measured |
|---|---|---|
| **Sub-second failure detection (BFD)** | Appliance health-checking with zero lines of custom code | VPC route switchover in ~1s |
| **Standard BGP policy** | Active/standby expressed with nothing but AS-path prepend — both routers run identical configs, differing by one policy line | Failover automatic; failback automatic (preemption) |
| **Combined with anycast** | Only the next-hop of the shared Lo0 /32 changes → GRE re-anchors to the standby router | Failover with **zero on-premises (CGW) changes** |
| **Persist Routes** | Keeps last routes on peer flap, preventing blackholes | No false switchover across 3 repeated reboots |
| **Injects into both route tables at once** | tgw-rtb (GRE ingress) and receiver-rtb (return path) updated by one session | Unidirectional multicast **lossless**; round-trip ~9s |

In short: the failover orchestration you used to write in Lambda is replaced
by BGP, BFD, and prepend — protocol behaviors proven over 30 years. Being able
to operate VPC routing with tools network engineers already know is what makes
this design work.

### Where it shines — MTS-style real-time market data distribution

This architecture is especially effective for environments like **real-time
market data distribution behind brokerage MTS/HTS platforms** (mobile/home
trading systems), where an on-premises multicast feed must be extended to
many receiving services in the cloud without interruption.

- **The traffic pattern matches exactly** — exchange market data feeds are
  **unidirectional UDP multicast streams**. The measured results of this
  design (no receive gap across the entire failure-and-failback cycle; see
  the unidirectional measurement in 4.1) apply to this pattern directly.
- **The feed source cannot be touched** — market data reception lines and
  equipment are hard to change for regulatory and contractual reasons. This
  design's property that failover happens entirely on the AWS side, with the
  on-premises CGW unchanged, fits that constraint as-is.
- **Receivers scale in the cloud** — the services consuming quotes
  (execution, order book, alerting) scale horizontally in AWS. With the TGW
  domain, adding a receiver is just an IGMP join, and TGW carries the
  replication load (router load unchanged).
- **Answers trading-hours availability questions with data** — planned
  switchover and failback are lossless, and even unplanned failure is bounded
  by BFD detection (~1s). Operational questions like "can we replace a router
  during market hours?" can be answered with measurements.

For the same reasons, it generalizes to workloads combining "unidirectional
multicast + untouchable source + cloud-side receiver scaling," such as
broadcast contribution feeds and telemetry/monitoring distribution.

```
Multicast server 172.20.51.x
  │ Gi4 (PIM)
  ▼
CGW (ASN 65000, Lo0 172.20.255.1 = GRE source + PIM RP)
  │
  │ DX Transit VIF ×2 ── underlay BGP 65000↔65300 + BFD, advertises Lo0 only
  ▼
DXGW (ASN 65300)
  │
  ▼
TGW (ASN 65400) ── attachment: tgw-sub-az1/2 (dedicated)
  │                multicast domain (IGMPv2): associated to receiver-sub
  ▼
VPC 10.1.1.0/24
  ├─ [tgw-sub .224/27]
  │    tgw-rtb: 10.255.255.1/32 → active router Gi1 (injected by Route Server)
  │
  ├─ [router-sub .0/26]
  │    C8000V-1 Gi1 .30 ↔ RS endpoint .29 (no prepend → active)
  │    C8000V-2 Gi1 .40 ↔ RS endpoint .41 (prepend ×3 → standby)
  │    Both terminate GRE Tunnel100 (shared anycast Lo0 10.255.255.1, overlay 172.16.100.2)
  │    Overlay eBGP (65000↔65200) + PIM ↔ CGW; learns LAN 172.20.51.0/24 → re-advertises to RS
  │    Route Server (ASN 65100) injects/switches the active router's Gi1 ENI route via BGP+BFD
  │
  └─ [receiver-sub .64/26]
       C8000V Gi2 .90/.100 (igmp static-group) → emits into TGW multicast domain
       Domain replicates to IGMPv2-joined members → Receiver-1 (az1), Receiver-2 (az2)

Data path: server → CGW(RP) → GRE → active C8000V → Gi2 → TGW domain → Receiver ×2
Failover:  RS detects via BFD → swaps tgw-rtb/receiver-rtb next-hops → re-anchors with CGW unchanged
```

---

## GRE overlay bring-up flow

The layer-by-layer sequence from tunnel establishment to traffic flow. GRE
operates with two address layers: the "outer" (underlay) and the "inner"
(overlay) addresses.

| Layer | CGW | C8000V |
|---|---|---|
| Tunnel outer (GRE encapsulation) | 172.20.255.1 (Lo0) | 10.255.255.1 (shared anycast Lo0) |
| Tunnel inner (overlay P2P /30) | 172.16.100.1 | 172.16.100.2 (shared by both routers) |

**① Underlay reachability** — GRE packets are unicast IP, so each tunnel
endpoint only needs to reach the other's outer address:

- **On-prem → AWS**: CGW learns the DXGW allowed prefixes
  (`10.1.1.0/24`, `10.255.255.1/32`) over DX BGP
- **AWS → on-prem**: CGW advertises only Lo0 (`172.20.255.1/32`) over DX
  (`LOOPBACK-ONLY`) → propagated to TGW. The C8000V returns via a static
  route `172.20.255.1/32 → Gi1 gateway` and the router-rtb `/32 → TGW` route

**② Actual path of the GRE-encapsulated packet** (on-prem → AWS):

```
CGW: encapsulates the original packet in GRE (outer src 172.20.255.1, dst 10.255.255.1)
 → DX Transit VIF → DXGW
 → TGW (static route 10.255.255.1/32 → VPC attachment)
 → arrives at the attachment ENI in tgw-sub
 → tgw-rtb lookup: 10.255.255.1/32 → [active C8000V Gi1 ENI]   ← injected/switched by Route Server
 → active C8000V receives and decapsulates GRE
```

That final hop (the `/32` route in tgw-rtb) is the **key steering point of the
anycast design** — both C8000Vs advertise the same Lo0 to the RS (standby with
prepend), and the RS best-path decides this route's next-hop.

**③ Tunnel establishment** — GRE is stateless; there is no negotiation. The
tunnel shows up/up as soon as the two ends' source and destination mirror
each other. MTU 1476 / TCP MSS 1436 compensate for the 24B encapsulation
overhead.

**④ Overlay control plane** — on the in-tunnel P2P (172.16.100.0/30):

- **eBGP 65000↔65200** (+BFD 500ms×3, hold 3/9): CGW advertises only
  `ONPREM-LAN` (172.20.51.0/24) → the active C8000V receives it →
  re-advertises to the RS → return route injected into receiver-rtb.
  The C8000V→CGW direction is `DENY-ALL` (advertises nothing — VPC
  reachability is the DX underlay's job). CGW's `TUNNEL-IN` blocks the VPC
  CIDR and anycast /32 from entering via the tunnel (**prevents GRE
  recursive routing** — if the route to the tunnel destination points at the
  tunnel itself, the tunnel goes down)
- **PIM sparse-mode**: RP = CGW Lo0. The C8000V forces RPF for the RP/source
  onto the tunnel with `ip mroute` (the unicast route points at the underlay,
  which has no PIM). The (*,G) join created by Gi2's `igmp static-group`
  travels up the tunnel to the RP, and source traffic flows down the tunnel
  and out Gi2 → TGW domain

**⑤ Re-anchoring on failover** — when the active C8000V dies, the RS swaps
the `/32` next-hop in ② to the standby router's Gi1 ENI. The CGW's tunnel
destination is unchanged (10.255.255.1), so **the same tunnel terminates at
the standby router with no changes**; only overlay BGP/PIM re-establish with
the new router. The standby uses the same overlay IP (172.16.100.2), so the
CGW's neighbor configuration is also unchanged.

---

## 1. Setup order

### 1.1 CloudFormation deployment

```bash
aws cloudformation create-stack --stack-name mcast-base \
  --template-body file://aws-tgw-mts.yaml
```

- **Single-pass deployment**: RS peers are created in the same pass. A peer
  only needs the Gi1 ENI to exist (`DependsOn`) and waits with
  `BgpStatus: down` until the C8000V boots and initiates BGP.
- `LatestAmiId` is a pinned AMI. An SSM `:latest` reference re-resolves on
  every stack update and forces receiver replacement (which fails the update
  on fixed-IP collisions), so it is not used.
- **Two manual steps outside the stack** (repeat if the stack is recreated):
  1. Anycast static route in the TGW default route table:
     `aws ec2 create-transit-gateway-route --destination-cidr-block 10.255.255.1/32 --transit-gateway-route-table-id <TGW default RT> --transit-gateway-attachment-id <VPC attachment>`
  2. Include `10.1.1.0/24` + `10.255.255.1/32` in the DXGW association
     allowed prefixes (how on-prem learns the tunnel destination over DX)

### 1.2 C8000V router configuration

1. Launch the two C8000V instances and attach the stack-created ENIs
   (Gi1 = device index 0, Gi2 = device index 1; all created with
   Src/Dest check disabled).
2. Apply [c8000v1-conf.txt](c8000v1-conf.txt) / [c8000v2-conf.txt](c8000v2-conf.txt) respectively.

Key points:
- Both routers carry the **same Loopback0 (10.255.255.1/32)**, the GRE tunnel anchor.
- Only C8000V-2 applies `as-path prepend 65200 65200 65200` toward the RS →
  the RS selects C8000V-1 as best.
- Gi2's `ip igmp static-group 239.1.1.1` keeps (*,G) permanently — the key to
  zero forward loss during failover.
- Overlay BGP uses `fall-over bfd` + `timers connect 5` (faster recovery).
- Keep Gi1 on DHCP (it receives the ENI's fixed IP as-is) — static
  reconfiguration only risks cutting the session.

### 1.3 CGW (on-premises) router configuration

Apply [cgw-conf.txt](cgw-conf.txt). Add it alongside the existing DX Transit
VIF BGP configuration.

Key points:
- Lo0 (172.20.255.1/32) = GRE source + PIM RP. `ip pim sparse-mode` is mandatory.
- **One tunnel only** (Tunnel100 → 10.255.255.1). Failover happens on the AWS
  side, so the CGW never changes.
- Advertisement filters: `LOOPBACK-ONLY` (Lo0 only) toward DX (underlay),
  `ONPREM-LAN` (LAN only) toward the tunnel (overlay). LAN unicast flows only
  over the overlay.
- `TUNNEL-IN` blocks the VPC CIDR and the anycast /32 from entering via the
  tunnel — prevents GRE recursive routing (the loop where the tunnel
  destination points at the tunnel).
- **`ip pim sparse-mode` on Gi4** (where the multicast server attaches) is
  mandatory (FHR role — without it the source never registers with the RP and
  no (S,G) is created).
- Overlay BGP: `fall-over bfd` + `timers 3 9` (safety net that clears stale
  sessions on failback).

### 1.4 EC2 receiver setup (IGMPv2)

Apply the [receiver-setup.txt](receiver-setup.txt) procedure to both receivers. Summary:

```bash
# TGW supports IGMPv2 only (AL2023 defaults to v3)
sudo sysctl -w net.ipv4.conf.all.force_igmp_version=2 net.ipv4.conf.default.force_igmp_version=2
# For multicast ping tests (no reply by default)
sudo sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=0
# Group join anchor - TGW membership lasts only while this process lives!
sudo systemd-run --unit=mcast-anchor --property=Restart=always \
  socat -u UDP4-RECVFROM:5001,ip-add-membership=239.1.1.1:0.0.0.0,fork STDOUT
```

- Package installs (socat etc.) go through the S3 gateway endpoint — no
  internet access needed.
- Either the socat unit above or mcast-recv.py from
  [receiver-setup.txt](receiver-setup.txt) works as the join anchor (both
  keep the group membership alive).
- **iperf3 does not support multicast** — send stream tests from on-prem
  with iperf2 (see 3.5).
- Access: no key pair needed —
  `aws ec2-instance-connect ssh --instance-id <id> --connection-type eice`

---

## 2. CloudFormation resources

### 2.1 VPC subnets (VPC 10.1.1.0/24, contiguous per service)

| CIDR | Subnet | AZ | Purpose | Service supernet (for SG/ACL) |
|---|---|---|---|---|
| 10.1.1.0/27 | mcast-router-sub-az1 | 2a | C8000V-1 Gi1, RS Endpoint-1 | Router 10.1.1.0/26 |
| 10.1.1.32/27 | mcast-router-sub-az2 | 2c | C8000V-2 Gi1, RS Endpoint-2 | Router 10.1.1.0/26 |
| 10.1.1.64/27 | mcast-receiver-sub-az1 | 2a | C8000V-1 Gi2, Receiver-1, **multicast domain association** | Receiver 10.1.1.64/26 |
| 10.1.1.96/27 | mcast-receiver-sub-az2 | 2c | C8000V-2 Gi2, Receiver-2, **multicast domain association** | Receiver 10.1.1.64/26 |
| 10.1.1.224/28 | mcast-tgw-sub-az1 | 2a | **Dedicated TGW attachment** + EICE | TGW 10.1.1.224/27 |
| 10.1.1.240/28 | mcast-tgw-sub-az2 | 2c | Dedicated TGW attachment | TGW 10.1.1.224/27 |

> A multicast domain can associate subnets that do not belong to the
> attachment (verified in a real environment). This design separates them:
> the attachment uses dedicated subnets, the domain associates the receiver
> subnets.

### 2.2 ENIs / IPs

| IP | Purpose | Fixed? |
|---|---|---|
| 10.1.1.30 | C8000V-1 Gi1 (GRE underlay + RS peer-1) | ✅ Fixed (parameter) |
| 10.1.1.40 | C8000V-2 Gi1 (GRE underlay + RS peer-2) | ✅ Fixed (parameter) |
| 10.1.1.90 | C8000V-1 Gi2 (multicast emission) | ✅ Fixed (parameter) |
| 10.1.1.100 | C8000V-2 Gi2 (multicast emission) | ✅ Fixed (parameter) |
| 10.255.255.1 | Shared C8000V anycast Lo0 (GRE anchor, steered to the active router by RS) | ✅ Fixed (router config) |
| (auto) | Receiver-1/2 EC2 | ❌ Auto-assigned |
| (auto) | RS Endpoint-1/2 (BGP peer addresses, see Outputs) | ❌ Auto-assigned (cannot be set) |
| (auto) | TGW attachment ENIs ×2, EICE | ❌ Auto-assigned (cannot be set) |

### 2.3 BGP ASNs / peerings

| Session | Side A | Side B | Detection | Role |
|---|---|---|---|---|
| DX underlay | CGW 65000 (169.254.96.54/.62) | DXGW 65300 (.49/.57) | BFD | Advertises only Lo0 (GRE source); receives VPC/anycast prefixes |
| RS peering ×2 | C8000V-1/2 65200 (Gi1 .30/.40) | Route Server 65100 (RSE, auto IP) | **BFD 300ms×3** | Advertises anycast Lo0 + LAN to RS → RS injects into tgw-rtb/receiver-rtb. **Failover trigger** |
| GRE overlay | CGW 65000 (172.16.100.1) | Active C8000V 65200 (172.16.100.2 shared) | BFD 500ms×3, hold 3/9 | Carries LAN (172.20.51.0/24) to AWS. Standby's session stays down (anycast property) |
| TGW | TGW 65400 | DXGW 65300 | — | DXGW association (allowed-prefix advertisement) |

### 2.4 Timer design — covering both graceful shutdown and sudden death

The reboot in the failover measurements (4.1) is a "polite" failure: IOS-XE
closes BGP gracefully before going down. Sudden death — kernel panic,
hypervisor failure, forced instance termination — gives no such signal, and
detection falls to the BFD/BGP timers below.

| Timer | Value | Detection time | What it covers |
|---|---|---|---|
| RS BFD (Gi1) | 300ms × 3 | ~0.9s | **Active router sudden death** — the key timer that makes the RS withdraw routes and trigger failover |
| Overlay BFD (Tu100) | 500ms × 3 | ~1.5s | Lets the CGW tear down the overlay session to a dead router before hold-time — speeds up return-path switchover |
| Overlay BGP hold (CGW, `timers 3 9`) | keepalive 3s / hold 9s | ≤9s | Failback safety net — with anycast, BFD can be fooled by the surviving router answering, so the stale session is cleaned up by hold-time |
| BGP `timers connect 5` (C8000V) | 5s | — | Shortens the retry interval for the new overlay session right after switchover (removes the default 30s wait) |
| RS Persist Routes | 2 min | — | Keeps last routes on peer flap — prevents a transient flap from becoming a blackhole |

- **Expectation for sudden death**: RS BFD detection (0.9s) + route swap ≈
  **~1s forward outage** (for unidirectional streams). The return path takes
  a few more seconds until the standby's overlay BGP establishes (comparable
  to the measured 9.2s in the reboot test).
- **For graceful shutdown**: the BGP notification arrives before BFD expiry,
  so make-before-break holds — measured lossless (4.1 unidirectional).
- **Why not tighter**: tightening BFD to e.g. 100ms×3 speeds detection but
  makes the overlay BFD — which crosses DX — sensitive to path jitter and
  prone to false flaps. The asymmetry of 300ms on the directly-attached
  segment (Gi1) and 500ms on the tunnel is the balance point.

---

## 3. Verification guide

### 3.1 C8000V health

```
show ip bgp summary                  ! RS (.29/.41) Established; overlay only on the active router
show bfd neighbors                   ! RS and overlay BFD Up
show ip pim neighbor                 ! CGW (172.16.100.1) on Tunnel100
show ip mroute 239.1.1.1 count       ! (S,G) forwarding counters increasing = traffic flowing
show interfaces Tunnel100 | include packets
```

### 3.2 CGW health

```
show ip route 10.255.255.1           ! BGP route via DX (169.254.96.x)
show interface Tunnel100             ! up/up
show ip bgp summary                  ! 172.16.100.2 Established (PfxRcd 0 is normal - DENY-ALL)
show ip pim neighbor                 ! active C8000V on Tunnel100
show ip pim interface                ! Gi4, Lo0, Tunnel100 all listed
show ip mroute 239.1.1.1             ! with a source sending: (S,G) + OIL Tunnel100
show ip pim tunnel                   ! Tu0(Encap)/Tu1(Decap) = auto-created by PIM, normal
```

### 3.3 VPC Route Server health

```bash
aws ec2 describe-route-server-peers \
  --query 'RouteServerPeers[].{Peer:PeerAddress,Bgp:BgpStatus.Status,Bfd:BfdStatus.Status}'
# Expect: both peers BGP/BFD up

aws ec2 get-route-server-routing-database --route-server-id <rs-id>
# Expect: 10.255.255.1/32 - C8000V-1 (AS×1, in-fib/installed), C8000V-2 (AS×4, in-rib)
#         172.20.51.0/24 - in-fib from the active router

aws ec2 describe-route-tables --filters "Name=tag:Name,Values=mcast-tgw-rtb"
# Expect: 10.255.255.1/32 -> active C8000V Gi1 ENI
```

### 3.4 End-to-end test (multicast ping)

From the on-premises multicast server:

```bash
ping -t 8 239.1.1.1        # TTL required (default 1 dies at the first router)
```

Joined receivers each reply with unicast. Measured output (2026-07-12):

```
labuser@vm51:~$ ping -t 8 239.1.1.1
PING 239.1.1.1 (239.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.84: icmp_seq=1 ttl=125 time=33.1 ms
64 bytes from 10.1.1.116: icmp_seq=1 ttl=125 time=34.4 ms
64 bytes from 10.1.1.84: icmp_seq=2 ttl=125 time=4.36 ms
64 bytes from 10.1.1.116: icmp_seq=2 ttl=125 time=5.16 ms
```

How to read it:

- **Two replies per seq** = both receivers replying. Ping to a multicast
  destination expects multiple replies, so no `DUP!` marks appear (different
  from duplicate replies to a unicast ping).
- **ttl=125** = 3 hops from the source (CGW → C8000V → receiver, 128−3). If
  the path is longer or shorter than designed, this value changes.
- **Only the first reply has a high RTT (33ms → then 4–5ms)** = RP-tree first,
  then SPT switchover. If the (S,G) tree already exists when you restart, the
  first seq answers immediately; on a cold start, 1–2 unanswered first seqs
  are normal.

Check TGW members: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <id>`

### 3.5 End-to-end test (iperf2 UDP stream — workload-like)

Ping is a quick round-trip check; workload validation and loss measurement
use a unidirectional UDP stream. **Only iperf2 supports multicast** (iperf3
does not).

From the on-premises server:

```bash
iperf -c 239.1.1.1 -u -T 8 -t 600 -i 1 -b 1M -l 1200
```

- `-T 8`: TTL required (default 1 dies at the CGW)
- `-l 1200`: **datagram size required** — the default 1470B exceeds the
  tunnel MTU (1476) and is silently lost in its entirety (see 5.2). 1200B
  leaves headroom at 1228B including UDP/IP
- `-b 1M -l 1200` = ~109 packets/s → loss-measurement resolution ~9ms

Per-checkpoint verification (measured 2026-07-12):

```
! C8000V (active) - pps/avg size/kbps must match the send rate
show ip mroute 239.1.1.1 count
  Source: 172.20.51.10/32, Forwarding: 73811/109/1266/1105, Other: 73813/2/0
                                       ↑total ↑pps ↑avgB ↑kbps
```

On the receiver (mcast-recv.py must be listening on port 5001 — receiver-setup.txt):

```bash
tail -f /home/ec2-user/mcast-recv.log        # packet arrival log
```

Precise gap (loss window) measurement — record tcpdump timestamps, then
compute the maximum inter-packet gap:

```bash
sudo sh -c 'nohup tcpdump -lni ens5 -tt "udp and dst 239.1.1.1" > /tmp/mcast-times.txt 2>/dev/null &'
# After the test:
awk '{if(prev){g=$1-prev; if(g>max){max=g; at=prev}} prev=$1}
     END {printf "max_gap=%.3fs at %.3f\n", max, at}' /tmp/mcast-times.txt
```

Measured example (entire 4.1 reboot test): receiver-1 `max_gap=0.056s`,
receiver-2 `max_gap=0.053s` — lossless including failover and failback.

---

## 4. Failover tests

### 4.1 C8000V router failure (measured)

Procedure: with `ping -D -t 8 239.1.1.1 -i 0.1` running from on-prem, stop the
active C8000V instance.

Sequence: RS BFD detection (~1s) → RIB withdrawal → prepended C8000V-2 route
promoted to best → tgw-rtb/receiver-rtb next-hops swapped → GRE re-anchors to
the standby router (CGW unchanged) → overlay BGP re-establishes → LAN return
path restored.

Measured results (with BFD + timer tuning):

| Item | Result | UDP multicast loss time |
|---|---|---|
| VPC route switchover | ~1s after BFD detection (effectively 0 on graceful stop — make-before-break) | **0s** (no receive gap during switchover) |
| **Forward multicast loss** | **0 packets** (entire failure+failback window, 5,114 packets at 0.1s interval) — thanks to the pre-built (*,G) from Gi2 static-group | **0s** (reboot test: max gap 56ms across the entire window = jitter level) |
| Return (reply) path switchover | ≤1s (overlay BFD + connect 5) | N/A (unidirectional streams need no return path) |
| Failback return gap | ~90s — preemption right after boot fires before the overlay BGP is ready (structural limit). Treat failback as a planned operation | **0s** (lossless during failback too — reconfirmed in the reboot test) |

Recovery to the original state is automatic (preemption) — once C8000V-1
boots, it becomes active again with no manual steps.

#### Reboot-mode re-verification (2026-07-12, 1s-interval ping, 3 runs)

The active C8000V-1 was **rebooted** (an unplanned-failure scenario, not a
graceful stop) and the full cycle measured with on-prem ping (1s interval)
plus AWS API polling timestamps (numbers below are from the run that captured
both failover and failback in one session):

| Time (UTC) | Elapsed | Event | ping |
|---|---|---|---|
| 11:20:20 | 0s | reboot API call | normal |
| 11:20:22 | 2s | interfaces down, outage begins | loss from seq 40 |
| 11:20:31 | 11s | failover complete (BFD detect → RS switch → GRE re-anchor → C8000V-2 overlay up) | resumes at seq 48 — **9.2s gap** |
| 11:23:12 | 2m 51s | C8000V-1 boot complete, RS BGP re-established | normal (via C8000V-2) |
| 11:24:22 | 4m 02s | failback preemption (automatic): next-hop → C8000V-1 | loss from seq 279 |
| 11:25:54 | 5m 33s | C8000V-1 overlay BGP re-established | resumes at seq 368 — **92.1s gap** |

Ping statistics consistency: 372 sent / 97 lost = exactly 8 (failover) + 89
(failback). Every answered seq was answered by both receivers (275 received +
275 duplicates).

Key takeaways:

- **Unplanned failure (reboot) failover outage is ~9s** — set expectations
  separately from the graceful-stop make-before-break (lossless/≤1s, table
  above).
- **The ~90s failback gap is structural and reproducible** (run 1: 88s,
  run 3: 92s) — the outage starts with the RS route preemption and ends with
  the returning router's overlay BGP re-establishment. Preemption always
  precedes overlay readiness, so this gap cannot be tuned away; this is the
  basis for treating failback as a planned operation.
- **Failback preemption fires 66–83s after RS BGP re-establishment**
  (3 observations) — it does not wait the full PersistRoutes value (2 min),
  so be careful predicting failback timing.
- C8000V boot (reboot → RS BGP re-established) was **~3 minutes** in all
  three runs.

#### Unidirectional stream measurement (2026-07-12, UDP 1Mbps · 109pps · 1200B, receiver tcpdump)

The ping measurements above are round-trip (forward multicast + return
unicast). Running a **unidirectional UDP stream** (iperf) closer to real
workloads and measuring the same reboot scenario with tcpdump timestamps on
both receivers:

| Phase | Measured (receive gap) |
|---|---|
| Failover (reboot) | **Below measurement threshold (no gap)** |
| Failback (automatic preemption) | **No gap** |
| Max receive gap, entire window | receiver-1 56ms / receiver-2 53ms — jitter unrelated to the switchover instants |

Interpretation:

- **Confirmed: the 9.2s/92s ping gaps were entirely the return unicast
  path.** Forward multicast flows the instant routes switch, independent of
  overlay BGP, thanks to the pre-built (*,G) (igmp static-group) and static
  mroute RPF. Unidirectional distribution workloads (video, market data
  feeds) are **effectively lossless** through the whole failure-and-failback
  cycle.
- On EC2 reboot, IOS-XE closes BGP gracefully, so the RS withdraws
  immediately without waiting for BFD expiry → make-before-break. For a hard
  death without graceful close (kernel panic, forced instance kill), BFD
  detection (~1s) is the upper bound of the gap.
- For request-response (bidirectional) workloads, set expectations from the
  ping measurements (~9s failure, ~90s failback).

### 4.2 Direct Connect failure (bring down BGP)

Bring down BGP on one of the two DX Transit VIFs to verify underlay redundancy.

```
! On the CGW (block the VIF-1 path)
router bgp 65000
 neighbor 169.254.96.49 shutdown
```

What to watch:
- CGW: `show ip bgp summary` — .49 Idle(Admin), only .57 remains.
  `show ip route 10.255.255.1` — switches to the .57 next-hop
- GRE/multicast: only the underlay path changes, so **tunnel, PIM, and
  overlay BGP must stay up** (no tunnel flap; only RTT changes)
- Expected loss: around ~1s with BFD (300ms×3) detection
- Recovery: `no neighbor 169.254.96.49 shutdown`

> A physical link failure can be simulated with a VIF subinterface shutdown
> (`interface Gi1.351` → `shutdown`).

---

## 5. Monitoring and troubleshooting

### 5.1 C8000V

| Symptom | Check | Cause/action |
|---|---|---|
| RS BGP down | `show ip bgp summary`, RS peer status (CLI) | SG (c8000v-sg) allows the whole VPC? Gi1 ENI attached? |
| No (S,G) | `show ip mroute`, `show ip rpf 172.20.255.1` | RPF must point at Tunnel100 (static `ip mroute` entry required — the underlay has no PIM) |
| Tunnel up but no traffic | `show ip mroute 239.1.1.1 count` | If counters stall, check (S,G)/Register on the CGW side |
| Post-failover logs | `show logging \| include %BGP\|%BFD` | Analyze switchover timing from ADJCHANGE/BFD event timeline |
| Access | EICE: `aws ec2-instance-connect ssh` or SSH ProxyCommand `open-tunnel` | If the pem fails with `error in libcrypto`, decode the base64 body to DER and regenerate |

### 5.2 CGW

| Symptom | Check | Cause/action |
|---|---|---|
| Tunnel down/flap | `show interface Tunnel100`, `show ip route 10.255.255.1` | Recursive routing — 10.255.255.1 or 10.1.1.0/24 must never be learned via the tunnel (check TUNNEL-IN). DXGW allowed prefixes include the /32? |
| (S,G) never created | `show ip mroute 239.1.1.1`, `show ip pim interface` | **Missing PIM on Gi4** is the most common cause. Check Tu0/Tu1 (Register tunnels) exist |
| (*,G) only, empty OIL | `show ip mroute` | C8000V's PIM Join not arriving — check tunnel/overlay state |
| `%IGMP-3-QUERY_INT_MISMATCH ... 0.0.0.0` | — | Proxy query from a LAN snooping switch. Harmless (no effect on querier election) |
| Overlay BGP flap | `show ip bgp neighbors 172.16.100.2` | hold 3/9 is intentional. If it flaps repeatedly, check DX path quality |
| **Large UDP silently lost, ping passes** | C8000V `show ip mroute <grp> count` — source counter 0 | **Tunnel MTU (1476) overrun** (measured: iperf's default 1470B is lost entirely — it never even reaches the C8000V; 1200B passes). If payload + UDP/IP 28B exceeds 1476, DF-set packets are dropped at the CGW tunnel ingress, and even fragments that make it through are dropped by TGW (fragmented multicast is unsupported, per official docs) — keep multicast app payloads **≤1400B** (iperf: `-l 1200`) |

### 5.3 AWS / receivers

| Symptom | Check | Cause/action |
|---|---|---|
| 0 TGW group members | `search-transit-gateway-multicast-groups`, group missing from receiver `ip maddr show ens5` | Receiver join anchor (socat/mcast-recv.py) died → restart (systemd recommended). **If the instance is replaced, the script disappears with it — re-run setup** (receiver-setup.txt). Also verify not joining as IGMPv3 (`force_igmp_version=2`) |
| Receiver doesn't reply (ping) | Receiver `tcpdump -ni ens5 icmp` | Packets arrive + no reply → `icmp_echo_ignore_broadcasts=0`. No arrival → receiver SG (on-prem ranges as ICMP/UDP source), TGW membership |
| RS routes not injected | `get-route-server-routing-database` | If in-rib but not installed, check propagation is enabled on that route table |
| Stack update fails (IP collision) | Replacement column in the change set | Replacing fixed-IP resources is create-before-delete, so IPs collide — keep the AMI pinned; change IPs via recreation |

---

## Appendix. Router status sample outputs (measured)

Real outputs collected via EICE in the normal operating state (C8000V-1
active / C8000V-2 standby, captured 2026-07-12). Compare against these during
operational checks — the easiest point of confusion is that **on the standby
router, "overlay BGP Active (not established)" is normal**.

### A.1 C8000V-1 (active router)

`show ip interface brief | exclude unassigned` — Tunnel100 up/up, Lo0 = anycast 10.255.255.1

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.1.1.30       YES DHCP   up                    up
GigabitEthernet2       10.1.1.90       YES NVRAM  up                    up
Loopback0              10.255.255.1    YES NVRAM  up                    up
Tunnel0                172.16.100.2    YES unset  up                    up
Tunnel100              172.16.100.2    YES NVRAM  up                    up
```

`show ip bgp summary` — PfxRcd 0 on the RS session (.29) is normal (the RS
advertises nothing); 1 prefix from the overlay (172.16.100.1 = CGW) = on-prem
LAN 172.20.51.0/24

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.29       4        65100    1922    1880       13    0    0 02:40:04        0
172.16.100.1    4        65000    3099    3098       13    0    0 02:38:27        1
```

`show bfd neighbors` — both sessions Up: RS (Gi1) and overlay (Tu100)

```
NeighAddr                              LD/RD         RH/RS     State     Int
10.1.1.29                            4097/2807522020 Up        Up        Gi1
172.16.100.1                         4103/4110       Up        Up        Tu100
```

`show ip pim neighbor` — CGW is a PIM neighbor across Tunnel100

```
Neighbor          Interface                Uptime/Expires    Ver   DR
Address                                                            Prio/Mode
172.16.100.1      Tunnel100                02:39:25/00:01:44 v2    1 / S P G
```

`show ip pim rp mapping` — static RP = CGW Lo0

```
Group(s): 224.0.0.0/4, Static
    RP: 172.20.255.1 (ip-172-20-255-1.ap-northeast-2.compute.internal)
```

`show ip mroute 239.1.1.1` — (*,G) incoming interface is Tunnel100 (RPF
neighbor = CGW), outgoing interface is Gi2 (emission into the TGW domain)

```
(*, 239.1.1.1), 02:41:17/stopped, RP 172.20.255.1, flags: SJC
  Incoming interface: Tunnel100, RPF nbr 172.16.100.1, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 02:41:17/00:00:44, flags:
```

`show ip mroute 239.1.1.1` — **while traffic is flowing**, an (S,G) entry
appears under the (*,G) with `flags: JT` (SPT switchover complete). Captured
while the on-prem source 172.20.51.10 was pinging:

```
(172.20.51.10, 239.1.1.1), 00:00:59/00:02:00, flags: JT
  Incoming interface: Tunnel100, RPF nbr 172.16.100.1, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 00:00:59/00:02:00, flags:
```

`show ip mroute 239.1.1.1 count` — quantify traffic with the forwarding
counters. With 1s-interval ping expect ~1 pps (second field of `52/1/122/0`
below); RPF failed and drops must be 0. When traffic stops, the (S,G) expires
after ~3 minutes and the counters reset — low counters by themselves are not
a problem.

```
Group: 239.1.1.1, Source count: 1, Packets forwarded: 54, Packets received: 54
  RP-tree: Forwarding: 2/0/122/0, Other: 2/0/0
  Source: 172.20.51.10/32, Forwarding: 52/1/122/0, Other: 52/0/0
```

`show ip bgp neighbors 10.1.1.29 advertised-routes` — re-advertises the
anycast /32 and the CGW-learned LAN to the RS (only the active router has the
LAN route)

```
     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.255.255.1/32  0.0.0.0                  0         32768 i
 *>   172.20.51.0/24   172.16.100.1             0             0 65000 i
```

### A.2 C8000V-2 (standby router)

`show ip interface brief` — all interfaces up/up, same as the active router.
Lo0 10.255.255.1 is shared by both routers (anycast), and GRE is stateless so
the tunnel also shows up.

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.1.1.40       YES DHCP   up                    up
GigabitEthernet2       10.1.1.100      YES manual up                    up
Loopback0              10.255.255.1    YES manual up                    up
Tunnel100              172.16.100.2    YES manual up                    up
```

`show ip bgp summary` — **the overlay session in Active (not established) is
the normal standby state.** The CGW's GRE anchors only at the router the RS
selected as active, so no CGW traffic arrives on the standby's tunnel and the
overlay BGP never establishes.

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.41       4        65100    3705    3623        8    0    0 05:08:38        0
172.16.100.1    4        65000       0       0        1    0    0 02:40:22 Active
```

`show bfd neighbors` — only the RS (Gi1) session exists (overlay BFD is
created after the session establishes)

```
NeighAddr                              LD/RD         RH/RS     State     Int
10.1.1.41                            4097/3274535456 Up        Up        Gi1
```

`show ip bgp neighbors 10.1.1.41 advertised-routes` — advertises only the
anycast /32 (AS-path prepend is outbound policy so it does not appear here;
confirm AS×4 in the RS routing database). The LAN 172.20.51.0/24 is not
advertised because there is no overlay — normal.

```
     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.255.255.1/32  0.0.0.0                  0         32768 i
```

`show ip mroute 239.1.1.1` — the (*,G) exists but the RPF neighbor is 0.0.0.0
(no PIM neighbor across the tunnel). On failover, once the overlay
establishes, it transitions to the same shape as the active router.

```
(*, 239.1.1.1), 05:09:20/stopped, RP 172.20.255.1, flags: SJC
  Incoming interface: Tunnel100, RPF nbr 0.0.0.0, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 05:09:20/00:00:13, flags:
```

### A.3 Active vs standby at a glance

| Item | Active (C8000V-1) | Standby (C8000V-2) |
|---|---|---|
| RS BGP/BFD (Gi1) | Established / Up | Established / Up (same) |
| Overlay BGP (172.16.100.1) | Established, PfxRcd 1 | **Active (not established) — normal** |
| Overlay BFD (Tu100) | Up | No session |
| PIM neighbor (Tu100) | CGW 172.16.100.1 | None |
| Advertised to RS | 10.255.255.1/32 + 172.20.51.0/24 | 10.255.255.1/32 only (prepend ×3) |
| mroute (*,239.1.1.1) | RPF nbr = 172.16.100.1 | RPF nbr = 0.0.0.0 |

If the two routers' outputs are reversed from this table, a failover has
occurred — cross-check the currently active router via the RS routing
database and the 10.255.255.1/32 next-hop ENI in `mcast-tgw-rtb`.

---

## References (official AWS documentation)

**Amazon VPC Route Server**
- [Dynamic routing in your VPC using VPC Route Server](https://docs.aws.amazon.com/vpc/latest/userguide/dynamic-routing-route-server.html)
- [How Amazon VPC Route Server works](https://docs.aws.amazon.com/vpc/latest/userguide/route-server-how-it-works.html) — endpoint/peer/BFD/Persist Routes concepts

**AWS Transit Gateway multicast**
- [Multicast in AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html) — basis for section 5: IGMPv2 query cycle (2 min), SG/NACL requirements (protocol 2, source 0.0.0.0/32), etc.
- [Multicast domains](https://docs.aws.amazon.com/vpc/latest/tgw/multicast-domains-about.html)
- [Transit Gateway quotas (incl. MTU)](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html) — **fragmented multicast packets are dropped**: directly related to the GRE MTU trap in 5.2

**AWS Direct Connect**
- [Direct Connect gateways](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-gateways-intro.html)
- [Transit gateway associations (Transit VIF)](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-transit-gateways.html)
- [Allowed prefixes interactions](https://docs.aws.amazon.com/directconnect/latest/UserGuide/allowed-to-prefixes.html) — basis for manual step ② in 1.1

**Miscellaneous**
- [EC2 Instance Connect Endpoint](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-ec2-instance-connect-endpoint.html) — key-pair-less SSH access to routers/receivers
