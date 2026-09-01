# Intra-VPC 검사 (2) Cross-VPC ENI 구성

- 전제

- Cross-VPC ENI 개념

- 시도 구성

- 적용 구성도

- 트래픽 흐름

- 검증

- 아키텍처 요약

- 설정 체크리스트

**TL;DR**

Hub VPC 방화벽에 보조 trust ENI를 Prod VPC의 trust 서브넷에 배치(Cross-VPC ENI)하면, 해당 ENI가 Prod VPC 내부의 Instance target으로 동작함. 이를 통해 방화벽 인스턴스 이전 없이 Prod App ↔ Prod DB 트래픽을 Hub 방화벽으로 검사함. 데이터플레인 검증 완료.

# Intra-VPC 트래픽 검사 - Cross-VPC ENI 구성

## 전제

선행 문서

Intra-VPC 트래픽 검사 - TGW Hairpin 방식 제약에서 확인된 사항:

KakaoCloud에서 Intra-VPC 목적지 경로의 허용 target은 Prod VPC 내부의 Instance(ENI)로 한정됨.

TGW target은 지정 불가.

이 제약에 따라 방화벽 인터페이스가 Prod VPC 내부에 존재해야 함. 다만 이것이 반드시 Prod VPC에 별도 방화벽 인스턴스를 두어야 함을 의미하지는 않음. Cross-VPC ENI를 이용하면 방화벽 인스턴스는 Hub VPC에 유지하면서 인터페이스만 Prod VPC에 배치할 수 있음.

## Cross-VPC ENI 개념

KakaoCloud는 한 VPC의 방화벽 인스턴스에 다른 VPC 서브넷에 속한 ENI를 추가로 attach하는 것을 허용함. 이 경우 ENI의 네트워크 소속과 ENI를 처리하는 인스턴스의 물리적 위치가 분리됨.

| 구분 | 위치 |
|---|---|
| ENI(보조 trust 포트)의 네트워크 소속 (IP / 서브넷 / 라우팅) | Prod VPC (trust 서브넷) |
| ENI를 처리하는 방화벽 인스턴스 | Hub VPC (GW-Games-External-FW) |

라우팅은 ENI의 네트워크 소속만 판단함

보조 trust ENI는 Prod VPC의 trust 서브넷에 속하고 Prod 내부 IP를 가지므로, 라우팅 관점에서 Prod VPC 내부의 로컬 next-hop으로 처리됨. 이 ENI를 처리하는 방화벽 인스턴스가 물리적으로 Hub VPC에 있다는 사실은 라우팅 판단에 영향을 주지 않음.

결과적으로 이 ENI는 *Prod VPC 내부에 존재하는 Instance*이므로, 문서 1에서 확인된 "Intra-VPC 목적지의 target은 Instance만 허용" 제약을 그대로 충족함. TGW로는 불가능했던 우회가, Prod 내부에 소속된 ENI를 target으로 지정함으로써 가능해짐.

## 시도 구성

![다이어그램](images/02_Cross-VPC_ENI_구성_img01.png)

Hub VPC의 방화벽(GW-Games-External-FW)에 보조 trust 포트(Secondary Trust ENI)를 추가하고, 이 ENI를 Prod VPC의 trust 서브넷에 배치함. 이후 Prod App/DB 서브넷의 more-specific 경로 target을 해당 ENI로 지정함.

**Prod VPC 라우팅**

```
Prod App 서브넷 RT:
10.228.248.0/22 → local
10.228.251.0/25 → Prod-trust-ENI (Instance) # DB 대역 검사 우회

Prod DB 서브넷 RT:
10.228.248.0/22 → local
10.228.249.0/23 → Prod-trust-ENI (Instance) # App 대역 검사 우회 (대칭 경로)

Prod Trust 서브넷 RT:
10.228.248.0/22 → local # loop-breaker

```

대칭 경로 필수

