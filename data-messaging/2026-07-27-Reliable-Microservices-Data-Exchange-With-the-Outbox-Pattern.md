# Reliable Microservices Data Exchange With the Outbox Pattern

- 읽은 날짜: 2026-07-27
- 회사/작성 주체: Debezium
- 원문: [Reliable Microservices Data Exchange With the Outbox Pattern](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)

---

## 들어가며

마이크로서비스를 구성하다 보면 하나의 서비스가 자신의 데이터만 저장하고 끝나는 경우는 많지 않다.
주문 서비스에서 주문이 생성되면 배송 서비스는 배송 준비를 시작해야 하고, 고객 서비스는 고객의 구매 금액이나 포인트를 갱신해야 한다. 검색 시스템이나 데이터 웨어하우스도 주문 데이터를 받아 별도의 조회 모델을 만들 수 있다.
처음에는 다른 서비스를 직접 호출하는 방식으로 구현할 수 있다.

```text
주문 서비스
→ 배송 서비스 REST API 호출
→ 고객 서비스 REST API 호출
```

구조가 단순해 보이지만 서비스 수가 늘어나면 문제가 생긴다.
주문 서비스는 어떤 서비스를 호출해야 하는지 알아야 하고, 상대 서비스의 주소와 API 형식도 알고 있어야 한다. 배송 서비스가 일시적으로 장애 상태라면 주문 처리까지 실패할 수 있다.
하나의 서비스 장애가 다른 서비스의 정상 동작에 직접 영향을 주는 구조가 된다.
이 문제를 줄이기 위해 Kafka와 같은 메시지 브로커를 사용할 수 있다.

```text
주문 서비스
→ OrderCreated 이벤트 발행
→ Kafka
→ 배송 서비스
→ 고객 서비스
→ 검색 서비스
```

주문 서비스는 누가 이벤트를 사용하는지 몰라도 된다. 이벤트를 Kafka에 저장하기만 하면 필요한 서비스가 각자 구독해 처리한다.
하지만 여기서 더 어려운 문제가 생긴다.
주문 서비스는 주문 정보를 자신의 데이터베이스에 저장해야 하고, 동시에 Kafka에도 이벤트를 발행해야 한다.

```text
주문 DB 저장

+

Kafka 이벤트 발행
```

두 작업 중 하나만 성공하면 시스템의 데이터가 서로 어긋난다.
이 문제를 해결하기 위해 사용하는 대표적인 방식이 **Transactional Outbox Pattern**이다.

---

## 1. 서비스 간 동기 호출의 한계

주문 서비스가 새로운 주문을 처리한다고 가정해 보자.
주문이 완료되면 배송 서비스와 고객 서비스에 결과를 알려야 한다.
가장 직접적인 방법은 REST나 gRPC를 이용해 다른 서비스를 호출하는 것이다.

```java
@Transactional
public PurchaseOrder createOrder(
        CreateOrderRequest request
) {
    PurchaseOrder order = orderRepository.save(
        PurchaseOrder.from(request)
    );

    shipmentClient.createShipment(order);
    customerClient.updateBalance(order);

    return order;
}
```

코드는 이해하기 쉽지만 주문 서비스가 여러 외부 서비스에 의존하게 된다.
배송 서비스가 응답하지 않으면 주문 요청도 오래 대기할 수 있다. 고객 서비스 호출이 실패하면 앞에서 성공한 배송 요청을 어떻게 취소할지도 결정해야 한다.
호출 대상이 늘어나면 실패 처리도 복잡해진다.

```text
배송 호출 성공
고객 호출 실패

→ 배송을 취소해야 하는가
→ 전체 주문을 실패로 처리해야 하는가
→ 고객 요청만 재시도해야 하는가
```

서비스 간 연결이 강해질수록 하나의 기능을 변경할 때 영향 범위도 커진다.
동기 호출은 요청 처리 시점에 반드시 결과가 필요한 경우에는 적합하다.
예를 들어 주문을 생성하기 전에 재고 수량을 반드시 확인해야 한다면 재고 서비스의 응답이 필요할 수 있다.
반대로 주문 완료 사실을 알림 서비스나 분석 서비스에 전달하는 작업은 즉시 완료되지 않아도 된다.
이런 후속 작업은 비동기 이벤트로 분리하는 것이 자연스럽다.

---

## 2. Kafka를 이용한 비동기 데이터 전달

Kafka를 사용하면 주문 서비스가 다른 서비스를 직접 호출하지 않아도 된다.
주문 서비스는 주문이 생성되었다는 이벤트를 Kafka에 발행한다.

```text
Order Service
     |
     | OrderCreated
     v
Kafka
     |
     +-- Shipment Service
     +-- Customer Service
     +-- Search Service
     +-- Data Warehouse
```

각 서비스는 필요한 이벤트만 구독한다.
배송 서비스는 배송 정보를 생성하고, 검색 서비스는 조회에 적합한 형태로 주문 데이터를 저장할 수 있다.
이 방식은 서비스 간 결합도를 낮춘다.
주문 서비스는 배송 서비스가 몇 개인지 알 필요가 없다. 이후 새로운 분석 서비스가 추가되더라도 주문 서비스 코드를 수정하지 않고 Kafka Topic을 구독하면 된다.
Kafka는 이벤트를 일정 기간 저장하기 때문에 새로운 Consumer가 나중에 추가되어도 과거 이벤트를 다시 읽을 수 있다.
이 특성을 **Replay 가능성**이라고 한다.

```text
기존 Consumer
→ 주문 이벤트를 실시간 처리

새로운 Consumer
→ Topic의 처음부터 이벤트를 읽음
→ 자신의 데이터베이스에 전체 주문 상태 구성
```

검색 시스템이나 데이터 웨어하우스를 새롭게 추가할 때 유용하다.
하지만 비동기 이벤트를 도입했다고 해서 모든 문제가 해결되는 것은 아니다.
주문 서비스는 여전히 두 시스템에 데이터를 기록해야 한다.

```text
PostgreSQL에 주문 저장

Kafka에 OrderCreated 저장
```

이 두 작업 사이에서 Dual Write 문제가 발생한다.

## 3. Dual Write 문제

Dual Write는 하나의 비즈니스 작업에서 서로 다른 두 저장소에 각각 데이터를 기록하는 상황을 말한다.
주문 생성 로직을 다음처럼 작성할 수 있다.

```java
@Transactional
public PurchaseOrder createOrder(
        CreateOrderRequest request
) {
    // 1. 주문 정보를 데이터베이스에 저장한다.
    PurchaseOrder order = orderRepository.save(
        PurchaseOrder.from(request)
    );

    // 2. 주문 생성 이벤트를 Kafka에 발행한다.
    kafkaTemplate.send(
        "order.created",
        String.valueOf(order.getId()),
        OrderCreatedEvent.from(order)
    );

    return order;
}
```

하나의 메서드에 `@Transactional`이 붙어 있기 때문에 두 작업이 함께 처리되는 것처럼 보인다.
하지만 실제 트랜잭션 범위는 데이터베이스에만 적용된다.

```text
Spring DB Transaction
→ PostgreSQL INSERT 관리

Kafka Broker
→ 별도의 시스템
→ DB Transaction에 자동 포함되지 않음
```

PostgreSQL과 Kafka는 서로 독립적인 시스템이다.
데이터베이스 트랜잭션이 Rollback된다고 해서 이미 Kafka에 저장된 메시지까지 자동으로 사라지지 않는다.
반대로 Kafka 발행에 실패했다고 해서 데이터베이스 Commit이 자동으로 취소되는 것도 아니다.
이 때문에 한쪽만 성공하는 상황이 발생할 수 있다.

---

## 4. DB를 먼저 저장하는 경우

첫 번째 방식은 주문 데이터를 먼저 Commit한 뒤 Kafka 이벤트를 발행하는 것이다.

```text
주문 DB Commit
        |
        v
Kafka 이벤트 발행
```

정상 상황에서는 문제가 없다.
하지만 DB Commit 직후 애플리케이션이 종료되면 이벤트 발행이 실행되지 않을 수 있다.

```text
PostgreSQL
→ 주문 저장 성공

Kafka
→ OrderCreated 없음
```

주문은 정상적으로 생성되었지만 배송 서비스는 주문이 생성된 사실을 알 수 없다.
사용자는 주문 내역에서 정상적으로 주문을 확인하지만 배송 작업은 시작되지 않을 수 있다.

```text
사용자
→ 주문 성공 확인

배송 서비스
→ 이벤트를 받지 못함
→ 배송 생성 안 됨
```

서버가 다시 시작되어도 발행해야 했던 이벤트가 메모리에만 있었다면 무엇을 다시 발행해야 하는지 알 수 없다.
데이터베이스에는 주문 정보가 있지만, 어떤 이벤트가 발행되지 않았는지 구분할 기록이 없기 때문이다.

---

## 5. Kafka를 먼저 발행하는 경우

순서를 반대로 바꾸면 문제가 해결될 것처럼 보일 수 있다.

```text
Kafka 이벤트 발행
        |
        v
주문 DB 저장
```

그러나 이번에는 Kafka 발행 후 DB 저장이 실패할 수 있다.

```text
Kafka
→ OrderCreated 존재

PostgreSQL
→ 주문 저장 실패
```

배송 서비스는 Kafka 이벤트를 받아 배송을 생성한다.
하지만 주문 서비스의 데이터베이스에는 주문이 존재하지 않는다.

```text
배송 서비스
→ 배송 생성 완료

주문 서비스
→ 주문 없음
```

실제로 존재하지 않는 주문에 대한 배송이나 알림이 만들어질 수 있다.
따라서 저장 순서를 바꾸는 것만으로는 해결되지 않는다.

```text
DB를 먼저 저장
→ 이벤트 유실 가능

Kafka를 먼저 발행
→ 존재하지 않는 데이터에 대한 이벤트 가능
```

DB Commit과 Kafka 저장 사이에는 항상 장애가 발생할 수 있는 구간이 존재한다.

## 6. 분산 트랜잭션을 사용하면 안 되는가

두 시스템을 하나의 트랜잭션으로 묶는 방법으로 XA 기반 분산 트랜잭션을 떠올릴 수 있다.
분산 트랜잭션은 여러 저장소가 하나의 Commit 결과를 갖도록 조정한다.

```text
Transaction Coordinator
        |
        +-- PostgreSQL
        |
        +-- Message Broker
```

모든 참여자가 준비되면 Commit하고 하나라도 실패하면 Rollback한다.
그러나 Kafka는 일반적인 XA 분산 트랜잭션에 참여하는 구조를 제공하지 않는다.
또한 분산 트랜잭션은 저장소 간 결합을 높이고 성능과 가용성에도 부담을 준다.
하나의 외부 시스템이 느려지면 전체 트랜잭션이 길어질 수 있다.
마이크로서비스에서는 가능한 한 하나의 서비스가 자신의 로컬 데이터베이스 트랜잭션만 책임지도록 설계하는 경우가 많다.
따라서 접근 방식을 바꿔야 한다.

```text
DB와 Kafka를 동시에 변경하려고 하지 않는다.

먼저 하나의 저장소만 안전하게 변경한다.

두 번째 저장소는
첫 번째 저장소의 변경을 기준으로 나중에 반영한다.
```

이것이 Outbox Pattern의 기본 아이디어다.

## 7. Kafka에만 먼저 저장하는 방법

Dual Write를 피하려면 한 번에 하나의 저장소만 변경해야 한다.
한 가지 방법은 주문을 데이터베이스에 바로 저장하지 않고 Kafka에 이벤트만 발행하는 것이다.

```text
주문 요청
→ Kafka OrderCreated 발행
→ Order Service가 자신의 이벤트를 다시 소비
→ 주문 DB 저장
```

처음 요청을 받을 때 Kafka만 변경하므로 Dual Write는 발생하지 않는다.
하지만 이 구조에는 **Read Your Own Writes** 문제가 생긴다.
사용자가 주문을 생성한 직후 주문 내역을 조회한다고 가정해 보자.

```text
1. 주문 생성 요청
2. Kafka 이벤트 발행 성공
3. 주문 생성 응답 반환
4. 사용자가 즉시 주문 내역 조회
5. Kafka Consumer는 아직 DB 저장 전
```

사용자는 주문 생성에 성공했다는 응답을 받았지만 주문 내역에서는 방금 만든 주문을 볼 수 없다.
비동기 처리가 완료되기 전까지 데이터베이스에 주문이 없기 때문이다.
이 상황은 사용자에게 혼란을 준다.

```text
주문은 성공했다고 했는데
왜 주문 내역에는 없는가
```

