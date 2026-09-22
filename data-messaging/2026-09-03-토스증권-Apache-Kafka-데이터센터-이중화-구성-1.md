# 토스증권 Apache Kafka 데이터센터 이중화 구성 #1

- 읽은 날짜: 2026-09-03
- 회사/작성 주체: 토스증권
- 원문: [토스증권 Apache Kafka 데이터센터 이중화 구성 #1](https://toss.tech/article/kafka-distribution-1)

---

서비스를 안정적으로 운영하려면 서버 한 대의 장애뿐 아니라 데이터센터 전체가 장애 나는 상황도 고려해야 한다.
토스증권은 두 데이터센터에 Kubernetes 클러스터를 구성하고, 평소에도 양쪽에서 모두 트래픽을 처리하는 Active-Active 구조를 사용하고 있었다.

```text
                User Traffic
                     │
             ┌───────┴───────┐
             ▼               ▼
       Data Center A    Data Center B
       Kubernetes       Kubernetes
          Active           Active
```

문제는 서비스 서버만 이중화한다고 끝나는 것이 아니라는 점이다.
MSA 환경에서 Kafka가 메시지 브로커 역할을 하고 있다면 Kafka 역시 데이터센터 장애를 견딜 수 있어야 한다.

```text
Service A ─┐
Service B ─┼──▶ Kafka ──▶ Service C
Service D ─┘
              ▲
              │
        여기가 죽는다면?

서비스 서버가 살아 있어도
전체 메시지 흐름이 멈출 수 있음
```

그래서 Kafka까지 데이터센터 단위로 이중화해야 했다.

## 1. Kafka는 왜 이중화하기 어려울까?

Stateless 서버는 상태를 가지고 있지 않기 때문에 다른 데이터센터에서 다시 실행하기가 상대적으로 쉽다.
하지만 Kafka는 메시지와 Offset 같은 상태를 가지고 있는 Stateful 시스템이다.

```text
Stateless Server

Server A  ──X

      ↓

Server B에서 다시 요청 처리

상태 이전 문제 ↓

Kafka

Message
Partition
Offset
Consumer Group
...

상태를 그대로 유지해야 함
```

따라서 Kafka를 단순히 다른 데이터센터에 한 대 더 설치한다고 이중화가 끝나는 것은 아니다.

## 2. Kafka 이중화의 두 가지 방법

Kafka 데이터센터 이중화에는 크게 두 가지 방법이 있다.

```text
          Kafka Multi-DC

			        │
			   ┌────┴─────┐
			   ▼          ▼

Active-Active   Stretched Cluster
```

### Active-Active

두 데이터센터에 서로 독립된 Kafka 클러스터를 각각 구성한다.

Kafka A와 Kafka B는 완전히 별개의 클러스터다.
따라서 한쪽 Kafka에 들어온 데이터를 다른 쪽에도 전달하는 데이터 미러링이 필요하다.

#### 장점

```text
Kafka A  ← 독립 → Kafka B

DC A 장애

Kafka A X

Kafka B O
```

한쪽 데이터센터의 장애가 다른 Kafka 클러스터에 영향을 주지 않는다.
또한 서비스는 같은 데이터센터의 Kafka와 통신하기 때문에 데이터센터 간 네트워크를 매번 사용할 필요가 없다.

#### 단점

대신 운영해야 할 것이 많아진다.

```text
Kafka A                Kafka B

Topic A ◀── Mirror ──▶ Topic A
Topic B ◀── Mirror ──▶ Topic B
Topic C ◀── Mirror ──▶ Topic C

Consumer Offset
        ◀──── Sync ────▶

ACL
Topic 설정
권한
모니터링
...
```

토픽뿐 아니라 Consumer Group Offset까지 서로 맞춰야 한다.
결국 두 Kafka를 하나처럼 운영하는 작업이 필요하다.

## 3. Stretched Cluster

Stretched Cluster는 두 데이터센터의 Kafka 노드를 하나의 Kafka 클러스터로 구성하는 방식이다.

클러스터가 하나이기 때문에 별도의 데이터 미러링이 필요하지 않는다.
운영 측면에서는 Active-Active보다 단순하다.

## 4. 하지만 Stretched Cluster에는 문제가 있었다

Kafka는 메시지를 저장할 때 Leader Partition이 요청을 받은 뒤 Replica에 데이터를 복제한다.

```text
Producer
   │
   ▼
Leader Partition
   │
   │ Replication
   ▼
Replica
   │
   ▼
ACK
```

Stretched Cluster에서는 Leader와 Replica가 서로 다른 데이터센터에 있을 수 있다.

```text
DC A                         DC B

Producer
   │
   ▼
Leader ───── DC Network ─────▶ Replica
   ◀──────────── ACK ──────────
```

따라서 Kafka 요청을 처리할 때마다 데이터센터 간 네트워크를 지나야 할 수 있다.
결국 데이터센터 거리가 멀수록 Latency가 증가한다.
또 하나의 문제는 데이터센터 간 네트워크 단절이다.

```text
정상

DC A ───────── DC B

Network 장애

DC A     X     DC B

??
어느 쪽이 정상 클러스터인가?
```

당시 구성에서는 Kafka 상태 관리에 ZooKeeper를 사용하고 있었고, 데이터센터가 둘로 갈라졌을 때 정족수와 Split-brain 문제를 고려해야 했다.

### 정족수(Quorum)와 Split-brain

둘 다 분산 시스템에서 여러 서버가 서로 통신하지 못할 때, 누가 진짜 정상 상태인지 결정하는 문제와 관련 있다.

#### 1. 정족수(Quorum)

의사결정을 하기 위해 필요한 최소 노드 수다.
예를 들어 ZooKeeper 서버가 5대라면 과반수인 3대 이상이 동의해야 정상적으로 의사결정을 할 수 있다.

```text
ZooKeeper 5대

ZK1 ─┐
ZK2 ─┼── 3대 이상 동의 → 정족수 충족 → 서비스 가능
ZK3 ─┘

ZK4 ── 장애
ZK5 ── 장애
```

과반수를 사용하는 이유는 동시에 두 그룹이 정족수를 확보하지 못하게 하기 위해서다.
따라서 ZooKeeper 같은 시스템은 일반적으로 홀수 개(3, 5, 7…)로 구성한다.

#### 2. Split-brain

Split-brain은 네트워크 장애 때문에 클러스터가 두 그룹으로 갈라졌는데, 양쪽 모두 자신이 정상이라고 판단하는 상황이다.
데이터센터가 두 개 있다고 가정해 보자.

```text
정상 상황

[DC A] ←────────────→ [DC B]
 Kafka / ZooKeeper     Kafka / ZooKeeper

데이터센터 간 네트워크 장애 발생

[DC A]       X       [DC B]
 "내가 정상"           "나도 정상"
     ↓                   ↓
 Leader A             Leader B

       Split-brain
```

이 상태에서 양쪽이 모두 쓰기를 허용하면:

```text
DC A → 데이터 X 저장
DC B → 데이터 Y 저장

네트워크 복구

        ↓

X와 Y 중 어떤 상태가 진짜인가?
```

데이터가 서로 달라지는 불일치 문제가 생길 수 있다.

#### 3. 그래서 정족수가 Split-brain을 막는다

ZooKeeper가 5대이고 데이터센터가 분리됐다고 가정해 보자.

```text
        네트워크 단절

DC A                  DC B
ZK1                   ZK4
ZK2        X          ZK5
ZK3

3대                   2대
 ↓                     ↓
정족수 O               정족수 X
서비스 계속             의사결정 불가
```

5대의 과반수는 3대다.
따라서 DC A만 정족수를 확보하고, DC B는 새로운 리더를 마음대로 선출하거나 중요한 상태 변경을 할 수 없다.
즉,
정족수(Quorum)는 “누가 결정할 권한을 가지는가”를 정해서 Split-brain을 방지하는 핵심 장치다.

##### 한 줄 정리

| 개념           | 의미                                                                                   |
| -------------- | -------------------------------------------------------------------------------------- |
| Quorum(정족수) | 클러스터가 의사결정하기 위해 필요한 최소 노드 수. 보통 과반수                          |
| Split-brain    | 네트워크 단절로 클러스터가 나뉘고 양쪽이 자신이 정상이라고 판단하는 문제               |
| 관계           | 과반수 정족수를 사용하면 두 그룹이 동시에 결정권을 가지는 것을 막아 Split-brain을 방지 |

Kafka + ZooKeeper 문맥에서는 “데이터센터를 2개로 나누면 ZooKeeper를 왜 2:2처럼 배치하면 곤란한가?”까지 이해하면 이 구조를 이해할 수 있다.

토스증권은 이러한 가용성과 성능 문제 때문에 Stretched Cluster를 선택하지 않았다.

## 5. 결국 Active-Active를 선택했다

토스증권이 중요하게 본 기준은 세 가지였다.

```text
1. 장애가 발생해도 가용성을 확보할 수 있는가?

2. 이중화 때문에 성능이 크게 떨어지지 않는가?

3. 운영 가능한 구조인가?
```

여기서 가장 중요했던 것은 가용성과 성능이었다.

```text
Active-Active

DC A Kafka        DC B Kafka
    │                 │
 독립 클러스터      독립 클러스터
    │                 │
Local 통신         Local 통신

장애 격리 O
Latency 감소
```

운영 복잡도는 높아지더라도 가용성과 성능을 확보할 수 있었기 때문에 Active-Active 구조를 선택했다.

## 6. 그런데 Active-Active도 문제가 있다

이제 두 Kafka 클러스터가 생겼다.

```text
DC A                      DC B

Kafka A                  Kafka B
```

하지만 두 Kafka가 서로 독립되어 있기 때문에

```text
Kafka A

Message
A
B
C

Kafka B

?
```

Kafka A에 들어온 메시지가 Kafka B에는 존재하지 않는다.
따라서 양쪽 Kafka를 동일하게 유지해야 한다.

```text
Kafka A                  Kafka B

A ─────────────────────▶ A
B ─────────────────────▶ B
C ─────────────────────▶ C

      Data Mirroring
```

토스증권은 실시간 양방향 데이터 미러링을 통해 양쪽 Kafka가 모든 메시지를 가지도록 구성했다.

```text
               Mirroring
        ┌─────────────────────┐
        ▼                     │
     Kafka A               Kafka B
        │                     ▲
        └─────────────────────┘
               Mirroring
```

하지만 이것만으로는 부족하다.
Consumer의 Offset까지 해결해야 한다.

## 7. 토픽 이름은 양쪽에서 동일하게

두 Kafka 클러스터를 직접 사용하는 서비스 개발자 입장에서는 이중화 구조를 최대한 몰라도 되는 것이 좋다.
예를 들어 다음처럼 데이터센터마다 토픽 이름이 다르다면

```text
DC A

dc-a.payment-created

DC B

dc-b.payment-created
```

서비스가 어느 데이터센터에서 실행되는지 코드가 알아야 한다.
토스증권은 이를 피하기 위해 두 Kafka에서 동일한 Topic 이름을 사용했다.

```text
DC A Kafka

payment-created

DC B Kafka

payment-created
```

따라서 서비스 개발자는 자신이 어느 데이터센터에 배포됐는지 신경 쓸 필요가 없다.

## 8. Producer는 Active-Active

Producer는 자신이 실행되고 있는 데이터센터의 Kafka에 메시지를 보낸다.
이를 위해 Split DNS를 사용했다.

```text
              Producer

          kafka.toss.internal
                  │
             Split DNS
          ┌───────┴───────┐

      DC A 배포          DC B 배포
          │                 │
          ▼                 ▼
       Kafka A           Kafka B
```

DC A에 배포된 Producer라면 Kafka A로,
DC B에 배포된 Producer라면 Kafka B로 접근한다.
이를 통해 지역성(Locality) 을 확보할 수 있다.

```text
DC A

Service ──▶ Kafka A

       짧은 경로
```

Producer가 메시지를 Local Kafka에 기록하면 미러링 시스템이 반대편 Kafka에도 전달한다.

```text
Producer A
    │
    ▼
 Kafka A
    │
    │ Mirroring
    ▼
 Kafka B

Producer B
    │
    ▼
 Kafka B
    │
    │ Mirroring
    ▼
 Kafka A
```

결과적으로 두 Kafka 모두 전체 데이터를 보유한다.

## 9. 데이터센터 A가 장애 나면?

정상 상황에서는

```text
            User Traffic
            50%      50%
             │        │
             ▼        ▼

            DC A     DC B
             │        │
             ▼        ▼
          Kafka A   Kafka B
```

처럼 양쪽 데이터센터를 모두 사용한다.
DC A 전체에 장애가 발생하면

```text
DC A
  X

             User Traffic
                  │
                 100%
                  │
                  ▼
                 DC B
                  │
                  ▼
               Kafka B
```

최상단 트래픽만 정상 데이터센터로 옮기면 된다.
Kafka B에는 미러링을 통해 기존 메시지 역시 존재하기 때문에 계속 서비스를 제공할 수 있다.

## 10. 그런데 Consumer는 Active-Active로 만들면 안 된다

Producer는 양쪽 Kafka에 자유롭게 데이터를 기록해도 된다.
하지만 Consumer는 다르다.
양쪽 Kafka가 동일한 메시지를 가지고 있기 때문이다.

```text
Kafka A

Message A
Message B
Message C

Kafka B

Message A
Message B
Message C
```

Consumer가 양쪽 Kafka를 동시에 읽으면

```text
Kafka A ──▶ Consumer
              │
Kafka B ──▶ Consumer

Message A 두 번 처리
Message B 두 번 처리
```

같은 데이터를 중복 소비할 수 있다.
따라서 Consumer는 Active-Standby 구조로 운영한다.

```text
Kafka A                    Kafka B
   │                          │
   ▼                          X
Consumer                  Standby

현재 Active
```

한쪽 Kafka에서만 메시지를 소비한다.

## 11. Consumer 전환에는 GSLB를 사용

Consumer가 어느 Kafka를 바라볼지는 GSLB(Global Server Load Balancing) DNS를 이용해 제어했다.

```text
Consumer
    │
    ▼
kafka-consumer.toss
    │
   GSLB
    │
    ▼
Kafka A
```

데이터센터 A에 장애가 발생하면

```text
Before

Consumer
   │
   ▼
Kafka A

After Failover

Consumer
   │
   ▼
Kafka B
```

DNS 설정을 변경해 Consumer가 다른 Kafka를 바라보도록 한다.
이렇게 Producer와 Consumer 모두 서비스 개발자가 데이터센터 이중화 구조를 직접 처리하지 않도록 했다.

## 12. 하지만 Consumer에는 또 하나의 문제가 있다

Consumer는 자신이 어디까지 데이터를 읽었는지 Consumer Group Offset으로 저장한다.

```text
Partition

0 1 2 3 4 5 6 7 8 9 10
                  ▲
               Consumer

Offset = 8
```

Consumer가 재시작하면 Offset을 확인한 뒤 이어서 읽는다.
문제는 Kafka A와 Kafka B의 Offset이 서로 독립적이라는 것이다.
같은 메시지가 있다고 해도 Offset은 완전히 다를 수 있다.

```text
Kafka A

Offset 10
Message = "증권 최고"

Kafka B

Offset 9,959,273
Message = "증권 최고"
```

두 Kafka에서 Offset `10`과 `9,959,273`은 아무런 수치적 관계가 없다.

## 13. 그래서 Offset Sync가 필요하다

Consumer가 Kafka A에서

```text
Message C까지 읽음
```

상태라고 가정해 보자.

```text
Kafka A

A → B → C → D → E
        ▲
      Consumer
```

장애가 발생해 Kafka B로 이동했다.

```text
Kafka B

A → B → C → D → E
```

Kafka B에서 `C`가 몇 번 Offset인지 알아야 `D`부터 이어서 읽을 수 있다.

```text
Kafka A

C = Offset 30

       Offset Sync

Kafka B

C = Offset 98423
```

즉,

```text
Kafka A Offset
       │
       ▼
어떤 메시지까지 읽었는지 확인
       │
       ▼
Kafka B Offset으로 변환
       │
       ▼
Standby Cluster에 저장
```

하는 과정이 필요하다.
이를 담당하는 것이 Consumer Group Offset Sync다.

## 14. 전체 구조를 한 번에 보면

토스증권의 구성을 단순화하면 다음과 같다.

Producer는

```text
Active - Active
```

Consumer는

```text
Active - Standby
```

로 운영한다.
그리고 Kafka 사이에는

```text
┌───────────────────────────┐
│                           │
│      Data Mirroring       │
│                           │
└───────────────────────────┘

┌───────────────────────────┐
│                           │
│ Consumer Group Offset Sync│
│                           │
└───────────────────────────┘
```

두 가지 동기화가 필요하다.

## 15. 장애 상황의 전체 흐름

평상시에는

```text
          Producer
        /          \
       ▼            ▼
   Kafka A       Kafka B
       ╲            ╱
        ╲          ╱
         ╲        ╱
          Mirroring
Consumer
   │
   └──▶ Kafka A
```

처럼 동작한다.
만약 DC A가 장애 나면

```text
DC A
 X

       User Traffic
            │
            ▼
           DC B
            │
            ▼
         Kafka B
            ▲
            │
      Consumer 이동
            │
       Offset Sync된
       위치부터 소비
```

서비스 트래픽과 Consumer를 정상 데이터센터로 전환한다.

## 16. 이 구조에서 중요한 점

처음 문제는 단순했다.

```text
데이터센터가 장애 나더라도
서비스가 계속 동작해야 한다.
```

서비스 서버는 이미 Active-Active였지만 Kafka가 단일 데이터센터에 의존하면 완전한 이중화라고 볼 수 없었다.
그래서 Kafka도 이중화해야 했다.

```text
Kafka 이중화 필요
       │
       ▼
아키텍처 비교
       │
 ┌─────┴─────┐
 ▼           ▼
Active      Stretched
Active      Cluster
 │            │
 │       Latency 문제
 │       Split-brain 위험
 │            │
 └───── 선택 ◀┘
       │
       ▼
Active-Active
```

하지만 Active-Active를 선택하면서 새로운 문제가 생겼다.

```text
독립된 Kafka 두 개
       │
       ├── 데이터가 다름
       │
       └── Offset이 다름
               │
               ▼

        Data Mirroring
              +
          Offset Sync
```

결국 Kafka 데이터센터 이중화의 핵심은 Kafka를 두 개 설치하는 것이 아니다.
두 개의 독립적인 Kafka 클러스터를 사용하면서도 서비스 입장에서는 하나의 Kafka처럼 사용할 수 있도록 만드는 것에 있다.

```text
서비스 개발자 관점

           Kafka
             ▲
             │
       하나처럼 사용

실제 내부 구조

Kafka A ◀──────▶ Kafka B
        Mirror
          +
      Offset Sync
```

그리고 이를 위해 토스증권은 Producer에는 Split DNS를 이용한 Active-Active, Consumer에는 GSLB를 이용한 Active-Standby 정책을 적용했다.
결과적으로 데이터센터 장애 상황에서도 서비스 개발자가 Kafka 이중화의 내부 구조를 크게 신경 쓰지 않고 정상 데이터센터로 전환할 수 있는 구조를 만든 것이다.
