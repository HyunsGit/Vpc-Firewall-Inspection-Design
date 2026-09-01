# Intra-VPC 검사 (3) Cross-VPC ENI 도입에 따른 Inter-VPC 비대칭 라우팅 대응

- 배경

- 증상

- 근본 원인 (2단계 비대칭)

- 1단계 — Strict RPF drop (forward 방향)

- 2단계 — 비대칭 return 경로 (reply 방향)

- 시도구성도

- 조치 (필수)

- 조치 1 — port2에 한해 RPF(src-check) 비활성화

- 조치 2 — reply 방향 policy route (1개)

- 검증

- 적용 대상

- 트러블슈팅 순서 (재발 시)

**TL;DR**

Intra-VPC 검사 (2) Cross-VPC ENI 구성 도입으로 Prod app/db 서브넷이 FortiGate에서 두 경로(inter-VPC=port2/hub-trust, intra-VPC=port4/prod-trust-ENI)를 갖게 됨.

이로 인해 prod가 목적지가 되는 inter-VPC 통신은 (1) port2 RPF 완화와 (2) reply 방향 policy route가 없으면 비대칭 라우팅으로 단방향만 동작함. 목적지가 prod인 inter-VPC 플로우에는 두 조치가 필수임.

# Cross-VPC ENI 도입에 따른 Inter-VPC 비대칭 라우팅 대응

## 배경

Intra-VPC 검사 (2) Cross-VPC ENI 구성 적용 이후, Prod app/db 서브넷은 FortiGate 상에서 두 개의 경로를 갖게 됨.

| 용도 | FortiGate 인터페이스 | 다음 홉 | 비고 |
|---|---|---|---|
| intra-VPC 검사 (prod app↔db) | port4 (prod trust ENI) | 10.228.248.129 | Cross-VPC ENI 경로 |
| inter-VPC 통신 (prod↔mgt 등) | port2 (hub trust) | 10.228.232.129 | TGW/hub 경유 경로 |

FortiGate FIB에서 prod app 대역(10.228.249.0/24 등)의 정적 경로는 port4로 향함. 그러나 inter-VPC 통신은 port2를 사용해야 함. 이 이중 경로가 비대칭 라우팅의 근본 원인임.

FIB란

※ FIB(Forwarding Information Base): FortiGate가 패킷 포워딩 시 실제 참조하는 유효 라우팅 테이블.

"get router info routing-table all"로 확인.

static/connected 등 여러 경로원으로 구성되며, 본 환경에서는 관심 대역이 static route로 구성됨.

## 증상

증상

inter-VPC 통신(예: prod↔mgt) 시 한 방향만 동작 (한 방향 ping 성공, 반대 방향 실패).

traceroute가 hub trust(10.228.232.129)까지는 도달하나 목적지 직전에서 응답이 사라짐.

방화벽 정책(firewall policy)은 정상 허용 상태이며, drop 원인이 정책이 아님.

## 근본 원인 (2단계 비대칭)

서로 다른 두 비대칭이 순차적으로 발생함.

### 1단계 — Strict RPF drop (forward 방향)

prod→mgt 패킷이 port2로 인입되나, 출발지(prod app, 10.228.249.x)로의 FIB 경로는 port4를 가리킴. strict RPF(reverse path forwarding) 검사는 "인입 인터페이스 ≠ 출발지로의 경로 인터페이스"를 spoofing으로 간주하여 drop함.

**debug flow — RPF drop**

```
received a packet(proto=1, 10.228.249.189 -> 10.228.237.244) from port2
reverse path check fail, drop

```

### 2단계 — 비대칭 return 경로 (reply 방향)

RPF 완화 후 forward는 통과하나, reply(mgt→prod, 목적지 10.228.249.x)가 FIB 기본 경로인 port4로 나감. forward는 port2를 사용했으므로 세션 경로가 비대칭이 되어 응답이 prod 호스트에 도달하지 못함.

**sniffer — reply가 port4로 잘못 egress**

```
port2 in 10.228.237.244 -> 10.228.249.189: icmp echo reply
port4 -- 10.228.237.244 -> 10.228.249.189: icmp echo reply # 비대칭시
```

## 시도구성도

![다이어그램](images/03_Inter-VPC_비대칭_라우팅_대응_img01.png)

## 조치 (필수)

목적지가 prod인 inter-VPC 플로우에 반드시 적용

아래 두 가지를 모두 적용해야 양방향 통신이 성립함. 하나만 적용 시 단방향만 동작함.

### 조치 1 — port2에 한해 RPF(src-check) 비활성화