이를 해결하기 위해 메모리에 임시 주문 정보를 보관하는 방법도 생각할 수 있다.
하지만 서버 인스턴스가 여러 개라면 다른 인스턴스로 조회 요청이 들어올 수 있다. 복잡한 검색 조건까지 지원하려면 임시 상태를 별도로 관리해야 한다.
구조가 빠르게 복잡해진다.
주문과 같이 사용자에게 즉시 조회되어야 하는 데이터는 DB에 동기적으로 저장하는 편이 자연스럽다.

---

## 8. Transactional Outbox Pattern

Transactional Outbox Pattern은 비즈니스 데이터와 발행할 이벤트를 같은 데이터베이스에 저장한다.
주문 서비스에는 주문 테이블과 함께 Outbox 테이블을 둔다.

```text
PostgreSQL

purchase_order
outbox_event
```

주문을 생성할 때 하나의 DB 트랜잭션에서 두 행을 함께 저장한다.

```text
DB Transaction 시작
        |
        v
purchase_order INSERT
        |
        v
outbox_event INSERT
        |
        v
Commit
```

두 작업이 같은 PostgreSQL 트랜잭션에 포함되므로 결과는 두 가지뿐이다.

```text
Commit 성공
→ 주문 저장
→ Outbox 이벤트 저장
```

```text
Rollback
→ 주문 없음
→ Outbox 이벤트 없음
```

주문만 저장되고 이벤트 기록이 없는 상태가 발생하지 않는다.
반대로 이벤트만 존재하고 주문이 없는 상태도 발생하지 않는다.
애플리케이션은 Kafka에 직접 발행하지 않는다.
대신 Kafka에 발행해야 할 이벤트를 Outbox 테이블에 저장한다.

```text
주문 서비스
        |
        | 같은 DB Transaction
        v
purchase_order + outbox_event
        |
        v
별도 프로세스
        |
        v
Kafka
```

여기서 Outbox 행의 의미를 정확히 구분해야 한다.

```text
Outbox 행 존재
≠ Kafka 발행 완료

Outbox 행 존재
= 발행해야 할 이벤트가 DB에 안전하게 기록됨
```

Kafka가 일시적으로 장애 상태여도 Outbox 이벤트는 데이터베이스에 남아 있다.
발행 프로세스가 나중에 다시 읽어 Kafka에 전달할 수 있다.

---

## 9. Outbox가 제공하는 Read Your Own Writes

Transactional Outbox Pattern에서는 주문 데이터가 요청 처리 과정에서 DB에 직접 저장된다.
따라서 주문 생성 트랜잭션이 Commit된 직후 조회하면 새 주문을 확인할 수 있다.

```text
주문 생성 Transaction Commit
        |
        v
주문 조회
        |
        v
방금 생성한 주문 반환
```

Kafka 전파는 비동기로 진행되지만 주문 서비스 내부의 읽기 일관성은 바로 확보된다.

```text
주문 서비스 내부
→ 즉시 일관성
→ Commit 직후 조회 가능

다른 서비스
→ 최종적 일관성
→ 이벤트를 처리한 뒤 상태 반영
```

이를 정리하면 다음과 같다.

```text
Source Service
→ Read Your Own Writes 제공

Downstream Service
→ Eventually Consistent
```

배송 서비스가 주문 이벤트를 수신하는 데 약간의 지연은 있을 수 있다.
하지만 주문이 생성된 직후 같은 순간에 배송 데이터까지 반드시 만들어져야 하는 것이 아니라면 일반적으로 허용할 수 있다.

---

## 10. 주문 생성 코드에 Outbox 적용하기

주문 생성 로직에서는 Kafka를 호출하지 않고 Outbox 이벤트를 저장한다.

```java
@Transactional
public PurchaseOrder addOrder(
        PurchaseOrder order
) {
    // 주문 도메인 데이터를 저장한다.
    PurchaseOrder savedOrder =
        entityManager.merge(order);

    // Kafka에 직접 보내지 않는다.
    // 같은 트랜잭션 안에서 발행할 이벤트를 생성한다.
    event.fire(
        OrderCreatedEvent.of(savedOrder)
    );

    return savedOrder;
}
```

코드에서 `event.fire()`를 사용하지만 이 이벤트가 바로 Kafka에 발행되는 것은 아니다.
애플리케이션 내부의 동기 이벤트다.
이 이벤트를 처리하는 `EventSender`가 Outbox 행을 저장한다.

```java
@ApplicationScoped
public class EventSender {

    @PersistenceContext
    private EntityManager entityManager;

    public void onExportedEvent(
            @Observes ExportedEvent event
    ) {
        // 도메인 이벤트를 Outbox Entity로 변환한다.
        OutboxEvent outboxEvent =
            new OutboxEvent(
                event.getAggregateType(),
                event.getAggregateId(),
                event.getType(),
                event.getPayload()
            );

        // 현재 주문 저장 트랜잭션에 함께 참여한다.
        entityManager.persist(outboxEvent);
    }
}
```

`@Observes` 메서드는 이벤트가 발생한 스레드에서 동기적으로 실행된다.
따라서 `purchase_order` 저장과 `outbox_event` 저장이 같은 트랜잭션에 포함된다.

```text
OrderService.addOrder()
        |
        | entityManager.merge(order)
        |
        | event.fire()
        v
EventSender.onExportedEvent()
        |
        | entityManager.persist(outbox)
        v
같은 DB Transaction Commit
```

여기서 중요한 점은 이벤트를 발행한다는 표현을 그대로 믿으면 안 된다는 것이다.

```text
event.fire()
→ Kafka 발행 아님

event.fire()
→ Outbox INSERT를 실행하기 위한
   애플리케이션 내부 동기 호출
```

실제 Kafka 발행은 이후 Debezium이 처리한다.

## 11. 이벤트 Payload는 외부 계약이다

Outbox에 저장되는 이벤트는 단순히 데이터베이스 행을 복사한 것이 아니다.
다른 서비스가 사용하기 적합한 구조로 만들어야 한다.
주문 생성 이벤트는 다음과 같이 구성할 수 있다.

```java
public class OrderCreatedEvent
        implements ExportedEvent {

    private final long id;
    private final JsonNode order;

    private OrderCreatedEvent(
            long id,
            JsonNode order
    ) {
        this.id = id;
        this.order = order;
    }

    public static OrderCreatedEvent of(
            PurchaseOrder order
    ) {
        ObjectNode payload =
            mapper.createObjectNode()
                .put("id", order.getId())
                .put(
                    "customerId",
                    order.getCustomerId()
                )
                .put(
                    "orderDate",
                    order.getOrderDate()
                        .toString()
                );

        ArrayNode items =
            payload.putArray("lineItems");

        for (OrderLine line : order.getLineItems()) {
            items.add(
                mapper.createObjectNode()
                    .put("id", line.getId())
                    .put("item", line.getItem())
                    .put(
                        "quantity",
                        line.getQuantity()
                    )
                    .put(
                        "totalPrice",
                        line.getTotalPrice()
                    )
                    .put(
                        "status",
                        line.getStatus().name()
                    )
            );
        }

        return new OrderCreatedEvent(
            order.getId(),
            payload
        );
    }

    @Override
    public String getAggregateId() {
        return String.valueOf(id);
    }

    @Override
    public String getAggregateType() {
        return "Order";
    }

    @Override
    public String getType() {
        return "OrderCreated";
    }

    @Override
    public JsonNode getPayload() {
        return order;
    }
}
```

이벤트 Payload는 주문 테이블의 내부 구조와 분리되어 있다.
데이터베이스 컬럼명이 변경되더라도 외부 서비스가 사용하는 이벤트 구조는 유지할 수 있다.

```text
내부 데이터 모델
→ 서비스 구현 세부사항

Outbox 이벤트 Payload
→ 다른 서비스와 공유하는 API 계약
```

따라서 이벤트 스키마를 변경할 때는 일반 API를 변경할 때처럼 호환성을 고려해야 한다.
Consumer가 사용하는 필드를 갑자기 제거하거나 타입을 바꾸면 여러 서비스가 동시에 장애를 겪을 수 있다.

---

## 12. Outbox Pattern이 해결하는 것

Transactional Outbox Pattern이 해결하는 핵심은 다음과 같다.

```text
비즈니스 데이터 Commit

+

이벤트 발행 의도 저장
```

두 작업을 같은 DB 트랜잭션으로 묶는다.
따라서 주문이 저장되었다면 발행해야 할 이벤트도 반드시 DB에 남는다.
Kafka 장애나 애플리케이션 재시작이 발생해도 복구할 기준이 존재한다.

```text
Kafka 장애
→ Outbox 이벤트 DB에 유지
→ 장애 복구 후 재발행

애플리케이션 종료
→ 메모리는 사라짐
→ Outbox 행은 유지
→ 재시작 후 이어서 처리
```

이것이 단순한 `AFTER_COMMIT` 이벤트와 가장 큰 차이다.
`AFTER_COMMIT` Listener는 실행할 작업이 메모리에만 존재할 수 있다.
Outbox는 실행해야 할 작업 자체를 데이터베이스에 영속화한다.

---

## 13. Outbox Pattern이 해결하지 않는 것

Outbox Pattern을 적용했다고 해서 모든 메시지가 정확히 한 번만 전달되는 것은 아니다.
다음 상황이 발생할 수 있다.

```text
Debezium이 Kafka 발행 성공
        |
        X 처리 위치 저장 전 장애
        |
        v
재시작 후 같은 이벤트 다시 발행
```

Consumer는 같은 이벤트를 두 번 받을 수 있다.
따라서 Outbox 기반 이벤트 전달은 일반적으로 다음 의미를 가진다.

```text
At-Least-Once Delivery

→ 최소 한 번 전달
→ 중복 전달 가능
```

Consumer는 이벤트 ID를 이용해 이미 처리한 메시지인지 확인해야 한다.
또한 Outbox는 다른 서비스의 처리 완료까지 보장하지 않는다.

```text
Outbox 저장 성공
≠ Kafka 전송 완료

Kafka 전송 완료
≠ 배송 서비스 처리 완료

배송 서비스 처리 완료
≠ 외부 택배 API 호출 완료
```

각 단계는 독립된 처리 단계를 가진다.
Outbox Pattern의 역할은 명확하다.

```text
Source Service의 DB 변경과
이벤트 발행 의도 사이의 유실을 막는다.
```

---

## 14. Outbox 테이블의 이벤트를 Kafka로 전달하는 방법

1편에서는 주문 데이터와 Outbox 이벤트를 같은 PostgreSQL 트랜잭션에 저장하는 구조를 살펴봤다.

```text
purchase_order INSERT
+
outbox_event INSERT
+
하나의 DB Transaction
```

이 구조를 사용하면 주문만 저장되고 이벤트 기록이 사라지는 상태를 막을 수 있다.
하지만 Outbox 테이블에 이벤트가 저장되었다고 해서 Kafka에 자동으로 메시지가 만들어지는 것은 아니다.
Outbox 테이블과 Kafka 사이에는 이벤트를 전달하는 별도의 과정이 필요하다.
대표적인 방식은 두 가지다.

```text
Polling Publisher
→ 애플리케이션이 Outbox 테이블을 주기적으로 조회

Change Data Capture
→ DB Transaction Log를 읽어 Outbox INSERT를 감지
```

Polling 방식에서는 애플리케이션이 직접 발행 대상 행을 조회하고 Kafka로 보낸다.
CDC 방식에서는 데이터베이스에 Commit된 변경 기록을 Debezium 같은 도구가 읽어 Kafka로 전달한다.
Debezium 공식 예제는 PostgreSQL의 WAL을 이용한 CDC 방식을 사용한다.
전체 흐름은 다음과 같다.

```text
Order Service
        |
        | 같은 DB Transaction
        v
purchase_order INSERT
outbox_event INSERT
        |
        | Commit
        v
PostgreSQL WAL
        |
        v
Debezium PostgreSQL Connector
        |
        v
Outbox Event Router
        |
        v
Kafka Topic
        |
        v
Shipment Service
```

애플리케이션은 자신의 데이터베이스만 변경한다.
Kafka 발행은 Commit 이후 Debezium이 담당한다.

---

## 15. Polling 방식은 어떻게 동작하는가

Polling 방식은 Outbox Pattern을 구현하는 가장 직접적인 방법이다.
애플리케이션 내부에 Scheduler나 별도의 Publisher를 두고 일정 주기로 Outbox 테이블을 조회한다.

```text
Scheduler 실행
        |
        v
발행되지 않은 Outbox 조회
        |
        v
Kafka 발행
        |
        v
발행 완료 상태 변경
```

단순하게 구현하면 다음과 같은 형태가 된다.

```java
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    // 아직 발행되지 않은 이벤트를 조회한다.
    List<OutboxEvent> events =
        outboxRepository.findPendingEvents();

    for (OutboxEvent event : events) {
        // Outbox에 저장된 Topic, Key, Payload를 사용해 발행한다.
        kafkaTemplate.send(
            event.getTopic(),
            event.getAggregateId(),
            event.getPayload()
        );
    }
}
```

