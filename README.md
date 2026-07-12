# Amazon VPC Route Server 기반 멀티캐스트 GRE 앵커 이중화

**GRE 애니캐스트 앵커 × TGW 멀티캐스트 도메인 × VPC Route Server(BGP+BFD) — 실측으로 검증한 하이브리드 멀티캐스트 아키텍처**

> English version: [README.en.md](README.en.md)

온프레미스 멀티캐스트 소스를 **GRE(애니캐스트 앵커) + TGW 멀티캐스트 도메인(IGMPv2)**으로
AWS 리시버까지 전달하는 하이브리드 멀티캐스트 HA 구성입니다.
장애 전환은 **VPC Route Server(BGP+BFD) + AS-path prepending**이 VPC 라우팅을
전환하는 방식으로, 온프레미스(CGW)는 아무 변경 없이 페일오버됩니다.

## 배경

### 문제: 하이브리드 멀티캐스트의 공백

시세 분배, 방송 컨트리뷰션, 관제 시스템처럼 멀티캐스트에 의존하는 워크로드를
AWS로 확장하려 하면 두 개의 공백을 만납니다.

1. **AWS 네트워크는 멀티캐스트를 기본 전달하지 않습니다.** TGW 멀티캐스트
   도메인이 유일한 네이티브 수단이지만 이는 VPC 안에서만 동작하고,
   Direct Connect/VPN은 멀티캐스트를 운반하지 않으므로 온프레미스 PIM
   도메인과 직접 연동되지 않습니다. → 해법: **GRE 터널**로 멀티캐스트를
   VPC 내부의 EC2 라우터(C8000V)까지 끌어와 TGW 도메인에 방출.
2. 그 순간 새 문제가 생깁니다 — **GRE를 받는 EC2 라우터가 단일 장애점**이
   됩니다. 라우터를 2대로 늘려도, VPC 라우팅 테이블은 정적이라 "장애를
   감지하고 경로를 대기 라우터로 바꿔주는 무언가"가 따로 필요합니다.

### 기존 해법의 한계

Route Server 이전에는 이 "무언가"를 직접 만들어야 했습니다: CloudWatch
알람 + Lambda로 라우팅 테이블 API를 호출해 넥스트홉을 교체하거나, ENI를
옮겨 달거나, GWLB를 끼워 넣는 방식입니다. 공통 한계는 감지가 수십 초~분
단위라는 것, 장애 판단·전환 로직이 전부 커스텀 코드라 검증과 유지보수
부담이 크다는 것, 그리고 활성/대기 우선순위 같은 라우팅 정책을 표현할
표준 수단이 없다는 것입니다.

### VPC Route Server의 가치 — 이 구성의 핵심

**Amazon VPC Route Server는 VPC 라우팅 테이블에 표준 라우팅 프로토콜
시맨틱을 도입합니다.** EC2 어플라이언스와 eBGP를 맺고, 광고받은 경로를
VPC 라우팅 테이블에 동적으로 주입·철회합니다. 이 구성에서 그 가치는:

| 가치 | 이 구성에서의 의미 | 실측 |
|---|---|---|
| **BFD 서브초 장애 감지** | 커스텀 코드 0줄로 어플라이언스 헬스체크 | VPC 경로 전환 ~1초 |
| **표준 BGP 정책** | AS-path prepend만으로 활성/대기 표현 — 두 라우터가 같은 설정, 정책 한 줄 차이 | 페일오버 자동, 복귀도 자동(선점) |
| **애니캐스트와 결합** | 공유 Lo0 /32의 넥스트홉만 교체 → GRE가 대기 라우터로 재앵커 | **온프레미스(CGW) 설정 무변경** 페일오버 |
| **Persist Routes** | 피어 플랩 시 마지막 경로 유지로 블랙홀 방지 | 재부팅 3회 반복에서 오탐 전환 없음 |
| **양쪽 라우팅 테이블 동시 주입** | tgw-rtb(GRE 유입)와 receiver-rtb(리턴 경로)를 한 세션으로 갱신 | 단방향 멀티캐스트 **무손실**, 왕복 ~9초 |

