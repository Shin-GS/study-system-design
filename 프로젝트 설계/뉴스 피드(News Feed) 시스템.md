# News Feed 설계
## 1) 목표
- 사용자에게 **개인화된 뉴스피드(Feed)**를 효율적으로 제공
- 수백만/수억 사용자 규모에서 확장 가능해야 함
- 주요 요구사항:
  - 지연(latency) 최소화 (수백 ms 이내)
  - 최신성 + 개인화 균형
  - 대규모 데이터 처리 및 저장소 최적화

---

## 2) 요구사항

### 기능적 요구사항
- 팔로우한 친구/페이지의 새 포스트 표시
- 인기/추천 포스트 반영
- 좋아요/댓글/공유 이벤트 반영
- 무한 스크롤(페이지네이션)
- 랭킹/정렬 알고리즘 기반의 개인화

### 비기능적 요구사항
- 확장성(수억 사용자)
- 낮은 응답 시간
- 데이터 일관성(Eventual consistency 허용)
- 고가용성, 장애 복원력

---

## 3) 아키텍처 개요

```
Producer (Post, Like, Comment)
        │
        ▼
 Message Queue (Kafka)
        │
        ▼
Fan-out Service ───► Timeline Store (per user feed)
        │
        └──► Ranking Service ─► Cache ─► API (Feed delivery)
```

- **Producer**: 포스트/이벤트 생성
- **Queue**: Kafka 등으로 비동기 전송
- **Fan-out Service**: 팔로워별 feed 반영 (push 기반)
- **Timeline Store**: 사용자별 타임라인 저장소
- **Ranking Service**: 머신러닝/규칙 기반 랭킹
- **Cache**: Redis/memcached, 빠른 응답 제공

---

## 4) Fan-out 전략

### Push model
- 새 포스트 생성 시 **팔로워별 타임라인에 즉시 기록**
- 장점: 읽기 빠름
- 단점: 쓰기 비용 ↑ (팔로워 수가 많을수록)

### Pull model
- 사용자가 feed 요청 시 **팔로우 관계 기반으로 on-demand 조회**
- 장점: 쓰기 비용 낮음
- 단점: 읽기 지연 ↑, 고QPS 대응 어려움

### Hybrid model
- 일반 사용자는 **Push**
- 슈퍼스타(팔로워 수 많은 계정)는 **Pull**
- 실제 SNS(Facebook, Twitter 등)에서 활용

---

## 5) Timeline Store 설계
- Key-Value 또는 Wide-column DB (Cassandra, HBase) 사용
- Key: user_id, Value: 포스트 ID 리스트
- TTL/Sliding Window: 최근 N개만 유지 (예: 1000개)
- Secondary Index: 최신성/인기도 기반 조회

---

## 6) Ranking & Personalization
- **랭킹 알고리즘**:
  - 최신성 (Recency)
  - 인기(Engagement: 좋아요/댓글/공유 수)
  - 개인화 점수 (사용자-포스트 상관성)
- **머신러닝 모델**:
  - Feature: 사용자 행동 로그, 콘텐츠 특성
  - Output: Relevance score
- **실시간 업데이트**:
  - 사용자의 행동(좋아요/댓글)이 랭킹 점수에 반영

---

## 7) 캐싱 전략
- **User Feed Cache**: Redis에 최근 feed 저장
- **Hot Post Cache**: 인기글/슈퍼스타 포스트 캐싱
- **Write-through**: Fan-out 시 cache에도 기록

---

## 8) 장애 대응 & 운영
- 메시지 큐에 **Retry/DLQ** 적용
- Timeline store 샤딩 (user_id 해시 기반)
- 백필(backfill) 작업으로 유실 복구
- 메트릭: Feed QPS, Latency, Ranking hit ratio

---

## 9) 백오브더엔벨롭 계산 (예시)
- 사용자 1억 명, 하루 1억 포스트
- 평균 팔로워 수 = 200명 → 하루 200억 feed 이벤트
- QPS: 200억 / 86400초 ≈ 23만 ops/s
- Timeline store 용량: 포스트 ID(8B) × 200억 ≈ 160 GB/일 (압축 전)

---

## 10) 한 줄 요약
> News Feed 시스템은 **대규모 fan-out + 랭킹/개인화 + 캐싱 전략**이 핵심이며,  
> Push/Pull 하이브리드 접근으로 확장성과 효율성을 확보한다.
