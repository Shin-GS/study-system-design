# Notification Service

## 1) 목표
- 다양한 채널(SMS, Email, Push, In-app)을 통해 사용자에게 알림을 안정적이고 확장 가능하게 전달  
- **핵심 요구사항**:
  - 대규모 트래픽 처리 (수백만/수천만 사용자)  
  - 지연 최소화 (Near real-time)  
  - 실패 시 재시도 및 보장 전달 (At-least-once)  
  - 채널별 확장성과 유연한 정책  

---

## 2) 아키텍처 개요
```
Producer → Queue/Topic → Notification Service → Channel Provider(SMS/Email/Push) → User
```

- **Producer**: 애플리케이션에서 알림 요청 발생 (예: 주문 완료, 친구 요청)  
- **Queue/Topic (Kafka, RabbitMQ, SQS)**: 비동기 메시징으로 확장성 확보  
- **Notification Service**:
  - 채널별 어댑터 (SMS, Email, Push, Webhook)  
  - 우선순위/스케줄링/템플릿 처리  
- **Channel Provider**: 외부 게이트웨이(Twilio, FCM, SES 등)  

---

## 3) 기능 요구사항
- 멀티 채널 지원 (SMS, Email, Push, Web)  
- 사용자 선호도 기반 라우팅 (예: 이메일 선호 vs SMS)  
- 재시도/백오프 정책 (exponential backoff)  
- 템플릿/국제화(i18n) 지원  
- 스케줄링 (즉시 vs 예약 알림)  
- 멀티 리전 배포 (재해 복구, 레이턴시 최소화)  

---

## 4) 저장소 설계
- **Notification Log**: 알림 요청/상태 저장  
  - `id, user_id, channel, payload, status, retry_count, created_at`  
- **User Preference Store**: 채널별 구독/거부 여부  
- **Template Store**: 다국어/다채널 메시지 템플릿  

---

## 5) 장애 처리 & 보장
- **At-least-once Delivery**: 큐 재처리 기반  
- **Idempotency Key**: 중복 전송 방지  
- **Dead Letter Queue (DLQ)**: 실패 메시지 보관  
- **백업 채널(Fallback)**: Push 실패 → SMS 전송  

---

## 6) 성능/확장 전략
- **수평 확장**: Notification worker pool 확장  
- **Batch 전송**: 대량 알림 시 provider API 최적화  
- **Rate Limiting**: provider API 쿼터 초과 방지  
- **분산 트레이싱**: 알림 전송 전체 흐름 추적  

---

## 7) 운영 & 모니터링
- 지표: 전송 성공률, 지연 시간, 재시도 횟수  
- 알림 채널별 모니터링 (SMS/Email/Push)  
- 사용자 불만/스팸 리포트 처리  
- SLA 기반 알림 보장 (예: 99.9% 5초 내 도착)  

---

## 8) 예시 아키텍처
```
App → Kafka Topic → Notification Service
         ├─ SMS Adapter → Twilio
         ├─ Email Adapter → SES
         ├─ Push Adapter → FCM/APNS
         └─ In-App/WebSocket
```

---

## 9) 한 줄 요약
> Notification Service는 **비동기 큐 기반의 멀티 채널 아키텍처**로,  
> **확장성, 보장 전달, 사용자 선호 반영, 모니터링**이 핵심이다.
