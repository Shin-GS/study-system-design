# 채팅 시스템(Chat System) 설계

## 1) 목표
- 사용자 간 **실시간 메시지 전송**을 안정적으로 지원
- 수백만 동시 연결을 처리할 수 있는 확장성 확보
- 기능 요구:
  - 1:1 채팅, 그룹 채팅
  - 메시지 읽음 상태(Seen/Delivered)
  - 푸시 알림, 오프라인 메시지 보관
  - 낮은 지연시간 (수십~수백 ms 이내)

---

## 2) 아키텍처 개요

```
Client (Web/Mobile)
        │ WebSocket/HTTP
        ▼
  Chat Gateway (Load Balancer)
        │
        ▼
  Chat Server Cluster ──► Message Queue (Kafka)
        │                       │
        ▼                       ▼
Persistence Store          Notification Service
        │
        ▼
   Search/Analytics
```

- **Client**: WebSocket/HTTP/GRPC로 연결  
- **Gateway/Load Balancer**: 세션 라우팅  
- **Chat Server Cluster**: 메시지 수신/전송 처리  
- **Message Queue**: 비동기 전송/재시도/순서 보장  
- **Persistence Store**: MongoDB, Cassandra 등 분산 저장소  
- **Notification Service**: 오프라인 사용자 푸시 알림  

---

## 3) 메시지 전달 모델

### 1) 실시간 전송
- WebSocket 기반 양방향 통신
- 낮은 지연, 연결 유지 필요

### 2) 저장 후 전달(Store & Forward)
- 사용자가 오프라인일 경우 DB/큐에 저장
- 온라인 시 전달 (Push Notification 포함)

### 3) 보장 전달 (Reliability)
- **At-least-once** 보장 (재전송, ACK 기반)
- **Idempotency** 필요 (메시지 중복 제거)

---

## 4) 데이터 모델 (예시)
```
TABLE messages (
  msg_id       STRING (UUID/ULID),
  chat_id      STRING,   -- 1:1 또는 그룹
  sender_id    STRING,
  content      TEXT/JSON,
  created_at   TIMESTAMP,
  delivered_at TIMESTAMP,
  seen_at      TIMESTAMP,
  status       ENUM('SENT','DELIVERED','SEEN')
);
```

- **Partitioning**: chat_id 기반 샤딩  
- **Index**: (chat_id, created_at) 조합으로 대화 조회 최적화  

---

## 5) 그룹 채팅 고려사항
- 대규모 그룹(수천 명) → fan-out 비용 ↑
- 해결:
  - **메시지 큐 브로커** 기반 멀티캐스트
  - 그룹 멤버 리스트 캐싱
  - 읽음 상태 집계(Aggregation)만 저장

---

## 6) 확장성 전략
- **Sharding**: chat_id/hash 기반 서버 분산
- **Replication**: 다중 데이터센터에 복제
- **CQRS 패턴**: 쓰기(메시지 저장)와 읽기(타임라인 조회) 분리
- **캐싱**: 최근 메시지는 Redis에 저장 후 빠른 응답

---

## 7) 장애 복구 & 운영
- Dead Letter Queue (DLQ)로 실패 메시지 처리
- 재시도 정책 (exponential backoff)
- 모니터링: 메시지 전송률, 지연 시간, 실패율
- 알림 시스템과 연계 (Slack/PagerDuty)

---

## 8) 보안
- 전송 암호화 (TLS, E2EE 선택적 적용)
- 사용자 인증/인가 (JWT, OAuth2)
- 스팸/악성 메시지 필터링
- 메시지 보존 정책 (GDPR/개인정보 보호 준수)

---

## 9) 백오브더엔벨롭 계산 (예시)
- **1억 사용자**, 동시접속 1백만, 50 QPS
- 메시지 크기 평균 1KB → 초당 50MB 처리
- 일간: 4.3TB 저장 필요 (복제 전)
- 저장소: Cassandra/HBase, SSD 기반 클러스터

---

## 10) 한 줄 요약
> 채팅 시스템은 **WebSocket 기반 실시간 전송 + 저장 후 전달 모델**을 결합하여,  
> **확장성·신뢰성·보안성**을 모두 고려한 설계가 필요하다.
