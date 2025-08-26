# 대규모 서비스 로깅 정리 (Logging at Scale)

## 1) 목적과 원칙
- **문제 식별/진단**: 장애 원인 추적, 회귀 분석, 성능 병목 파악
- **관측성 3요소 연계**: Logs ↔ Metrics ↔ Traces (상호 점프 가능)
- **보안/규정 준수**: PII/민감정보 최소 수집, 마스킹/익명화, 보존 정책
- **비용/성능 균형**: 로깅은 “필요 충분 + 후회 최소화” 수준으로 운영

---

## 2) 로그 레벨/정책
- **FATAL/ERROR**: 사용자 영향/데이터 손상 가능. 페이지/알림 연동
- **WARN**: 비정상 시그널(자가복구 가능). 대시보드에서 추이 관리
- **INFO**: 상태 전이, 주요 비즈니스 이벤트
- **DEBUG/TRACE**: 상세 진단용. **샘플링/조건부 활성화** 권장
- **정책**: 서비스별 **레벨 가드레일**(예: 프로덕션 기본 INFO, 기능단 DEBUG 토글)
- **동적 레벨 변경**: 런타임에 모듈/패키지별 레벨 조정(관리자 API)

---

## 3) 구조화 로그 (JSON) 권장
- 포맷 예시(JSON 한 줄):  
  ```json
  {"ts":"2025-08-26T05:10:00Z","level":"INFO","service":"checkout","env":"prod",
   "trace_id":"f1c9...","span_id":"a12b...","user_id_hash":"u:9f2...",
   "event":"payment.authorized","order_id":"o_123","latency_ms":132}
  ```
- **필수 키**: `ts`, `level`, `service`, `env`, `host/pod`, `trace_id/span_id`, `req_id`, `endpoint`, `status`, `latency`, `error.class/message`
- **비즈니스 키**: 주문/결제/회원 등 **도메인 식별자**(민감정보는 해싱/토큰화)
- **로테이션**: 줄바꿈 금지(한 줄 JSON), 크기 제한(메시지 분할/압축)

---

## 4) 상관관계 & 추적 연동
- **Correlation/Request ID**: 요청 단위 ID 부여, **다운스트림으로 전파**
- **분산 추적**: **OpenTelemetry**(W3C Trace Context)로 `trace_id/span_id` 생성/전파
- **로그↔트레이스 조인**: 로그에 `trace_id` 포함 → APM(Jeager/Tempo/X-Ray)에서 점프

---

## 5) 샘플링 전략
- **헤드 샘플링**: 최초 진입 시 확률 적용(예: 1%)
- **테일 샘플링**: 에러/지연 큰 요청은 **강제 보존**
- **동적 샘플링**: 트래픽 급증/비용 초과 시 비율 자동 조정
- **에러 로그 예외**: ERROR는 **무샘플**(항상 수집) 원칙 권장

---

## 6) 개인정보/보안
- **금지**: 비밀번호, 카드 PAN/CVV, 주민/여권번호, 민감 PII 원문 로깅
- **마스킹/토큰화**: 이메일/전화/주소는 규칙 기반 마스킹(`jo***@mail.com`)
- **비식별화**: 해시+솔트, K-anonymity 고려
- **규정**: 보존 기간(예: 30~90일 핫, 1년 콜드), 삭제권(DSR) 처리 경로
- **접근 통제**: RBAC, 감사 로그(Audit), IP/VPC 격리, 전송/휴지 시 암호화

---

## 7) 중앙화 파이프라인
- **수집(Agent/Sidecar)**: Fluent Bit/Vector/Filebeat/Datadog Agent
- **전송**: gRPC/HTTP + 압축 + 배치; **백프레셔**와 재시도/버퍼 튜닝
- **집계/저장**: ELK/EFK(OpenSearch), Loki, ClickHouse, Datadog/CloudWatch/Stackdriver
- **인덱스 수명주기(ILM)**: 핫(SSD) → 웜 → 콜드(오브젝트 스토리지) → 삭제
- **스키마 관리**: 필드 타입 고정, 스키마 진화(추가는 OK, 삭제/변경 신중)

---

