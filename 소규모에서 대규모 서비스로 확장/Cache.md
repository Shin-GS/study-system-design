# Cache 정리

## 1) 개념 & 목표
- **Cache**: 느린 저장소(원천 데이터) 앞단에 놓아 **읽기 성능 향상**과 **원천 부하 감소**를 달성하는 계층.
- 목표: **지연시간 감소**, **QPS 증가**, **비용 절감**, **신뢰도 유지(일관성·오류 전파 방지)**.

---

## 2) 계층별 캐시
- **클라이언트/브라우저 캐시**: HTTP 캐시 헤더(`Cache-Control`, `ETag`, `Last-Modified`).
- **CDN/엣지 캐시**: 정적 콘텐츠, 이미지/JS/CSS, 그리고 캐시 가능한 API 응답.
- **애플리케이션 캐시**: 프로세스 내(in‑memory) 또는 분산 캐시(예: Redis, Memcached).
- **DB 앞 캐시**: 읽기 집중 쿼리에 대한 결과 캐시, Materialized View와 조합.

> 다층 캐시 사용 시 **TTL·무효화 규칙**의 일관성이 중요.

---

## 3) 캐시 전략(패턴)
- **Cache‑Aside (Lazy Loading)**  
  1) 캐시에 없으면 DB 조회 → 캐시에 적재 → 응답.  
  2) 쓰기는 DB에 먼저 반영 후 캐시 **무효화**.  
  - 장점: 단순/유연. 단점: 최초 요청은 느림(미스).
- **Read‑Through**  
  - 애플리케이션이 캐시에 질의하면 캐시가 **자동으로 원천**에서 로드. 구현 복잡↑.
- **Write‑Through**  
  - 쓰기 시 **캐시와 원천을 동시에** 업데이트. 쓰기 지연↑, 읽기 일관성↑.
- **Write‑Back(Write‑Behind)**  
  - 먼저 캐시에 반영 후 비동기로 원천에 적용. 지연·손실 리스크 관리 필요(내구성/큐).
- **Refresh‑Ahead**  
  - TTL 만료 전에 백그라운드 갱신. 핫키의 미스 방지.

---

## 4) 일관성/무효화
- **TTL(Time‑to‑Live)**: 오래된 데이터 제거; 너무 길면 신선도↓, 너무 짧으면 미스↑.
- **정밀 무효화**: 쓰기/삭제 시 관련 키 정확히 삭제(키 설계 중요).
- **버전 키(명명 규칙)**: `user:{id}:v{schemaVersion}` → 스키마 변경, 배포 시 캐시 버스트.
- **이벤트 기반 무효화**: CDC, 메시지 큐로 변경 이벤트를 구독해 캐시 정합성↑.
- **Read‑Your‑Writes**: 특정 사용자/세션은 리더·최신 캐시로 라우팅.

---

## 5) 키/값 설계
- **키**: 짧고 충돌 방지(네임스페이스), 파티셔닝 고려(해시/샤딩).
- **값**: 직렬화 포맷(JSON/MsgPack/ProtoBuf). 너무 큰 값은 **분할** 또는 **요약**.
- **부분 캐시(Partial)**: 자주 쓰는 필드만 캐시(선택적 직렬화).

---

## 6) 성능 & 용량 관리
- **Eviction 정책**: LRU, LFU, FIFO, TTL 기반, 세트 단위(예: Redis volatile-lru/lfu).
- **핫키(Hot Key) 완화**: 복제 키, 일시적 랜덤화(키 접미사), 샤딩/분산.
- **압축(Compression)**: 네트워크 비용↓, CPU 비용↑ → 임계값 기반 적용.
- **메트릭**: Cache Hit Ratio, Byte Hit Ratio, P50/P95/P99 Latency, Evictions, 미스율.

---

## 7) 장애/이슈 패턴 & 대응
- **Cache Stampede(동시 미스 폭주)**:  
  - 대책: **Dogpile 방지**(Mutex/Single‑flight), **Lazy/Soft TTL**, **Refresh‑Ahead**.
- **Cache Penetration(존재하지 않는 키 반복 조회)**:  
  - 대책: **Negative Caching(짧은 TTL)**, **Bloom Filter**.
- **Cache Pollution**(재사용 낮은 키가 캐시 공간 잠식):  
  - 대책: 채택 임계치(최소 hit 수 이상일 때만 캐시), LFU.
- **Thundering Herd**(대규모 만료 동시 발생):  
  - 대책: TTL **분산(Randomized TTL)**, 배치 갱신.
- **Cold Start**:  
  - 대책: **캐시 워밍**, 배포 직후 백그라운드 프리로드.

---

## 8) 분산 캐시(예: Redis/Memcached)
- **토폴로지**: 단일/샤드형/클러스터, 리더‑레플리카, Sentinel(자동 페일오버).
- **일관성**: `SETNX`/Lua로 원자적 연산, 분산 락(Redlock 주의), 트랜잭션/파이프라이닝.
- **내구성**(Redis): RDB 스냅샷, AOF(Append‑Only File), 복제본, `min-replicas-to-write`.
- **네트워크**: 연결 풀, Keep‑Alive, 패킷 사이즈, Nagle 비활성(TCP_NODELAY) 고려.
- **보안**: ACL, TLS, VPC 피어링/프라이빗 서브넷.

---

## 9) 스프링 부트 실무 팁
- **애노테이션 캐시**: `@Cacheable`, `@CacheEvict`, `@CachePut`, 조건(`unless`, `condition`).  
- **TTL 설정**: 캐시 매니저별 TTL 다르게 구성(예: `CacheManager` + `RedisCacheConfiguration`).  
- **키 스키마**: `@Cacheable(cacheNames="user", key="'id:' + #id")` 식으로 명확히.  
- **분산 락**: Redisson/Lettuce, **단일 비즈니스 키**에만 잠금 범위 최소화.  
- **원자적 갱신**: Lua 스크립트로 다중 명령 묶기(SET+EX, 체크‑앤‑셋).  
- **관측성**: Micrometer + Prometheus/Grafana로 **hit/miss/latency** 대시보드.

---

## 10) 체크리스트
- [ ] 캐시 대상 선정(재사용률, 계산 비용, 정합성 요구)  
- [ ] TTL·무효화 규칙 정의(도메인 이벤트/버전 키 포함)  
- [ ] Stampede/Her d 방지 메커니즘 적용(Mutex/Soft TTL)  
- [ ] 핫키/용량/압축 정책 수립 및 테스트  
- [ ] 장애/페일오버/데이터 유실 리허설(캐시 비우기 상황 포함)  
- [ ] 배포/스키마 변경 시 캐시 버스트 계획

---

## 11) 한 줄 요약
> 캐시는 **응답 시간을 극적으로 단축**하지만, **무효화·일관성·폭주 방지**가 성공의 핵심이다.