요약하면, Route Server가 없을 때 "Lambda로 짜야 했던 페일오버 오케스트레이션"이
BGP·BFD·prepend라는 30년 된 검증된 프로토콜 동작으로 대체됩니다. 네트워크
엔지니어에게 익숙한 도구로 VPC 라우팅을 다룰 수 있다는 것 — 그것이 이 구성이
성립하는 이유입니다.

### 어떤 환경에 효과적인가 — MTS류 실시간 시세 분배

이 아키텍처가 특히 효과적인 곳은 **증권사 MTS/HTS를 받치는 실시간 시세 분배**처럼,
온프레미스의 멀티캐스트 피드를 클라우드의 여러 수신 서비스로 끊김 없이 넓혀야
하는 환경입니다.

- **트래픽 패턴이 정확히 일치** — 거래소 시세 피드는 UDP 멀티캐스트 **단방향
  스트림**입니다. 이 구성의 실측 결과(장애·복귀 전 과정 수신 공백 없음,
  4.1 단방향 실측)가 그대로 적용되는 패턴입니다.
- **피드 소스를 건드릴 수 없음** — 시세 수신 회선·장비 구성은 규제와 계약상
  변경이 어렵습니다. 페일오버가 전적으로 AWS 쪽에서 일어나고 온프레미스(CGW)는
  무변경이라는 이 구성의 특성이 그대로 유효합니다.
- **수신자는 클라우드에서 늘어남** — 시세를 소비하는 체결·호가·알림 서비스가
  AWS에서 수평 확장됩니다. TGW 도메인 덕에 수신자 추가는 IGMP 조인 하나로
  끝나고, 복제 부하는 TGW가 담당합니다 (라우터 부하 불변).
- **장 중 무중단 요구에 실측 근거로 답함** — 계획 전환·복귀는 무손실, 비계획
  장애도 BFD 감지(~1초)가 상한입니다. "장 시간 중 라우터를 교체할 수 있는가"
  같은 운영 질문에 측정값으로 답할 수 있습니다.

같은 이유로 방송 컨트리뷰션 피드, 관제·텔레메트리 분배처럼 "단방향 멀티캐스트
+ 소스 불가침 + 클라우드 측 수신자 확장" 조합의 워크로드에 일반화됩니다.

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

조인된 리시버들이 각자 유니캐스트로 응답합니다. 실측 출력(2026-07-12):

```
labuser@vm51:~$ ping -t 8 239.1.1.1
PING 239.1.1.1 (239.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.84: icmp_seq=1 ttl=125 time=33.1 ms
64 bytes from 10.1.1.116: icmp_seq=1 ttl=125 time=34.4 ms
64 bytes from 10.1.1.84: icmp_seq=2 ttl=125 time=4.36 ms
64 bytes from 10.1.1.116: icmp_seq=2 ttl=125 time=5.16 ms
```

판독 요점:

- **seq당 응답 2개** = 리시버 2대가 각자 응답. 멀티캐스트 대상 ping은 다중 응답이
  전제라 `DUP!` 표시는 붙지 않습니다 (유니캐스트 ping의 중복 응답과 다름).
- **ttl=125** = 소스 기준 3홉(CGW → C8000V → 리시버, 128−3). 경로가 설계보다 길거나
  짧으면 이 값이 달라집니다.
- **첫 응답만 RTT 큼(33ms → 이후 4~5ms)** = RP-tree 경유 후 SPT 전환. (S,G) 트리가
  이미 형성된 상태에서 재시작하면 첫 seq부터 바로 응답하고, 콜드 스타트면
  첫 1~2개 seq 무응답이 정상입니다.

