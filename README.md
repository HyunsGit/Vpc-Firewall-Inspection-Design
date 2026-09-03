# 방화벽 고도화 — Intra-VPC 트래픽 검사

KakaoCloud 환경에서 **Prod VPC 내부 서브넷 간 East-West 트래픽**(Intra-VPC)에 대한 방화벽 검사를 적용하기 위한 아키텍처 설계 및 구현 문서입니다.

## 문서 목록

| # | 문서 | 요약 |
|---|---|---|
| 1 | [TGW Hairpin 제약](https://github.com/HyunsGit/vpc-firewall-inspection-design/blob/main/01_TGW_Hairpin_%E1%84%8C%E1%85%A6%E1%84%8B%E1%85%A3%E1%86%A8.md) | Intra-VPC 트래픽을 TGW로 우회하는 방식이 KakaoCloud 라우팅 제약으로 불가능함을 확인 |
| 2 | [Cross-VPC ENI 구성](https://github.com/HyunsGit/vpc-firewall-inspection-design/blob/main/02_Cross-VPC_ENI_%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC.md) | Hub VPC 방화벽의 보조 trust ENI를 Prod VPC에 배치하여 Intra-VPC 검사를 실현한 해법 |
| 3 | [Inter-VPC 비대칭 라우팅 대응](https://github.com/HyunsGit/vpc-firewall-inspection-design/blob/main/03_Inter-VPC_%E1%84%87%E1%85%B5%E1%84%83%E1%85%A2%E1%84%8E%E1%85%B5%E1%86%BC_%E1%84%85%E1%85%A1%E1%84%8B%E1%85%AE%E1%84%90%E1%85%B5%E1%86%BC_%E1%84%83%E1%85%A2%E1%84%8B%E1%85%B3%E1%86%BC.md) | Cross-VPC ENI 도입 후 발생한 Inter-VPC 비대칭 라우팅 문제의 원인 분석 및 조치 |

---

## 배경 및 목표

### 목표
Prod VPC 내부 서브넷 간 트래픽(App `10.228.249.0/23` ↔ DB `10.228.251.0/25`)에 대해 Hub VPC 방화벽(`External-FW`)을 통한 **중앙 집중형 검사(Centralized Inspection)** 적용.

### 설계 원칙
- 방화벽 인스턴스·정책·라이선스를 **Hub VPC 단일 지점**에서 운영
- Prod VPC에 별도 방화벽 인스턴스를 추가하지 않음
- 모든 트래픽 유형(North-South, Inter-VPC, Intra-VPC)을 **하나의 방화벽**으로 검사

---

## 아키텍처 요약

```
트래픽 유형                  검사 경로
────────────────────────────────────────────────────────
North-South (외부 피어)      TGW → Hub VPC 방화벽
Inter-VPC (Mgt ↔ Prod)      TGW → Hub VPC 방화벽
Intra-VPC (Prod App ↔ DB)   Cross-VPC ENI (Prod trust 서브넷 보조 포트)
```

---

## 문서별 핵심 내용

### 1. TGW Hairpin 제약 → 불가

- **시도**: Prod App 서브넷 RT에 `10.228.251.0/25 → TGW` more-specific 경로 추가
- **결과**: KakaoCloud가 경로 생성 단계에서 거부
- **이유**: VPC 내부 목적지 경로의 target은 `Instance`(ENI)만 허용, `TGW` 불가
- **결론**: Intra-VPC 검사는 방화벽 ENI가 **Prod VPC 내부에 위치**해야 함

### 2. Cross-VPC ENI 구성 → 해법

- Hub VPC 방화벽에 **보조 trust ENI**(`port4`)를 Prod VPC trust 서브넷에 배치
- ENI의 네트워크 소속(IP/서브넷/라우팅)은 Prod VPC, 처리 인스턴스는 Hub VPC
- **라우팅 구성**:

  ```
  Prod App 서브넷 RT:  10.228.251.0/25 → Prod-trust-ENI (Instance)  # DB 방향
  Prod DB 서브넷 RT:   10.228.249.0/23 → Prod-trust-ENI (Instance)  # App 방향 (대칭)
  Prod Trust 서브넷 RT: local만 유지 (loop-breaker)
  ```

- **데이터플레인 검증 완료** (세션 로그 + 패킷 캡처)

### 3. Inter-VPC 비대칭 라우팅 대응 → 필수 조치

Cross-VPC ENI 도입으로 FortiGate FIB에서 Prod 대역이 `port4`(intra)와 `port2`(inter) 두 경로를 갖게 되어, Inter-VPC 통신 시 비대칭 라우팅 발생.

**필수 조치 2가지 (모두 적용 필수)**:

| 조치 | 내용 |
|---|---|
| 조치 1 | `port2`에 한해 RPF(src-check) 비활성화 |
| 조치 2 | reply(mgt→prod) 방향 policy route 1개 추가 → `port2` 강제 egress |

> ⚠️ **주의**: 목적지가 Prod app/db인 신규 inter-VPC 플로우 구성 시, reply 방향 policy route 추가 및 양방향 검증을 표준 절차로 적용할 것.

---

## 디렉토리 구조

```
firewall-upgrade/
├── README.md                               # 본 문서
├── 01_TGW_Hairpin_제약.md                  # TGW Hairpin 시도 및 제약 분석
├── 02_Cross-VPC_ENI_구성.md               # Cross-VPC ENI 구현 해법
├── 03_Inter-VPC_비대칭_라우팅_대응.md      # 비대칭 라우팅 문제 해결
└── images/                                 # 구성도 및 다이어그램
    ├── 01_TGW_Hairpin_제약_img01.png       # TGW Hairpin 시도 구성
    ├── 01_TGW_Hairpin_제약_img02.png       # 오류 화면
    ├── 01_TGW_Hairpin_제약_img03.png       # 오류 화면
    ├── 01_TGW_Hairpin_제약_img04.png       # 적용 구성도
    ├── 02_Cross-VPC_ENI_구성_img01.png     # Cross-VPC ENI 구성
    ├── 02_Cross-VPC_ENI_구성_img02.png     # 적용 구성도
    └── 03_Inter-VPC_비대칭_라우팅_대응_img01.png  # 비대칭 라우팅 구성도
```

---

## 관련 환경

- **클라우드**: KakaoCloud
- **방화벽**: FortiGate (External-FW, Hub VPC)
- **VPC 구조**: Hub VPC (방화벽) + Prod VPC (App/DB) + Mgt VPC
- **네트워크 연결**: TGW(Transit Gateway) 중심 Hub-and-Spoke
