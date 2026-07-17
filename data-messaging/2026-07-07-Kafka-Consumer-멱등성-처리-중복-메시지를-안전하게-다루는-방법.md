# Kafka Consumer 멱등성 처리: 중복 메시지를 안전하게 다루는 방법

- 읽은 날짜: 2026-07-07
- 회사/작성 주체: Conduktor
- 원문: [Kafka Consumer 멱등성 처리: 중복 메시지를 안전하게 다루는 방법](https://www.conduktor.io/blog/building-idempotent-consumers)

---

Kafka를 사용하면 Consumer가 같은 메시지를 두 번 이상 받을 수 있다.

이것은 Kafka의 버그가 아니라, **at-least-once 전달 방식에서 자연스럽게 발생하는 상황**이다.

따라서 Kafka 기반 시스템에서는 “중복 메시지를 절대 받지 않게 만들자”보다,

**같은 메시지를 여러 번 받아도 결과가 한 번만 반영되도록 Consumer를 멱등하게 설계하는 것**이 중요하다.

---

## 1. Kafka에서 중복 메시지가 발생하는 이유

Kafka Consumer가 중복 메시지를 받는 대표적인 상황은 세 가지다.

```
1. Producer 재시도
2. Consumer 장애
3. Consumer Group Rebalance
```

### 1-1. Producer 재시도

Producer가 메시지를 Kafka Broker에 보냈고, Broker도 메시지를 정상적으로 저장했다고 해보자.

그런데 Producer가 받아야 할 ACK 응답이 네트워크 문제로 유실되면 Producer는 전송 실패로 판단할 수 있다.

이 경우 Producer는 같은 메시지를 다시 보낸다.

```
[Producer]
   |
   | 메시지 전송
   v
[Kafka Broker]
   |
   | 메시지 저장 성공
   | 하지만 ACK 응답 유실
   v
[Producer]
   |
   | 실패로 판단하고 재전송
   v
[Kafka Broker]
   |
   | 같은 메시지가 다시 저장될 수 있음
```

즉, Broker에는 이미 저장됐지만 Producer가 성공을 확인하지 못해서 중복 메시지가 생길 수 있다.

---

### 1-2. Consumer 장애

Consumer가 메시지를 읽고 DB 저장까지 성공했다고 해보자.

그런데 Kafka offset commit 전에 Consumer가 장애로 죽으면 문제가 생긴다.

Kafka 입장에서는 해당 메시지가 아직 처리 완료되지 않은 것으로 보인다.

그래서 Consumer가 재시작되면 같은 메시지를 다시 전달한다.

```
[Consumer]
   |
   | 메시지 처리
   v
[DB 저장 성공]
   |
   | offset commit 전 장애 발생
   v
[Consumer 재시작]
   |
   | Kafka가 같은 메시지를 다시 전달
   v
[중복 처리 위험]
```

이 상황이 Kafka Consumer 멱등성 설계에서 가장 자주 언급되는 문제다.

---

### 1-3. Consumer Group Rebalance

Consumer Group에서 Consumer가 추가되거나 제거되면 Partition 재분배가 일어난다.

이 과정을 Rebalance라고 한다.

Rebalance 중에 아직 offset commit이 끝나지 않은 메시지가 있다면,

새로운 Consumer가 같은 메시지를 다시 처리할 수 있다.

```
[Consumer A]
   |
   | 메시지 처리 중
   v
[Rebalance 발생]
   |
   | Partition 소유권 이동
   v
[Consumer B]
   |
   | commit 안 된 메시지를 다시 처리
```

결국 Kafka에서는 여러 이유로 Consumer가 같은 메시지를 다시 받을 수 있다.

---

## 2. enable.idempotence=true만으로는 부족하다

Kafka에는 `enable.idempotence=true` 설정이 있다.

이 설정은 Producer가 Broker로 메시지를 보낼 때 발생할 수 있는 중복 전송을 줄여준다.

하지만 이것은 **Producer 쪽 중복 방지 기능**이다.

```java
1. Producer는 메시지를 Broker에 보낸 뒤 ACK를 받아야 성공 여부를 알 수 있다.

2. Broker에는 메시지가 저장됐지만 ACK가 손실되면,
   Producer는 실패로 판단하고 같은 메시지를 재전송할 수 있다.

3. enable.idempotence=false라면
   Broker는 재전송 메시지를 새 메시지로 저장할 수 있다.

4. enable.idempotence=true라면
   Producer ID와 Sequence Number를 사용해서
   Broker가 이미 처리한 메시지인지 판단할 수 있다.

5. 그래서 같은 Producer가 같은 Partition에 같은 메시지를 재전송해도
   Broker 로그에는 한 번만 저장되도록 막아준다.

6. 하지만 이것은 Producer -> Broker 구간의 중복 방지일 뿐이다.

7. Consumer 재처리, DB 중복 저장, 사용자 중복 요청은 별도 멱등성 처리가 필요하다.

8. 다음 둘을 같이 써야 안전하다.

   Kafka Producer:
   enable.idempotence=true

   DB:
   UNIQUE(event_id, user_id)
   UNIQUE(request_id)
```

```
enable.idempotence=true
        |
        v
Producer → Kafka Broker 사이의 중복 전송 완화
```

반면 Consumer 쪽 문제는 여전히 남아 있다.

```
Kafka Broker → Consumer
        |
        v
Consumer 장애, offset commit 실패, rebalance로 인한 중복 처리 가능
```

즉, Producer 멱등성과 Consumer 멱등성은 다르다.

```
구분                     역할
------------------------------------------------
Idempotent Producer      Producer의 중복 전송 방지
Idempotent Consumer      Consumer의 중복 처리 방지
```

이 글의 핵심은 **Consumer를 멱등하게 설계해야 한다**는 것이다.

---

## 3. 멱등 Consumer란 무엇인가?

멱등성이란 같은 작업을 여러 번 수행해도 결과가 한 번 수행한 것과 같도록 만드는 성질이다.

Kafka Consumer에 적용하면 다음과 같다.

```
같은 메시지를 1번 처리해도 DB 결과가 1개
같은 메시지를 2번 처리해도 DB 결과가 1개
같은 메시지를 10번 처리해도 DB 결과가 1개
```

즉, Kafka Consumer는 같은 메시지를 여러 번 받아도 실제 비즈니스 결과는 한 번만 반영되도록 만들어야 한다.

기본 흐름은 다음과 같다.

```
[Kafka Message]
   |
   v
[Idempotency Key 추출]
   |
   v
[이미 처리한 메시지인지 확인]
   |
   +-- YES --> [중복 메시지이므로 스킵]
   |
   +-- NO  --> [비즈니스 로직 처리]
                |
                v
             [처리 완료 기록 저장]
```

겉으로 보면 단순해 보이지만, 핵심은 **원자성**이다.

---

## 4. 확인, 처리, 기록이 분리되면 위험하다

Consumer가 다음 순서로 동작한다고 해보자.

```
1. 이 메시지를 처리했는지 확인
2. 처리하지 않은 메시지라면 비즈니스 로직 실행
3. 처리 완료 기록 저장
```

이 흐름에서 2번과 3번 사이에 장애가 발생하면 문제가 생긴다.

```
[처리 여부 확인]
   |
   v
[처리 안 된 메시지로 판단]
   |
   v
[DB에 쿠폰 발급 저장 성공]
   |
   v
[처리 완료 기록 저장 전 Consumer 장애]
   |
   v
[Consumer 재시작]
   |
   v
[처리 완료 기록이 없으므로 다시 처리]
```

즉, 실제 비즈니스 처리는 성공했는데 처리 완료 기록이 남지 않아서 중복 처리가 발생할 수 있다.

그래서 멱등 Consumer 설계에서는 다음 두 작업을 하나의 원자적 단위로 묶어야 한다.

```
1. 이 메시지를 처리했다는 기록
2. 실제 비즈니스 데이터 변경
```

둘 다 성공하거나, 둘 다 실패해야 한다.

---

## 5. Idempotency Key 설계

Consumer가 중복 메시지를 구분하려면 기준이 필요하다.

이 기준이 바로 **Idempotency Key**다.

좋은 Idempotency Key는 다음 조건을 만족해야 한다.

```
1. 하나의 논리적 작업마다 고유해야 한다.
2. 재시도되어도 값이 바뀌면 안 된다.
3. 가능하면 Producer 또는 API 계층에서 생성하는 것이 좋다.
```

대표적인 방식은 세 가지다.

---

### 5-1. Client 또는 API 계층에서 생성한 UUID

가장 안전한 방식이다.

API 요청이 들어온 시점에 `requestId`를 생성하고, 이 값을 Kafka 메시지까지 전달한다.

```
[Client]
   |
   | 쿠폰 발급 요청
   v
[API Server]
   |
   | requestId = uuid-123 생성
   v
[Kafka Message]
   |
   | requestId = uuid-123 포함
   v
[Consumer]
   |
   | requestId 기준으로 중복 판단
```

이 방식은 같은 요청이 재시도되어도 동일한 requestId를 사용할 수 있기 때문에 멱등성 처리에 적합하다.

---

### 5-2. Composite Business Key

비즈니스 값을 조합해서 Key를 만들 수도 있다.

예를 들어 주문 시스템이라면 다음과 같이 만들 수 있다.

```
customerId:orderId
```

선착순 쿠폰 발급 시스템에서는 다음과 같은 조합이 가능하다.

```
couponEventId:userId
```

이 방식은 비즈니스적으로 “한 사용자는 하나의 쿠폰 이벤트에서 한 번만 발급받을 수 있다”는 조건이 명확할 때 유용하다.

다만 조합한 값이 정말 유일한지 주의해야 한다.

---

### 5-3. Kafka Coordinates

Kafka의 `topic-partition-offset`을 Key로 사용할 수도 있다.

```
coupon.issue.requested-0-152
```

이 방식은 간단하지만 단점이 있다.

같은 메시지를 다른 Topic으로 옮겨 재처리하거나, DLQ에서 다시 replay하면 offset이 달라질 수 있다.

그러면 논리적으로는 같은 메시지인데 다른 메시지처럼 인식될 위험이 있다.

따라서 비즈니스 멱등성에는 보통 `requestId`나 `business key`를 사용하는 것이 더 안전하다.

---

## 6. Pattern 1: DB Unique Constraint 방식

가장 단순하고 실용적인 방식은 DB Unique Constraint를 사용하는 것이다.

처리한 메시지의 Idempotency Key를 DB 테이블에 저장하고, 같은 Key가 다시 들어오면 DB 제약 조건으로 막는다.

예시 테이블은 다음과 같다.

```sql
CREATE TABLE processed_messages (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Consumer 처리 흐름은 다음과 같다.

```
[Kafka Message]
   |
   v
[DB Transaction 시작]
   |
   v
[processed_messages에 idempotency_key INSERT]
   |
   +-- 성공 --> [비즈니스 로직 실행]
   |
   +-- 실패 --> [이미 처리한 메시지이므로 스킵]
   |
   v
[Transaction Commit]
```

중요한 점은 `processed_messages INSERT`와 실제 비즈니스 처리를 하나의 트랜잭션으로 묶는 것이다.

```
[Transaction 시작]
   |
   v
[processed_messages INSERT]
   |
   v
[쿠폰 발급 데이터 INSERT]
   |
   v
[Commit]
```

만약 쿠폰 발급 저장 중 실패하면 트랜잭션 전체가 롤백된다.

```
[processed_messages INSERT 성공]
   |
   v
[쿠폰 발급 INSERT 실패]
   |
   v
[Transaction Rollback]
   |
   v
[processed_messages 기록도 롤백]
```

이렇게 해야 나중에 같은 메시지가 다시 들어왔을 때 정상적으로 재처리할 수 있다.

---

## 7. DB Constraint 방식의 장단점

DB Unique Constraint 방식의 장점은 단순함이다.

```
장점
- 기존 DB만으로 구현 가능
- 비즈니스 데이터와 중복 처리 기록을 같은 트랜잭션으로 묶기 쉬움
- CRUD 중심 서비스에 적합
- 구조가 명확해서 운영과 디버깅이 쉬움
```

단점도 있다.

```
단점
- 모든 메시지마다 deduplication 테이블에 쓰기 발생
- 트래픽이 많으면 write hotspot이 될 수 있음
- 오래된 처리 기록 정리 전략이 필요함
```

대부분의 서비스가 Kafka 메시지를 읽고 DB에 저장하는 구조라면,

이 방식이 가장 현실적인 출발점이다.

---

## 8. Pattern 2: Redis SETNX 방식

고성능 처리가 필요하다면 Redis의 `SETNX`를 사용할 수 있다.

`SETNX`는 “Key가 없을 때만 저장”하는 명령이다.

```
SETNX dedup:request-123 1
```

처음 들어온 메시지라면 Key가 없으므로 저장에 성공한다.

이미 처리한 메시지라면 Key가 존재하므로 저장에 실패한다.

```
[Consumer]
   |
   v
SETNX dedup:request-123 1 EX 7days
   |
   +-- 성공 --> [처음 처리하는 메시지]
   |
   +-- 실패 --> [중복 메시지, 스킵]
```

Redis 방식의 장점은 빠르다는 것이다.

```
장점
- DB보다 빠름
- TTL로 오래된 dedup key 자동 삭제 가능
- 고성능 Consumer에 적합
```

하지만 단점도 있다.

```
단점
- Redis 장애 시 Consumer 처리에 영향
- Redis 데이터 유실 시 중복 처리 기록도 유실 가능
- Redis와 DB 작업을 하나의 트랜잭션으로 묶기 어려움
```

특히 Redis를 중복 처리 저장소로 사용할 경우 영속성 설정이 중요하다.

```
appendonly yes
appendfsync everysec
```

RDB snapshot만 사용하면 Redis 장애 시 최근 dedup key가 사라질 수 있다.

```java
상황:
Kafka Consumer가 메시지를 처리할 때 Redis에 dedup key를 저장해서
"이미 처리한 메시지인지" 확인한다고 가정하자.

예시 메시지:
requestId = req-100

처리 흐름:

[Kafka Message]
requestId = req-100
        |
        v
[Consumer]
Redis에 key 확인
        |
        v
dedup:req-100 없음
        |
        v
DB에 쿠폰 발급 처리
        |
        v
Redis에 저장
SET dedup:req-100 true

이제 Redis 상태는 이렇게 된다.

Redis:
dedup:req-100 = true

즉, 같은 메시지가 다시 와도 Consumer는 이렇게 판단한다.

[Kafka Message 재전달]
requestId = req-100
        |
        v
[Consumer]
Redis에 dedup:req-100 있음
        |
        v
"이미 처리한 메시지네?"
        |
        v
DB 처리 안 함

여기까지는 정상이다.

==================================================

그런데 Redis가 장애로 재시작됐다고 해보자.

Redis 영속성을 RDB snapshot만 사용하면
Redis는 일정 주기마다만 데이터를 디스크에 저장한다.

예를 들어:

12:00:00  마지막 RDB snapshot 저장
12:00:05  Consumer가 req-100 처리
12:00:05  Redis에 dedup:req-100 저장
12:00:07  Redis 장애 발생
12:00:08  Redis 재시작

문제는 12:00:05에 저장된 dedup:req-100이
아직 RDB snapshot에 포함되지 않았을 수 있다는 것이다.

그러면 Redis 재시작 후 상태는 이렇게 된다.

Redis:
dedup:req-100 없음

즉, Redis는 req-100을 처리한 적이 없다고 착각한다.

==================================================

그 상태에서 Kafka가 같은 메시지를 다시 전달하면?

[Kafka Message 재전달]
requestId = req-100
        |
        v
[Consumer]
Redis에 dedup:req-100 확인
        |
        v
없음
        |
        v
"처음 처리하는 메시지네?"
        |
        v
DB에 쿠폰 발급 처리 다시 시도

이게 바로 "이미 처리한 메시지가 다시 처리될 위험"이다.

==================================================

그래서 AOF 설정을 사용한다.

appendonly yes
appendfsync everysec

의미:

appendonly yes
→ Redis 명령을 로그 파일처럼 계속 기록한다.

appendfsync everysec
→ 최대 1초 단위로 디스크에 반영한다.

예:

SET dedup:req-100 true

이 명령 자체를 AOF 파일에 남긴다.

Redis가 죽었다가 다시 살아나도
AOF 로그를 재생해서 dedup:req-100을 복구할 수 있다.

==================================================

단, appendfsync everysec도 완벽한 것은 아니다.

최대 1초 정도의 데이터는 장애 시 유실될 수 있다.

그래서 진짜 최종 정합성이 중요하면
Redis dedup만 믿으면 안 되고,
DB에도 unique 제약 조건을 둬야 한다.

예:

coupon_issues 테이블

UNIQUE(event_id, user_id)
또는
UNIQUE(request_id)

이렇게 하면 Redis dedup key가 사라져도
DB가 마지막 방어선 역할을 한다.

==================================================

핵심 정리:

RDB snapshot만 사용
→ Redis가 주기적으로 저장한다.
→ 장애 직전에 저장된 dedup key가 사라질 수 있다.
→ Consumer가 같은 메시지를 처음 보는 메시지로 착각할 수 있다.

AOF 사용
→ Redis 명령 로그를 남긴다.
→ 장애 후 dedup key 복구 가능성이 높아진다.
→ 중복 처리 위험을 줄인다.

하지만 최종 정합성은 Redis가 아니라 DB unique 제약 조건으로 막아야 한다.

```

그러면 이미 처리한 메시지가 다시 처리될 위험이 있다.

---

## 9. Pattern 3: Natural Idempotency 방식

비즈니스 로직 자체가 멱등하게 설계되어 있다면 별도의 deduplication 저장소가 필요 없을 수도 있다.

예를 들어 재고 수량을 “증가”시키는 것이 아니라 “특정 값으로 교체”하는 이벤트라면 같은 이벤트가 여러 번 와도 결과가 같다.

```
product_id = 1
quantity = 100
event_version = 5
```

이 이벤트를 여러 번 적용해도 최종 quantity는 계속 100이다.

이런 경우 UPSERT를 사용할 수 있다.

```sql
INSERT INTO inventory (product_id, quantity, event_version)
VALUES (?, ?, ?)
ON CONFLICT (product_id)
DO UPDATE SET quantity = EXCLUDED.quantity
WHERE inventory.event_version < EXCLUDED.event_version;
```

이 방식은 상태 교체 이벤트에 적합하다.

```
잘 맞는 경우
- 특정 상태로 교체하는 이벤트
- 최신 버전만 반영하면 되는 이벤트
- UPSERT로 처리 가능한 데이터
```

하지만 모든 경우에 사용할 수 있는 것은 아니다.

```
잘 안 맞는 경우
- 포인트 +100 적립
- 쿠폰 1개 발급
- 이메일 발송
- 결제 요청
- 외부 API 호출
```

이런 작업은 같은 메시지가 두 번 처리되면 실제 부작용도 두 번 발생할 수 있다.

따라서 선착순 쿠폰 발급 같은 시스템에서는 Natural Idempotency만으로는 부족하다.

DB Unique Constraint나 requestId 기반 멱등성 처리가 필요하다.

---

## 10. Deduplication Window

중복 처리 기록을 얼마나 오래 보관할지도 중요하다.

이를 Deduplication Window라고 볼 수 있다.

예를 들어 Redis dedup key의 TTL을 7일로 설정했다고 해보자.

```
SET dedup:request-123 1 EX 7days
```

그러면 7일 이내에 같은 메시지가 다시 들어오면 중복으로 판단할 수 있다.

하지만 10일 뒤에 같은 메시지를 replay하면 어떻게 될까?

```
[7일 경과]
   |
   v
[dedup key 삭제됨]
   |
   v
[10일 뒤 같은 메시지 replay]
   |
   v
[Consumer는 처음 보는 메시지로 판단]
   |
   v
[중복 처리 가능]
```

따라서 Deduplication Window는 서비스의 replay 가능 기간보다 길어야 한다.

고려해야 할 기준은 다음과 같다.

```
1. Consumer가 최대 얼마 동안 중단될 수 있는가?
2. 장애 복구를 위해 며칠 뒤 메시지를 replay할 가능성이 있는가?
3. Producer retry가 얼마나 오래 지속될 수 있는가?
4. DLQ 메시지를 언제까지 재처리할 수 있는가?
```

일반적으로 7일 정도가 많이 언급되지만, 결제나 쿠폰 발급처럼 중복 처리가 치명적인 시스템은 더 길게 가져갈 수도 있다.

---

## 11. 멱등성 테스트

Consumer 멱등성은 반드시 테스트해야 한다.

가장 기본적인 테스트는 같은 메시지를 두 번 처리해도 결과가 한 번만 생성되는지 확인하는 것이다.

```
[동일 메시지 1회 처리]
   |
   v
[DB row 1개 생성]

[동일 메시지 2회 처리]
   |
   v
[DB row 여전히 1개]
```

또한 동시성 테스트도 필요하다.

```
[Consumer A] ---- 같은 메시지 처리 시도
[Consumer B] ---- 같은 메시지 처리 시도
```

이 상황에서도 DB에 결과가 하나만 생성되어야 한다.

실제 운영 장애는 단순한 중복 처리보다, 동시에 같은 메시지를 처리하는 경쟁 상황에서 더 자주 발생한다.

---

## DB Unique Constraint를 쓰면 deduplication 테이블 단점이 그대로 있는 걸까?

```java
결론부터 말하면:

"중복 방지를 DB에서 한다"는 점은 비슷하지만,
deduplication 테이블을 따로 두는 것과
비즈니스 테이블에 unique constraint를 거는 것은 부담이 다르다.

==================================================
1. deduplication 테이블 방식
==================================================

예를 들어 Consumer가 메시지를 처리할 때마다
processed_messages 테이블에 request_id를 저장한다고 해보자.

[Kafka Message]
requestId = req-100
        |
        v
[Consumer]
        |
        v
INSERT INTO processed_messages(request_id)
VALUES ('req-100')
        |
        v
성공하면 실제 비즈니스 로직 처리
        |
        v
INSERT INTO coupon_issues(event_id, user_id)

테이블 구조:

processed_messages
- request_id unique
- processed_at

coupon_issues
- event_id
- user_id
- issued_at

이 방식은 모든 메시지마다
"중복 확인용 테이블"에 write가 추가로 발생한다.

즉, 쿠폰 발급 1번에 DB write가 최소 2번이다.

1. processed_messages insert
2. coupon_issues insert

그래서 단점이 생긴다.

- 모든 메시지마다 deduplication 테이블 쓰기 발생
- 특정 테이블에 write가 몰림
- processed_messages가 계속 커짐
- 오래된 request_id 정리 필요

==================================================
2. 비즈니스 테이블 unique constraint 방식
==================================================

이번에는 별도 deduplication 테이블 없이
coupon_issues 자체에 unique constraint를 건다고 해보자.

coupon_issues
- id
- event_id
- user_id
- request_id
- issued_at

제약 조건:

UNIQUE(event_id, user_id)

또는

UNIQUE(request_id)

처리 흐름:

[Kafka Message]
eventId = 1
userId = 100
requestId = req-100
        |
        v
[Consumer]
        |
        v
INSERT INTO coupon_issues(event_id, user_id, request_id)
VALUES (1, 100, 'req-100')
        |
        v
성공하면 발급 성공
실패하면 이미 발급된 요청으로 판단

이 방식은 따로 deduplication 테이블에 쓰지 않는다.

즉, 쿠폰 발급 1번에 핵심 write는 1번이다.

1. coupon_issues insert

중복이면 DB unique constraint가 막는다.

==================================================
3. 그래서 단점이 완전히 같은가?
==================================================

완전히 같지는 않다.

비교하면 이렇게 된다.

| 구분 | deduplication 테이블 | 비즈니스 테이블 unique |
|---|---|---|
| 쓰기 횟수 | dedup 테이블 + 비즈니스 테이블 | 비즈니스 테이블 1번 |
| 중복 방지 위치 | 별도 processed_messages | coupon_issues 자체 |
| 테이블 증가 | processed_messages도 계속 증가 | coupon_issues만 증가 |
| 정리 필요성 | dedup 기록 정리 필요 | 보통 발급 기록은 보존 |
| write hotspot | dedup 테이블에 집중 가능 | 실제 발급 테이블에 분산 |
| 의미 | 메시지 처리 여부 저장 | 실제 발급 결과 저장 |

즉, DB unique constraint도 write 비용은 있지만
"어차피 저장해야 하는 비즈니스 데이터"에 제약을 거는 것이다.

반면 deduplication 테이블은
중복 방지만을 위해 추가 write를 만드는 구조다.

==================================================
4. 그림으로 보면 차이가 더 명확하다
==================================================

deduplication 테이블 방식:

[Consumer]
    |
    | 1. dedup 기록 insert
    v
[processed_messages]
    |
    | 2. 실제 발급 insert
    v
[coupon_issues]

문제:
- 메시지마다 processed_messages에 write
- processed_messages가 계속 커짐
- 나중에 정리 필요

비즈니스 테이블 unique 방식:

[Consumer]
    |
    | 1. 실제 발급 insert
    v
[coupon_issues + UNIQUE(event_id, user_id)]

장점:
- 별도 dedup write 없음
- 발급 기록 자체가 중복 방어선
- 오래된 dedup 기록 정리 문제가 줄어듦

==================================================
5. 그럼 DB unique도 write hotspot이 없는 건가?
==================================================

아니다. DB unique도 당연히 비용이 있다.

예를 들어 선착순 쿠폰 발급에서
모든 요청이 같은 event_id = 1로 몰리면
coupon_issues 테이블의 unique index에 쓰기가 집중된다.

UNIQUE(event_id, user_id)

인덱스 예시:

(event_id, user_id)
(1, 100)
(1, 101)
(1, 102)
(1, 103)
...

event_id가 모두 1이라서
같은 인덱스 범위에 쓰기가 몰릴 수 있다.

즉, DB unique constraint도
고트래픽 상황에서는 병목이 될 수 있다.

다만 차이는 이거다.

deduplication 테이블 병목:
"중복 방지를 위해 추가로 생긴 병목"

비즈니스 테이블 unique 병목:
"최종 발급 저장을 위해 어차피 필요한 병목"

==================================================
6. 그래서 Redis를 앞단에 두는 이유
==================================================

DB unique constraint만 쓰면
모든 요청이 DB까지 도달한다.

[Client]
   |
   v
[API]
   |
   v
[Kafka]
   |
   v
[Consumer]
   |
   v
[DB unique constraint]

트래픽이 많으면 DB가 모든 중복 요청과 초과 요청을 직접 맞아야 한다.

그래서 Redis를 앞단에 둔다.

[Client]
   |
   v
[API]
   |
   v
[Redis]
   |  중복 요청 빠르게 차단
   |  수량 초과 빠르게 차단
   v
[Kafka]
   |
   v
[Consumer]
   |
   v
[DB unique constraint]

Redis 역할:
- 빠른 1차 필터
- 중복 요청 대부분 제거
- 수량 초과 요청 대부분 제거
- DB로 가는 요청량 감소

DB unique 역할:
- Redis 장애
- Kafka 재전달
- Consumer 재시도
- 동시성 경쟁
- Redis key 유실

이런 상황에서 최종 중복 저장을 막는 마지막 방어선

==================================================
7. 정리하면 구조는 이렇게 가져가는 게 좋다
==================================================

가장 현실적인 구조:

[Redis]
1차 방어선
- 빠른 중복 차단
- 빠른 수량 차단
- DB 부하 감소

[DB unique constraint]
최종 방어선
- 실제 발급 기록 저장
- 절대 중복 발급 방지
- Redis가 놓친 중복도 차단

deduplication table
선택적 방어선
- 비즈니스 테이블만으로 중복 판단이 어려울 때 사용
- 여러 종류의 메시지를 공통으로 멱등 처리해야 할 때 사용
- "메시지 처리 여부" 자체를 추적해야 할 때 사용

==================================================
8. 그러면 deduplication 테이블은 언제 쓰나?
==================================================

쿠폰 발급처럼 중복의 의미가 명확하면
보통 비즈니스 테이블 unique가 더 자연스럽다.

예:

한 유저는 한 이벤트에서 쿠폰을 한 번만 받을 수 있다.

그러면:

UNIQUE(event_id, user_id)

이게 가장 깔끔하다.

하지만 이런 경우에는 deduplication 테이블이 유용할 수 있다.

예:

1. 메시지 타입이 여러 개다.
   - coupon.issue.requested
   - coupon.cancel.requested
   - notification.send.requested
   - point.pay.requested

2. 각 메시지의 비즈니스 테이블이 다르다.

3. 공통적으로 request_id 기준으로 한 번만 처리하고 싶다.

그러면:

processed_messages
- message_id
- message_type
- processed_at

같은 공통 dedup 테이블을 둘 수 있다.

==================================================
9. 네가 말한 의문에 대한 답
==================================================

질문:
"DB에서 unique constraint를 똑같이 한다면 deduplication 테이블 단점이 그대로 있는 거 아니야?"

답:
일부는 맞고, 일부는 다르다.

맞는 부분:
- DB write가 발생하는 것은 맞다.
- unique index 비용도 있다.
- 트래픽이 많으면 DB 병목이 될 수 있다.
- 같은 event_id에 요청이 몰리면 인덱스 경합이 생길 수 있다.

다른 부분:
- deduplication 테이블처럼 추가 write가 생기는 것은 아니다.
- 별도 dedup 기록을 정리할 필요가 줄어든다.
- 실제 비즈니스 데이터 저장과 중복 방지를 한 번에 처리한다.
- "중복 방지만을 위한 테이블 병목"은 피할 수 있다.

==================================================
핵심 정리
==================================================

1. deduplication 테이블 방식은
   중복 방지를 위해 별도 테이블에 매번 write한다.

2. 비즈니스 테이블 unique constraint 방식은
   어차피 저장해야 하는 발급 기록에 unique를 거는 것이다.

3. 그래서 둘 다 DB 비용은 있지만,
   deduplication 테이블은 추가 비용이고,
   unique constraint는 최종 저장 과정에 포함된 비용이다.

4. Redis는 DB unique를 대체하는 게 아니라
   DB에 도달하는 중복/초과 요청을 줄이는 1차 필터다.

5. DB unique constraint는 Redis가 놓친 중복을 막는 최종 방어선이다.

한 줄 요약:

Redis = 빠르게 대부분 걸러주는 앞단 필터
DB unique = 절대 중복 저장을 막는 마지막 방어선
deduplication table = 메시지 처리 여부를 별도로 추적해야 할 때 쓰는 선택지
```
