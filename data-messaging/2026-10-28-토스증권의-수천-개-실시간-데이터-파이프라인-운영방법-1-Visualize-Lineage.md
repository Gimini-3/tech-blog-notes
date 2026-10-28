# 토스증권의 수천 개 실시간 데이터 파이프라인 운영방법 #1: Visualize Lineage

- 읽은 날짜: 2026-10-28
- 회사/작성 주체: 토스증권
- 원문: [토스증권의 수천 개 실시간 데이터 파이프라인 운영방법 #1: Visualize Lineage](https://toss.tech/article/toss-securities-visualize-lineage)
- 노션 정리: [원본 페이지](https://app.notion.com/p/407a987f6c4080c591f1e4aa759bb5c2)

---

토스증권은 데이터 분석뿐 아니라 서비스 영역에서도 실시간 데이터 파이프라인을 적극적으로 사용한다.

대표적으로

```txt
데이터 영역

OLTP
  │
  ▼
실시간 Pipeline
  │
  ▼
OLAP / Data Lake
```

- **OLTP (Online Transaction Processing)**: 주문, 결제, 계좌이체처럼 **실시간으로 데이터를 조회·추가·수정·삭제하는 시스템**.
- **OLAP (Online Analytical Processing)**: 매출 분석, 통계, 보고서처럼 **대량의 데이터를 분석하여 의사결정을 지원하는 시스템**.

✅ **핵심 차이:** OLTP는 **실시간 거래 처리**, OLAP는 **데이터 분석**.

뿐 아니라 CQRS를 적용하는 서비스에서도

```txt
Command DB
   │
   │ 실시간 변경
   ▼
Data Pipeline
   │
   ▼
Query용 저장소
```

형태로 활용한다.

서비스가 커지면서 이런 파이프라인은 어느새 **수천 개 규모**가 됐다.

문제는 단순히 파이프라인을 만드는 것이 아니라,

> 수천 개의 실시간 파이프라인을 어떻게 안정적으로 이해하고 운영할 것인가?

였다.

---

## 1. 실시간 데이터 파이프라인이란?

일반적인 데이터 파이프라인은 데이터를 단계별로 가공한다.

```txt
Table A
   │
   │ 가공
   ▼
Table B
   │
   │ 집계
   ▼
Table C
```

예를 들어

```txt
주문 원본

       ↓

주문 + 구분값

       ↓

구분별 주문 수 집계
```

처럼 중간 산출물을 만들면서 데이터를 활용하기 좋은 형태로 바꾼다.

**실시간 데이터 파이프라인**은 이 과정이 Batch가 아니라 데이터가 발생하는 즉시 수행되는 구조다.

```txt
Event 발생
    │
    ▼
즉시 Processing
    │
    ▼
즉시 Target 반영
```

---

## 2. 서비스가 커지면 왜 Batch만으로 부족해질까?

초기에는 이런 방식으로도 충분할 수 있다.

```txt
OLTP DB
   │
   │ 하루 한 번 Dump
   ▼
OLAP DB
```

하지만 데이터가 계속 커지면 문제가 생긴다.

```txt
데이터 증가
   │
   ├─ Dump 시간 증가
   ├─ DB 부하 증가
   └─ 데이터 최신성 저하
```

결국

```txt
하루치 Batch

      ↓

변경되는 순간 바로 전달
```

하는 구조가 필요해진다.

---

## 3. 서비스 영역에서도 실시간 파이프라인이 필요하다

OLTP DB의 목적은 일반적으로 빠른 트랜잭션 처리다.

그런데 서비스가 커지면 OLTP DB에 이런 요구까지 들어온다.

```txt
대규모 JOIN

집계

전체 Table Scan

Ranking

복잡한 조회
```

모든 작업을 원본 DB에서 처리하면 부담이 커진다.

그래서 CQRS 형태로 역할을 분리할 수 있다.

```txt
                ┌─────────────┐
Write Request → │   OLTP DB   │
                └──────┬──────┘
                       │
                 Change Event
                       │
                       ▼
               Real-time Pipeline
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        ClickHouse            Data Lake
             │
             ▼
        Query / Serving
```

즉 실시간 데이터 파이프라인은 **OLTP와 조회·분석용 저장소 사이를 연결하는 역할**도 한다.

---

## 4. 실시간 파이프라인을 "잘 운영한다"는 것은?

실시간이라는 단어가 붙으면 운영 요구사항이 커진다.

토스증권에서는 크게 세 가지를 중요하게 본다.

```txt
1. SLA

2. 확장성

3. 시각화
```

#### SLA (Service Level Agreement)

서비스 제공자가 고객에게 보장하기로 약속한 서비스 품질 기준

```txt
Latency 최소화

365일 무중단

데이터 유실 최소화

중복 최소화

모든 Pipeline 모니터링
```

#### 확장성

```txt
Pipeline 100개
      ↓
1,000개
      ↓
수천 개

에서도 운영 가능
```

또 중요한 주문/체결 파이프라인이 다른 파이프라인 추가 때문에 영향을 받으면 안 된다.

#### 시각화

```txt
이 Job 아직 쓰나?

누가 만든 거지?

어떤 Topic을 읽지?

결과는 어디로 가지?

이 Table은 어디서 생성됐지?
```

수천 개가 되면 이 질문을 사람이 문서만 보고 찾는 것이 어려워진다.

---

## 5. 그래서 Data Lineage가 필요했다

예를 들어 다음 파이프라인이 있다고 하자.

```txt
MySQL A Table
      │
      │ CDC
      ▼
 Kafka Topic A
      │
      ▼
    Flink
      │
      ▼
 Kafka Topic B
      │
      ├─────────────┐
      ▼             ▼
   Iceberg       Ranking Service
```

이를 단순하게 표현하면

```txt
A → B → C
     └→ D
```

이다.

여기서

```txt
B는 A에서 만들어졌고

C와 D는 B에서 만들어졌다.
```

라는 **데이터의 출처와 파생 관계**를 Data Lineage라고 본다.

Apache Flink는 **실시간으로 들어오는 데이터를 처리하고 분석하는 분산 스트림 처리 엔진**이다.<br>즉, Kafka에서 데이터를 받아 **실시간으로 가공·집계한 뒤 다른 시스템에 전달하는 기술**이다.

---

## 6. Lineage에는 Graph가 잘 맞는다

데이터 파이프라인에는 순서와 방향이 있다.

```txt
A → B → C
    └→ D
```

그래서 단순한 Table보다 Graph 자료구조가 자연스럽다.

특히 토스증권에서는 **DAG(Directed Acyclic Graph)** 형태로 표현했다.

```txt
Directed

A → B

방향이 존재


Acyclic

A → B → C

C → A

같은 순환은 없음
```

개념적으로는

```txt
Node
= 시스템 / Topic / Job / Table


Edge
= 데이터가 이동하거나 파생되는 관계
```

로 모델링한다.

---

## 7. Lineage Metadata 구조

토스증권에서는 다음과 같은 형태로 메타데이터를 표현했다.

```txt
Node From
  ├─ type
  ├─ cluster
  ├─ name
  └─ properties


Edge
  ├─ type
  ├─ cluster
  ├─ name
  └─ properties


Node To
  ├─ type
  ├─ cluster
  ├─ name
  └─ properties
```

즉 가장 기본 단위는

```txt
[Node From]

      │
      │ Edge
      ▼

[Node To]
```

다.

---

## 8. 왜 MongoDB를 사용했을까?

모든 Lineage Metadata를 MongoDB에 저장했다.

그리고 MongoDB의 **Graph Search**를 이용해 연결 관계를 탐색한다.

개념적으로

```txt
MongoDB

A → B
B → C
B → D
C → E
```

가 저장돼 있다면

```txt
"A에서 파생된 것은?"
```

이라는 탐색으로

```txt
A
└─ B
   ├─ C
   │  └─ E
   └─ D
```

를 얻는 식이다.

이 결과를 Web에서 Graph 형태로 시각화한다.

```txt
Metadata
   │
   ▼
MongoDB
   │
   │ Graph Search
   ▼
Lineage API
   │
   ▼
Web UI
   │
   ▼
DAG Visualization
```

---

## 9. 실제로 어떤 시스템을 Node로 관리할까?

토스증권의 실시간 데이터 환경에는 다양한 시스템이 존재한다.

### 원천 데이터

```txt
Service Server

Kafka Producer

Filebeat
Fluent Bit
Vector

MySQL CDC
Oracle CDC
MongoDB CDC
Vitess CDC
```

### Processing

```txt
Flink

Kafka Connect

ClickHouse MView

Kafka Consumer

ksqlDB
```

### Storage

```txt
Kafka

ClickHouse

Hadoop
 ├─ Parquet
 └─ Iceberg

Kudu

Elasticsearch

Vitess
```

이 모든 구성 요소에 대해 Metadata를 생성하고 관계를 연결한다.

---

## 10. 전체 Lineage 시스템 아키텍처

단순화하면 다음과 같다.

```txt
                     Metadata Sources

         ┌──────────────┼───────────────┐
         ▼              ▼               ▼

       MySQL          Kafka           Flink
       CDC            Topic            Job

         │              │               │
         └──────────────┼───────────────┘
                        │
                        ▼
                Lineage Metadata
                        │
                        ▼
                    MongoDB
                        │
                        │ Graph Search
                        ▼
                  Lineage Service
                        │
                        ▼
                    Web UI
                        │
                        ▼

               A ──▶ B ──▶ C
                     │
                     └────▶ D
```

---

## 11. 실제 사례 1 — 시세 데이터 Fan-out

서비스 서버에서 시세 이벤트를 발행한다.

```txt
Spring Service
      │
      ▼
Service Kafka
      │
      │ Mirror
      ▼
정보계 Kafka
      │
 ┌────┼──────────┐
 ▼    ▼          ▼
Hadoop ClickHouse Kudu
```

하나의 시세 Event가 여러 저장소로 Fan-out되는 구조다.

각 저장소는 서로 다른 목적에 사용된다.

```txt
Hadoop
→ 분석


ClickHouse
→ 빠른 집계 / Serving


Kudu
→ 다른 데이터 활용
```

Lineage를 보면 데이터가 어디에서 시작해서 어디까지 퍼지는지 한 번에 알 수 있다.

---

## 12. 실제 사례 2 — MySQL CDC → ClickHouse

커뮤니티 서비스에서는 두 MySQL Table이 원천이다.

```txt
MySQL Table A
      │
      │ CDC
      ▼
Kafka Topic A
      │
      ▼
ClickHouse Table A


MySQL Table B
      │
      │ CDC
      ▼
Kafka Topic B
      │
      ▼
ClickHouse Table B
```

그다음 ClickHouse Materialized View가 실행된다.

```txt
ClickHouse A ─┐
              ├──▶ MView
ClickHouse B ─┘
                   │
                   │ Join / Processing
                   ▼
              Final Table
                   │
                   ▼
             Community Service
```

즉 전체 Lineage는

```txt
MySQL A → CDC → Kafka → ClickHouse A ─┐
                                      ├→ MView → Final Table
MySQL B → CDC → Kafka → ClickHouse B ─┘
```

처럼 표현된다.

---

## 13. ClickHouse MView는 어떤 역할인가?

원문 사례에서는 데이터가 ClickHouse Table에 들어오는 순간 MView가 트리거되고, Join 및 Processing을 수행해 최종 Table을 만든다.

```txt
INSERT
  │
  ▼
Source Table
  │
  ▼
Materialized View
  │
  ├─ Join
  ├─ Processing
  └─ Transform
  │
  ▼
Final Table
```

서비스에서는 무거운 집계를 매 요청마다 수행하지 않고 이 결과 Table을 조회할 수 있다.

```txt
Service Request

      ↓

Final ClickHouse Table

      ↓

이미 계산된 결과
```

---

## 14. 실제 사례 3 — Kafka 데이터센터 이중화

서비스 서버가 Kafka로 Event를 발행한다.

그리고 두 데이터센터 사이에서 Kafka가 양방향 Mirror된다.

```txt
                 Service
                    │
                    ▼
               Kafka DC A
                    │
             Mirror ⇄ Mirror
                    │
               Kafka DC B

        ორივე DC에 Event 존재
```

그 결과 서비스 Consumer는 한 데이터센터의 Kafka에서 메시지를 소비할 수 있다.

Lineage에서는

```txt
Producer
   │
   ▼
Kafka A
   │ ⇄ Mirror
   ▼
Kafka B
   │
   ▼
Consumer
```

같은 관계까지 확인할 수 있다.

---

## 15. Graph만 보여주는 것이 아니라 상세 정보도 제공한다

단순히

```txt
A → B → C
```

만 보여주면 운영자가 결국 다시 코드나 설정을 찾아봐야 한다.

그래서 Node와 Edge에는 `properties` 정보를 저장했다.

예를 들어 ClickHouse MView를 클릭하면

```txt
ClickHouse MView

├─ 어떤 Source Table?
├─ 어떤 Target Table?
├─ 어떤 Processing Logic?
└─ 어떤 설정?
```

같은 정보를 확인할 수 있다.

즉 Lineage Service는

```txt
"어디에서 어디로 가는가?"

        +

"어떻게 만들어졌는가?"
```

까지 제공한다.

---

## 16. Lineage Metadata는 어떻게 최신 상태를 유지할까?

수천 개의 Pipeline에서는 하루에도 새로운 Pipeline이 생성되고 변경되거나 삭제된다.

따라서 사람이 직접 Metadata를 수정하는 방식으로는 운영하기 어렵다.

토스증권은 처음부터

```txt
CREATE

UPDATE

DELETE
```

를 반영할 수 있는 Interface를 만들었다.

전체 흐름은 다음과 같다.

```txt
Pipeline 변화
     │
     ▼
Metadata Event
     │
     ▼
Kafka Topic
     │
     ▼
Flink
     │
     │ Lineage Metadata 형태로 가공
     ▼
MongoDB
     │
     ├─ Create
     ├─ Update
     └─ Delete
```

즉 **Lineage 자체도 Event Driven Pipeline으로 관리한다.**

---

## 17. Debezium Connector도 자동으로 Lineage에 반영한다

CDC는 Kafka Connect + Debezium을 사용한다.

예를 들어 Connector가 생성됐다고 하자.

```txt
Kafka Connect

Connector Create
      │
      ▼
Metadata Event
      │
      ▼
Kafka
      │
      ▼
Flink
      │
      ▼
MongoDB Lineage
```

Connector가 수정되거나 삭제돼도 동일한 흐름으로 관계를 갱신한다.

따라서

```txt
MySQL
 ↓
Debezium
 ↓
Kafka Topic
```

이라는 새로운 Pipeline이 만들어지면 Lineage에 자동으로 반영될 수 있다.

---

## 18. 서비스 서버와 Kafka 연결은 조금 다르게 찾았다

Kubernetes에서 동작하는 Service Server는 일반 Data Pipeline과 조금 다른 방식으로 처리했다.

서비스 서버와 Kafka Topic 사이 관계를 파악하기 위해 **Kafka Broker Request Log**를 활용했다.

개념적으로는

```txt
Service A
   │
   │ Produce
   ▼
Kafka Topic X
```

가 발생하면 Broker Log를 통해

```txt
누가 Topic X에 접속했는가?
```

를 확인해서 관계를 만들어낸다.

이렇게 하면 MSA 환경에서

```txt
어떤 서비스가

어떤 Kafka Topic을

Produce / Consume하는가?
```

까지 시스템 가시성에 포함할 수 있다.

---

## 19. Lineage를 만들면 운영이 어떻게 달라질까?

기존에는 문제가 생기면 사람이 찾아야 했다.

```txt
이 Table 어디서 만들어졌지?

        ↓

문서 검색

        ↓

Slack 검색

        ↓

Repository 검색

        ↓

담당자 질문
```

Lineage가 있으면

```txt
Table 검색
    │
    ▼
Graph
    │
    ├─ Source
    ├─ Processing
    ├─ Target
    └─ Consumer
```

을 바로 확인할 수 있다.

즉 **운영 탐색 비용과 팀 간 커뮤니케이션 비용**을 줄인다.

---

## 20. 앞으로는 Graph에 Metric도 넣는다

현재는

```txt
A → B → C
```

라는 구조를 보여준다.

하지만 다음 목표는 Edge 자체에 Metric을 표현하는 것이다.

```txt
A
 │
 │ Latency = 120ms
 │ Throughput = 50K/s
 ▼
B
 │
 │ Latency = 30ms
 │ Throughput = 48K/s
 ▼
C
```

토스증권은 시스템 Metric을 ClickHouse에 통합 저장하고 있기 때문에 Lineage와 연동할 계획이다.

그러면 Graph 하나로

```txt
어디가 연결돼 있는가?

+

어디가 느린가?

+

처리량은 얼마인가?
```

를 볼 수 있다.

---

## 21. SLA도 Lineage에서 관리하려 한다

각 Pipeline마다 목표 SLA가 있다.

앞으로는 이런 정보까지 Metadata로 관리하려 한다.

```txt
Pipeline A

Latency SLA
< 1 sec

Availability
99.99%

Mode
Realtime

Write Type
Append

Owner
Realtime Data Team
```

그리고 실제 Metric과 비교한다.

```txt
목표 Latency

1 sec

      vs

현재 Latency

1.4 sec

      ↓

SLA 위반
```

즉 단순 Data Lineage를 넘어 **Pipeline 운영 Metadata Platform**으로 확장하려는 것이다.

---

## 22. 담당자와 영향 범위까지 보여주기

장애가 발생했을 때 가장 먼저 드는 질문은 이것이다.

```txt
누가 담당하지?

누구한테 영향 가지?
```

Graph가 있다면 Downstream을 탐색할 수 있다.

```txt
Pipeline A 장애
     │
     ▼
Topic B
     │
 ┌───┴─────┐
 ▼         ▼
Table C   Service D
             │
             ▼
          User API
```

따라서

```txt
A 장애

→ C 영향
→ D 영향
→ 사용자 API 영향
```

이라는 영향 범위(Blast Radius)를 탐색할 수 있는 기반이 된다.

---

## 23. DBT와 연결하면 End-to-End Lineage가 된다

현재 Lineage의 중심은 실시간 영역이다.

```txt
Source

  ↓

Realtime Pipeline

  ↓

Data Lake
```

하지만 Data Lake 이후에는 Batch/Data Warehouse 영역이 존재한다.

```txt
Data Lake
   │
   ▼
  DBT
   │
   ▼
Data Warehouse
   │
   ▼
Mart / Analytics
```

토스증권에서는 Batch Pipeline이 DBT로 일원화되어 있기 때문에 둘을 연결하면

```txt
MySQL
  │
  ▼
Debezium
  │
  ▼
Kafka
  │
  ▼
Flink
  │
  ▼
Iceberg
  │
  ▼
DBT
  │
  ▼
Warehouse
  │
  ▼
Mart
```

까지 하나의 **End-to-End Data Lineage**로 표현할 수 있다.

---

## 24. 최종 아키텍처

전체를 한 장으로 합치면 다음과 같다.

```txt
                    [실제 데이터 영역]

 MySQL ─CDC─┐
 Oracle ─CDC┤
 Service ───┼──▶ Kafka ───▶ Flink ───▶ Kafka
 File ──────┘                  │           │
                               │           ├──▶ ClickHouse
                               │           ├──▶ Iceberg
                               │           ├──▶ Kudu
                               │           └──▶ Elasticsearch
                               │
                               ▼
                           Processing


-------------------------------------------------


                   [Lineage Metadata 영역]

Pipeline 생성 / 변경 / 삭제
          │
          ▼
   Metadata Event
          │
          ▼
        Kafka
          │
          ▼
        Flink
          │
          ▼
       MongoDB
          │
          │ Graph Search
          ▼
    Lineage Service
          │
          ▼
       Web UI


          A
          │
          ▼
          B
        ┌─┴─┐
        ▼   ▼
        C   D


-------------------------------------------------


                     [향후 확장]

Lineage
  │
  ├─ Latency
  ├─ Throughput
  ├─ SLA
  ├─ Owner
  ├─ Impact
  ├─ Processing Logic
  └─ DBT
       │
       ▼
End-to-End Lineage
```

---

## 정리

토스증권이 해결하려던 문제는 단순히

```txt
수천 개 Pipeline을
어떻게 실행할까?
```

가 아니다.

더 중요한 문제는

```txt
이 Pipeline은 어디에서 왔나?

어디로 가나?

중간에 무슨 Job이 있나?

누가 사용하나?

장애가 나면 어디까지 영향을 주나?

어떻게 만들어졌나?
```

를 빠르게 확인하는 것이었다.

그래서

```txt
Pipeline 관계

      ↓

DAG로 모델링

      ↓

Node / Edge Metadata

      ↓

MongoDB 저장

      ↓

Graph Search

      ↓

Web Visualization
```

구조를 만들었다.

그리고 Metadata 자체도

```txt
Pipeline 변경
   ↓
Kafka Event
   ↓
Flink
   ↓
MongoDB 자동 반영
```

하도록 만들어 수천 개 Pipeline의 변화까지 따라갈 수 있게 했다.

결국 이 사례의 핵심은 **Data Lineage를 단순 문서가 아니라 실제 시스템의 상태를 자동으로 반영하는 Graph 기반 운영 플랫폼으로 만든 것**이다.

향후에는 여기에

```txt
Metric
+
SLA
+
Owner
+
Impact
+
Processing Logic
+
DBT
```

를 붙여서 단순한 Pipeline 그림을 넘어 **토스증권 전체 데이터 시스템의 End-to-End 가시성을 제공하는 플랫폼**으로 확장하려는 방향이다.
