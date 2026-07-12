# TGW Multicast 설정 가이드

온프레미스 멀티캐스트 소스를 **GRE(애니캐스트 앵커) + TGW 멀티캐스트 도메인(IGMPv2)**으로
AWS 리시버까지 전달하는 하이브리드 멀티캐스트 HA 구성입니다.
장애 전환은 **VPC Route Server(BGP+BFD) + AS-path prepending**이 VPC 라우팅을
전환하는 방식으로, 온프레미스(CGW)는 아무 변경 없이 페일오버됩니다.

```
멀티캐스트 서버 172.20.51.x
  │ Gi4 (PIM)
  ▼
CGW (ASN 65000, Lo0 172.20.255.1 = GRE 소스 + PIM RP)
  │
  │ DX Transit VIF ×2 ── 언더레이 BGP 65000↔65300 + BFD, Lo0만 광고
  ▼
DXGW (ASN 65300)
  │
  ▼
TGW (ASN 65400) ── attachment: tgw-sub-az1/2 (전용)
  │                멀티캐스트 도메인(IGMPv2): receiver-sub 연결
  ▼
VPC 10.1.1.0/24
  ├─ [tgw-sub .224/27]
  │    tgw-rtb: 10.255.255.1/32 → 활성 라우터 Gi1 (Route Server가 주입)
  │
  ├─ [router-sub .0/26]
  │    C8000V-1 Gi1 .30 ↔ RS 엔드포인트 .29 (prepend 없음 → 활성)
  │    C8000V-2 Gi1 .40 ↔ RS 엔드포인트 .41 (prepend ×3 → 대기)
  │    두 대가 GRE Tunnel100 종단 (공유 애니캐스트 Lo0 10.255.255.1, 오버레이 172.16.100.2)
  │    오버레이 eBGP(65000↔65200) + PIM ↔ CGW, LAN 172.20.51.0/24 학습 → RS에 재광고
  │    Route Server(ASN 65100)가 BGP+BFD로 활성 라우터의 Gi1 ENI 경로 주입/전환
  │
  └─ [receiver-sub .64/26]
       C8000V Gi2 .90/.100 (igmp static-group) → TGW 멀티캐스트 도메인으로 방출
       도메인이 IGMPv2 조인 멤버에게 복제 → Receiver-1 (az1), Receiver-2 (az2)

데이터 경로: 서버 → CGW(RP) → GRE → 활성 C8000V → Gi2 → TGW 도메인 → Receiver ×2
페일오버:    RS가 BFD 감지 → tgw-rtb/receiver-rtb 넥스트홉 교체 → CGW 무변경 재앵커
```

---

## GRE 오버레이 연결 플로우

터널이 성립하고 트래픽이 흐르기까지의 계층별 순서입니다. GRE는 "바깥(언더레이) 주소"와
"안(오버레이) 주소" 두 겹으로 동작합니다.

| 계층 | CGW | C8000V |
|---|---|---|
| 터널 바깥 (GRE 캡슐 주소) | 172.20.255.1 (Lo0) | 10.255.255.1 (공유 애니캐스트 Lo0) |
| 터널 안 (오버레이 P2P /30) | 172.16.100.1 | 172.16.100.2 (두 라우터가 공유) |

**① 언더레이 도달성 확보** — GRE는 유니캐스트 IP 패킷이므로, 양쪽 터널 종단이 서로의
바깥 주소에 닿기만 하면 됩니다:

- **온프렘 → AWS**: CGW가 DX BGP로 DXGW 허용 프리픽스(`10.1.1.0/24`, `10.255.255.1/32`)를 학습
- **AWS → 온프렘**: CGW가 DX로 Lo0(`172.20.255.1/32`)만 광고(`LOOPBACK-ONLY`) → TGW에 전파.
  C8000V는 정적 경로 `172.20.255.1/32 → Gi1 게이트웨이` → router-rtb의 `/32 → TGW` 경로로 되돌아감

**② GRE 캡슐 패킷의 실제 경로** (온프렘 → AWS 방향):