처음에는 간단해 보이지만 실제 운영에서는 고려할 내용이 많다.

```text
여러 Publisher가 같은 행을 조회할 수 있음

Kafka 발행 성공 후 상태 변경 전에 종료될 수 있음

오래 처리 중인 행을 다른 Publisher가 회수해야 함

실패한 이벤트를 다시 시도해야 함

PUBLISHED 행을 언제 삭제할지 결정해야 함
```

따라서 Polling Publisher를 안정적으로 운영하려면 다음과 같은 상태 관리가 필요하다.

```text
PENDING
→ PROCESSING
→ PUBLISHED
```

여러 인스턴스가 동시에 동작한다면 `FOR UPDATE SKIP LOCKED`, Claim Token, Lease도 필요하다.
Polling 방식의 장점은 동작을 애플리케이션 코드에서 직접 제어할 수 있다는 점이다.
반면 Claim, 재시도, 상태 복구, 삭제 정책을 직접 구현해야 한다.
Debezium은 이 Polling 과정을 대신해 DB Transaction Log를 읽는다.

---

## 16. CDC는 무엇인가

CDC는 `Change Data Capture`의 약자로 데이터베이스에서 발생한 변경을 감지해 외부 시스템으로 전달하는 기술이다.
여기서 변경은 일반적으로 다음 작업을 의미한다.

```text
INSERT
UPDATE
DELETE
```

가장 단순한 CDC 방식은 테이블을 반복 조회하는 것이다.

```text
최근 1분 동안 변경된 행 조회
→ 외부 시스템에 전달
```

하지만 이 방식은 변경 시각 컬럼을 관리해야 하고 반복적인 조회 부하도 발생한다.
Debezium은 테이블을 직접 반복 조회하는 대신 데이터베이스의 Transaction Log를 읽는다.

```text
Application
→ PostgreSQL 변경

PostgreSQL
→ WAL에 변경 기록

Debezium
→ WAL을 읽음

Kafka
→ 변경 이벤트 저장
```

이 방식을 **Log-Based CDC**라고 한다.
데이터베이스는 복구와 복제를 위해 원래 Transaction Log를 기록한다.
Debezium은 이미 존재하는 이 기록을 활용하므로 전체 테이블을 반복해서 검색하지 않아도 된다.

---

## 17. PostgreSQL WAL은 무엇인가

PostgreSQL은 데이터 페이지를 변경하기 전에 변경 내용을 WAL에 기록한다.
WAL은 `Write-Ahead Log`의 약자다.
이름 그대로 실제 데이터 파일보다 로그를 먼저 기록한다.

```text
1. 변경 내용을 WAL에 기록
2. WAL이 디스크에 안전하게 저장됨
3. Transaction Commit
4. 실제 데이터 페이지는 이후 반영 가능
```

예를 들어 주문 행을 저장한다고 가정하자.

```sql
INSERT INTO purchase_order (
    id,
    customer_id,
    status
) VALUES (
    100,
    10,
    'CREATED'
);
```

PostgreSQL은 이 변경을 바로 데이터 파일에만 쓰고 끝내지 않는다.
먼저 WAL에 변경 정보를 기록한다.
서버가 비정상 종료되더라도 WAL을 다시 읽어 Commit된 변경을 복구할 수 있다.

```text
Transaction Commit 성공
        |
        X PostgreSQL 비정상 종료
        |
        v
재시작
        |
        v
WAL 재생
        |
        v
Commit된 변경 복구
```

WAL은 복구뿐 아니라 Replication에도 사용된다.
Primary의 WAL을 Replica가 받아 동일한 변경을 재생하면 데이터 복제가 가능하다.
Debezium도 이 WAL을 이용한다.

```text
PostgreSQL WAL
        |
        +-- 장애 복구
        +-- Replica 복제
        +-- Debezium CDC
```

Debezium은 PostgreSQL의 Logical Decoding 기능을 통해 WAL의 저수준 변경 정보를 행 단위 이벤트로 해석한다.

## 18. Debezium이 WAL을 읽는 과정

주문 서비스에서 하나의 주문을 생성한다고 가정해 보자.
애플리케이션은 하나의 트랜잭션에서 주문과 Outbox 이벤트를 함께 저장한다.

```java
@Transactional
public PurchaseOrder addOrder(
        PurchaseOrder order
) {
    // 1. 주문 정보를 저장한다.
    PurchaseOrder savedOrder =
        entityManager.merge(order);

    // 2. Kafka에 직접 보내지 않고
    //    같은 Transaction에서 Outbox INSERT를 발생시킨다.
    event.fire(
        OrderCreatedEvent.of(savedOrder)
    );

    return savedOrder;
}
```

실제로 DB에는 다음 변경이 발생한다.

```text
purchase_order INSERT
outbox_event INSERT
```

트랜잭션이 Commit되면 이 변경들이 WAL에 기록된다.

```text
BEGIN

purchase_order INSERT
outbox_event INSERT

COMMIT
```

Debezium은 WAL을 읽다가 `outbox_event`의 INSERT를 발견한다.
그다음 이 행을 Kafka Record로 변환한다.

```text
WAL의 outbox_event INSERT
        |
        v
Debezium Change Event
        |
        v
Outbox Event Router
        |
        v
Kafka Record
```

트랜잭션이 Rollback되면 Debezium은 해당 이벤트를 Kafka로 전달하지 않는다.

```text
purchase_order INSERT
outbox_event INSERT
        |
        v
Rollback

→ Commit된 Outbox 이벤트 없음
→ Kafka 발행 없음
```

이것이 중요한 이유는 Debezium이 단순히 SQL 실행 여부를 보는 것이 아니라 **Commit된 데이터 변경**을 기준으로 처리하기 때문이다.

## 19. 왜 비즈니스 테이블을 그대로 CDC하지 않는가

Debezium은 `purchase_order` 테이블의 변경을 직접 읽을 수도 있다.

```text
purchase_order INSERT
→ Debezium
→ Kafka
```

이렇게 하면 Outbox 테이블이 없어도 되는 것처럼 보일 수 있다.
하지만 비즈니스 테이블의 변경과 외부에 공개할 도메인 이벤트는 같은 개념이 아니다.
주문 테이블에는 서비스 내부 구현을 위한 데이터가 저장된다.

```text
낙관적 락 버전
내부 상태 코드
삭제 여부
감사 컬럼
내부 정규화 구조
```

반면 외부 Consumer가 원하는 것은 비즈니스 의미가 명확한 이벤트다.

```text
OrderCreated
OrderLineCancelled
OrderCompleted
```

테이블 변경을 그대로 외부에 노출하면 내부 DB 구조가 이벤트 계약이 된다.

```text
DB 컬럼명 변경
→ Kafka 메시지 구조 변경

테이블 분리
→ Consumer 영향

내부 정규화 변경
→ 외부 서비스 수정 필요
```

Outbox를 사용하면 내부 데이터 모델과 외부 이벤트 모델을 분리할 수 있다.

```text
purchase_order
→ 서비스 내부 저장 구조

outbox_event.payload
→ 외부 Consumer를 위한 이벤트 계약
```

Outbox 이벤트는 단순한 Row Change가 아니라 서비스가 의도적으로 공개한 Domain Event다.

## 20. Outbox 테이블 구조

Debezium 예제의 Outbox 테이블은 다음과 같은 구조를 가진다.

```text
id
aggregatetype
aggregateid
type
payload
```

SQL로 표현하면 다음과 비슷하다.

```sql
CREATE TABLE outbox_event (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL
);
```

각 컬럼은 Kafka 메시지를 구성하는 데 사용된다.

### `id`

각 이벤트 메시지를 식별하는 고유 ID다.

```text
id
→ 이벤트 자체의 식별자
→ Consumer 중복 처리에 사용
```

동일한 이벤트가 Kafka에 두 번 전달되어도 `id`는 변하지 않는다.
Consumer는 이 값을 저장해 이미 처리한 이벤트인지 확인할 수 있다.

### `aggregate_type`

이벤트가 어떤 종류의 Aggregate에 속하는지 나타낸다.

```text
Order
Customer
Payment
Shipment
```

Debezium 예제에서는 이 값을 Kafka Topic을 결정하는 데 사용한다.

```text
aggregate_type = Order
→ OrderEvents Topic

aggregate_type = Customer
→ CustomerEvents Topic
```

### `aggregate_id`

이벤트가 속한 Aggregate의 ID다.

```text
Order ID = 100
Customer ID = 10
```

이 값은 Kafka Message Key로 사용된다.
같은 Aggregate의 이벤트를 같은 Partition으로 보내기 위해서다.

### `event_type`

어떤 비즈니스 사건이 발생했는지 나타낸다.

```text
OrderCreated
OrderLineUpdated
OrderCancelled
```

Consumer는 이 값으로 어떤 Handler를 실행할지 결정한다.

### `payload`

실제 이벤트 데이터다.

```json
{
  "orderId": 100,
  "customerId": 10,
  "orderDate": "2026-07-27T03:00:00Z",
  "lineItems": [
    {
      "itemId": 50,
      "quantity": 2
    }
  ]
}
```

Payload는 Outbox INSERT 시점에 완성된 형태로 저장해야 한다.
Publisher나 CDC Connector가 나중에 현재 비즈니스 테이블을 다시 조회해 만드는 것이 아니다.

---

## 21. Aggregate란 무엇인가

Aggregate는 DDD에서 하나의 단위로 일관성을 관리하는 도메인 객체 묶음이다.
주문을 예로 들면 다음 객체들이 있을 수 있다.

```text
PurchaseOrder
OrderLine
DeliveryAddress
PaymentInformation
```

이 객체들을 모두 외부에서 독립적으로 변경하도록 허용하면 일관성을 유지하기 어렵다.
따라서 하나의 Aggregate Root를 통해서만 내부 객체에 접근하도록 한다.

```text
PurchaseOrder
→ Aggregate Root

OrderLine
→ PurchaseOrder 내부 Entity
```

주문 상품 하나가 취소되더라도 이벤트의 Aggregate는 `OrderLine`이 아니라 `Order`다.

```text
eventType
→ OrderLineCancelled

aggregateType
→ Order

aggregateId
→ 주문 ID
```

그 이유는 주문 상품이 주문 Aggregate 안에 포함된 객체이기 때문이다.

```text
주문 100 생성
주문 100의 상품 변경
주문 100의 상품 취소
주문 100 완료

→ 모두 aggregateId = 100
```

이렇게 하면 같은 주문에 대한 이벤트를 하나의 기준으로 묶을 수 있다.

## 22. Aggregate ID를 Kafka Key로 사용하는 이유

Kafka는 Message Key를 기준으로 Partition을 선택한다.

```text
같은 Key
→ 같은 Topic 안에서 같은 Partition
```

`aggregate_id`를 Kafka Key로 사용하면 같은 Aggregate의 이벤트가 같은 Partition으로 전달된다.

```text
Order 100 생성
Order 100 상품 취소
Order 100 완료

→ Key = 100
→ 같은 Partition
```

예를 들어 다음 이벤트들이 있다고 가정하자.

```text
OrderCreated
OrderLineUpdated
OrderCancelled
```

모두 같은 주문에 대한 이벤트라면 같은 Kafka Partition으로 전달된다.
Debezium의 Transaction Log 기반 전파가 원래 발생 순서를 유지하고, 같은 Aggregate ID를 같은 Key로 사용한다면 Consumer는 한 Partition 안에서 순서대로 읽을 수 있다.

```text
Offset 10: OrderCreated
Offset 11: OrderLineUpdated
Offset 12: OrderCancelled
```

다만 여기서 보장 범위를 정확히 구분해야 한다.

```text
같은 aggregateId
→ 같은 Partition으로 전달

서로 다른 aggregateId
→ 서로 다른 Partition일 수 있음
→ 전체 이벤트 간 전역 순서 없음
```

주문 100과 주문 200 사이의 순서는 일반적으로 중요하지 않다.
중요한 것은 같은 주문 안에서 발생한 이벤트 순서다.

---

## 23. Event 객체의 역할

애플리케이션에서 Outbox 행을 직접 생성할 수도 있다.

```java
outboxRepository.save(
    new OutboxEvent(...)
);
```

Debezium 예제에서는 도메인 코드가 Outbox 테이블 구조에 직접 의존하지 않도록 `ExportedEvent` 인터페이스를 사용한다.

```java
public interface ExportedEvent {

    // 이벤트가 속한 Aggregate의 ID
    String getAggregateId();

    // Aggregate 종류
    String getAggregateType();

    // 외부 Consumer가 사용할 데이터
    JsonNode getPayload();

    // 발생한 비즈니스 사건
    String getType();
}
```

주문 생성 이벤트는 이 인터페이스를 구현한다.

