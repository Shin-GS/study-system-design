# 동적 설정(Dynamic Configuration)

## 1) 정의
- **Dynamic Configuration**:  
  - 애플리케이션을 재배포하지 않고 **시스템 동작을 즉시 변경·조정**할 수 있는 기능  
  - 운영 중인 서비스의 **튜닝, 기능 플래그, 실험 제어** 등에 활용됨  

---

## 2) 활용 사례
- **A/B 테스트** (Online Controlled Experimentation)  
  - 노출 비율 동적 조정 (Exposure rate adjustment)  
- **Feature Gateway / Feature Flag**  
  - 특정 기능을 점진적으로 활성화 또는 비활성화  
- **시스템 튜닝**  
  - 임계치, timeout 값, 자원 제한 등을 실시간 변경  

---

## 3) 요구사항
- **구현 단순화**: 문자열 기반 속성 위주  
- **대규모 구성 지원**: 설정 크기가 수십 MB 수준 가능  
- **갱신 지연(latency)**: < 10초 이내 업데이트 전파  
- **대규모 확장성**: 최대 **100만 인스턴스**까지 확장 가능  
- **고가용성**: 설정 서버 장애 시에도 지속 동작 가능  

---

## 4) 설계 패턴
### Key-Value 기반 단순 설계
- 중앙의 **Key-Value 저장소**에 속성을 저장  
- 클라이언트는 주기적으로 **polling** 하여 값 갱신 확인 (Pull 모델)  
- 예: Netflix **Archaius**  

### Observer Pattern
- 주체(Subject)가 다수의 옵저버(Observers)를 등록·관리  
- 설정 변경 발생 시 옵저버들에게 자동 알림 (Push 모델)  
- → Pub/Sub 시스템(Kafka, Redis PubSub, gRPC Stream)으로 확장 가능  

---

## 5) 분산 환경 고려
- **데이터 일관성**: 모든 인스턴스에 동일한 설정이 반영되어야 함  
- **성능**: Config 서버 과부하 방지 → 캐싱, 샤딩, CDN 활용  
- **내결함성**: 장애 시 fallback 값 사용, 최근 캐시된 값 유지  
- **보안**: 설정 값에 비밀정보(secret)가 포함될 경우 → 암호화 및 접근 제어 필수  

---

## 6) 운영 이슈
- **롤백 전략**: 잘못된 설정 적용 시 즉시 이전 버전으로 되돌릴 수 있어야 함  
- **버전 관리**: Config 변경 이력 추적, 감사 로그(Audit Log)  
- **관측성**: Config 변경 이벤트 알림, 메트릭(갱신 성공률, 전파 시간)  

---

## 7) 구현 예시
- **Key-Value 저장소**: Consul, Etcd, Zookeeper, Redis  
- **클라우드 서비스**: AWS AppConfig, Azure App Configuration, Spring Cloud Config  
- **프레임워크/라이브러리**: Netflix Archaius, Spring Cloud Config Client  

---

## 8) 한 줄 요약
> Dynamic Configuration은 **재배포 없는 실시간 시스템 제어**를 가능하게 하며,  
> 대규모 환경에서는 **저지연 전파, 고가용성, 안전한 롤백**이 핵심이다.