```
CGW: 원본 패킷을 GRE로 캡슐 (outer src 172.20.255.1, dst 10.255.255.1)
 → DX Transit VIF → DXGW
 → TGW (정적 경로 10.255.255.1/32 → VPC attachment)
 → tgw-sub의 어태치먼트 ENI 도착
 → tgw-rtb 조회: 10.255.255.1/32 → [활성 C8000V Gi1 ENI]   ← Route Server가 주입/전환
 → 활성 C8000V가 수신, GRE 역캡슐
```

이 마지막 홉(tgw-rtb의 `/32` 경로)이 **애니캐스트 설계의 핵심 조향 지점**입니다 —
두 C8000V가 같은 Lo0를 RS에 광고하고(대기 측은 prepend), RS의 best-path가 이 경로의
넥스트홉을 결정합니다.

**③ 터널 성립** — GRE는 무상태(stateless)라 협상이 없습니다. 양측의 source/destination이
서로 대칭이면 up/up이 되고, MTU 1476 / TCP MSS 1436으로 캡슐 오버헤드(24B)를 보정합니다.

**④ 오버레이 컨트롤 플레인** — 터널 안 P2P(172.16.100.0/30) 위에서:

- **eBGP 65000↔65200** (+BFD 500ms×3, hold 3/9): CGW가 `ONPREM-LAN`(172.20.51.0/24)만 광고
  → 활성 C8000V가 수신 → RS에 재광고 → receiver-rtb에 리턴 경로 주입.
  C8000V→CGW 방향은 `DENY-ALL`(아무것도 광고 안 함 — VPC 도달성은 DX 언더레이 담당),
  CGW의 `TUNNEL-IN`은 VPC CIDR·애니캐스트 /32 유입 차단(**GRE 재귀 라우팅 방지** — 터널
  목적지로 가는 경로가 터널 자신을 가리키면 터널이 다운됨)
- **PIM sparse-mode**: RP = CGW Lo0. C8000V는 `ip mroute`로 RP/소스의 RPF를 터널로 강제
  (유니캐스트 경로는 언더레이를 가리키는데 언더레이엔 PIM이 없기 때문).
  Gi2의 `igmp static-group`이 만든 (*,G) 조인이 터널을 타고 RP로 올라가고,
  소스 트래픽이 터널로 내려와 Gi2 → TGW 도메인으로 방출됩니다

**⑤ 페일오버 시 재앵커** — 활성 C8000V가 죽으면 RS가 ②의 `/32` 넥스트홉을 대기 라우터
Gi1 ENI로 교체합니다. CGW는 터널 목적지가 그대로(10.255.255.1)이므로 **아무 변경 없이**
같은 터널이 대기 라우터에 종단되고, 오버레이 BGP/PIM만 새 라우터와 재수립됩니다.
대기 라우터도 같은 오버레이 IP(172.16.100.2)를 쓰므로 CGW의 네이버 설정도 불변입니다.

---

## 1. 설정 순서

### 1.1 CloudFormation 배포

```bash
aws cloudformation create-stack --stack-name mcast-base \
  --template-body file://aws-tgw-mts.yaml
```

- **단일 패스 배포**: RS 피어까지 한 번에 생성됩니다. 피어는 Gi1 ENI만 있으면
  만들어지고(`DependsOn`), C8000V가 부팅해 BGP를 개시할 때까지 `BgpStatus: down`으로 대기합니다.
- `LatestAmiId`는 고정 AMI입니다. SSM `:latest` 참조는 스택 업데이트마다 재해석되어
  리시버 교체(고정 IP 충돌로 업데이트 실패)를 유발하므로 사용하지 않습니다.
- **스택 밖 수동 작업 2가지** (스택 재생성 시 반복 필요):
  1. TGW 기본 라우트 테이블에 애니캐스트 정적 경로:
     `aws ec2 create-transit-gateway-route --destination-cidr-block 10.255.255.1/32 --transit-gateway-route-table-id <TGW 기본 RT> --transit-gateway-attachment-id <VPC attachment>`
  2. DXGW 연결 허용 프리픽스에 `10.1.1.0/24` + `10.255.255.1/32` 포함
     (온프렘이 DX로 터널 목적지를 배우는 경로)