TGW 멤버 확인: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <id>`

### 3.5 종단 테스트 (iperf2 UDP 스트림 — 실제 워크로드형)

ping은 왕복 확인용 간이 테스트이고, 배포형 워크로드 검증과 손실 측정은 단방향
UDP 스트림으로 합니다. **iperf2만 멀티캐스트를 지원**합니다 (iperf3 불가).

온프렘 서버에서:

```bash
iperf -c 239.1.1.1 -u -T 8 -t 600 -i 1 -b 1M -l 1200
```

- `-T 8`: TTL 필수 (기본 1이면 CGW에서 소멸)
- `-l 1200`: **데이터그램 크기 필수** — 기본 1470B는 GRE 오버헤드로 경로 MTU를
  넘어 전량 소멸합니다 (5.2 참조). 1200B면 GRE 포함 1252B로 여유
- `-b 1M -l 1200` = 초당 약 109패킷 → 손실 측정 해상도 ~9ms

체크포인트별 확인 (2026-07-12 실측값):

```
! C8000V(활성) - 초당 패킷/평균 크기/kbps가 송신율과 일치해야 함
show ip mroute 239.1.1.1 count
  Source: 172.20.51.10/32, Forwarding: 73811/109/1266/1105, Other: 73813/2/0
                                       ↑누적  ↑pps ↑평균B ↑kbps
```

리시버에서 (mcast-recv.py가 5001 포트 수신 중이어야 함 — receiver-setup.txt):

```bash
tail -f /home/ec2-user/mcast-recv.log        # 패킷 도착 로그
```

수신 공백(손실 구간) 정밀 측정 — tcpdump 타임스탬프로 기록 후 최대 간격 계산:

```bash
sudo sh -c 'nohup tcpdump -lni ens5 -tt "udp and dst 239.1.1.1" > /tmp/mcast-times.txt 2>/dev/null &'
# 테스트 후:
awk '{if(prev){g=$1-prev; if(g>max){max=g; at=prev}} prev=$1}
     END {printf "max_gap=%.3fs at %.3f\n", max, at}' /tmp/mcast-times.txt
