# 토스증권 Apache Kafka 데이터센터 이중화 구성 #2: 데이터 미러링

- 읽은 날짜: 2026-09-11
- 회사/작성 주체: 토스증권
- 원문: [토스증권 Apache Kafka 데이터센터 이중화 구성 #2: 데이터 미러링](https://toss.tech/article/kafka-distribution-2)

---

1편에서 토스증권은 두 데이터센터에 독립적인 Kafka 클러스터를 두고 Active-Active 구조를 선택했다.

```text
Data Center A                  Data Center B

┌─────────────┐              ┌─────────────┐
│   Kafka A   │              │   Kafka B   │
└─────────────┘              └─────────────┘
```

하지만 두 Kafka는 서로 독립된 클러스터다.
Kafka A에 들어온 메시지가 Kafka B에 자동으로 생기지는 않는다.

```text
Kafka A

message 1
message 2

Kafka B

message 3
```

데이터센터 하나가 장애 나더라도 서비스를 이어가려면 최종적으로 양쪽 Kafka가 같은 데이터를 가지고 있어야 한다.

```text
Kafka A                  Kafka B

message 1                message 1
message 2                message 2
message 3                message 3
```

그래서 필요한 것이 양방향 데이터 미러링이다.

## 1. 먼저 MirrorMaker2를 사용할 수 없을까?

Kafka에는 클러스터 간 데이터를 복제하기 위한 MirrorMaker2(MM2) 가 있다.
MM2는 Kafka Connect의 Source Connector 기반으로 동작한다.

```text
Source Kafka
     │
     ▼
MirrorMaker2
Source Connector
     │
     ▼
Target Kafka
```

대표적으로 두 가지 역할을 한다.

```text
1. Kafka A의 데이터
       │
       ▼
   Kafka B로 복제

2. A ↔ B Offset 차이
       │
       ▼
   내부 Topic에 기록
```

클러스터 간 복제 자체는 MM2만으로도 가능하다.

## 2. 그런데 Topic 이름이 달라진다

MM2는 데이터를 다른 Kafka로 복제할 때 Source Cluster의 Alias를 Topic 이름 앞에 붙인다.
예를 들어

```text
Kafka A

topic.test.1

        │
        ▼

Kafka B

A.topic.test.1
```

형태가 된다.
토스증권이 원했던 구조는 달랐다.

```text
Kafka A

topic.test.1

Kafka B

topic.test.1
```

어느 데이터센터에 서비스가 배포되더라도 항상 같은 Topic 이름을 사용하게 만들고 싶었다.
그래야 서비스 개발자가

```text
내 서비스가

DC A에 있나?
DC B에 있나?

어떤 Topic을 써야 하지?
```

를 신경 쓰지 않아도 된다.
이 요구사항 때문에 MM2를 그대로 사용하기 어려웠다.

## 3. 그래서 직접 데이터 미러링 도구를 만들었다

토스증권은 Kafka Connect의 Sink Connector를 이용해 직접 미러링 도구를 만들었다.
일반적인 Sink Connector는

```text
Kafka
  │
  ▼
Sink Connector
  │
  ▼
DB / Storage
```

처럼 Kafka 데이터를 외부 시스템으로 전달한다.
토스증권은 Target을 DB가 아니라 다른 데이터센터의 Kafka로 설정했다.

```text
Kafka A
   │
   ▼
Sink Connector
   │
   ▼
Kafka B
```

반대 방향도 똑같이 구성한다.

```text
Kafka A
   │
   ├──────────────▶ Kafka B
   │
   ◀───────────────┤
       Mirroring
```

자체 도구에서는 세 가지 문제에 집중했다.

```text
① 동일 Topic 이름 유지

② 데이터 유실 방지

③ 대규모 Job 모니터링
```

## 4. 첫 번째 문제: 무한 루프

단순히 양쪽 Kafka의 데이터를 서로 전달하면 큰 문제가 생긴다.
DC1에 메시지 하나가 들어왔다고 가정해 보자.

```text
User
 │
 ▼
DC1 Kafka

message 1
```

미러링 시스템이 DC2로 보낸다.

```text
DC1 Kafka
   │
   │ mirror
   ▼
DC2 Kafka

message 1
```

그런데 DC2의 미러링 시스템 입장에서는 이 메시지도 새로운 메시지다.
그래서 다시 DC1로 보낸다.

```text
DC1
message 1
   │
   ▼
DC2
message 1
   │
   ▼
DC1
message 1
   │
   ▼
DC2
message 1

...
```

결과적으로 같은 메시지가 두 Kafka 사이를 무한히 오간다.

## 5. Kafka Header로 원본을 구분한다

토스증권은 이 문제를 Kafka Message Header로 해결했다.
사용자가 직접 보낸 메시지에는 Source DC 정보가 없다.

```text
User Producer

message
Header:
Source DC = 없음
```

이 메시지를 다른 Kafka로 미러링할 때 Header를 추가한다.

```text
DC1 Kafka
   │
   ▼
Mirror

message
Header:
Source DC = DC1

   │
   ▼
DC2 Kafka
```

DC2에서는 Header를 확인한다.

```text
Message 수신
    │
    ▼
Source DC Header 존재?
    │
 ┌──┴──┐
 │     │
NO     YES
 │     │
 ▼     ▼
사용자   이미 미러링된
메시지   메시지
 │     │
 ▼     ▼
상대 DC  전송하지 않음
전송
```

즉,

```text
Header 없음
→ 실제 Producer가 생성
→ 상대 Kafka로 전송

Header 있음
→ 이미 미러링된 데이터
→ 다시 전송하지 않음
```

으로 구분했다.
이 방식으로 양방향 복제를 유지하면서 무한 루프를 막았다.

## 6. Offset 정보도 Header에 같이 넣는다

토스증권은 메시지를 미러링할 때 원본 Kafka의 Offset도 Header에 넣었다.

```text
Kafka A

Topic : payment
Partition : 0
Offset : 120

      │
      ▼

Mirror

Header
sourceOffset = 120

      │
      ▼

Kafka B
```

이 정보는 이후 Consumer가 다른 데이터센터로 넘어갈 때 필요한 Offset Sync에 사용된다.
즉, 2편의 데이터 미러링은 단순히 데이터를 복사하는 것뿐 아니라 3편의 Offset Sync를 위한 정보도 함께 남기고 있다.

## 7. 두 번째 문제: 메시지 전송이 실패하면?

Kafka A에서 Kafka B로 데이터를 보내다가 오류가 발생할 수 있다.

```text
Kafka A
   │
   ▼
Mirror
   │
   X
Kafka B

전송 실패
```

이 메시지를 그냥 버리면 두 Kafka의 데이터가 달라진다.

```text
Kafka A          Kafka B

1                1
2                2
3                X
4                4
```

Active-Active 구조에서 이런 데이터 유실은 치명적이다.

## 8. DLQ로 실패한 메시지를 보관한다

이를 위해 Kafka Connect의 Dead Letter Queue(DLQ) 기능을 사용했다.
정상 메시지는 그대로 Target Kafka로 간다.

```text
Message
   │
   ▼
Sink Connector
   │
   ▼
Target Kafka
```

처리에 실패하면 별도의 DLQ Topic으로 보낸다.

```text
Message
   │
   ▼
Sink Connector
   │
   X Error
   │
   ▼
DLQ Topic
```

DLQ에는 원본 메시지뿐 아니라 다음과 같은 정보도 함께 들어간다.

```text
Original Message

Topic
Partition
Offset
Exception
...
```

이후 별도의 재처리 시스템이 DLQ 메시지를 다시 Target Kafka로 전달한다.

```text
Target 전송 실패
      │
      ▼
    DLQ
      │
      ▼
 Retry System
      │
      ▼
Target Kafka
```

이를 통해 전송 오류가 발생하더라도 메시지를 버리지 않고 다시 처리할 수 있게 했다.

## 9. 세 번째 문제: Job이 수천 개가 되었다

처음에는 단순했다.
토픽 하나당 Connector 하나를 만들었다.

```text
Topic A → Connector A
Topic B → Connector B
Topic C → Connector C
```

이 방식은 장점이 많다.

```text
Topic별

독립 실행
독립 성능
독립 모니터링
```

문제는 Topic이 계속 증가하면서 발생했다.

```text
Topic 1
Topic 2
Topic 3
...
Topic 500
...
Topic 1000+
```

Connector 수도 같이 증가했다.

## 10. Kafka Connect 내부 구조

Kafka Connect에서는 크게 Connector와 Task가 동작한다.

```text
Kafka Connect Cluster

       Connector
           │
     작업을 분할
           │
   ┌───────┼───────┐
   ▼       ▼       ▼
 Task 0   Task 1   Task 2
```

Connector
어떤 데이터를 어디로 이동시킬지 관리한다.
Task
실제 데이터 이동을 수행하는 작업 단위다.
Task를 여러 개 두면 병렬 처리가 가능하다.

## 11. 하나의 거대한 Connect Cluster

처음에는 Kubernetes 위에 큰 Kafka Connect Cluster 하나를 만들었다.

```text
         Kafka Connect Cluster

┌───────────────────────────────────┐
│ Connector 1                       │
│ Connector 2                       │
│ Connector 3                       │
│ Connector 4                       │
│ ...                               │
│ Connector 500                     │
└───────────────────────────────────┘
```

토픽 하나마다 Connector 하나가 있으니 시간이 지나면서 수백 개의 Connector가 한 클러스터에 모이기 시작했다.
그 결과

```text
많은 Connector
      │
      ▼
Task Scheduling 부담
      │
      ▼
운영 복잡도 증가
```

문제가 나타났다.

## 12. 큰 클러스터를 여러 작은 클러스터로 나눴다

해결 방법은 단순했다.

```text
기존

┌────────────────────────────┐
│      Connect Cluster       │
│ Connector 1 ~ 1000        │
└────────────────────────────┘

개선

┌───────────┐
│ Cluster A │
└───────────┘

┌───────────┐
│ Cluster B │
└───────────┘

┌───────────┐
│ Cluster C │
└───────────┘
```

하나의 큰 클러스터를 N개의 작은 Kafka Connect Cluster로 분리했다.
이렇게 하면 Connector와 Task를 여러 클러스터에 분산할 수 있다.

```text
Cluster A

Connector 1
Connector 2

Cluster B

Connector 3
Connector 4

Cluster C

Connector 5
Connector 6
```

그 결과 Task를 더 유연하게 배치할 수 있었고, 스케줄링 부담도 줄었다.
또 특정 Connect Cluster만 작업할 수 있기 때문에 운영 중 변경의 영향 범위도 작아졌다.

## 13. 그런데 수천 개 Job을 어떻게 감시할까?

데이터 미러링에서 중요한 것은

> "지금 메시지가 잘 복제되고 있는가?"
> 를 계속 확인하는 것이다.
> 처음에는 Kafka Connect Metric을 MBean으로 등록하고

```text
Kafka Connect
     │
     ▼
   MBean
     │
     ▼
    JMX
     │
     ▼
Prometheus
```

형태로 수집했다.
하지만 문제가 있었다.

```text
Prometheus

① 장기간 데이터 보관 어려움

② 복잡한 PromQL
   Label Matching 필요
```

특히 장기간의 데이터를 이용한 추세 분석이 어려웠다.

## 14. Metric도 Kafka로 보낸다

그래서 Custom Metric Reporter를 만들었다.

```text
Kafka Connect
      │
      ▼
Custom Metric Reporter
      │
      ▼
    Kafka
```

각 미러링 Job에서 생성되는 Metric을 Kafka Topic으로 직접 보낸다.

```text
Job 1 ──┐
Job 2 ──┤
Job 3 ──┤
...     ├──▶ Metric Kafka Topic
Job N ──┘
```

이렇게 하면 수천 개 Job에서 발생하는 Metric을 하나의 데이터 파이프라인으로 관리할 수 있다.

## 15. 가장 중요한 Metric은 미러링 지연 시간

특히 중요하게 본 것은 Replication Latency다.

```text
Source Kafka
Message 도착
10:00:00.100

      │
      ▼
Mirroring

      │
      ▼

Target Kafka
Message 도착
10:00:00.120
```

그러면

```text
Replication Latency

= 20ms
```

가 된다.
즉,

```text
Source Kafka에 들어온 시점

        ↓

Target Kafka에 들어온 시점

        ↓

두 시간 차이 측정
```

을 통해 미러링 시스템의 상태를 판단한다.
토스증권은 ms 단위의 지연 시간을 중요한 성능 지표로 활용했다.

## 16. Metric 저장은 ClickHouse

Metric은 Kafka에 모은 뒤 ClickHouse에 저장했다.

```text
Kafka Connect Jobs
       │
       ▼
Metric Reporter
       │
       ▼
     Kafka
       │
       ▼
  ClickHouse
       │
       ▼
Dashboard / 분석
```

ClickHouse는 OLAP 성격의 데이터베이스라 대량의 데이터를 SQL로 분석하기 좋다.
장기간 Metric 데이터를 저장하면서

```text
오늘

어제

지난주

지난달
```

단위의 추세도 분석할 수 있게 됐다.

## 17. 이상 징후도 탐지할 수 있다

예를 들어 특정 Topic의 평소 메시지 인입량이

```text
평소

1000
1050
980
1020
990
```

정도였다고 가정해 보자.
갑자기

```text
1000
1050
980
50
20
```

으로 떨어진다면 문제가 발생했을 가능성이 높다.
장기간 저장된 Metric을 이용하면 이러한 변화도 빠르게 찾을 수 있다.

```text
Metric History
      │
      ▼
Trend 분석
      │
      ▼
이상치 탐지
      │
      ▼
운영 문제 조기 발견
```

Prometheus만 사용했을 때보다 장기 분석과 이상 징후 탐지가 쉬워졌다.

## 18. 전체 동작 다시 보기

최종 구조를 단순화하면 다음과 같다.

```text
                 Data Center A

Producer
   │
   ▼
Kafka A
   │
   │ User Message
   ▼
Sink Connector
   │
   │ Header 추가
   │ Source DC = A
   │ Source Offset = XXX
   ▼
Kafka B
   │
   ▼
Header 확인
   │
   └── 이미 Mirroring된 메시지
       → 다시 A로 보내지 않음

                 Data Center B
```

반대 방향도 똑같이 동작한다.

```text
Kafka A ◀══════════════════▶ Kafka B

        Bidirectional
         Mirroring

Header를 이용한
Loop Prevention
```

전송 실패가 발생하면

```text
Mirroring
    │
    X
    │
    ▼
   DLQ
    │
    ▼
Retry System
    │
    ▼
Target Kafka
```

로 처리한다.

## 19. 운영 구조까지 합치면

```text
              Kafka Connect

       ┌─────────┼─────────┐
       ▼         ▼         ▼

 Connect A   Connect B   Connect C
    │           │           │
Connector    Connector    Connector
    │           │           │
   Task        Task        Task

       │
       └──────────────┐
                      ▼
               Metric Reporter
                      │
                      ▼
                    Kafka
                      │
                      ▼
                 ClickHouse
                      │
                      ▼
             Monitoring Dashboard
```

하나의 큰 Connect Cluster에 모든 Connector를 몰아넣는 대신 여러 Cluster로 분산하고, Metric 역시 별도의 데이터 파이프라인으로 관리한 것이다.

## 20. 이 사례에서 중요한 점

처음 문제는 단순했다.

```text
Kafka A와 Kafka B의
데이터를 동일하게 만들자.
```

하지만 실제 구현을 시작하면 여러 문제가 생긴다.

```text
양방향 미러링
      │
      ├── 무한 루프
      │
      ├── 메시지 유실
      │
      ├── 수백~수천 Connector
      │
      └── 운영 상태 확인
```

토스증권은 각각을 다음 방식으로 해결했다.

```text
무한 루프
   ↓
Kafka Header

전송 실패
   ↓
DLQ + Retry

대규모 Connector
   ↓
Connect Cluster 분리

대규모 모니터링
   ↓
Custom Metric Reporter
   ↓
Kafka
   ↓
ClickHouse
```

결국 이 사례에서 중요한 부분은 Kafka 데이터를 복제했다는 것 자체보다, 두 개의 독립된 Kafka 클러스터를 실제 운영 환경에서 하나처럼 유지하기 위해 필요한 문제들을 하나씩 해결했다는 점이다.

```text
Kafka A                  Kafka B
   │                        │
   └────── Mirroring ───────┘
              │
        Loop Prevention
              │
             DLQ
              │
         Monitoring
              │
      대규모 운영 구조
```

그리고 미러링 과정에서 원본 Offset을 Header에 함께 기록해둔 이유는 다음 단계인 Consumer Group Offset Sync로 이어진다.
즉, 2편이
“데이터를 어떻게 동일하게 만들 것인가”
에 대한 이야기라면,
3편은
“Consumer가 다른 Kafka로 넘어갔을 때 어디서부터 다시 읽어야 하는가”
를 해결하는 이야기다.
