# MySQL Orchestrator 기반의 새로운 HA 표준 개발기

- 읽은 날짜: 2026-09-23
- 회사/작성 주체: Kakao
- 원문: [MySQL Orchestrator 기반의 새로운 HA 표준 개발기](https://tech.kakao.com/posts/775)
- 노션 정리: [원본 페이지](https://app.notion.com/p/3e6a987f6c40801c8714f38a9d62468e)

---

## MySQL Orchestrator 기반 HA 표준 개발기

카카오는 대규모 MySQL 환경의 고가용성을 위해 오랫동안 **MHA(Master High Availability)** 기반 구성을 사용했다.

```text
                Source
                  │
            Replication
           ┌──────┴──────┐
           ▼             ▼
       Replica 1      Replica 2

                  +

                 MHA
```

Source에 장애가 발생하면 Replica 하나를 새로운 Source로 승격해 서비스를 이어가는 구조다.
하지만 MySQL 버전과 OS가 계속 변하고, Replica 활용이나 네트워크 장애 대응 같은 새로운 요구사항이 생기면서 MHA만으로는 대응하기 어려워졌다.
카카오는 2023년부터 차세대 HA 구조를 검토했고 최종적으로 **Orchestrator 기반의 새로운 MySQL HA 표준**을 만들었다.

---

## 1. 먼저 HA와 Failover

HA는 일부 서버가 장애 나더라도 서비스를 계속 제공할 수 있도록 만드는 구조다.
정상 상태에서는 Source가 Write를 받고 Replica가 이를 복제한다.

```text
Application
     │
     ▼
   Source
     │
     ├────────▶ Replica 1
     │
     └────────▶ Replica 2
```

문제는 Source가 죽었을 때다.

```text
Source
   X

Replica 1
Replica 2

     │
     ▼

Replica 중 하나를
새 Source로 승격
```

이 과정이 **Failover**다.

---

## 2. 기존 MHA의 역할

MHA는 Source 상태를 감시하다 장애를 감지하면 Replica를 승격한다.

```text
              MHA
               │
          Health Check
               │
               ▼
             Source
               │
        ┌──────┴──────┐
        ▼             ▼
    Replica 1      Replica 2
```

```text
Source Down
     │
     ▼
장애 감지
     │
     ▼
Replica 상태 확인
     │
     ▼
승격 대상 선택
     │
     ▼
New Source
```

카카오는 2015년경부터 이런 MHA 기반의 MySQL HA 표준을 사용했다.

---

## 3. 그런데 MHA가 환경 변화를 따라가기 어려워졌다

MHA는 장기간 큰 기능 추가와 유지보수가 거의 없었다.

```text
과거

MySQL
OS
단순 Source/Replica


          ↓


현재

새로운 MySQL
새로운 OS
DBaaS
Replica 서비스 활용
복잡한 네트워크
대규모 자동화
```

또 Perl 기반이라 내부 로직 분석이나 카카오 환경에 맞는 수정에도 부담이 컸다.

---

## 4. 더 큰 문제는 장애 판단 방식

MHA의 기본적인 관점은 Source 중심이다.

```text
MHA
 │
 ▼
Source

접속 가능?
접속 불가능?
```

그런데 네트워크만 단절됐다고 해보자.

```text
MHA
 │
 X Network
 │
Source

MySQL Process = 정상
```

MHA 관점에서는 Source 자체가 죽은 것과 네트워크가 끊긴 것을 구분하기 어려워질 수 있다.
또 카카오에서는 Replica를 실제 서비스에 활용하고 있었기 때문에 Source만이 아니라 Replica 상태까지 함께 관리하고 싶었다.

```text
필요한 정보

Source Health

+

Replica Health

+

Replication State

+

Network State
```

MHA 구조로는 이런 요구사항을 확장하기 어려웠다.

---

## 5. 새로운 HA 솔루션 선정

카카오는 여러 후보를 비교했다.

```text
MySQL Cluster
ProxySQL
Pacemaker
Percona XtraDB Cluster
MySQL InnoDB Cluster
Orchestrator
...
```

그리고 최종적으로 **Orchestrator**를 선택했다.

---

## 6. Orchestrator란?

Orchestrator는 하나의 Source만 감시하는 도구라기보다 **MySQL Replication Topology 전체를 이해하고 관리하는 HA 도구**다.

```text
                  Orchestrator
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
         Source     Replica 1   Replica 2
            │          │          │
            └──── Replication ────┘
```

Orchestrator는

```text
현재 Source는 누구인가?

각 Replica는 누구를 복제하는가?

Replication은 정상인가?

어떤 서버를 승격할 수 있는가?
```

를 Topology 관점에서 판단한다.

---

## 7. 여러 MySQL Cluster를 하나에서 관리

MHA에서는 각각의 HA 구성을 별도로 관리하는 형태였다면,

```text
MHA A → Cluster A

MHA B → Cluster B

MHA C → Cluster C
```

Orchestrator에서는 하나의 Cluster가 여러 MySQL Replication Topology를 관리할 수 있다.

```text
                  Orchestrator

        ┌────────────┼────────────┐
        ▼            ▼            ▼

     Cluster A    Cluster B    Cluster C

      Source       Source       Source
       │            │            │
    Replicas     Replicas     Replicas
```

대규모 MySQL 운영 환경에 더 잘 맞는 구조다.

---

## 8. 그런데 Orchestrator 자체가 죽으면?

HA 관리자가 하나뿐이면 그 자체가 SPOF가 된다.

```text
Orchestrator
     X

     +

MySQL Source
     X

     ↓

Failover를 수행할
주체가 없음
```

그래서 **HA 시스템 자체도 HA가 필요하다.**

---

## 9. Orchestrator의 HA 방식

대표적으로 네 가지 방식이 있다.

```text
No HA

Semi HA

Shared Backend HA

Raft HA
```

카카오는 이 중 **Raft HA**를 선택했다.

```text
Orchestrator 1

Orchestrator 2

Orchestrator 3

      │
      ▼
Raft Consensus
      │
      ▼
Leader 선출
```

구성 난이도는 높지만 Orchestrator Manager 자체의 HA까지 확보할 수 있기 때문이다.

---

## 10. Raft HA의 동작

3개의 Orchestrator가 있다고 해보자.

```text
              Leader
                │
         ┌──────┴──────┐
         ▼             ▼
     Follower       Follower
```

Leader가 MySQL Topology를 관리하고 장애 판단을 수행한다.
중요한 것은 **Quorum**이다.

```text
3 Nodes

Quorum = 2
```

한 대가 죽어도

```text
Node 1  O
Node 2  O
Node 3  X

    ↓

2 / 3

    ↓

정상 동작
```

할 수 있다.

---

## 11. 네트워크가 갈라졌을 때

```text
         Network Partition

 Node 1 + Node 2       Node 3
```

왼쪽은

```text
2 / 3

Quorum O
```

오른쪽은

```text
1 / 3

Quorum X
```

다.
따라서 Node 3이 혼자 새로운 Leader가 될 수 없다.

```text
2 Nodes
   │
   ▼
Leader 가능


1 Node
   │
   X
Leader 불가
```

양쪽에서 서로 Leader라고 주장하는 **Split-Brain**을 막을 수 있다.

---

## 12. 왜 3대, 5대처럼 홀수가 좋을까?

```text
3 Nodes

Quorum = 2
장애 허용 = 1
```

```text
4 Nodes

Quorum = 3
장애 허용 = 1
```

한 대를 추가했지만 장애 허용 수는 그대로다.
반면

```text
5 Nodes

Quorum = 3
장애 허용 = 2
```

가 된다.
그래서 Raft Cluster는 보통 홀수 Node 구성이 효율적이다.

---

## 13. 전체 기본 구조

```text
                    Orchestrator Cluster

              Orchestrator 1 - Leader
                    /       \
                   /         \
         Orchestrator 2     Orchestrator 3
            Follower           Follower

                       │
                    Raft
                       │
                       ▼

             MySQL Topology 관리

       ┌───────────────┼───────────────┐
       ▼               ▼               ▼

    Cluster A        Cluster B        Cluster C

     Source           Source           Source
     /   \            /   \            /   \
 Replica Replica   Replica Replica   Replica Replica
```

즉

```text
MySQL HA

+

HA Manager의 HA
```

를 동시에 구성했다.

---

## 14. 오픈소스를 그대로 사용하지는 않았다

카카오에는 이미 대규모 MySQL을 관리하는 DBaaS가 있었다.
따라서 Orchestrator를 독립적인 섬처럼 운영할 수 없었다.

```text
Kakao DBaaS
      │
      ▼
    Protego
      │
      ▼
Orchestrator
```

이 연동 시스템을 **Protego**라고 만들었다.
Protego는

```text
DB Meta 정보 제공

DBaaS ↔ Orchestrator 연동
```

등의 역할을 담당했다.

---

## 15. Orchestrator 내부 구성

Orchestrator는 크게 다음 영역으로 볼 수 있다.

```text
Orchestrator

├─ Core Engine
├─ Web Interface
├─ Client API
├─ Backend DB
└─ Hook
```

카카오는 upstream 변경을 계속 따라갈 수 있도록 **코드 수정은 최소화**하는 방향을 선택했다.

```text
Open Source Orchestrator

         +

최소한의 Kakao Patch

         +

별도의 연동 시스템
```

---

## 16. 카카오가 추가한 주요 기능

Web UI에서는 위험한 기능을 제거했다.
Client API에는 Protego에서 사용할 API를 추가했다.
Hook에서는 장애나 상태 변화에 대한 알림을 연결했다.
Backend DB는 Protego에서 여러 자동화 기능을 수행할 때 활용했다.

---

## 17. Replica도 서비스에 사용한다

카카오에서는 Replica를 Read 용도로 서비스에 투입할 필요가 있었다.
문제는 Replica가 장애 났을 때다.

```text
Application
    │
    ▼
Replica Domain
    │
    ▼
Replica A

Replica A
   X
```

이때 장애 Replica를 서비스에서 제외해야 한다.
그래서 **DNS Failover**를 만들었다.

---

## 18. DNS Failover 동작

```text
Application
     │
     ▼
Replica Domain
     │
     ▼
Replica A
```

Orchestrator Backend DB에서 Replica 상태를 확인한다.

```text
Orchestrator
     │
     ▼
Replica 상태 확인
     │
 ┌───┴────┐
 ▼        ▼
정상      장애
 │         │
 ▼         ▼
유지      Domain 변경
```

Replica A가 죽었다면

```text
Before

read.db.example
       │
       ▼
Replica A


After

read.db.example
       │
       ▼
Replica B
```

처럼 DNS Mapping만 변경한다.
원문에서는 현재 약 5분 정도의 간격을 두고 도메인을 변경하도록 구성했다고 설명한다.

---

## 19. 이제 가장 중요한 실제 Failover 동작

Orchestrator는 MySQL 상태를 세 가지 정보를 이용해 판단한다.

```text
Source

SELECT 1 결과


Replica

SHOW REPLICA STATUS


Replica

SHOW SOURCE STATUS
```

즉 Source에 직접 접근한 결과뿐 아니라 **Replica가 Source를 어떻게 보고 있는지**까지 함께 본다.

---

## 20. Source Process 자체가 죽은 경우

정상:

```text
             Source
               │
         Replication
         ┌─────┴─────┐
         ▼           ▼
     Replica A   Replica B

          ▲
          │
    Orchestrator
```

Source의 MySQL Process가 내려갔다.

```text
Source
  X
```

Orchestrator가 Source에 접근하지 못한다.

```text
SELECT 1

    X
```

동시에 Replica의 Replication 상태도 변한다.

```text
Replica

IO Thread

Running
   │
   ▼
Connecting
```

Orchestrator는

```text
Source Unreachable

       +

Replica Replication 이상

       ↓

실제 Source 장애
```

로 판단하고 Failover를 진행한다.
원문에서는 이런 경우 장애 감지부터 Failover까지 약 **2\~6초** 정도가 걸렸다고 설명한다.

---

## 21. Source는 살아있고 Network만 끊기면?

이번에는 MySQL Process는 정상이다.

```text
Source

MySQL = O
```

하지만 Network가 끊겼다.

```text
Orchestrator

     X Network

Source
```

Orchestrator 입장에서는 Source 접근이 안 된다.
하지만 Replica는 Source Process가 실제로 종료된 것이 아니기 때문에 즉시 Dead Signal을 받지 못한다.

```text
Replica

"Source가 죽었나?"

아직 확정 못함
```

이때 중요한 설정이 `slave_net_timeout`이다.

---

## 22. slave_net_timeout

Replica가 Source로부터 일정 시간 동안 이벤트를 받지 못했을 때

```text
Replication Connection이
끊어진 것으로 판단하기까지
얼마나 기다릴 것인가?
```

를 결정한다.

```text
Network 끊김
    │
    ▼
Replica 대기
    │
    │ slave_net_timeout
    ▼
Replication 이상 확정
    │
    ▼
Orchestrator 판단
    │
    ▼
Failover
```

값이 너무 크면 Network 장애 Failover가 늦어진다.

---

## 23. Process 장애와 Network 장애의 차이

#### Source Process Down

```text
Source Down
   │
   ▼
Replica가 빠르게 이상 감지
   │
   ▼
Orchestrator
   │
   ▼
Failover

약 2~6초
```

#### Network 장애

```text
Source Process = 정상

Network = X
     │
     ▼
Replica 즉시 확정 못함
     │
     ▼
slave_net_timeout 기다림
     │
     ▼
Replication 이상
     │
     ▼
Failover
```

같은 "Source에 접근할 수 없음"이라도 내부 동작이 다르다.

---

## 24. 그래서 Timeout을 조정했다

카카오는 이 동작을 분석한 뒤 MySQL의 `slave_net_timeout` 값을 변경했다.
그 결과 Network 장애에서도 **약 5\~10초 내 Failover**가 이루어지도록 개선했다.
이 부분이 이 사례에서 중요한 포인트다.
HA 솔루션만 교체한다고 끝나는 것이 아니라
> **HA 솔루션이 장애를 판단하는 방식에 맞춰 DB 설정까지 함께 변경해야 했다.**

---

## 25. 배포도 자동화했다

대규모 MySQL 환경에서 Orchestrator를 수동으로 설치하고 설정하면 휴먼 에러 가능성이 크다.
그래서

```text
초기 Cluster 구성

배포

빌드
```

세 과정을 자동화했다.

```text
Git

  │
  ▼
Jenkins

  │
  ├─ Build
  ├─ Config
  ├─ Deploy
  └─ Restart
```

초기 구성의 여러 수동 작업도 Script로 자동화했고, 배포와 빌드 역시 Jenkins 기반으로 관리했다.

---

## 26. 전체 동작

```text
                      Application
                           │
                           ▼
                       MySQL Source
                        /        \
                       /          \
                 Replica A      Replica B
                    │              │
                    └──────┬───────┘
                           │
                           ▼
                Orchestrator Leader
                           │
             ┌─────────────┼─────────────┐
             │             │             │
          SELECT 1   SHOW REPLICA   SHOW SOURCE
             │          STATUS         STATUS
             └─────────────┬─────────────┘
                           │
                           ▼
                     장애 판단
                           │
                     ┌─────┴─────┐
                     │           │
                   정상        장애
                                 │
                                 ▼
                          Replica 승격
                                 │
                                 ▼
                           New Source


Orchestrator 자체

Leader + Followers
       │
       ▼
Raft / Quorum


Replica 서비스

Replica 상태
    │
    ▼
Protego
    │
    ▼
DNS Failover
```

---

## 정리

카카오의 HA 개선은 단순한

```text
MHA → Orchestrator
```

교체가 아니다.
실제로는

```text
MHA 구조적 한계
       │
       ▼
Topology 기반 관리 필요
       │
       ▼
Orchestrator
       │
       ▼
Orchestrator 자체 HA
       │
       ▼
Raft
       │
       ▼
DBaaS 연동
       │
       ▼
Protego
       │
       ├─ Replica DNS Failover
       │
       └─ 운영 자동화
       │
       ▼
실제 장애 판단 분석
       │
       ▼
slave_net_timeout 조정
```

까지 이어진 **HA 운영 표준 자체의 재설계**다.
특히 중요한 것은

```text
"Source에 접속이 안 된다"

        ≠

"Source Process가 죽었다"
```

라는 점이다.
Orchestrator는 Source 자체뿐 아니라 Replica에서 관찰되는 Replication 상태까지 함께 보면서 장애를 판단하고, 카카오는 그 판단 방식에 맞춰 MySQL 설정까지 조정했다.
결국 HA에서 중요한 것은 단순히 **장애가 나면 Replica를 승격하는 것**이 아니라,
**장애를 얼마나 정확하고 빠르게 판단하면서 Split-Brain이나 잘못된 Failover 없이 안전하게 역할을 전환할 것인가**에 있다.
