# Microservices Architecture

## 1) 개념
- **Microservices Architecture**:  
  - 애플리케이션을 여러 개의 작은 독립 서비스로 분리하여 개발·배포·운영하는 아키텍처 스타일  
  - 각 서비스는 **단일 비즈니스 기능**에 집중하며, 독립적으로 배포 가능  
  - 서비스 간 통신은 주로 **경량 프로토콜(HTTP/REST, gRPC, 메시징 큐)** 사용   

---

## 2) 특징
- **Loose Coupling (약결합)**: 서비스 간 의존성 최소화  
- **High Cohesion (강한 응집도)**: 하나의 서비스가 특정 도메인 기능에 집중  
- **Autonomous (독립성)**: 독자적인 배포·스케일링 가능  
- **Polyglot 가능**: 각 서비스가 다른 언어/프레임워크 사용 가능  

---

## 3) 장점
- **확장성(Scalability)**: 특정 서비스만 독립적으로 수평 확장 가능  
- **유연한 배포(Deployment Flexibility)**: 무중단 배포, CI/CD 용이  
- **장애 격리(Fault Isolation)**: 한 서비스 장애가 전체 시스템에 미치는 영향 최소화  
- **기술 다양성**: 팀별로 최적의 기술 스택 선택 가능  
- **조직 구조 정렬**: 팀별 서비스 소유 → 애자일/DevOps 친화적  

---

## 4) 단점/도전 과제
- **운영 복잡성 증가**: 수십~수백 개의 서비스 관리 필요  
- **네트워크 비용 증가**: 서비스 간 통신 오버헤드  
- **데이터 일관성 문제**: 분산 트랜잭션 복잡  
- **테스트 어려움**: 통합 테스트 복잡도 증가  
- **관측성 요구**: 로깅, 모니터링, 트레이싱 필수  

---

## 5) 핵심 설계 고려사항
1. **서비스 경계 정의 (Bounded Context)**  
   - 도메인 주도 설계(DDD) 기반 분할  
2. **API 설계 & 통신**  
   - REST, gRPC, 메시징 (Kafka, RabbitMQ)  
3. **데이터베이스 분리 (Database per Service)**  
   - 각 서비스가 고유 DB 관리  
   - 데이터 일관성은 Eventual Consistency / Saga Pattern 활용  
4. **서비스 등록 & 발견 (Service Discovery)**  
   - Consul, Eureka, Kubernetes DNS  
5. **구성 관리 (Configuration Management)**  
   - Central Config Server, Dynamic Config  
6. **장애 내성 (Resilience)**  
   - Circuit Breaker, Retry, Rate Limiting, Bulkhead  
7. **보안(Security)**  
   - API Gateway, OAuth2/JWT 기반 인증/인가  

---

## 6) 운영 인프라
- **API Gateway**: 인증, 라우팅, Rate limiting, 로깅  
- **Service Mesh (Istio, Linkerd)**: 서비스 간 통신 제어, 보안, 모니터링  
- **CI/CD**: 독립적 빌드·배포 자동화  
- **Observability**: 로그 집계(ELK), 모니터링(Prometheus), 트레이싱(Jaeger, Zipkin)  

---

## 7) 적용 사례
- Netflix: API Gateway + 수천 개의 마이크로서비스 운영  
- Amazon: 팀별 독립 서비스 → 빠른 배포 가능  
- Uber: Monolith → Microservices 전환 후 확장성 확보  

---

## 8) 한 줄 요약
> Microservices는 **대규모 애플리케이션을 독립적이고 작은 서비스 단위로 나누어 확장성과 유연성을 확보하는 아키텍처**이지만, **운영 복잡성과 데이터 일관성** 문제를 반드시 고려해야 한다.
