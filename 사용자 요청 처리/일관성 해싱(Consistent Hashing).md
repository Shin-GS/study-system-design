# 일관성 해싱(Consistent Hashing)
## 1) 왜 필요한가?
- 전통적 분산(샤딩) 방식: `hash(key) % N`  
  - **문제**: 노드 수 `N`이 변하면 **대부분의 키가 재배치** → 캐시 미스 폭증, 리밸런싱 비용↑
- **Consistent Hashing**은 노드 증감/장애 시 **이동하는 키 수를 최소화**(대략 전체의 `~1/N`)하여 안정적인 분산을 제공.

---

## 2) 기본 모델: 해시 링(Hash Ring)
- 해시 공간(예: `0 ~ 2^m-1`)을 **원형 링**으로 보고, `hash(node)`와 `hash(key)`를 동일한 공간에 매핑.
- 각 키는 링에서 **시계방향으로 가장 가까운 노드(successor)** 에 할당.
- **노드 추가**: 새 노드 위치 이후 구간의 일부 키만 이동.  
- **노드 제거/장애**: 다음 노드가 자동 승계 → 가용성 유지.

```
     0 ─────────▶ (해시 링) ─────────▶ 2^m-1
          ↑keyA       ↑keyB          ↑keyC
        [N1]──────[N2]──────[N3]──────[N4]
            (시계방향 가장 가까운 노드가 담당)
```

---

## 3) 가상 노드(Virtual Nodes, vnodes)
- 한 물리 노드가 **여러 포인트**(vnode)로 링에 매핑되게 함: `hash(node#i)`
- 효과
  - **분포 균등화**(Hot partition 완화)
  - 노드 성능에 따른 **가중치 분배**(성능 좋은 노드는 더 많은 vnode)
- 운영 팁
  - vnode 개수는 데이터량·노드 수·부하 편차를 보며 실측으로 조정
  - 장애 시 **vnode 단위 재배치**로 영향 최소화

---

## 4) 복제(Replication)와 읽기/쓰기
- 내결함성을 위해 키를 **연속된 R개 노드**에 복제(Primary + Replicas).
- 읽기/쓰기는 **쿼럼(Quorum) 규칙**으로 정합성 제어: `R(read) + W(write) > N(replication factor)`
- 재동기화:
  - **Hinted handoff**, **anti-entropy(머클 트리 기반 동기화)** 등으로 지연 복제 수렴

---

## 5) 리밸런싱(Rebalancing) 계산
- **이동 키 비율**: 노드 1대 추가 시 대략 `~1/(기존 노드 수 + 1)`
- **이동 데이터량(바이트)** = 이동 키 수 × 평균 객체 크기
- **시간** ≈ 이동 데이터량 ÷ (유효 네트워크 처리량)  
  - 야간/저부하 시간대에 백그라운드 스트리밍 + 스로틀링 권장

---

## 6) 구현 세부
- **해시 함수**: 균일 분포 + 빠른 계산 (예: Murmur, xxHash 등)
- **탐색**: 링 상에서 첫 **successor** 찾기(정렬 맵/바이너리 서치)
- **자료구조**
  - 정렬 맵(TreeMap)/스킵 리스트: 삽입·탐색 용이
  - 배열 기반 링 + 이진 검색: 메모리 효율↑
- **키 설계**
  - 복제 키: `hash(key + replicaIndex)`
  - vnode 키: `hash(nodeId + i)`

---

## 7) 대안 알고리즘
- **Rendezvous/HRW Hashing**: 각 노드에 점수 계산 → 최고 점수 노드 선택(간단·균등).
- **Jump Consistent Hash**: 링 구조 없이 O(1) 시간에 파티션 선택, 재배치 최소화.
- **Maglev/Consistent Hash for LBs**: L4/L7 로드밸런서에서 연결 고정(stickiness) 제공.

> 선택 기준: 구현 복잡도, 동적 변화 빈도, 균등성 요구, 캐시 미스/콜드스타트 비용.

---

## 8) 운영 이슈 & 모니터링
- **키 분포 균등성**: 샤드별 키/바이트 편차, p95/p99 처리지연
- **리밸런싱 이벤트**: 이동량, 소요 시간, 실패율
- **장애 시나리오**: 노드 플랩핑(불안정), 네트워크 분리 시 읽기/쓰기 정책
- **콜드 캐시 비용**: 노드 추가 직후 miss율 급증 → **워밍**/프리로드/TTL 분산

---

## 9) 예시 의사코드
```python
# 간단한 Consistent Hashing with vnodes (파이썬 유사)
class ConsistentHash:
    def __init__(self, vnode=100, hash_fn=hash):
        self.vnode = vnode
        self.hash = hash_fn
        self.ring = []           # (hash, nodeId) 정렬 리스트
        self.nodes = set()

    def _h(self, s): return self.hash(str(s))

    def add_node(self, node_id):
        if node_id in self.nodes: return
        self.nodes.add(node_id)
        for i in range(self.vnode):
            self.ring.append((self._h(f"{node_id}-{i}"), node_id))
        self.ring.sort(key=lambda x: x[0])

    def remove_node(self, node_id):
        self.ring = [p for p in self.ring if p[1] != node_id]
        self.nodes.discard(node_id)

    def get(self, key):
        if not self.ring: return None
        h = self._h(key)
        # 첫 successor 찾기
        lo, hi = 0, len(self.ring) - 1
        ans = None
        while lo <= hi:
            mid = (lo + hi) // 2
            if self.ring[mid][0] >= h:
                ans = mid
                hi = mid - 1
            else:
                lo = mid + 1
        idx = ans if ans is not None else 0
        return self.ring[idx][1]
```

---

## 10) 실무 체크리스트
- [ ] vnode 수/가중치로 **분포 균등화** 확보  
- [ ] **복제 R**·쿼럼 규칙으로 내결함성·정합성 목표 달성  
- [ ] 리밸런싱 스로틀링/워밍/TTL 분산으로 **콜드스타트 완화**  
- [ ] 모니터링: 샤드별 키/바이트 편차, 에러율, 지연분포  
- [ ] 장애훈련: 노드 장애/복구, 네트워크 분리, 급격한 스케일 변화

---

## 11) 한 줄 요약
> Consistent Hashing은 **노드 변화에도 최소 재배치**로 균등 분산을 유지하는 핵심 기법이며,  
> vnodes/복제/쿼럼/리밸런싱 전략을 함께 설계해야 운영 비용을 최소화할 수 있다.