Prod app 서브넷이 의도적으로 두 경로(port2/port4)를 가지므로 strict RPF는 구조적으로 부적합함. port2에 한해서만 완화하고, port1(untrust/edge)·port4는 strict 유지.

```
config system interface
edit port2
set src-check disable
next
end

```

최소 권한 원칙

전역(config system settings → set strict-src-check disable)이 아닌 port2 인터페이스 단위로 비활성화함. 인터넷/외부 대면 인터페이스(port1)와 intra-VPC 인터페이스(port4)의 anti-spoofing은 그대로 유지되어 보안 영향이 최소화됨.

### 조치 2 — reply 방향 policy route (1개)

reply(mgt→prod)는 목적지가 prod app(10.228.249.x)이므로 FIB상 port4로 향함. 이를 port2로 강제하는 policy route가 필요함. forward(prod→mgt)는 목적지가 mgt이고 static route(10.228.237.0/24 → port2)가 이미 port2로 향하므로 policy route 불필요.

여기서 10.228.239.129는 hub trust subnet의 gateway ip.

**reply 방향 policy route**

```
config router policy
edit 1
set input-device "port2"
set srcaddr "sub-games-mgt-prod-app-a-10.228.237.0/24" "sub-games-mgt-prod-app-b-10.228.238.0/24"
set dstaddr "sub-games-prod-app-a-10.228.249.0/24" "sub-games-prod-app-b-10.228.250.0/24"
set gateway 10.228.232.129
set output-device "port2"
set comments "mgt -> prod reply: Hub Trust(port2) 강제 (대칭 세션)"
next
end

```

forward용 policy route는 왜 불필요한가

forward(prod→mgt)의 목적지 mgt 대역은 static route가 이미 port2로 향함. 반면 reply(mgt→prod)의 목적지 prod 대역은 static route가 port4로 향하므로, 이 방향만 policy route로 override가 필요함. KakaoCloud 라우팅 테이블은 destination-only라 port2/port4 트래픽을 목적지로 구분할 수 없으나, FortiGate policy route는 source+destination으로 매칭하므로 reply 플로우만 선별하여 port2로 강제 가능함.

검증 시 확인

다중경로(ECMP)나 특수 라우팅 상황에서는 forward용 policy route도 필요할 수 있음. 신규 구성 시 reply 방향만 설정한 뒤 반드시 양방향 검증을 수행하고, forward가 static route만으로 정상 동작하는지 확인할 것.

## 검증

**양방향 성공 확인**

```
prod 호스트에서

ping 10.228.237.244 # 정상 응답 (steady-state ~0.6ms)
traceroute 10.228.237.244 # 목적지까지 완주 (hub trust 10.228.232.129 경유)

FortiGate에서 reply가 port2로 egress 확인

diagnose sniffer packet any 'host 10.228.249.189 and host 10.228.237.244' 4

기대: reply가 port2 in / port2 out (port4 아님)

```

## 적용 대상

목적지가 prod app/db인 inter-VPC 플로우는 조치 필수

prod app/db가 목적지가 되는 inter-VPC 통신은 return이 port4 기본 경로를 타므로 reply 방향 policy route가 필요함.

| 플로우 | 방향 (목적지 prod) | 조치 | 상태 |
|---|---|---|---|
| prod ↔ mgt | mgt → prod (관리 접속) | RPF 완화 + reply policy route | :tick: 적용 완료 |
| hub ↔ prod | hub → prod (agent/관리 등) | RPF 완화 + reply policy route | 예방적 구성 권장 (워크로드 발생 가능) |

범위 밖

sbox ↔ prod 및 prod ↔ sbox 통신은 요구사항에 없으므로 대상에서 제외함. (sbox는 mgt와만 통신: mgt↔sbox)

신규 플로우 구성 시 표준 절차

"목적지가 prod app/db 서브넷인 inter-VPC 트래픽"은 FortiGate FIB에서 port4로 기본 라우팅됨. 신규 inter-VPC 플로우(특히 prod가 목적지) 구성 시, 단방향 확인에 그치지 말고 reply 방향 policy route 추가 + 양방향 검증을 표준 절차로 삼을 것.

## 트러블슈팅 순서 (재발 시)

diagnose debug flow 로 drop 원인 확인 (RPF fail / policy deny / no route 구분)
diagnose sniffer packet any 'host <A> and host <B>' 4 로 양방향 인터페이스 확인 (forward/reply가 각각 어느 port로 in/out 되는지)

reply가 port4로 egress → 조치 2(reply policy route) 누락

forward가 RPF로 drop → 조치 1(port2 src-check) 누락

NPU offload로 egress가 sniffer에 안 보일 수 있으므로, 필요 시 offload 비활성화 후 재확인