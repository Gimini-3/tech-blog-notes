# Redis on Kubernetes 플랫폼을 구성해 나가기

- 읽은 날짜: 2026-08-07
- 회사/작성 주체: 카카오페이증권
- 원문: [Redis on Kubernetes 플랫폼을 구성해 나가기](https://tech.kakaopay.com/post/kakaopaysec-redis-on-kubernetes/)
- 노션 정리: [원본 페이지](https://app.notion.com/p/3e6a987f6c408053956af687cde0174b)

---

## Redis on Kubernetes 플랫폼 구성기

카카오페이증권은 Redis를 Kubernetes 환경에서 직접 운영하기로 했다.
Public Cloud만 사용한다면 Managed Redis도 좋은 선택이지만, 카카오페이증권은 AWS와 IDC를 함께 사용하는 하이브리드 환경이었다.

```text
AWS

+

IDC

      ↓

Kubernetes 기반
통일된 Redis 플랫폼
```

목표는 단순히 Redis를 컨테이너로 옮기는 것이 아니었다.

```text
빠른 Scale Out

리소스 효율화

Self Healing

선언형 관리

Redis as a Service
```

를 함께 얻는 것이었다.

---

## 1. Redis는 Stateful하다

Stateless Application은 Pod가 사라져도 새 Pod를 실행하면 비교적 쉽게 복구할 수 있다.

```text
Web Pod
   X

   ↓

New Pod
```

하지만 Redis는 데이터를 가진다.

```text
Redis

Memory
 ├─ Key A
 ├─ Key B
 └─ Key C
```

따라서 Kubernetes에 Redis를 올리려면

```text
Network

Replication

Failover

Persistence

Backup

Migration
```

까지 같이 설계해야 한다.

---

## 2. 제공할 Redis 유형

카카오페이증권은 크게 두 가지 형태를 제공했다.

```text
Redis Platform

     ├─ Sentinel
     └─ Cluster
```

#### Sentinel

```text
        Master
          │
      Replication
      ┌───┴───┐
      ▼       ▼
  Replica   Replica
```

비교적 작은 데이터와 가벼운 Cache에 사용한다.

#### Redis Cluster

```text
Master 1       Master 2       Master 3
   │              │              │
Replica 1      Replica 2      Replica 3
```

Sharding과 더 큰 데이터 규모가 필요한 경우 사용한다.

---

## 3. 첫 문제는 네트워크였다

Redis Cluster에서는 여러 Redis 노드가 서로 상태 정보를 주고받는다.

```text
Redis A ───── Redis B
   │ ╲         ╱ │
   │   ╲     ╱   │
   │     ╲ ╱     │
   │     ╱ ╲     │
   ▼   ╱     ╲   ▼
Redis C ───── Redis D
```

Fully Connected Mesh에 가까운 구조이기 때문에 노드 간 통신이 중요하다.

---

## 4. HostNetwork

일반 Kubernetes Pod는 별도의 Pod IP를 가진다.

```text
Node

10.0.0.10

    │

Pod

172.16.1.3
```

카카오페이증권은 Redis에서 네트워크 계층을 단순화하기 위해 `HostNetwork`를 사용했다.

```text
Node Network
      │
      ▼
Redis Pod
```

Pod가 Host Network를 사용하면서 Redis Node 간 통신 구조를 단순화하고 네트워크 오버헤드를 낮추는 방향을 선택했다.

---

## 5. Redis Cluster Redirect

Redis Cluster에서는 데이터를 여러 Master에 Sharding한다.

```text
Node A
Slot 0 ~ 5000

Node B
Slot 5001 ~ 10000

Node C
Slot 10001 ~ 16383
```

Client가 Node A에게 Key를 요청했는데 실제 데이터가 Node C에 있다고 해보자.

```text
Client
   │
   ▼
Node A
   │
   │ "Node C로 가세요"
   ▼
Client
   │
   ▼
Node C
```

Client가 Redirect를 받고 실제 Node에 직접 연결한다.
따라서 Client가 Redis Node 주소에 직접 접근할 수 있어야 한다.

---

## 6. Kubernetes 외부 Client 문제

Redis가 내부 Pod IP를 반환하면 Kubernetes 외부에서는 접근이 어려울 수 있다.

```text
Redis

Redirect

172.16.10.20:6379
```

```text
Kubernetes 내부
→ 접근 가능


외부 IDC / 다른 Cluster
→ 접근 어려움
```

HostNetwork를 적용해 이런 네트워크 구성을 단순화했다.

---

## 7. Sentinel Endpoint는 HAProxy

Sentinel 형태에서는 외부 Client가 Master 변경을 직접 관리하지 않도록 앞단에 HAProxy를 두었다.

```text
Application
     │
     ▼
  HAProxy
     │
     ▼
Redis Master
     │
     ▼
 Replica
```

Client는 고정된 Endpoint만 사용한다.

```text
redis.domain:6379
```

HAProxy가 현재 Master를 찾는다.

---

## 8. HAProxy의 Health Check

단순 Port Check만 하는 것이 아니다.
원문의 설정 흐름을 보면

```text
AUTH
  │
  ▼
PING
  │
  ▼
PONG
  │
  ▼
INFO REPLICATION
  │
  ▼
role:master 확인
```

방식이다.
즉 Redis 자체에 Query를 보내

```text
Redis A

role:master


Redis B

role:slave
```

를 구분한다.
현재 Master인 Redis에 요청을 전달한다.

---

## 9. Master Failover

정상:

```text
Application
    │
    ▼
 HAProxy
    │
    ▼
 Master A
    │
    ▼
Replica B
```

Master A가 장애 나고 B가 새로운 Master로 승격됐다면

```text
Master A
   X

Replica B
   │
   ▼
Master B
```

HAProxy의 다음 Health Check에서는 B가 Master로 확인된다.

```text
Application
    │
    ▼
 HAProxy
    │
    ▼
 Master B
```

Client는 Endpoint를 바꿀 필요가 없다.

---

## 10. Redis Cluster에는 Headless Service

Redis Cluster에서는 Client가 Redis Topology를 이해하고 각 노드에 직접 연결해야 한다.
그래서 일반 Service보다 **Headless Service**가 적합했다.

```yaml
clusterIP: None
```

일반 Service:

```text
Client
   │
   ▼
Service Virtual IP
   │
   ▼
Load Balancing
   │
   ▼
Pod
```

Headless Service:

```text
Client
   │
   ▼
DNS Lookup
   │
   ▼
Pod A
Pod B
Pod C
```

DNS 조회 시 여러 Redis Pod IP가 반환된다.

---

## 11. Redis Cluster 연결 흐름

```text
Client
   │
   ▼
Headless Service
   │
   ▼
Redis Pod 목록 확인
   │
   ▼
Redis A 연결
   │
   │ GET key
   ▼
Redis A
   │
   │ Redirect
   ▼
Redis C
   │
   ▼
Data
```

Kubernetes Service가 중간에서 모든 Redis 요청을 Proxy하는 것이 아니라 Client가 Redis Node로 직접 연결한다.

---

## 12. Node가 추가되면?

Redis Cluster를 확장한다.

```text
기존

A B C


확장

A B C D
```

Headless Service의 DNS/Endpoint 정보도 갱신된다.

```text
DNS

A
B
C
D
```

Client는 새 Cluster Node를 확인할 수 있다.

---

## 13. Sentinel과 Cluster의 Endpoint 차이

정리하면

```text
Sentinel

Client
  │
  ▼
HAProxy
  │
  ▼
Current Master
```

반면

```text
Redis Cluster

Client
  │
  ▼
Headless Service
  │
  ▼
Redis Nodes
```

다.
둘이 해결하려는 문제가 다르다.

```text
Sentinel

"현재 Master가 어디인가?"


Cluster

"이 Key가 어느 Shard에 있는가?"
```

---

## 14. Kubernetes 업그레이드는 어떻게 할까?

Stateful Redis에서 Kubernetes Node를 한 번에 교체하면 위험하다.
그래서 신규 NodeGroup으로 **Replication을 유지하면서 이동**한다.
전체 순서는 다음과 같다.

```text
① 신규 NodeGroup 생성

② 신규 Redis Pod 배포

③ 기존 Redis의 Replica로 Join

④ 데이터 Replication

⑤ 신규 Replica를 Master로 승격

⑥ 기존 Redis 제거

⑦ 정보 갱신
```

---

## 15. Migration 내부 흐름

기존:

```text
Old NodeGroup

Master A
Replica A
```

신규 NodeGroup 생성:

```text
Old NodeGroup            New NodeGroup

Master A                  Redis B
Replica A
```

Redis B를 Replica로 붙인다.

```text
Master A
   │
   │ Replication
   ▼
Redis B
```

데이터를 모두 따라잡는다.
그다음

```text
Redis B

Replica
   │
   ▼
Master
```

로 승격한다.

```text
Old Master
     │
     X 제거

New Master
```

즉 **데이터를 먼저 복제한 뒤 역할을 넘기고 기존 노드를 제거하는 방식**이다.

---

## 16. Zone 분산

Redis Master와 Replica가 같은 Zone에 있으면 Zone 장애 시 함께 죽을 수 있다.

```text
Zone A

Master
Replica

     X Zone 장애
```

그래서 Pod Anti-Affinity와 Zone Spread를 적용했다.

```text
Zone A        Zone B        Zone C

Redis A       Redis B       Redis C
```

가능하면 Host까지 분리한다.

```text
Redis Replica가
같은 장애 Domain에 몰리지 않도록
배치
```

---

## 17. Persistence는 AOF

Redis는 Memory 기반이지만 장애 상황에서 데이터를 복구하기 위해 Persistence가 필요하다.
카카오페이증권에서는 AOF를 사용했다.

```text
Command

SET A 1
SET B 2
INCR C

     │
     ▼

AOF File
```

Redis가 수행한 Write 명령을 기록해 두고 재시작할 때 다시 재생할 수 있다.

---

## 18. Pod 종료 시 AOF 백업

Kubernetes Pod가 종료되기 전에 실행할 수 있는 `preStop` Hook을 활용했다.

```text
Pod 종료 요청
     │
     ▼
preStop
     │
     ▼
AOF Backup
     │
     ▼
Object Storage
     │
     ▼
Pod 종료
```

원문에서는 `preStop`에서 백업 Script를 실행하도록 구성했다.

---

## 19. 평소에도 Backup

Pod가 정상적으로 종료된다는 보장은 없다.

```text
Process Crash

Node 장애

Network 문제

강제 종료
```

등이 있을 수 있다.
그래서 평상시에도 AOF 파일을 주기적으로 Object Storage에 저장했다.

```text
Redis
  │
  │ AOF
  ▼
Object Storage
```

기존 Redis를 RTO 내 복구하기 어렵다고 판단하면

```text
New Redis 생성
      │
      ▼
AOF 다운로드
      │
      ▼
Restore
      │
      ▼
서비스 복구
```

할 수 있도록 했다.

---

## 20. Kubernetes Self Healing

Bare Metal에서는 Redis 서버가 죽으면 운영자가 새로운 서버를 만들고 Redis를 설치한 뒤 Cluster에 Join해야 할 수 있다.

```text
Redis 장애
   │
   ▼
운영자 확인
   │
   ▼
새 서버
   │
   ▼
Redis 설치
   │
   ▼
Cluster Join
```

Kubernetes에서는 Pod 상태를 감시한다.

```text
Redis Pod
   X
   │
   ▼
Kubernetes 감지
   │
   ▼
새 Pod 실행
```

이를 통해 Redis 프로세스 복구를 자동화했다.

---

## 21. Self Healing과 Redis Failover는 다르다

여기서 구분해야 한다.

```text
Redis Failover

Master 장애
   │
   ▼
Replica → Master


Kubernetes Self Healing

죽은 Pod
   │
   ▼
새 Pod 생성
```

즉

```text
Redis
→ 데이터 역할 전환


Kubernetes
→ 실행 프로세스 복원
```

두 계층이 함께 고가용성을 만든다.

---

## 22. MaxMemory

Redis는 Memory 기반 시스템이다.
Redis가 사용할 수 있는 Memory를 적절하게 제한하지 않으면 OOM이나 Swap으로 인해 성능 문제가 생길 수 있다.

```text
Redis Memory 증가
      │
      ▼
Physical Memory 초과
      │
      ▼
OOM / Swap
      │
      ▼
Latency 급증
```

그래서 `maxmemory`를 설정했다.

```text
used_memory

      │
      ▼

maxmemory
```

한도를 넘으면 Eviction Policy에 따라 기존 데이터를 제거한다.

```text
Memory Full
    │
    ▼
Eviction
    │
    ▼
새 데이터 저장
```

부하 테스트를 통해 적정 MaxMemory 값을 찾았다.

---

## 23. 성능 검증

컨테이너로 옮겼다고 성능이 떨어지면 플랫폼으로 사용하기 어렵다.
그래서 `redis-benchmark`를 이용해 여러 명령의 성능을 검증했다.

```text
PING

SET

GET

INCR

LPUSH

HSET

...
```

그리고 Amazon ElastiCache와 비교했다.
원문에서는 AOF를 `everysec`으로 설정한 상태에서도 ElastiCache와 동등한 수준의 성능을 확인했다고 설명한다.

---

## 24. Redis as a Service

최종 목표는 DevOps 팀이 매번 Redis를 만들어주는 것이 아니었다.
서비스 개발팀이 필요할 때 직접 Redis를 생성한다.

```text
Developer
    │
    ▼
Developer Platform
    │
    ├─ Sentinel 선택
    │
    ├─ Cluster 선택
    │
    ├─ Spec 선택
    │
    └─ Replica / Shard 설정
    │
    ▼
Redis 자동 구성
```

지원 기능은 다음과 같았다.

```text
Redis Cluster / Sentinel 빠른 생성

Master / Replica / Shard 확장

Failover Test

Backup 설정
```

원문에서는 Redis Cluster나 Sentinel 구성을 약 5분 내 제공하는 자동화 환경을 목표로 설명한다.

---

## 25. Helm Chart

Redis 생성 과정을 표준화하기 위해 Helm Chart를 사용했다.

```text
Redis Helm Chart

├─ templates
├─ values
│   ├─ sandbox
│   ├─ stage
│   └─ production
└─ default values
```

환경별 차이는 Values로 관리하고 기본 구조는 동일한 Chart로 배포한다.

```text
Developer Request
       │
       ▼
Values 생성
       │
       ▼
Helm
       │
       ▼
Redis Deployment
```

---

## 26. Scale Out 자동화

트래픽이 증가하면 Redis를 확장해야 한다.
Sentinel이라면 Replica 확장,
Cluster라면 Master/Replica와 Shard 확장이 필요할 수 있다.

```text
트래픽 증가
     │
     ▼
Metric Threshold 초과
     │
     ▼
Redis Scale Out
```

Kubernetes 기반으로 이런 확장 작업을 자동화할 수 있도록 했다.

---

## 27. Failover Test도 자동화

HA 시스템은 구성만 해놓는다고 끝이 아니다.
실제로 장애를 만들어봐야 한다.

```text
정상 Cluster
     │
     ▼
Master 장애 주입
     │
     ▼
Failover
     │
     ▼
Application 정상 동작 확인
     │
     ▼
장애 해제
     │
     ▼
Cluster 정상화
```

원문에서는 플랫폼에서 다음 절차를 자동화했다.

```text
Cluster 상태 확인

Master 장애 발생

가용성 테스트

장애 복구

Cluster 상태 확인
```

이전에는 DevOps 팀이 하던 작업을 개발자가 직접 수행할 수 있게 했다.

---

## 28. Config도 선언형으로 관리

Bare Metal에서는 각 Redis Server의 설정 파일을 직접 관리해야 한다.
Kubernetes에서는 ConfigMap으로 관리한다.

```text
ConfigMap
    │
    ▼
Redis Config
    │
    ▼
Redis Pods
```

그리고 Helm Chart와 함께 배포한다.

```text
Git / Helm Values

       ↓

ConfigMap

       ↓

Redis
```

설정 변경 역시 코드 기반으로 관리할 수 있다.

---

## 29. Observability

Pod Annotation 등을 이용해 Prometheus가 Redis Metric을 자동 수집하게 했다.

```text
Redis Exporter
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

추가로

```text
OpenSearch

Grafana

Slack
```

등을 통해 Pod와 Redis 상태 이상을 감지할 수 있도록 구성했다.

---

## 30. 최종 아키텍처

전체를 합치면 다음과 같다.

```text
                     Developer
                         │
                         ▼
                Developer Platform
                         │
                         ▼
                    Helm Chart
                         │
              ┌──────────┴──────────┐
              ▼                     ▼

       Redis Sentinel          Redis Cluster

              │                     │
              ▼                     ▼
           HAProxy             Headless Service
              │                     │
              ▼                     ▼
        Current Master         Redis Pod List
                                    │
                                    ▼
                             Cluster-Aware Client


                 Kubernetes Cluster
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
            Zone A     Zone B     Zone C
              │          │          │
            Redis      Redis      Redis
              │          │          │
              └──── Replication ────┘

                         │
                         ▼
                        AOF
                         │
                         ▼
                  Object Storage


Monitoring

Redis Exporter
      │
      ▼
Prometheus
      │
      ▼
Grafana / Alert
```

---

## 31. 핵심

처음에는

```text
Redis를 Kubernetes에서 실행한다
```

라는 문제처럼 보인다.
하지만 실제 운영 문제는 훨씬 많다.

```text
Redis Node는 어떻게 통신할까?

외부 Client는 어떻게 접근할까?

Master가 변경되면?

Cluster Redirect는?

Pod가 죽으면?

Node를 교체하면?

데이터는 어떻게 보존할까?

Failover가 실제로 되는지 어떻게 검증할까?
```

이를 각각

```text
HostNetwork

HAProxy

Headless Service

Redis Replication

Kubernetes Self Healing

AOF Backup

Zone 분산

Helm 자동화

Failover Test
```

로 해결했다.
이 사례에서 가장 중요한 것은 **Redis를 단순히 Container에 넣은 것이 아니다.**
Redis가 원래 가지고 있던

```text
Replication

Master / Replica

Cluster Sharding

Redirect

Persistence
```

동작을 유지하면서 그 위에 Kubernetes의

```text
Scheduling

Service Discovery

Self Healing

Declarative Configuration

Automation
```

을 결합해 하나의 **Redis Platform**으로 만든 것이다.
