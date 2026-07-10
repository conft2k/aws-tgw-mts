# tgw-mts — Hybrid Multicast HA Base Infrastructure

온프레미스 멀티캐스트 소스를 GRE + TGW 멀티캐스트 도메인으로 AWS까지 전달하는
하이브리드 멀티캐스트 HA 기반 인프라입니다. `aws-tgw-mts.yaml` 하나로
VPC, 멀티캐스트 TGW, IGMPv2 도메인, VPC Route Server(BGP+BFD), EICE,
리시버 EC2까지 배포합니다. **C8000V 라우터 인스턴스는 이 스택에 없습니다**
(별도 배포 후 2단계 업데이트).

```
온프렘 라우터 (ASN 65000, Lo0 192.168.255.1, CGW)
   │  DX / Transit VIF
   ▼
DXGW (ASN 65300, 별도 연결)
   │
   ▼
TGW 65400 (MulticastSupport) ──── TGW Multicast Domain (IGMPv2)
   │  VPC attachment = Receiver 서브넷 (AZ당 1개 제한 때문)
   ▼
Prod VPC 10.1.1.0/24
├─ RouterSubnet1   10.1.1.0/27   C8000V-1 Gi1(.10) + RS Endpoint-1 + EICE
├─ ReceiverSubnet1 10.1.1.32/27  C8000V-1 Gi2(.40) + Receiver-1(.50)  [mcast]
├─ RouterSubnet2   10.1.1.64/27  C8000V-2 Gi1(.90) + RS Endpoint-2
└─ ReceiverSubnet2 10.1.1.96/27  C8000V-2 Gi2(.110) + Receiver-2(.120) [mcast]

GRE:  온프렘 Lo0 ↔ C8000V Gi1 (언더레이: RouterRouteTable → TGW → DXGW)
BGP:  C8000V(65200) Gi1 ↔ Route Server(65100) Endpoint, BFD 페일오버
멀티캐스트: C8000V Gi2 → TGW 도메인 → Receiver
```

## IP / ASN 플랜

| 항목 | 값 | 비고 |
|---|---|---|
| C8000V-1 Gi1 | 10.1.1.10 | RS Peer-1 (AZ1), `Router1PeerAddress` |
| C8000V-1 Gi2 | 10.1.1.40 | 멀티캐스트 인터페이스 (AZ1), `Router1Gi2Address` |
| C8000V-2 Gi1 | 10.1.1.90 | RS Peer-2 (AZ2), `Router2PeerAddress` |
| C8000V-2 Gi2 | 10.1.1.110 | 멀티캐스트 인터페이스 (AZ2), `Router2Gi2Address` |
| Receiver-1 | 10.1.1.50 | AZ1, `Receiver1Address` |
| Receiver-2 | 10.1.1.120 | AZ2, `Receiver2Address` |
| 온프렘 Lo0 (GRE 소스) | 192.168.255.1/32 | RouterRouteTable에서 TGW로 라우팅, `OnPremGreSourceCidr` |
| 온프렘 라우터 ASN | 65000 | 온프렘 장비에서 설정 |
| Route Server ASN | 65100 | `RouteServerAsn` |
| C8000V ASN | 65200 | `RouterBgpAsn` |
| DXGW ASN | 65300 | 별도 생성 시 지정, TGW ASN과 달라야 함 |
| TGW ASN | 65400 | `TgwAsn` |

RS Endpoint ENI IP, TGW 어태치먼트 ENI IP는 자동 할당 — 스택 Outputs 확인.

## 배포 절차

### 0. 사전 조건

- DX 연결 + Transit VIF, 온프렘 라우터 (Lo0 = GRE 소스, 벤더 무관 — GRE+PIM+BGP 지원 필요)
- 리전: ap-northeast-2 (AZ 기본값 2a / 2c)

### 1. 1단계 — 기반 스택 생성 (`CreateRouteServerPeers=No`)

```bash
aws cloudformation deploy \
  --region ap-northeast-2 \
  --stack-name mcast-base \
  --template-file aws-tgw-mts.yaml
# 파라미터 전부 기본값 사용 시 옵션 생략 가능
```

Outputs에서 `RouteServerEndpoint1EniAddress`(RSE-1-IP),
`RouteServerEndpoint2EniAddress`(RSE-2-IP), `TransitGatewayId` 를 기록해 둡니다.

### 2. DXGW ↔ TGW 연결 (콘솔/CLI 별도 작업)

```bash
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id <dxgw-id> \
  --gateway-id <TransitGatewayId 출력값> \
  --add-allowed-prefixes-to-direct-connect-gateway cidr=10.1.1.0/24
```

allowed prefixes에 최소 VPC CIDR(10.1.1.0/24)을 포함해야 온프렘에서
C8000V Gi1(GRE 목적지)에 도달합니다.

### 3. C8000V 인스턴스 배포 (별도 스택/수동)

- Gi1/Gi2 ENI는 기반 스택이 **고정 IP + Source/Dest Check 비활성화 +
  라우터 SG 연결** 상태로 미리 생성해 둠 — Outputs의
  `Router1Gi1EniId`/`Router1Gi2EniId`/`Router2Gi1EniId`/`Router2Gi2EniId` 사용
