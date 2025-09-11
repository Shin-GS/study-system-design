# URL Shortener 설계
## 1) 목표와 요구사항

### 기능적 요구사항 (FR)
- 긴 URL → **짧은 코드** 생성 (단축 URL 발급)
- 단축 URL 접속 시 **원본 URL로 리다이렉트**
- **커스텀 별칭**(선택) 지원: `https://sho.rt/{alias}`
- **만료(Expiration/TTL)** 옵션: 특정 기간/클릭 수 이후 무효화
- (선택) **분석/통계**: 클릭 수, 지역/UA, 참조(Referrer), 최근 사용 이력

### 비기능적 요구사항 (NFR)
- **낮은 지연**(p95 ~ 수 ms 단위) 리다이렉트
- **고가용성**(HA), **수평 확장**(Horizontal scaling)
- **데이터 정합성**: 생성/조회 경합 시 충돌 방지
- **비용 절감**: 저장/네트워크 효율 고려
- **악용 방지**: Rate limit, 스팸/피싱 링크 차단

---

## 2) API 설계 (예시)

### 단축 URL 생성
```
POST /api/shorten
Body: { "long_url": "https://example.com/a?b=c", "custom_alias": "promo-2025", "expire_at": "2026-01-01T00:00:00Z" }
Res : { "short_url": "https://sho.rt/Ab3xYz" }
```

### 리다이렉트
```
GET /{code|alias}  ->  301/302 Location: original_url
```

### 통계/관리 (선택)
```
GET  /api/urls/{code}/stats
POST /api/urls/{code}/disable
```

---

## 3) 코드/키 설계

### 길이/문자셋
- **Base62**: [0-9a-zA-Z] → URL-safe, 짧고 가독성 양호
- 길이 6~8 문자를 권장 (62^6 ≈ 56.8B 조합)

### 생성 방식
1) **시퀀스 기반 + Base62 인코딩**
   - 전역 증가 시퀀스(분산 ID: Snowflake/ULID) → Base62 변환
   - **장점**: 충돌 없음, 순차성(캐시/저장 효율)
   - **단점**: 시계 역행/워커 ID 충돌 주의(분산 ID 운용 이슈)

2) **해시 기반**
   - `hash(long_url)` (SHA‑256/MD5) → **접두 n바이트만 사용**
   - **장점**: 구현 단순, 중복 URL에 동일 코드
   - **단점**: 충돌 가능성(짧게 자를수록↑) → **충돌 시 재시도/확장**

3) **랜덤 토큰**
   - `secureRandom(n bytes)` → Base62
   - **장점**: 추측 난이도↑, 보안 우수
   - **단점**: 존재 여부 조회 필요 → **중복 체크 비용**

4) **커스텀 별칭**
   - 사용자 입력 시 **정규식/블랙리스트**로 유효성 검증(욕설/피싱 차단)

### 충돌 처리
- **Upsert with unique index**: `(code)` 유니크, 충돌 시 리트라이
- **Hash+Salt**: 동일 URL의 코드 충돌 회피
- **길이 확장**: 충돌율 높아지면 6→7→8자 확대

---

## 4) 저장소 & 스키마

### 기본 스키마(관계형 모델 예)
```
TABLE url_map (
  code        VARCHAR(10) PRIMARY KEY,
  long_url    TEXT NOT NULL,
  created_at  TIMESTAMP NOT NULL,
  expire_at   TIMESTAMP NULL,
  owner_id    BIGINT NULL,
  is_active   BOOLEAN DEFAULT TRUE,
  meta_hash   CHAR(32) NULL,        -- (옵션) long_url 해시
  click_count BIGINT DEFAULT 0
);

INDEX idx_owner_created_at (owner_id, created_at DESC);
INDEX idx_expire_at (expire_at);
```

### 선택 스키마(분석/로그)
- `click_log(code, ts, ip, ua, ref, country)` → **배치/스트림 집계** 후 요약 테이블로 반영
- 고QPS 환경: 원클릭마다 DB 쓰기 대신 **비동기 큐(Kafka)** 로 수집

### 저장소 선택
- **핵심 매핑**: **Key-Value 스토어**(code → long_url, TTL) 또는 **RDB**(+캐시)
- **분석 로그**: 객체 스토리지 + 스트리밍/OLAP(ClickHouse/BigQuery)

