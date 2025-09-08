# Unique ID 생성 
## 1) 왜 Unique ID 필요한가?
- 대규모 분산 시스템에서 각 이벤트·레코드·객체에 대해 **충돌 없는 고유 식별자** 필요  
- 요구사항:
  - 전역 고유성(Global Uniqueness)  
  - 고성능(저지연, 대량 생성)  
  - 정렬 가능 여부(시간 순서)  
  - 짧고 관리 가능한 크기  
  - 장애/분산 환경에서의 안정성  

---

## 2) 단순 접근 방식
- **데이터베이스 Auto Increment**
  - 장점: 단순, 정렬 가능
  - 단점: 단일 DB 의존 → 병목·스케일링 어려움
- **UUID (128-bit)**
  - 장점: 분산 환경에서도 충돌 거의 없음
  - 단점: 길이가 김(32자), 정렬 불가, 인덱스 효율↓
- **GUID**
  - 윈도우/마이크로소프트 계열 UUID, 유사 특성

---

## 3) 개선된 분산 ID 기법
### (1) Twitter Snowflake
- 64비트 ID 구조:
  - 41비트: Timestamp(ms)  
  - 10비트: 머신/데이터센터 ID  
  - 12비트: Sequence(같은 ms 내 카운트)  
- 장점: 정렬 가능, 짧음, 분산 환경 적합  
- 단점: 시계 동기화 필요

### (2) Instagram Sharding + DB
- 여러 DB 샤드의 Auto Increment를 활용, 샤드 ID + 시퀀스 결합
- 장점: 충돌 없음
- 단점: 샤드 관리 부담

### (3) KSUID (K-Sortable UID)
- 128비트 ID, 앞부분은 timestamp, 나머지는 랜덤
- 장점: 시간순 정렬 가능, UUID보다 분산성·정렬성 개선

### (4) ULID (Universally Unique Lexicographically Sortable ID)
- 128비트, Base32 인코딩
- 장점: 문자열 정렬 가능, URL-safe

### (5) 기타
- **MongoDB ObjectId**: 12바이트, timestamp + machine + pid + counter
- **Flake Id, Sonyflake**: Snowflake 파생 구현

---

## 4) 선택 기준
- **고성능 로그/이벤트 스트리밍** → Snowflake, KSUID  
- **범용·충돌 최소화** → UUID  
- **정렬 필요 + 짧은 문자열** → ULID, Snowflake  
- **DB 중심** → Auto Increment + Sharding  

---

## 5) 운영 이슈
- **시계 동기화 문제**
  - NTP 장애 시 ID 충돌/중복 가능 → 시계 역행 방어 로직 필요
- **분산 환경**
  - 머신/샤드 ID 충돌 방지 필요 (설정 관리)
- **저장 효율**
  - PK/인덱스에 긴 문자열 UUID 쓰면 성능 저하
  - 64비트 정수형 ID 권장
- **ID 추측 가능성**
  - 보안/프라이버시 필요 시 랜덤성 강화 (UUID v4, KSUID)

---

## 6) 한 줄 요약
> Unique ID 생성은 **분산 환경에서 전역 충돌 없는 고유성**을 제공해야 하며,  
> 성능·정렬성·저장 효율·보안 요구에 따라 **UUID, Snowflake, ULID, KSUID** 등 다양한 선택지가 존재한다.
