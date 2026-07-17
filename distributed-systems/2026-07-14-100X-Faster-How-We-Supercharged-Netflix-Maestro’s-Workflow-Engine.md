# 100X Faster: How We Supercharged Netflix Maestro’s Workflow Engine

- 읽은 날짜: 2026-07-14
- 회사/작성 주체: Netflix
- 원문: [100X Faster: How We Supercharged Netflix Maestro’s Workflow Engine](https://netflixtechblog.com/100x-faster-how-we-supercharged-netflix-maestros-workflow-engine-028e9637f041)

# Netflix Maestro는 어떻게 워크플로 엔진을 100배 빠르게 만들었을까?

## 1. Maestro는 어떤 시스템인가?

Maestro는 Netflix가 대규모 데이터·머신러닝 작업을 실행하기 위해 만든 **워크플로 오케스트레이터**다.

예를 들어 다음과 같은 작업 흐름이 있다고 하자.

```
데이터 수집
    ↓
데이터 정제
    ↓
통계 계산 ─┐
           ├─→ 결과 배포
모델 학습 ─┘
```

Maestro는 단순히 작업을 실행하는 것이 아니라 다음과 같은 역할을 담당한다.

- 선행 작업이 성공했는지 확인
- 병렬 실행 가능한 작업 판단
- 실패한 작업 재시도
- 워크플로 시작·중지·일시 정지·재시작
- Workflow와 Step 상태 저장
- Spark, Trino, Container 등 외부 실행 환경 연동
- 조건 분기, `foreach`, 하위 Workflow 처리

기존 Maestro도 하나의 Workflow에 수십만 개의 Job을 포함하고, 하루 수백만 건의 실행을 처리할 정도로 확장성이 뛰어났다.

문제는 **처리 가능한 작업량이 아니라 하나의 Step이 끝난 후 다음 Step으로 전환되는 속도**였다. 실제 작업 외에도 Queue 대기, DB 조회, 상태 동기화 등에 수 초가 소요됐다.

---

## 2. 확장성은 좋았지만 지연시간이 길었다

확장성, 처리량, 지연시간은 서로 다른 개념이다.

| 개념 | 의미 |
| --- | --- |
| Scalability | 작업량이 증가했을 때 서버를 추가해 대응할 수 있는가 |
| Throughput | 일정 시간 동안 얼마나 많은 작업을 처리할 수 있는가 |
| Latency | 하나의 요청이 실제 처리되기까지 얼마나 걸리는가 |

기존 Maestro는 서버를 늘리면 더 많은 작업을 동시에 처리할 수 있었다.

하지만 하나의 Task가 실행될 때마다 다음 과정이 반복됐다.

<img width="1576" height="998" alt="image" src="https://github.com/user-attachments/assets/9cfb1e78-bece-4f54-b376-e041cb96d1d0" />

1. Worker가 Distributed Queue에서 Task ID를 Polling한다.
2. DB에서 Workflow 전체 상태를 조회한다.
3. Task 실행 함수를 한 번 호출한다.
4. 변경된 상태를 DB에 저장한다.
5. Task를 다시 Queue에 등록한다.
6. 다음 Polling까지 기다린다.

# **Task 1회 실행마다 반복되는 기존 처리 흐름**

Worker를 추가하면 처리량은 증가하지만, 각 Task가 Queue Polling을 기다리고 DB를 조회하는 시간은 줄어들지 않는다.

따라서 기존 Maestro는 **규모는 잘 감당했지만 Step 전환은 느린 구조**였다.

---

## 3. 기존 Maestro의 3계층 구조

기존 Maestro는 크게 세 계층으로 구성됐다.

| 계층 | 주요 역할 |
| --- | --- |
| Maestro API / Step Runtime | 사용자 요청, 인증, 모니터링, Spark·Trino 등 외부 환경 연동 |
| Maestro Engine | Workflow·Step 상태 머신, DAG를 Flow와 Task로 변환, 상태 저장 |
| Internal Flow Engine | Task 실행, 다음 Task 시작, Queue와 내부 실행 흐름 관리 |

가장 아래에 있는 Internal Flow Engine은 **Conductor 2.x**를 기반으로 구현돼 있었다.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/b038473a-1a5f-47b3-87b4-e7a442e90b95" />

```
Maestro API / Step Runtime
             ↓
       Maestro Engine
             ↓
Conductor 기반 Flow Engine
             ↓
 Conductor DB + Distributed Queue
```

API와 Runtime 계층보다 더 큰 문제는 Maestro Engine 아래에 있던 범용 Flow Engine이었다.

---

## 4. Maestro와 Conductor가 상태를 중복 관리했다

기존 구조에서는 Maestro와 Conductor가 각각 별도의 상태 머신과 데이터베이스를 가지고 있었다.

| Maestro DB | Conductor DB |
| --- | --- |
| Workflow State | Flow State |
| Step State | Task State |

예를 들어 하나의 작업이 완료되면 다음 두 상태가 모두 변경돼야 했다.

- Maestro DB: `Step A = SUCCESS`
- Conductor DB: `Task A = COMPLETED`

하지만 분산 시스템에서는 두 저장소의 변경이 항상 함께 성공하지 않는다.

Conductor DB 업데이트는 성공했지만 네트워크 장애로 Maestro DB 업데이트가 실패할 수 있다. 그러면 다음과 같이 상태가 달라진다.

- Maestro DB: `RUNNING`
- Conductor DB: `COMPLETED`

반대 상황도 발생할 수 있다.

이중 상태 관리 구조는 다음 문제를 만들었다.

- 두 DB의 상태 불일치
- 상태 동기화 Callback 필요
- 장애 복구 로직 복잡성 증가
- Orphaned Job 발생
- 어떤 상태가 실제 상태인지 판단하기 어려움

Netflix는 Conductor가 제공하는 기능 중 일부만 사용했지만, 이를 위해 범용 Workflow Engine 전체를 운영하고 있었다.

---

## 5. Stateless Worker와 Polling의 한계

기존 Flow Engine은 Stateless Worker 구조였다.

Worker는 Task를 처리한 뒤 현재 Workflow 상태를 자신의 메모리에 계속 보관하지 않는다. 다음 실행에서는 다른 Worker가 같은 Workflow를 처리할 수 있다.

예를 들면 다음과 같다.

- 첫 번째 처리: Worker 1
- 두 번째 처리: Worker 5
- 세 번째 처리: Worker 2

이 구조는 Worker 사이에 상태를 공유하지 않으므로 수평 확장하기 쉽다.

하지만 Task를 처리할 때마다 다음 작업이 반복된다.

- Distributed Queue 접근
- DB에서 Workflow 전체 상태 조회
- 상태 객체 재구성
- Task 상태 저장
- Queue 재등록

또한 Polling 방식에서는 Task가 완료되더라도 다음 Task를 즉시 실행하지 못한다.

```
Task A 실제 완료
       ↓
Polling Interval 대기
       ↓
Worker가 완료 상태 확인
       ↓
Task B 실행
```

Workflow가 여러 Step으로 구성되면 각 Step 사이의 Polling 시간이 계속 누적된다.

복잡한 Workflow에서는 실제 작업 시간이 아니라 Maestro 내부 처리만으로 약 10초의 오버헤드가 발생할 수 있었다.

---

## 6. At-Least-Once Queue와 중복 실행 문제

기존 Distributed Queue는 **At-Least-Once 전달 방식**을 사용했다.

이는 메시지가 사라지지 않도록 최소 한 번 이상 전달하지만, 동일한 메시지가 여러 Worker에 전달될 수 있다는 뜻이다.

예를 들어 Worker 1과 Worker 2가 동시에 Task A를 가져갔다고 하자.

1. 두 Worker가 모두 Task A의 상태를 조회한다.
2. Worker 2가 긴 JVM GC Pause에 빠진다.
3. Worker 1이 Task A를 완료하고 `COMPLETED` 상태를 저장한다.
4. Worker 1이 다음 Task B를 시작한다.
5. Worker 2가 GC에서 깨어난다.
6. Worker 2가 과거에 읽은 상태를 기준으로 `RUNNING`을 다시 저장한다.

그러면 상태가 다음과 같이 과거로 돌아갈 수 있다.

```
RUNNING → COMPLETED → RUNNING
                       ↑
                 잘못된 상태 복원
```

이 경우 Task B가 실행 중인데 선행 Task A가 다시 `RUNNING`으로 표시되는 모순이 발생한다.

Netflix는 새로운 엔진에서 다음 조건을 보장해야 했다.

- 하나의 Step은 같은 시점에 한 Worker만 실행한다.
- Step 상태는 과거 상태로 돌아가지 않는다.
- 모든 Step은 결국 `SUCCESS`, `FAILED` 같은 종료 상태에 도달한다.
- 내부 Flow 상태와 Maestro DB 상태는 최종적으로 일치해야 한다.
- 사용자의 시작·중지·일시 정지·재시작 요청이 실행 중인 Workflow와 충돌하지 않아야 한다.

---

## 7. Netflix가 검토한 세 가지 선택지

Netflix는 다음 세 가지 방법을 검토했다.

| 선택지 | 내용 |
| --- | --- |
| Option 1 | Maestro 전용 Flow Engine 직접 구현 |
| Option 2 | Conductor 2.x를 Conductor 4로 업그레이드 |
| Option 3 | Temporal 도입 |

### Conductor 4 업그레이드

Conductor 4는 이전 버전보다 개선됐지만 다음 구조는 그대로 유지된다.

- Maestro 상태 머신
- Conductor 상태 머신
- Maestro DB
- Conductor DB
- 두 시스템 사이의 상태 동기화

즉, 일부 성능은 개선될 수 있어도 이중 상태 관리라는 구조적인 문제는 해결되지 않는다.

### Temporal 도입

Temporal은 장기 실행 Workflow, 재시도, Timer, Signal, 장애 복구에 강한 시스템이다.

하지만 Maestro 내부의 가벼운 상태 전환마다 외부 Temporal Service를 호출하면 네트워크 통신과 새로운 장애 지점이 추가된다.

Temporal의 기능이 부족해서가 아니라, Maestro 내부 Flow 전환에 사용하기에는 너무 강력하고 복잡한 시스템이었다.

### 최종 선택

Netflix는 Maestro가 실제로 필요로 하는 기능만 포함한 **경량 Flow Engine을 직접 구현**하기로 했다.

---

## 8. 새로운 구조: DB는 진실이고 메모리는 실행 상태다

새로운 Flow Engine은 현재 실행 상태를 JVM 메모리에 유지한다.

그렇다고 DB를 제거한 것은 아니다. DB와 메모리의 역할을 명확하게 나눴다.

| Maestro DB | JVM Memory |
| --- | --- |
| 영구 상태 저장 | 현재 실행 상태 저장 |
| 장애 복구 기준 | 빠른 상태 접근 |
| Source of Truth | Actor의 즉각적인 상태 전환 |
| Workflow·Step 상태 | Flow·Task 활성 상태 |

전체 구조는 다음과 같다.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/539253f9-4ec6-410c-8588-7c349e1550ad" />


평상시에는 메모리 상태를 이용해 빠르게 처리한다.

- Event 도착
- Actor Queue에 등록
- 메모리 상태 확인
- 즉시 상태 전환

Node가 종료돼 메모리 상태가 사라지면 Maestro DB를 기준으로 복구한다.

1. 다른 Node가 Flow Group 소유권을 획득한다.
2. Maestro DB에서 Workflow와 Step 상태를 조회한다.
3. Flow와 Task의 메모리 상태를 재구성한다.
4. 실행을 재개한다.

즉, 메모리는 빠른 실행을 위한 공간일 뿐 Source of Truth가 아니다.

- DB: 복구 가능한 영구 상태
- Memory: 빠른 실행을 위한 재구성 가능한 상태

이를 통해 성능과 장애 복구를 동시에 확보했다.

---

## 9. Write-Through Cache와 유사한 구조

Workflow나 Step 상태가 변경되면 영구 상태와 메모리 상태를 함께 갱신한다.

1. 상태 전환이 발생한다.
2. Maestro DB에 상태를 반영한다.
3. JVM 메모리의 실행 상태도 변경한다.

메모리 상태는 사라져도 DB를 기준으로 다시 만들 수 있다.

기존 구조에서는 Maestro DB와 Conductor DB를 동기화해야 했다.

변경 후에는 영구 상태를 저장하는 DB가 Maestro DB 하나로 줄어들었다. JVM 메모리는 Maestro DB에서 언제든 재구성할 수 있다.

이 구조를 통해 상태 동기화 문제와 반복적인 DB 조회를 크게 줄일 수 있었다.

---

## 10. Locality: Flow와 Task를 같은 JVM에 배치한다

새 엔진의 핵심 성능 개선 요소는 **Locality**다.

하나의 Flow와 그 Flow에 포함된 Task Actor들을 같은 Maestro Node의 JVM에 배치한다.

기존에는 Task A1이 완료되면 다음 Task A2를 실행하기 위해 외부 Queue와 DB를 거쳐야 했다.

변경 후에는 같은 JVM 안에서 Actor끼리 메시지를 전달한다.

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/d524b8a4-6789-4375-b5a1-f0e60441b8a0" />

이 구조를 통해 다음 비용이 줄어들었다.

- Network I/O
- Queue Polling
- Workflow 전체 상태 재조회
- 상태 객체 재구성
- Worker 간 이동

Task가 완료되면 같은 JVM에서 다음 Task를 즉시 시작할 수 있다.

---

## 11. Locality가 만드는 새로운 문제

Flow를 하나의 Node에 유지하면 성능은 좋아지지만 새로운 문제가 생긴다.

기존 Stateless 구조에서는 어떤 Worker든 Queue에서 Task를 가져갈 수 있었다.

반면 새로운 구조에서는 특정 Flow가 특정 Node에 붙어 있다.

- Flow A → Node 1
- Flow B → Node 2
- Flow C → Node 3

이를 Stickiness라고 볼 수 있다.

### 장점

- 상태를 반복해서 DB에서 읽지 않아도 된다.
- 같은 JVM에서 다음 Task를 즉시 실행할 수 있다.
- Actor 사이에 로컬 메시지 통신이 가능하다.

### 단점

- 특정 Node에 Flow가 몰릴 수 있다.
- Node 장애 시 메모리 상태를 복구해야 한다.
- Flow 소유권을 관리해야 한다.
- 한가한 Worker가 다른 Node의 Flow를 자유롭게 가져오기 어렵다.

Netflix는 이 문제를 **Flow Group**으로 해결했다.

---

## 12. Flow Group으로 수평 확장 유지

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/f1f8c321-b15c-47ea-88e2-12929cbb5ea6" />

실행 중인 Flow가 수백만 개일 때 각각의 Flow에 대해 소유권과 Heartbeat를 관리하면 비용이 매우 크다.

Netflix는 여러 Flow를 하나의 Flow Group으로 묶었다.

예를 들면 다음과 같다.

- Flow Group 0: Flow A, Flow B, Flow C
- Flow Group 1: Flow D, Flow E
- Flow Group 2: Flow F, Flow G

Node는 개별 Flow가 아니라 Group 단위로 소유한다.

- Node 1 → Group 0, 1, 2
- Node 2 → Group 3, 4, 5
- Node 3 → Group 6, 7, 8

각 Group에는 Group Actor가 있고, 그 아래에 여러 Flow Actor와 Task Actor가 존재한다.

Flow Group을 사용하면 개별 Flow 수백만 개의 Heartbeat와 소유권을 관리하는 대신, 상대적으로 적은 수의 Group만 관리하면 된다.

---

## 13. Node 장애와 Group 소유권 이동

Node 1이 Group 0을 소유하고 있다고 하자.

Node 1이 종료되거나 긴 GC Pause에 빠지면 Group 0의 Heartbeat가 더 이상 갱신되지 않는다.

다른 Node는 Heartbeat가 만료된 것을 확인하고 Group 0의 소유권을 가져간다.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/c09aefa9-6371-43b7-8362-df9fb34fff67" />

```
Node 1이 Group 0 소유
        ↓
Node 1 장애 또는 긴 GC Pause
        ↓
Heartbeat 만료
        ↓
Node 2가 Group 0 Claim
        ↓
Generation ID 증가
        ↓
Maestro DB에서 상태 조회
        ↓
Flow·Task 메모리 상태 재구성
        ↓
Workflow 실행 재개
```

메모리에 실행 상태를 두더라도 DB에 영구 상태가 남아 있기 때문에 다른 Node에서 실행을 이어갈 수 있다.

---

## 14. Generation ID가 필요한 이유

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/8f9f3ee5-72e2-4e57-85c5-e324f2e993ba" />

Node가 완전히 종료되지 않고 긴 GC Pause에 들어갔다가 다시 실행될 수 있다.

다음과 같은 상황이 가능하다.

1. Node 1이 Group 0을 처리하다가 GC Pause에 빠진다.
2. Heartbeat가 만료된다.
3. Node 2가 Group 0의 새로운 소유자가 된다.
4. Node 1의 GC가 끝나고 다시 작업을 실행하려고 한다.

이때 Node 1과 Node 2가 모두 자신이 Group 0의 Owner라고 생각할 수 있다.

Netflix는 이를 막기 위해 Group 소유권이 변경될 때마다 **Generation ID**를 증가시킨다.

예를 들어 다음과 같다.

- 기존 Group 0 Generation: 7
- Node 1 Generation: 7
- Node 2가 Group 0 Claim
- 새로운 Group 0 Generation: 8
- Node 2 Generation: 8

이후 Node 1이 작업을 실행하려 하면 다음 값이 비교된다.

- Node 1이 가진 Generation: 7
- DB의 현재 Generation: 8

두 값이 다르므로 Node 1의 작업은 거부된다.

Generation ID는 오래된 Owner의 늦은 실행을 차단하는 **Fencing Token**과 같은 역할을 한다.

---

## 15. Flow Partitioning

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/9a77497b-0ade-4d90-b4d7-ac95a33bc40b" />

각 Flow는 안정적인 Partition 함수를 통해 자신이 속할 Group을 결정한다.

개념적으로는 다음과 같다.

`groupId = stableHash(workflowId) % maxGroupCount`

예를 들어 최대 Group 수가 9개라면 Workflow ID에 따라 다음처럼 배치된다.

- Workflow A → Group 1
- Workflow B → Group 7
- Workflow C → Group 4

Flow Operator는 계산된 Group ID를 소유한 Node로 요청을 전달한다.

Netflix는 실행 중인 Stateful Flow를 자주 이동시키는 대신, Group ID를 안정적으로 유지하는 방식을 선택했다.

Stateful Flow를 이동하려면 다음 문제를 함께 해결해야 하기 때문이다.

- 현재 실행 상태 이전
- Group 소유권 변경
- 중복 실행 차단
- 진행 중인 Task 처리
- Parent·Child 관계 유지

따라서 복잡한 동적 재배치보다 예측 가능하고 안정적인 Partition 방식을 선택했다.

---

## 16. Parent Flow와 Child Flow

Maestro는 `foreach`나 Subworkflow를 실행할 때 Child Flow를 생성한다.

- Parent Flow
    - Child Flow 1
    - Child Flow 2
    - Child Flow 3

Parent와 Child는 서로 다른 Group에 배치될 수 있다.

예를 들어 다음과 같다.

- Parent Flow → Group 2
- Child Flow 1 → Group 5
- Child Flow 2 → Group 7

Parent가 Child의 Group을 계산할 수 있도록 Parent가 사용한 최대 Group 수 `N'`을 Child에게 전달한다.

Child Group은 개념적으로 다음과 같이 계산된다.

`childGroupId = partition(childWorkflowId, N')`

Parent는 Child Workflow ID와 `N'`만 알고 있으면 Child가 속한 Group을 계산할 수 있다.

별도의 중앙 Mapping Table을 매번 조회하지 않아도 된다.

---

## 17. Group 수가 증가해도 기존 Workflow는 이동하지 않는다

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/d434e4cd-1af9-476d-9fcb-8770ea7c5deb" />

처음에 Group 수가 9개였다고 하자.

이후 서버 확장을 위해 Group 수를 12개로 늘릴 수 있다.

하지만 기존 Workflow의 Group을 새로운 Group 수로 다시 계산하면 배치 위치가 바뀔 수 있다.

예를 들어 Workflow Hash가 121이라면 다음과 같다.

- `121 % 9 = Group 4`
- `121 % 12 = Group 1`

실행 중인 Workflow가 갑자기 다른 Group으로 이동하면 상태 이전과 중복 실행 문제가 발생한다.

Netflix는 기존 Workflow가 생성될 당시 사용한 최대 Group 수 `N'`을 계속 유지하도록 했다.

- 기존 Workflow: 기존 `N'` 사용 → 기존 Group 유지
- 새로운 Workflow: 새로운 Group 수 사용 → 새 Group 활용

이를 통해 기존 Workflow를 이동시키지 않으면서 Group 수를 점진적으로 늘릴 수 있다.

---

# 핵심 내용 정리

Netflix Maestro의 문제는 많은 작업을 처리하지 못하는 것이 아니었다.

많은 작업을 동시에 처리할 수는 있었지만, 하나의 Task가 완료된 후 다음 Task로 넘어가는 과정에서 Queue Polling, DB 조회, 상태 동기화가 반복돼 지연시간이 길었다.

## 기존 구조의 문제

1. Polling으로 인한 Step 사이의 대기 시간
2. Task 처리 때마다 Workflow 전체 상태 조회
3. Maestro DB와 Conductor DB의 이중 상태 관리
4. At-Least-Once Queue로 인한 중복 실행
5. 긴 GC 이후 오래된 Worker가 상태를 되돌리는 문제
6. Maestro에 필요하지 않은 범용 DAG Engine의 복잡성

## 새로운 구조의 핵심

| 구성 요소 | 역할 |
| --- | --- |
| Maestro DB | 영구적인 Source of Truth |
| JVM Memory | 빠른 Flow·Task 실행 상태 |
| Actor | 하나의 Entity에 대한 Event를 순차 처리 |
| Locality | 같은 Flow와 Task를 같은 JVM에 배치 |
| Flow Group | 여러 Flow를 묶어 Node에 분산 |
| Generation ID | 이전 Owner의 늦은 실행 차단 |

기존 실행 흐름은 Queue, Worker, DB 조회, 상태 저장, Queue 재등록을 반복했다.

새로운 구조에서는 Event가 Actor Queue로 전달되고, 메모리에 저장된 상태를 기준으로 즉시 다음 상태로 전환된다.

Netflix는 이를 통해 Step 실행 오버헤드를 약 **5초에서 50밀리초로 줄여 약 100배 개선**했다.

또한 다음과 같은 효과를 얻었다.

- DB Query Traffic 약 90% 감소
- 기존 엔진 관련 테이블 약 40TB 제거
- 상태 동기화 복잡성 감소
- Race Condition 감소
- Workflow 실행 속도 향상
- 낮은 지연시간이 필요한 서비스 지원

핵심은 단순히 상태를 메모리에 올린 것이 아니다.

**DB를 Source of Truth로 유지하면서 메모리에 실행 상태를 두고, Actor로 상태 변경을 직렬화했으며, Flow Group과 Generation ID로 분산 환경의 소유권과 장애 복구 문제를 해결한 것**이다.

---

# 느낀 점

확장성이 좋은 시스템이라도 개별 작업의 지연시간이 크면 사용자 경험과 개발 생산성이 떨어질 수 있다는 점이 인상적이었다.

특히 서버를 단순히 추가하는 대신, 상태 저장 구조와 실행 방식을 변경해 성능을 개선했다는 점이 중요하다.

DB는 장애 복구를 위한 진실로 유지하고, 메모리는 빠른 실행에 사용하면서 각각의 역할을 명확히 분리했다. 여기에 Actor, Flow Group, Generation ID를 결합해 성능뿐 아니라 중복 실행과 장애 복구 문제도 함께 해결했다.

결국 기능이 많은 범용 시스템을 그대로 사용하는 것보다, 실제 요구사항에 맞게 구조를 단순화하는 것이 더 큰 성능 향상과 운영 효율로 이어질 수 있다는 점을 배울 수 있었다.