App 서브넷과 DB 서브넷 양쪽 모두에 상대 대역을 ENI로 향하게 하는 대칭 경로를 구성해야 함. 방화벽은 stateful 검사를 수행하므로, 세션의 양방향(App→DB 요청, DB→App 응답)이 모두 동일 방화벽을 경유해야 세션 추적이 유지됨. 한쪽만 우회시키면 응답 트래픽이 방화벽을 우회하여 세션이 비대칭 상태가 되고 검사가 실패함.

loop-breaker

Prod Trust 서브넷의 라우팅 테이블은 {{local}}만 유지함. 방화벽이 검사 완료 후 목적지로 전달하는 트래픽은 이 서브넷의 라우팅 테이블을 참조하는데, 여기에 재우회 경로가 없어야 검사된 트래픽이 다시 방화벽으로 돌아가는 루프를 방지할 수 있음.

## 적용 구성도

![다이어그램](images/02_Cross-VPC_ENI_구성_img02.png)

## 트래픽 흐름

Prod App(10.228.249.100) → DB(10.228.251.120) 트래픽 발생

App 서브넷 RT의 10.228.251.0/25 → Prod-trust-ENI 경로에 매칭, 보조 trust ENI로 전달

해당 ENI를 소유한 Hub VPC 방화벽이 검사 수행 (App-zone → DB-zone 정책 적용)

검사 완료 후 DB(10.228.251.120)로 전달. Prod Trust 서브넷 RT의 local 경로로 처리되어 재우회 없이 DB에 도달

응답 트래픽(DB → App)은 DB 서브넷 RT의 대칭 경로를 통해 동일 방화벽을 경유, stateful 세션 유지

one-arm 구조

보조 trust ENI는 트래픽이 들어오고 나가는 인터페이스가 동일한 one-arm 구조로 동작함. 방화벽은 동일 인터페이스로 수신한 트래픽을 검사 후 같은 인터페이스로 다시 내보냄.

## 검증

확인 필요

데이터플레인 전 구간 검증 완료

방화벽 세션 로그 및 패킷 캡처를 통해 Prod App ↔ Prod DB 양방향 트래픽이 보조 trust ENI를 경유하여 검사되고, 정상적으로 목적지에 전달됨을 확인함.

## 아키텍처 요약

트래픽 유형별 검사 방화벽 및 경로 방식.

| 트래픽 유형 | 검사 방화벽 | 경로 방식 |
|---|---|---|
| North-South (외부 피어) | Hub VPC 방화벽 | TGW 경유 |
| Inter-VPC (Mgt ↔ Prod) | Hub VPC 방화벽 | TGW 경유 |
| Intra-VPC (Prod App ↔ DB) | Hub VPC 방화벽 | Cross-VPC ENI (Prod trust 서브넷 보조 trust 포트) |

중앙 집중형 검사 구조 유지

세 가지 트래픽 유형 모두 Hub VPC의 단일 방화벽에서 검사됨. North-South 및 Inter-VPC는 TGW 경유로, Intra-VPC는 Cross-VPC ENI로 처리됨. 방화벽 인스턴스·정책·라이선스는 Hub VPC에서 일원화 운영되며, Prod VPC에는 검사용 ENI만 배치되므로 별도의 Prod 전용 방화벽 없이 중앙 집중형 검사 구조를 유지함.

## 설정 체크리스트

확인 필요

필수 설정 항목

보조 trust ENI의 source/destination check 비활성화 — ENI가 자신에게 향하지 않는 transit 트래픽을 forwarding하려면 필수. 미설정 시 방화벽이 통과 트래픽을 drop함.

방화벽 one-arm inspection 설정 — 동일 인터페이스로 트래픽이 in/out되는 구조이므로, FortiOS의 동일 인터페이스 트래픽 허용 설정이 필요함. 기본값은 차단이므로 명시적 허용 필요.

App/DB 양방향 대칭 경로 설정 — stateful 검사 세션 유지를 위해 양쪽 서브넷 모두 상대 대역을 ENI로 향하게 구성.

Prod Trust 서브넷 RT는 {{local}}만 유지 — 검사 완료 트래픽의 재우회(루프) 방지.