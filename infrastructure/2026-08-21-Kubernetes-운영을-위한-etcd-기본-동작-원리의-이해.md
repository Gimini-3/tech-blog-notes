# Kubernetes 운영을 위한 etcd 기본 동작 원리의 이해

- 읽은 날짜: 2026-08-21
- 회사/작성 주체: Kakao
- 원문: [Kubernetes 운영을 위한 etcd 기본 동작 원리의 이해](https://tech.kakao.com/posts/484)

---

Kubernetes는 클러스터의 상태를 etcd에 저장한다.
어떤 노드가 존재하는지, 어떤 Pod가 어디에서 동작하는지 같은 정보가 모두 etcd에 들어간다.

```text
Kubernetes Control Plane
          │
          ▼
       ┌──────┐
       │ etcd │
       └──────┘
          │
          ├─ Node 정보
          ├─ Pod 정보
          ├─ Deployment
          └─ Cluster 상태
```

따라서 etcd가 정상적으로 동작하지 않으면 Kubernetes 클러스터 역시 정상적으로 동작하기 어렵다.

## 1. etcd란?

etcd는 Key-Value 형태로 데이터를 저장하는 분산 스토리지다.
하지만 단순히 데이터를 여러 서버에 복사하는 것만으로는 충분하지 않다.

```text
Client
  │
  ▼
┌────────┐
│ etcd 1 │
└────────┘

┌────────┐
│ etcd 2 │
└────────┘

┌────────┐
│ etcd 3 │
└────────┘
```

각 서버가 서로 다른 데이터를 가지고 있다면 어떤 값이 올바른지 판단할 수 없기 때문이다.
그래서 etcd는 RSM(Replicated State Machine) 구조와 Raft Consensus Algorithm을 이용한다.

## 2. RSM과 Consensus

RSM은 같은 명령을 여러 서버에 동일한 순서로 적용해 같은 상태를 유지하는 방식이다.

```text
Command

SET x = 1
    │
    ▼
──────── Log ────────
    │       │       │
    ▼       ▼       ▼
 Server1  Server2  Server3

   x=1      x=1      x=1
```

데이터 변경은 바로 DB에 쓰는 것이 아니라 먼저 Log 형태로 처리된다.
중요한 것은 모든 서버가

```text
Log 1
Log 2
Log 3
```

같은 순서로 Log를 처리해야 한다는 점이다.
이렇게 분산된 서버들이 하나의 상태에 합의하는 것을 Consensus라고 한다.
etcd는 이 Consensus를 구현하기 위해 Raft를 사용한다.

## 3. Raft의 기본 구성

etcd 서버는 세 가지 상태 중 하나를 가진다.

```text
Leader

Follower

Candidate
```

일반적으로 하나의 Leader와 여러 Follower가 존재한다.

```text
             Leader
               │
        ┌──────┴──────┐
        ▼             ▼
    Follower       Follower
```

### Leader

Client의 요청을 받고 Log 복제를 관리한다.

### Follower

Leader의 Log를 전달받아 복제한다.

### Candidate

Leader가 사라졌을 때 새로운 Leader가 되기 위해 선거에 참여하는 상태다.

## 4. Quorum

분산 시스템에서는 모든 서버의 응답을 기다릴 필요가 없다.
etcd는 Quorum, 즉 과반수의 서버가 동의하면 작업을 진행한다.
3개의 etcd 서버가 있다면

```text
전체 서버 = 3

Quorum = 3 / 2 + 1
       = 2
```

따라서 2개의 서버가 정상적으로 데이터를 복제했다면 작업을 완료할 수 있다.

```text
           Leader
             │
       Write Request
             │
       ┌─────┴─────┐
       ▼           ▼
  Follower 1   Follower 2
      OK           X

Leader + Follower 1
       = 2

      Quorum 충족
          ↓
        Commit
```

즉, 서버 한 대에 장애가 발생해도 클러스터는 계속 동작할 수 있다.

## 5. Leader는 어떻게 결정될까?

etcd 클러스터를 처음 실행하면 모든 서버는 Follower다.

```text
Follower A
Follower B
Follower C
```

아직 Leader가 없기 때문에 Heartbeat도 없다.
Follower가 일정 시간 동안 Heartbeat를 받지 못하면 Election Timeout이 발생한다.

```text
Heartbeat 없음
      │
      ▼
Election Timeout
      │
      ▼
Follower → Candidate
```

Candidate는 자신의 Term을 증가시키고 다른 서버에 `RequestVote RPC`를 보낸다.

```text
A : Candidate
Term = 1

      RequestVote
       /        \
      ▼          ▼
     B            C
    OK           OK
```

Quorum 이상의 Vote를 받으면 Leader가 된다.

```text
Candidate
    │
 Quorum 확보
    │
    ▼
  Leader
```

Leader가 된 이후에는 주기적으로 Heartbeat를 보내 자신의 존재를 알린다.

## 6. 데이터는 어떻게 복제될까?

Client가 데이터를 변경하면 요청은 Leader가 받는다.
예를 들어

```text
SET x = 1
```

이라는 요청이 들어왔다고 가정해 보자.
Leader는 먼저 자신의 Log에 기록한다.

```text
Leader

Log
┌───────────┐
│ SET x = 1 │
└───────────┘
```

그리고 `AppendEntry RPC`를 이용해 Follower에게 전달한다.

```text
                Leader
                   │
             SET x = 1
                   │
           ┌───────┴───────┐
           ▼               ▼
       Follower 1       Follower 2
       SET x = 1
```

모든 서버의 응답을 기다리는 것이 아니라 Quorum만큼 복제가 완료되면 Commit한다.

```text
Leader      O
Follower 1  O
Follower 2  X

   2 / 3 성공

       ↓

     Commit
```

Commit 이후 실제 데이터에 반영되고 Client가 읽을 수 있는 값이 된다.

## 7. Leader가 죽으면?

현재 상태가 다음과 같다고 가정해 보자.

```text
        Leader
          X
         DOWN

Follower 1
Follower 2
```

Follower들은 일정 시간 동안 Heartbeat가 오지 않으면 Leader가 죽었다고 판단한다.

```text
Heartbeat 없음
      │
      ▼
Election Timeout
      │
      ▼
Candidate 등장
      │
      ▼
Leader Election
```

하지만 아무 서버나 Leader가 될 수 있는 것은 아니다.
Raft는 더 최신 Log를 가진 서버가 Leader가 되도록 한다.
예를 들어

```text
F1 LastIndex = 10
F2 LastIndex = 9
```

라면 F2가 Candidate가 되어도 F1은 Vote를 거절할 수 있다.
결국 최신 Log를 가진 F1이 Leader가 된다.

```text
Old Leader   DOWN

F1 ──▶ New Leader
F2 ──▶ Follower
```

기존 Leader가 다시 살아나더라도 새로운 Leader보다 Term과 Log가 오래됐기 때문에 Follower로 복귀한다.

## 8. etcd 서버를 추가하면?

운영 중에도 etcd 서버를 추가하거나 제거할 수 있다.
이를 Runtime Reconfiguration이라고 한다.

```text
기존

Leader
Follower
Follower

      ↓ 서버 추가

Leader
Follower
Follower
New Member
```

문제는 새로운 서버가 기존 데이터를 가지고 있지 않다는 것이다.
그래서 Leader가 기존 데이터를 Snapshot으로 전달한다.

```text
Leader
  │
  │ Snapshot
  ▼
New Member
  │
  ▼
기존 상태 복구
  │
  ▼
이후 Log Replication
```

하지만 새로운 서버가 아직 데이터를 따라잡지 못한 상태에서 Quorum 계산에 포함되면 가용성이 떨어질 수 있다.
그래서 etcd에는 Learner라는 상태가 존재한다.

```text
New Server

Learner
  │
  │ 기존 Log Catch-up
  ▼
Follower
```

Learner는 클러스터에는 포함되지만 Quorum 계산에서는 제외된다.
모든 Log를 충분히 따라잡은 뒤 `promote`를 통해 정상 Follower가 된다.

## 9. etcd 서버를 삭제하면?

서버 삭제 역시 Configuration 변경을 Log로 복제하는 방식으로 처리한다.

```text
Before

Leader
Follower 1
Follower 2

      ↓ Remove

Leader
Follower 1
```

특히 Leader 자신을 삭제해야 한다면 조금 다르다.
Configuration 변경을 Commit한 뒤 Leader가 자리에서 내려온다.
이를 Step Down이라고 한다.

```text
Leader 삭제 요청
      │
      ▼
Config Log 복제
      │
      ▼
Commit
      │
      ▼
Leader Step Down
      │
      ▼
새로운 Leader Election
```

이 과정을 통해 서버를 제거하는 중에도 클러스터의 일관성을 유지한다.

## 10. etcd를 오래 운영하면 생기는 문제

Raft만 잘 동작한다고 etcd 운영이 끝나는 것은 아니다.
데이터가 계속 쌓이면 메모리와 디스크 사용량도 계속 증가한다.
그래서 etcd에는 여러 유지보수 기능이 존재한다.

```text
            etcd 운영
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
 Snapshot   Compaction   Defrag
     │          │          │
 Log 정리    Revision 정리  Disk 정리
```

## 11. Snapshot과 Log Retention

etcd는 Log Entry를 메모리에 보관한다.
하지만 무한정 저장하면 결국 OOM이 발생한다.

```text
Log 1
Log 2
Log 3
...
Log 100000
...
```

그래서 일정 수의 Log가 쌓이면 Snapshot을 생성한다.

```text
Memory Log

Log 1
Log 2
...
Log 100000

      │
      ▼

Snapshot 생성
      │
      ▼
이전 Log Truncate
```

기본적으로 `snapshot-count`를 이용해 Snapshot 생성 주기를 설정할 수 있다. 원문에서 설명한 기본값은 100,000이다.

## 12. Revision과 Compaction

etcd는 단순히 현재 Value만 가지고 있는 것이 아니다.
하나의 Key가 변경된 이력을 Revision으로 저장한다.

```text
x = 1   Revision 1
x = 2   Revision 2
x = 3   Revision 3
x = 4   Revision 4
```

계속 쌓이면 디스크 사용량도 증가한다.
그래서 오래된 Revision을 제거하는 Compaction이 필요하다.

```text
Revision

1  2  3  4  5  6  7  8

└─────────┘
 Compaction

            5  6  7  8
```

etcd는 수동 Compaction뿐 아니라 Auto Compaction도 지원한다.
Auto Compaction에는

```text
Revision Mode
Periodic Mode
```

두 가지 방식이 있다.

## 13. Compaction했다고 디스크가 줄어드는 것은 아니다

여기서 중요한 부분이 하나 있다.

```text
Compaction
    │
    ▼
데이터 삭제

BUT

Disk 용량 그대로
```

Compaction은 필요 없는 Revision을 논리적으로 제거하지만 파일의 실제 크기를 바로 줄여주지는 않는다.
그래서 Defragmentation이 필요하다.

```text
Compaction
    │
    ▼
불필요한 Revision 제거
    │
    ▼
Fragmentation 발생
    │
    ▼
Defragmentation
    │
    ▼
실제 Disk 공간 확보
```

RDB에서 `DELETE`를 수행했다고 바로 파일 크기가 줄어들지 않는 것과 비슷하다.
단, Defragmentation을 수행하는 동안에는 read/write가 Block될 수 있기 때문에 운영 시 주의해야 한다.

## 14. Backup과 Restore

etcd 데이터가 유실되는 상황에 대비해 Backup도 필요하다.
대표적으로 다음 명령으로 Snapshot Backup을 만들 수 있다.

```bash
etcdctl snapshot save
```

여기서 Snapshot은 앞서 Log Retention에서 사용했던 Snapshot과 개념이 다르다.

```text
Log Snapshot
→ Log 관리 목적

Backup Snapshot
→ etcd DB 백업 목적
```

`etcdctl snapshot save`로 만든 파일에는 무결성을 확인하기 위한 Hash도 포함된다.
복구할 때는 새로운 etcd Cluster Metadata를 사용해 시작하는 것이 중요하다.
기존 Cluster ID나 Member ID를 그대로 사용하면 기존 노드가 다시 살아났을 때 문제가 발생할 수 있기 때문이다.

## 15. 전체 흐름 정리

etcd의 핵심 동작을 한 번에 보면 다음과 같다.

```text
              Client Write
                   │
                   ▼
                Leader
                   │
            Log Append
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
    Follower 1            Follower 2
        │                     │
        └──────────┬──────────┘
                   │
             Quorum 확보
                   │
                   ▼
                 Commit
                   │
                   ▼
              상태 변경
```

Leader가 장애가 나면

```text
Heartbeat 끊김
     │
     ▼
Election Timeout
     │
     ▼
Candidate
     │
     ▼
RequestVote
     │
     ▼
Quorum 확보
     │
     ▼
New Leader
```

운영 중에는 별도로

```text
Log 증가
  │
  └──▶ Snapshot

Revision 증가
  │
  └──▶ Compaction
          │
          ▼
     Fragmentation
          │
          ▼
    Defragmentation

DB 장애
  │
  └──▶ Backup / Restore
```

까지 관리해야 한다.

## 정리

etcd를 단순히 Kubernetes에서 사용하는 Key-Value DB라고만 보면 중요한 부분을 놓치기 쉽다.
실제로 etcd의 핵심은
여러 서버가 장애 상황에서도 같은 상태를 유지하도록 만드는 분산 시스템에 있다.

```text
RSM
 ↓
Raft
 ↓
Leader Election
 ↓
Log Replication
 ↓
Quorum
 ↓
Consensus
```

그리고 안정적으로 운영하려면 Consensus뿐 아니라

```text
Snapshot
Compaction
Defragmentation
Backup / Restore
```

같은 데이터 관리도 함께 이해해야 한다.
결국 Kubernetes에서 etcd를 제대로 이해한다는 것은 Raft 기반 분산 합의와 상태 복제가 실제 시스템에서 어떻게 동작하는지 이해하는 것에 가깝다.
