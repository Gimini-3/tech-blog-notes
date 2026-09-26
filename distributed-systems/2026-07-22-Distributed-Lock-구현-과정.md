# Distributed Lock 구현 과정

- 읽은 날짜: 2026-07-22
- 회사: 채널톡
- 원문: [Distributed Lock 구현 과정](https://tech.channel.io/kr/articles/fd7094b7)

## 1. 왜 분산 락이 필요할까?

하나의 서버 안에서 여러 스레드가 같은 자원에 접근한다면 Java의 `synchronized`나 `ReentrantLock`으로 동시 실행을 막을 수 있다.

```java
public synchronized void processMessage(String messageId) {
    // 동일한 JVM 안에서는 한 번에 하나의 스레드만 실행한다.
    messageService.process(messageId);
}
```

하지만 서버가 여러 대라면 각 서버는 서로 다른 JVM과 메모리를 사용한다.

```
서버 A의 synchronized       서버 B의 synchronized
        │                           │
        ▼                           ▼
   작업 실행 가능               작업 실행 가능
```

서버 A에서 락을 획득해도 서버 B는 그 사실을 알 수 없다. 결국 같은 메시지를 두 서버가 동시에 처리하거나, 동일한 결제가 두 번 실행되는 문제가 발생할 수 있다.

```
서버 A ── 메시지 100 처리 ──▶ DB 저장
서버 B ── 메시지 100 처리 ──▶ DB 저장

결과: 중복 저장 또는 중복 이벤트 발생
```

따라서 여러 서버가 함께 접근할 수 있는 Redis, 데이터베이스, ZooKeeper 같은 외부 시스템에 락을 저장해야 한다. 이를 **분산 락, Distributed Lock**이라고 한다.

---

## 2. 분산 락을 구현하는 대표적인 방법

분산 락은 크게 Redis, 데이터베이스, ZooKeeper를 이용해 구현할 수 있다.

### Redis

Redis의 `SET NX EX` 명령을 사용해 락이 존재하지 않을 때만 값을 저장한다.

장점은 빠르고 구현이 비교적 단순하며, 만료 시간을 설정할 수 있다는 것이다. 이미 Redis를 사용 중인 서비스라면 별도의 인프라를 추가하지 않아도 된다.

### 데이터베이스 락

`SELECT ... FOR UPDATE` 같은 Row Lock이나 MySQL의 User-Level Lock을 사용할 수 있다.

```sql
BEGIN;

-- 동일한 행을 다른 트랜잭션이 수정하지 못하도록 잠근다.
SELECT *
FROM external_messages
WHERE external_id = 'message-100'
FOR UPDATE;

UPDATE external_messages
SET status = 'PROCESSING'
WHERE external_id = 'message-100';

COMMIT;
```

별도 인프라가 필요 없다는 장점이 있지만, 락을 기다리는 요청이 DB 커넥션을 계속 점유할 수 있다.

```
Connection 1: 락 획득 후 작업 중
Connection 2: 락 대기
Connection 3: 락 대기
Connection 4: 락 대기
```

락 경쟁이 심하면 커넥션 풀이 고갈되고 일반적인 조회 요청까지 영향을 받을 수 있다.

### ZooKeeper

ZooKeeper는 세션, Heartbeat, 임시 노드를 이용해 분산 시스템의 상태를 조정한다.

클라이언트의 세션이 만료되면 해당 클라이언트가 생성한 임시 노드가 자동으로 제거되기 때문에 안정성이 중요한 환경에서 고려할 수 있다.

다만 ZooKeeper 자체가 무겁고 클러스터 운영과 장애 대응, 성능 튜닝에 대한 학습 비용이 크다.

채널톡은 이미 캐시와 소켓 처리를 위해 Redis를 사용하고 있었으며, 락 요청량도 적지 않을 것으로 예상했다. 따라서 Redis를 이용해 분산 락을 구현했다.

---

## 3. Redis에서 락을 획득하는 방법

Redis에서는 다음 명령으로 락을 획득할 수 있다.

```
SET distributed-lock:message:100 client-a NX EX 10
```

각 옵션의 의미는 다음과 같다.

```
SET      값을 저장한다.
NX       Key가 존재하지 않을 때만 저장한다.
EX 10    10초 후 Key를 자동으로 삭제한다.
```

두 서버가 동시에 락을 요청하더라도 Redis가 명령을 순차적으로 처리하므로 하나의 요청만 성공한다.

```
서버 A: SET ... NX EX → 성공
서버 B: SET ... NX EX → 실패
```

Java 코드로 표현하면 다음과 같다.

```java
public String tryAcquire(String lockKey, Duration expireTime) {

    // 모든 클라이언트가 서로 다른 값을 사용해야 한다.
    // 보통 UUID나 서버·요청 식별자를 사용한다.
    String uniqueId = UUID.randomUUID().toString();

    String result = redis.set(
            lockKey,
            uniqueId,
            SetArgs.Builder
                    .nx()                       // Key가 없을 때만 저장
                    .ex(expireTime.toSeconds()) // 지정한 시간 후 자동 만료
    );

    if (result != null) {
        // SET 명령이 성공했으므로 락 획득에 성공했다.
        return uniqueId;
    }

    // 이미 다른 클라이언트가 락을 보유하고 있다.
    return "LOCK_ACQUIRE_FAILED";
}
```

여기서 중요한 점은 락 생성과 만료 시간 설정을 하나의 명령으로 처리해야 한다는 것이다.

다음처럼 `SETNX`와 `EXPIRE`를 따로 실행하면 안 된다.

```java
if (redis.setNx(lockKey, uniqueId)) {

    // 이 시점에 서버가 종료되면
    // 만료 시간이 없는 락이 Redis에 영구적으로 남을 수 있다.
    redis.expire(lockKey, 10);
}
```

```
SETNX 성공
    │
    ▼
서버 장애 발생
    │
    ▼
EXPIRE 실행 실패
    │
    ▼
영구적으로 해제되지 않는 락 발생
```

따라서 `SET + NX + EX`를 하나의 원자적인 명령으로 실행해야 한다.

---

## 4. 락의 값이 클라이언트마다 달라야 하는 이유

락의 Key뿐 아니라 Value도 중요하다.

```
Key   : distributed-lock:message:100
Value : client-a-uuid
```

모든 클라이언트가 단순히 `LOCKED`라는 같은 값을 사용하면 자신이 획득하지 않은 락을 삭제할 수 있다.

```
1. 서버 A가 락 획득
2. 서버 A의 작업이 예상보다 오래 걸림
3. 락 TTL이 만료됨
4. 서버 B가 같은 Key로 새로운 락 획득
5. 서버 A가 뒤늦게 작업을 마치고 Key 삭제
6. 서버 B의 락이 잘못 삭제됨
```

따라서 락을 해제할 때는 현재 Redis에 저장된 값이 자신이 발급받은 `uniqueId`와 같은지 확인해야 한다.

---

## 5. 락을 안전하게 해제하는 방법

단순히 `GET`으로 값을 확인한 뒤 `DEL`을 실행하면 두 명령 사이에 다른 클라이언트가 락을 획득할 수 있다.

```java
if (redis.get(lockKey).equals(uniqueId)) {
    // GET과 DEL 사이에 락 소유자가 바뀔 수 있다.
    redis.del(lockKey);
}
```

따라서 값 비교와 삭제를 하나의 원자적 연산으로 실행해야 한다. Redis에서는 Lua Script를 사용할 수 있다.

```lua
-- KEYS[1]: 삭제할 락의 Key
-- ARGV[1]: 락 획득 시 발급받은 클라이언트 식별자
-- ARGV[2]: 락 해제 사실을 알릴 Pub/Sub 채널

if redis.call('get', KEYS[1]) == ARGV[1] then

    -- 현재 락의 소유자가 요청한 클라이언트와 같을 때만 삭제한다.
    redis.call('del', KEYS[1])

    -- 해당 락을 기다리는 클라이언트에게 해제 사실을 전달한다.
    redis.call('publish', ARGV[2], KEYS[1])

    return 1
end

-- 다른 클라이언트가 소유한 락이므로 삭제하지 않는다.
return 0
```

이 스크립트는 다음 세 작업을 원자적으로 처리한다.

```
1. 현재 락의 Value 확인
2. 본인의 락이면 Key 삭제
3. 락 해제 메시지 Publish
```

Java에서는 다음과 같은 형태로 호출할 수 있다.

```java
public boolean release(
        String lockKey,
        String uniqueId
) {
    Long result = redis.eval(
            RELEASE_SCRIPT,
            List.of(lockKey),
            List.of(
                    uniqueId,
                    "distributedLockChannel"
            )
    );

    // 1이면 정상적으로 자신의 락을 삭제한 것이다.
    return result != null && result == 1L;
}
```

---

## 6. 락 획득에 실패하면 어떻게 기다릴까?

가장 단순한 방법은 락을 획득할 때까지 반복 요청하는 것이다.

```java
while (!tryAcquire(lockKey)) {

    // 일정 시간 대기한 후 다시 Redis에 요청한다.
    Thread.sleep(100);
}
```

이를 스핀 락이라고 부른다.

하지만 락을 기다리는 클라이언트가 많으면 Redis에 불필요한 요청이 계속 발생한다.

```
대기 클라이언트 1,000개
재시도 간격 100ms

1초당 약 10,000번의 락 획득 요청 발생
```

채널톡은 이러한 재시도를 줄이기 위해 Redis Pub/Sub를 이용했다.

```
서버 A: 락 보유
서버 B: 락 획득 실패 후 대기 등록
서버 C: 락 획득 실패 후 대기 등록

서버 A: 락 해제
    │
    ▼
Redis Publish
    │
    ├─ 서버 B 재시도
    └─ 서버 C 재시도
```

락 획득에 실패한 작업은 메모리의 대기 목록에 등록한다.

```java
public void waitForLock(
        String lockKey,
        Runnable waitingTask
) {
    // 같은 lockKey를 기다리는 작업들을 저장한다.
    waitingRunnableMap.put(lockKey, waitingTask);
}
```

각 애플리케이션 인스턴스는 Redis Pub/Sub 채널을 구독한다.

락 하나마다 Redis 구독 연결을 만들지 않고, 애플리케이션 서버 한 대가 Redis 구독 하나를 만들어 여러 락 해제 메시지를 한꺼번에 받는다.

```
Application Instance A
└─ Redis Pub/Sub Connection 1
   └─ distributed-lock:* 패턴 구독
      ├─ account:1 락 해제 메시지
      ├─ account:2 락 해제 메시지
      ├─ order:100 락 해제 메시지
      └─ coupon:50 락 해제 메시지
```

```java
public void subscribe() {

    /*
     * Redis Pub/Sub에서 특정 채널 하나만 구독하는 것이 아니라,
     * "distributed-lock:*" 패턴과 일치하는 모든 채널을 구독한다.
     *
     * 예를 들어 다음 채널들이 모두 구독 대상이 된다.
     *
     * distributed-lock:account:1
     * distributed-lock:account:2
     * distributed-lock:order:100
     * distributed-lock:coupon:50
     *
     * 즉, 락 키마다 별도의 subscribe를 호출하거나
     * 별도의 Redis Connection을 생성할 필요가 없다.
     *
     * 애플리케이션 인스턴스마다 Pub/Sub 전용 Connection 하나를 유지하고,
     * 그 Connection이 distributed-lock:* 패턴에 해당하는
     * 모든 락 해제 메시지를 수신한다.
     */
    redisPubSub
            .psubscribe("distributed-lock:*")
            .subscribe();

    /*
     * 위에서 구독한 패턴에 해당하는 Redis Pub/Sub 메시지를 관찰한다.
     *
     * 어떤 서버가 락을 해제하면 다음과 같은 메시지를 발행한다고 가정한다.
     *
     * PUBLISH distributed-lock:account:1 account:1
     *
     * 그러면 distributed-lock:* 패턴을 구독하고 있는
     * 모든 애플리케이션 인스턴스가 이 메시지를 받는다.
     */
    redisPubSub
            .observePatterns()

            /*
             * Redis에서 메시지가 도착할 때마다 실행된다.
             */
            .doOnNext(message -> {

                /*
                 * 메시지에서 어떤 락이 해제되었는지 식별한다.
                 *
                 * 구현에 따라 다음 중 하나일 수 있다.
                 *
                 * 1. 실제 Redis 채널 이름
                 *    distributed-lock:account:1
                 *
                 * 2. 메시지 본문에 들어 있는 락 키
                 *    account:1
                 *
                 * 현재 코드에서는 getMessage()가
                 * waitingRunnableMap의 key와 동일한 값을 반환한다고 가정한다.
                 */
                String lockKey = message.getMessage();

                /*
                 * 현재 애플리케이션 인스턴스에서
                 * 해당 락이 풀리기를 기다리고 있던 작업을 찾는다.
                 *
                 * 예:
                 *
                 * waitingRunnableMap
                 * ├─ account:1 → account:1 락 재획득 작업
                 * └─ order:100 → order:100 락 재획득 작업
                 *
                 * Redis는 "어떤 스레드가 어떤 락을 기다리는지" 알지 못한다.
                 * 그래서 애플리케이션 메모리에 락 키와 재시도 작업을 연결해 둔다.
                 */
                Runnable waitingTask =
                        waitingRunnableMap.get(lockKey);

                /*
                 * 현재 인스턴스에 해당 락을 기다리는 작업이 존재하면
                 * 그 작업을 실행한다.
                 *
                 * 여기서 waitingTask.run()이 곧바로 임계 구역을 실행한다는 뜻은 아니다.
                 *
                 * 일반적으로 waitingTask는 다음 작업을 수행한다.
                 *
                 * 1. Redis에 다시 락 획득 요청
                 * 2. 락 획득 성공 여부 확인
                 * 3. 성공하면 임계 구역 실행
                 * 4. 실패하면 다시 대기
                 *
                 * Pub/Sub 메시지는 단지
                 * "락이 풀렸을 가능성이 있으니 다시 시도해 봐라"
                 * 라는 알림이다.
                 */
                if (waitingTask != null) {

                    // 락 획득을 다시 시도한다.
                    waitingTask.run();
                }
            })

            /*
             * observePatterns()가 반환한 메시지 스트림을 실제로 구독한다.
             *
             * Reactor 계열 API에서는 subscribe()를 호출하지 않으면
             * 위 doOnNext 로직이 실행되지 않는다.
             */
            .subscribe();
}
```

각 락마다 별도의 Redis Connection을 생성하지 않고, 인스턴스 단위로 Pattern Subscribe Connection을 유지해 커넥션 부담을 줄였다.

---

## 7. Pub/Sub 메시지가 유실되면 어떻게 할까?

Redis Pub/Sub는 메시지를 영구 저장하지 않는다.

다음과 같은 상황에서는 락 해제 메시지를 받지 못할 수 있다.

```
- 애플리케이션 네트워크 장애
- Redis 연결 장애
- 애플리케이션 인스턴스 재시작
- 긴 GC Stop-The-World
- 락이 DEL이 아니라 TTL 만료로 제거된 경우
```

특히 락을 가진 서버가 정상적으로 `release()`를 호출하지 못하고 TTL에 의해 락이 사라지면 Publish 메시지가 발생하지 않는다.

```
서버 A 락 획득
    │
    ▼
서버 A 장애
    │
    ▼
TTL 만료로 락 자동 삭제
    │
    ▼
Publish 메시지 없음
```

따라서 Pub/Sub만 믿어서는 안 된다.

채널톡은 락의 만료 시간이 지나도 메시지를 받지 못했다면 다시 락 획득을 시도하는 보완 로직을 추가했다.

```java
public void scheduleRetry(
        String lockKey,
        Duration expireTime,
        Runnable waitingTask
) {
    scheduler.schedule(
            () -> {
                // 아직 대기 목록에 남아 있다면
                // 락 해제 메시지를 받지 못한 것으로 판단한다.
                if (waitingRunnableMap.contains(lockKey, waitingTask)) {
                    waitingTask.run();
                }
            },
            expireTime.toMillis(),
            TimeUnit.MILLISECONDS
    );
}
```

전체 재시도 구조는 다음과 같다.

```
락 획득 시도
    │
    ├─ 성공 ──▶ 작업 실행 ──▶ Lua Script로 해제 및 Publish
    │
    └─ 실패
         │
         ├─ 대기 목록 등록
         ├─ Pub/Sub 메시지 대기
         └─ TTL 이후 보완 재시도 예약
```

Pub/Sub는 빠른 재시도를 담당하고, TTL 기반 예약 작업은 메시지 유실을 보완한다.

---

## 8. Single Redis 분산 락의 한계

이 구조는 하나의 Redis 인스턴스를 락 저장소로 사용한다.

따라서 Redis 장애가 발생하면 락을 획득하거나 해제할 수 없다.

```
서버 A ─┐
서버 B ─┼──▶ Single Redis 장애
서버 C ─┘
```

이때 서비스는 정책을 결정해야 한다.

```
Fail-open
Redis 장애 시 락 없이 작업을 실행한다.
→ 가용성은 높지만 중복 실행 가능성이 있다.

Fail-closed
Redis 장애 시 작업을 실행하지 않는다.
→ 정합성은 지키지만 서비스가 중단될 수 있다.

Retry
일정 시간 이후 다시 락 획득을 시도한다.
→ 일시적 장애에는 대응할 수 있지만 지연이 증가한다.
```

어떤 정책이 정답인지는 업무 성격에 따라 달라진다.

추천 메시지 생성처럼 중복 실행이 어느 정도 허용된다면 Fail-open을 고려할 수 있지만, 결제나 재고 차감처럼 중복 실행이 치명적이라면 Fail-closed 또는 재시도가 더 적합하다.

---

## 9. Redis Replica가 있어도 락이 유실될 수 있다

Redis를 Master-Replica로 구성하더라도 복제는 일반적으로 비동기적으로 이루어진다.

```
1. Master에 락 저장
2. 클라이언트에게 성공 응답
3. Replica로 복제되기 전에 Master 장애
4. Replica가 새로운 Master로 승격
5. 새 Master에는 락 정보가 없음
```

```
기존 Master
lock:message:100 = client-a
        │
        │ 복제 전 장애
        ▼
새로운 Master
lock:message:100 없음
```

이 상태에서 다른 서버가 새로운 락을 획득하면 두 클라이언트가 동시에 작업을 실행할 수 있다.

---

## 10. Redlock을 사용하면 해결될까?

Redlock은 여러 개의 독립적인 Redis 인스턴스에 락 획득을 시도하는 알고리즘이다.

```
Redis 1 ── 락 성공
Redis 2 ── 락 성공
Redis 3 ── 락 성공
Redis 4 ── 락 실패
Redis 5 ── 락 실패

과반수 성공 → 락 획득으로 판단
```

일반적인 흐름은 다음과 같다.

```
1. 락 획득 시작 시간을 기록한다.
2. 여러 Redis에 순차적으로 락 획득을 요청한다.
3. 과반수 Redis에서 락을 얻었는지 확인한다.
4. 전체 소요 시간이 락 TTL보다 짧은지 확인한다.
5. 조건을 만족하지 못하면 획득한 락을 모두 해제한다.
```

하지만 Redis 인스턴스를 여러 개 운영해야 하며 네트워크 요청도 증가한다. 시스템 시간과 네트워크 지연을 고려해야 해 구현과 운영 복잡도도 높아진다.

채널톡은 Single Redis보다 훨씬 높은 안정성이 필요하다면 Redlock을 복잡하게 운영하기보다는ㅔ, 분산 조정 용도로 설계된 ZooKeeper Cluster를 고려하는 편이 더 적절하다고 판단했다.

---

## 11. 순서 보장은 별개의 문제다

Redis의 `SET NX` 락은 대기 요청의 순서를 보장하지 않는다.

```
요청 순서: A → B → C

락 획득 순서: A → C → B 가능
```

락이 해제된 순간 네트워크와 스레드 스케줄링 상황에 따라 먼저 Redis에 요청한 클라이언트가 락을 가져간다.

FIFO 순서가 필요하다면 Redis List나 Sorted Set을 이용해 대기열을 별도로 구성하거나, ZooKeeper의 Sequential Node 같은 기능을 사용해야 한다.

하지만 순서 보장을 추가하면 구현 복잡도와 처리 지연이 커진다. 채널톡의 사용 사례에서는 순서 보장이 필요하지 않았기 때문에 구현하지 않았다.

---

## 결론

분산 락은 단순히 Redis에 Key 하나를 저장하는 기능이 아니다.

안전한 분산 락을 구현하려면 다음 문제를 모두 고려해야 한다.

```
1. 락 획득은 SET NX EX로 원자적으로 처리한다.

2. 락마다 고유한 클라이언트 식별자를 저장한다.

3. 락 해제 시 소유자를 확인한다.

4. 값 확인과 삭제는 Lua Script로 원자적으로 처리한다.

5. 스핀 락 대신 Pub/Sub를 이용해 재시도를 줄인다.

6. Pub/Sub 메시지 유실을 고려해 TTL 기반 재시도를 추가한다.

7. Redis 장애 시 Fail-open, Fail-closed, Retry 정책을 결정한다.

8. Master-Replica 구성에서도 비동기 복제로 락이 유실될 수 있다.

9. Redlock과 ZooKeeper는 안정성을 높이지만 운영 복잡도도 커진다.

10. 상호 배제와 요청 순서 보장은 서로 다른 요구사항이다.
```

채널톡은 이미 Redis를 운영하고 있었고, 높은 락 요청량이 예상됐으며, 해당 작업에서 완벽한 순서 보장이나 강한 장애 허용성이 필요하지 않았다.

따라서 다음 구조를 선택했다.

```
SET NX EX
    +
고유한 Lock Value
    +
Lua Script 기반 Release
    +
Redis Pub/Sub 재시도
    +
TTL 기반 보완 재시도
```

핵심은 Redis가 항상 최선이라는 것이 아니다.

```
데이터 변경과 하나의 트랜잭션으로 묶어야 한다.
→ DB Lock 고려

높은 처리량과 빠른 락이 필요하고 Redis를 이미 사용한다.
→ Redis Lock 고려

결제처럼 속도보다 강한 분산 조정과 안정성이 중요하다.
→ ZooKeeper 같은 조정 시스템 고려
```

분산 락을 선택할 때는 구현 편의성만 볼 것이 아니라, 장애가 발생했을 때 중복 실행과 작업 중단 중 무엇을 허용할 수 있는지를 먼저 결정해야 한다.

---

## 느낀점

Redis 분산 락은 단순히 `SET NX EX`로 키를 생성하는 것만으로 끝나는 기능이 아니었다. 락 소유권 확인, 안전한 해제, 메시지 유실, Redis 장애와 복제 지연까지 함께 고려해야 실제 서비스에서 사용할 수 있다는 점을 알게 되었다. 특히 Pub/Sub를 활용해 불필요한 재시도를 줄이면서도 TTL 기반 재시도로 메시지 유실을 보완한 구조가 인상 깊었다. 결국 중요한 것은 특정 기술을 사용하는 것이 아니라, 장애 상황에서 중복 실행과 작업 중단 중 무엇을 허용할지 먼저 결정하는 것이라고 느꼈다.
