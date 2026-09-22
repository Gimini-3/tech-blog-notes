# Kafka Rebalancing과 메시지 처리 병목 개선

- 읽은 날짜: 2026-08-12
- 회사/작성 주체: 넥스트리
- 원문: [Kafka Rebalancing과 메시지 처리 병목 개선](https://www.nextree.io/kafka-rebalancinggwa-mesiji-ceori-byeongmog-gaeseon/)

---

## 1. 문제 상황

Kafka Consumer를 운영하던 중 메시지가 정상적으로 처리되지 않고 Consumer Group의 Rebalancing이 약 10분 간격으로 반복되는 현상이 발생했다.
로그와 Consumer 설정을 확인한 결과 다음과 같은 상황이었다.

```text
max.poll.interval.ms = 10분
메시지 1건 처리 시간 = 10분 이상
```

즉, Kafka에서 메시지를 가져오는 것 자체가 문제가 아니라 Consumer가 가져온 메시지를 처리하는 데 너무 오랜 시간이 걸리는 것이 문제였다.
전체 흐름을 먼저 보면 다음과 같다.

```text
Kafka 메시지 Consume
        ↓
메시지 처리 시작
        ↓
약 46,000건의 데이터 생성
        ↓
MSSQL 저장
        ↓
DB 저장 시간이 길어짐
        ↓
다음 poll() 호출 지연
        ↓
max.poll.interval.ms 초과
        ↓
Consumer Group Rebalancing
```

문제를 이해하려면 먼저 Kafka의 Consumer 구조와 Rebalancing을 알아야 한다.

## 2. Kafka 기본 구조

Kafka는 대량의 이벤트나 메시지를 여러 시스템 사이에서 전달하기 위한 분산 메시징 플랫폼이다.
기본적인 구조는 다음과 같다.

```text
Producer
   │
   │ 메시지 전송
   ▼
Kafka
   │
   │ 메시지 조회
   ▼
Consumer
   │
   ▼
비즈니스 로직 처리
```

예를 들어 주문 서비스에서 주문이 생성되었을 때 다음처럼 사용할 수 있다.

```text
주문 서버
   │
   │ OrderCreated 이벤트
   ▼
Kafka Topic
   │
   ├──────────────→ 결제 Consumer
   │
   └──────────────→ 알림 Consumer
```

Producer는 메시지를 Kafka에 보내고 Consumer는 필요한 메시지를 읽어 처리한다.

## 3. Topic과 Partition

Kafka에서는 메시지를 Topic이라는 단위로 구분해서 저장한다.
예를 들어 다음과 같은 Topic을 만들 수 있다.

```text
order-created
payment-completed
user-created
```

그리고 하나의 Topic은 여러 개의 Partition으로 나눌 수 있다.

```text
order-created Topic

┌───────────────┐
│ Partition 0   │
├───────────────┤
│ Partition 1   │
├───────────────┤
│ Partition 2   │
└───────────────┘
```

Partition은 Kafka에서 메시지가 실제로 저장되는 단위이며, Partition을 여러 개 두면 메시지를 병렬로 처리할 수 있다.
예를 들어 Partition이 하나뿐이라면 하나의 Consumer Group에서는 실질적으로 한 Consumer만 해당 Partition을 처리한다.

```text
Partition 0
    ↓
Consumer A
```

반면 Partition이 4개라면 여러 Consumer가 나눠 처리할 수 있다.

```text
Partition 0 ──→ Consumer A
Partition 1 ──→ Consumer A

Partition 2 ──→ Consumer B
Partition 3 ──→ Consumer B
```

따라서 Partition 수는 Kafka의 병렬 처리 수준과 관련이 있다.

## 4. Consumer Group

여러 Consumer를 하나의 그룹으로 묶은 것을 Consumer Group이라고 한다.
같은 Consumer Group에 속한 Consumer들은 하나의 Topic에 있는 Partition을 나눠서 처리한다.

```text
             Consumer Group

Partition 0 ─────→ Consumer A
Partition 1 ─────→ Consumer A
Partition 2 ─────→ Consumer B
Partition 3 ─────→ Consumer B
```

여기서 중요한 특징이 있다.

> 같은 Consumer Group 안에서는 하나의 Partition을 동시에 하나의 Consumer가 담당한다.
> Consumer를 추가하면 Kafka는 Partition을 다시 나눠줄 수 있다.

```text
기존

P0 ──→ Consumer A
P1 ──→ Consumer A
P2 ──→ Consumer B
P3 ──→ Consumer B

Consumer C 추가

P0 ──→ Consumer A
P1 ──→ Consumer B
P2 ──→ Consumer C
P3 ──→ Consumer A
```

이처럼 Consumer들에게 Partition을 다시 배분하는 과정이 Rebalancing이다.

## 5. Rebalancing이란?

Rebalancing은 Consumer Group에 속한 Consumer와 Partition의 관계를 다시 조정하는 과정이다.
예를 들어 다음과 같은 상황에서 발생할 수 있다.

- 새로운 Consumer가 Consumer Group에 들어온 경우
- 기존 Consumer가 종료된 경우
- Consumer가 정상적으로 동작하지 않는다고 판단된 경우
- Topic의 Partition 구성이 변경된 경우
  구조는 다음과 같다.

```text
Consumer A
Consumer B
     │
     ▼
Consumer C 추가
     │
     ▼
Rebalancing
     │
     ▼
Partition 다시 할당
```

Rebalancing 자체가 비정상적인 동작은 아니다.
Consumer가 추가되거나 제거되면 Partition을 다시 분배해야 하기 때문에 Kafka에서 필요한 기능이다.
문제는 Rebalancing이 지속적으로 반복되는 경우다.

```text
메시지 처리
   ↓
Rebalancing
   ↓
Partition 재할당
   ↓
메시지 처리
   ↓
Rebalancing
   ↓
Partition 재할당
   ↓
...
```

이러한 상황이 반복되면 안정적으로 메시지를 처리하기 어려워진다.
따라서 Rebalancing이 지나치게 자주 발생한다면 왜 Consumer가 Consumer Group에서 정상 상태를 유지하지 못하고 있는지 확인해야 한다.

## 6. Consumer와 poll()

Kafka Consumer는 `poll()`을 호출하여 Kafka Broker에서 메시지를 가져온다.
개념적으로 보면 다음 동작이 반복된다.

```text
poll()
  ↓
메시지 가져오기
  ↓
메시지 처리
  ↓
DB 저장
  ↓
처리 완료
  ↓
다음 poll()
```

코드로 단순화하면 다음과 비슷하다.

```java
while (true) {

    ConsumerRecords<String, String> records =
        consumer.poll(Duration.ofSeconds(1));

    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
}
```

`process()`가 빠르게 끝난다면 큰 문제가 없다.

```text
poll()
 ↓
1초 처리
 ↓
poll()
 ↓
2초 처리
 ↓
poll()
```

하지만 메시지 처리 시간이 지나치게 길어지면 다음 `poll()` 호출도 함께 늦어진다.

```text
poll()
 ↓
메시지 처리
 ↓
DB 처리
 ↓
DB 처리
 ↓
DB 처리
 ↓
10분 경과
 ↓
아직 처리 중
 ↓
다음 poll() 호출 못함
```

이번 문제에서 중요한 부분이 바로 여기에 있었다.

## 7. max.poll.interval.ms

Kafka Consumer에는 `max.poll.interval.ms`라는 설정이 있다.
쉽게 말하면

> 이전 `poll()` 이후 다음 `poll()`을 호출하기까지 허용되는 최대 시간
> 이다.
> 문제가 발생했던 환경에서는 다음과 같이 설정되어 있었다.

```text
max.poll.interval.ms=600000
```

600,000ms이므로 10분이다.
그런데 실제 메시지 처리 시간도 10분 이상이었다.

```text
0분

poll()
 │
 ▼
메시지 처리 시작
 │
 ├── 데이터 가공
 ├── DB 저장
 ├── DB 저장
 ├── DB 저장
 │
 ▼

10분

max.poll.interval.ms 도달
 │
 ▼
아직 메시지 처리 중
 │
 ▼
다음 poll() 호출하지 못함
```

결국 Consumer가 `max.poll.interval.ms` 안에 다음 `poll()`을 호출하지 못하면서 정상적인 Consumer로 유지되지 못하고 Consumer Group Rebalancing으로 이어질 수 있다.

## 8. Heartbeat와 max.poll.interval.ms의 차이

Kafka Consumer를 공부할 때 `Heartbeat`, `session.timeout.ms`, `max.poll.interval.ms`가 헷갈릴 수 있다.
간단하게 구분하면 다음과 같다.
| 개념 | 확인하는 것 |
| --- | --- |
| Heartbeat | Consumer가 Consumer Group과 계속 통신하고 있는지 |
| `session.timeout.ms` | Consumer가 일정 시간 동안 응답하지 않는지 |
| `max.poll.interval.ms` | 애플리케이션이 적절한 시간 안에 다음 `poll()`을 호출하는지 |
즉 Consumer 프로세스 자체가 살아 있어도 메시지 하나를 처리하는 데 지나치게 오래 걸릴 수 있다.

```text
Consumer 프로세스

살아 있음
Heartbeat 정상
       │
       ▼
poll()
       │
       ▼
15분 동안 메시지 처리
       │
       ▼
poll()
```

만약

```text
max.poll.interval.ms=600000
```

이라면 허용 시간은 10분이므로 문제가 될 수 있다.
따라서 다음처럼 구분해서 이해하면 쉽다.

```text
Heartbeat / session.timeout.ms

"Consumer가 살아 있는가?"

max.poll.interval.ms

"Consumer가 메시지를 정상적인 속도로 처리하고 있는가?"
```

## 9. 문제 원인 파악

실제 로그에서는 약 10분 간격으로 Rebalancing이 반복되고 있었다.
Consumer 설정을 확인하면

```text
max.poll.interval.ms=600000
```

역시 10분이었다.
여기에 실제 메시지 처리 시간을 확인하니 하나의 메시지를 처리하는 데 10분 이상이 소요되고 있었다.
따라서 다음과 같이 연결할 수 있었다.

```text
메시지 Consume
      ↓
메시지 처리 시작
      ↓
처리 시간이 10분 이상 소요
      ↓
다음 poll() 호출 지연
      ↓
max.poll.interval.ms 10분 초과
      ↓
Consumer Group에서 정상적으로 유지되지 못함
      ↓
Rebalancing
```

즉 처음 보이는 현상은

```text
Kafka 메시지가 정상적으로 처리되지 않는다.
```

였지만 실제 원인은

```text
Consumer의 메시지 처리 시간이 너무 길다.
```

에 가까웠다.

## 10. 왜 메시지 하나를 처리하는 데 10분 이상 걸렸을까?

여기서 `max.poll.interval.ms`만 늘리면 문제의 원인을 해결한 것이 아니다.
다음으로 확인해야 할 것은

> 왜 메시지 하나를 처리하는 데 10분이나 걸리는가?
> 이다.
> Consumer 내부의 메시지 처리 과정을 확인한 결과 하나의 Kafka 메시지를 처리하면서 약 46,000건의 데이터를 MSSQL에 저장하고 있었다.

```text
Kafka
  │
  ▼
Consumer
  │
  ▼
메시지 1건
  │
  ▼
데이터 가공
  │
  ▼
약 46,000건 데이터 생성
  │
  ▼
MSSQL 저장
  │
  ▼
처리 완료
  │
  ▼
다음 poll()
```

Kafka 입장에서는 메시지 하나를 Consumer에게 전달했지만 애플리케이션 내부에서는 해당 메시지를 처리하기 위해 수만 건의 DB 저장 작업이 수행되고 있었다.
따라서 실제 병목은 Kafka 메시지 수신이 아니라 DB 저장 구간에 있었다.

## 11. 문제를 단계적으로 좁혀보기

이번 문제는 다음과 같이 한 단계씩 원인을 따라가면 이해하기 쉽다.

```text
Rebalancing이 반복된다.
        ↓
왜?

max.poll.interval.ms를 초과했다.
        ↓
왜?

다음 poll() 호출이 너무 늦었다.
        ↓
왜?

메시지 처리 시간이 너무 길었다.
        ↓
왜?

하나의 메시지를 처리하면서
약 46,000건을 DB에 저장하고 있었다.
        ↓
왜 오래 걸렸나?

DB 저장 방식에 병목이 있었다.
```

결국

```text
Kafka Rebalancing
```

은 가장 바깥에서 보이는 현상이고,
실제로 개선해야 하는 부분은

```text
대량 DB 저장 성능
```

이었다.

## 12. 1차 조치: max.poll.interval.ms 증가

운영 환경에서는 근본적인 원인을 분석하고 코드를 수정하는 동안에도 계속 Rebalancing이 발생할 수 있다.
따라서 먼저 `max.poll.interval.ms`를 10분에서 30분으로 증가시켰다.

```text
# 기존
max.poll.interval.ms=600000

# 변경
max.poll.interval.ms=1800000
```

이렇게 하면 현재 메시지 처리 시간이 10분을 조금 넘더라도 Consumer가 처리할 수 있는 시간을 확보할 수 있다.

```text
기존

메시지 처리    12분
허용 시간      10분

12분 > 10분

→ 문제 발생

변경 후

메시지 처리    12분
허용 시간      30분

12분 < 30분

→ 처리 가능
```

하지만 이것은 운영 장애를 우선 완화하기 위한 조치에 가깝다.
데이터가 증가하면서 처리 시간도 증가할 수 있기 때문이다.

```text
12분
 ↓
18분
 ↓
25분
 ↓
31분
```

처리 시간이 다시 30분을 넘어가면 동일한 문제가 발생한다.
따라서 해결 방법을 두 단계로 나눌 수 있다.

```text
1. 운영 안정화

max.poll.interval.ms
10분 → 30분

2. 근본적인 개선

DB 저장 시간을 줄여
메시지 처리 시간 자체를 감소
```

## 13. DB 저장 방식 개선

메시지 처리 과정에서 약 46,000건을 MSSQL에 저장하는 구간이 병목이었다.
기존 애플리케이션에서는 JPA를 사용하고 있었다.
대량 데이터를 저장할 때는 Hibernate Batch 기능을 적용하는 방법도 생각할 수 있다.
예를 들어 Hibernate에서는 Batch Size를 설정하여 여러 INSERT를 묶어서 처리할 수 있다.
하지만 전역적인 JPA Batch 설정을 변경하면 해당 Consumer 로직뿐만 아니라 Hibernate를 사용하는 다른 저장 로직에도 영향을 줄 가능성이 있다.
개선이 필요한 부분은 특정 Kafka 메시지를 처리하는 DB 저장 구간뿐이었다.

```text
전체 애플리케이션

JPA 저장 로직 A
JPA 저장 로직 B
JPA 저장 로직 C

Kafka Consumer
    │
    └── 46,000건 저장 ← 이 부분만 개선
```

따라서 기존 JPA 구조는 유지하면서 해당 저장 구간에만 `JdbcTemplate.batchUpdate()`를 적용했다.

## 14. Batch Insert란?

일반적인 INSERT 방식에서는 데이터를 여러 번 저장할 경우 DB와 반복적으로 작업하게 된다.
개념적으로 단순화하면 다음과 같다.

```text
데이터 1 → INSERT
데이터 2 → INSERT
데이터 3 → INSERT
데이터 4 → INSERT
...
데이터 46,000 → INSERT
```

이 과정에서 SQL 실행과 DB 통신 등의 작업이 반복되면서 대량 데이터 처리 시 비용이 커질 수 있다.
Batch Insert는 여러 데이터의 INSERT 작업을 일정 크기로 묶어서 처리하는 방식이다.
예를 들어 Batch Size가 1,000이라면 다음과 같은 형태가 된다.

```text
46,000건

↓ Batch Size = 1,000

1 ~ 1,000
    ↓
Batch 실행

1,001 ~ 2,000
    ↓
Batch 실행

2,001 ~ 3,000
    ↓
Batch 실행

...

45,001 ~ 46,000
    ↓
Batch 실행
```

여러 작업을 묶어서 DB에 전달하기 때문에 반복적인 DB 작업 비용을 줄일 수 있다.

## 15. JdbcTemplate.batchUpdate()

문제가 발생한 DB 저장 구간에는 다음과 같은 방식으로 `JdbcTemplate.batchUpdate()`를 사용했다.

```java
jdbcTemplate.batchUpdate(
    sql,
    data,
    batchSize,
    (PreparedStatement ps, T obj) -> {
        // 데이터 바인딩
    }
);
```

각 요소를 보면 다음과 같다.

### sql

실행할 SQL이다.

```sql
INSERT INTO TABLE_NAME (
    ID,
    COLUMN_A,
    COLUMN_B
)
VALUES (
    NEWID(),
    ?,
    ?
)
```

### data

저장할 데이터 목록이다.

```text
46,000건의 데이터
```

### batchSize

한 번의 Batch에서 처리할 데이터 개수다.

```text
예)

1,000개씩 Batch 처리
```

### PreparedStatement

각 데이터에 맞는 값을 SQL의 `?` 위치에 바인딩한다.
전체적인 흐름은 다음과 같다.

```text
List<데이터> 46,000건
        ↓
batchSize 기준으로 분할
        ↓
PreparedStatement에 값 바인딩
        ↓
Batch 실행
        ↓
DB 저장
```

이를 통해 기존 JPA를 사용하는 다른 저장 로직에는 영향을 주지 않으면서 병목이 발생한 특정 구간만 최적화할 수 있었다.

## 16. 왜 JPA 전체 설정을 변경하지 않았을까?

### `JdbcTemplate.batchUpdate()`와 Hibernate Batch 비교

둘 다 JDBC Batch를 이용해 여러 SQL을 묶어서 DB로 전달하지만, 어느 계층에서 배치를 다루는지가 가장 큰 차이다.
핵심만 먼저 정리하면 다음과 같다.
| 구분 | `JdbcTemplate.batchUpdate()` | Hibernate Batch |
| --- | --- | --- |
| 계층 | JDBC에 가까움 | JPA/Hibernate |
| 작성 방식 | SQL 직접 작성 | `repository.save()`, `persist()` 등 |
| Entity 관리 | 없음 | 영속성 컨텍스트 |
| Dirty Checking | 없음 | 지원 |
| 변경 감지 | 직접 SQL 필요 | 자동 |
| 생명주기 콜백 | 거의 없음 | `@PrePersist` 등 |
| 메모리 사용 | 상대적으로 적음 | Entity가 Persistence Context에 쌓일 수 있음 |
| 성능 | 보통 더 빠름 | 편하지만 오버헤드 있음 |
| 복잡한 도메인 로직 | 불편 | 편함 |
| 대량 데이터 처리 | 매우 적합 | 적절한 `flush/clear` 필요 |

#### 1. `JdbcTemplate.batchUpdate()`

예를 들어 10,000건을 INSERT한다고 가정해 보자.

```java
String sql =
    "INSERT INTO users(name, email) VALUES (?, ?)";

jdbcTemplate.batchUpdate(
    sql,
    users,
    1000,
    (ps, user) -> {
        ps.setString(1, user.getName());
        ps.setString(2, user.getEmail());
    }
);
```

Spring이 내부적으로 다음과 같이 처리한다고 볼 수 있다.

```text
PreparedStatement 생성

user1 -> addBatch()
user2 -> addBatch()
user3 -> addBatch()
...
user1000 -> addBatch()

executeBatch()

user1001 -> addBatch()
...
```

즉,

```text
Application
   ↓
JdbcTemplate
   ↓
PreparedStatement.addBatch()
   ↓
executeBatch()
   ↓
DB
```

여기에는 JPA의 영속성 컨텍스트가 없다.
따라서 Entity를 관리하지도 않고,

```text
Dirty Checking
1차 캐시
Entity 상태 관리
Cascade
연관관계 관리
```

같은 기능도 제공하지 않는다.
SQL을 빠르게 실행하는 데 집중한다.

#### 2. Hibernate Batch

Hibernate Batch는 JPA를 그대로 사용하면서 Hibernate가 JDBC Batch를 이용하도록 하는 방식이다.
설정 예시는 다음과 같다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 1000
```

코드에서는 다음과 같이 작성한다.

```java
for (User user : users) {
    entityManager.persist(user);
}
```

겉으로는 `persist()`를 반복하는 것처럼 보이지만 Hibernate 내부에서는 다음과 같이 처리된다.

```text
persist(user1)
persist(user2)
persist(user3)
...
        ↓
Persistence Context
        ↓
Hibernate ActionQueue
        ↓
JDBC Batch
        ↓
PreparedStatement.addBatch()
        ↓
executeBatch()
```

즉 Hibernate가 중간에서 SQL을 모아서 JDBC Batch로 만들어주는 것이다.

#### 3. 가장 중요한 차이: 영속성 컨텍스트

이 부분이 핵심이다.

##### JdbcTemplate

```text
User 객체
 ↓
SQL parameter 변환
 ↓
JDBC
 ↓
DB
```

이것으로 처리 과정이 끝난다.
반면 Hibernate는:

```text
User Entity
 ↓
Persistence Context 등록
 ↓
Entity 상태 관리
 ↓
Dirty Checking 준비
 ↓
ActionQueue
 ↓
SQL 생성
 ↓
JDBC Batch
 ↓
DB
```

이 과정이 있다.
그래서 Hibernate가 훨씬 편한 대신 그만큼 오버헤드가 존재한다.

#### 4. Hibernate Batch에서 `flush()` / `clear()`가 중요한 이유

예를 들어:

```java
for (int i = 0; i < 100_000; i++) {
    entityManager.persist(users.get(i));
}
```

이렇게 하면 10만 개 Entity가 영속성 컨텍스트에 계속 쌓일 수 있다.

```text
Persistence Context

1번 Entity
2번 Entity
3번 Entity
...
100,000번 Entity
```

메모리 사용량이 커지고 Dirty Checking 비용도 증가할 수 있다.
따라서 일반적으로 다음과 같이 처리한다.

```java
for (int i = 0; i < users.size(); i++) {

    entityManager.persist(users.get(i));

    if ((i + 1) % 1000 == 0) {
        entityManager.flush();
        entityManager.clear();
    }
}
```

이렇게 처리한다.

```text
1000개 persist
     ↓
flush()
     ↓
SQL 실행
     ↓
clear()
     ↓
Persistence Context 비움

다음 1000개
```

`JdbcTemplate.batchUpdate()`에서는 애초에 Persistence Context가 없기 때문에 이런 문제가 없다.

#### 5. 성능 차이가 나는 이유

100만 건을 INSERT한다고 가정하면 처리 계층은 다음과 같이 비교할 수 있다.

##### Hibernate

```text
Entity 생성
 ↓
Persistence Context 등록
 ↓
Entity 상태 관리
 ↓
Hibernate SQL 생성
 ↓
ActionQueue 관리
 ↓
JDBC Batch
 ↓
DB
```

##### JdbcTemplate

```text
객체
 ↓
PreparedStatement parameter 설정
 ↓
JDBC Batch
 ↓
DB
```

따라서 순수 대량 INSERT/UPDATE에서는 보통

```text
JdbcTemplate Batch
        ↓
Hibernate Batch
        ↓
일반 JPA save 반복
```

순으로 유리한 경우가 많다.
다만 실제 성능 차이는 DB, 네트워크, 인덱스, 트랜잭션 크기 등에 따라서 달라진다.

#### 6. Hibernate Batch의 장점

그렇다고 항상 `JdbcTemplate`이 좋은 것은 아니다.
예를 들어:

```java
Order order = new Order(...);

order.addOrderItem(item1);
order.addOrderItem(item2);

entityManager.persist(order);
```

Hibernate에서는

```java
@OneToMany(cascade = CascadeType.ALL)
private List<OrderItem> items;
```

같은 연관관계와 Cascade를 이용할 수 있다.
또한:

```java
@PrePersist
public void prePersist() {
    this.createdAt = LocalDateTime.now();
}
```

같은 Entity Lifecycle도 사용할 수 있다.
JdbcTemplate에서는 이 작업을 모두 직접 처리해야 한다.

```java
INSERT INTO orders ...
INSERT INTO order_items ...
INSERT INTO order_items ...
```

즉,
도메인 모델을 유지하면서 성능을 개선하고 싶다 → Hibernate Batch
가 적합하다.

#### 7. 특히 주의할 점: `IDENTITY`

Hibernate Batch를 사용할 때 자주 언급되는 중요한 문제다.
예를 들어:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

MySQL의 `AUTO_INCREMENT` 같은 방식이다.
Hibernate는 ID를 얻기 위해 INSERT를 즉시 실행해야 할 수 있다.

```text
persist(user1)
 ↓
INSERT
 ↓
DB가 ID 1 반환

persist(user2)
 ↓
INSERT
 ↓
DB가 ID 2 반환
```

그래서 INSERT Batch가 제대로 적용되지 않을 수 있다.
반대로 Sequence를 사용하면:

```java
@GeneratedValue(strategy = GenerationType.SEQUENCE)
```

ID를 INSERT 전에 확보할 수 있기 때문에 batching이 훨씬 수월하다.

```text
ID 확보

user1 = 1
user2 = 2
user3 = 3

       ↓

INSERT
INSERT
INSERT
       ↓
JDBC Batch
```

그래서 Hibernate Batch 성능을 볼 때 ID 생성 전략도 반드시 확인해야 한다.

#### 8. 결국 둘 다 JDBC Batch인가?

그렇다.
이 점을 이해하는 것이 중요하다.

```text
JdbcTemplate.batchUpdate()
          │
          │ 직접
          ▼
      JDBC Batch
          ▼
         DB
```

Hibernate는:

```text
JPA
 ↓
Hibernate
 ↓
ActionQueue
 ↓
JDBC Batch
 ↓
DB
```

즉 Hibernate Batch라는 완전히 별개의 DB 기술이 있는 게 아니다.
Hibernate가 내부적으로 JDBC Batch를 사용해주는 것이다.

#### 9. 실무 선택 기준

실무에서는 다음 기준으로 구분할 수 있다.

##### 일반적인 비즈니스 CRUD

```java
orderRepository.save(order);
```

→ JPA를 사용한다.

##### 수백~수천 건 정도를 Entity 기반으로 한 번에 처리

```java
entityManager.persist()
+
hibernate.jdbc.batch_size
+
flush()/clear()
```

→ Hibernate Batch.

##### 수만~수백만 건의 단순 INSERT / UPDATE

```java
jdbcTemplate.batchUpdate()
```

→ JdbcTemplate Batch를 적극 고려.
예를 들어:

```text
CSV 100만 건 적재
통계 데이터 대량 INSERT
Embedding 결과 수십만 건 저장
배치 Job 결과 저장
Migration
대량 상태 UPDATE
```

같은 경우라면 JdbcTemplate이 더 적합할 수 있다.

#### 한 문장으로 정리

```text
Hibernate Batch
= JPA의 편의성/Entity 관리를 유지하면서
  Hibernate가 JDBC Batch를 대신 사용해주는 것

JdbcTemplate.batchUpdate()
= JPA를 거치지 않고
  JDBC Batch를 직접 사용해서 대량 SQL 실행에 집중하는 것
```

Hibernate Batch와 JdbcTemplate Batch는 모두 내부적으로 JDBC Batch를 사용한다. 차이는 JDBC 위에서 Hibernate의 영속성 컨텍스트와 Entity 관리 계층을 거치는지에 있다.

이번 개선에서 중요한 부분 중 하나다.
단순히

```text
Batch가 빠르다.
→ 전체 JPA에 Batch 설정 적용
```

하는 것이 아니라 변경 범위를 최소화했다.

```text
기존

전체 애플리케이션
        │
        ├── JPA 로직 A
        ├── JPA 로직 B
        ├── JPA 로직 C
        │
        └── Kafka 대량 저장
```

이번 문제는 Kafka 대량 저장 구간에서 발생했다.
따라서

```text
전체 애플리케이션
        │
        ├── JPA 로직 A ── 기존 유지
        ├── JPA 로직 B ── 기존 유지
        ├── JPA 로직 C ── 기존 유지
        │
        └── Kafka 대량 저장
                  ↓
            JdbcTemplate Batch
```

형태로 변경했다.
성능을 개선하면서 기존 코드에 미치는 영향을 최소화한 것이다.

### Hibernate Batch를 전체 적용하지 않고 JdbcTemplate을 선택한 이유

Hibernate Batch는 보통 `hibernate.jdbc.batch_size` 같은 설정을 통해 Hibernate가 수행하는 INSERT/UPDATE 전반에 영향을 줄 수 있다.
즉 Kafka 저장 구간만 최적화하고 싶어도,

```text
JPA 로직 A
JPA 로직 B
JPA 로직 C
Kafka 저장 로직
       ↓
Hibernate Batch 설정의 영향 범위
```

가 넓어질 수 있다.
특히 배치가 적용되면 SQL이 즉시 실행되지 않고 일정량 모였다가 `flush` 시점에 실행되므로, 기존 코드의 SQL 실행 시점, flush 동작, 영속성 컨텍스트 메모리 사용량, 장애 발생 시점 등이 달라질 수 있다. `order_inserts`, `order_updates` 같은 추가 최적화 설정까지 사용하면 SQL 실행 순서에도 영향을 줄 수 있다.
반면 `JdbcTemplate.batchUpdate()`는:

```text
기존 JPA 로직 → 아무 변화 없음

Kafka 대량 저장
      ↓
JdbcTemplate.batchUpdate()
      ↓
JDBC Batch
```

처럼 성능 문제가 있는 특정 구간만 바꿀 수 있다.
즉, Hibernate Batch가 다른 로직을 반드시 망가뜨린다는 뜻이 아니라, 전역 설정으로 불필요하게 영향 범위를 넓히는 대신 문제가 발생한 대량 저장 경로만 최적화하여 회귀 위험을 줄였다는 의미다.

## 17. JdbcTemplate으로 변경하면서 발생하는 차이

JPA를 사용하면 객체를 저장할 때 Hibernate가 여러 작업을 대신 처리한다.

```text
Entity
  ↓
JPA
  ↓
Hibernate
  ↓
SQL 생성
  ↓
DB
```

하지만 `JdbcTemplate`을 사용하면 SQL을 직접 작성한다.

```text
데이터
  ↓
JdbcTemplate
  ↓
작성한 SQL
  ↓
DB
```

따라서 기존에 Hibernate가 처리하던 부분 중 필요한 것은 직접 구현해야 한다.
이번 사례에서는 특히 ID 생성 방식을 확인해야 했다.

## 18. 기존 ID 생성 방식 유지

기존 Entity는 Hibernate의 GUID 생성 전략을 사용하고 있었고, MSSQL의 `NEWID()`를 통해 ID가 생성되고 있었다.
기존 흐름은 다음과 같이 볼 수 있다.

```text
Entity 저장
   ↓
Hibernate
   ↓
MSSQL
   ↓
NEWID()
   ↓
ID 생성
```

그런데 `JdbcTemplate`을 사용하면 JPA/Hibernate의 저장 과정을 거치지 않는다.

```text
JdbcTemplate
   ↓
직접 작성한 INSERT
   ↓
MSSQL
```

따라서 ID 생성도 고려해야 한다.
한 가지 방법은 Java에서 UUID를 만드는 것이다.

```java
UUID.randomUUID();
```

하지만 그렇게 하면 기존의 ID 생성 방식을 변경하게 된다.
이번 개선에서는 성능 문제와 관련 없는 기존 데이터 생성 방식까지 변경하지 않는 것을 우선했다.
따라서 INSERT SQL에서 기존과 동일하게 MSSQL의 `NEWID()`를 사용했다.

```sql
INSERT INTO TABLE_NAME (
    ID,
    COLUMN_A,
    COLUMN_B
)
VALUES (
    NEWID(),
    ?,
    ?
)
```

결과적으로

```text
기존

JPA
 ↓
Hibernate
 ↓
MSSQL NEWID()

변경

JdbcTemplate
 ↓
MSSQL NEWID()
```

처럼 DB 저장 방식은 변경하되 ID 생성 방식은 유지할 수 있었다.

## 19. 전체 개선 과정

전체 문제 해결 과정을 한 번에 보면 다음과 같다.

```text
[1] Kafka 메시지 처리 이상
            ↓
[2] 로그 확인
            ↓
Rebalancing 반복 발견
            ↓
[3] Rebalancing 주기 확인
            ↓
약 10분
            ↓
[4] Consumer 설정 확인
            ↓
max.poll.interval.ms = 10분
            ↓
[5] 메시지 처리 시간 확인
            ↓
10분 이상
            ↓
[6] 처리 내부 로직 확인
            ↓
약 46,000건 MSSQL 저장
            ↓
[7] DB 저장 병목 확인
            ↓
[8] 우선 운영 조치
            ↓
max.poll.interval.ms
10분 → 30분
            ↓
[9] 근본적인 개선
            ↓
JPA 대량 저장 구간
→ JdbcTemplate Batch Insert
            ↓
[10] 기존 ID 생성 방식 유지
            ↓
MSSQL NEWID()
            ↓
[11] DB 처리 시간 감소
            ↓
메시지 전체 처리 시간 감소
            ↓
Rebalancing 완화
```

## 20. 개선 결과

Batch Insert를 적용하면서 약 46,000건의 데이터를 저장하는 DB 처리 시간이 감소했다.
DB 처리 시간이 줄어들자 자연스럽게 Kafka 메시지 하나의 전체 처리 시간도 감소했다.

```text
개선 전

Kafka 메시지
    ↓
46,000건 DB 저장
    ↓
처리 시간 10분 이상
    ↓
max.poll.interval.ms 초과
    ↓
Rebalancing

개선 후

Kafka 메시지
    ↓
Batch Insert
    ↓
DB 저장 시간 감소
    ↓
전체 처리 시간 감소
    ↓
다음 poll() 빠르게 호출
    ↓
Rebalancing 완화
```

여기서 중요한 것은 `max.poll.interval.ms`를 단순히 증가시키고 끝내지 않았다는 것이다.
두 조치의 목적이 서로 다르다.
| 방법 | 목적 |
| --- | --- |
| `max.poll.interval.ms` 증가 | 현재 운영 문제를 빠르게 완화 |
| Batch Insert | 실제 메시지 처리 시간을 줄이는 근본적인 개선 |
즉 운영 안정화와 근본 원인 개선을 분리해서 접근한 것이다.

## 21. 이 사례에서 중요한 점

### 1. 장애가 보이는 위치와 실제 원인은 다를 수 있다

처음 보이는 문제는 Kafka였다.

```text
Kafka Rebalancing 발생
```

하지만 원인을 따라가 보면

```text
Kafka
 ↓
Consumer
 ↓
메시지 처리
 ↓
DB 저장
 ↓
병목
```

실제로 개선해야 했던 부분은 DB 저장 로직이었다.
따라서 장애가 발생한 컴포넌트만 보고 원인을 판단하면 안 된다.

### 2. 설정값을 늘리는 것이 항상 해결책은 아니다

`max.poll.interval.ms`를 늘리면 Consumer가 메시지를 처리할 수 있는 시간을 확보할 수 있다.
하지만 처리 시간이 계속 증가한다면

```text
10분 → 30분
```

으로 변경한 뒤에도 언젠가는 다시

```text
처리 시간 35분
> max.poll.interval.ms 30분
```

상황이 발생할 수 있다.
따라서 설정 조정과 함께 왜 처리 시간이 긴지도 확인해야 한다.

### 3. Rebalancing이 발생하면 처리 시간도 같이 확인해야 한다

Rebalancing 로그만 보고 Kafka 설정만 살펴보기보다는 실제 메시지 처리 시간을 같이 확인할 필요가 있다.

```text
Rebalancing 발생
        ↓
Consumer 설정 확인
        +
실제 처리 시간 확인
        ↓
두 값 비교
```

특히

```text
처리 시간 ≈ max.poll.interval.ms
```

라면 메시지 처리 로직의 병목을 의심할 수 있다.

### 4. 대량 DB 저장은 저장 방식을 확인해야 한다

수만 건의 데이터를 저장해야 한다면 단순히 데이터를 반복해서 저장하는 것보다 Batch 처리를 고려할 수 있다.

```text
반복 INSERT

INSERT
INSERT
INSERT
INSERT
...

Batch

[INSERT INSERT INSERT ...]
        ↓
     Batch 실행
```

특히 특정 대량 저장 구간만 병목이라면 애플리케이션 전체 설정을 변경하기보다 해당 구간만 `JdbcTemplate.batchUpdate()` 같은 방식으로 최적화하는 것도 방법이다.

### 5. 성능 개선에서도 변경 범위를 생각해야 한다

성능을 개선한다고 해서 관련된 모든 코드를 변경할 필요는 없다.
이번 경우에도

```text
DB 저장 방식
JPA → JdbcTemplate Batch
```

은 변경했지만

```text
ID 생성 방식
MSSQL NEWID()
```

은 기존 방식을 유지했다.
즉 문제와 관계없는 영역은 최대한 유지하면서 필요한 부분만 변경하는 것도 중요하다.

## 최종 정리

이번 문제의 핵심 흐름은 다음과 같다.

```text
46,000건의 DB 저장
        ↓
메시지 처리 시간 증가
        ↓
다음 poll() 호출 지연
        ↓
max.poll.interval.ms 초과
        ↓
Consumer Group Rebalancing
        ↓
메시지 처리 불안정
```

이를 해결하기 위해 먼저

```text
max.poll.interval.ms
10분 → 30분
```

으로 늘려 운영 환경에서 반복되는 Rebalancing을 완화했다.
이후 실제 병목이었던 대량 DB 저장 부분을

```text
기존 JPA 저장
      ↓
JdbcTemplate.batchUpdate()
```

방식으로 변경했다.
그 결과 DB 처리 시간이 감소했고, Kafka 메시지 하나의 전체 처리 시간도 함께 줄어들면서 Rebalancing 문제도 완화할 수 있었다.
이 사례에서 가장 중요한 부분은 특정 Kafka 설정 하나를 외우는 것이 아니라 다음과 같이 현상에서 시작해 원인을 계속 아래로 추적하는 과정이다.

```text
Rebalancing이 발생한다.
        ↓
왜?

poll() 호출이 늦다.
        ↓
왜?

메시지 처리 시간이 길다.
        ↓
왜?

DB 저장이 오래 걸린다.
        ↓
어떻게 줄일까?

Batch Insert 적용
```

Kafka Consumer에서 Rebalancing이 반복된다면 Kafka 자체만 확인하기보다는 Consumer가 실제로 어떤 작업을 수행하고 있는지, 메시지 한 건을 처리하는 데 얼마나 걸리는지, DB나 외부 API 같은 하위 작업에서 병목이 발생하고 있지는 않은지까지 함께 확인하는 것이 중요하다.
