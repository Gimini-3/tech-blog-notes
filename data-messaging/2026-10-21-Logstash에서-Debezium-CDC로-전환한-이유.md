# Logstash에서 Debezium CDC로 전환한 이유

- 읽은 날짜: 2026-10-21
- 회사/작성 주체: 넥스트리
- 원문: [Logstash에서 Debezium CDC로 전환한 이유](https://www.nextree.io/logstasheseo-debezium-cdcro-jeonhwanhan-iyu/)
- 노션 정리: [원본 페이지](https://app.notion.com/p/3e6a987f6c4080f38f84cf672f6f33cb)

---

여행 장소와 관련 콘텐츠를 여러 언어로 제공하는 서비스가 있었다.

사용자가

```txt
제주도

강남 맛집

도쿄 숙소
```

같은 검색어를 입력하면 관련 장소와 콘텐츠를 찾아주는 것이 핵심 기능이었다.

기존에는 PostgreSQL을 직접 조회했다.

```txt
Client
   │
   ▼
Backend
   │
   ▼
PostgreSQL

JOIN
LIKE
ORDER BY
OFFSET
```

하지만 검색 품질과 성능 요구가 커지면서 PostgreSQL만으로 검색을 계속 처리하기 어려워졌다.

---

## 1. 기존 검색 구조

장소 데이터는 정규화되어 있었다.

예를 들어

```txt
place

id
latitude
longitude
visible
```

같은 언어와 무관한 정보가 있고,

```txt
place_lang

place_id
lang_code
place_name
address
description
```

처럼 언어별 정보가 따로 존재했다.

따라서 장소 하나를 검색하려면 최소 두 테이블을 JOIN해야 했다.

```txt
place
  │
  └── place_lang
```

카테고리나 태그까지 붙으면

```txt
place
  │
  ├── place_lang
  │
  ├── category
  │
  └── tag
```

처럼 JOIN이 계속 늘어난다.

---

## 2. LIKE 검색의 한계

기존 검색은 대략 이런 SQL이었다.

```sql
SELECT ...
FROM place p
JOIN place_lang pl
  ON p.id = pl.place_id
WHERE pl.lang_code = :langCode
  AND pl.place_name LIKE '%' || :keyword || '%'
ORDER BY pl.registered_on DESC
LIMIT :limit OFFSET :offset;
```

문제는 `LIKE`가 단순 문자열 포함 여부만 판단한다는 것이다.

예를 들어

```txt
검색어

"제주도 펜션"
```

인데 데이터가

```txt
"제주 펜션"
```

이라면 원하는 수준의 검색 품질을 만들기 어렵다.

```txt
조사

띄어쓰기

복합명사

동의어

오타
```

같은 언어적 특성을 처리하기 힘들다.

---

## 3. 검색 조건이 늘어날수록 DB Query가 무거워진다

검색 결과 하나를 만들기 위해 여러 Table을 JOIN한다.

```txt
Place
  │
  ├─ Place Language
  ├─ Category
  └─ Tag
```

Pagination까지 들어가면 추가 비용이 생긴다.

```txt
검색 결과 Query

+

전체 Count Query
```

또 OFFSET Pagination은 뒤 페이지로 갈수록 읽고 버리는 Row가 늘어난다.

```txt
OFFSET 10

→ 10개 버림


OFFSET 100,000

→ 100,000개 읽고 버림
```

즉 검색 조건을 추가할수록 원본 DB의 부담도 증가했다.

---

## 4. 관련도 정렬도 어렵다

SQL의 `WHERE`는 기본적으로

```txt
조건에 맞는가?

YES / NO
```

를 판단한다.

하지만 검색에서는

```txt
이 결과가 얼마나 검색어와 비슷한가?
```

가 중요하다.

예를 들어

```txt
검색어 = 제주도 여행
```

이라면

```txt
제주도 여행 추천     Score 0.95

제주 맛집            Score 0.72

서울 여행            Score 0.31
```

처럼 관련도 점수를 기반으로 정렬하고 싶다.

기존 PostgreSQL 검색에서는 등록일 기준 정렬 정도만 사용하고 있었다.

그래서 검색 전용 저장소인 Elasticsearch를 도입하기로 했다.

---

## 5. 그런데 Elasticsearch를 추가하면 새로운 문제가 생긴다

기존에는 데이터가 PostgreSQL 하나에만 있었다.

```txt
PostgreSQL
```

이제는

```txt
PostgreSQL
    │
    │ Sync
    ▼
Elasticsearch
```

두 저장소의 상태를 맞춰야 한다.

예를 들어 PostgreSQL의 장소 이름이 변경되면

```txt
PostgreSQL

제주호텔
   ↓
제주 리조트
```

Elasticsearch에도 똑같이 반영되어야 한다.

```txt
Elasticsearch

제주호텔
   ↓
제주 리조트
```

---

## 6. Application에서 둘 다 저장하면 안 될까?

가장 쉬운 방법은 Backend가

```txt
DB 저장

+

Elasticsearch 색인
```

을 같이 수행하는 것이다.

```txt
Request
   │
   ▼
Backend
   │
   ├── PostgreSQL Write
   │
   └── Elasticsearch Write
```

하지만 둘은 하나의 Transaction으로 묶이지 않는다.

```txt
PostgreSQL

COMMIT O


Elasticsearch

INDEX X
```

가 발생하면 데이터가 어긋난다.

```txt
PostgreSQL = 최신

Elasticsearch = 이전 상태
```

반대로도 가능하다.

이 문제 때문에 검색 동기화는 Application 밖에서 처리하기로 했다.

---

## 7. 첫 번째 선택: Logstash JDBC Input

가장 먼저 선택한 것이 Logstash였다.

구조가 단순했다.

```txt
PostgreSQL
    │
    │ JDBC SELECT
    ▼
 Logstash
    │
    ▼
Elasticsearch
```

Logstash JDBC Input을 이용하면 SQL Query를 실행한 결과를 바로 Elasticsearch에 넣을 수 있다.

특히 여러 Table을 JOIN해 하나의 검색 Document를 만들기도 쉽다.

```txt
PostgreSQL

place
place_lang
category

      │
      │ JOIN
      ▼

Logstash

      │
      ▼

Elasticsearch Document
```

별도의 Kafka 같은 인프라도 필요하지 않았다.

---

## 8. 변경 데이터는 어떻게 찾았을까?

`modified_on` 같은 수정 시각 컬럼을 사용했다.

예를 들어 마지막으로

```txt
12:00:00
```

까지 읽었다면 다음 Poll에서

```sql
SELECT ...
FROM content
WHERE modified_on > :sql_last_value
ORDER BY modified_on;
```

처럼 조회한다.

Logstash는 마지막 조회 값을 기억한다.

```txt
12:00

SELECT modified_on > 12:00

      ↓

새로운 데이터 처리

      ↓

Last Value = 12:01
```

그리고 1분마다 반복했다.

---

## 9. INSERT는 잘 됐다

새로운 Row가 생성되면

```txt
12:01

INSERT Place
```

`modified_on`이 최신이기 때문에 다음 Poll에서 조회된다.

```txt
PostgreSQL

INSERT
  │
  ▼
Row 존재
  │
  ▼
Logstash SELECT
  │
  ▼
Elasticsearch INDEX
```

문제가 없다.

---

## 10. UPDATE도 잘 됐다

Row가 수정되면

```txt
modified_on

12:01 → 12:10
```

이 된다.

다음 Poll에서 다시 조회된다.

```txt
UPDATE
  │
  ▼
modified_on 변경
  │
  ▼
Logstash 조회
  │
  ▼
같은 Document ID
  │
  ▼
Elasticsearch 덮어쓰기
```

Document ID를 원본 PK와 동일하게 사용했기 때문에 같은 Row가 여러 번 읽혀도 중복 Document가 생성되지는 않았다.

---

## 11. 그런데 DELETE가 반영되지 않았다

문제는 삭제였다.

PostgreSQL에서

```sql
DELETE FROM place
WHERE id = 100;
```

을 실행했다.

DB에서는 Row가 사라졌다.

```txt
PostgreSQL

id = 100

X
```

그런데 Elasticsearch에는 그대로 남아 있었다.

```txt
Elasticsearch

id = 100

O
```

따라서 검색 결과에는 나타나지만 클릭하면 실제 데이터는 없는 문제가 발생했다.

---

## 12. 왜 Logstash Polling은 DELETE를 모를까?

이게 이 글에서 가장 중요한 부분이다.

Logstash JDBC 방식은 다음 질문을 DB에 던진다.

```txt
"지금 Table에 존재하면서
modified_on이 마지막 시각보다
큰 Row를 주세요."
```

즉 **현재 존재하는 Row만 볼 수 있다.**

```txt
Table

A
B
C
```

C를 삭제한다.

```txt
Table

A
B
```

다음 Poll:

```sql
SELECT *
FROM table
WHERE modified_on > ...
```

결과:

```txt
A
B
```

Logstash 입장에서는

```txt
C가 방금 삭제됐다.
```

라는 정보가 없다.

단지

```txt
현재 C가 없다.
```

만 알 수 있다.

더 정확하게는 그것조차 모른다.

```txt
처음부터 C가 없었는지

방금 C가 삭제된 건지
```

구분할 정보가 없기 때문이다.

---

## 13. Polling 방식의 근본적인 한계

정리하면

```txt
INSERT

Row 존재
→ 조회 가능
```

```txt
UPDATE

Row 존재
→ 조회 가능
```

하지만

```txt
DELETE

Row 자체가 없음
→ 조회 불가능
```

이다.

즉 Polling 방식은 **State를 관찰**한다.

```txt
현재 DB 상태가 무엇인가?
```

반면 필요한 것은

```txt
무슨 변경이 일어났는가?
```

였다.

---

## 14. 해결책 ① Soft Delete?

첫 번째로 생각한 방법은 실제 Row를 삭제하지 않는 것이다.

```txt
Before

DELETE Row


After

UPDATE deleted = true
```

그러면 삭제가 UPDATE가 되기 때문에 Polling으로 잡을 수 있다.

```txt
deleted = false
      ↓
deleted = true
      │
      ▼
modified_on 변경
      │
      ▼
Logstash 감지
```

하지만 검색 때문에 원본 DB Schema와 모든 삭제 로직을 수정해야 했다.

---

## 15. 해결책 ② DB와 Elasticsearch ID 비교?

주기적으로

```txt
PostgreSQL ID

vs

Elasticsearch ID
```

를 비교한다.

```txt
DB

1 2 3


ES

1 2 3 4


→ 4 삭제
```

가능은 하지만 데이터가 증가할수록 비교 비용이 커진다.

또 비교 주기 동안 잘못된 Document가 계속 검색된다.

```txt
30분마다 비교

→ 최대 30분 동안
삭제 데이터 노출 가능
```

이 역시 근본적인 해결책은 아니었다.

---

## 16. 필요한 것은 State가 아니라 Change였다

결국 필요한 정보는

```txt
현재 Row 목록
```

이 아니라

```txt
INSERT가 발생했다.

UPDATE가 발생했다.

DELETE가 발생했다.
```

라는 **변경 기록**이었다.

그래서 Logstash JDBC Polling에서 **Debezium CDC**로 변경했다.

---

## 17. PostgreSQL WAL

PostgreSQL은 데이터 변경 내용을 WAL에 기록한다.

WAL은 **Write-Ahead Log**다.

예를 들어

```sql
INSERT

UPDATE

DELETE
```

가 발생하면 변경 내용이 WAL에 남는다.

```txt
Application
     │
     ▼
PostgreSQL
     │
     ├── Table 변경
     │
     └── WAL 기록
```

Debezium은 Table을 반복 조회하는 대신 이 WAL을 읽는다.

---

## 18. Polling과 CDC의 차이

#### Logstash JDBC

```txt
PostgreSQL Table
       │
       │ SELECT
       ▼
   Logstash

"현재 무엇이 존재하는가?"
```

#### Debezium

```txt
PostgreSQL WAL
       │
       ▼
    Debezium

"무슨 변경이 발생했는가?"
```

이 차이가 DELETE 처리 여부를 결정했다.

---

## 19. DELETE Event

Row가 삭제되면 WAL에는 삭제 기록이 남는다.

Debezium은 이를 다음과 같은 Event로 전달한다.

```json
{
  "op": "d",
  "before": {
    "id": "...",
    "lang_code": "ko"
  },
  "after": null
}
```

즉

```txt
op = d

→ Delete
```

이고

```txt
before

→ 삭제되기 직전 데이터
```

를 알 수 있다.

이제 검색 서비스는

```txt
Delete Event
     │
     ▼
Document ID 확인
     │
     ▼
Elasticsearch DELETE
```

를 수행할 수 있다.

---

## 20. 전체 구조 변화

#### 기존

```txt
PostgreSQL
     │
     │ 1분 Polling
     ▼
 Logstash
     │
     ▼
Elasticsearch
```

#### 변경

```txt
PostgreSQL
     │
     │ WAL
     ▼
 Debezium
     │
     ▼
Kafka Connect
     │
     ▼
   Kafka
     │
     ▼
Search Service
     │
     ▼
Elasticsearch
```

---

## 21. Logstash vs Debezium

핵심 차이만 정리하면

```txt
Logstash JDBC

관찰 대상
→ 현재 Row

변경 감지
→ 주기적 SELECT

삭제
→ 감지 어려움

기준
→ modified_on

초기 적재
→ 별도 Query
```

반면

```txt
Debezium

관찰 대상
→ WAL Change Event

변경 감지
→ Logical Replication

삭제
→ DELETE Event

기준
→ LSN

초기 적재
→ Snapshot
```

이다.

---

## 22. 물론 Debezium이 공짜는 아니다

Logstash 방식은 단순했다.

```txt
PostgreSQL

+

Logstash

+

Elasticsearch
```

Debezium을 쓰면

```txt
PostgreSQL

+

Debezium

+

Kafka Connect

+

Kafka

+

Search Consumer

+

Elasticsearch
```

로 구성 요소가 늘어난다.

또 PostgreSQL Replication Slot도 관리해야 한다.

즉 **운영 복잡도는 증가했다.**

하지만

```txt
단순하지만 DELETE가 틀림
```

보다는

```txt
조금 복잡하지만 정확하게 동기화
```

가 더 중요했다.

---

## 23. PostgreSQL에서 CDC를 사용하려면

Debezium이 WAL을 읽으려면 PostgreSQL Logical Replication을 활성화해야 한다.

원문에서는 다음처럼 구성했다.

```txt
wal_level = logical

max_replication_slots = 10

max_wal_senders = 10
```

```txt
PostgreSQL
     │
     ▼
Logical WAL
     │
     ▼
Replication Slot
     │
     ▼
Debezium
```

---

## 24. Replication Slot이란?

WAL은 계속 생성된다.

```txt
WAL 100

WAL 101

WAL 102

WAL 103

...
```

그런데 Debezium이 아직 WAL 102까지밖에 못 읽었다고 해보자.

PostgreSQL이 WAL 103 이후를 포함해 필요한 WAL까지 너무 빨리 삭제하면 Debezium이 데이터를 놓칠 수 있다.

Replication Slot은

> 이 Consumer가 어디까지 읽었는가?

를 PostgreSQL에 알려준다.

```txt
WAL

100 101 102 103 104 105
        ▲
        │
 Debezium confirmed
```

PostgreSQL은 Consumer에게 필요한 WAL을 유지한다.

---

## 25. LSN

PostgreSQL WAL 위치는 **LSN(Log Sequence Number)** 으로 표현한다.

개념적으로는

```txt
WAL

────────────●──────────────▶
            ▲
            │
           LSN
```

이다.

Debezium은

```txt
어디까지 처리했는가?
```

를 LSN을 기준으로 추적한다.

Logstash가

```txt
modified_on = 12:30
```

같은 Application Column에 의존했다면,

Debezium은

```txt
WAL LSN
```

이라는 DB 내부 변경 위치를 이용한다.

---

## 26. REPLICA IDENTITY가 왜 필요할까?

PostgreSQL에서 UPDATE나 DELETE Event를 만들 때 **이전 Row 데이터가 얼마나 WAL에 기록되는가**를 결정하는 설정이 있다.

기본 설정에서는 보통 PK 중심 정보만 사용할 수 있다.

하지만 Elasticsearch Document를 삭제하거나 복합 데이터를 만들기 위해 삭제 직전 Row 전체가 필요할 수도 있다.

그래서 필요한 Table에는

```txt
REPLICA IDENTITY FULL
```

을 사용했다.

```txt
DELETE

      │
      ▼

WAL

before 전체 Row
```

다만 이전 Row 전체를 WAL에 기록하면 WAL 양이 증가한다.

따라서 모든 Table이 아니라 실제 필요한 Table에만 적용했다.

---

## 27. Debezium Connector 설정

원문에서 핵심 설정은 대략 다음과 같다.

```txt
connector.class
→ PostgreSQL Connector


plugin.name
→ pgoutput


slot.name
→ 사용할 Replication Slot


table.include.list
→ CDC 대상 Table


snapshot.mode
→ initial
```

특히 전체 DB를 구독하지 않았다.

```txt
전체 Table

X


검색에 필요한 Table

place

place_lang

content
```

만 지정했다.

불필요한 WAL Event까지 Kafka로 보내는 것을 막기 위해서다.

---

## 28. pgoutput

PostgreSQL Logical Replication 데이터를 해석하려면 Logical Decoding Plugin이 필요하다.

이 사례에서는 PostgreSQL에 내장된

```txt
pgoutput
```

을 사용했다.

```txt
PostgreSQL WAL
      │
      ▼
  pgoutput
      │
      ▼
Debezium
```

별도 Plugin을 PostgreSQL에 설치할 필요가 없다는 장점이 있었다.

---

## 29. 초기 데이터는 Snapshot으로 적재

기존 PostgreSQL에는 이미 데이터가 있다.

```txt
PostgreSQL

100,000 Rows


Elasticsearch

0 Documents
```

WAL CDC만 시작하면 이후 변경만 들어온다.

그래서 Debezium의

```txt
snapshot.mode = initial
```

을 사용했다.

```txt
Connector Start
      │
      ▼
Snapshot
      │
      ▼
현재 데이터 전체 적재
      │
      ▼
Snapshot 완료
      │
      ▼
WAL Streaming
```

Logstash에서 별도 초기 적재 SQL을 만들던 작업도 줄었다.

---

## 30. 검색 서비스도 분리했다

기존에는 Backend가

```txt
Business API

+

Search API

+

Search Module
```

을 함께 담당했다.

변경 후:

```txt
Business Service

PostgreSQL Write


Search Service

Kafka Consume
Elasticsearch Index
Search API
```

로 분리했다.

```txt
                 PostgreSQL
                      │
                      ▼
                   Debezium
                      │
                      ▼
                    Kafka
                      │
                      ▼
                Search Service
                      │
                      ▼
                Elasticsearch
```

---

## 31. 왜 검색 서비스를 분리했을까?

Elasticsearch Index를 다시 생성한다고 해보자.

```txt
Reindex

1,000,000 Documents
```

CPU와 Network를 많이 사용한다.

기존 Backend와 같이 실행하면

```txt
Reindex 부하

      +

일반 API Traffic

      ↓

Backend 영향
```

이 생길 수 있다.

서비스를 분리하면

```txt
Business Service

영향 최소화


Search Service

Reindex / Mapping 변경
독립 수행
```

이 가능하다.

또 Elasticsearch 관련 설정이 기존 Business Service에서 완전히 제거되었다.

---

## 32. Elasticsearch를 사용하면서 한국어 검색도 개선

Elasticsearch에는 **Nori 형태소 분석기**를 적용했다.

기본적으로 단순 문자열 비교만 하면

```txt
제주도

!=

제주
```

같은 문제가 있다.

Nori는 한국어 형태소를 분석한다.

예를 들어

```txt
제주도
```

를

```txt
제주

도
```

처럼 분석할 수 있다.

그러면 검색 시 훨씬 유연한 Matching이 가능해진다.

---

## 33. 이제 가장 중요한 운영 문제: Replication Slot

Replication Slot은 CDC 신뢰성을 위해 필요하다.

하지만 위험한 점도 있다.

Debezium이 멈췄다고 해보자.

```txt
Debezium

X
```

DB Write는 계속 발생한다.

```txt
PostgreSQL

WAL 100
WAL 101
WAL 102
WAL 103
...
```

Replication Slot은

```txt
Debezium이 아직 읽지 않았다.
```

고 판단한다.

그러면 PostgreSQL은 WAL을 삭제하지 않는다.

```txt
WAL

계속 보관
계속 보관
계속 보관
```

---

## 34. Connector 장애가 DB Disk 장애로 이어질 수 있다

Debezium이 오랫동안 멈추면

```txt
Connector Down
      │
      ▼
LSN 진행 X
      │
      ▼
Replication Slot 유지
      │
      ▼
WAL 삭제 불가
      │
      ▼
WAL 계속 증가
      │
      ▼
DB Disk 사용량 증가
      │
      ▼
Disk Full 위험
```

즉 재미있는 점은

> CDC Consumer 장애가 Source DB 장애로 전파될 수 있다.

는 것이다.

그래서 Replication Slot Lag을 반드시 모니터링해야 한다.

---

## 35. Replication Slot 상태 확인

원문에서는 다음과 같은 Query로 Slot 상태를 확인했다.

```sql
SELECT
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            confirmed_flush_lsn
        )
    ) AS retained_wal
FROM pg_replication_slots;
```

확인할 값은

```txt
slot_name

active

confirmed_flush_lsn

retained_wal
```

등이다.

특히

```txt
retained_wal
```

이 계속 증가한다면 Debezium이 WAL을 따라가지 못하고 있거나 Connector가 내려갔을 가능성을 확인해야 한다.

---

## 36. 전체 동작 흐름

최종 구조를 하나로 합치면 다음과 같다.

```txt
                    User
                     │
                     ▼
              Business Service
                     │
                     ▼
                 PostgreSQL
                     │
              Transaction
                     │
                     ▼
                    WAL
                     │
            Logical Replication
                     │
                     ▼
              Replication Slot
                     │
                     ▼
                  pgoutput
                     │
                     ▼
                  Debezium
                     │
                     ▼
               Kafka Connect
                     │
                     ▼
                   Kafka
                     │
                     ▼
              Search Service
                     │
            ┌────────┴────────┐
            │                 │
       INSERT/UPDATE        DELETE
            │                 │
            ▼                 ▼
        Index/Update      Document Delete
            │                 │
            └────────┬────────┘
                     ▼
                Elasticsearch
                     │
                     ▼
                 Search API
```

---

## 37. INSERT 동작

```txt
PostgreSQL

INSERT
   │
   ▼
WAL
   │
   ▼
Debezium
   │
   ▼
op = c
   │
   ▼
Kafka
   │
   ▼
Search Service
   │
   ▼
Elasticsearch INDEX
```

---

## 38. UPDATE 동작

```txt
PostgreSQL

UPDATE
   │
   ▼
WAL
   │
   ▼
Debezium
   │
   ▼
op = u
   │
   ▼
Kafka
   │
   ▼
Search Service
   │
   ▼
Elasticsearch UPDATE
```

---

## 39. DELETE 동작

이 구조를 도입한 가장 중요한 이유다.

```txt
PostgreSQL

DELETE
   │
   ▼
WAL
   │
   ▼
Debezium
   │
   ▼
op = d
before = 삭제 데이터
   │
   ▼
Kafka
   │
   ▼
Search Service
   │
   ▼
Document ID 확인
   │
   ▼
Elasticsearch DELETE
```

이제 삭제 정보가 사라지지 않는다.

---

## 40. Logstash와 Debezium의 가장 큰 차이

단순하게 표현하면 이렇다.

```txt
Logstash JDBC

DB에게 질문한다.

"지금 뭐가 있어?"
```

반면

```txt
Debezium CDC

DB의 변경 기록을 읽는다.

"무슨 일이 있었어?"
```

그래서

```txt
INSERT / UPDATE

둘 다 가능
```

하지만 DELETE에서는 차이가 결정적으로 드러난다.

```txt
Polling

DELETE 이후 Row 없음

→ 사건 자체가 사라짐


CDC

WAL에 DELETE 기록 존재

→ 사건을 알 수 있음
```

---

## 41. 검색 시스템 구조 자체도 달라졌다

기존:

```txt
                    Backend
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  Business API     Search SQL      PostgreSQL
```

변경:

```txt
Business Service
      │
      ▼
PostgreSQL
      │
      ▼
WAL
      │
      ▼
Debezium
      │
      ▼
Kafka
      │
      ▼
Search Service
      │
      ▼
Elasticsearch
```

검색과 OLTP DB의 책임을 분리했다.

---

## 42. 최종적으로 얻은 것

#### 검색 품질

```txt
LIKE

        ↓

Elasticsearch + Nori
```

#### 검색 성능

```txt
복잡한 JOIN

        ↓

검색 전용 Index
```

#### 관련도

```txt
등록일 정렬

        ↓

Score 기반 정렬
```

#### 데이터 동기화

```txt
1분 Polling

        ↓

WAL CDC
```

#### DELETE

```txt
감지 불가능

        ↓

Delete Event
```

#### 초기 적재

```txt
별도 SQL

        ↓

Debezium Snapshot
```

---

## 43. 이 사례에서 가장 중요한 교훈

처음에는 Logstash가 상당히 좋은 선택처럼 보였다.

```txt
설치 간단

Kafka 불필요

SQL 작성 쉬움

INSERT O

UPDATE O
```

실제로 등록과 수정만 테스트하면 아무 문제가 없어 보인다.

하지만 DELETE를 테스트하는 순간 기술의 근본적인 한계가 드러났다.

```txt
Polling

현재 상태를 관찰한다.


CDC

변경 사건을 관찰한다.
```

이 둘은 비슷해 보여도 완전히 다른 모델이다.

---

## 정리

전체 기술 선택 과정은 다음과 같다.

```txt
PostgreSQL 직접 검색
        │
        ▼
검색 품질 / JOIN / 관련도 문제
        │
        ▼
Elasticsearch 도입
        │
        ▼
DB ↔ ES 동기화 필요
        │
        ▼
Logstash JDBC Polling
        │
        ├─ INSERT O
        ├─ UPDATE O
        └─ DELETE X
              │
              ▼
        근본 원인 분석
              │
              ▼
"Polling은 현재 존재하는
 Row만 볼 수 있다."
              │
              ▼
Debezium CDC
              │
              ▼
PostgreSQL WAL
              │
              ▼
Logical Replication
              │
              ▼
Replication Slot
              │
              ▼
Kafka
              │
              ▼
Search Service
              │
              ▼
Elasticsearch
```

결국 이번 전환의 핵심은

**Logstash보다 Debezium이 더 좋은 도구여서 바꾼 것이 아니다.**

필요한 문제 자체가

```txt
주기적으로 현재 데이터를 조회한다
```

가 아니라

```txt
DB에서 발생한 모든 변경 사건을
빠짐없이 전달한다
```

였기 때문에 CDC가 더 맞는 구조였던 것이다.

그리고 Debezium을 운영할 때는 `WAL → Replication Slot → LSN → Kafka` 흐름만 보는 것이 아니라,

```txt
Connector가 멈추면

Replication Slot이 WAL을 잡고

WAL이 쌓이면

Source DB Disk까지 위험해질 수 있다.
```

는 운영 특성까지 함께 이해해야 한다.

**기술을 고를 때는 "이 도구가 무엇을 잘하나?"보다 "이 방식으로는 원천적으로 볼 수 없는 것이 무엇인가?"를 먼저 보는 게 중요하다.**