### 1.2 C8000V 라우터 설정

1. C8000V 인스턴스 2대를 기동하고 스택이 만든 ENI를 연결합니다
   (Gi1 = device index 0, Gi2 = device index 1, 모두 Src/Dest check 비활성 상태로 생성됨).
2. [c8000v1-conf.txt](c8000v1-conf.txt) / [c8000v2-conf.txt](c8000v2-conf.txt)를 각각 적용합니다.

핵심 포인트:
- 두 라우터가 **동일한 Loopback0(10.255.255.1/32)** 을 가지며 GRE 터널 앵커가 됩니다.
- C8000V-2만 RS 방향 광고에 `as-path prepend 65200 65200 65200` → RS가 C8000V-1을 best로 선택.
- Gi2의 `ip igmp static-group 239.1.1.1`이 (*,G)를 상시 유지 → 페일오버 시 순방향 무손실의 핵심.
- 오버레이 BGP에 `fall-over bfd` + `timers connect 5` (복구 시간 단축).
- Gi1은 DHCP 유지(ENI 고정 IP를 그대로 받음) — 정적 재설정은 세션 단절 위험만 있습니다.

### 1.3 CGW(온프렘) 라우터 설정

[cgw-conf.txt](cgw-conf.txt)를 적용합니다. DX Transit VIF BGP는 기존 설정을 유지한 채 추가합니다.

핵심 포인트:
- Lo0(172.20.255.1/32) = GRE 소스 + PIM RP. `ip pim sparse-mode` 필수.
- **터널은 1개**(Tunnel100 → 10.255.255.1). 페일오버는 AWS 쪽에서 일어나므로 CGW는 무변경.
- 광고 필터: DX(언더레이)에는 `LOOPBACK-ONLY`(Lo0만), 터널(오버레이)에는 `ONPREM-LAN`(LAN만).
  LAN 유니캐스트는 오버레이로만 흐릅니다.
- `TUNNEL-IN`이 VPC CIDR과 애니캐스트 /32의 터널 유입을 차단 — GRE 재귀 라우팅(터널 목적지가
  터널을 가리키는 순환) 방지.
- 멀티캐스트 서버가 붙는 **Gi4에 `ip pim sparse-mode` 필수** (FHR 역할 — 없으면 소스가 RP에
  등록되지 않아 (S,G)가 생기지 않음).
- 오버레이 BGP: `fall-over bfd` + `timers 3 9` (복귀 시 스테일 세션 정리 안전망).

### 1.4 EC2 리시버 설정 (IGMPv2)

[receiver-setup.txt](receiver-setup.txt) 절차를 두 리시버에 적용합니다. 요약:

```bash
# TGW는 IGMPv2만 지원 (AL2023 기본은 v3)
sudo sysctl -w net.ipv4.conf.all.force_igmp_version=2 net.ipv4.conf.default.force_igmp_version=2
# 멀티캐스트 ping 테스트용 (기본은 무응답)
sudo sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=0
# 그룹 조인 앵커 - 이 프로세스가 살아있는 동안만 TGW 멤버십 유지!
sudo systemd-run --unit=mcast-anchor --property=Restart=always \
  socat -u UDP4-RECVFROM:5001,ip-add-membership=239.1.1.1:0.0.0.0,fork STDOUT
```

- 패키지 설치(socat/iperf3)는 S3 게이트웨이 엔드포인트 경유라 인터넷 불필요.
- **iperf3는 멀티캐스트 미지원** — 수신 테스트는 socat/ping을 사용하세요.
- 접속: 키페어 없이 `aws ec2-instance-connect ssh --instance-id <id> --connection-type eice`

---

## 2. CloudFormation 배포 자원

### 2.1 VPC 서브넷 (VPC 10.1.1.0/24, 서비스별 연속 배치)