```java
public class OrderCreatedEvent
        implements ExportedEvent {

    private final long orderId;
    private final JsonNode payload;

    private OrderCreatedEvent(
            long orderId,
            JsonNode payload
    ) {
        this.orderId = orderId;
        this.payload = payload;
    }

    @Override
    public String getAggregateId() {
        // 같은 주문 이벤트를 같은 Partition으로 보내기 위한 Key
        return String.valueOf(orderId);
    }

    @Override
    public String getAggregateType() {
        // Kafka Topic을 결정하는 기준
        return "Order";
    }

    @Override
    public String getType() {
        // Consumer가 Handler를 선택하는 기준
        return "OrderCreated";
    }

    @Override
    public JsonNode getPayload() {
        return payload;
    }
}
```

도메인 서비스는 Kafka나 Debezium을 직접 알 필요가 없다.

```java
@Transactional
public PurchaseOrder addOrder(
        PurchaseOrder order
) {
    PurchaseOrder savedOrder =
        entityManager.merge(order);

    // 애플리케이션 내부 이벤트를 동기적으로 발생시킨다.
    // 이 시점에는 Kafka를 호출하지 않는다.
    event.fire(
        OrderCreatedEvent.of(savedOrder)
    );

    return savedOrder;
}
```

이 구조는 도메인 로직과 Outbox 저장 구현을 분리한다.

## 24. EventSender가 Outbox 행을 저장하는 과정

`event.fire()`가 호출되면 CDI가 `ExportedEvent`를 관찰하는 메서드를 실행한다.

```java
@ApplicationScoped
public class EventSender {

    @PersistenceContext
    private EntityManager entityManager;

    public void onExportedEvent(
            @Observes ExportedEvent event
    ) {
        OutboxEvent outboxEvent =
            new OutboxEvent(
                event.getAggregateType(),
                event.getAggregateId(),
                event.getType(),
                event.getPayload()
            );

        // 현재 OrderService의 Transaction에 참여한다.
        entityManager.persist(outboxEvent);

        // Debezium 예제는 CDC가 WAL의 INSERT를 읽은 뒤
        // 테이블이 계속 증가하지 않도록 곧바로 DELETE도 예약한다.
        entityManager.remove(outboxEvent);
    }
}
```

중요한 점은 `@Observes`가 기본적으로 동기 실행된다는 것이다.

```text
OrderService.addOrder()
        |
        | event.fire()
        v
EventSender.onExportedEvent()
        |
        | persist(outbox)
        v
같은 Thread
같은 Transaction
```

따라서 세 작업이 하나의 트랜잭션에 포함된다.

```text
purchase_order INSERT
outbox_event INSERT
outbox_event DELETE
```

처음 보면 저장한 Outbox 행을 즉시 삭제하는 부분이 이상해 보인다.
하지만 Debezium 방식에서는 실제 테이블 내용이 아니라 WAL의 변경 이력을 읽는다.

---

## 25. Outbox 행을 저장하자마자 삭제하는 이유

Polling 방식에서는 Publisher가 Outbox 테이블의 행을 직접 조회해야 한다.
따라서 Kafka 발행이 끝날 때까지 행을 유지해야 한다.

```text
PENDING 행 저장
→ Publisher 조회
→ Kafka 발행
→ PUBLISHED 변경
→ 일정 기간 후 삭제
```

하지만 Debezium은 테이블 현재 상태를 조회하지 않는다.
WAL에 기록된 변경 내역을 읽는다.
하나의 트랜잭션에서 다음 작업이 발생한다고 가정하자.

```java
entityManager.persist(outboxEvent);
entityManager.remove(outboxEvent);
```

Commit되면 WAL에는 논리적으로 다음 기록이 남는다.

```text
outbox_event INSERT
outbox_event DELETE
```

Debezium은 INSERT 이벤트를 Kafka로 전달하고 DELETE 이벤트는 무시할 수 있다.
최종 테이블에는 행이 남지 않는다.

```text
Outbox Table
→ 최종적으로 비어 있음

PostgreSQL WAL
→ INSERT와 DELETE 기록 존재

Debezium
→ INSERT만 Kafka로 전달
```

이 방식의 장점은 별도의 Outbox 정리 작업이 필요 없다는 것이다.

```text
PUBLISHED 상태 관리 없음
Outbox 보관 기간 없음
대량 DELETE Batch 없음
Table Bloat 감소
```

하지만 이것은 **CDC 방식이기 때문에 가능한 설계**다.
Polling Publisher에서 같은 방식을 사용하면 안 된다.

```text
Polling Publisher
→ 테이블의 현재 행을 조회해야 함
→ 즉시 삭제하면 발행할 이벤트를 찾지 못함
```

또한 실무에서는 감사와 재처리 요구 때문에 Outbox 행을 일정 기간 유지하는 구조를 선택할 수도 있다.
즉시 삭제 방식은 Debezium 예제의 선택이지 Outbox Pattern의 필수 조건은 아니다.

---

## 26. Debezium PostgreSQL Connector 등록

Debezium이 PostgreSQL WAL을 읽으려면 Kafka Connect에 Connector를 등록해야 한다.
예제 설정은 다음과 같은 형태다.

```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",

    "database.hostname": "order-db",
    "database.port": "5432",
    "database.user": "postgresuser",
    "database.password": "postgrespw",
    "database.dbname": "orderdb",

    "database.server.name": "dbserver1",

    "schema.whitelist": "inventory",
    "table.whitelist": "inventory.outboxevent",

    "tombstones.on.delete": "false",

    "transforms": "router",
    "transforms.router.type": "io.debezium.examples.outbox.routingsmt.EventRouter"
  }
}
```

중요한 설정만 나누어 보면 다음과 같다.

### Connector 종류

```json
"connector.class":
  "io.debezium.connector.postgresql.PostgresConnector"
```

PostgreSQL WAL을 읽는 Debezium Connector를 사용한다.

### 대상 데이터베이스

```json
"database.hostname": "order-db",
"database.port": "5432",
"database.dbname": "orderdb"
```

어떤 PostgreSQL 인스턴스를 감시할지 지정한다.

### 감시 대상 테이블

```json
"table.whitelist": "inventory.outboxevent"
```

모든 비즈니스 테이블 변경을 Kafka로 보내지 않고 Outbox 테이블만 감시한다.
이렇게 하면 외부에 공개하기로 결정한 이벤트만 전달된다.

### Tombstone 비활성화

```json
"tombstones.on.delete": "false"
```

Outbox 행 삭제는 Kafka 이벤트 삭제를 의미하지 않는다.
테이블 정리를 위한 기술적인 DELETE이므로 Kafka Topic에 Tombstone을 만들 필요가 없다.

### Event Router 등록

```json
"transforms": "router",
"transforms.router.type":
  "io.debezium.examples.outbox.routingsmt.EventRouter"
```

Debezium이 만든 일반적인 Row Change Event를 도메인 이벤트 형태의 Kafka Record로 변경한다.

## 27. Debezium의 기본 Change Event 구조

Debezium은 데이터베이스의 Row Change를 그대로 단순한 Payload로 전달하지 않는다.
일반적으로 다음과 같은 정보를 포함한다.

```text
before
→ 변경 전 행

after
→ 변경 후 행

op
→ 작업 종류

ts_ms
→ 이벤트 처리 시각

source
→ DB와 Transaction 관련 정보
```

예를 들어 Outbox INSERT는 다음과 비슷한 구조가 된다.

```json
{
  "before": null,
  "after": {
    "id": "d03dfb18-8af8-464d-890b-09eb8b2dbbdd",
    "aggregatetype": "Order",
    "aggregateid": "4",
    "type": "OrderCreated",
    "payload": "{\"id\":4,\"customerId\":123}"
  },
  "op": "c",
  "ts_ms": 1550307598558
}
```

`op`는 어떤 작업이 발생했는지를 나타낸다.

```text
c
→ create, INSERT

u
→ update

d
→ delete

r
→ snapshot read
```

Outbox 테이블은 Append-Only 방식으로 사용하기 때문에 정상적인 이벤트는 `c`다.
일반 Consumer에게는 이 Debezium 내부 구조가 불필요하게 복잡하다.
배송 서비스는 `before`, `after`, `source` 같은 정보를 알고 싶지 않다.
배송 서비스가 필요한 값은 다음과 같다.

```text
eventId
eventType
aggregateId
payload
occurredAt
```

Event Router는 Debezium의 복잡한 Change Event를 외부 Consumer에게 적합한 형태로 바꾼다.

## 28. Event Router의 역할

Event Router는 Kafka Connect의 SMT로 동작한다.
SMT는 `Single Message Transform`의 약자다.
Connector가 만든 Record 한 건을 받아 Topic, Key, Header, Value를 변경한다.

```text
Debezium Change Event
        |
        v
Event Router SMT
        |
        +-- Topic 변경
        +-- Key 변경
        +-- Header 추가
        +-- Payload 단순화
        |
        v
Kafka Record
```

예제의 핵심 로직은 다음과 같다.

```java
@Override
public R apply(R record) {
    // 값이 없는 Tombstone Record는 그대로 반환한다.
    if (record.value() == null) {
        return record;
    }

    Struct changeEvent =
        (Struct) record.value();

    // c, u, d 중 어떤 DB 작업인지 확인한다.
    String operation =
        changeEvent.getString("op");

    if (operation.equals("d")) {
        // Outbox DELETE는 테이블 정리 목적이다.
        // 비즈니스 이벤트가 아니므로 Kafka에 전달하지 않는다.
        return null;
    }

    if (operation.equals("c")) {
        Struct after =
            changeEvent.getStruct("after");

        // 같은 Aggregate 이벤트를 같은 Partition으로 보내기 위한 Key
        String key =
            after.getString("aggregateid");

        // Aggregate 종류에 따라 Topic을 선택한다.
        String topic =
            after.getString("aggregatetype")
                + "Events";

        String eventId =
            after.getString("id");

        String eventType =
            after.getString("type");

        String payload =
            after.getString("payload");

        // Consumer가 사용할 최소한의 이벤트 Value를 만든다.
        Struct value =
            createEventValue(
                eventType,
                payload,
                changeEvent.getInt64("ts_ms")
            );

        // 중복 처리에 사용할 Event ID를 Header에 넣는다.
        Headers headers =
            record.headers();

        headers.addString(
            "eventId",
            eventId
        );

        return record.newRecord(
            topic,
            null,
            Schema.STRING_SCHEMA,
            key,
            value.schema(),
            value,
            record.timestamp(),
            headers
        );
    }

    // Outbox 행은 UPDATE하지 않는다는 전제를 둔다.
    throw new IllegalArgumentException(
        "Unexpected operation: " + operation
    );
}
```

이 코드가 하는 일은 네 가지다.

```text
1. DELETE 이벤트 제거
2. aggregateType으로 Topic 결정
3. aggregateId를 Kafka Key로 사용
4. eventId를 Header에 추가
```

---

## 29. Topic은 어떻게 결정되는가

Debezium 기본 설정만 사용하면 하나의 Outbox 테이블에서 발생한 모든 이벤트가 같은 Topic으로 전달된다.

```text
dbserver1.inventory.outboxevent
```

그러면 주문 이벤트와 고객 이벤트가 한 Topic에 섞인다.

```text
OrderCreated
CustomerCreated
OrderCancelled
CustomerGradeChanged
```

배송 서비스는 고객 이벤트에 관심이 없지만 모든 메시지를 받아 구분해야 한다.
Event Router는 `aggregate_type`을 이용해 Topic을 분리한다.

```text
aggregateType = Order
→ OrderEvents

aggregateType = Customer
→ CustomerEvents
```

결과적으로 Consumer는 자신에게 필요한 Topic만 구독할 수 있다.

```text
Shipment Service
→ OrderEvents 구독

Customer Service
→ CustomerEvents 구독
```

서비스 간 결합을 낮추고 불필요한 메시지 처리를 줄일 수 있다.
Topic 이름을 Aggregate Type에서 직접 만드는 방식은 이해하기 쉽지만, 실제 환경에서는 Naming Convention을 별도로 관리하는 편이 안전하다.

```text
Order
→ order.events.v1

Customer
→ customer.events.v1
```

이벤트 버전, 환경 구분, 조직 규칙을 Topic 이름에 어떻게 반영할지도 함께 결정해야 한다.

## 30. Kafka Key는 어떻게 결정되는가

Event Router는 `aggregate_id`를 Kafka Key로 사용한다.

```java
String key =
    after.getString("aggregateid");
```

예를 들어 주문 ID가 `4`라면 Kafka Record는 다음처럼 만들어진다.

```text
Topic
→ OrderEvents

Key
→ 4

Value
→ OrderCreated Payload
```

같은 주문에 대한 후속 이벤트도 Key로 `4`를 사용한다.

```text
OrderCreated
Key = 4

OrderLineUpdated
Key = 4

OrderCancelled
Key = 4
```