```

실측 예 (4.1 재부팅 테스트 전 구간): 리시버1 `max_gap=0.056s`, 리시버2 `max_gap=0.053s`
— 페일오버·페일백 포함 무손실.

---

## 4. Failover 테스트

### 4.1 C8000V 라우터 장애 (실측 완료)

절차: 온프렘에서 `ping -D -t 8 239.1.1.1 -i 0.1`을 켠 상태로 활성 C8000V 인스턴스를 중지.

동작 순서: RS BFD 감지(~1초) → RIB 철회 → prepend된 C8000V-2 경로가 best 승격 →
tgw-rtb/receiver-rtb 넥스트홉 교체 → GRE가 대기 라우터로 재앵커(CGW 무변경) →
오버레이 BGP 재수립 → LAN 리턴 경로 복구.

실측 결과 (BFD + 타이머 튜닝 적용 후):

| 항목 | 결과 | UDP 멀티캐스트 손실 시간 |
|---|---|---|
| VPC 경로 전환 | BFD 감지 후 ~1초 (그레이스풀 중지 시 make-before-break로 사실상 0) | **0초** (전환 중 수신 공백 없음) |
| **순방향 멀티캐스트 손실** | **0건** (장애+복귀 전 구간, 0.1초 간격 5,114 패킷 검증) — Gi2 static-group의 사전 (*,G) 덕 | **0초** (재부팅 재실측: 전 구간 최대 간격 56ms = 지터 수준) |
| 리턴(응답) 경로 전환 | ≤1초 (오버레이 BFD + connect 5) | 해당 없음 (단방향 스트림은 리턴 경로 불필요) |
| 복귀(failback) 리턴 공백 | ~90초 — 부팅 직후 선점이 오버레이 BGP 준비보다 먼저 발생 (구조적 한계). 복귀는 계획 작업으로 취급 권장 | **0초** (복귀 중에도 무손실 — 재부팅 재실측으로 재확인) |

복구 후 원상 복귀는 자동입니다(선점) — C8000V-1이 부팅하면 별도 작업 없이 활성으로 돌아옵니다.

#### 재부팅 방식 재검증 (2026-07-12, ping 1초 간격, 3회 반복)

활성 C8000V-1을 **reboot**(그레이스풀 중지가 아닌 비계획 장애 시나리오)하고,
온프렘 ping(1초 간격)과 AWS API 폴링 타임스탬프로 전 과정을 실측한 결과
(아래 수치는 페일오버·페일백이 한 세션에 모두 담긴 회차 기준):

| 시각 (UTC) | 경과 | 이벤트 | ping |
|---|---|---|---|
| 11:20:20 | 0초 | reboot API 실행 | 정상 |
| 11:20:22 | 2초 | 인터페이스 다운, 단절 시작 | seq 40부터 손실 |
| 11:20:31 | 11초 | 페일오버 완료 (BFD 감지 → RS 전환 → GRE 재앵커 → C8000V-2 오버레이 수립) | seq 48 재개 — **공백 9.2초** |
| 11:23:12 | 2분 51초 | C8000V-1 부팅 완료, RS BGP 재수립 | 정상 (C8000V-2 경유) |
| 11:24:22 | 4분 2초 | 페일백 선점(자동): 넥스트홉 → C8000V-1 | seq 279부터 손실 |
| 11:25:54 | 5분 33초 | C8000V-1 오버레이 BGP 재수립 | seq 368 재개 — **공백 92.1초** |

ping 통계 정합: 372 송신 / 손실 97 = 페일오버 8개 + 페일백 89개와 정확히 일치.
응답이 온 모든 seq는 리시버 2대 모두 응답(수신 275 + 중복 275).

판독 요점:

- **비계획 장애(reboot)의 페일오버 단절은 ~9초** — 그레이스풀 중지의
  make-before-break(무손실/≤1초, 위 표)와 구분해서 기대치를 잡을 것.
- **페일백 공백 ~90초는 구조적이며 반복 재현됨** (1차 88초, 3차 92초) —
  단절 시작은 RS의 경로 선점, 종료는 복귀 라우터의 오버레이 BGP 재수립.
  선점이 오버레이 준비보다 항상 먼저 오므로 이 공백은 튜닝으로 제거 불가,
  복귀를 계획 작업으로 취급해야 하는 근거.
- **페일백 선점은 RS BGP 재수립 후 66~83초에 발생** (3회 관측) —
  PersistRoutes(2분) 설정값 그대로 대기하지 않으므로 복귀 시점 예측에 주의.
- C8000V 부팅(reboot → RS BGP 재수립)은 3회 모두 **약 3분**.

#### 단방향 스트림 실측 (2026-07-12, UDP 1Mbps·초당 109패킷·1200B, 리시버 tcpdump)

위 ping 측정은 왕복(순방향 멀티캐스트 + 리턴 유니캐스트) 기준입니다. 실제 워크로드에
가까운 **단방향 UDP 스트림**(iperf)을 흘리고 리시버 2대의 tcpdump 타임스탬프로 같은
재부팅 시나리오를 측정한 결과:

| 구간 | 실측 (수신 공백) |
|---|---|
| 페일오버 (재부팅) | **측정 불가 수준 (공백 없음)** |
| 페일백 (자동 선점) | **공백 없음** |
| 전 구간 최대 수신 간격 | 리시버1 56ms / 리시버2 53ms — 전환 시점과 무관한 지터 수준 |

판독:

- **ping의 9.2초/92초 공백은 전부 리턴 유니캐스트 경로 때문**이었음이 확정.
  순방향 멀티캐스트는 (*,G) 사전 형성(igmp static-group)과 정적 mroute RPF 덕에
  오버레이 BGP와 무관하게 경로 전환 즉시 흐른다. 단방향 배포형 워크로드
  (영상/시세 피드 등)는 장애·복귀 전 과정에서 **사실상 무손실**.
- EC2 reboot은 IOS-XE가 BGP를 정상 종료(graceful close)하므로 RS가 BFD 만료를
  기다리지 않고 즉시 철회 → make-before-break가 성립한 것. graceful close가
  없는 급사(커널 패닉, 인스턴스 강제 종료 등)라면 BFD 감지(~1초)가 공백 상한.
- 양방향(요청-응답형) 워크로드의 기대치는 ping 실측(장애 ~9초, 복귀 ~90초)을 따른다.

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
| **큰 UDP만 소멸, ping은 통과** | C8000V `show ip mroute <grp> count` — 소스 카운터 0 | **GRE MTU 초과** (실측: iperf 기본 1470B는 전량 소멸, 1200B는 통과). 페이로드 + IP/UDP 28B + GRE 24B가 경로 MTU 1500을 넘으면 언더레이에서 드롭 — 멀티캐스트 앱 페이로드는 **1400B 이하** 권장 (iperf는 `-l 1200`) |

### 5.3 AWS / 리시버 공통

| 증상 | 확인 | 원인/조치 |
|---|---|---|
| TGW 그룹 멤버 0 | `search-transit-gateway-multicast-groups`, 리시버 `ip maddr show ens5`에 그룹 없음 | 리시버 조인 앵커(socat/mcast-recv.py) 사망 → 재기동(systemd 권장). **인스턴스가 교체되면 스크립트째 사라지므로 재설정 필요** (receiver-setup.txt). IGMPv3로 조인 중인지(`force_igmp_version=2`) |
| 리시버가 응답 안 함 (ping) | 리시버 `tcpdump -ni ens5 icmp` | 패킷 도착 O + 응답 X → `icmp_echo_ignore_broadcasts=0`. 도착 X → 리시버 SG(ICMP/UDP 소스에 온프렘 대역), TGW 멤버십 |
| RS 경로 미주입 | `get-route-server-routing-database` | in-rib인데 미설치면 해당 RTB의 propagation 활성 여부 |
| 스택 업데이트 실패 (IP 충돌) | 변경 세트의 Replacement 컬럼 | 고정 IP 자원의 교체는 생성→삭제 순서라 충돌 — AMI 고정 유지, IP 변경은 재생성으로 |

---

## Appendix. 라우터 상태 샘플 출력 (실측)

정상 운용 상태(C8000V-1 활성 / C8000V-2 대기, 2026-07-12 캡처)에서 EICE 경유로 수집한
실제 출력입니다. 운영 중 상태 점검 시 아래와 비교하세요 — 특히 **대기 라우터는
"오버레이 BGP Active(미수립)"가 정상**이라는 점이 헷갈리기 쉽습니다.

### A.1 C8000V-1 (활성 라우터)

`show ip interface brief | exclude unassigned` — Tunnel100 up/up, Lo0 = 애니캐스트 10.255.255.1

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.1.1.30       YES DHCP   up                    up
GigabitEthernet2       10.1.1.90       YES NVRAM  up                    up
Loopback0              10.255.255.1    YES NVRAM  up                    up
Tunnel0                172.16.100.2    YES unset  up                    up
Tunnel100              172.16.100.2    YES NVRAM  up                    up
```

