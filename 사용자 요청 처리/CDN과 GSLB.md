# CDN과 GSLB
## 1) CDN (Content Delivery Network)
### 개념
- 정적 콘텐츠(HTML, CSS, JS, 이미지, 동영상 등)를 전 세계 **Edge 서버**에 분산 배치하여 사용자에게 가장 가까운 서버에서 제공.
- **목표**: 응답 지연(latency) 최소화, 대역폭 절감, 서버 부하 분산.

### 동작 흐름
1. 사용자가 도메인 요청 → 로컬 DNS → CDN DNS/Anycast.
2. CDN이 가장 가까운 Edge 서버로 요청 라우팅.
3. Edge 서버에 캐시가 있으면 즉시 제공, 없으면 Origin 서버에서 가져와 캐싱 후 전달.

### 기술 요소
- **Anycast Network**: 동일 IP를 전 세계 여러 노드에 광고, BGP를 통해 가장 가까운 노드로 라우팅.
- **GeoDNS**: 사용자 위치 기반으로 가장 가까운 Edge IP 반환.
- **캐싱 정책**: TTL 기반, Cache Invalidation, Stale-While-Revalidate.

### 장점
- 사용자 경험 개선 (빠른 로딩 속도).
- 트래픽 급증(Flash Crowd) 대응.
- 보안 기능 (DDoS 완화, TLS termination, WAF).

---

## 2) GSLB (Global Server Load Balancing)
### 개념
- 서로 다른 리전에 배치된 여러 데이터센터 간 **트래픽 분산 기술**.
- 단순 로드밸런서를 넘어, **글로벌 최적화**와 **장애 대응**을 지원.

### 동작 흐름 (DNS 기반 GSLB)
1. Client → Local DNS → GSLB DNS 질의.
2. GSLB → 상태/규칙 기반 최적 서버 IP 반환.
3. Local DNS가 캐싱 후 클라이언트에 전달.
4. 클라이언트가 해당 서버로 접속.

### Routing 방식
- **Geo Routing**: 사용자 위치 기반 라우팅.
- **Health Check Routing**: 서버 상태 확인 후 정상 노드로만 라우팅.
- **Load-based Routing**: 서버 부하 고려 (CPU, QPS 등).
- **Latency-based Routing**: 네트워크 지연시간 최소화.
- **Bandwidth-based Routing**: 네트워크 사용률 고려.
- **Disaster Recovery Routing**: 장애 시 DR 센터로 자동 전환.
- **Round Robin**: 단순 순환 분배.

### 주요 기능
- **DNS + Anycast 연계**: 가장 가까운 서버 + 서버 상태 반영.
- **정책 기반 트래픽 분산**: VIP 사용자, 특정 리전 우선 라우팅 등.

---

## 3) CDN vs GSLB 비교

| 구분 | CDN | GSLB |
|------|-----|------|
| 대상 | 정적 콘텐츠 전달 | 동적 콘텐츠 및 API, 전체 트래픽 |
| 단위 | Edge 서버 캐시 | 데이터센터/리전 |
| 주요 목적 | 속도 최적화, 캐싱 | 글로벌 트래픽 분산, 장애 복구 |
| 기술 | Anycast, GeoDNS, 캐싱 | DNS 기반 라우팅, Health Check, 정책 기반 |
| 활용 | 웹 리소스, 동영상 스트리밍 | API 요청, DB 동기화, 리전 간 트래픽 |

---

## 4) 클라이언트 & 서버 전략

### 클라이언트
- **데이터 전송 최소화**: 압축, 배치 전송, 선택적 동기화.
- **로컬 캐싱**: localStorage, sessionStorage, PWA 캐시 활용.

### 서버
- **네트워크 최적화**: CDN 적극 활용, BGP/Anycast 최적화.
- **비동기 처리**: 장시간 작업은 비동기로 수행.
- **모니터링**: Health check, 네트워크 상태 추적, 자동 Failover.

---

## 5) 실제 적용 시나리오
- **CDN**: 정적 파일(JS, CSS, 이미지, 동영상)을 글로벌 사용자에게 빠르게 제공.
- **GSLB**: API 요청, 동적 콘텐츠, DB/통계 동기화 트래픽을 지역별로 최적화 분배.
- 예: 사용자는 가까운 CDN Edge에서 정적 리소스를 받고, API는 GSLB를 통해 가장 빠른 데이터센터에 연결.

---

## 6) 한 줄 요약
> **CDN은 정적 콘텐츠를 전 세계에 캐싱/전송하는 네트워크**,  
> **GSLB는 글로벌 리전 단위로 트래픽을 최적화·분산하는 기술**로,  
> 둘을 함께 사용하면 **속도 + 가용성 + 안정성**을 모두 확보할 수 있다.
