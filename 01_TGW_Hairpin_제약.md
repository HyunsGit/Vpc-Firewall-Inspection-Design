# Intra-VPC 검사 (1) TGW Hairpin 제약

- 목적

- 트래픽 유형별 검사 가능 여부

- 시도 구성: TGW Hairpin

- 적용 구성도

- 제약 사항

- 결론

**TL;DR**

KakaoCloud는 VPC 내부(intra-VPC) 목적지 경로의 target으로 TGW를 허용하지 않음. 따라서 Prod App ↔ Prod DB 트래픽을 Hub VPC 방화벽으로 우회(TGW hairpin)시키는 검사 방식은 불가능함.

# Intra-VPC 트래픽 검사 - TGW Hairpin 방식 제약

## 목적

Prod VPC 내부 서브넷 간 트래픽(App 10.228.249.0/23 ↔ DB 10.228.251.0/25, Intra-VPC East-West)에 대한 방화벽 검사 적용.

설계 방향은 모든 검사 트래픽을 Hub VPC의 단일 방화벽(GW-Games-External-FW)으로 집중시키는 중앙 집중형 검사(centralized inspection) 구조임. 방화벽 인스턴스·정책·라이선스를 단일 지점에서 운영하여 관리 부담을 줄이는 것이 목표임.

## 트래픽 유형별 검사 가능 여부

중앙 집중형 검사 구조에서 트래픽 유형에 따라 Hub 방화벽 검사 적용 가능 여부가 달라짐.

| 트래픽 유형 | 예시 | TGW 경유 검사 |
|---|---|---|
| North-South | Prod ↔ 외부 IPsec 피어 | :tick: 가능 |
| Inter-VPC East-West | Mgt VPC ↔ Prod VPC | :tick: 가능 |
| Intra-VPC East-West | Prod App ↔ Prod DB | :cross: 불가 |

경계 통과 여부가 검사 가능 여부를 결정함

North-South 및 Inter-VPC 트래픽은 VPC 경계를 넘기 때문에 출발지 서브넷에서 목적지로 가는 과정에서 자연스럽게 TGW를 경유함. TGW 경유 구간에서 Hub 방화벽으로 우회시킬 수 있으므로 검사 적용이 가능함.

반면 Intra-VPC 트래픽(Prod App ↔ DB)은 동일 VPC 내부에서 발생하며, VPC 경계를 넘지 않으므로 TGW를 경유하지 않음. 이 트래픽을 Hub 방화벽으로 검사하려면 별도의 우회 경로를 인위적으로 구성해야 함.

## 시도 구성: TGW Hairpin

![[다이어그램](images/01_TGW_Hairpin_제약_img01.png)](https://github.com/HyunsGit/vpc-firewall-inspection-design/blob/main/images/01_TGW_Hairpin_%E1%84%8C%E1%85%A6%E1%84%8B%E1%85%A3%E1%86%A8_img01.png)

Intra-VPC 트래픽을 Hub 방화벽으로 검사하기 위해, Prod 내부 트래픽을 일단 TGW로 내보낸 뒤 검사 후 다시 Prod로 되돌리는 hairpin 경로 구성을 시도함. 이를 위해 App 서브넷 라우팅 테이블에 DB 대역을 TGW로 향하게 하는 more-specific 경로를 추가하려 함.

**Prod App 서브넷 RT (시도)**

```
10.228.248.0/22 → local # 기본 경로, 수정/삭제 불가
10.228.251.0/25 → TGW # DB 대역 TGW 우회

```

의도한 흐름은 다음과 같음: App(10.228.249.100)이 DB(10.228.251.120)로 보내는 트래픽이 10.228.251.0/25 → TGW 경로에 매칭되어 TGW로 향하고, TGW에서 Hub 방화벽으로 전달되어 검사된 뒤 다시 Prod DB로 되돌아오는 구조.

## 적용 구성도

![[다이어그램](images/01_TGW_Hairpin_제약_img04.png)](https://github.com/HyunsGit/vpc-firewall-inspection-design/blob/main/images/01_TGW_Hairpin_%E1%84%8C%E1%85%A6%E1%84%8B%E1%85%A3%E1%86%A8_img04.png)

## 제약 사항

라우팅 경로 추가 시도 결과, KakaoCloud가 경로 생성 단계에서 이를 거부함. 다음 두 가지 제약이 원인임.

KakaoCloud 라우팅 제약

서브넷 생성 시 VPC CIDR 대역에 대한 local 경로가 기본 생성되며, 이 경로는 수정하거나 삭제할 수 없음. 따라서 VPC 내부 목적지는 항상 local 경로가 우선 적용됨.

local보다 구체적인(more-specific) 경로 추가 자체는 허용되나, 목적지가 VPC 내부 서브넷 CIDR인 경우 target type은 {{Instance}}만 지정 가능함. {{TGW}}는 target으로 지정할 수 없음.

즉 VPC 내부 목적지에 대한 트래픽은, 그 VPC 내부에 존재하는 Instance(ENI)로만 우회시킬 수 있으며, VPC 외부로 나가는 게이트웨이(TGW)로는 우회 불가함.

**실제 오류 메시지**

```
Target destination does not match any subnet CIDR block.

When the destination is a subnet CIDR block within the VPC,
only 'Instance' can be set as the target type.

```

## 결론

KakaoCloud에서 Intra-VPC 목적지 경로의 허용 target은 해당 VPC 내부의 Instance(ENI)로 한정됨.

TGW hairpin 방식은 경로 생성 단계에서 거부되므로 Intra-VPC 트래픽 검사에 사용할 수 없음.

Intra-VPC 트래픽을 방화벽으로 검사하려면, 방화벽의 인터페이스(ENI)가 반드시 Prod VPC 내부에 위치해야 함. 이는 오류 메시지가 명시하는 유일한 합법적 target인 Instance 조건을 충족하기 위함임.

관련 문서

Instance target 확보를 위한 구현 방식은 Intra-VPC 트래픽 검사 - Cross-VPC ENI 구성 참조.