Kafka는 같은 Topic의 같은 Key를 같은 Partition으로 보낸다.
따라서 하나의 주문에 대한 이벤트를 하나의 Consumer 흐름에서 처리할 수 있다.

```text
Partition 1

Offset 100
OrderCreated, Key=4

Offset 101
OrderLineUpdated, Key=4

Offset 102
OrderCancelled, Key=4
```

이벤트가 Aggregate의 하위 Entity에 대한 변경이라도 Aggregate Root의 ID를 Key로 사용해야 한다.

```text
잘못된 Key
→ orderLineId

올바른 Key
→ orderId
```

`orderLineId`를 사용하면 같은 주문의 이벤트가 서로 다른 Partition으로 분산될 수 있다.

## 31. Event ID를 Header에 넣는 이유

Event Router는 Outbox의 UUID를 Kafka Header에 넣는다.

```java
headers.addString(
    "eventId",
    eventId
);
```

Kafka Record는 다음처럼 구성된다.

```text
Header
→ eventId=d03dfb18-...

Key
→ 4

Value
→ OrderCreated Payload
```

이벤트 ID를 Payload 안에 넣어도 중복 처리는 가능하다.
하지만 Header에 두면 Consumer가 Payload 전체를 파싱하기 전에 중복 여부를 확인할 수 있다.

```text
Record 수신
→ Header의 eventId 확인
→ 이미 처리한 이벤트면 Payload 파싱 생략
```

Event ID는 Kafka의 Partition과 Offset을 대신하는 값이 아니다.

```text
Partition + Offset
→ Kafka 내부 물리 위치

Event ID
→ 논리적인 도메인 이벤트 식별자
```

같은 이벤트가 다시 발행되면 Offset은 달라질 수 있지만 Event ID는 유지된다.

```text
Partition 1 / Offset 100 / eventId=evt-001
Partition 1 / Offset 101 / eventId=evt-001
```

Consumer 멱등성은 Event ID를 기준으로 처리해야 한다.

## 32. Outbox INSERT가 Kafka Record로 변환되는 전체 과정

주문 생성 요청부터 Kafka Record 저장까지 흐름을 정리하면 다음과 같다.

```text
1. Client가 주문 생성 요청

2. Order Service가 DB Transaction 시작

3. purchase_order INSERT

4. OrderCreatedEvent 생성

5. outbox_event INSERT

6. Transaction Commit

7. PostgreSQL WAL에 변경 기록

8. Debezium이 WAL 읽음

9. outbox_event INSERT 감지

10. Event Router가 Record 변환

11. aggregateType으로 Topic 결정

12. aggregateId를 Kafka Key로 사용

13. eventId를 Header에 추가

14. Payload를 Kafka Value로 저장

15. Shipment Service가 이벤트 수신
```

애플리케이션이 직접 Kafka Broker에 접근하는 지점은 없다.

```text
Order Service
→ PostgreSQL만 변경

Debezium
→ PostgreSQL WAL을 읽어 Kafka 발행
```

이 구조는 주문 서비스와 Kafka 발행 책임을 분리한다.

## 33. Polling Outbox와 Debezium CDC 비교

두 방식 모두 비즈니스 데이터와 Outbox 이벤트를 같은 DB 트랜잭션에 저장한다.
차이는 Outbox 이후의 전달 방식이다.
| **항목** | **Polling Publisher** | **Debezium CDC** |
| --- | --- | --- |
| 변경 감지 | 테이블 주기 조회 | WAL 읽기 |
| 발행 주체 | 애플리케이션 Publisher | Debezium Connector |
| 지연 | Polling 주기에 영향 | 일반적으로 낮음 |
| DB 조회 부하 | 반복 조회 발생 | 반복 조회 없음 |
| 상태 관리 | PENDING, PROCESSING 필요 | Connector Offset 관리 |
| Lease와 Claim | 직접 구현 | 일반적으로 불필요 |
| 재시도 | 애플리케이션 구현 | Kafka Connect가 관리 |
| 운영 대상 | 애플리케이션 코드 | Kafka Connect·Debezium |
| Outbox 정리 | 직접 삭제·Archive | 즉시 삭제 구조 가능 |
| 구현 이해도 | 코드로 추적하기 쉬움 | CDC 인프라 이해 필요 |

Polling은 별도 CDC 인프라 없이 시작하기 쉽다.
Debezium은 애플리케이션 코드가 줄어들고 낮은 지연으로 이벤트를 전달할 수 있다.
대신 다음 구성요소를 운영해야 한다.

```text
Kafka Connect
Debezium Connector
PostgreSQL Logical Replication
Replication Slot
WAL 보관량
Connector Lag
```

Connector가 장기간 중단되면 읽지 못한 WAL이 계속 보관될 수 있다.

```text
Debezium 중단
→ Replication Slot의 처리 위치가 이동하지 않음
→ PostgreSQL이 필요한 WAL을 삭제하지 못함
→ 디스크 사용량 증가
```

따라서 CDC 방식도 운영 비용이 없는 것은 아니다.

## 34. Debezium 방식에서 이벤트 순서를 이해하는 방법

Polling Publisher에서는 여러 Publisher, `SKIP LOCKED`, Backoff 때문에 DB 저장 순서와 Kafka 발행 순서가 달라질 수 있다.
Debezium은 PostgreSQL WAL을 읽기 때문에 Commit된 변경 순서를 따라가는 데 유리하다.

```text
DB Commit 순서
        |
        v
WAL 기록 순서
        |
        v
Debezium 처리 순서
        |
        v
Kafka Producer 전송 순서
```

여기에 같은 Aggregate ID를 Kafka Key로 사용하면 같은 Aggregate 이벤트는 같은 Partition으로 전달된다.

```text
Order 4 Created
Order 4 Updated
Order 4 Cancelled

→ 같은 Key
→ 같은 Partition
→ Partition 안에서 순차 소비
```

하지만 순서 보장이 모든 이벤트에 대해 전역적으로 제공되는 것은 아니다.

```text
Order 4
→ Partition 1

Order 7
→ Partition 3
```

서로 다른 Partition 사이의 순서는 비교할 수 없다.
또한 Connector 구성, Topic Routing, 병렬 처리 설정을 잘못하면 기대한 순서가 깨질 수 있다.
따라서 순서가 중요한 Aggregate는 반드시 같은 Routing 기준을 사용해야 한다.

---

## 35. CDC 방식에서도 중복은 발생할 수 있다

Debezium이 WAL을 읽는다고 해서 이벤트가 물리적으로 정확히 한 번만 전달되는 것은 아니다.
다음 상황을 생각해 보자.

```text
Debezium
→ Kafka 발행 성공
        |
        X 처리 위치 저장 전 종료
```

재시작 후 Debezium은 마지막으로 확실히 처리한 위치부터 다시 읽을 수 있다.
이미 Kafka에 전달한 Outbox 이벤트가 다시 전달될 수 있다.

```text
Kafka Record 1
eventId = evt-001

Kafka Record 2
eventId = evt-001
```

따라서 Debezium 기반 Outbox도 기본적으로 At-Least-Once 전달을 전제로 한다.

```text
At-Least-Once
→ 최소 한 번 전달
→ 중복 가능
```

Event ID가 필요한 이유가 여기에 있다.
Producer 쪽에서 중복 가능성을 완전히 제거하려 하지 않고 Consumer가 동일 Event ID를 한 번만 반영하도록 만든다.

---

## 36. CDC가 Outbox Pattern을 대신하는 것은 아니다

CDC와 Outbox Pattern은 같은 개념이 아니다.
CDC는 데이터베이스 변경을 외부로 전달하는 기술이다.
Outbox Pattern은 비즈니스 데이터와 이벤트 발행 의도를 같은 트랜잭션에 저장하는 설계 패턴이다.

```text
Outbox Pattern
→ 무엇을 DB Transaction에 저장할 것인가

CDC
→ 저장된 변경을 어떻게 외부로 전달할 것인가
```

둘을 결합하면 다음 구조가 된다.

```text
Transactional Outbox
→ 주문과 도메인 이벤트를 함께 저장

Debezium CDC
→ Outbox INSERT를 WAL에서 감지

Kafka
→ 이벤트를 다른 서비스에 전달
```

비즈니스 테이블을 그대로 CDC하는 것과 Outbox를 CDC하는 것도 다르다.

```text
비즈니스 테이블 CDC
→ 내부 데이터 변경 노출

Outbox CDC
→ 의도적으로 설계한 도메인 이벤트 노출
```

Debezium은 전달 수단이고, 이벤트의 의미와 스키마는 애플리케이션이 설계해야 한다.

## 37. 이번 구조에서 애플리케이션과 Debezium의 책임

주문 서비스의 책임은 다음과 같다.

```text
주문 비즈니스 규칙 실행

PurchaseOrder 저장

외부에 공개할 Domain Event 생성

Outbox에 완성된 Payload 저장

같은 DB Transaction으로 Commit
```

Debezium의 책임은 다음과 같다.

```text
PostgreSQL WAL 읽기

Commit된 Outbox INSERT 감지

Event Router 적용

Kafka Topic으로 전달

처리 위치 관리

장애 후 이어서 처리
```

Kafka Consumer의 책임은 다음과 같다.

```text
Event ID 중복 확인

Event Type에 맞는 Handler 실행

자신의 DB에 결과 저장

처리 완료 기록과 비즈니스 변경을
같은 Transaction으로 Commit
```

각 역할을 분리하면 장애가 발생했을 때 어느 단계의 문제인지 추적하기 쉬워진다.

```text
Outbox 행 없음
→ 주문 서비스 Transaction 문제

Outbox는 Commit됐지만 Kafka에 없음
→ Debezium 또는 Kafka Connect 문제

Kafka에 있지만 Consumer 결과 없음
→ Consumer 처리 문제
```

다음 내용에서는 Kafka에 같은 이벤트가 여러 번 전달되더라도 Consumer가 비즈니스 결과를 한 번만 반영하는 방법을 이어서 살펴본다.

## 38. Consumer는 왜 중복 처리를 준비해야 하는가

Debezium이 PostgreSQL WAL을 읽어 Kafka로 이벤트를 전달하더라도 같은 이벤트가 두 번 이상 전달될 수 있다.
이유는 이벤트 처리 과정에 여러 개의 완료 지점이 존재하기 때문이다.
예를 들어 Debezium이 Kafka에 이벤트를 정상적으로 저장한 직후 장애가 발생할 수 있다.

```text
PostgreSQL WAL 읽기
        |
        v
Kafka Record 저장 성공
        |
        X Connector 처리 위치 저장 전 종료
```

Kafka에는 이미 이벤트가 저장되어 있다.
하지만 Debezium은 해당 WAL 위치까지 처리했다는 사실을 확실하게 저장하지 못했다.
재시작 후 마지막으로 확인된 위치부터 다시 읽으면 같은 Outbox 이벤트가 다시 Kafka로 전달될 수 있다.

```text
첫 번째 Record
eventId = evt-001
offset = 100

두 번째 Record
eventId = evt-001
offset = 101
```

두 Record의 Kafka Offset은 다르지만 논리적으로 같은 이벤트다.
따라서 Consumer는 Offset이 아니라 이벤트의 고유 ID로 중복 여부를 판단해야 한다.

```text
Kafka Offset
→ Kafka 안에서 Record의 물리적 위치

Event ID
→ 논리적인 이벤트 자체의 식별자
```

이벤트 중복을 완전히 제거하려고 하기보다, 중복이 발생해도 최종 결과가 한 번만 반영되도록 만드는 것이 중요하다.
이러한 성질을 멱등성이라고 한다.

```text
같은 이벤트를 한 번 처리한 결과

=

같은 이벤트를 여러 번 처리한 결과
```

---

## 39. Event ID를 이용한 중복 확인

Debezium 예제에서는 Outbox 이벤트의 UUID를 Kafka Header에 넣는다.
Consumer는 메시지를 받으면 가장 먼저 이 ID를 확인한다.

```text
Kafka Record 수신
        |
        v
Header에서 eventId 조회
        |
        v
이미 처리한 이벤트인지 확인
        |
        +-- 이미 처리함 → 건너뜀
        |
        +-- 처음 처리함 → 비즈니스 로직 실행
```

처리한 이벤트 ID는 Consumer 자신의 데이터베이스에 저장한다.

```sql
CREATE TABLE consumed_message (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);
```

`event_id`를 Primary Key로 사용하면 같은 이벤트 ID를 두 번 저장할 수 없다.

```text
evt-001 최초 저장
→ 성공

evt-001 다시 저장
→ Primary Key 충돌
```

이 테이블은 메시지의 Payload를 보관하는 용도가 아니다.
해당 이벤트를 이미 처리했는지 확인하기 위한 기록이다.

---

## 40. Debezium 예제의 Consumer 구조

