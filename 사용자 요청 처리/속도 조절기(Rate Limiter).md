# 속도 조절기(Rate Limiter)

## 1) 정의 (What is a Rate Limiter?)
- **Rate Limiter**는 단위 시간당 요청 수를 제한하는 기법  
- 예시:
  - 한 사용자는 **초당 2회 이상 글 작성 불가**  
  - 동일 IP에서 **하루 최대 10개의 계정 생성 허용**   
- **필요성**:
  - DoS/DDoS 공격 방어  
  - 과도한 요청으로 인한 **비용 증가 방지**  
  - 서비스 남용(Misuse) 제어   

---

## 2) 요구사항 (Requirements)
- **정확한 속도 제어 (Precise Control)**  
- **고성능**: 높은 처리량(Throughput) + 낮은 지연시간(Latency)  
- **메모리 효율성**: 많은 키를 추적할 수 있어야 함  
- **고가용성, 장애 허용성** (HA/FT)  
- **분산 환경 지원**: 여러 서버/노드에서 일관된 제한 보장   

---

## 3) Rate Limiter의 배치 위치 (Placement)
1. **Client Side**  
   - 앱/브라우저 수준에서 제어  
   - 우회 가능성이 크므로 보조적 수단  

2. **Server Side**  
   - API Gateway, Reverse Proxy, Load Balancer 앞단  
   - 요청을 수신하는 곳에서 **429 Too Many Requests** 응답으로 차단   

---

## 4) 구현 알고리즘
1. **Token Bucket**  
   - 일정 비율로 토큰이 채워짐, 요청 시 토큰 사용  
   - burst 허용 가능, 네트워크 QoS에도 활용  

2. **Leaky Bucket**  
   - 요청이 큐에 들어가고 일정 속도로 배출  
   - 안정적인 출력 속도 유지  

3. **Fixed Window Counter**  
   - 시간 구간(예: 1분)에 카운터 증가  
   - 단순하지만 경계(boundary) 시점에 burst 가능  

4. **Sliding Window Log**  
   - 요청 timestamp 기록 후 윈도우 내 요청 수 계산  
   - 메모리 사용량 많음  

5. **Sliding Window Counter**  
   - 최근 구간 + 이전 구간의 비율을 이용해 근사치 계산  
   - 정확도와 메모리 효율 사이의 절충안  

---

## 5) 분산 환경 고려
- 여러 서버/인스턴스에서 같은 사용자/IP 제한을 일관되게 적용해야 함  
- **중앙 저장소 사용**:
  - Redis, Memcached 같은 in-memory store에서 카운터/버킷 관리  
  - `INCR`, `EXPIRE` 같은 원자적 연산 활용  
- **동기화 문제**:
  - Redis Cluster, Sharding 시 키 배치 주의  
  - 네트워크 파티션 대비 → Fail-open/Fail-close 정책 결정  

---

## 6) 운영 이슈
- **429 응답 처리**: 클라이언트가 retry-after 헤더 기반으로 재시도하도록 안내  
- **Burst vs Smooth**: 순간 폭주를 허용할지, 균등 분산만 허용할지 정책 필요  
- **멀티 레벨 제한**:  
  - User-level, IP-level, API key-level, Endpoint별 제한  
- **모니터링 지표**:
  - 제한 발생률, 차단된 요청 수, 사용자 불만족도, Redis QPS  

---

## 7) 보완 전략
- **Rate Limiting + WAF**: 공격성 트래픽 탐지/차단  
- **Quota 관리**: 일 단위/월 단위 사용량 제한  
- **Adaptive Rate Limiting**: 상황에 따라 동적으로 조정 (예: 시스템 부하 높을 때 강화)  

---

## 8) 한 줄 요약
> Rate Limiter는 **서비스 안정성과 비용 제어를 위한 핵심 방어선**으로, 구현 시 **알고리즘 선택·분산 환경 동기화·운영 정책**을 반드시 고려해야 한다.
