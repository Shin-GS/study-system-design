# Message Queue 정리

## 1) 개념
- **Message Queue (MQ)**: 비동기 통신을 위한 컴포넌트.  
  Producer(발행자)가 큐에 메시지를 넣고, Consumer(구독자)가 꺼내 처리.  
  시스템 간 결합도를 낮추고, 확장성과 내결함성을 제공.

---

## 2) 주요 목적
- **비동기 처리**: 요청-응답을 분리, 긴 작업을 백그라운드로 이동  
- **버퍼링**: 트래픽 급증 시 큐가 완충 역할 수행  
- **확장성**: 여러 Consumer가 병렬로 처리  
- **내결함성**: Consumer 장애 시 메시지를 큐에 보관 후 재처리 가능

---

## 3) 아키텍처 구성
- **Producer**: 메시지 발행 (API 서버, 서비스 A 등)  
- **Queue/Broker**: 메시지 저장·라우팅 (Kafka, RabbitMQ, SQS 등)  
- **Consumer**: 메시지 구독·처리 (서비스 B, 워커 등)

---

## 4) 전달 보장(Delivery Guarantee)
1. **At Most Once** — 최대 한 번 전달(중복 없음, 손실 가능)  
2. **At Least Once** — 최소 한 번 전달(손실 없음, **중복 가능** → 멱등성 필요)  
3. **Exactly Once** — 정확히 한 번 전달(구현 복잡, Kafka 트랜잭션 등)

---

## 5) 메시지 처리 모델
- **Queue (점대점)**: 여러 Consumer 중 **하나**가 처리(작업 분산)  
- **Pub/Sub (발행-구독)**: 메시지가 **여러 구독자**에게 동시에 전달

---

## 6) 주요 기능
- **Ack/Nack**: 성공 시 Ack, 실패 시 재전송  
- **Dead Letter Queue (DLQ)**: 반복 실패 메시지 보관  
- **Retry/Backoff**: 지수 백오프(Exponential Backoff)  
- **순서 보장(Ordering)**: 파티션/FIFO로 순서 유지  
- **지연 메시지/스케줄링**: 지정 시간 후 전달

---

## 7) 장점
- **결합도 감소** (Loose Coupling)  
- **비동기 확장** 가능 → 대규모 트래픽 처리  
- **탄력성** 확보 → 일부 Consumer 장애에도 서비스 지속  
- **백프레셔(Backpressure)** 관리 가능

---

## 8) 단점
- 시스템 복잡성 증가 (운영·모니터링 필요)  
- **중복/유실** 가능 → 멱등성(idempotency) 처리 필수  
- 지연(latency) 증가 가능  
- 운영 비용(브로커 관리, 클러스터링)

---

## 9) 대표 기술
- **RabbitMQ** — AMQP 기반, 유연한 라우팅, Ack/Nack  
- **Apache Kafka** — 대규모 스트리밍, 파티션 기반 확장성, 로그 저장 기능  
- **Amazon SQS**, **Google Pub/Sub** — 매니지드, 간단/확장성↑  
- **ActiveMQ**, **NATS**, **Pulsar** 등

---

## 10) 운영 체크리스트
- [ ] 메시지 **중복** 처리 → 멱등성 보장(`idempotent consumer`)  
- [ ] **DLQ** 모니터링 및 알림  
- [ ] **Lag** 모니터링(큐 적체) 및 자동 스케일링  
- [ ] 보안(TLS, 인증/권한)  
- [ ] **스키마 관리**(Schema Registry, Avro/Protobuf)  
- [ ] 장애 테스트(브로커 다운, Consumer 장애, 네트워크 단절)

---

## 11) 스프링 부트 힌트
- `@KafkaListener` / `@RabbitListener`로 Consumer 정의  
- **재시도/백오프**: Spring Retry, DLT(DLQ) 사용  
- **멱등성**: 비즈니스 키 기준 중복 방지(Unique Index, 토큰 테이블)

---

## 12) 한 줄 요약
> MQ는 **비동기·확장·내결함성**의 핵심이지만, **멱등성·중복·지연**을 설계 단계에서 반드시 다뤄야 한다.
