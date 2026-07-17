# Kafka Consumer 멱등성 처리: 중복 메시지를 안전하게 다루는 방법

- 읽은 날짜: 2026-07-07
- 회사/작성 주체: Conduktor
- 원문: [Kafka Consumer 멱등성 처리: 중복 메시지를 안전하게 다루는 방법](https://www.conduktor.io/blog/building-idempotent-consumers)

---

## 중복 메시지가 발생하는 이유

Kafka Consumer는 같은 메시지를 두 번 받을 수 있다. 이는 버그라기보다 at-least-once 전달 방식에서 자연스럽게 발생하는 상황이다.

- Producer가 메시지를 전송하고 Broker가 저장했지만 ACK가 유실되면 Producer가 재전송할 수 있다.
- Consumer가 비즈니스 처리를 마친 뒤 offset을 commit하기 전에 종료되면 재시작 후 같은 메시지를 다시 받는다.
- Consumer Group rebalance 과정에서 처리됐지만 아직 commit되지 않은 메시지가 재전달될 수 있다.

`enable.idempotence=true`는 Producer 재시도로 Broker에 중복 레코드가 생기는 문제를 줄이지만, Consumer가 같은 논리적 이벤트를 다시 처리하는 문제까지 해결하지는 않는다. 따라서 Consumer는 같은 메시지를 여러 번 받아도 비즈니스 결과가 한 번만 반영되도록 설계해야 한다.

## 기본 패턴과 원자성

멱등 Consumer의 기본 흐름은 다음과 같다.

1. 메시지에서 멱등성 키를 추출한다.
2. 이미 처리한 키인지 확인한다.
3. 처음 받은 메시지라면 비즈니스 로직을 실행한다.
4. 처리 완료 상태를 기록한다.

이 단계들을 각각 실행하면 중간 장애가 발생할 수 있다. 예를 들어 주문 저장은 성공했지만 처리 완료 기록 전에 프로세스가 종료되면, 재전달된 메시지가 주문을 다시 생성할 수 있다. 비즈니스 처리와 중복 방지 기록을 가능한 한 하나의 원자적 경계로 묶어야 한다.

## 멱등성 키 설계

좋은 멱등성 키는 논리적 작업마다 유일하고, 재시도 사이에서도 변하지 않아야 한다.

- **클라이언트가 생성한 UUID:** API 요청 시 생성하고 전체 이벤트 흐름에서 유지한다.
- **복합 비즈니스 키:** `customerId:orderId:timestamp`처럼 실제로 유일한 속성 조합을 사용한다.
- **Kafka 좌표:** `topic-partition-offset`을 사용한다. 구현은 단순하지만 다른 토픽으로 복사하거나 재발행하면 같은 논리적 이벤트의 키가 달라진다.

가능하다면 Producer가 생성한 비즈니스 식별자를 이벤트에 포함하는 방식이 가장 명확하다.

## 패턴 1: 데이터베이스 Unique Constraint

처리한 메시지의 키를 별도 테이블에 저장하고 Unique Constraint로 중복을 차단한다.

```sql
CREATE TABLE processed_messages (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

중복 방지 레코드와 실제 비즈니스 데이터 변경을 같은 DB 트랜잭션에 포함한다. 비즈니스 처리가 실패하면 중복 방지 레코드도 rollback되어 재처리가 가능해야 한다.

장점은 구조가 단순하고 기존 관계형 데이터베이스의 트랜잭션을 활용할 수 있다는 점이다. 반면 모든 메시지마다 추가 write가 발생하고, 트래픽이 많으면 중복 방지 테이블이 병목이 될 수 있다.

비즈니스 테이블 자체에 적절한 Unique Constraint를 둘 수 있다면 별도 테이블 없이 결과 저장과 중복 방지를 함께 처리할 수도 있다. 다만 외부 API 호출이나 여러 테이블에 걸친 작업처럼 비즈니스 결과만으로 처리 여부를 판단하기 어려운 경우에는 별도 기록이 필요하다.

## 패턴 2: Redis `SET NX`

처리 키를 Redis에 저장하면 높은 처리량으로 중복을 빠르게 걸러낼 수 있다.

```text
SET dedup:request-123 1 NX EX 604800
```

TTL은 최대 replay 가능 기간보다 길게 설정해야 한다. TTL이 끝난 뒤 과거 메시지를 재생하면 Consumer는 처음 받은 메시지로 판단한다.

Redis를 중복 방지의 유일한 저장소로 사용한다면 영속성과 가용성도 고려해야 한다. RDB snapshot 사이에 장애가 발생하면 최근 키가 유실될 수 있으므로 AOF와 복제를 검토해야 한다. Redis 장애 시 Consumer를 중단할지, DB 방식으로 전환할지도 정책으로 정해야 한다.

또한 `SET NX` 성공 직후 비즈니스 처리가 실패하면 키만 남아 정상 재처리를 막을 수 있다. 처리 전 선점 키와 처리 완료 상태를 구분하거나, 실패 시 키를 안전하게 해제하는 등 상태 전이 설계가 필요하다.

## 패턴 3: 자연적 멱등성

이벤트가 현재 상태를 특정 값으로 교체하는 형태라면 별도 중복 저장소 없이 upsert와 버전 비교로 멱등성을 만들 수 있다.

```sql
INSERT INTO inventory (product_id, quantity, event_version)
VALUES (?, ?, ?)
ON CONFLICT (product_id)
DO UPDATE SET
    quantity = EXCLUDED.quantity,
    event_version = EXCLUDED.event_version
WHERE inventory.event_version < EXCLUDED.event_version;
```

같은 이벤트를 여러 번 적용해도 최종 값이 같고, 오래된 이벤트가 최신 상태를 덮어쓰는 것도 막을 수 있다. 반면 수량을 10 증가시키는 delta 연산, 이메일 발송, 결제 요청처럼 실행 횟수 자체가 결과에 영향을 주는 작업에는 적용하기 어렵다.

## 방식 선택 기준

| 방식                 | 적합한 상황                                    | 주요 비용과 위험                                |
| -------------------- | ---------------------------------------------- | ----------------------------------------------- |
| DB Unique Constraint | 기존 DB에 결과를 저장하는 일반적인 CRUD 서비스 | 추가 write, 인덱스 경합과 테이블 정리           |
| Redis `SET NX`       | 높은 처리량과 빠른 중복 판정이 필요한 서비스   | Redis 의존성, 영속성, 처리 실패 후 상태 관리    |
| 자연적 멱등성        | 상태 교체와 버전 비교가 가능한 이벤트          | delta 연산과 외부 부수 효과에는 적용하기 어려움 |

대부분의 Kafka-to-DB 서비스는 DB Constraint 방식으로 충분하다. 처리량 때문에 실제 병목이 확인되었을 때 Redis를 추가하거나, 도메인 모델이 허용한다면 자연적 멱등성을 사용하는 편이 단순하다.

## Deduplication Window와 테스트

중복 기록의 보관 기간은 다음 기준으로 정한다.

- Consumer가 중단될 수 있는 최대 시간
- 장애 복구나 DLQ 처리로 과거 메시지를 replay할 수 있는 기간
- Producer retry가 지속될 수 있는 시간
- 법적·업무적으로 같은 요청을 다시 처리하면 안 되는 기간

테스트에서는 같은 메시지를 순차적으로 두 번 처리하는 경우뿐 아니라, 두 Consumer가 같은 키를 동시에 처리하는 경쟁 상황도 검증해야 한다. 최종 비즈니스 결과가 하나만 남고 실패한 처리는 안전하게 재시도할 수 있어야 한다.

## 느낀점

Kafka의 중복 전달을 없애려고 하기보다 중복이 와도 안전한 Consumer를 만드는 것이 핵심이라는 점이 인상 깊었다. 특히 중복 확인과 실제 처리를 따로 구현하면 그 사이의 장애가 새로운 불일치를 만든다. 어떤 저장소를 선택하든 원자성, 장애 복구와 replay 기간을 함께 설계해야 한다.
