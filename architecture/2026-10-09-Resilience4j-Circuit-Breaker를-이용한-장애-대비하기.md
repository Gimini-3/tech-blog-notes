# Resilience4j Circuit Breaker를 이용한 장애 대비하기

- 읽은 날짜: 2026-10-09
- 회사/작성 주체: NHN CLOUD
- 원문: [Resilience4j Circuit Breaker를 이용한 장애 대비하기](https://meetup.nhncloud.com/posts/385)
- 노션 정리: [원본 페이지](https://app.notion.com/p/3e6a987f6c4080f4bdd5c680bf531307)

---

## Resilience4j Circuit Breaker를 이용한 장애 전파 방지

마이크로서비스 환경에서는 하나의 서비스가 다른 서비스를 호출하는 일이 많다.

```text
Client
  │
  ▼
Service A
  │
  ▼
Service B
  │
  ▼
Database
```

평소에는 문제가 없다.
하지만 Service B가 느려지거나 응답하지 않으면 어떻게 될까?

```text
Service A
   │
   │ API 호출
   ▼
Service B

응답 없음...

1초
2초
3초
...
Timeout
```

A는 설정된 Timeout까지 B의 응답을 기다린다.
이런 요청이 계속 들어오면 단순한 외부 API 장애가 **내 서비스의 장애로 전파****될 수 있다.**

---

## 1. Timeout 하나가 왜 전체 장애로 이어질까?

애플리케이션이 요청을 처리하려면 Thread가 필요하다.
하지만 Thread Pool의 크기는 제한되어 있다.

```text
Thread Pool

┌──────────────┐
│ Thread 1     │
│ Thread 2     │
│ Thread 3     │
│ Thread 4     │
│ Thread 5     │
└──────────────┘
```

외부 서비스가 느려지면

```text
Thread 1 ──▶ Service B 응답 대기
Thread 2 ──▶ Service B 응답 대기
Thread 3 ──▶ Service B 응답 대기
Thread 4 ──▶ Service B 응답 대기
Thread 5 ──▶ Service B 응답 대기
```

모든 Thread가 외부 응답을 기다리며 점유될 수 있다.
이후 들어오는 요청은 실행할 Thread가 없어 Queue에서 기다린다.
> 원문 이미지: [원문에서 보기](https://meetup.nhncloud.com/posts/385)

```text
하위 서비스 장애
      │
      ▼
응답 지연
      │
      ▼
Thread 장시간 점유
      │
      ▼
Thread Pool 고갈
      │
      ▼
Queue 증가
      │
      ▼
Timeout 증가
      │
      ▼
상위 서비스까지 장애
```

NHN Cloud 글에서 Circuit Breaker를 도입한 이유도 이런 연쇄 장애를 막기 위해서였다.

---

## 2. 이미 장애인데 계속 호출할 필요가 있을까?

외부 서비스가 이미 장애 상태라면 이런 처리가 반복된다.

```text
요청
 ↓
외부 API 호출
 ↓
오랫동안 대기
 ↓
실패
```

계속 요청을 보내는 것보다

```text
"현재 외부 서비스 장애"

        ↓

호출하지 않음

        ↓

즉시 실패
```

시키는 것이 내 서비스에는 더 안전하다.
이 역할을 하는 것이 **Circuit Breaker**다.

---

## 3. Circuit Breaker란?

Circuit Breaker는 실제 전기 차단기와 비슷하다.

```text
전기

과전류 발생
    │
    ▼
차단기 동작
    │
    X
전류 차단
```

소프트웨어에서는 외부 서비스의 실패가 일정 수준 이상 쌓이면 요청을 차단한다.

```text
External Service

실패 증가
    │
    ▼
Circuit Breaker OPEN
    │
    X
외부 호출 차단
```

즉 이미 장애가 난 서비스에 계속 요청을 보내면서 Thread와 자원을 소비하지 않도록 **Fail Fast**한다.

---

## 4. Resilience4j

Resilience4j는 Java 환경에서 사용할 수 있는 경량 Fault Tolerance 라이브러리다.
Circuit Breaker 외에도 다음 모듈을 제공한다.

```text
Resilience4j

├─ CircuitBreaker
├─ Retry
├─ RateLimiter
├─ Bulkhead
├─ TimeLimiter
└─ Cache
```

이번 글에서는 이 중 **CircuitBreaker**에 집중한다.

---

## 5. Circuit Breaker는 상태 머신이다

Resilience4j Circuit Breaker에는 주요 상태가 세 가지 있다.

```text
CLOSED

OPEN

HALF_OPEN
```

그리고 특수 상태로

```text
DISABLED

FORCED_OPEN
```

이 존재한다.
전체 상태 전환을 먼저 보면 다음과 같다.
> 원문 이미지: [원문에서 보기](https://meetup.nhncloud.com/posts/385)

```text
                   실패율 임계치 초과
             ┌────────────────────────┐
             │                        ▼
        ┌─────────┐               ┌────────┐
        │ CLOSED  │               │  OPEN  │
        └─────────┘               └────────┘
             ▲                        │
             │                        │ 일정 시간
             │                        ▼
             │                  ┌────────────┐
             │                  │ HALF_OPEN  │
             │                  └────────────┘
             │                    │        │
             │             정상   │        │ 실패
             └────────────────────┘        │
                                           ▼
                                         OPEN
```

---

## 6. CLOSED

정상 상태다.

```text
Request
   │
   ▼
Circuit Breaker [CLOSED]
   │
   ▼
External Service
```

요청을 외부 서비스에 그대로 전달하고 결과를 기록한다.

```text
SUCCESS
SUCCESS
FAILED
SUCCESS
FAILED
...
```

이 데이터를 이용해 외부 서비스가 정상인지 판단한다.

---

## 7. OPEN

실패율이나 Slow Call 비율이 설정한 임계치를 넘으면 OPEN 상태가 된다.

```text
CLOSED
   │
   │ 임계치 초과
   ▼
 OPEN
```

OPEN 상태에서는 외부 서비스까지 요청을 보내지 않는다.

```text
Request
   │
   ▼
Circuit Breaker [OPEN]
   │
   X
External Service
```

즉시 실패시키기 때문에 외부 서비스 Timeout을 기다릴 필요가 없다.

```text
외부 장애
   │
   ▼
OPEN
   │
   ▼
Fail Fast
   │
   ▼
Thread 점유 감소
```

---

## 8. HALF_OPEN

그렇다고 OPEN 상태를 영원히 유지할 수는 없다.
외부 서비스가 복구됐을 수도 있기 때문이다.
그래서 일정 시간이 지나면 일부 요청만 다시 허용한다.

```text
OPEN
 │
 │ 일정 시간
 ▼
HALF_OPEN
```

```text
HALF_OPEN

Request 1 ──▶ External Service
Request 2 ──▶ External Service
Request 3 ──▶ External Service
```

시험 요청 결과가 정상적이면

```text
HALF_OPEN
    │
    ▼
CLOSED
```

다시 문제가 있다면

```text
HALF_OPEN
    │
    ▼
OPEN
```

으로 돌아간다.

---

## 9. 실패율은 어떻게 계산할까?

Circuit Breaker는 모든 과거 요청을 계속 저장하지 않는다.
최근 일정 구간만 보고 판단한다.
이를 **Sliding Window**라고 한다.

```text
전체 요청

1 2 3 4 5 6 7 8 9 10 11 12 13

              └───────────────┘
                 최근 구간만 사용
```

Resilience4j에는 두 가지 Sliding Window가 있다.

---

## 10. Count-Based Sliding Window

최근 N개의 호출을 기준으로 판단한다.

```text
slidingWindowSize = 10
```

이라면 최근 10개만 본다.

```text
S  S  F  S  F  S  S  F  S  F

성공 = 6
실패 = 4

Failure Rate = 40%
```

새 요청이 들어오면 가장 오래된 결과가 빠진다.

```text
기존

1 2 3 4 5 6 7 8 9 10


새로운 요청 11

  2 3 4 5 6 7 8 9 10 11
```

즉 과거 전체가 아니라 **최근 상태**에 반응한다.

---

## 11. Time-Based Sliding Window

최근 N초 동안 발생한 요청을 본다.

```text
slidingWindowSize = 60
```

이라면

```text
현재 15:00:00

        최근 60초

14:59:00 ──────────── 15:00:00
```

구간에 포함된 호출만 집계한다.
차이는 간단하다.

```text
Count-Based

최근 100개 요청


Time-Based

최근 60초의 요청
```

---

## 12. minimumNumberOfCalls

실패율만 보고 바로 Circuit Breaker를 열면 문제가 있다.
첫 요청 하나가 실패했다고 해보자.

```text
총 요청 = 1

실패 = 1

Failure Rate = 100%
```

이것만 보고 OPEN시키는 것은 너무 민감하다.
그래서 판단을 시작하기 위한 **최소 호출 수**를 설정한다.

```text
minimumNumberOfCalls = 6
```

이면 최소 6개가 쌓이기 전까지 상태를 변경하지 않는다.

---

## 13. 실제 예제

다음 설정을 사용한다고 해보자.

```text
Sliding Window

Count-Based
Size = 10


minimumNumberOfCalls

6


failureRateThreshold

50%
```

즉

```text
최근 10개 호출을 보고

최소 6개가 모이면

실패율 50% 이상에서 OPEN
```

이라는 의미다.

---

## 14. 실패율 60%인데 CLOSED인 이유

5개의 요청이 들어왔다.

```text
1   SUCCESS
2   FAILED
3   FAILED
4   SUCCESS
5   FAILED
```

실패율은

```text
3 / 5 = 60%
```

이다.
임계치 50%보다 높다.
하지만

```text
minimumNumberOfCalls = 6
```

을 만족하지 않았다.
따라서

```text
Failure Rate = 60%

하지만

호출 수 = 5

        ↓

CLOSED 유지
```

한다.

---

## 15. 6번째 요청이 성공해도 OPEN될 수 있다

6번째 요청이 성공했다고 해보자.

```text
1   SUCCESS
2   FAILED
3   FAILED
4   SUCCESS
5   FAILED
6   SUCCESS
```

실패율은

```text
3 / 6 = 50%
```

이다.

```text
minimumNumberOfCalls 충족

        +

Failure Rate = Threshold

        ↓

CLOSED → OPEN
```

임계치와 같아도 OPEN된다.

---

## 16. 실패율이 낮다면 그대로 CLOSED

6개의 요청 중 2개만 실패했다면

```text
2 / 6 ≈ 33%
```

이다.

```text
Call Count >= 6

       O


Failure Rate >= 50%

       X

       ↓

CLOSED 유지
```

한다.

---

## 17. 느린 요청도 장애가 될 수 있다

Exception이 발생하지 않아도 지나치게 느리다면 서비스 입장에서는 장애와 비슷하다.

```text
HTTP 200 OK

하지만

응답시간 = 30초
```

Thread는 30초 동안 점유된다.
그래서 Resilience4j는 **Slow Call**도 따로 집계한다.

---

## 18. slowCallDurationThreshold

몇 초 이상이면 느린 요청으로 판단할지를 설정한다.

```text
slowCallDurationThreshold = 3초
```

라면

```text
500ms
→ 정상

2초
→ 정상

5초
→ Slow
```

이다.
성공 여부와 Slow 여부는 별개다.

```text
5초 + SUCCESS

→ SLOW_SUCCESS


5초 + FAILED

→ SLOW_ERROR
```

따라서 실패한 요청이면서 동시에 Slow Call일 수도 있다.

---

## 19. slowCallRateThreshold

느린 요청의 비율이 얼마나 높아졌을 때 OPEN할지도 설정할 수 있다.

```text
slowCallRateThreshold = 50%
```

에서

```text
총 10개

Slow = 5개
```

라면 Slow Call Rate는 50%다.
따라서 Circuit Breaker는

```text
Failure Rate

또는

Slow Call Rate
```

를 이용해 외부 서비스의 이상 상태를 판단할 수 있다.

---

## 20. OPEN 상태는 얼마나 유지할까?

OPEN 상태가 되자마자 다시 요청을 보내면 Circuit Breaker를 둔 의미가 없다.
그래서 일정 시간을 기다린다.

```text
waitDurationInOpenState = 60초
```

라면

```text
OPEN

0초
 │
30초
 │
60초
 ▼

HALF_OPEN 전환 가능
```

이다.

---

## 21. automaticTransitionFromOpenToHalfOpenEnabled

OPEN에서 HALF_OPEN으로 넘어가는 방식도 설정할 수 있다.
기본값이 false라면 시간이 지나도 요청이 들어오기 전까지 상태 전환 확인이 이루어지지 않는다.

```text
OPEN
 │
 │ 60초 경과
 │
 │ 다음 요청
 ▼
HALF_OPEN
```

true라면 설정 시간이 지나면 자동으로 HALF_OPEN으로 전환한다.

---

## 22. permittedNumberOfCallsInHalfOpenState

HALF_OPEN에서 모든 트래픽을 한꺼번에 다시 열면 위험하다.

```text
서비스 복구 직후

1000 Request

       ↓

한꺼번에 유입

       ↓

다시 장애
```

그래서 시험할 호출 수를 제한한다.

```text
permittedNumberOfCallsInHalfOpenState = 6
```

이라면 6개의 요청 결과만 보고 복구 여부를 확인한다.

---

## 23. HALF_OPEN → CLOSED

시험 요청 6개 중 2개가 실패했다고 해보자.

```text
SUCCESS
SUCCESS
FAILED
SUCCESS
SUCCESS
FAILED
```

```text
Failure Rate

2 / 6

≈ 33%
```

임계치 50%보다 낮다.

```text
HALF_OPEN

   │
   │ 33%
   ▼

CLOSED
```

외부 서비스가 복구됐다고 판단한다.

---

## 24. HALF_OPEN → OPEN

반대로 6개 중 3개가 실패하면

```text
Failure Rate = 50%
```

이다.

```text
HALF_OPEN

    │
    │ 50%
    ▼

OPEN
```

장애가 계속되고 있다고 판단하고 다시 호출을 차단한다.

---

## 25. HALF_OPEN이 너무 오래 지속되는 것도 막는다

HALF_OPEN 상태인데 요청 자체가 거의 없다면 충분한 결과를 모으지 못할 수 있다.

```text
HALF_OPEN

Request 1

...

이후 요청 없음
```

이때 사용하는 것이

```text
maxWaitDurationInHalfOpenState
```

다.
최대 시간을 넘기면

```text
HALF_OPEN
    │
    │ 시간 초과
    ▼
OPEN
```

으로 돌아간다.

---

## 26. 모든 Exception이 장애는 아니다

Circuit Breaker에서 또 중요한 부분은
> 어떤 오류를 실패로 볼 것인가?
다.
예를 들어 사용자의 잘못된 요청으로 400 응답이 발생했다고 해보자.

```text
잘못된 Request
      │
      ▼
HTTP 400
```

외부 서비스가 장애인 것은 아니다.
이런 요청까지 실패율에 포함하면

```text
사용자 오류 증가
      │
      ▼
Failure Rate 증가
      │
      ▼
Circuit Breaker OPEN
```

이라는 잘못된 판단이 가능하다.

---

## 27. recordException

`recordException`은 발생한 Exception을 Circuit Breaker의 실패로 기록할지를 판단한다.

```text
Exception
    │
    ▼
recordException
    │
 ┌──┴──┐
 ▼     ▼
true  false
 │      │
 ▼      ▼
실패   실패로
집계   기록하지 않음
```

Timeout이나 외부 서비스 5xx 관련 Exception만 장애로 판단하는 식으로 사용할 수 있다.

---

## 28. recordResult

반대로 Exception은 발생하지 않았는데 응답 내용 자체가 실패일 수도 있다.

```text
HTTP 200

{
   "result": "FAIL"
}
```

이런 경우 `recordResult`를 사용한다.

```text
Response
   │
   ▼
recordResult
   │
   ├─ true  → 실패로 기록
   │
   └─ false → 정상
```

즉 Exception뿐 아니라 **Response 자체를 기준으로 실패를 정의할 수 있다.**

---

## 29. ignoreException

어떤 Exception은 아예 통계에서 제외할 수도 있다.

```text
Exception
   │
   ▼
ignoreException == true
   │
   ▼
성공에도 포함 X

실패에도 포함 X
```

즉 단순히 성공 처리하는 것이 아니라 **Circuit Breaker의 집계 대상 자체에서 제외**한다.

---

## 30. 실패 판정 구조

정리하면 하나의 외부 요청 결과도 다음과 같은 과정을 거칠 수 있다.

```text
External API
     │
     ▼
응답 / Exception
     │
     ▼
실패 판단
     │
 ┌───┴────────────┐
 │                │
Exception       Response
 │                │
 ▼                ▼
recordException  recordResult
 │                │
 └───────┬────────┘
         ▼
   성공 / 실패 기록

또는

ignoreException
      │
      ▼
집계에서 제외
```

따라서 Circuit Breaker 설정에서 중요한 것은 단순한 임계치뿐 아니라 **무엇을 장애라고 정의할 것인지**다.

---

## 31. Stack Trace 비용도 줄일 수 있다

OPEN 상태에서는 많은 요청이 빠르게 차단될 수 있다.

```text
1000 Requests
     │
     ▼
OPEN
     │
     ▼
1000 Exceptions
```

매번 Stack Trace까지 생성하면 불필요한 로그 비용이 커질 수 있다.
Resilience4j는

```text
writableStackTraceEnabled
```

설정을 제공한다.
원문에서는 이를 비활성화하면 반복되는 OPEN 상태 오류에서 Stack Trace 생성을 줄이는 데 활용할 수 있다고 설명한다.

---

## 32. 주요 설정을 흐름으로 보면

설정을 각각 외우기보다 상태 머신에 붙여서 보면 이해하기 쉽다.

```text
               CLOSED
                  │
                  ▼
           Sliding Window
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
failureRate             slowCallRate
Threshold                Threshold
       │                     │
       └──────────┬──────────┘
                  │
        minimumNumberOfCalls
                  │
                  ▼
                OPEN
                  │
                  ▼
       waitDurationInOpenState
                  │
                  ▼
             HALF_OPEN
                  │
                  ▼
permittedNumberOfCallsInHalfOpenState
                  │
             시험 호출
                  │
            ┌─────┴─────┐
            ▼           ▼
         CLOSED        OPEN
```

---

## 33. 주요 설정 한 번에 정리

```text
slidingWindowSize
→ 최근 어느 정도의 호출을 볼 것인가


minimumNumberOfCalls
→ 최소 몇 개가 쌓여야 판단할 것인가


failureRateThreshold
→ 실패율 몇 %부터 장애인가


slowCallDurationThreshold
→ 몇 초 이상이면 Slow Call인가


slowCallRateThreshold
→ Slow Call 몇 %부터 장애인가


waitDurationInOpenState
→ OPEN 상태에서 얼마나 기다릴 것인가


permittedNumberOfCallsInHalfOpenState
→ 복구 확인 시 몇 개를 시험할 것인가


maxWaitDurationInHalfOpenState
→ HALF_OPEN을 얼마나 오래 유지할 것인가


recordException
→ 어떤 Exception을 실패로 볼 것인가


recordResult
→ 어떤 Response를 실패로 볼 것인가


ignoreException
→ 어떤 오류를 집계에서 제외할 것인가
```

---

## 34. Circuit Breaker만 있으면 충분할까?

Circuit Breaker가 Timeout 자체를 대신하는 것은 아니다.
예를 들어 Circuit Breaker가 CLOSED인데 외부 API Timeout이 30초라면

```text
Request
   │
   ▼
Circuit Breaker
   │
   ▼
External API
   │
   │
30초 대기
```

동안 Thread는 계속 점유될 수 있다.
그래서 Resilience4j에서는 Circuit Breaker 외에도

```text
TimeLimiter

Retry

Bulkhead

RateLimiter
```

등을 함께 제공한다.
Circuit Breaker는 전체 장애 대응 전략 중 하나라고 보는 것이 맞다.

---

## 35. Circuit Breaker가 보호하는 것은 누구일까?

Circuit Breaker가 외부 서비스를 복구해주는 것은 아니다.

```text
External Service

여전히 장애
```

Circuit Breaker가 보호하는 것은 **호출하는 쪽 서비스**다.

```text
External Service 장애
        │
        X
Circuit Breaker OPEN
        │
        ▼
우리 서비스의

Thread
Connection
Resource

보호
```

즉 핵심은 **Failure Isolation**, 장애 격리다.

---

## 36. 전체 동작 원리

지금까지 내용을 하나로 연결하면 다음과 같다.

```text
                    Request
                       │
                       ▼
              Circuit Breaker
                       │
                   CLOSED
                       │
                       ▼
                External API
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          SUCCESS              FAILED
             │                   │
             └─────────┬─────────┘
                       ▼
                Slow 여부 확인
                       │
                       ▼
                Sliding Window
                       │
                 결과 집계
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   Failure Rate               Slow Call Rate
          │                         │
          └────────────┬────────────┘
                       │
             minimumNumberOfCalls
                       │
                       ▼
                임계치 초과?
                       │
                       ▼
                     OPEN
                       │
                 호출 즉시 차단
                       │
                       ▼
                 일정 시간 대기
                       │
                       ▼
                  HALF_OPEN
                       │
               일부 요청만 허용
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          정상화              장애 지속
             │                   │
             ▼                   ▼
           CLOSED              OPEN
```

---

## 37. 이 사례에서 중요한 점

처음 문제는 단순했다.

```text
외부 서비스가 느리다.
```

하지만 실제 장애 흐름은

```text
외부 서비스 지연
       │
       ▼
Thread 대기
       │
       ▼
Thread Pool 고갈
       │
       ▼
Queue 증가
       │
       ▼
Timeout 증가
       │
       ▼
정상 서비스까지 장애
```

였다.
Circuit Breaker는 이 흐름을 중간에서 끊는다.

```text
외부 장애 감지
       │
       ▼
OPEN
       │
       ▼
추가 요청 차단
       │
       ▼
Fail Fast
       │
       ▼
Thread / Resource 보호
       │
       ▼
장애 전파 방지
```

따라서 Circuit Breaker의 목적은 **장애가 난 시스템을 고치는 것이 아니다.**
**장애 난 시스템 때문에 정상인 내 서비스까지 같이 장애가 나는 것을 막는 것**이다.

---

## 정리

Circuit Breaker는 단순히

```text
실패가 많다

→ 요청 차단
```

하는 기능이 아니다.
실제로는

```text
최근 호출 결과 수집
       │
       ▼
Sliding Window 집계
       │
       ▼
Failure / Slow Rate 계산
       │
       ▼
외부 서비스 장애 판단
       │
       ▼
OPEN
       │
       ▼
Fail Fast
       │
       ▼
일정 시간 후 HALF_OPEN
       │
       ▼
일부 요청으로 복구 확인
       │
       ├─ 정상 → CLOSED
       │
       └─ 장애 → OPEN
```

이라는 **상태 기반 피드백 루프**다.
그리고 이 구조에서 가장 중요한 것은 설정 값을 외우는 것이 아니라

```text
어떤 요청을 실패로 볼 것인가?

최근 어느 정도의 요청을 볼 것인가?

몇 번의 요청이 있어야 장애라고 판단할 것인가?

얼마 동안 요청을 차단할 것인가?

복구 여부를 몇 개의 요청으로 확인할 것인가?
```

를 서비스 특성에 맞게 결정하는 것이다.
결국 Circuit Breaker는 **외부 시스템의 최근 상태를 관찰하면서 호출을 계속할지, 잠시 끊을지 스스로 결정하는 상태 머신**이라고 이해하면 된다.
