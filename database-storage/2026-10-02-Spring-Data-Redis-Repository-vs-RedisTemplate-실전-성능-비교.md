# Spring Data Redis: Repository vs RedisTemplate — 실전 성능 비교

- 읽은 날짜: 2026-10-02
- 회사/작성 주체: 여기 어때
- 원문: [Spring Data Redis: Repository vs RedisTemplate — 실전 성능 비교](https://techblog.gccompany.co.kr/spring-data-redis-repository-vs-redistemplate-%EC%8B%A4%EC%A0%84-%EC%84%B1%EB%8A%A5-%EB%B9%84%EA%B5%90-3e1a6ab8bda3)
- 노션 정리: [원본 페이지](https://app.notion.com/p/3e6a987f6c4080cf8abef85b76f27d57)

---

## Redis Repository를 썼더니 CPU가 100%까지 올라간 이유

여기어때 전시개발팀에서는 상품 데이터를 빠르게 제공하기 위해 Redis를 캐시로 사용하고 있었다.
처음에는 `Spring Data Redis Repository`를 사용했다.

```java
public interface StayPdpRepository
        extends CrudRepository<StayPdp, Long> {
}
```

코드도 단순하고

```text
save()

findById()

delete()
```

같은 CRUD를 바로 사용할 수 있어서 편리했다.
하지만 트래픽과 캐시 데이터가 커지면서 문제가 발생했다.

```text
피크 타임

Redis CPU
85%
 ↓
90%
 ↓
100%

     │
     ▼

API 응답시간 3~5배 증가

     │
     ▼

일부 요청 Timeout
```

실제 운영 환경에서는 피크 시간 Redis CPU가 100%까지 유지됐고, 일부 요청에서는 Timeout까지 발생했다.

---

## 1. 처음에는 왜 문제가 없었을까?

처음에는

```text
데이터 수 ↓

객체 크기 ↓

동시 요청 ↓
```

였기 때문에 Repository의 추가 작업이 크게 드러나지 않았다.
하지만 시간이 지나면서

```text
캐시 객체 크기 증가

        +

사용자 증가

        +

캐시 저장 요청 증가
```

가 동시에 발생했다.
그리고 이때 `Redis Repository`가 객체 하나를 저장하기 위해 내부적으로 생각보다 많은 일을 하고 있다는 사실이 드러났다.

---

## 2. Redis Repository는 객체를 어떻게 저장할까?

다음과 같은 객체를 Redis에 저장한다고 해보자.

```text
StayPdp

├─ placeId
├─ placeName
├─ reviewRate
├─ reviewCount
└─ lowestPrice
     ├─ discountPrice
     ├─ roomId
     └─ ...
```

`@RedisHash`를 사용하면 Spring Data Redis Repository는 객체를 Redis **Hash** 형태로 변환한다.

```text
Java Object

StayPdp
   │
   ▼
Spring Data Redis
   │
   ▼
Redis Hash
```

예를 들어:

```text
StayPdp:88146

placeId                                  88146
placeName                                홍대...
reviewCount                              26
reviewRate                               90.0
lowestPrice.discountPrice                16290
lowestPrice.roomId                       752543
...
```

실제 Redis에서는 이런 형태의 `HMSET` 명령이 실행된다.

---

## 3. 문제는 실제 객체가 훨씬 컸다

예제 객체는 단순했지만 실제 프로덕션 객체는 훨씬 복잡했다.

```text
StayPdp

├─ 기본 정보
│
├─ roomGroups 14개
│    │
│    ├─ 기본 정보
│    ├─ stayInfo
│    ├─ additionalInfo
│    ├─ facilityInfo
│    └─ bedInfoList
│
├─ filterRoomGroups
│
└─ 기타 정보
```

하나의 객체가 Redis Hash로 펼쳐지면서 최대

```text
약 1,600 ~ 2,000 Fields
```

가 만들어졌다.
즉 Repository에서 `save()` 한 번을 호출하면 Redis 입장에서는 대략 이런 요청을 받는다.

```text
HMSET StayPdp:72244

_class                  ...
id                      72244

roomGroups[0].roomId
roomGroups[0].roomName
roomGroups[0].stayInfo.checkInHour
roomGroups[0].stayInfo.checkOutHour
roomGroups[0].stayInfo.discountPrice
...

roomGroups[13]....

...

총 1,600 ~ 2,000 Fields
```

명령은 하나처럼 보여도 Redis가 처리해야 할 데이터 양 자체가 상당히 커진다.

---

## 4. 그런데 HMSET만 실행되는 것도 아니다

Repository는 단순히 데이터를 Hash에 넣는 것으로 끝나지 않는다.
Spring Data Redis Repository는 Repository 기능을 제공하기 위해 **별도의 Set과 Index 구조도 관리한다.**

```text
Redis

┌─────────────────────────────┐
│ StayPdp:88146               │
│                             │
│ 실제 데이터 Hash            │
└─────────────────────────────┘

┌─────────────────────────────┐
│ StayPdp                     │
│                             │
│ 전체 객체 ID 목록 Set        │
└─────────────────────────────┘
```

그래서 객체 하나를 저장하면

```text
HMSET
  +
SADD
```

가 발생한다.

---

## 5. 왜 Set까지 필요한가?

Redis는 기본적으로

```text
Key → Value
```

형태로 접근하는 Key-Value Store다.
예를 들어

```text
GET product:100
```

처럼 특정 Key를 알고 있다면 조회하기 쉽다.
하지만 Repository에서는 이런 기능도 제공한다.

```text
findAll()

count()

findByCategory(...)
```

이런 기능을 지원하려면
> Redis에 현재 어떤 객체들이 존재하는지
알아야 한다.
그래서 Spring Data Redis Repository가 별도의 Set을 관리한다.

```text
StayPdp

[
  100,
  101,
  102,
  103,
  ...
]
```

그리고

```text
SCARD StayPdp
```

를 사용하면 전체 개수를 구할 수 있다.

```text
SMEMBERS StayPdp
```

를 사용하면 전체 ID 목록도 얻을 수 있다.

---

## 6. @Indexed를 쓰면 더 많은 자료구조가 생긴다

예를 들어:

```java
@RedisHash("Product")
public class Product {

    @Id
    private Long id;

    @Indexed
    private String category;
}
```

`category`로 검색하고 싶다면 Redis에는 또 다른 Set이 만들어진다.

```text
Product:category:electronics

[
   productId 1
   productId 2
   productId 3
]
```

그래서

```java
findByCategory("electronics")
```

가 호출되면

```text
Category Index Set
       │
       ▼
ID 목록 조회
       │
       ▼
실제 Hash 조회
```

과정으로 데이터를 찾는다.
편리한 Repository 기능 뒤에서 Redis 자료구조가 추가로 만들어지고 있는 것이다.

---

## 7. save() 하나가 실제로 하는 일

Repository 코드에서는 단순하게

```java
repository.save(pdp);
```

한 줄이다.
하지만 내부에서는 여러 Redis 명령이 실행될 수 있다.

```text
repository.save()

       │
       ▼
기존 데이터 확인 / 정리

       │
       ▼
DEL

       │
       ▼
HMSET
객체 Hash 저장

       │
       ▼
SADD
전체 ID Set 관리

       │
       ▼
SADD
@Indexed 관리

       │
       ▼
SADD
Index Key 관리
```

즉 하나의 Java 메서드가 실제 Redis에서는 여러 명령으로 확장된다.

---

## 8. 왜 Redis CPU가 올라갔을까?

이제 문제를 연결해보자.
하나의 캐시 객체가 2,000개 가까운 Field를 가진다.

```text
한 번의 저장

HMSET
2,000 Fields

+

SADD
```

이 요청이 피크 시간에 초당 수백 번 발생한다.

```text
Request 1 ──▶ HMSET + SADD

Request 2 ──▶ HMSET + SADD

Request 3 ──▶ HMSET + SADD

Request 4 ──▶ HMSET + SADD

...

수백 요청 / sec
```

Redis 입장에서는

```text
큰 Hash 처리

+

많은 데이터 전송

+

Set 관리

+

Index 관리

+

명령 수 증가
```

를 모두 수행해야 한다.
결국

```text
Traffic 증가
      │
      ▼
Repository.save() 증가
      │
      ▼
Redis 명령 증가
      │
      ▼
Redis 처리량 증가
      │
      ▼
CPU 상승
      │
      ▼
CPU 100%
      │
      ▼
Latency 증가
      │
      ▼
Timeout
```

으로 이어졌다.
원문에서는 Repository의 Hash 변환과 보조 Set 관리가 부하를 누적시킨 것이 CPU 포화의 주요 원인이었다고 설명한다.

---

## 9. 그런데 캐시에서 이런 기능이 정말 필요했을까?

여기서 핵심 질문이 나온다.
이 Redis의 목적은 **캐시**였다.
즉 필요한 기능은 거의 이것뿐이었다.

```text
Key로 저장

Key로 조회

TTL 만료
```

하지만 Repository는

```text
findAll()

count()

Secondary Index

Hash Mapping

Index 관리
```

등 훨씬 많은 기능을 제공하고 있었다.

```text
필요했던 것

Key → Value


실제로 사용했던 것

Repository
 │
 ├─ Hash
 ├─ 전체 Key Set
 ├─ Secondary Index
 ├─ 객체 Mapping
 └─ 여러 Redis Command
```

즉 필요 이상의 기능 때문에 비용을 지불하고 있었다.

---

## 10. 그래서 RedisTemplate로 변경했다

해결 방법은 Redis를 훨씬 단순하게 사용하는 것이었다.

```java
redisTemplate.opsForValue().set(
    "staypdp:" + pdp.placeId(),
    pdp,
    Duration.ofMinutes(10)
);
```

Redis에서는 다음 명령 하나가 실행된다.

```text
SET staypdp:88146 {JSON...} EX 600
```

끝이다.
구조를 비교하면 차이가 명확하다.

---

## 11. Repository 내부 구조

```text
                 Java Object
                      │
                      ▼
             Spring Data Redis
                      │
             Object Mapping
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼

        Hash        ID Set      Index Set

          │           │           │
          ▼           ▼           ▼

        HMSET        SADD        SADD
```

객체를 Redis 친화적인 여러 자료구조로 변환해서 관리한다.

---

## 12. RedisTemplate 구조

```text
Java Object
     │
     ▼
JSON Serialize
     │
     ▼
Redis String

Key
 │
 ▼
Value
```

그리고 Redis 명령은

```text
SET key value EX TTL
```

하나다.

```text
Repository

Object
  │
  ├──▶ Hash
  ├──▶ Set
  └──▶ Index


RedisTemplate

Object
  │
  ▼
String
```

캐시라는 목적만 놓고 보면 후자가 훨씬 단순하다.

---

## 13. TTL도 한 번에 처리된다

Repository에서는 데이터 저장과 TTL 관리가 내부적인 여러 작업과 함께 처리된다.
반면 RedisTemplate의 String 방식에서는

```text
SET key value EX 600
```

한 번으로

```text
데이터 저장

+

TTL 설정
```

을 같이 처리할 수 있다.

---

## 14. 읽을 때도 차이가 있다

Repository는 Redis Hash를 사용하므로 조회할 때

```text
HGETALL StayPdp:88146
```

을 사용한다.

```text
Hash

field1 → value1
field2 → value2
field3 → value3
...
```

모든 Field를 읽은 뒤 다시 Java Object로 Mapping해야 한다.
반면 RedisTemplate은

```text
GET staypdp:88146
```

을 실행한다.

```text
JSON
 │
 ▼
Deserialize
 │
 ▼
Java Object
```

둘 다 Redis 명령 자체는 하나지만 처리 형태가 다르다.

---

## 15. 전체 동작을 비교하면

#### Repository

```text
Application

repository.save()

       │
       ▼
Object Mapping
       │
       ▼
기존 데이터 / Index 처리
       │
       ▼
DEL
       │
       ▼
HMSET
1,600~2,000 Fields
       │
       ▼
SADD
전체 ID 관리
       │
       ▼
SADD
Index 관리
       │
       ▼
Redis
```

#### RedisTemplate

```text
Application

cache.save()

       │
       ▼
JSON Serialize
       │
       ▼
SET Key Value EX TTL
       │
       ▼
Redis
```

차이는 단순히

```text
Repository API

vs

RedisTemplate API
```

가 아니다.
실제로는 **Redis에 데이터를 어떤 자료구조로 저장하고 어떤 부가 기능을 유지하는가**의 차이다.

---

## 16. 결과

RedisTemplate로 변경한 이후 실제 운영 환경에서는

```text
Before

Redis Repository

CPU
100%

응답시간
약 150ms

Timeout 발생
```

에서

```text
After

RedisTemplate

CPU
약 10%

응답시간
약 50ms

Timeout 제거
```

수준으로 개선됐다고 한다.
단순히 라이브러리를 바꿔서 빨라진 것이 아니다.

```text
불필요한 Redis 명령 제거

        +

불필요한 자료구조 제거

        +

객체 Mapping 단순화

        +

Redis가 해야 하는 작업량 감소

        ↓

CPU 감소
```

라고 보는 것이 정확하다.

---

## 17. 이 사례에서 중요한 점

처음 보면 문제는 단순하다.

```text
Redis CPU = 100%

       ↓

Redis가 느리다?
```

하지만 실제 원인은 Redis 자체라기보다 **Redis를 사용하는 방법**에 있었다.

```text
Spring Data Redis Repository
        │
        ▼
편리한 추상화
        │
        ▼
Hash 변환
        │
        ├─ 전체 Key Set
        ├─ Secondary Index
        └─ 다수 Redis Command
        │
        ▼
큰 객체 + 높은 트래픽
        │
        ▼
Redis CPU 100%
```

그리고 캐시의 요구사항을 다시 살펴보니

```text
필요한 것

Key → Value

TTL
```

뿐이었다.
그래서

```text
Repository

     ↓

RedisTemplate

     ↓

SET 1회
```

라는 훨씬 단순한 구조로 변경했다.

---

## 정리

Spring Data Redis Repository 자체가 나쁜 것은 아니다.
Repository는

```text
findAll()

count()

Secondary Index

Derived Query
```

같은 기능을 편리하게 제공하기 위해 Redis 위에 추가적인 자료구조와 명령을 사용한다.
문제는 **단순 캐시인데 이 기능들이 전혀 필요하지 않았다는 것**이다.

```text
단순 Cache

Key → Value
      │
      ▼
Repository 사용
      │
      ▼
Hash + Set + Index
      │
      ▼
불필요한 작업 증가
```

따라서 이번 사례의 핵심은
> Repository보다 RedisTemplate이 무조건 빠르다.
가 아니다.

```text
Redis에서 어떤 기능이 필요한가?

        ↓

그 기능을 위해
어떤 자료구조를 만드는가?

        ↓

실제로 Redis에
어떤 명령이 나가는가?

        ↓

트래픽이 증가했을 때
그 비용이 얼마나 커지는가?
```

를 보고 기술을 선택해야 한다는 것이다.
**추상화가 편리할수록 그 아래에서 실제로 어떤 명령이 실행되는지 확인하는 것이 중요하다.**

---

## 18. 실제로 성능 차이는 얼마나 날까?

Repository와 RedisTemplate의 구조가 다른 것은 확인했다.
그렇다면 실제 성능도 차이가 날까?
이를 확인하기 위해 저장, 조회, 저장+조회의 세 가지 작업을 비교했다.

```text
Benchmark

1. Save

2. Get

3. Save + Get
```

JMH를 이용해 각각의 평균 실행 시간을 측정했다.

---

## 19. 저장 성능

먼저 객체 하나를 저장하는 경우다.
결과는 다음과 같았다.

```text
Repository

약 2,272 μs


RedisTemplate

약 926 μs
```

RedisTemplate이 약 **2.45배 빨랐다.**
왜 저장에서 차이가 크게 났을까?
앞에서 본 내부 동작을 다시 보면 이해하기 쉽다.

```text
Repository

save()
   │
   ▼
Hash 변환
   │
   ▼
HMSET
   │
   ▼
SADD
   │
   ▼
필요한 경우
Index 관리
```

반면 RedisTemplate은

```text
RedisTemplate

save()
   │
   ▼
Serialize
   │
   ▼
SET
```

으로 끝난다.
즉 단순한 캐시 저장에서는 **Repository의 부가 기능이 그대로 추가 비용**이 된 것이다.

---

## 20. 조회 성능

조회에서도 차이가 있었다.

```text
Repository

약 2,035 μs


RedisTemplate

약 1,509 μs
```

RedisTemplate이 약 **1.35배 빨랐다.**
조회 시 실행되는 Redis 명령을 비교하면

```text
Repository

HGETALL StayPdp:999


RedisTemplate

GET staypdp:999
```

형태다.
Repository는 Hash 전체를 가져온 뒤 Spring Data의 객체 Mapping 과정을 거친다.

```text
Redis Hash
    │
    ▼
HGETALL
    │
    ▼
Field Mapping
    │
    ▼
Java Object
```

RedisTemplate에서는 직렬화된 Value 하나를 가져와 역직렬화한다.

```text
Redis String
    │
    ▼
GET
    │
    ▼
Deserialize
    │
    ▼
Java Object
```

저장만큼 큰 차이는 아니지만 조회에서도 Repository의 Mapping 비용이 존재했다.

---

## 21. 저장 + 조회에서는 차이가 줄었다

이번에는 실제 캐시 사용처럼 저장한 뒤 바로 조회했다.

```text
Repository

약 3,291 μs


RedisTemplate

약 3,192 μs
```

차이는 약 **1.03배**로 크게 줄었다.
왜 저장 성능에서는 차이가 컸는데 여기서는 줄었을까?
원문에서는 네트워크 왕복 시간도 전체 실행시간에서 상당한 비중을 차지하기 때문이라고 분석했다.

```text
Application
     │
     │ Network RTT
     ▼
Redis
     │
     │ Network RTT
     ▼
Application
```

Redis 연산 자체의 차이뿐 아니라 네트워크 비용까지 섞이기 때문에 여러 연산을 합친 전체 시간에서는 차이가 상대적으로 작아질 수 있다.

---

## 22. 명령어 수만 봐도 차이가 난다

Repository가 제공하는 기능이 많아질수록 실제 Redis 명령도 늘어난다.

#### 저장

```text
Repository

SISMEMBER
    │
    ▼
DEL
    │
    ▼
HMSET
    │
    ▼
SADD
    │
    ▼
Index 처리

최소 여러 개의 명령


RedisTemplate

SET

1개
```

#### 조회

```text
Repository

HGETALL


RedisTemplate

GET
```

#### 삭제

```text
Repository

HGETALL
   │
   ▼
DEL
   │
   ▼
SREM
   │
   ▼
Index 정리


RedisTemplate

DEL
```

Repository는 Redis 위에서 Repository 추상화를 구현해야 하기 때문에 저장·삭제 시 특히 많은 작업이 필요하다.

---

## 23. 메모리 사용량도 달라진다

CPU만 문제가 아니었다.
Repository는 실제 데이터 외에 추가 자료구조도 저장한다.

```text
Repository

Redis

├─ StayPdp:999
│    └─ 실제 Hash
│
├─ StayPdp
│    └─ 전체 ID Set
│
└─ Index
     └─ 추가 Set
```

반면 RedisTemplate을 단순 캐시로 사용하면

```text
RedisTemplate

Redis

└─ staypdp:999
      └─ 실제 Value
```

만 존재한다.
원문 테스트에서는 대략

```text
Repository

약 1,400 Bytes


RedisTemplate

약 860 Bytes
```

수준으로 약 **1.6배 차이**가 확인됐다.
캐시 데이터가 몇 개 없으면 큰 차이가 아니다.
하지만

```text
100개

1,000개

100,000개

1,000,000개
```

로 증가하면 이야기가 달라진다.

```text
객체당 추가 메모리
      │
      ▼
수십만 ~ 수백만 객체
      │
      ▼
메모리 사용량 차이 확대
```

Redis에서는 메모리 자체가 중요한 자원이기 때문에 불필요한 Index나 Set도 비용이 된다.

---

## 24. 실제 운영 환경에서는 더 큰 차이가 났다

벤치마크보다 중요한 것은 실제 서비스 결과다.
Repository를 사용하던 당시에는

```text
Redis CPU

100%


API 평균 응답

약 150ms


일부 요청

Timeout
```

문제가 있었다.
RedisTemplate으로 변경한 이후에는

```text
Redis CPU

약 10%


API 평균 응답

약 50ms


Timeout

제거
```

수준으로 개선됐다.
즉 단순한 마이크로 벤치마크보다 실제 고트래픽 환경에서 차이가 훨씬 크게 나타났다.

---

## 25. 왜 트래픽이 많아지면 차이가 더 커질까?

Redis는 요청 하나만 보면 굉장히 빠르다.
그래서 개발 환경에서는

```text
1개 요청

HMSET + SADD

→ 별 문제 없음
```

처럼 보일 수 있다.
하지만 피크 시간에 초당 수백 개의 저장 요청이 들어오면

```text
Request 1 ─▶ HMSET + SADD + ...

Request 2 ─▶ HMSET + SADD + ...

Request 3 ─▶ HMSET + SADD + ...

Request 4 ─▶ HMSET + SADD + ...

...

수백 번 / sec
```

가 된다.
그리고 객체 하나가 2,000개 가까운 Field를 가지고 있다면 Redis가 처리해야 하는 양도 함께 커진다.

```text
큰 객체
   │
   ×
높은 QPS
   │
   ×
여러 Redis Command
   │
   ▼
Redis CPU 증가
```

작은 오버헤드라도 **데이터 크기와 트래픽이 곱해지면 운영에서는 큰 문제가 될 수 있다.**

---

## 26. 그렇다면 Repository를 쓰면 안 될까?

그렇지는 않다.
Spring Data Redis Repository가 제공하는 기능이 실제로 필요하다면 충분히 사용할 이유가 있다.
예를 들어

```text
findAll()

count()

exists()

@Indexed 조회

Derived Query
```

같은 기능이 필요하다면 Repository가 편리하다.

```text
Application

findByCategory("HOTEL")
       │
       ▼
Repository
       │
       ▼
Redis Secondary Index
       │
       ▼
ID 검색
       │
       ▼
Object 조회
```

직접 이런 Index를 구현하지 않아도 된다는 장점이 있다.

---

## 27. Repository가 잘 맞는 경우

원문에서는 Redis를 단순 캐시가 아니라 **데이터 저장소처럼 사용하는 경우**를 예로 든다.
예를 들어 실시간 투표 시스템이라고 해보자.

```text
Vote

├─ id
├─ userId
├─ pollId
└─ votedAt
```

여기서는

```text
특정 User가 투표했는가?

특정 Poll의 투표 목록은?

전체 Vote 수는?

특정 조건의 Vote는?
```

같은 조회가 필요할 수 있다.

```text
Redis
  │
  ├─ Primary Key
  ├─ userId Index
  └─ pollId Index
```

이런 경우 Repository가 제공하는 Index와 Query 기능을 활용할 가치가 있다.

---

## 28. 설정이나 메타데이터 관리에도 사용할 수 있다

데이터가 많지 않고 다양한 조건으로 조회해야 하는 경우도 있다.
예를 들어 Feature Flag다.

```text
FeatureFlag

├─ flagName
├─ environment
├─ enabled
└─ description
```

필요한 조회가

```text
DEV 환경 Feature Flag 조회

PROD Feature Flag 조회

특정 Flag 존재 확인
```

같다면 `@Indexed`가 편리할 수 있다.

```text
작은 데이터 규모

       +

조회 조건 다양

       +

변경 빈도 낮음

       ↓

Repository 사용 고려
```

원문에서는 수백\~수천 건 수준의 설정 데이터 관리 같은 사례를 예로 들었다.

---

## 29. 단순 캐시라면 RedisTemplate이 더 자연스럽다

반대로 Redis의 목적이 단순 캐시라면 필요한 것은 대부분 이것뿐이다.

```text
Key

   ↓

Value

   +

TTL
```

예를 들어 상품 상세 캐시라면

```text
product:123
      │
      ▼
Product JSON
      │
      ▼
TTL 10분
```

이면 충분하다.

```text
Application
    │
    ▼
RedisTemplate
    │
    ▼
SET product:123 {...} EX 600
```

이 경우 Repository의

```text
전체 ID Set

Secondary Index

findAll()

count()
```

기능은 필요하지 않다.
그러면 그 기능을 유지하는 비용 역시 불필요하다.

---

## 30. Spring Cache도 좋은 선택이다

캐시가 목적이라면 RedisTemplate을 직접 사용하는 것 외에도 Spring Cache를 사용할 수 있다.

```java
@Cacheable(
    value = "products",
    key = "#id"
)
public Product getProduct(Long id) {
    return productService.findById(id);
}
```

애플리케이션에서는

```text
getProduct()
     │
     ▼
Cache 존재?
     │
  ┌──┴──┐
 YES    NO
  │      │
  ▼      ▼
Redis   DB 조회
         │
         ▼
       Cache 저장
```

형태로 동작한다.
단순 DB 조회 결과나 API 응답을 캐싱하는 용도라면 이런 구조가 Repository보다 목적에 더 잘 맞을 수 있다.

---

## 31. 특히 대량 캐시에서는 Repository 사용을 조심해야 한다

캐시가 수십만\~수백만 건으로 증가한다고 해보자.
Repository는 전체 ID를 관리하기 위해 하나의 Set을 사용할 수 있다.

```text
StayPdp

[
 ID 1
 ID 2
 ID 3
 ...
 ID 1,000,000
]
```

그리고 `@Indexed`까지 있다면 Index Set들도 계속 증가한다.

```text
Product:category:HOTEL

[
 1
 2
 3
 ...
]
```

결국 실제 Cache 데이터뿐 아니라

```text
실제 데이터

+

Key 목록

+

Secondary Index

+

Index 관리 데이터
```

까지 Redis Memory를 사용한다.
원문에서도 수십만\~수백만 건의 단순 캐시라면 RedisTemplate 방식이 더 적절하다고 설명한다.

---

## 32. RedisTemplate이라고 무조건 String만 써야 하는 것은 아니다

여기서 하나 주의해야 한다.
핵심은

```text
Repository = 나쁨

RedisTemplate = 좋음
```

이라는 것이 아니다.
RedisTemplate은 Redis의 여러 자료구조를 직접 선택할 수 있다.

```text
opsForValue()
    → String

opsForHash()
    → Hash

opsForSet()
    → Set

opsForZSet()
    → Sorted Set

opsForList()
    → List
```

즉 중요한 것은
> **내 데이터 접근 패턴에 맞는 Redis 자료구조를 직접 선택한다**
는 점이다.
이번 사례에서는 단순 Cache였기 때문에 `String + SET`이 가장 단순했던 것이다.

---

## 33. 내부 동작을 직접 확인하려면 Redis MONITOR

이 문제에서 중요한 점은 Spring 코드만 보고 판단하지 않았다는 것이다.
Redis가 실제 어떤 명령을 받고 있는지 확인했다.

```bash
redis-cli
```

이후

```bash
MONITOR
```

를 실행한다.
그러면 Redis에 들어오는 명령을 확인할 수 있다.

```text
HMSET ...

SADD ...

DEL ...

HGETALL ...

SET ...
```

따라서

```text
repository.save()
```

한 줄을 실행했는데

```text
왜 Redis 명령이 여러 개 나가지?
```

를 직접 확인할 수 있다.

---

## 34. Redis 자료구조도 직접 확인해야 한다

Redis에 어떤 Key가 생겼는지도 확인할 수 있다.
Repository를 사용했다면

```text
StayPdp:88146

StayPdp

StayPdp:88147

...
```

처럼 데이터 이외의 Key가 보일 수 있다.
각 Key의 Type을 보면

```text
TYPE StayPdp:88146

→ hash


TYPE StayPdp

→ set
```

처럼 어떤 자료구조가 만들어졌는지 알 수 있다.
RedisTemplate의 String Cache라면

```text
TYPE staypdp:88146

→ string
```

정도로 끝난다.

---

## 35. 메모리도 확인해야 한다

CPU만 보는 것도 부족하다.
Redis에서는

```text
INFO memory
```

나

```text
MEMORY USAGE key
```

를 통해 실제 메모리 사용량을 확인할 수 있다.

```text
Redis Monitoring

CPU

Memory

Command Count

Latency

Network

Eviction
```

을 함께 봐야 한다.
특히 Repository를 사용한다면 데이터 Hash만 보는 것이 아니라

```text
ID Set

Index Set

Index 관리 Key
```

가 차지하는 메모리까지 확인해야 한다.

---

## 36. P50보다 P95, P99도 확인하자

평균 응답시간만 보면 장애 징후를 놓칠 수 있다.
예를 들어

```text
대부분 요청

5ms


일부 요청

500ms
```

라면 평균은 크게 나빠 보이지 않을 수도 있다.
그래서

```text
P50

P95

P99
```

같은 Percentile Latency를 함께 확인하는 것이 좋다.
원문의 최적화 체크리스트에서도 Repository와 Template을 비교할 때 P50/P95/P99 응답시간 비교를 권장한다.

---

## 37. Repository 사용 전에 확인할 것

Repository를 도입하기 전에 다음 질문을 던져볼 수 있다.

```text
정말 findAll()이 필요한가?

        │

정말 count()가 필요한가?

        │

@Indexed 검색이 필요한가?

        │

Redis를 Cache로 쓰는가?
DB처럼 쓰는가?

        │

데이터가 얼마나 많아질까?

        │

피크 트래픽은 얼마나 될까?
```

필요한 기능이 단순히

```text
GET

SET

TTL
```

이라면 Repository가 지나치게 큰 추상화일 가능성이 있다.

---

## 38. 전체 문제를 다시 보면

처음에는 단순한 캐시였다.

```text
DB / Service
     │
     ▼
Redis Cache
```

편하게 구현하기 위해 Repository를 선택했다.

```text
Application
     │
     ▼
Spring Data Redis Repository
```

하지만 실제 내부에서는

```text
Java Object
     │
     ▼
Hash Mapping
     │
     ├──▶ Hash
     ├──▶ Key Set
     ├──▶ Index Set
     └──▶ 여러 Redis Command
```

가 발생했다.
여기에

```text
객체 크기 증가

+

트래픽 증가
```

가 겹치면서

```text
Redis 작업량 증가
       │
       ▼
CPU 100%
       │
       ▼
Latency 증가
       │
       ▼
Timeout
```

으로 이어졌다.

---

## 39. 해결 흐름

문제를 해결한 과정은 다음과 같이 정리할 수 있다.

```text
Redis CPU 100%
      │
      ▼
Redis MONITOR 확인
      │
      ▼
Repository 내부 명령 분석
      │
      ▼
HMSET + SADD + Index...
      │
      ▼
"이 기능들이 정말 필요한가?"
      │
      ▼
단순 Cache라는 요구사항 확인
      │
      ▼
RedisTemplate
      │
      ▼
SET 1회
      │
      ▼
Redis 작업량 감소
      │
      ▼
CPU 감소
      │
      ▼
Latency 개선
```

즉 Redis 서버의 스펙을 먼저 올린 것이 아니라 **애플리케이션이 Redis에 어떤 작업을 시키고 있는지를 먼저 줄였다.**

---

## 40. 최종 정리

Repository와 RedisTemplate을 단순히 비교하면

```text
Repository

편리함 ↑
추상화 ↑
부가 기능 ↑
Redis 작업량 ↑


RedisTemplate

직접 구현 ↑
제어권 ↑
단순화 가능
Redis 작업량 ↓
```

라고 볼 수 있다.
하지만 핵심은 어떤 기술이 무조건 더 좋다는 것이 아니다.

```text
요구사항
   │
   ▼
필요한 조회 패턴
   │
   ▼
필요한 Redis 자료구조
   │
   ▼
실제 실행되는 명령
   │
   ▼
데이터 크기 × 트래픽
   │
   ▼
운영 비용
```

까지 보고 선택해야 한다.
단순 캐시라면

```text
Key → Value → TTL
```

정도로 충분하기 때문에 RedisTemplate이나 Spring Cache가 자연스럽다.
반대로

```text
count()

findAll()

Secondary Index

조건 조회
```

가 실제로 필요하고 Redis를 하나의 데이터 저장소처럼 사용한다면 Repository의 추상화가 가치가 있을 수 있다.
이번 사례에서 가장 중요한 것은
> **Spring Data Redis Repository가 느리다**
가 아니다.

```text
편리한 추상화를 사용한다
        │
        ▼
그 아래 실제 동작을 모른다
        │
        ▼
트래픽이 증가한다
        │
        ▼
숨겨져 있던 비용이 운영 문제로 나타난다
```

는 흐름이다.
그래서 운영 환경에서 Redis 성능 문제가 발생한다면 단순히

```text
CPU가 높다

→ Redis Scale-up
```

부터 생각하기보다

```text
우리 애플리케이션이

Redis에 무슨 명령을

몇 번 보내고 있는가?
```

부터 확인해야 한다.
**추상화를 쓰더라도, 결국 운영에서 비용을 지불하는 것은 그 아래에서 실제로 실행되는 명령이다.**