`show ip bgp summary` — RS(.29) 세션은 PfxRcd 0이 정상(RS는 광고하지 않음),
오버레이(172.16.100.1=CGW)에서 1개 = 온프렘 LAN 172.20.51.0/24

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.29       4        65100    1922    1880       13    0    0 02:40:04        0
172.16.100.1    4        65000    3099    3098       13    0    0 02:38:27        1
```

`show bfd neighbors` — RS(Gi1)와 오버레이(Tu100) 두 세션 모두 Up

```
NeighAddr                              LD/RD         RH/RS     State     Int
10.1.1.29                            4097/2807522020 Up        Up        Gi1
172.16.100.1                         4103/4110       Up        Up        Tu100
```

`show ip pim neighbor` — Tunnel100 너머로 CGW가 PIM 네이버

```
Neighbor          Interface                Uptime/Expires    Ver   DR
Address                                                            Prio/Mode
172.16.100.1      Tunnel100                02:39:25/00:01:44 v2    1 / S P G
```

`show ip pim rp mapping` — 정적 RP = CGW Lo0

```
Group(s): 224.0.0.0/4, Static
    RP: 172.20.255.1 (ip-172-20-255-1.ap-northeast-2.compute.internal)
```

`show ip mroute 239.1.1.1` — (*,G)의 수신 인터페이스는 Tunnel100(RPF 네이버 = CGW),
송출 인터페이스는 Gi2(TGW 도메인으로 방출)

```
(*, 239.1.1.1), 02:41:17/stopped, RP 172.20.255.1, flags: SJC
  Incoming interface: Tunnel100, RPF nbr 172.16.100.1, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 02:41:17/00:00:44, flags:
```

`show ip mroute 239.1.1.1` — **트래픽이 흐르는 중**에는 (*,G) 아래에 (S,G) 엔트리가
추가되고 `flags: JT`(SPT 전환 완료)가 붙습니다. 온프렘 소스 172.20.51.10에서
ping 송신 중 캡처:

```
(172.20.51.10, 239.1.1.1), 00:00:59/00:02:00, flags: JT
  Incoming interface: Tunnel100, RPF nbr 172.16.100.1, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 00:00:59/00:02:00, flags:
```

`show ip mroute 239.1.1.1 count` — 포워딩 카운터로 트래픽 통과를 정량 확인.
ping 1초 간격이면 초당 약 1패킷(아래 `52/1/122/0`의 두 번째 필드), RPF failed와
드롭은 0이어야 합니다. 트래픽이 멎으면 (S,G)는 약 3분 뒤 만료되고 카운터가
리셋됩니다 — 낮은 카운터 자체는 이상이 아닙니다.

```
Group: 239.1.1.1, Source count: 1, Packets forwarded: 54, Packets received: 54
  RP-tree: Forwarding: 2/0/122/0, Other: 2/0/0
  Source: 172.20.51.10/32, Forwarding: 52/1/122/0, Other: 52/0/0
```

`show ip bgp neighbors 10.1.1.29 advertised-routes` — RS에 애니캐스트 /32와
CGW에서 배운 LAN을 재광고 (활성 라우터만 LAN 경로를 가짐)

```
     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.255.255.1/32  0.0.0.0                  0         32768 i
 *>   172.20.51.0/24   172.16.100.1             0             0 65000 i
```

### A.2 C8000V-2 (대기 라우터)

`show ip interface brief` — 활성과 동일하게 모든 인터페이스 up/up.
Lo0 10.255.255.1은 두 라우터가 공유(애니캐스트)하며, GRE는 무상태라 터널도 up으로 보입니다.

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.1.1.40       YES DHCP   up                    up
GigabitEthernet2       10.1.1.100      YES manual up                    up
Loopback0              10.255.255.1    YES manual up                    up
Tunnel100              172.16.100.2    YES manual up                    up
```