## 8) 쿼리/지표
- **핵심 지표**: ERROR Rate, WARN Rate, 요청별 Log Events 수, Log Size GB/day
- **성능**: p50/p95/p99 **latency 로그**(엔드포인트별), 상위 에러 클래스 Top N
- **비용 감시**: 인덱스 크기, Ingest 실패율, 압축률
- **예시 쿼리**(Loki/LogQL 유사):
  ```
  {service="api", level="ERROR"} |= "NullPointerException" | unwrap latency_ms | avg by (endpoint)
  ```

---

## 9) 운영 시나리오
- **장애 트리아지**: 최근 배포 버전 태그로 필터 → 에러 스파이크 원인 식별
- **릴리즈 검증**: 신규 기능 플래그 켠 뒤 WARN/ERROR 추이/샘플링률 확인
- **성능 병목**: 느린 쿼리/외부 호출 구간 span ↔ 로그 상관분석
- **보안 사고**: 인증 실패/비정상 트래픽 패턴 탐지, IP/UA 서명

---

## 10) Kubernetes/클라우드 팁
- **stdout/stderr 수집 표준화**(사이드카 대신 데몬셋 에이전트 권장)
- **Pod 메타데이터**: `namespace`, `pod`, `container`, `node`, `image`, `revision`
- **로그 손실 방지**: 임시 디스크 제한/압력 대응, 파일 핸들/롤링 파라미터 튜닝
- **멀티리전**: 리전별 버퍼/집계 후 중앙 아카이브(네트워크 비용 절감)

---

## 11) Spring Boot 실무
- **JSON 로거**: logback + `logstash-logback-encoder` 또는 `spring-boot-starter-actuator` + Micrometer
- **필드 주입**: `MDC`로 `trace_id`, `req_id`, `user` 주입(필터/인터셉터에서 세팅)
- **동적 레벨**: `/actuator/loggers` 로 패키지별 레벨 변경
- **샘플링**: OpenTelemetry SDK + `ParentBased(root=TRACE:0.01)` 등
- **민감정보 필터**: `MaskingCompositeConverter`, `Desensitization` 유틸
- **쿼리 로깅**: 느린 쿼리만 샘플링(`p6spy`, Hibernate `org.hibernate.SQL` 제한)

---

## 12) 품질/테스트
- **카나리/스테이징 비교**: 에러/경고 비율, 필드 누락 검증
- **Chaos/Load 테스트**: 폭주 상황에서 에이전트/브로커 백프레셔 확인
- **리그레션**: 배포 전 필드 스키마 검증(계약 테스트)

---

## 13) 대시보드 & 알림
- **대시보드**: 서비스/엔드포인트별 ERROR Rate, p99 latency, 최근 배포 태그
- **알림 기준**: 
  - ERROR Rate 3배↑(5분 이동평균) 
  - 신규 에러 시그니처 등장(에러 핑거프린팅)
  - 로그 유입 중단/지연(수집 파이프라인 장애)
- **런북 링크**: 알림 → 런북/Jira 자동 연결

---

## 14) 비용 절감 레시피
- **필드 다이어트**: 중복/대형 필드 제거, 스택트레이스 샘플링
- **압축/Batch**: ingest 단계에서 압축+배치 전송
- **TTL 축소**: 핫 7~14일, 웜 30~90일, 콜드 6~12개월
- **요청 샘플링**: 정상 요청 일부만 보존, 에러/지연 초과는 항상 보존

---

## 15) 체크리스트
- [ ] JSON 구조화 로그, 필수 키 표준화
- [ ] Correlation/Trace ID 전파 및 로그-트레이스 조인
- [ ] 샘플링(정상↓/이상↑) 및 동적 레벨 조정 체계
- [ ] PII 마스킹/토큰화, 보존/삭제 정책, 접근 제어
- [ ] 중앙 수집 파이프라인(재시도/버퍼/백프레셔) 구성
- [ ] 대시보드/알림/런북 연결, 비용 SLO 정의(GB/일)
- [ ] 배포 전 스키마/필드 검증, 스테이징에서 쿼리 성능 점검

---

## 16) 한 줄 요약
> 대규모 로깅은 **구조화 + 상관관계 + 샘플링 + 보안/비용 관리**가 전부다—이 4가지를 자동화하면 운영이 편해진다.