| CIDR | 서브넷 | AZ | 용도 | 서비스 수퍼넷 (SG/ACL용) |
|---|---|---|---|---|
| 10.1.1.0/27 | mcast-router-sub-az1 | 2a | C8000V-1 Gi1, RS Endpoint-1 | 라우터 10.1.1.0/26 |
| 10.1.1.32/27 | mcast-router-sub-az2 | 2c | C8000V-2 Gi1, RS Endpoint-2 | 라우터 10.1.1.0/26 |
| 10.1.1.64/27 | mcast-receiver-sub-az1 | 2a | C8000V-1 Gi2, Receiver-1, **멀티캐스트 도메인 연결** | 리시버 10.1.1.64/26 |
| 10.1.1.96/27 | mcast-receiver-sub-az2 | 2c | C8000V-2 Gi2, Receiver-2, **멀티캐스트 도메인 연결** | 리시버 10.1.1.64/26 |
| 10.1.1.224/28 | mcast-tgw-sub-az1 | 2a | **TGW 어태치먼트 전용** + EICE | TGW 10.1.1.224/27 |
| 10.1.1.240/28 | mcast-tgw-sub-az2 | 2c | TGW 어태치먼트 전용 | TGW 10.1.1.224/27 |

> 멀티캐스트 도메인은 어태치먼트 소속이 아닌 서브넷도 연결 가능합니다(실환경 검증).
> 어태치먼트는 전용 서브넷, 도메인은 리시버 서브넷으로 분리한 구성입니다.

### 2.2 ENI / IP

| IP | 용도 | 고정 여부 |
|---|---|---|
| 10.1.1.30 | C8000V-1 Gi1 (GRE 언더레이 + RS 피어-1) | ✅ 고정 (파라미터) |
| 10.1.1.40 | C8000V-2 Gi1 (GRE 언더레이 + RS 피어-2) | ✅ 고정 (파라미터) |
| 10.1.1.90 | C8000V-1 Gi2 (멀티캐스트 방출) | ✅ 고정 (파라미터) |
| 10.1.1.100 | C8000V-2 Gi2 (멀티캐스트 방출) | ✅ 고정 (파라미터) |
| 10.255.255.1 | C8000V 공유 애니캐스트 Lo0 (GRE 앵커, RS가 활성 라우터로 조향) | ✅ 고정 (라우터 설정) |
| (자동) | Receiver-1/2 EC2 | ❌ 자동 할당 |
| (자동) | RS Endpoint-1/2 (BGP 상대 주소, Output으로 확인) | ❌ 자동 할당 (지정 불가) |
| (자동) | TGW 어태치먼트 ENI ×2, EICE | ❌ 자동 할당 (지정 불가) |

### 2.3 BGP ASN / 피어링

| 세션 | A측 | B측 | 감지 | 역할 |
|---|---|---|---|---|
| DX 언더레이 | CGW 65000 (169.254.96.54/.62) | DXGW 65300 (.49/.57) | BFD | Lo0(GRE 소스)만 광고, VPC/애니캐스트 프리픽스 수신 |
| RS 피어링 ×2 | C8000V-1/2 65200 (Gi1 .30/.40) | Route Server 65100 (RSE, 자동 IP) | **BFD 300ms×3** | 애니캐스트 Lo0 + LAN을 RS에 광고 → RS가 tgw-rtb/receiver-rtb에 주입. **페일오버 트리거** |
| GRE 오버레이 | CGW 65000 (172.16.100.1) | 활성 C8000V 65200 (172.16.100.2 공유) | BFD 500ms×3, hold 3/9 | LAN(172.20.51.0/24)을 AWS로 전달. 대기 라우터는 세션 미성립(애니캐스트 특성) |
| TGW | TGW 65400 | DXGW 65300 | — | DXGW 연동(허용 프리픽스 광고) |

---

## 3. 검증 가이드

### 3.1 C8000V 상태 확인

```
show ip bgp summary                  ! RS(.29/.41) Established, 오버레이는 활성 라우터만
show bfd neighbors                   ! RS·오버레이 BFD Up
show ip pim neighbor                 ! Tunnel100에 CGW(172.16.100.1)
show ip mroute 239.1.1.1 count       ! (S,G) 포워딩 카운터 증가 = 트래픽 흐름 중
show interfaces Tunnel100 | include packets
```

