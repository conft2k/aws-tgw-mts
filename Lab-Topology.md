# Lab Topology — 하이브리드 멀티캐스트 HA (실배포 기준)

배포된 mcast-base 스택과 라우터 설정 그대로의 토폴로지입니다.
상세 설명은 [README.md](README.md)를 참고하세요.

## 1. 데이터 경로 — 멀티캐스트 스트림

```mermaid
flowchart TB
    SENDER["Sender<br/>172.20.51.10"]
    CGW["CGW · AS 65000<br/>Lo0 172.20.255.1"]
    DXGW["DXGW · AS 65300"]
    TGW["TGW · AS 65400<br/>멀티캐스트 도메인 IGMPv2"]
    TGWRT["TGW 기본 RT 조회 (정적)<br/>10.255.255.1/32 → VPC att"]

    SENDER ==>|"UDP 239.1.1.1"| CGW
    CGW ==>|"GRE 캡슐<br/>DX VIF ×2"| DXGW
    DXGW ==> TGW
    TGW ==> TGWRT

    subgraph VPC["VPC 10.1.1.0/24"]
        direction TB
        TGWRTB["tgw-rtb 조회 (RS 주입)<br/>10.255.255.1/32 → 활성 Gi1 ENI"]
        RRTB["router-rtb 조회 (정적)<br/>172.20.255.1/32 → TGW"]
        C1["C8000V-1 · 활성<br/>Gi2 .90 = receiver-sub-az1"]
        C2["C8000V-2 · 대기<br/>Gi2 .100 = receiver-sub-az2"]
        subgraph RSUB1["receiver-sub-az1 10.1.1.64/27 (AZ1)<br/>멀티캐스트 도메인 연결"]
            R1["EC2-1 리시버<br/>10.1.1.84"]
        end
        subgraph RSUB2["receiver-sub-az2 10.1.1.96/27 (AZ2)<br/>멀티캐스트 도메인 연결"]
            R2["EC2-2 리시버<br/>10.1.1.116"]
        end
    end

    TGWRT ==>|"att ENI 인입"| TGWRTB
    TGWRTB ==>|"GRE 종단<br/>애니캐스트 10.255.255.1"| C1
    TGWRTB -.->|"페일오버 시 재앵커"| C2
    C1 ==>|"Gi2 송출"| TGW
    TGW ==>|"도메인 연결 서브넷으로 복제<br/>(IGMPv2 조인 멤버 ENI 수신)"| RSUB1
    TGW ==>|"도메인 연결 서브넷으로 복제<br/>(IGMPv2 조인 멤버 ENI 수신)"| RSUB2
    C1 -.->|"PIM Join · 오버레이 BGP<br/>(GRE 상행)"| RRTB
    RRTB -.->|"TGW → DX → CGW"| CGW

    classDef active stroke:#22a06b,stroke-width:3px
    classDef standby stroke:#8993a4,stroke-dasharray:5 4
    classDef rtb stroke:#b8860b,stroke-width:2px
    class C1 active
    class C2 standby
    class TGWRT,TGWRTB,RRTB rtb
```

굵은 화살표가 스트림 경로이고, 황토색 테두리가 경로 위에서 조회되는
라우팅 테이블입니다:

1. **TGW 기본 RT** (정적) — DX로 들어온 GRE 패킷을 VPC 어태치먼트로 보냄
2. **tgw-rtb** (RS 주입) — 어태치먼트 ENI에 내린 패킷을 **활성** C8000V Gi1로 보냄.
   페일오버 시 Route Server가 바꾸는 지점이 바로 이 라우트
3. **router-rtb** (정적) — 역방향(PIM Join·오버레이 BGP가 GRE를 타고 CGW로
   올라갈 때) C8000V가 온프렘 Lo0로 가는 경로

복제는 **멀티캐스트 도메인에 연결된 두 서브넷**(receiver-sub-az1
`10.1.1.64/27`, receiver-sub-az2 `10.1.1.96/27`)으로만 일어납니다. 활성
C8000V가 Gi2(.90, receiver-sub-az1 소속)로 송출하면 TGW 도메인이 IGMPv2
조인 멤버가 있는 두 서브넷의 ENI로 복제합니다 — 이 구간은 **라우팅 테이블을
조회하지 않고** 도메인 멤버십으로 대상이 결정됩니다. receiver-rtb는 단방향
스트림에는 등장하지 않는 유니캐스트 리턴 경로라 2번 그림에 있습니다.

## 2. 컨트롤 플레인 — Route Server 페일오버

```mermaid
flowchart TB
    RS["VPC Route Server<br/>AS 65100 · Persist Routes 2분"]
    C1["C8000V-1 · AS 65200 · 활성<br/>Lo0+LAN 광고, prepend 없음"]
    C2["C8000V-2 · AS 65200 · 대기<br/>Lo0 광고, prepend ×3"]
    RTB["RS가 주입하는 라우트<br/>tgw-rtb: 10.255.255.1/32 → 활성 Gi1<br/>receiver-rtb: 172.20.51.0/24 → 활성 Gi1"]
    STATIC["정적 라우트<br/>router-rtb: 172.20.255.1/32 → TGW<br/>TGW 기본 RT: 10.255.255.1/32 → VPC att"]

    RS <-->|"RSE-1 .29 ↔ Gi1 .30<br/>eBGP + BFD 300ms×3"| C1
    RS <-->|"RSE-2 .41 ↔ Gi1 .40<br/>eBGP + BFD 300ms×3"| C2
    RS ==>|"베스트패스 주입<br/>BFD 다운 시 넥스트홉 전환"| RTB
    RTB -.- STATIC

    classDef active stroke:#22a06b,stroke-width:3px
    classDef standby stroke:#8993a4,stroke-dasharray:5 4
    class C1 active
    class C2 standby
```

활성 라우터가 죽으면 RS가 BFD로 감지(~1초)해 두 라우트의 넥스트홉을
C8000V-2 Gi1로 바꿉니다. GRE 목적지는 애니캐스트라 CGW는 무변경입니다.

## 3. IP / 서브넷 플랜

| 서브넷 | CIDR | AZ | 배치 |
|---|---|---|---|
| router-sub-az1 | 10.1.1.0/27 | 2a | C8000V-1 Gi1 `.30`, RSE-1 `.29` |
| router-sub-az2 | 10.1.1.32/27 | 2c | C8000V-2 Gi1 `.40`, RSE-2 `.41` |
| receiver-sub-az1 | 10.1.1.64/27 | 2a | C8000V-1 Gi2 `.90`, EC2-1 `.84` — 멀티캐스트 도메인 연결 |
| receiver-sub-az2 | 10.1.1.96/27 | 2c | C8000V-2 Gi2 `.100`, EC2-2 `.116` — 멀티캐스트 도메인 연결 |
| tgw-sub-az1 | 10.1.1.224/28 | 2a | TGW 어태치먼트 ENI + EICE |
| tgw-sub-az2 | 10.1.1.240/28 | 2c | TGW 어태치먼트 ENI |

| 항목 | 값 |
|---|---|
| GRE outer | CGW Lo0 `172.20.255.1` ↔ C8000V 공유 애니캐스트 Lo0 `10.255.255.1` |
| GRE inner (Tunnel100) | CGW `172.16.100.1` ↔ C8000V `172.16.100.2` (두 라우터 공유) |
| ASN | CGW 65000 · RS 65100 · C8000V 65200 · DXGW 65300 · TGW 65400 |
| 온프렘 LAN | 172.20.51.0/24 (Sender 172.20.51.10) |
