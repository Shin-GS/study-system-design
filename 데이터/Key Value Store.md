# Key-Value Store

## 1) 정의
- **Key-Value Store**는 데이터를 `(Key, Value)` 쌍으로 저장하고 조회하는 단순한 데이터베이스 모델  
- 특징:
  - 빠른 읽기/쓰기 (O(1) 수준 접근)  
  - 스키마가 단순하여 유연성 높음  
  - 대규모 분산 환경에 적합  

---

## 2) 주요 요구사항
- **고성능 (Low Latency, High Throughput)**  
- **확장성 (Scalability)**: 수평 확장 기반의 분산 저장 지원  
- **내결함성 (Fault Tolerance)**: 노드 장애에도 데이터 보존  
- **일관성 vs 가용성 (CAP Trade-off)**: 시스템 특성에 맞는 균형 필요  

---

## 3) 내부 구조
- **저장 구조**:
  - 해시 테이블 기반 인덱스  
  - Log-structured Merge Tree(LSM-Tree) 기반 구조 (예: LevelDB, RocksDB)  
- **데이터 배치**:
  - **Consistent Hashing**으로 파티션 분산  
  - 리플리케이션(Replication factor R)으로 내결함성 확보  

---

## 4) 읽기/쓰기 경로
- **쓰기(Write Path)**:
  - (1) 클라이언트 → Coordinator Node  
  - (2) 해당 Key를 담당하는 파티션 노드로 전달  
  - (3) 메모리(Write-Ahead Log + MemTable) → 디스크(SSTable)  
  - (4) 복제 노드에도 전파  

- **읽기(Read Path)**:
  - (1) 클라이언트 → Coordinator Node  
  - (2) 파티션 노드에서 Key 조회  
  - (3) 캐시/메모리 → 디스크 순서 확인  
  - (4) 복제본 간 정합성 확인 (Quorum, Version Vector, Timestamp 비교)  

---

## 5) 일관성 모델
- **Strong Consistency**: 항상 최신값 반환 (대가: 지연시간 증가)  
- **Eventual Consistency**: 시간이 지나면 일관성 수렴 (대가: 최신성 희생)  
- **Tunable Consistency**: 읽기/쓰기 쿼럼 파라미터(R, W, N) 조절  

---

## 6) 장애 처리
- **Hinted Handoff**: 노드 장애 시 임시 저장 후 복구 시 전달  
- **Anti-Entropy**: Merkle Tree 기반 동기화로 데이터 수렴  
- **Read Repair**: 읽기 시점에 복제본 간 불일치 자동 수정  

---

## 7) 최적화 기법
- **캐싱 계층**: Hot key 대응 (Redis/Memcached)  
- **압축/Compaction**: LSM-Tree의 쓰기 최적화  
- **TTL 지원**: 자동 만료 키로 메모리 관리  
- **Bloom Filter**: 디스크 접근 전 존재 여부 확인  

---

## 8) 대표 시스템
- **Dynamo 계열**: Amazon Dynamo, Cassandra, Riak  
- **Embedded KV**: LevelDB, RocksDB  
- **In-memory KV**: Redis, Memcached  

---

## 9) 활용 사례
- 세션 관리 (로그인 세션, 인증 토큰)  
- 캐싱 (API 응답, DB 조회 결과)  
- 카운터/랭킹 서비스 (좋아요, 조회수)  
- 분산 스토리지의 메타데이터 저장  

---

## 10) 한 줄 요약
> Key-Value Store는 **간단한 구조 + 강력한 확장성**으로 대규모 분산 시스템의 핵심 빌딩 블록이며,  
> Consistent Hashing, Replication, Tunable Consistency 같은 설계가 필수적이다.
