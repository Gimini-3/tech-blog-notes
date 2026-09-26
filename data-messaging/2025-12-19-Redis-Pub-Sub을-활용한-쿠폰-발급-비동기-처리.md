# Redis Pub/Sub을 활용한 쿠폰 발급 비동기 처리

- 읽은 날짜: 2025-12-19
- 회사/작성 주체: 올리브영
- 원문: [Redis Pub/Sub을 활용한 쿠폰 발급 비동기 처리](https://oliveyoung.tech/2023-08-07/async-process-of-coupon-issuance-using-redis/)

---

올영 세일 기간에는 0시, 12시 선착순 쿠폰 다운로드 이벤트를 진행하고 있다.

→ 트래픽 몰림 → 대기시간 증가 → 좋지 못한 사용자 경험 제공

Redis Pub/Sub을 활용하여 쿠폰 발급을 비동기로 처리하여 개선

---

## 기존 쿠폰 발급 프로세스

<img width="1040" height="930" alt="image" src="https://github.com/user-attachments/assets/7529304f-30ce-4926-bec0-76a09c23a248" />

위의 모든 과정이 Sync(동기식 처리)로 처리되며, 트래픽이 몰리는 경우 쿠폰 발급 과정에서 DB Connection 등의 자원을 많이 점유하게 된다.

높은 트래픽 환경에서도 안정적으로 서비스를 제공하기 위해 접속한 사용자가 순서대로 쿠폰 발급을 시도할 수 있도록 제어하고 있으며, 사용자는 본인의 순서가 돌아올 때까지 긴 시간을 대기해야 한다.

## 프로세스를 어떻게 개선할까?

<img width="960" height="564" alt="image" src="https://github.com/user-attachments/assets/9478a0ac-a681-44b6-be08-2397fbefb301" />

쿠폰 발급에 필요한 유효셩 검사까지 기본 로직을 사용하고, 발급 처리는 신규 시스템인 쿠폰 발급 Worker로 위임했다.

올리브영 온라인몰 ↔ 쿠폰 발급 Worker 사이의 데이터 전달은 Redis Pub/Sub과 Redis의 List 자료구조를 활용했다.

---

## Redis Pub/Sub이란?

<img width="960" height="546" alt="image" src="https://github.com/user-attachments/assets/a5f873e1-8803-431b-86f2-5b0c709eed57" />

특정 주제(Topic/Channel)를 구독한 구독자에게 메시지를 발행하는 통신 방법

publisher가 메시지를 publish하면 해당 Topic을 구독하고 있는 다수의 subscriber가 메시지를 받을 수 있는 구조이다.

Redis는 In-Memory 기반이기 때문에 네트워크 통신이 필요한 웹소켓을 이용하는 것보다 매우 빠르게 메시지를 주고받을 수 있다.

다만, 구조가 매우 단순하게 되어 있어서 publish 한 메시지는 따로 보관되지 않으며 subsriber가 수신했는지 확인하지 않는다.

메시지를 빠르게 보내고 싶을 때, 전송한 메시지를 저장하거나 수신 확인이 필요하지 않으며 100% 전송 보장이 되지 않아도 문제없는 케이스에 사용하기 좋다.

- 특정한 작업을 반복 수행하는 시스템에 비동기적으로 작업을 보내 처리하도록 하거나 앱 사용자에게 푸시를 발송하도록 하는 과정을 publ/sub원리로 실행할 수 있다.

---

## Redis Pub/Sub을 쿠폰 발급에 적용

<img width="1380" height="462" alt="image" src="https://github.com/user-attachments/assets/1f841d84-e80c-4e35-9cd1-283573b658c0" />

‘쿠폰발급’이라는 Topic에 쿠폰 발급 데이터 publish→ 쿠폰발급 Topic을 구독하고 있는쿠폰 발급 Worker

→ 전달받은 쿠폰 발급 데이터를 Main DB에 INSERT하여 발급 처리를 진행

### Redis Pub/Sub 적용 후 이슈 첫 번째: 쿠폰 과발급 현상

<img width="2240" height="760" alt="image" src="https://github.com/user-attachments/assets/708959c9-93d9-4e33-8eb1-e6d018ddfb77" />

단일 쿠폰 발급 Worker로 발급 테스트를 진행했을 때는 다중 Worker로 스트레스 테스트 중에 발생했다.

- 스트레스 테스트: 시스템이 감당할 수 있는 최대 한계 상황을 인위적으로 만들어보는 성능 테스트

쿠폰 발급 Worker는 실제 운영환경에서 여러 대가 실행되며, 각 Worker는 쿠폰 발급 정보를 전달해 주는 하나의 Topic을 구독하고 있다.

문제점: message를 각각 받아 쿠폰 발급 처리 로직을 실행하게 되어, 1건의 쿠폰 발급 데이터가 publish 될 때마다 쿠폰 발급은 4회 실행된다.

해결 방법: message를 publish 할 때 여러 개의 Worker 중 1개의 Worker를 고르고, 해당 Worker만 message를 받아 쿠폰 발급 처리를 할 수 있도록 변경했다.

### Redis Pub/Sub 적용 후 이슈 두 번째: 쿠폰 미발급 현상

<img width="960" height="326" alt="image" src="https://github.com/user-attachments/assets/56b57777-a0cb-4533-b305-c9a86e9e75c1" />

미발급 현상이 발생

문제점: Redis Pub/Sub의 특징 중 하나인 ‘100% 전송 보장이 되지 않아도 문제없는 케이스에 사용하기 좋다.’를 간과한 것이 문제라고 판단

해결 방법: 쿠폰 발급 데이터의 유실을 막기 위해 Redis에서 제공하는 List 자료구조를 활용했다.

- 먼저 기존처럼 올리브영 온라인물에서 여러 개의 Worker 중 1개의 Worker를 골라 일련번호를 획득한다.
- 그리고 list 자료구조로 되어 있는 ‘쿠폰발급 저장소’에 쿠폰 발급 데이터를 RPush한다.
- 쿠폰발급 Worker는 본인의 일련번호로 채번된 쿠폰발급 저장소의 크기를 주기적으로 체크하면서, 0보다 크면 LPop 하여 쿠폰 발급 처리를 한다.

이렇게 하면 쿠폰 발급 데이터의 유실을 막으면서 지정된 1개의 Worker만 쿠폰 발급 처리를 할 수 있게 된다.

---

## 최종으로 적용된 쿠폰 발급 로직

<img width="960" height="442" alt="image" src="https://github.com/user-attachments/assets/3deb90e6-b9c0-41b2-8ab3-064aa1ffa164" />

1. 쿠폰 발급 Worker가 구동되면 '쿠폰발급' 이라는 Topic에 대한 '일련번호'가 생성됩니다. Worker마다 각각의 '일련번호'가 할당되어 있다.
2. 각 Worker들은 '쿠폰발급'이라는 Topic을 구독한다.
3. 주기적으로 '쿠폰 발급 저장소' List의 크기가 0보다 큰지 확인하고, 0보다 크면 List에서 데이터를 Pop 하여 쿠폰을 발급하는 프로세스가 실행된다.

- 위의 1~3번 과정은 쿠폰 발급을 위해 기본적으로 미리 실행되어 있어야 한다.

아래 4~8번 과정은 실제 사용자가 쿠폰 발급을 요청했을 때 어떤 흐름으로 동작하는지 설명한다.

1. 올리브영 온라인몰을 통해 고객이 쿠폰 발급을 요청한다.
2. 올리브영 온라인몰에서는 각 Worker들이 가지고 있는 '일련번호' 중 1개를 골라서 '쿠폰발급' 이라는 Topic에 publish 한다.
3. 고객의 쿠폰 발급 데이터를 각 '일련번호'마다 할당된 '쿠폰 발급 저장소' list에 RPush 한다.
4. 6번 과정으로 '쿠폰 발급 저장소'의 크기가 0보다 커졌으므로 쿠폰발급 Worker는 해당 list를 LPop하여 쿠폰 발급 데이터를 가져온다.
5. 7번에서 가져온 쿠폰 발급 데이터를 온라인몰 Main DB에 INSERT 하여 발급 처리를 완료한다.

**결론적으로, 여러 Worker 중 어떤 Worker를 통해 쿠폰을 발급할지 결정지어 주는 역할을 Redis Pub/Sub 기능을 활용하여 구현(1개의 '일련번호' 선택)하였고, 실제 쿠폰 발급에 필요한 데이터는 각 Worker 별로 지정된 '쿠폰 발급 저장소'에 저장되었다가 쿠폰 발급이 진행된다.**

## 결과

기존 대비 분당 최대 쿠폰 발급량이 2.2배에 달하는 속도 개선이 있었다고 한다.

사용자의 쿠폰 발급 대기 시간도 많이 줄어들어 모든 쿠폰이 1~2분 만에 소진되었고, 덕분에 사용자 경험도 좋아지고 매출 증가에도 좋은 영향을 끼쳤다고 한다.

## 느낀점

Redis Pub/Sub에 대해서 공부하고 싶어서 찾아봤는데 Redis Pub/Sub의 동작원리 뿐만 아니라 문제점과 어떻게 해결하는가 까지 공부 할 수 있어서 좋았습니다. 이번 프로젝트에서 Redis Pub/Sub을 써서 개발해보고 싶습니다.