배송 서비스는 주문 이벤트를 수신하면 `OrderEventHandler`를 호출한다.
예제의 핵심 흐름은 다음과 같다.

```java
@ApplicationScoped
public class OrderEventHandler {

    @Inject
    private MessageLog messageLog;

    @Inject
    private ShipmentService shipmentService;

    @Transactional
    public void onOrderEvent(
            UUID eventId,
            String key,
            String event
    ) {
        // 이미 처리한 이벤트라면
        // 배송 데이터를 다시 변경하지 않는다.
        if (messageLog.alreadyProcessed(eventId)) {
            return;
        }

        JsonObject message =
            Json.createReader(
                new StringReader(event)
            ).readObject();

        String eventType =
            message.getString("eventType");

        String payload =
            message.getString("payload");

        JsonObject payloadObject =
            Json.createReader(
                new StringReader(payload)
            ).readObject();

        // 이벤트 종류에 따라
        // 서로 다른 비즈니스 로직을 실행한다.
        if (eventType.equals("OrderCreated")) {
            shipmentService.orderCreated(
                payloadObject
            );
        } else if (
            eventType.equals("OrderLineUpdated")
        ) {
            shipmentService.orderLineUpdated(
                payloadObject
            );
        }

        // 비즈니스 처리가 끝난 이벤트를 기록한다.
        messageLog.processed(eventId);
    }
}
```

처리 흐름을 단순하게 정리하면 다음과 같다.

```text
1. eventId 중복 확인
2. Event Type 확인
3. Payload 파싱
4. 배송 비즈니스 로직 실행
5. 처리 완료 ID 저장
```

여기서 가장 중요한 부분은 전체 메서드가 하나의 데이터베이스 트랜잭션으로 실행된다는 점이다.

## 41. 중복 기록과 비즈니스 변경을 같은 트랜잭션으로 묶어야 한다

다음 두 작업은 반드시 함께 Commit되어야 한다.

```text
배송 데이터 변경

+

이벤트 처리 완료 기록
```

두 작업을 서로 다른 트랜잭션으로 처리하면 다시 Dual Write 문제가 발생한다.
예를 들어 배송 데이터를 먼저 Commit하고 처리 기록을 나중에 저장한다고 가정해 보자.

```text
배송 데이터 저장 성공
        |
        X consumed_message 저장 전 장애
```

Consumer가 재시작하면 같은 이벤트가 다시 전달될 수 있다.
하지만 `consumed_message`에는 기록이 없으므로 처음 받은 이벤트로 판단한다.

```text
같은 OrderCreated 재처리
→ 배송 정보 중복 생성 가능
```

반대로 처리 완료 기록을 먼저 저장한 뒤 배송 데이터 변경이 실패할 수도 있다.

```text
consumed_message 저장 성공
        |
        X 배송 데이터 저장 실패
```

다시 이벤트를 받으면 이미 처리한 이벤트로 판단해 건너뛴다.
결과적으로 배송 데이터는 만들어지지 않는다.
이 문제를 막으려면 두 작업을 같은 로컬 DB 트랜잭션에 포함해야 한다.

```text
Consumer DB Transaction 시작
        |
        v
중복 이벤트 확인
        |
        v
배송 데이터 변경
        |
        v
consumed_message 저장
        |
        v
Commit
```

성공하면 둘 다 저장되고, 실패하면 둘 다 Rollback된다.

## 42. MessageLog 구현

Debezium 예제의 `MessageLog`는 처리한 이벤트 ID를 데이터베이스에 저장한다.

```java
@ApplicationScoped
public class MessageLog {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(TxType.MANDATORY)
    public void processed(UUID eventId) {
        // 현재 Consumer Transaction에 참여해
        // 처리 완료 ID를 저장한다.
        entityManager.persist(
            new ConsumedMessage(
                eventId,
                Instant.now()
            )
        );
    }

    @Transactional(TxType.MANDATORY)
    public boolean alreadyProcessed(
            UUID eventId
    ) {
        // 같은 eventId가 존재하면
        // 이미 처리한 이벤트다.
        return entityManager.find(
            ConsumedMessage.class,
            eventId
        ) != null;
    }
}
```

`TxType.MANDATORY`는 반드시 이미 시작된 트랜잭션 안에서 호출되어야 한다는 뜻이다.
즉 `MessageLog`가 독립적인 새 트랜잭션을 만드는 것이 아니다.

```text
OrderEventHandler Transaction
        |
        +-- shipmentService 처리
        |
        +-- messageLog.processed()
```

두 작업이 같은 트랜잭션에 참여한다.
이렇게 하면 배송 데이터 저장이 실패했을 때 처리 완료 기록도 함께 Rollback된다.

---

## 43. 조회 후 저장 방식의 동시성 문제

예제에서는 먼저 조회한 뒤 처리 완료 ID를 저장한다.

```text
alreadyProcessed(eventId)
        |
        v
처리되지 않음
        |
        v
비즈니스 로직 실행
        |
        v
processed(eventId)
```

일반적인 단일 Kafka Consumer 흐름에서는 같은 Partition의 Record가 순차적으로 전달되므로 문제가 드러나지 않을 수 있다.
하지만 다음 상황에서는 같은 Event ID가 동시에 처리될 가능성을 생각해야 한다.

```text
같은 논리 이벤트가 서로 다른 Topic이나 Partition에 중복 발행됨

여러 Consumer 경로가 같은 처리 테이블을 공유함

운영자가 메시지를 별도로 재처리함

잘못된 Routing으로 같은 이벤트가 병렬 전달됨
```

두 트랜잭션이 동시에 조회하면 모두 처리되지 않은 이벤트로 판단할 수 있다.

```text
Consumer A
→ evt-001 없음 확인

Consumer B
→ evt-001 없음 확인

Consumer A
→ 비즈니스 로직 실행

Consumer B
→ 비즈니스 로직 실행
```

마지막 `INSERT`에서 한쪽은 Primary Key 충돌로 실패하더라도, 그전에 외부 호출이나 트랜잭션 밖의 작업을 했다면 중복 영향을 되돌리기 어렵다.
보다 안전한 구조는 처리 권한을 먼저 확보하는 것이다.

```sql
INSERT INTO consumed_message (
    event_id,
    processed_at
) VALUES (
    :eventId,
    CURRENT_TIMESTAMP
)
ON CONFLICT DO NOTHING
RETURNING event_id;
```

반환된 행이 있으면 최초 처리자다.

```text
RETURNING 결과 있음
→ 처음 처리하는 이벤트
→ 비즈니스 로직 실행
```

반환된 행이 없으면 이미 다른 처리자가 권한을 얻었거나 처리를 완료한 이벤트다.

```text
RETURNING 결과 없음
→ 중복 이벤트
→ 비즈니스 변경 생략
```

이 INSERT와 비즈니스 로직은 같은 트랜잭션에서 수행해야 한다.

```text
consumed_message INSERT
        |
        v
비즈니스 로직
        |
        v
Commit
```

비즈니스 로직이 실패하면 `consumed_message` INSERT도 Rollback되므로 이후 다시 처리할 수 있다.

## 44. `ON CONFLICT DO NOTHING`이 올바른 이유

Producer의 Outbox 저장과 Consumer의 중복 기록은 서로 다른 성격을 가진다.
Outbox INSERT에서 중복 충돌은 정상적인 상황이 아니다.

```text
같은 비즈니스 결과의 Outbox가 이미 존재함

→ Producer 로직의 가정이 깨졌을 가능성
→ Rollback과 확인 필요
```

반면 Consumer의 `consumed_message` 충돌은 At-Least-Once 구조에서 예상되는 상황이다.

```text
같은 Event ID가 다시 전달됨

→ 이미 처리된 이벤트
→ 비즈니스 로직을 다시 실행하지 않음
```

따라서 Consumer 중복 처리 테이블에서는 `ON CONFLICT DO NOTHING`을 사용할 수 있다.

```text
Outbox 중복 충돌
→ 이상 상황

Consumer 처리 기록 충돌
→ 정상적인 중복 방어
```

같은 SQL 문법이라도 사용하는 목적이 다르다.

## 45. Consumer Offset은 언제 Commit해야 하는가

Kafka Consumer는 Record를 처리한 뒤 자신이 어디까지 처리했는지 Offset을 저장한다.
Consumer DB 트랜잭션보다 Offset을 먼저 Commit하면 안 된다.

```text
Kafka Offset Commit
        |
        X Consumer DB Transaction 실패
```

Kafka는 해당 메시지가 처리되었다고 판단한다.
하지만 배송 데이터와 처리 완료 기록은 저장되지 않는다.
같은 Consumer Group에서는 해당 Record가 다시 전달되지 않을 수 있다.

```text
Offset은 넘어감
배송 데이터는 없음

→ 메시지 유실
```

따라서 처리 순서는 다음과 같아야 한다.

```text
Kafka Record 수신
        |
        v
Consumer DB Transaction
        |
        +-- consumed_message 저장
        +-- 배송 데이터 변경
        |
        v
DB Commit
        |
        v
Consumer Offset Commit
```

DB Commit 후 Offset Commit 전에 Consumer가 종료될 수 있다.

```text
배송 데이터 Commit
consumed_message Commit
        |
        X Offset Commit 전 종료
```

이 경우 같은 Record가 다시 전달된다.
하지만 처리 완료 ID가 존재하므로 비즈니스 로직을 다시 실행하지 않는다.

```text
같은 eventId 재전달
→ consumed_message 충돌
→ 배송 데이터 변경 생략
→ 정상 처리
→ Offset Commit
```

이것이 At-Least-Once Consumer의 일반적인 처리 방식이다.

## 46. Event Type에 따라 Handler를 분리하기

예제에서는 `if` 문으로 Event Type을 구분한다.

```java
if (eventType.equals("OrderCreated")) {
    shipmentService.orderCreated(payload);
} else if (
    eventType.equals("OrderLineUpdated")
) {
    shipmentService.orderLineUpdated(payload);
}
```

이벤트 종류가 적을 때는 충분하다.
하지만 Event Type이 늘어나면 조건문이 길어지고 하나의 Handler가 여러 책임을 가지게 된다.

```text
OrderCreated
OrderLineUpdated
OrderCancelled
OrderCompleted
OrderReturned
```

이 경우 Event Type별 Handler를 등록하는 구조를 사용할 수 있다.

```java
public interface OrderEventHandler {

    String supports();

    void handle(JsonNode payload);
}
```

각 Handler는 자신이 처리할 Event Type을 선언한다.

```java
@Component
public class OrderCreatedHandler
        implements OrderEventHandler {

    @Override
    public String supports() {
        return "OrderCreated";
    }

    @Override
    public void handle(JsonNode payload) {
        // 주문 생성에 따른 배송 데이터 생성
    }
}
```

Consumer는 Event Type으로 적절한 Handler를 찾는다.

```text
OrderCreated
→ OrderCreatedHandler

OrderLineUpdated
→ OrderLineUpdatedHandler
```

이렇게 하면 새로운 이벤트가 추가되어도 기존 조건문을 계속 수정하지 않아도 된다.

## 47. 알 수 없는 Event Type을 어떻게 처리할 것인가

Consumer가 알 수 없는 Event Type을 받을 수 있다.

```text
Producer
→ OrderPartiallyCancelled 추가

Consumer
→ 아직 해당 Event Type을 모름
```

이 경우 무조건 예외를 던질지, 로그를 남기고 건너뛸지는 이벤트의 성격에 따라 결정해야 한다.
중요한 상태 변경 이벤트를 조용히 무시하면 Consumer 데이터가 잘못될 수 있다.

```text
OrderCancelled 무시
→ 배송 서비스에는 계속 배송 중으로 남음
```

반대로 자신과 관련 없는 선택적인 이벤트 때문에 Consumer 전체가 중단되는 것도 바람직하지 않을 수 있다.
일반적으로 다음 기준을 정해야 한다.

```text
필수 이벤트
→ 처리할 수 없으면 실패
→ 재시도 또는 DLQ

선택적 이벤트
→ 명확한 정책에 따라 무시
→ 지표와 로그 기록
```

Event Type을 문자열로만 처리하면 오타도 런타임까지 발견되지 않는다.

```text
OrderCreated
OrderCraeted
```

가능하다면 Producer와 Consumer 사이에서 이벤트 계약을 문서화하고 Schema Registry 또는 공통 스키마 관리 방식을 사용하는 것이 좋다.

## 48. Event Payload는 외부 API 계약이다

Outbox 이벤트의 Payload는 Producer 내부 구현 정보가 아니다.
다른 서비스가 의존하는 외부 계약이다.
따라서 비즈니스 테이블의 Entity를 그대로 직렬화하는 방식은 피하는 것이 좋다.

```java
// 권장하지 않는 방식
String payload =
    objectMapper.writeValueAsString(
        purchaseOrderEntity
    );
```

