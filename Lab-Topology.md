# Lab Topology — 하이브리드 멀티캐스트 HA (실배포 기준)

배포된 mcast-base 스택과 라우터 설정 그대로의 토폴로지입니다.
자세한 설명은 [README.md](README.md)를 참고하세요.

```mermaid
flowchart LR
    subgraph ONPREM["온프렘 172.20.51.0/24"]
        direction TB
        SENDER["Sender<br/>172.20.51.10"]
        CGW["CGW · AS 65000<br/>Lo0 172.20.255.1/32<br/>(GRE 소스 · PIM RP)"]
        SENDER -->|"UDP 239.1.1.1:5001"| CGW
    end

    DXGW["DXGW<br/>AS 65300"]
    TGW["TGW · AS 65400<br/>멀티캐스트 도메인 (IGMPv2)<br/>기본 RT: 10.255.255.1/32 → VPC att (정적)"]

    CGW ==>|"DX Transit VIF ×2<br/>BGP+BFD · Lo0만 광고"| DXGW
    DXGW ==> TGW

    subgraph VPC["VPC 10.1.1.0/24"]
        direction TB
        RS["VPC Route Server · AS 65100<br/>Persist Routes 2분"]

        subgraph AZ1["AZ1 (2a)"]
            direction TB
            RSE1(("RSE-1<br/>.29"))
            C1["C8000V-1 · AS 65200 · 활성<br/>Gi1 .30 (router-sub-az1 10.1.1.0/27)<br/>Gi2 .90 (receiver-sub-az1 10.1.1.64/27)<br/>Lo0 10.255.255.1 (애니캐스트 공유)<br/>Tunnel100 172.16.100.2"]
            EC21["EC2-1 리시버<br/>10.1.1.84 (receiver-sub-az1)"]
            ATT1["TGW att ENI + EICE<br/>(tgw-sub-az1 10.1.1.224/28)"]
        end

        subgraph AZ2["AZ2 (2c)"]
            direction TB
            RSE2(("RSE-2<br/>.41"))
            C2["C8000V-2 · AS 65200 · 대기<br/>Gi1 .40 (router-sub-az2 10.1.1.32/27)<br/>Gi2 .100 (receiver-sub-az2 10.1.1.96/27)<br/>Lo0 10.255.255.1 (애니캐스트 공유)<br/>Tunnel100 172.16.100.2"]
            EC22["EC2-2 리시버<br/>10.1.1.116 (receiver-sub-az2)"]
            ATT2["TGW att ENI<br/>(tgw-sub-az2 10.1.1.240/28)"]
        end

        RTBS["라우트 테이블<br/>router-rtb: 172.20.255.1/32 → TGW<br/>receiver-rtb: 172.20.51.0/24 → 활성 Gi1 ENI<br/>tgw-rtb: 10.255.255.1/32 → 활성 Gi1 ENI"]
    end

    RS ---|"RSE-1"| RSE1
    RS ---|"RSE-2"| RSE2
    RSE1 -. "eBGP+BFD 300ms×3<br/>Lo0+LAN 광고 (prepend 없음)" .- C1
    RSE2 -. "eBGP+BFD 300ms×3<br/>Lo0 광고 prepend ×3" .- C2
    RS -. "베스트패스 주입 / BFD 다운 시 전환" .-> RTBS

    TGW --- ATT1
    TGW --- ATT2

    CGW ==>|"GRE Tunnel100<br/>outer → 10.255.255.1 (애니캐스트)<br/>inner 172.16.100.1 ↔ .2<br/>오버레이 eBGP 65000↔65200 + PIM"| C1
    CGW -. "페일오버 시 재앵커<br/>(CGW 설정 무변경)" .- C2

    C1 ==>|"Gi2 → 멀티캐스트 도메인 송출"| TGW
    TGW ==>|"복제 (IGMPv2 조인)"| EC21
    TGW ==>|"복제 (IGMPv2 조인)"| EC22

    classDef active fill:#e6f4ea,stroke:#34a853,stroke-width:2px
    classDef standby fill:#f1f3f4,stroke:#9aa0a6,stroke-dasharray:4 3
    classDef rtb fill:#fef7e0,stroke:#f9ab00
    class C1 active
    class C2 standby
    class RTBS rtb
```

**표기 원칙**

- 굵은 실선 화살표 = 데이터 경로(GRE 언더레이 · 멀티캐스트), 점선 = 컨트롤 플레인(BGP/BFD)과 대기 경로
- 초록 = 활성(C8000V-1, prepend 없음), 회색 점선 테두리 = 대기(C8000V-2, prepend ×3)
- 라우트 테이블 3종의 넥스트홉 "활성 Gi1 ENI"가 Route Server가 BFD 감지로 전환하는 지점
- GRE 터널은 1개 — 목적지가 애니캐스트 Lo0(10.255.255.1)라 페일오버 시 CGW는 아무것도 바꾸지 않음