---

## 5) 읽기 경로 (핫패스) 최적화

1) **캐시 우선 (Cache-aside)**
   - `GET /{code}` 처리: **Redis GET(code)** → miss 시 DB 조회 → 캐시에 채우고 301/302
   - TTL과 **버전 키**로 무효화 제어

2) **CDN/엣지 캐시**
   - `/{code}` 경로도 에지에서 **짧은 TTL**로 캐시 가능(활성/만료 여부 헤더와 함께)

3) **HTTP 리다이렉트 선택**
   - **302**: 임시(통계/실험 시), **301**: 영구(캐시 강함)  
   - 검색엔진/브라우저 캐시 전략 고려

---

## 6) 쓰기 경로 (단축 생성)

- **코드 생성** → **유니크 인덱스 충족** 확인 → **DB/KV 저장** → (옵션) **캐시 preload**
- **Idempotency**: 동일 long_url 요청의 중복 생성 방지(요청 키/해시로 멱등 처리)

---

## 7) 확장/파티셔닝

- **샤딩 키**: `code` 또는 `hash(code)` → 균등 분산 (Consistent Hashing 권장)
- **읽기 스케일아웃**: 레플리카/캐시/에지 사용
- **쓰기 스케일아웃**: 멀티 리전 시 **리더 선출** 또는 **멀티-리더 + 충돌해결**

---

## 8) 만료/비활성/정책

- **TTL/expire_at** 경과 시 비활성 처리: 캐시 제거 + 404/410 반환
- **비활성/삭제**: `is_active = false` → 즉시 차단, 캐시 무효화
- **블랙리스트**: 피싱/악성 도메인 필터링, Safe Browsing API 연계

---

## 9) 관측성 & 운영

- 메트릭: **QPS, p95/p99 latency, cache hit ratio, redirect 성공률, 4xx/5xx**
- 로그/추적: 요청ID, code, UA, Referrer
- **Rate Limiting**: 생성 API/리다이렉트에 차등 적용(IP, 토큰, 클라이언트별)
- **백업/보존**: url_map 스냅샷, 만료 URL의 아카이브 정책
- **런북**: 코드 충돌 급증, 캐시 대규모 만료(스탬피드), 피싱 탐지 시나리오

---

## 10) 보안

- **도메인 보호**: HSTS, TLS 강제, CAA 레코드
- **악성 링크 차단**: 실시간 검사/샌드박스 리졸브
- **오너 권한**: 소유자만 수정/삭제, 관리자 정책

---

## 11) 백오브더엔벨롭 계산 (예시)

- **생성 QPS**: 5k/s, **조회 QPS**: 100k/s (읽기:쓰기 = 20:1)
- **캐시 히트율**: 0.95 → 원본 DB 읽기 5k/s
- **코드 길이 7(Base62)**: 62^7 ≈ 3.5e12 (여유 충분)
- **스토리지**: 평균 URL 120 bytes, 메타 포함 256 bytes
  - 일 5M 생성 → 5M × 256B ≈ 1.2 GB/일 → 1년 ≈ 438 GB(+복제/인덱스 2×)

---

## 12) 예시 아키텍처

```
Client ─▶ CDN/Edge ─▶ API Gateway
                      ├─▶ Redirect Service ─▶ Redis(Cache) ─▶ DB/Key-Value
                      └─▶ Shorten Service  ─▶ ID Service    ─▶ DB
                                     └─▶ Kafka(Click Log) ─▶ OLAP/BI
```

---

## 13) 체크리스트

- [ ] Base62/길이/커스텀 별칭 정책 정의
- [ ] 충돌/중복 방지(유니크 인덱스 + 리트라이, 멱등키)
- [ ] 캐시 전략(hit≥90%), 스탬피드 방어(soft TTL/lock)
- [ ] 악성 URL 필터링, Rate limit/WAF
- [ ] 만료/삭제 정책과 캐시 무효화
- [ ] 관측성 대시보드와 알림, 데이터 백업/보존
- [ ] 샤딩/복제/리밸런싱 계획(Consistent Hashing)

---

## 14) 한 줄 요약
> URL Shortener는 **고QPS 읽기-중심** 워크로드로, **짧고 충돌 없는 코드 생성 + 캐시/샤딩/보안**이 핵심이다.