Entity를 그대로 직렬화하면 다음 문제가 생긴다.

```text
JPA 연관관계가 의도치 않게 포함됨

Lazy Loading 예외가 발생할 수 있음

내부 컬럼 변경이 이벤트 스키마 변경으로 이어짐

민감한 내부 필드가 노출될 수 있음

순환 참조가 발생할 수 있음
```

외부 Consumer가 필요한 데이터만 별도의 Event DTO로 구성해야 한다.

```java
public record OrderCreatedPayload(
    long orderId,
    long customerId,
    Instant orderDate,
    List<OrderLinePayload> lineItems
) {
}
```

이렇게 하면 내부 Entity와 외부 이벤트 구조를 독립적으로 변경할 수 있다.

```text
PurchaseOrder Entity 변경
≠
OrderCreated Payload 자동 변경
```

이벤트 스키마도 REST API처럼 버전과 호환성을 관리해야 한다.

## 49. 이벤트 스키마 변경 시 호환성

이미 여러 Consumer가 이벤트를 사용하고 있다면 Payload를 마음대로 바꿀 수 없다.
예를 들어 기존 이벤트가 다음과 같다고 가정하자.

```json
{
  "orderId": 100,
  "customerId": 10
}
```

새로운 요구사항으로 `currency` 필드가 추가될 수 있다.

```json
{
  "orderId": 100,
  "customerId": 10,
  "currency": "KRW"
}
```

Consumer가 알 수 없는 필드를 무시하도록 구현되어 있다면 이런 필드 추가는 비교적 안전하다.
반면 기존 필드를 삭제하거나 타입을 변경하면 문제가 생긴다.

```text
customerId 삭제
→ 기존 Consumer 파싱 실패 가능

orderId 숫자 → 문자열 변경
→ 타입 불일치 가능
```

일반적으로 안전한 변경은 다음과 같다.

```text
선택적 필드 추가

기존 필드 의미 유지

기본값을 사용할 수 있는 변경
```

주의해야 하는 변경은 다음과 같다.

```text
필수 필드 삭제

필드 타입 변경

같은 필드의 의미 변경

Enum 값의 예상하지 못한 추가
```

큰 변경이 필요하면 Event Version을 올리고 이전 Consumer와 공존할 수 있는 기간을 둔다.

```text
OrderCreated v1
OrderCreated v2
```

---

## 50. 이벤트 발생 시각과 CDC 처리 시각

Debezium 예제의 Event Router는 Change Event의 `ts_ms`를 Kafka Value에 포함한다.
하지만 이 값이 정확히 무엇을 의미하는지 구분해야 한다.
다음 시각들은 서로 다를 수 있다.

```text
비즈니스 사건 발생 시각

DB Transaction Commit 시각

Debezium이 변경을 읽은 시각

Kafka에 Record가 저장된 시각

Consumer가 처리한 시각
```

예를 들어 주문 생성 시각은 애플리케이션에서 결정한 `orderDate`일 수 있다.
Debezium의 `ts_ms`는 CDC 처리 과정에서 생성된 기술적인 시각일 수 있다.
비즈니스 순서나 사용자 화면에 표시할 시각이 필요하다면 Payload에 별도의 필드를 저장하는 것이 좋다.

```json
{
  "eventId": "evt-001",
  "eventType": "OrderCreated",
  "occurredAt": "2026-07-27T03:00:00Z",
  "data": {
    "orderId": 100
  }
}
```

```text
occurredAt
→ 실제 비즈니스 사건 발생 시각

Debezium ts_ms
→ CDC 처리와 추적에 사용하는 기술 시각
```

둘을 혼동하면 장애로 이벤트 전달이 늦어진 경우 잘못된 비즈니스 시각이 기록될 수 있다.

## 51. Kafka Topic 보관 정책

Kafka는 이벤트를 Consumer가 읽은 직후 삭제하지 않는다.
설정된 보관 기간 동안 Topic에 유지한다.
이 덕분에 Consumer를 나중에 추가하거나 Offset을 되돌려 과거 이벤트를 다시 처리할 수 있다.

```text
OrderEvents
→ 30일 보관

새로운 Analytics Consumer 추가
→ Topic 처음부터 읽기
→ 과거 주문 통계 구성
```

하지만 모든 이벤트를 영구적으로 보관하면 저장 공간이 계속 증가한다.
보관 정책은 다음 요소를 고려해 정해야 한다.

```text
이벤트 발생량

Record 크기

재처리가 필요한 최대 기간

법적 보관 요구사항

장애 복구 가능 기간

새 Consumer 초기 구축 방식
```

이벤트가 오래 보관되지 않는다면 과거 전체 상태가 필요한 Consumer는 별도의 Snapshot이나 Cold Storage를 사용해야 한다.

```text
과거 이벤트
→ Object Storage에 보관

최근 이벤트
→ Kafka Topic에서 처리
```

---

## 52. Log Compaction을 사용할 때의 차이

Kafka Topic에는 시간 기반 보관 외에 Log Compaction을 적용할 수 있다.
Log Compaction은 같은 Key를 가진 여러 Record 중 최신 값을 중심으로 유지한다.

```text
Key = Order 100

OrderCreated
OrderUpdated
OrderCompleted

→ 최종 상태 중심으로 정리
```

이 방식은 특정 Aggregate의 최신 상태를 복원할 때 유용하다.
하지만 이벤트 하나하나의 이력을 모두 보존하는 구조와는 목적이 다르다.
일반적인 Domain Event는 변화 사실을 나타낸다.

```text
OrderCreated
OrderLineCancelled
OrderCompleted
```

이 이벤트들을 Compaction하면 중간 이력이 사라질 수 있다.
Compacted Topic을 사용하려면 각 메시지가 Aggregate의 전체 최신 상태를 담도록 설계하는 경우가 많다.

```json
{
  "orderId": 100,
  "status": "COMPLETED",
  "lineItems": [
    {
      "id": 1,
      "status": "CANCELLED"
    }
  ]
}
```

이것은 Event Stream이라기보다 State Change Log에 가까워진다.
따라서 다음 목적을 구분해야 한다.

```text
모든 사건 이력 보존
→ 일반 Retention Topic

Aggregate 최신 상태 복원
→ Compacted Topic
```

---

## 53. Outbox DELETE와 Kafka Record 삭제는 다른 의미다

Debezium 예제는 Outbox 행을 저장한 직후 같은 트랜잭션에서 삭제한다.
하지만 Outbox 테이블의 DELETE가 Kafka 이벤트 삭제로 전달되면 안 된다.

```text
Outbox 행 DELETE
→ DB 테이블 정리 목적

Kafka Record DELETE
→ Consumer에게 Aggregate 삭제를 알리는 비즈니스 의미
```

두 작업은 완전히 다르다.
따라서 Connector 설정에서 Outbox DELETE에 대한 Tombstone 생성을 비활성화한다.

```json
{
  "tombstones.on.delete": "false"
}
```

Aggregate가 실제로 삭제되었다는 사실을 전달하려면 별도의 도메인 이벤트를 만들어야 한다.

```text
Outbox 기술적 DELETE
≠ OrderDeleted 이벤트
```

주문 삭제가 비즈니스 사건이라면 다음과 같이 명시적으로 저장한다.

```text
eventType = OrderDeleted
aggregateId = 100
payload = 삭제에 필요한 데이터
```

---

## 54. processed message 테이블의 보관 정책

Consumer가 처리한 모든 Event ID를 계속 저장하면 테이블 크기가 무한히 증가한다.

```text
하루 1,000만 이벤트
→ 하루 1,000만 행 증가
```

따라서 오래된 처리 기록을 정리해야 한다.
하지만 너무 빨리 삭제하면 중복 방어가 사라진다.

```text
1. evt-001 처리 완료
2. consumed_message에서 evt-001 삭제
3. Kafka Offset을 과거로 되돌림
4. evt-001 재전달
5. 처리 기록 없음
6. 비즈니스 로직 다시 실행
```

보관 기간은 Kafka에서 해당 이벤트가 다시 전달될 수 있는 기간보다 길어야 한다.

```text
처리 기록 보관 기간
>
Kafka Topic 보관 기간
+
운영자가 Offset을 되돌릴 수 있는 기간
+
최대 장애 복구 지연
```

삭제 작업도 한 번에 대량으로 처리하지 않는 것이 좋다.

```sql
DELETE FROM consumed_message
WHERE event_id IN (
    SELECT event_id
    FROM consumed_message
    WHERE processed_at < :cutoff
    ORDER BY processed_at
    LIMIT 1000
);
```

작은 Batch로 삭제하면 긴 Lock과 대량의 WAL 생성을 줄일 수 있다.

## 55. 처리 기록만으로 모든 부작용을 막을 수 있는 것은 아니다

Consumer의 비즈니스 로직이 자신의 데이터베이스만 변경한다면 `consumed_message`와 같은 트랜잭션으로 묶을 수 있다.
하지만 외부 API를 호출하면 새로운 Dual Write 문제가 생긴다.
예를 들어 배송 서비스가 외부 택배사의 API를 호출한다고 가정하자.

```text
consumed_message 저장

+

외부 택배 API 호출
```

외부 API는 배송 서비스의 로컬 DB 트랜잭션에 포함되지 않는다.
다음 상황이 가능하다.

```text
택배 API 호출 성공
        |
        X Consumer DB Commit 전 장애
```

이벤트가 다시 전달되면 택배 API를 한 번 더 호출할 수 있다.
이 문제는 Consumer의 처리 기록만으로 완전히 해결되지 않는다.
다음과 같은 추가 설계가 필요하다.

```text
외부 Provider가 제공하는 Idempotency Key 사용

배송 서비스 내부 Outbox 사용

택배 요청 작업 테이블 저장

고유 shipmentRequestId로 중복 호출 방어
```

Outbox Pattern은 한 번 적용하고 끝나는 기술이 아니다.
서비스 경계에서 새로운 Dual Write가 나타날 때마다 같은 문제를 다시 검토해야 한다.

---

## 56. Debezium Connector가 중단되면 어떻게 되는가

Debezium Connector가 일시적으로 중단되어도 PostgreSQL에 Commit된 Outbox 변경은 WAL에 남아 있다.
Connector가 다시 시작되면 마지막으로 저장한 처리 위치부터 이어서 읽는다.

```text
Connector 중단
        |
        v
Outbox INSERT 계속 발생
        |
        v
WAL에 변경 누적
        |
        v
Connector 재시작
        |
        v
마지막 처리 위치부터 재개
```

이 구조 덕분에 애플리케이션과 Debezium이 동시에 실행되고 있을 필요는 없다.
하지만 Connector가 오래 중단되면 PostgreSQL의 WAL 보관량이 증가할 수 있다.
Debezium은 Replication Slot을 사용해 자신이 어디까지 WAL을 읽었는지 관리한다.
PostgreSQL은 아직 Consumer가 읽지 않은 WAL을 함부로 제거할 수 없다.

```text
Debezium 처리 지연
→ Replication Slot 위치 정체
→ WAL 파일 삭제 불가
→ 디스크 사용량 증가
```

따라서 다음 지표를 반드시 모니터링해야 한다.

```text
Connector 상태

Connector Lag

Replication Slot 위치

미처리 WAL 크기

PostgreSQL 디스크 사용량

Kafka 전송 실패 수
```

Outbox 테이블이 비어 있다는 사실만으로 이벤트 전달이 정상이라고 판단하면 안 된다.
즉시 삭제 구조에서는 행이 없어져도 WAL과 Connector에서 적체가 발생할 수 있다.

---

## 57. Snapshot과 Outbox 이벤트

Debezium Connector를 처음 시작하면 기존 테이블 상태를 읽는 Snapshot이 실행될 수 있다.
일반 비즈니스 테이블 CDC에서는 기존 데이터를 Kafka로 전달하기 위해 유용하다.
하지만 Outbox 테이블은 과거 이벤트를 다시 발행하면 중복이 발생할 수 있다.
Debezium Change Event의 `op` 값에서 Snapshot Record는 일반적으로 `r`로 표현된다.

```text
c
→ 새 INSERT

r
→ Snapshot에서 읽은 기존 행
```

Outbox Router는 어떤 연산을 허용할지 명확히 정해야 한다.
즉시 INSERT 후 DELETE하는 방식이라면 Connector가 시작될 때 Outbox 테이블은 보통 비어 있다.
반면 Outbox 행을 일정 기간 유지하는 방식이라면 Snapshot 설정에 따라 과거 이벤트가 다시 발행될 수 있다.

```text
Connector 재구성
→ Snapshot 실행
→ 기존 Outbox 행 읽기
→ 과거 이벤트 재발행 가능
```

따라서 다음 설정을 함께 검토해야 한다.

```text
Snapshot Mode

기존 Outbox 데이터 처리 여부

Connector 재생성 절차

중복 이벤트 처리 정책
```

