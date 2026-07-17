# Server-Sent Events로 실시간 알림 전달하기

- 읽은 날짜: 2025-11-13
- 회사/작성 주체: 우아한형제들
- 원문: [Server-Sent Events로 실시간 알림 전달하기](https://techblog.woowahan.com/23199/)

---

그동안 알림 시스템은 큰 문제 없이 잘 동작해 왔다. 하지만 서비스가 커지고 사용자가 늘어나면서, 더 빠르고 안정적인 알림 전송이 중요해졌다.

## 기존의 알림 전송 방식

---

기존의 알림은 MQTT(Message Queuing for Telemetry Transport)를 통하여 전달하였다.
MQTT는 경량 메시지 프로토콜의 한 종류이다.

## MQTT란?

MQTT는 경량 발행/구독 메시징 프로토콜이다. Publisher와 Subscriber는 서로 직접 통신하지 않고 Broker를 통해 메시지를 주고받는다. 작은 헤더와 QoS 수준을 제공해 네트워크와 전력이 제한된 IoT 환경에서 널리 사용된다.

MQTT는 애플리케이션 계층의 메시징 규칙이고, WebSocket은 양방향 통신 채널이다. 두 기술은 대체 관계가 아니며 MQTT 메시지를 WebSocket 연결 위로 전송할 수도 있다.

### 전체 아키텍처 개요

![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-01.png)

- 도메인 서비스: 주문, 배달 등 각 도메인에서 이벤트 발행
- MQTT Publisher: 메시지 브로커로 전달
- MQTT Broker: 클라이언트 연결 관리 및 메시지 전달
- 클라이언트: PC, 모바일, POS 등 다양한 접수 프로그램

### 기존 알림 전송 방식의 문제점

기존 시스템은 MQTT 메시지에 실제 알림 내용을 넣지 않고 알림 발생 신호나 식별자만 보내는 Zero Payload 방식을 사용했다. 레거시 클라이언트 라이브러리의 페이로드 크기 제약을 피할 수 있었지만, 클라이언트가 신호를 받은 뒤 API를 다시 호출해야 했다.

이 우회 방식에는 두 가지 문제가 있었다.

1. 기본 알림 문구가 클라이언트에 하드코딩되어 문구를 바꾸려면 앱을 다시 배포해야 했다.
2. 실제 내용을 얻기 위한 추가 API 호출 때문에 사용자에게 알림이 보이기까지 지연이 발생했다.

#### 접근 제어 부족

Zero Payload는 노출되는 정보의 양을 줄이지만 보안을 대신하지는 않는다. 식별자 자체가 민감할 수 있고, 권한 없는 클라이언트가 다른 파트너의 Topic을 구독할 수 있다는 사실만으로도 접근 제어 문제다. 따라서 Broker의 연결·구독 권한과 후속 API의 인증·인가를 모두 적용해야 한다.

#### 네트워크 방화벽 차단

MQTT는 일반적으로 1883 포트, TLS 연결은 8883 포트를 사용한다. 일부 회사나 공용 네트워크에서는 이 포트가 차단되어 지속 연결을 만들지 못했다. 메시지 누락을 보완하기 위해 HTTP 기반 Polling을 병행했지만, 주기적인 요청 때문에 네트워크 비용과 서버 부하가 늘어났다.

#### WebView 연결의 어려움

모바일 앱과 WebView를 함께 지원해야 했지만 WebView에서는 MQTT 연결, 인증서 처리와 여러 탭의 연결 관리가 복잡했다. 이 제약들은 HTTP 기반 SSE를 검토하게 된 배경이 됐다.

## AWS IoT 도입

---

AWS IoT(Internet of Things)는 AWS에서 사물 인터넷을 관리하기 쉽게 도와주는 서비스이다.
쉽게 사용할 수 있도록 MQTT Broker를 제공하고, 추가로 보안, 각각의 디바이스 관리 및 모니터링을 제공한다.

### 메시지 보안 적용

AWS IoT를 통해 제일 먼저 연결과 구독에 보안을 적용하였다.
메시지를 클라이언트가 정의된 형식에 따라 노출하기만 하면 되도록 변경할 예정이었기에 보안이 필수였다.

```java
/**
 * AWS IoT 디바이스가 특정 Topic에 접근할 수 있도록 허용하는 정책(Policy)을 생성한다.
 * 이 정책은 AWS IAM Policy 문법(Statement)을 사용하여 구독 및 수신 권한을 정의한다.
 *
 * @param accountId AWS 계정 ID. 리소스 ARN을 구성하는 데 사용된다.
 * @param merchantNo 가맹점 번호 또는 식별자. MQTT Topic 필터의 일부로 사용되어 해당 가맹점 전용 Topic에만 접근하도록 제한한다.
 * @return IotPolicy 객체. AWS IoT에 적용할 수 있는 정책 정의를 포함한다.
 */
fun create(accountId: String, merchantNo: String): IotPolicy {
    return IotPolicy(
        // AWS 정책 문법 버전. 이 버전("2012-10-17")이 현재 표준이다.
        version = "2012-10-17",
        statement = listOf(
            // --- 1. Topic 구독(Subscribe) 권한 부여 Statement ---
            Statement(
                effect = "Allow", // 권한 허용
                action = "iot:Subscribe", // 수행할 작업: 특정 Topic Filter를 구독할 수 있는 권한
                resource = listOf(
                    // 리소스(Topic Filter) 정의:
                    // arn:aws:iot:리전:계정ID:topicfilter/고정프리픽스/가맹점번호/*
                    // 'rch/$merchantNo/*' 하위의 모든 Topic Filter에 대해 구독을 허용한다.
                    // Topic Filter는 # 또는 + 와일드카드를 포함할 수 있는 패턴이다.
                    "arn:aws:iot:ap-northeast-2:$accountId:topicfilter/rch/$merchantNo/*",
                )
            ),
            // --- 2. 메시지 수신(Receive) 권한 부여 Statement ---
            Statement(
                effect = "Allow", // 권한 허용
                action = "iot:Receive", // 수행할 작업: 구독한 Topic을 통해 실제로 메시지를 수신하는 권한
                resource = listOf(
                    // 리소스(Topic) 정의:
                    // arn:aws:iot:리전:계정ID:topic/고정프리픽스/가맹점번호/*
                    // 'rch/$merchantNo/*' 하위의 모든 실제 Topic으로부터 메시지 수신을 허용한다.
                    // 'topicfilter'와 달리 'topic'은 실제 메시지가 게시되는 Topic을 나타낸다.
                    "arn:aws:iot:ap-northeast-2:$accountId:topic/rch/$merchantNo/*",
                )
            )
        )
    )
}
```

이를 통해 각 클라이언트 인가된 Topic의 메시지만 구독하도록 하여 보안을 강화할 수 있었다.

### 메시지 정형화

메시지의 형식을 변경하여도 문제가 없다.
기존의 Zero Payload 방식에서 정형화된 메시지 형식으로 제공하고, 클라이언트는 노출만 하도록 하였다.

#### 장점

- 클라이언트의 버전을 업데이트하지 않더라도, 새로운 알림을 노출할 수 있다.
- 메시지의 표현이 바뀌어도 모든 버전에서 다 같이 바뀔 수 있다.

```java
{
  "id": "123",
  "notification": {
    "groupId": "order-notification",
    "title": "신규 주문",
    "content": "배달ABCD - 김치찌개 외 2건 / 15,000원",
    "eventTypes": [
      "팝업 실행"
    ],
    "soundTypes": [
      "신규 주문 알림 재생"
    ]
  },
  "data": {
    "..."
  },
  "timestamp": "2025-02-05T20:30:29.443+09:00"
}
```

이렇게 AWS IoT를 적용하였지만, 남은 문제들이 있다.
AWS IoT도 MQTT를 사용하기 때문에, 네트워크 방화벽으로 MQTT 수신이 차단된 환경에서는 Polling이 필요했다. 그리고 아직 Webview에서는 메시지를 수신할 수 없었다.

## Server-Sent Events 도입

---

나머지 문제를 해결하기 위해 Server-Sent Events를 도입하였다.
SSE(Server-Sent Events)는 HTTP 연결을 통해 서버에서 클라이언트로 향하는 단방향 통신 채널을 만든다.
웹소켓(WebSockets)의 양방향 연결과는 달리, SSE는 서버가 클라이언트에게 업데이트 정보를 보내주기 위해 HTTP 연결을 계속 열어두는 방식이다.

### 실시간 통신 방식 비교 (Polling vs SSE vs WebSockets)

<table header-row="true">
<tr>
<td>**특징**</td>
<td>**Polling (폴링)**</td>
<td>**Server-Sent Events (SSE)**</td>
<td>**WebSockets (웹소켓)**</td>
</tr>
<tr>
<td>**연결 방식**</td>
<td>HTTP/S 요청-응답</td>
<td>HTTP/S 기반의 단일 연결 유지</td>
<td>HTTP에서 업그레이드된 별도 프로토콜 (ws://, wss://)</td>
</tr>
<tr>
<td>**데이터 흐름**</td>
<td>**클라이언트 → 서버**(단방향 요청)</td>
<td>**서버 → 클라이언트**(단방향 푸시)</td>
<td>**클라이언트 ↔ 서버**(양방향 소통)</td>
</tr>
<tr>
<td>**효율성**</td>
<td>🔴 **좋지 않음**</td>
<td>🟢 **좋음**</td>
<td>🟢 **좋음**</td>
</tr>
<tr>
<td>**구현 난이도**</td>
<td>쉬움</td>
<td>보통</td>
<td>다소 복잡함</td>
</tr>
</table>
### WebSocket 대신 SSE를 선택한 이유
이미 양방향 통신을 하고 있었다.
클라이언트에서 서버로는 REST API를 사용하고, 서버에서 클라이언트로는 실시간 메시지를 전달하는 구조이다.
그렇다면 왜 WebSocket을 사용하지 않았을까?

#### 1. 기존 아키텍처와의 정합성

- 이미 안정적으로 운영 중인 REST API 인프라가 있는 상황에서 WebSocket으로 전환하려면 모든 API를 WebSocket 기반으로 재구현해야 한다.
- 두 가지 프로토콜을 동시에 운영하는 것보다 REST API + SSE 조합이 관리 비용 측면에서 효율적이다.

#### 2. 단방향 통신으로 충분

- 현재 서비스는 서버에서 클라이언트로의 알람 전달이 핵심이다.
- 클라어언트의 응답이 필요한 경우 기존 REST API를 활용하면 충분하다.
- WebSocket의 양방향 통신은 오버엔지니어링이 될 수 있다.
  이러한 이유로 SSE가 최고의 선택이었다.

## 메시지 전달 아키텍처

---

### 메시지 전달 방식 검토

클라이언트에게 메시지를 전달할 수 있는 방식에는 크게 세 가지 방법을 고민하였다.

1. 세션이 연결된 서버의 API 호출
2. 세션이 연결된 서버로 메시지 전달
3. 모든 서버로 메시지 전달

### 메시지 전달 방식 선택

#### 1. 세션이 연결된 서버의 API 호출

메시지 발행자가 클라이언트가 연결된 서버를 찾아 직접 API를 호출하는 방식이다.
![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-02.png)
**장점:**

- **직접 전달**: 연결된 서버에만 메시지를 보내므로 불필요한 네트워크 홉 최소화
- **레이턴시 최적화**: 브로커를 거치지 않아 상대적으로 빠른 전달 가능
  **단점:**

1. **세션 추적 복잡도**
   - 클라이언트가 어느 서버에 연결되어 있는지 별도 관리 필요
   - Redis 등 세션 저장소 구축 및 실시간 동기화 필요
2. **메시지 유실 위험**
   - API 호출 시점에 서버 재시작 중이면 메시지 유실
   - 네트워크 타임아웃 발생 시 재시도 로직 복잡
3. **보안 문제**
   - SSE 서버의 메시지 전송 API를 외부에 노출 시 보안 위험

---

#### 2. 세션이 연결된 서버로 메시지 전달

브로커를 사용하되 특정 서버 인스턴스에만 메시지를 라우팅하는 방식
![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-03.png)
**장점:**

- **순서 보장 및 유실 방지**: 브로커의 장점을 활용하여 메시지 순서 보장 및 영속화
- **타겟팅 전달**: 특정 서버에만 메시지를 전달하여 불필요한 네트워크 홉 감소
  **단점:**

1. **복잡한 토픽 관리**
   - SSE 서버 인스턴스마다 별도 토픽 필요
   - 서버 10대면 토픽도 10개 관리해야 하는 운영 부담
2. **세션 추적 여전히 필요**
   - 클라이언트가 어느 서버에 연결되어 있는지 알아야 올바른 토픽에 발행 가능
   - 결국 1번 방식의 세션 추적 문제 존재
3. **동적 확장 어려움**
   - 서버 추가/제거 시 토픽 생성/삭제 필요
   - Auto Scaling 환경에서 관리 복잡도 증가
4. **부하 불균형**
   - 현재 서비스에서 균등한 클라이언트 라우팅 어려움
   - 서버 간 메시지 처리량 불균형 발생 가능

---

#### 3. 모든 서버로 메시지 전달

메시지 발행자와 소비자 사이에 브로커를 두는 방식
![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-04.png)
**장점:**

- **느슨한 결합**: 메시지 발행자는 클라이언트의 연결 상태나 서버 위치를 알 필요 없음
- **메시지 순서 보장**: 브로커의 파티션/토픽 기반으로 순서 보장 가능
- **확장성**: 서버 인스턴스 추가 시 브로커 구독만 하면 됨
- **장애 격리**: 특정 서버 장애가 전체 메시지 흐름에 영향을 주지 않음
  **단점:**
- **레이턴시 증가**: 세션의 위치를 모르기 때문에 모든 SSE 서버가 메시지를 받아 처리해야 하며, 불필요한 네트워크 홉 발생
- **브로커 인프라 운영 필요**: 별도의 메시지 브로커 시스템 구축 및 관리 필요

---

#### 3번 메시지 브로커 방식 선택

의사결정 이유:

1. **메시지 순서 및 유실 방지**: Kafka와 같은 브로커는 파티션 내 메시지 순서를 보장하고, 컨슈머가 처리할 때까지 메시지를 보관
2. **확장성**: SSE 서버 인스턴스를 추가하더라도 브로커를 구독하기만 하면 되므로, Auto Scaling 환경에서도 유연하게 대응
3. **느슨한 결합**: 메시지 발행자는 클라이언트 연결 상태를 전혀 신경 쓸 필요가 없다. 각 도메인 서비스는 브로커에 메시지만 발행하면 된다.
4. **장애 격리**: 특정 SSE 서버가 재시작되거나 장애가 발생해도, 브로커에 메시지가 보관되어 있어 복구 후 처리 가능하다.
5. **개발 용이성**: 여러 인프라를 사용하지 않고도 코드 레벨에서 서비스를 관리할 수 있음
   네트워크 홉이 증가하고 수십ms단위의 레이턴시가 증가하지만, **정확성과 안정성**이 더 중요하므로 허용 가능한 수준이었다.

## 메시지 순서 보장 및 유실 방지

---

클라이언트와 연결이 되어 있더라도 메시지를 전달하는 Broker에 따라 메시지 순서가 역전되거나 유실이 생길 수 있다.
이를 해결하기 위해 다음과 같은 작업을 했다.

- 파트너별로 파티션을 분리하여 메시지 순서를 보장
- SSE 연결이 끊어진 경우를 대비하여 메시지 이력을 일정 기간 보관
- Last-Event-ID를 통해 재연결 시 놓친 메시지를 차례대로 재전송
- 메시지 전송 실패 시 재시도 메커니즘 구현

### 클라이언트와의 연결

Spring Webflux + Coroutine 환경으로 구축
클라이언트는 Last-Event-ID 값과 함께 text/event-stream으로 연결하고, 서버는 해당 요청에 Flow를 응답한다.

### Last-Event-ID란?

`Last-Event-ID`는 클라이언트가 마지막으로 정상 수신한 SSE 이벤트 ID를 서버에 전달하는 HTTP 헤더다. 연결이 끊긴 뒤 재접속할 때 서버는 이 값을 기준으로 이후 이벤트를 다시 전송할 수 있다. 단, 서버가 이벤트 ID와 이력을 보관하고 재전송 로직을 구현해야 실제 유실 방지가 가능하다.

```java
/**
 * REST 컨트롤러 정의.
 * 이 클래스는 Server-Sent Events (SSE)를 통해 클라이언트에게 실시간 알림 스트림을 제공하는 엔드포인트를 관리한다.
 */
@RestController
class ServerSentEventController(
    // SSE 연결 로직을 처리하는 UseCase(서비스 레이어)를 의존성 주입(DI)으로 받는다.
    private val serverSentEventConnectUseCase: ServerSentEventConnectUseCase,
) {

    /**
     * 클라이언트의 SSE 연결 요청을 처리하는 엔드포인트.
     *
     * @GetMapping(produces = [MediaType.TEXT_EVENT_STREAM_VALUE]):
     * - HTTP GET 요청을 처리한다.
     * - `produces` 속성을 `text/event-stream`으로 설정하여 이 응답이 SSE 스트림임을 명시한다.
     *
     * @param requestUserInfo 인증된 사용자/디바이스 정보 (가맹점 번호, 디바이스 ID 등 포함).
     * @param lastEventId 클라이언트가 HTTP 헤더를 통해 전달하는 마지막으로 수신한 이벤트 ID.
     * 네트워크 단절 후 재연결 시 누락된 이벤트부터 다시 받기 위해 사용된다 (SSE의 핵심 기능).
     * 헤더가 없을 수 있으므로 Nullable(`String?`)로 정의한다.
     * @return Flow<ServerSentEvent<String>>:
     * - Kotlin의 코루틴 Flow를 반환하며, 이는 스트림 형태의 데이터를 의미한다.
     * - 서버는 이 Flow를 통해 이벤트 스트림을 클라이언트에 지속적으로 밀어 넣는다(Push).
     * - `ServerSentEvent<String>`은 실제 이벤트 데이터와 ID, 이벤트 타입 등을 포함하는 객체다.
     */
    @GetMapping(produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    suspend fun connect(
        requestUserInfo: RequestUserInfo,
        @RequestHeader("Last-Event-ID") lastEventId: String?,
    ): Flow<ServerSentEvent<String>> = serverSentEventConnectUseCase.connect(
        // UseCase에 필요한 식별 정보와 마지막 이벤트 ID를 전달하여 연결 및 스트림을 시작한다.
        requestUserInfo.merchantNo,
        requestUserInfo.deviceId,
        requestUserInfo.channelType,
        lastEventId,
    )
}
```

connect 메서드는 Coroutine Flow를 반환하여, 연결이 유지되는 동안 서버에서 flow의 emit을 호출할 때마다 실시간으로 메시지를 수신할 수 있다.

```java
event:heartbeat

id:test-id
event:test
data:test
```

### Last-Event-ID를 활용한 메시지 유실 방지

클라이언트는 최초 연결일 경우 Last-Event-ID에 값을 넣어주지 않지만, 한 번 연결된 이후로부터는 메시지 유실을 방지하기 위해 마지막으로 수신했던 ID를 header에 포함한다.
![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-05.png)

### 세션 관리 및 생명주기

서버는 클라이언트와 연결되었을 때 Session을 생성하고 메모리에 저장한다. 그리고 Last-Event-ID 기준으로 전달받지 못한 이벤트가 있다면 전달한다. 동시에 heartbeat 메시지를 전달한다.
연결이 종료되면 세션을 정리하는 작업을 수행한다.

```java
override suspend fun connect(
    merchantNo: String,
    deviceId: String,
    channelType: ChannelType,
    lastEventId: String?,
): Flow<ServerSentEvent<String>> {
    val session = serverSentEventSessionUseCase.createSession(merchantNo, deviceId, channelType)
    return channelFlow {
        // 과거 이벤트 전송 (Last-Event-ID 기준)
        launch {
            serverSentEventPublishUseCase.getPastEvents(
                lastEventId,
                merchantNo,
                deviceId,
            ).forEach {
                session.emit(it)
            }
        }

        // Heartbeat 전송
        launch {
            startHeartBeat(session)
        }

        // 연결 종료 시 세션 취소
        awaitClose {
            session.cancel()
        }
    }.onCompletion {
        // 세션 정리
        withContext(NonCancellable) {
            serverSentEventSessionUseCase.closeSession(session)
        }
    }
}
```

## 메시지 보안

---

현재 구성한 메시지 전달 방식은 각각의 파트너에게 메시지를 전달하고 있어서 다른 클라이언트의 메시지를 수신할 방법은 없다.
하지만 나쁜 의도로 각각의 클라이언트의 정보로 연결하게 된다면 메시지가 유출될 수 있는 상황이 생길 수 있다. (물론 연결에 인증과 HMAC을 통해 철저한 보안을 하고 있다.)

### 중복 연결 방지

각각의 기기에 대해 중복으로 메시지를 수신할 수 없도록 하였다.

- 연결 시 Session ID 기반으로 활성 세션을 관리
- 새로운 연결 시 기존 연결을 자동으로 종료
- 메시지 전송 시 활성 세션인지 확인하여 활성 세션에만 메시지 전달
- 잘못된 연결의 세션은 강제 종료
  ![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-06.png)

## 메시지 유실 방어

---

주문 접수 프로그램에서는 반드시 딱 1번 전달해야 하는 이벤트들이 있다. 여러 이벤트가 있지만 그중 하나의 예로 주문서 프린트가 있다. 주문서 프린트는 주문 알림뿐만 아니라 파트너가 조리를 시작하고 신규 주문이 인입되었다는 가장 중요한 신호로 활용된다.
![](assets/2025-11-13-Server-Sent-Events로-실시간-알림-전달하기/image-07.png)
만약 같은 주문서가 여러 번 출력되면 파트너가 중복 주문으로 오인할 수 있고, 반대로 주문서가 출력되지 않으면 신규 주문을 인지하지 못하는 치명적인 상황이 발생할 수 있다. 이처럼 중요한 이벤트가 유실되거나 중복으로 전송된다면 사용자 경혐을 크게 해치게 된다.

### CommitEvent를 통한 수신 확인

이런 문제를 해결하고자 중요한 이벤트들에 대해 수신 확인을 하는 절차를 만들었다.
중요한 이벤트들을 CommitEvent라는 interface로 정의하고, 해당 이벤트에는 commitUrl이 포함되어 내려간다.
클라이언트는 이벤트를 수신하면 해당 URL을 호출하게 되어있다.

```java
{
  "id": "123",
  "notification": {
    "groupId": "order-notification",
    "title": "신규 주문",
    "content": "배달ABCD - 김치찌개 외 2건 / 15,000원",
    "eventTypes": [
      "팝업실행"
    ],
    "soundTypes": [
      "신규 주문 알림 재생"
    ]
  },
  "data": {
    "..."
  },
  "timestamp": "2025-02-05T20:30:29.443+09:00",
  // 중요한 이벤트라면 아래 두 필드 추가
  "commitRequired": true,
  "commitUrl": "https://~~~~~"
}
```

서버는 해당 URL이 호출되면 디바이스가 이벤트를 수신한 것으로 인지하고 DB에 마킹합니다.

### 비즈니스 로직을 고려한 재전송

```java
override suspend fun getValidUncommittedEvents(
    fromCreatedAt: LocalDateTime,
    toIdNotInclude: String?,
    merchantNo: String,
    deviceId: String,
    size: Int,
): List<ServerSentEventMessage> {
    // 1. 커밋되지 않은 이벤트 목록을 조회합니다.
    val uncommittedEvents = clientEventMessagePersistentOutputPort.findUncommittedEvents(
        fromCreatedAt,
        toIdNotInclude,
        merchantNo,
        deviceId,
        size
    )

    // 2. 재발행이 가능한 이벤트만 필터링하여 반환합니다.
    return uncommittedEvents.filter { isAvailableToRepublish(it) }
}
```

수신 확인이 되지 않은 이벤트 중, 현재 주문 상태와 비교하여 불필요한 이벤트를 제외하고 전송한다.
예를 들어, 클라이언트가 신규 주문에 대한 이벤트 수신을 하지 못했지만 이미 주문이 접수된 상태라면 주문서를 다시 출력하면 안 되는 정책이 있기 때문이다. 이처럼 단순히 미수신 이벤트를 재전송하는 것이 아니라,
비즈니스 로직을 고려하여 유효한 이벤트만 재전송한다.

## 클라이언트 Polling 제거

---

이제 SSE가 안전하고 정확하게 전달될 수 있는 환경이 구축되었다.
따라서 클라이언트에 존재하는 많은 Polling 로직들을 제거할 수 있다.
클라이언트에서는 알림을 노출하기 위한 것 외에 크게 2가지의 Polling 로직이 존재했다.

1. 클라이언트의 Online 상태를 확인하기 위한 Polling
2. 클라이언트의 화면을 갱신하기 위한 Polling

### 1. 클라이언트의 Online 상태를 확인하기 위한 Polling

클라이언트는 접수할 수 있는 환경인지 확인하기 위해 주기적으로 서버의 API를 호출하여 Ping을 보내고 있었다. 이제 이 로직을 제거할 수 있다.

```kotlin
private suspend fun startHeartBeat(
    session: ServerSentEventSession,
) {
    repeat(HEARTBEAT_COUNT) {
        session.heartbeat()
        authenticationUseCase.liveOn(
            session.merchantNo,
            session.deviceId,
            session.channelType,
        )
        delay(interval)
    }
    session.close()
}
```

위와 같은 로직으로 서버 내부적으로 heartbeat을 전송하는 시점에 online 상태 정보를 갱신하도록 하였다. 이를 통해 클라이언트는 별도의 Ping API 호출 없이도 온라인 상태가 유지된다.

### 2. 클라이언트의 화면을 갱신하기 위한 Polling

모든 화면 변경 정보가 실시간 이벤트로 제공되지는 않는다. 예를 들어, 특정 데이터는 다른 도메인 팀에서 관리하며, 해당 데이터의 변경 서버가 즉시 알 수 없는 경우가 있었다. 이 때문에 클라이언트는
데이터 변경 여부를 확인하기 위해 주기적으로 API를 호출(Polling)해야 했다.
이제는 세션 연결 시 클라이언트가 Polling하던 API를 서버가 대신 호출하도록 변경했다. 그리고 해당 결과가 **갱신이 필요**한 상태라면 서버는 클라이언트에게 SSE를 전송한다.

```kotlin
override suspend fun getRequiredEvents(
    merchantNo: String,
    deviceId: String,
    version: Version?,
): List<ServerSentEventMessage> = buildList {
    // 새로운 알림이 있는지 확인하고, 있다면 이벤트를 추가합니다.
    if (notificationManagementUseCase.hasNewNotificationMessage(merchantNo)) {
        add(HasNewNotificationEventMessage())
    }

    // 앱 버전 업데이트가 필요한지 확인하고, 필요하다면 이벤트를 추가합니다.
    if (minimumVersionManagementUseCase.checkRuntimeUpdateRequired(version)) {
        add(RuntimeVersionUpdateClientEventMessage())
    }
}
```

이를 통해 **클라이언트 → 서버 → 외부 서버**를 주기적으로 호출하고 있던 것을 필요할 때만 **서버 → 외부 서버**만 호출될 수 있도록 변경하여 네트워크 비용을 크게 줄였다.
