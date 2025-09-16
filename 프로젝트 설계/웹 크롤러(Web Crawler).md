# Web Crawler 설계

## 1) 목표
- 웹 크롤러는 **웹 페이지를 자동으로 수집·다운로드**하고, 콘텐츠를 파싱하여 **색인/분석 시스템에 전달**하는 시스템
- 대규모 크롤러는 **수십억 페이지**를 처리할 수 있어야 하며, **네트워크·저장소·처리 파이프라인**을 모두 고려해야 함

---

## 2) 요구사항

### 기능적 요구사항
- URL 큐로부터 URL을 가져와 **다운로드 → 파싱 → 링크 추출 → URL 큐에 추가**
- robots.txt 및 크롤링 정책 준수
- 중복 페이지 제거
- 에러/재시도 정책

### 비기능적 요구사항
- **확장성**: 수평 확장 가능 (수천 개 워커)
- **고성능**: 고병렬 다운로드, 저지연 파이프라인
- **신뢰성**: 중복·실패·네트워크 오류 복원
- **정책준수**: 크롤링 빈도/딜레이/도메인 차단

---

## 3) 아키텍처 구성요소

```
                +-----------------+
 Seed URLs ---> |  URL Frontier   | ---> [URL]
                +-----------------+
                         |
                         v
               +------------------+
               |   Fetcher Pool   | ---> robots.txt 검사
               +------------------+
                         |
                         v
               +------------------+
               |    Parser/ETL    | ---> 링크 추출, 콘텐츠 파싱
               +------------------+
                         |
                         v
                +----------------+
                |  Storage/Index |
                +----------------+
```

- **URL Frontier (Queue)**: BFS/priority 기반 URL 관리
- **Fetcher Pool**: 다수의 병렬 워커 → HTTP 요청, robots.txt 확인
- **Parser/ETL**: HTML 파싱, 링크 추출, 정규화, 중복 제거
- **Storage**: 다운로드한 콘텐츠, 메타데이터 저장 (HDFS, S3 등)

---

## 4) 핵심 설계 포인트

### URL Frontier
- 우선순위 큐 기반 (PageRank, Freshness 점수)
- 도메인별 rate limiting (호스트별 1~2req/s)
- 중복방지: **Bloom Filter / Hash Set**

### Fetcher
- HTTP/HTTPS 지원, 헤더·쿠키 관리
- robots.txt 파싱 및 정책 준수
- 압축 컨텐츠(Gzip) 처리
- 실패 재시도, 백오프(backoff)

### Parser
- HTML 파싱 (DOM → 링크 추출)
- canonical URL 정규화
- 콘텐츠 압축/토큰화

---

## 5) 확장성 & 분산 설계
- **Sharding**: URL을 도메인/해시 기준으로 분산
- **Kafka 등 메시지 큐**: URL Frontier와 Fetcher 간 decoupling
- **분산 저장소**: HDFS, S3, Bigtable, Elasticsearch 등
- **Scheduler**: 주기적 재크롤링 스케줄 관리

---

## 6) 중복 제거 & 무한 크롤링 방지
- **URL Normalization**: 쿼리 파라미터/프래그먼트 제거
- **Content Hash (SHA-256)**: 동일 콘텐츠 판별
- **Visited Set / Bloom Filter**: 방문 이력 관리
- **Crawl Budget 정책**: 사이트당 최대 페이지 수 제한

---

## 7) 정책 준수 & 윤리
- robots.txt, noindex, meta robots 등 준수
- 사이트 소유자 요청 시 차단
- User-Agent 식별 제공
- 과도한 요청 방지 (도메인당 rate-limit)

---

## 8) 관측성 & 운영
- Metrics: Fetch QPS, Success/Error 비율, 평균 페이지 크기
- Logging: URL, Status, Latency, Retry
- 경보(알람): 에러율, 스로틀링 상태
- 백오프 및 서킷브레이커로 장애 도메인 격리

---

## 9) 성능 최적화
- HTTP Keep-Alive, 커넥션 풀
- 압축 다운로드(Gzip/Deflate)
- 콘텐츠 압축 저장(Zstd, Snappy)
- 파서/인덱서 비동기화, 파이프라인 병렬화

---

## 10) 한 줄 요약
> Web Crawler는 **URL Frontier–Fetcher–Parser 파이프라인**을 중심으로 동작하며,  
> **확장성·정책 준수·중복 제거·관측성**이 핵심 설계 포인트다.