Consumer 멱등성이 있더라도 대량의 과거 이벤트가 다시 발행되면 시스템 부하가 커질 수 있다.

## 58. 장애 위치별로 확인해야 할 대상

Outbox 기반 이벤트 전달은 여러 단계로 나뉜다.
문제가 발생했을 때 어느 구간에서 멈췄는지 구분해야 한다.

```text
Order Service
→ PostgreSQL
→ WAL
→ Debezium
→ Kafka
→ Consumer
→ Consumer DB
```

주문은 생성되었지만 Kafka 이벤트가 없다면 다음 순서로 확인할 수 있다.

```text
1. 주문 Transaction에서 Outbox INSERT가 실행되었는가

2. Transaction이 Commit되었는가

3. PostgreSQL WAL과 Replication Slot이 정상인가

4. Debezium Connector가 실행 중인가

5. Event Router에서 Record를 제거하지 않았는가

6. Kafka Topic Routing이 올바른가
```

Kafka에는 이벤트가 있지만 배송 데이터가 없다면 다음을 확인한다.

```text
1. Consumer Group이 Topic을 구독하고 있는가

2. Consumer Lag가 증가하고 있는가

3. Event Type을 인식하는가

4. Payload 파싱에 실패하지 않았는가

5. Consumer DB Transaction이 Rollback되지 않았는가

6. DLQ로 이동하지 않았는가
```

단순히 Kafka에 메시지가 있거나 Outbox INSERT가 있었다는 사실만으로 전체 흐름이 성공한 것은 아니다.

## 59. 운영에서 확인해야 할 주요 지표

Debezium 기반 Outbox를 운영할 때는 애플리케이션과 CDC 인프라를 함께 관찰해야 한다.
Producer 쪽에서는 다음 지표가 중요하다.

```text
Outbox 이벤트 생성 수

비즈니스 Transaction Rollback 수

이벤트 직렬화 실패 수

Event Type별 발생량
```

Debezium과 PostgreSQL에서는 다음 지표가 중요하다.

```text
Connector 상태

Connector 재시작 횟수

Source Record Poll 지연

Replication Slot Lag

미처리 WAL 크기

PostgreSQL 디스크 여유 공간
```

Kafka에서는 다음을 확인한다.

```text
Topic별 입력 처리량

Producer 오류

Partition별 Record 분포

Under-Replicated Partition

Broker 저장 지연
```

Consumer에서는 다음을 확인한다.

```text
Consumer Lag

Event Type별 성공과 실패 수

중복 이벤트 감지 수

처리 시간

DLQ 이동 수

처리 기록 테이블 크기
```

중복 감지 수가 갑자기 증가하면 Connector 재시작이나 Offset 문제가 있었을 수 있다.

```text
중복 Event ID 급증
→ Debezium 재전송
→ Connector Offset 복구
→ 운영자 Replay 수행
```

중복을 정상적으로 흡수했다고 해서 원인을 확인하지 않아도 되는 것은 아니다.

## 60. 재시도와 Dead Letter Queue

일시적인 오류는 재시도하면 성공할 수 있다.

```text
DB Connection 일시 실패

Lock Timeout

외부 시스템의 짧은 장애

일시적인 네트워크 오류
```

반면 Payload 형식이 잘못되었거나 지원하지 않는 Event Type이라면 계속 재시도해도 성공하지 않는다.

```text
필수 필드 없음

잘못된 JSON

지원하지 않는 Event Version

비즈니스 제약 조건 위반
```

모든 실패를 무한히 재시도하면 해당 Partition의 뒤 이벤트까지 처리하지 못할 수 있다.
따라서 실패를 분류해야 한다.

```text
일시적 오류
→ Backoff 후 재시도

영구적 오류
→ 정해진 횟수 후 DLQ 이동
```

DLQ에는 원본 이벤트와 오류 정보를 함께 저장한다.

```text
eventId

원본 Topic과 Partition

원본 Offset

Event Type

오류 분류

실패 시각
```

민감 정보와 전체 Stack Trace를 Payload에 그대로 넣는 것은 피해야 한다.
상세 진단 정보는 로그와 Trace 시스템에 남기고 DLQ에는 재처리에 필요한 최소 정보만 저장한다.

---

## 61. Replay를 고려한 Consumer 설계

Kafka의 장점 중 하나는 과거 이벤트를 다시 처리할 수 있다는 점이다.
하지만 Consumer가 Replay를 안전하게 지원하려면 멱등성만으로 충분하지 않을 수 있다.
이미 처리 기록이 남아 있으면 Offset을 과거로 되돌려도 모든 이벤트를 중복으로 판단해 건너뛴다.

```text
기존 View를 다시 구축하려고 Replay

→ consumed_message에 기존 Event ID 존재
→ 모든 이벤트 무시
```

조회 모델을 완전히 새로 만들 때는 별도의 구독자 ID나 새로운 Consumer Group과 저장소를 사용할 수 있다.

```text
기존 구독자
→ shipment-projection-v1

새로운 재구축 구독자
→ shipment-projection-v2
```

처리 기록 테이블을 다음처럼 구성할 수도 있다.

```sql
CREATE TABLE consumed_message (
    subscriber_id VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (
        subscriber_id,
        event_id
    )
);
```

같은 이벤트라도 서로 다른 논리 구독자는 각각 한 번씩 처리할 수 있다.

```text
shipment-projection-v1 + evt-001
→ 처리 완료

shipment-projection-v2 + evt-001
→ 별도 처리 가능
```

`subscriber_id`는 실행 중인 서버 인스턴스 ID가 아니라 안정적인 논리 Handler 이름이어야 한다.

## 62. Debezium 예제에서 주의해서 볼 부분

Debezium 공식 글은 Outbox Pattern과 CDC의 핵심을 이해하기에 좋은 예제다.
다만 실제 운영 시스템에서는 그대로 사용하기 전에 몇 가지를 보완해야 한다.
첫 번째는 중복 확인 방식이다.

```text
조회
→ 비즈니스 처리
→ 처리 ID 저장
```

동시 처리 가능성이 있다면 처리 ID 선점 방식이 더 안전할 수 있다.
두 번째는 Event Version이다.
예제 테이블에는 Event Type과 Payload가 있지만 별도의 스키마 버전 필드가 없다.
실제 시스템에서는 다음과 같은 필드를 고려할 수 있다.

```text
event_version

occurred_at

trace_id

correlation_id

tenant_id
```

세 번째는 즉시 삭제 방식이다.

```text
persist()
remove()
```

이 방식은 테이블 정리 부담을 줄이지만 운영자가 Outbox 이력을 DB에서 직접 확인하기 어렵다.
감사와 장애 조사가 중요하다면 일정 기간 Outbox 행을 보관하는 구조가 더 적합할 수 있다.
네 번째는 Custom SMT다.
글 작성 당시에는 직접 `EventRouter`를 구현했지만 이후 Debezium은 공식 Outbox Event Router SMT를 제공한다.
따라서 현재 구현에서는 공식 기능을 먼저 검토하는 것이 좋다.

---

## 63. Polling 방식과 CDC 방식 중 무엇을 선택할 것인가

두 방식 중 하나가 항상 더 좋은 것은 아니다.
Polling 방식은 애플리케이션 코드만으로 시작할 수 있다.

```text
적은 이벤트량

낮은 운영 복잡도 선호

Kafka Connect 운영 경험 부족

발행 상태를 DB에서 직접 확인해야 함
```

이런 상황에서는 Polling 방식이 현실적일 수 있다.
CDC 방식은 낮은 지연과 높은 처리량이 중요할 때 유리하다.

```text
이벤트 발생량이 많음

Polling 부하를 줄이고 싶음

PostgreSQL WAL 운영 경험이 있음

Kafka Connect 인프라를 운영할 수 있음
```

선택 기준을 정리하면 다음과 같다.
| **기준** | **Polling** | **Debezium CDC** |
| --- | --- | --- |
| 초기 구현 | 비교적 단순 | 인프라 설정 필요 |
| 애플리케이션 코드 | Publisher 직접 구현 | Publisher 코드 감소 |
| DB 부하 | 반복 조회 | WAL 기반 |
| 전달 지연 | Polling 주기에 영향 | 일반적으로 낮음 |
| 재시도 상태 | 애플리케이션 관리 | Connector 관리 |
| 운영 난이도 | 애플리케이션 중심 | DB·Kafka Connect 포함 |
| 발행 이력 조회 | 상태 테이블로 쉬움 | 별도 관찰 도구 필요 |
| 순서 | 병렬 Polling에서 복잡 | WAL 순서 활용 가능 |

작은 프로젝트에서는 Polling으로 시작하고 이벤트량과 운영 요구가 커질 때 CDC로 전환할 수도 있다.
중요한 것은 전달 방법이 아니라 Outbox의 핵심 불변식을 유지하는 것이다.

```text
비즈니스 데이터가 Commit되었다면
발행할 이벤트도 같은 DB Transaction에 남아 있어야 한다.
```

---

## 64. 전체 처리 흐름

Transactional Outbox와 Debezium을 이용한 전체 과정은 다음과 같다.

```text
Client
        |
        | 주문 생성 요청
        v
Order Service
        |
        v
PostgreSQL Transaction
        |
        +-- purchase_order INSERT
        +-- outbox_event INSERT
        +-- outbox_event DELETE
        |
        v
Commit
        |
        v
PostgreSQL WAL
        |
        v
Debezium Connector
        |
        +-- Commit된 변경만 읽음
        +-- Outbox INSERT 감지
        |
        v
Outbox Event Router
        |
        +-- aggregateType으로 Topic 결정
        +-- aggregateId를 Kafka Key로 사용
        +-- eventId를 Header에 추가
        +-- Payload를 Value로 변환
        |
        v
Kafka
        |
        v
Shipment Consumer
        |
        v
Consumer DB Transaction
        |
        +-- consumed_message INSERT
        +-- 배송 데이터 변경
        |
        v
Commit
        |
        v
Consumer Offset Commit
```

각 단계에서 사용하는 식별자의 역할도 다르다.

```text
aggregateId
→ 같은 Aggregate 이벤트의 Partition Key

eventId
→ 같은 논리 이벤트의 중복 처리 방지

Kafka Offset
→ Consumer Group의 물리적 처리 위치
```

세 값을 혼동하면 안 된다.

## 65. 핵심 정리

Transactional Outbox Pattern은 데이터베이스와 Kafka를 하나의 분산 트랜잭션으로 묶지 않는다.
대신 애플리케이션이 자신의 데이터베이스만 안전하게 변경하도록 만든다.

```text
비즈니스 데이터 저장
+
발행할 이벤트 저장
+
같은 DB Transaction
```

Debezium은 PostgreSQL WAL을 읽어 Commit된 Outbox INSERT를 Kafka로 전달한다.
Polling처럼 테이블을 반복 조회하지 않으며, Outbox Event Router가 데이터베이스 변경 이벤트를 외부 Consumer가 사용할 도메인 이벤트로 변환한다.

```text
aggregateType
→ Kafka Topic

aggregateId
→ Kafka Key

eventId
→ 중복 처리 식별자

payload
→ 외부 이벤트 계약
```

같은 Aggregate ID를 Key로 사용하면 같은 Aggregate의 이벤트가 같은 Partition으로 전달된다.
하지만 모든 이벤트에 대한 전역 순서를 보장하는 것은 아니다.
Debezium 기반 전달도 At-Least-Once를 전제로 한다.
Kafka 발행 후 처리 위치 저장 전에 장애가 발생하면 같은 Event ID가 다시 전달될 수 있다.
따라서 Consumer는 처리 기록과 비즈니스 데이터 변경을 같은 로컬 DB 트랜잭션에 저장해야 한다.

```text
consumed_message INSERT
+
Consumer 비즈니스 데이터 변경
+
같은 Transaction
```

Consumer DB Commit 뒤에 Kafka Offset을 Commit하면 Commit 이후 장애로 메시지가 재전달되어도 Event ID를 이용해 중복 반영을 막을 수 있다.
다만 Consumer가 외부 API를 호출하면 새로운 Dual Write 문제가 생긴다.
이때는 외부 Provider의 Idempotency Key나 Consumer 내부 Outbox 같은 추가 설계가 필요하다.
결국 Outbox Pattern의 핵심은 메시지를 물리적으로 정확히 한 번 보내는 데 있지 않다.

```text
DB에 확정된 비즈니스 결과에 대응하는 이벤트가
장애가 발생해도 사라지지 않도록 저장하고

중복 전달이 발생하더라도
최종 비즈니스 결과를 한 번만 반영하도록 만드는 것
```

이 구조를 통해 서비스 내부에서는 Commit 직후 변경된 데이터를 조회할 수 있고, 다른 서비스에는 Kafka를 통해 신뢰할 수 있는 비동기 방식으로 이벤트를 전달할 수 있다.