### 3.2 CGW 상태 확인

```
show ip route 10.255.255.1           ! DX(169.254.96.x) 경유 BGP 경로
show interface Tunnel100             ! up/up
show ip bgp summary                  ! 172.16.100.2 Established (PfxRcd 0 정상 - DENY-ALL)
show ip pim neighbor                 ! Tunnel100에 활성 C8000V
show ip pim interface                ! Gi4, Lo0, Tunnel100 모두 표시
show ip mroute 239.1.1.1             ! 소스 송신 시 (S,G) + OIL Tunnel100
show ip pim tunnel                   ! Tu0(Encap)/Tu1(Decap) = PIM 자동 생성, 정상
```

### 3.3 VPC Route Server 상태 확인

```bash
aws ec2 describe-route-server-peers \
  --query 'RouteServerPeers[].{Peer:PeerAddress,Bgp:BgpStatus.Status,Bfd:BfdStatus.Status}'
# 기대: 두 피어 모두 BGP/BFD up

aws ec2 get-route-server-routing-database --route-server-id <rs-id>
# 기대: 10.255.255.1/32 - C8000V-1(AS×1, in-fib/installed), C8000V-2(AS×4, in-rib)
#       172.20.51.0/24 - 활성 라우터에서 in-fib

aws ec2 describe-route-tables --filters "Name=tag:Name,Values=mcast-tgw-rtb"
# 기대: 10.255.255.1/32 -> 활성 C8000V Gi1 ENI
```

### 3.4 종단 테스트 (멀티캐스트 ping)

온프렘 멀티캐스트 서버에서:

```bash
ping -t 8 239.1.1.1        # TTL 필수 (기본 1이면 첫 라우터에서 소멸)
```

조인된 리시버들이 각자 유니캐스트로 응답합니다 — seq당 응답 2개(두 번째는 `DUP!`)가 정상,
첫 1~2개 seq 무응답은 (S,G) 트리 형성 시간으로 정상입니다.
TGW 멤버 확인: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <id>`

---

## 4. Failover 테스트

### 4.1 C8000V 라우터 장애 (실측 완료)

절차: 온프렘에서 `ping -D -t 8 239.1.1.1 -i 0.1`을 켠 상태로 활성 C8000V 인스턴스를 중지.

동작 순서: RS BFD 감지(~1초) → RIB 철회 → prepend된 C8000V-2 경로가 best 승격 →
tgw-rtb/receiver-rtb 넥스트홉 교체 → GRE가 대기 라우터로 재앵커(CGW 무변경) →
오버레이 BGP 재수립 → LAN 리턴 경로 복구.

실측 결과 (BFD + 타이머 튜닝 적용 후):

| 항목 | 결과 |
|---|---|
| VPC 경로 전환 | BFD 감지 후 ~1초 (그레이스풀 중지 시 make-before-break로 사실상 0) |
| **순방향 멀티캐스트 손실** | **0건** (장애+복귀 전 구간, 0.1초 간격 5,114 패킷 검증) — Gi2 static-group의 사전 (*,G) 덕 |
| 리턴(응답) 경로 전환 | ≤1초 (오버레이 BFD + connect 5) |
| 복귀(failback) 리턴 공백 | ~90초 — 부팅 직후 선점이 오버레이 BGP 준비보다 먼저 발생 (구조적 한계). 복귀는 계획 작업으로 취급 권장. 복귀 중에도 멀티캐스트는 무손실 |

복구 후 원상 복귀는 자동입니다(선점) — C8000V-1이 부팅하면 별도 작업 없이 활성으로 돌아옵니다.

### 4.2 Direct Connect 장애 (Bring down BGP)

DX Transit VIF 2개 중 하나의 BGP를 내려 언더레이 이중화를 검증합니다.

```
! CGW에서 (VIF-1 경로 차단)
router bgp 65000
 neighbor 169.254.96.49 shutdown
