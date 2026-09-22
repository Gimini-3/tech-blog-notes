# 토스증권 Apache Kafka 데이터센터 이중화 구성 #3: Offset Sync

- 읽은 날짜: 2026-09-19
- 회사/작성 주체: 토스증권
- 원문: [토스증권 Apache Kafka 데이터센터 이중화 구성 #3: Offset Sync](https://toss.tech/article/kafka-distribution-3)

---

1편에서는 두 데이터센터에 Kafka를 Active-Active로 구성했고,
2편에서는 두 Kafka의 데이터를 양방향 미러링해서 동일하게 유지했다.
이제 마지막 문제가 남는다.

```text
Kafka A                     Kafka B

message A                   message A
message B                   message B
message C                   message C
```

데이터는 양쪽에 모두 존재한다.
그런데 Kafka A를 읽던 Consumer가 장애 때문에 Kafka B로 이동한다면

> Kafka B에서는 어디서부터 읽어야 할까?
> 이 문제를 해결하는 것이 Consumer Offset Sync다.

## 1. Offset이란?

Kafka의 각 Partition에 저장되는 메시지에는 순서대로 Offset이 붙는다.

```text
Partition 0

Offset
  0      1      2      3      4
  │      │      │      │      │
  ▼      ▼      ▼      ▼      ▼

 [A]    [B]    [C]    [D]    [E]
                       ▲
                    Consumer
```

Consumer는 자신이 어디까지 읽었는지 Offset을 저장한다.
예를 들어 Offset 3까지 처리했다면 다음에는 Offset 4부터 읽으면 된다.

```text
저장된 Offset = 3

다시 시작

[A] [B] [C] [D] [E]
             ▲    ▲
           완료   여기부터
```

이 Offset 정보가 사라지면 설정에 따라 처음부터 읽거나(`earliest`), 최신 메시지부터 읽게 된다(`latest`).
따라서 Kafka 이중화 환경에서는 Consumer Offset까지 다른 Kafka에 맞춰놓는 것이 중요하다.

## 2. 왜 단순히 Offset 숫자를 복사하면 안 될까?

같은 메시지가 양쪽 Kafka에 존재해도 Offset은 서로 다를 수 있다.

```text
Kafka A

Offset 100
Message C

Kafka B

Offset 9281
Message C
```

`100 → 9281`처럼 메시지의 위치가 다르다.
따라서

```text
Kafka A Offset 100

        ↓ 변환 필요

Kafka B Offset 9281
```

이 필요하다.
Offset 숫자를 그대로 복사하면 완전히 다른 메시지를 가리킬 수 있다.

## 3. 기존 방법 ① MirrorMaker2

Kafka의 MirrorMaker2에서는

```text
MirrorSourceConnector
        │
        ├─ Data Mirror
        │
        └─ Offset 관계 저장

MirrorCheckpointConnector
        │
        └─ Consumer Offset 변환
```

방식으로 Offset Sync를 수행한다.
데이터를 미러링하면서 Source와 Target의 Offset 관계를 저장한다.

```text
Kafka A                  Kafka B

Offset 5000
     │
     │ Mirror
     ▼
                         Offset 10000
```

이 관계를 다음처럼 저장한다.

```text
upstreamOffset   = 5000
downstreamOffset = 10000
```

이 정보가 저장되는 곳이 OffsetSync Topic이다.

## 4. MM2는 Offset을 어떻게 변환할까?

예를 들어 다음 Offset 관계가 저장돼 있다고 하자.

```text
Source        Target

100    ───▶   200
```

Consumer의 Source Offset이 정확히 100이라면

```text
100 → 200
```

으로 변환하면 된다.
하지만 Source Consumer Offset이 199라면 문제가 생긴다.
우리는 직관적으로

```text
Source 199

     ↓

Target 299 근처?
```

라고 생각할 수 있다.
하지만 실제로 Source와 Target 사이에서 모든 메시지가 반드시 1:1로 저장된다고 보장할 수 없다.
그래서 MM2는 매우 보수적으로 처리한다.

```text
마지막으로 확실한 관계

100 → 200

Source Consumer Offset = 199

        ↓

Target Offset = 201
```

정확한 위치를 추측해 앞으로 보내지 않고, 확실히 안전한 위치 바로 다음부터 다시 읽는다.
그 결과

```text
중복 발생 가능 ⭕

데이터 유실 ❌
```

이 된다.

## 5. 왜 이렇게 보수적으로 처리할까?

Offset을 너무 앞으로 잡으면 아직 읽지 않은 메시지를 건너뛸 수 있다.

```text
실제 위치

200 201 202 203 204
         ▲
      읽어야 함

잘못된 Offset Sync

                 ▲
                205

→ 202~204 유실
```

반대로 Offset을 뒤쪽으로 잡으면

```text
200 201 202 203 204
 ▲
다시 읽기

→ 중복은 발생하지만 유실은 없음
```

이 된다.
토스증권 역시 여기서 중복보다 유실을 훨씬 위험한 문제로 봤다.

## 6. MM2 방식의 한계

토스증권 환경에서는 세 가지 문제가 있었다.

```text
① OffsetSync 정보가 일정 간격으로만 저장
   → 정확한 Offset 변환이 어려움

② 동일 Topic 이름으로
   양방향 Mirror하는 구조와 맞지 않음

③ Data Mirror가 없는 두 Cluster 사이에서는
   Offset 관계 자체가 없음
```

특히 토스증권은 양쪽 Kafka에서 같은 Topic 이름을 유지하는 Active-Active 구조를 사용하고 있었기 때문에 그대로 적용하기 어려웠다.

## 7. 기존 방법 ② Confluent Replicator

Confluent는 다른 방식을 사용한다.
핵심은 Timestamp다.
Consumer가 Offset을 Commit할 때

```text
Offset
+
Timestamp
```

를 같이 저장한다.
그리고 Target Kafka에서 같은 시간대의 메시지를 찾는다.

```text
Source Kafka

Offset 300
Timestamp 10:30:15

        │
        ▼

Target Kafka

offsetsForTimes(10:30:15)

        │
        ▼

Offset 8241
```

즉,

```text
Source Offset
    │
    ▼
Timestamp 확인
    │
    ▼
Target에서 Timestamp 검색
    │
    ▼
Target Offset 발견
```

방식이다.
하지만 이 방식은 Confluent 기능에 의존하며, Apache Kafka 환경에서 그대로 사용하려면 Consumer 자체에 추가 기능을 구현하고 배포해야 하는 문제가 있었다.

## 8. 토스증권의 요구사항

토스증권은 상황을 두 가지로 나눴다.

### 재난 상황

화재나 데이터센터 장애처럼 갑자기 Kafka를 다른 데이터센터로 옮겨야 하는 상황이다.

```text
중복 최소화

유실은 절대 금지
```

### 계획된 작업

장비 점검이나 데이터센터 작업처럼 미리 준비할 수 있는 상황이다.

```text
중복 ❌

유실 ❌
```

특히 유실은 서비스에서 복구하기 어렵기 때문에 어떤 경우에도 허용하지 않았다.
중복은 서비스가 멱등성을 가지고 있다면 어느 정도 대응할 수 있지만, 읽지 않은 메시지를 건너뛰는 유실은 Kafka 단계에서 막아야 한다는 판단이었다.

## 9. 그런데 Topic 구조가 두 종류였다

토스증권에서는 Topic에 데이터가 들어오는 방식이 크게 두 가지였다.

### 100:0 구조

한쪽 Kafka에서 모든 데이터가 발생한다.

```text
Kafka A                    Kafka B

100% Produce
    │
    │ Mirror
    └────────────────────▶ 100%
```

양쪽에서 메시지 순서가 동일하다.

### 50:50 구조

두 데이터센터에서 각각 메시지가 발생한다.

```text
Kafka A                    Kafka B

50% Produce                50% Produce
     │                         │
     └─────── Mirror ──────────┘
```

미러링이 끝나면 양쪽 Kafka 모두 100% 데이터를 갖는다.
하지만 메시지 순서는 서로 달라질 수 있다.
그래서 두 구조에서 Offset Sync 방식도 달라진다.

## 10. 전략 ① 100:0에서는 Timestamp를 사용한다

100:0 구조에서는 원본 메시지가 한쪽에서만 발생한다.

```text
Kafka A

A → B → C → D

      Mirror

Kafka B

A → B → C → D
```

Partition 수도 같고 같은 Partition으로 미러링되기 때문에 메시지 순서를 유지할 수 있다.
그래서 Timestamp를 이용한다.

```text
Source Consumer Offset
          │
          ▼
해당 Record Consume
          │
          ▼
Timestamp 확인
          │
          ▼
Target Kafka
offsetsForTimes()
          │
          ▼
Target Offset 찾기
          │
          ▼
Consumer Offset Commit
```

## 11. offsetsForTimes의 함정

문제는 Kafka의 `offsetsForTimes()`가

> 해당 Timestamp 이상을 가진 첫 번째 Offset
> 을 반환한다는 것이다.
> Timestamp가 항상 순서대로 증가한다면 문제가 없다.

```text
Offset      Timestamp

100         10:00:01
101         10:00:02
102         10:00:03
103         10:00:04
```

하지만 실제 Kafka에서는 Timestamp가 항상 증가한다고 보장할 수 없다.

```text
Offset      Timestamp

100         10:00:01
101         10:00:03
102         10:00:02
103         10:00:04
```

10:00:02를 찾더라도 더 앞쪽의 Offset 101이 먼저 반환될 수 있다.

```text
찾고 싶던 위치

Offset 102

하지만 결과

Offset 101
```

Consumer가 이전 위치에서 다시 시작하게 되므로 중복이 발생한다.
다만 더 이전 위치를 선택하는 것이기 때문에 이 구조에서는 유실은 발생하지 않는다.

## 12. 중복 가능성까지 모니터링한다

토스증권은 Timestamp 순서가 꼬인 상황 자체를 Metric으로 기록했다.

```text
Offset Sync
    │
    ▼
Timestamp 검사
    │
    ▼
순서 이상 발견
    │
    ▼
Metric 기록
    │
    ▼
Dashboard
```

계획된 작업 전에 해당 Topic에서 중복이 발생할 가능성이 있는지 미리 확인할 수 있게 만들었다.
즉 Timestamp 방식의 한계를 없애기보다는
문제가 발생할 가능성을 관측 가능하게 만든 것이다.

## 13. 그런데 50:50에서는 Timestamp 방식이 위험하다

50:50 구조에서는 양쪽 Kafka에서 동시에 메시지가 발생한다.
예를 들어

```text
Kafka A에서 생성

1 → 2 → 3 → 4

Kafka B에서 생성

A → B → C
```

각 그룹 내부의 순서는 유지된다.
하지만 미러링 타이밍에 따라 전체 순서는 달라질 수 있다.

```text
Kafka A

1 → A → 2 → B → 3 → C → 4

Kafka B

A → 1 → B → 2 → C → 3 → 4
```

각 Kafka가 가지고 있는 데이터는 같지만 순서는 다르다.

## 14. 여기서 Timestamp Sync를 하면 유실될 수 있다

Kafka A의 Consumer가 `3`까지 읽었다고 가정해 보자.

```text
Kafka A

1 → A → 2 → B → 3 → C → 4
                  ▲
                Consumer
```

그런데 Kafka B에서 `3`의 위치를 찾아 Offset을 Commit하면

```text
Kafka B

A → 1 → B → 2 → C → 3 → 4
                      ▲
                 Offset Commit
```

Consumer가 Kafka B로 이동한 뒤 `3` 다음부터 읽는다.
그런데 Kafka A에서는 아직 `C`를 읽지 않았다.

```text
C가 건너뛰어짐

→ 데이터 유실
```

즉 50:50에서는 Timestamp 방식이 실제 데이터 유실을 만들 수 있다.

## 15. 전략 ② 50:50에서는 Header를 사용한다

2편에서 미러링할 때 메시지 Header에 원본 정보를 넣었다.

```text
Header

IDC    = IDC2
Offset = 1234516
```

따라서 메시지를 보면

```text
이 메시지는 어디에서 왔는가?

원본 Kafka에서 Offset이 몇이었는가?
```

를 알 수 있다.
토스증권은 이 정보를 Offset Sync에 활용했다.

## 16. 왜 Mirror된 메시지만 사용하면 유실이 없을까?

50:50에서는 메시지 그룹의 순서가 서로 뒤집힐 수 있다.

```text
Kafka A

1 → 2 → 3 → A → B → C

Kafka B

A → B → C → 1 → 2 → 3
```

이 상황에서 잘못된 그룹을 기준으로 Offset을 잡으면 아직 읽지 않은 다른 그룹을 건너뛸 수 있다.
그래서 Target Kafka에서 미러링되어 Source로 들어온 메시지일 때만 Offset Sync한다.
Header를 보면 이 메시지가 상대 Cluster에서 미러링된 것인지 바로 알 수 있다.

```text
Consumer Offset Record
       │
       ▼
Header 존재?
       │
   ┌───┴────┐
   │        │
  YES       NO
   │        │
   ▼        ▼
Mirror된    Offset Sync
메시지      하지 않음
   │
   ▼
Header의 원본 Offset 사용
```

이렇게 하면 중복 가능성은 남지만 읽지 않은 메시지를 건너뛰는 유실은 막을 수 있다.

## 17. 그런데 Offset Sync 성공 확률이 낮아진다

50:50으로 트래픽이 들어온다면 대략 절반의 메시지만 상대 Cluster에서 미러링된 메시지다.

```text
전체 메시지

50% Local Produce
50% Mirrored
```

따라서 단순히 현재 Consumer Offset의 메시지만 보면 Header를 발견할 확률도 약 50%가 된다.
토스증권은 30초마다 Offset Sync를 시도했고, 추가로 현재 메시지 주변의 최신 Record N개를 읽어서 Header가 존재하는 메시지를 찾도록 개선했다.
그 결과 Offset Sync 성공 가능성을 약 65~70% 수준으로 높였다.

## 18. 계획된 작업에서는 중복도 없애야 한다

Header 방식은 유실은 막았지만 중복 가능성은 남아 있다.
계획된 작업에서는 이를 해결할 시간이 있다.
그래서 평소 50:50이던 Producer 트래픽을 잠시 0:100으로 변경한다.

```text
평소

DC A           DC B

50%             50%

작업 직전

DC A           DC B

0%             100%
```

잠시 기다리면 Source Kafka에 새로 들어오는 데이터가 모두 Target Kafka에서 미러링된 데이터가 된다.

```text
Target에서 Produce
       │
       ▼
Target Kafka
       │
       │ Mirror
       ▼
Source Kafka

→ Source의 최신 데이터가 전부
  Header를 가진 Mirror 데이터
```

이제 메시지 순서가 꼬이는 문제를 제거할 수 있다.
따라서 계획된 작업에서는

```text
유실 ❌
중복 ❌
```

상태를 만든 뒤 Consumer를 이동할 수 있다.

## 19. 데이터가 거의 없는 Topic은?

또 다른 예외도 있었다.
한 시간에 메시지가 하나만 발생한다고 가정해 보자.

```text
10:00 message
11:00 message
12:00 message
...
```

운이 나쁘면 몇 시간 동안 Header를 가진 메시지를 만나지 못할 수 있다.
그러면 Offset Sync도 오랫동안 수행되지 않는다.
토스증권은

```text
오랫동안 데이터 발생 X

Consumer Lag = 0

Data Mirror도 없음
```

인 Topic에 대해서는 Target Consumer Offset을 최신 상태로 유지하는 별도 보완 로직을 두었다.

## 20. 결국 두 가지 전략을 사용한다

전체를 정리하면 다음과 같다.

```text
                  Topic 구조
                     │
          ┌──────────┴──────────┐
          ▼                     ▼

        100:0                  50:50
          │                     │
          ▼                     ▼
 Timestamp 검색            Header 참조
          │                     │
          ▼                     ▼
offsetsForTimes()       Mirror된 메시지만
                         Offset Sync
```

100:0은 양쪽 메시지 순서가 동일하기 때문에 Timestamp를 사용할 수 있다.
50:50은 양쪽 메시지 순서가 달라질 수 있기 때문에 Header를 이용해 안전한 메시지만 기준으로 Offset을 맞춘다.

## 21. Offset Sync도 모니터링해야 한다

Offset Sync를 만들었다고 끝나는 것은 아니다.
정상적으로 계속 동작하고 있는지를 확인해야 한다.
토스증권은 크게 두 가지 방식으로 모니터링했다.

```text
Offset Sync Monitoring

       │
       ├─ Cluster 전체 그래프
       │
       └─ Topic / Consumer Dashboard
```

## 22. 정상적인 Offset Sync의 모양

Source Consumer는 계속 메시지를 읽기 때문에 Offset이 증가한다.

```text
Source Offset

100
120
140
160
180
```

Target은 일정 주기로 Offset Sync를 받는다.

```text
Target Offset

100
100
140
140
180
```

그래서 두 Consumer Offset의 차이는

```text
증가
  │
  │ Offset Sync
  ▼
감소
  │
  │
증가
  │
  ▼
감소
```

를 반복한다.
긴 Window에서 보면 증가량과 감소량이 서로 상쇄되어 0에 수렴해야 한다.

```text
정상

   /\      /\      /\
__/  \____/  \____/  \__

Window 변화량 합 ≈ 0
```

반대로 Offset Sync가 멈추면

```text
Source Offset ↑↑↑

Target Offset ───

차이 ↑↑↑↑↑
```

가 된다.

## 23. Dashboard에서 InSync를 확인한다

Topic과 Consumer 단위로도 상태를 확인한다.

```text
Topic
Consumer
Mirror 상태
Source Offset
Target Offset
Offset 차이
Offset Sync 상태
```

등을 한 화면에서 볼 수 있도록 했다.
그리고 Data Mirror와 Offset Sync까지 정상이라면 상태를 표시한다.

```text
Mirroring
     │
     ▼
Offset도 정확히 일치
     │
     ▼
   InSync
```

`InSync`는

> 이제 Consumer를 다른 Kafka로 옮겨도 된다
> 는 신호로 사용된다.

## 24. 1~3편 전체 구조

지금까지 세 편을 연결하면 다음과 같다.

```text
                    User Traffic
                         │
              ┌──────────┴──────────┐
              ▼                     ▼

            DC A                   DC B
              │                     │
              ▼                     ▼
           Kafka A ◀════════════▶ Kafka B
                  Data Mirroring
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
      Header 정보               DLQ / Retry
          │
          ▼
      Offset Sync
          │
      ┌───┴────┐
      ▼        ▼
   100:0      50:50
      │        │
 Timestamp   Header
      │        │
      └───┬────┘
          ▼
       InSync
          │
          ▼
 Consumer DC 전환
```

## 정리

Kafka Active-Active 구성에서 어려운 부분은 단순히 데이터를 두 Kafka에 복제하는 것이 아니다.
데이터는 같아도

```text
Kafka A                    Kafka B

같은 Message              같은 Message

다른 Offset               다른 Offset

다른 순서 가능            다른 순서 가능
```

이기 때문이다.
토스증권은 이 문제를 Topic 특성에 따라 나눠 해결했다.

```text
100:0
 │
 └─▶ Timestamp 기반 Offset Sync

50:50
 │
 └─▶ Header 기반 Offset Sync
```

그리고 가장 중요한 기준은 일관됐다.

```text
재난 상황

유실 ❌
중복 최소화

계획된 작업

유실 ❌
중복 ❌
```

결국 1~3편의 흐름은 이렇게 정리할 수 있다.

```text
① Kafka 자체를 어떻게 이중화할까?
          │
          ▼
     Active-Active

② 두 Kafka의 데이터를
   어떻게 동일하게 만들까?
          │
          ▼
   Data Mirroring

③ Consumer가 다른 Kafka로
   이동하면 어디서부터 읽을까?
          │
          ▼
      Offset Sync
```

즉, 두 개의 독립된 Kafka를 서비스 입장에서는 하나의 Kafka처럼 사용할 수 있도록 만드는 과정이 토스증권 Kafka 데이터센터 이중화의 핵심이다.