- **ENI는 launch 시점에 지정** (배포 후 교체 아님):
  - Gi1 ENI = device index 0 — **primary ENI는 분리·교체가 불가능**하므로
    반드시 RunInstances/라우터 스택에서 `NetworkInterfaceId`로 지정해 시작
  - Gi2 ENI = device index 1 — launch 시 함께 지정이 가장 깔끔.
    실행 중 hot-attach는 IOS-XE가 reload 전까지 Gi2를 인식 못할 수 있으니
    부득이하면 인스턴스 중지 상태에서 attach하거나 attach 후 reload
- ENI가 기반 스택 소유라 라우터 인스턴스를 종료·재생성해도 IP/SG/멀티캐스트
  그룹 등록(ENI 단위)이 유지됨 — 같은 ENI로 다시 launch하면 온프렘 GRE 설정,
  RS Peer 변경 불필요. 인스턴스 타입은 ENI 2개 이상 지원 필요(c5.large 등 충족)
- 라우터 SG(`RouterSecurityGroupId`)는 VPC 대역·온프렘 대역(`OnPremCidr`,
  기본 192.168.0.0/16) 인바운드 전체 + TGW IGMP query(proto 2)를 허용
  (GRE 47, BGP 179, BFD 3784-3785, EICE SSH 포함)
- IOS-XE 핵심 설정: GRE 터널(소스 Gi1, 목적지 온프렘 Lo0), 터널에 PIM,
  Gi2에 PIM + (AWS→온프렘 필요 시) `ip igmp join-group`,
  RPF용 `ip mroute <온프렘 소스대역> Tunnel0`,
  RS 피어링 `router bgp 65200` + neighbor RSE-1/2-IP, BFD 활성화
- 피어 IP가 같은 서브넷이므로 BGP는 eBGP 직접 연결 — BGP가 안 올라오면
  IOS-XE에서 피어 IP로 가는 경로(connected/스태틱)를 먼저 확인

### 4. 2단계 — RS Peer + 전파 활성화

C8000V가 Gi1 IP로 살아있는 것을 확인한 뒤:

```bash
aws cloudformation deploy \
  --region ap-northeast-2 \
  --stack-name mcast-base \
  --template-file aws-tgw-mts.yaml \
  --parameter-overrides CreateRouteServerPeers=Yes
```

RS Peer 2개(BFD)와 Receiver 라우트 테이블 전파가 생성됩니다.
Router 라우트 테이블은 스태틱 유지.

## 검증

```bash
# RS 피어 BGP/BFD 상태
aws ec2 describe-route-server-peers --region ap-northeast-2 \
  --query 'RouteServerPeers[].{IP:PeerAddress,BGP:BgpStatus.Status,BFD:BfdStatus.Status}'

# Receiver RT에 전파된 온프렘 경로
aws ec2 describe-route-tables --region ap-northeast-2 \
  --filters Name=tag:Name,Values=mcast-rtb-receiver-rs-propagated

# 멀티캐스트 그룹 멤버 (리시버가 IGMP JOIN 후)
aws ec2 search-transit-gateway-multicast-groups --region ap-northeast-2 \
  --transit-gateway-multicast-domain-id <MulticastDomainId 출력값>
```

리시버 접속(EICE) 및 수신 테스트:

```bash
aws ec2-instance-connect ssh --instance-id <ReceiverInstance1Id> \
  --connection-type eice --region ap-northeast-2

# (필수) IGMPv2 강제 — TGW는 IGMPv2 리포트만 인식하는데 리눅스 기본은
# IGMPv3이라 조인이 TGW 도메인에 등록되지 않음. /proc/net/igmp 에서
# ens5가 V2인지 확인 (부팅 후에도 유지하려면 /etc/sysctl.d 에 저장)
sudo sysctl -w net.ipv4.conf.all.force_igmp_version=2 \
  net.ipv4.conf.ens5.force_igmp_version=2

# 리시버에서 그룹 조인 + 수신 (예: 239.1.1.1:5000)
python3 - <<'EOF'
import socket, struct
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('', 5000))
mreq = struct.pack('4sl', socket.inet_aton('239.1.1.1'), socket.INADDR_ANY)
s.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)
print('joined 239.1.1.1, waiting...')
while True: print(s.recvfrom(1500))
EOF
```

## 삭제

1. `CreateRouteServerPeers=No` 로 스택 업데이트 (피어/전파 먼저 제거)
2. DXGW association 해제, C8000V 인스턴스 스택 삭제
   (ENI가 in-use면 기반 스택 삭제가 실패하므로 라우터를 먼저 제거)
3. `aws cloudformation delete-stack --stack-name mcast-base`

## 설계 노트 / 제약

- **멀티캐스트 도메인 연결은 어태치먼트 소속 서브넷만 가능**하고 어태치먼트는
  AZ당 서브넷 1개라, 어태치먼트를 Receiver 서브넷에 배치했습니다. Router
  서브넷은 같은 AZ 어태치먼트 덕분에 라우트 테이블 경유로 TGW에 도달합니다.
- 멀티캐스트는 DX/VPN/피어링 위로 직접 지원되지 않음 → GRE로 우회하는 설계.
- TGW가 IGMP 쿼리어(proto 2, src 0.0.0.0/32) — 리시버 SG에 이미 허용됨.
- TGW는 IGMPv2만 지원 — 리시버(리눅스)는 `force_igmp_version=2` 필수
  (기본 IGMPv3 리포트는 무시되어 그룹 멤버로 등록되지 않음).
- EICE는 AZ1 1개로 전 AZ 접속(cross-AZ라 `PreserveClientIp=false`).
- 2-byte ASN 사용, DXGW ASN은 TGW(65400)와 달라야 함.