```

관찰 포인트:
- CGW: `show ip bgp summary` — .49 Idle(Admin), .57만 유지. `show ip route 10.255.255.1` — .57 넥스트홉으로 전환
- GRE/멀티캐스트: 언더레이 경로만 바뀌므로 **터널·PIM·오버레이 BGP는 유지**되어야 함 (터널 플랩 없이 RTT만 변화)
- 예상 손실: BFD(300ms×3) 감지 ~1초 내외
- 복구: `no neighbor 169.254.96.49 shutdown`

> 물리 단선 시나리오는 VIF 서브인터페이스 shutdown(`interface Gi1.351` → `shutdown`)으로 모사할 수 있습니다.

---

## 5. 모니터링 및 트러블슈팅

### 5.1 C8000V

| 증상 | 확인 | 원인/조치 |
|---|---|---|
| RS BGP down | `show ip bgp summary`, RS 피어 상태(CLI) | SG(c8000v-sg)가 VPC 전체 허용인지, Gi1 ENI 연결 상태 |
| (S,G) 없음 | `show ip mroute`, `show ip rpf 172.20.255.1` | RPF가 Tunnel100인지 (`ip mroute` 정적 항목 필수 — 언더레이엔 PIM 없음) |
| 터널 up인데 트래픽 없음 | `show ip mroute 239.1.1.1 count` | 카운터 정지면 CGW 쪽 (S,G)/Register 확인 |
| 페일오버 후 로그 | `show logging \| include %BGP\|%BFD` | ADJCHANGE/BFD 이벤트 타임라인으로 전환 시각 분석 |
| 접속 | EICE: `aws ec2-instance-connect ssh` 또는 SSH ProxyCommand `open-tunnel` | pem이 `error in libcrypto`면 base64 본문을 DER로 디코드 후 재생성 |

### 5.2 CGW

| 증상 | 확인 | 원인/조치 |
|---|---|---|
| 터널 down/flap | `show interface Tunnel100`, `show ip route 10.255.255.1` | 재귀 라우팅 여부 — 터널로 10.255.255.1이나 10.1.1.0/24를 배우면 안 됨(TUNNEL-IN 확인). DXGW 허용 프리픽스에 /32 포함 여부 |
| (S,G) 안 생김 | `show ip mroute 239.1.1.1`, `show ip pim interface` | **Gi4 PIM 누락**이 최다 원인. Tu0/Tu1(Register 터널) 존재 확인 |
| (*,G)만 있고 OIL 비어있음 | `show ip mroute` | C8000V의 PIM Join 미도달 — 터널/오버레이 상태 확인 |
| `%IGMP-3-QUERY_INT_MISMATCH ... 0.0.0.0` | — | LAN 스누핑 장비의 프록시 쿼리. 무해(쿼리어 선출에 영향 없음) |
| 오버레이 BGP flap | `show ip bgp neighbors 172.16.100.2` | hold 3/9은 의도된 설정. 반복 플랩이면 DX 경로 품질 확인 |

### 5.3 AWS / 리시버 공통

| 증상 | 확인 | 원인/조치 |
|---|---|---|
| TGW 그룹 멤버 0 | `search-transit-gateway-multicast-groups` | 리시버 조인 앵커(socat) 사망 → 재기동(systemd 권장). IGMPv3로 조인 중인지(`force_igmp_version=2`) |
| 리시버가 응답 안 함 (ping) | 리시버 `tcpdump -ni ens5 icmp` | 패킷 도착 O + 응답 X → `icmp_echo_ignore_broadcasts=0`. 도착 X → 리시버 SG(ICMP/UDP 소스에 온프렘 대역), TGW 멤버십 |
| RS 경로 미주입 | `get-route-server-routing-database` | in-rib인데 미설치면 해당 RTB의 propagation 활성 여부 |
| 스택 업데이트 실패 (IP 충돌) | 변경 세트의 Replacement 컬럼 | 고정 IP 자원의 교체는 생성→삭제 순서라 충돌 — AMI 고정 유지, IP 변경은 재생성으로 |