`show ip bgp summary` — **오버레이 세션이 Active(미수립)인 것이 대기 상태의 정상**입니다.
CGW의 GRE는 RS가 활성으로 정한 라우터에만 앵커되므로, 대기 라우터의 터널로는
CGW 트래픽이 오지 않아 오버레이 BGP가 수립되지 않습니다.

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.41       4        65100    3705    3623        8    0    0 05:08:38        0
172.16.100.1    4        65000       0       0        1    0    0 02:40:22 Active
```

`show bfd neighbors` — RS(Gi1) 세션만 존재 (오버레이 BFD는 세션 수립 후 생성됨)

```
NeighAddr                              LD/RD         RH/RS     State     Int
10.1.1.41                            4097/3274535456 Up        Up        Gi1
```

`show ip bgp neighbors 10.1.1.41 advertised-routes` — 애니캐스트 /32만 광고
(AS-path prepend는 아웃바운드 정책이라 이 출력에는 안 보이고, RS 라우팅 DB에서
AS×4로 확인됩니다). LAN 172.20.51.0/24는 오버레이가 없어 미광고 — 정상.

```
     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.255.255.1/32  0.0.0.0                  0         32768 i
```

`show ip mroute 239.1.1.1` — (*,G)는 있으나 RPF 네이버가 0.0.0.0(터널 너머 PIM 네이버 없음).
페일오버 시 오버레이가 수립되면 활성과 동일한 형태로 전환됩니다.

```
(*, 239.1.1.1), 05:09:20/stopped, RP 172.20.255.1, flags: SJC
  Incoming interface: Tunnel100, RPF nbr 0.0.0.0, Mroute
  Outgoing interface list:
    GigabitEthernet2, Forward/Sparse, 05:09:20/00:00:13, flags:
```

### A.3 활성 vs 대기 판독 요점

| 항목 | 활성 (C8000V-1) | 대기 (C8000V-2) |
|---|---|---|
| RS BGP/BFD (Gi1) | Established / Up | Established / Up (동일) |
| 오버레이 BGP (172.16.100.1) | Established, PfxRcd 1 | **Active (미수립) — 정상** |
| 오버레이 BFD (Tu100) | Up | 세션 없음 |
| PIM 네이버 (Tu100) | CGW 172.16.100.1 | 없음 |
| RS로 광고 | 10.255.255.1/32 + 172.20.51.0/24 | 10.255.255.1/32만 (prepend ×3) |
| mroute (*,239.1.1.1) | RPF nbr = 172.16.100.1 | RPF nbr = 0.0.0.0 |

두 라우터의 출력이 위 표와 반대라면 페일오버가 일어난 상태입니다 — RS 라우팅 DB와
`mcast-tgw-rtb`의 10.255.255.1/32 넥스트홉 ENI로 현재 활성 라우터를 교차 확인하세요.

---

## 참조 (AWS 공식 문서)

**Amazon VPC Route Server**
- [VPC Route Server를 이용한 동적 라우팅](https://docs.aws.amazon.com/vpc/latest/userguide/dynamic-routing-route-server.html)
- [VPC Route Server 동작 방식](https://docs.aws.amazon.com/vpc/latest/userguide/route-server-how-it-works.html) — 엔드포인트/피어/BFD/Persist Routes 개념

**AWS Transit Gateway 멀티캐스트**
- [Transit Gateway 멀티캐스트 개요](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html) — IGMPv2 쿼리 주기(2분), SG/NACL 요건(프로토콜 2, 소스 0.0.0.0/32) 등 이 문서 5장의 근거
- [멀티캐스트 도메인](https://docs.aws.amazon.com/vpc/latest/tgw/multicast-domains-about.html)
- [Transit Gateway 할당량 (MTU 포함)](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html) — **멀티캐스트 패킷 단편화 미지원(드롭)**: 5.2의 GRE MTU 함정과 직결

**AWS Direct Connect**
- [Direct Connect 게이트웨이](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-gateways-intro.html)
- [Transit Gateway 연동 (Transit VIF)](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-transit-gateways.html)
- [허용 프리픽스(Allowed prefixes) 동작](https://docs.aws.amazon.com/directconnect/latest/UserGuide/allowed-to-prefixes.html) — 1.1의 수동 작업 ②의 근거

**기타**
- [EC2 Instance Connect Endpoint](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-ec2-instance-connect-endpoint.html) — 키페어 없는 라우터/리시버 SSH 접속
