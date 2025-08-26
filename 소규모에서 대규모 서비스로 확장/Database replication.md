# Database Replication 정리

## 1) 개념
- **Replication**: 하나의 데이터베이스 노드에서 **데이터 변경(로그)**을 다른 노드로 **복제**하여 동일/유사한 상태를 유지하는 기술.
- 목적: **가용성(HA), 읽기 확장성(Read Scale-out), 재해 복구(DR), 무중단 운영**.

---

## 2) 토폴로지(Topology)
- **Single-Leader (Primary–Replica, Master–Slave)**  
  - 하나의 리더(쓰기 허용)와 여러 개의 팔로워(읽기 중심). 가장 흔한 구조.
- **Multi-Leader (Active–Active)**  
  - 여러 리더가 쓰기를 처리, 지리적 분산에 유리하나 **충돌 해결**이 필요.
- **Leaderless (Dynamo 스타일)**  
  - 중앙 리더 없이 다수 노드가 읽기/쓰기를 처리. **Eventual Consistency** 기반, 높은 내결함성.

---

## 3) 동기 방식
- **Synchronous**  
  - 리더가 **모든(또는 과반수)** 팔로워에 기록 성공을 확인한 뒤 클라이언트에 응답. **강한 일관성** 확보, **지연(latency)** 증가.
- **Asynchronous**  
  - 리더가 자신의 커밋만 확인하고 응답, 팔로워는 **지연 복제**. 쓰기 성능↑, **데이터 손실 위험**(리더 장애 시 마지막 커밋 미전파).
- **Semi-synchronous / Quorum**  
  - 최소 1개 또는 **과반수(Ack quorum)** 확인 후 응답. 성능과 일관성의 절충.

---

## 4) 복제 단위 & 방법
- **Statement-based**: `INSERT/UPDATE/DELETE` 문장을 복제. 결정적이지 않으면 문제(비결정 함수, 트리거, NOW() 등).  
- **Row-based (Logical)**: 변경된 **행** 자체를 복제(MySQL RBR, Postgres logical). 재현성↑, 로그 용량↑.  
- **WAL/Physical shipping**: **저수준 로그/페이지**를 스트리밍(Postgres WAL, Oracle redo). 성능/정합 우수, 이종 시스템 적용 어려움.  
- **Snapshot + Change Stream**: 초기 **스냅샷** 이후 **증분(체인지 로그)** 적용(예: Debezium/CDC).

---

## 5) 일관성 모델(Reads)
- **Strong**: 리더 쓰기 직후 읽어도 최신 데이터 보장(보통 리더에서 읽기).
- **Read-Your-Writes**: 자신이 쓴 내용은 이후 읽기에서 보장(세션/사용자 단위 라우팅 필요).
- **Monotonic Reads**: 시간이 지날수록 더 오래된 값을 보지 않음(레플리카 라우팅 주의).
- **Eventual**: 언젠가는 일관; 지연 복제 환경에서 흔함.

---

## 6) 장점과 한계
- **장점**: 읽기 확장, 장애 시 빠른 전환(페일오버), 지역 근접성 향상(지연 감소), 온라인 유지보수.  
- **한계/리스크**: 복제 지연(lag), 쓰기 충돌(Multi-leader), **Split-brain**, 네트워크 파티션, 운영 복잡도↑.

---

## 7) 운영 이슈 & 메트릭
- **Replication Lag**: 초(`seconds_behind_master`, `replication_lag_bytes` 등), **WAL 바이트 지연**, 적용 큐 길이.  
- **Apply Latency**: 팔로워가 로그를 적용하는 시간.  
- **Conflict Rate**(Multi-leader), **Divergence**(Leaderless).  
- **Failover 시간**: 장애 감지 + 승격(promote) 소요.  
- **Catch-up 속도**: 재합류 노드가 최신 상태로 도달하는 시간/대역폭.

> **중요**: 레플리카는 **백업의 대체가 아님**. 정기 **스냅샷/증분 백업, PITR**(Point-in-time recovery) 별도 운영.

---

## 8) 충돌 해결(주로 Multi‑Leader)
- **Last Write Wins(LWW)**: 타임스탬프 최신값 우선(간단하지만 데이터 손실 가능).
- **버전 벡터/Vector Clock**: 인과관계 추적 후 머지 로직 적용(복잡하지만 정확성↑).
- **도메인 규칙 기반 머지**: 예) 카운터는 합산, 상태머신은 우선순위/승격 규칙.

---

## 9) 페일오버 & 승격
- **자동 페일오버**: 헬스체크 + 리더 선출(과반수/라프트/ZooKeeper/etcd).  
- **수동 승격**: 운영자가 특정 레플리카를 **Primary**로 승격.  
- **주의**: 두 리더가 동시 활성화되는 **Split-brain 방지**(Stonith, fencing, quorum).

---

## 10) 읽기 스케일‑아웃 패턴
- **Read Replica 라우팅**: 읽기는 레플리카, 쓰기는 리더.  
- **Session Affinity**: `Read-your-writes` 보장을 위해 **사용자/세션**은 리더 또는 최신 레플리카로 라우팅.  
- **지리 분산**: 지역 레플리카로 지연 최소화, 쓰기는 지역 리더 또는 멀티리더.

```text
예) API 게이트웨이/ORM 라우팅
- WRITE: primary.example.com
- READ : replica-{region}.example.com (lag < 200ms 일 때만 사용)
```

---

## 11) 기술별 힌트(개념 위주)
- **MySQL**: Binlog 기반(Statement/Row/ Mixed), GTID, Semi-sync, Group Replication(InnoDB Cluster).  
- **PostgreSQL**: WAL 스트리밍(물리), Logical Replication, `synchronous_standby_names`, `replication slots`.  
- **MongoDB**: Replica Set, Write Concern(`w`, `majority`), Read Preference(Primary/Secondary).  
- **Cassandra**: 가십/토큰 링, RF(Replication Factor), Consistency Level(ONE/QUORUM/ALL), Hinted Handoff, Repair.

---

## 12) 운영 체크리스트
- [ ] 리더/레플리카 헬스 체크(애플리케이션 레벨 `/healthz` 포함)  
- [ ] **Lag SLO** 정의(예: p99 lag < 2s), 초과 시 읽기 라우팅 차단  
- [ ] 백업 + 복구 리허설(PITR 포함), 암호화/키관리  
- [ ] 네트워크/IO 대역폭 모니터링(복제 채널)  
- [ ] 장애 훈련(리더 장애, 분할 네트워크, 재합류) 정기 수행  
- [ ] 스키마 변경 시 순서/전개 전략(온라인 마이그레이션, backwards‑compatible)  

---

## 13) 한 줄 요약
> 복제는 **가용성과 읽기 확장**을 제공하지만, **일관성·충돌·운영 복잡도**를 관리할 수 있어야 진짜 가치가 난다.
