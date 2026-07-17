# Spring Cloud Config HA 적용을 위한 커스터마이징

- 읽은 날짜: 2026-01-20
- 회사/작성 주체: NAVER
- 원문: [Spring Cloud Config HA 적용을 위한 커스터마이징](https://d2.naver.com/helloworld/6269411)

---

---
## **1. 배포 없이 프로퍼티를 바꾸고 싶다**
서비스를 운영하다 보면 애플리케이션 코드를 바꾸지 않고도 설정값만 바꾸고 싶은 상황이 자주 생긴다.
예를 들어 외부 API 호출 timeout 값을 조정하거나, 특정 기능을 켜고 끄거나, 실험용 옵션을 바꾸거나, 장애 상황에서 임시로 특정 로직을 비활성화해야 할 수 있다.
이런 값들은 코드 자체라기보다는 설정에 가깝다.
```yaml
feature:
  new-payment-enabled: false

external:
  payment-timeout: 3000

event:
  coupon-enabled: true
```
문제는 이런 설정이 애플리케이션 내부 `application.yml`에 들어 있거나, 배포 시점에 고정되는 값이면 변경이 쉽지 않다는 점이다.
설정 하나를 바꾸기 위해 다시 빌드하고, 다시 배포하고, 인스턴스를 재시작해야 한다면 운영 부담이 커진다. 특히 설정값 변경이 긴급 대응과 관련되어 있다면 재배포 시간 자체가 문제가 될 수 있다.
```text
운영 중 프로퍼티를 바꾸고 싶다.
하지만 매번 배포하고 싶지는 않다.
설정 변경을 빠르고 안전하게 반영하고 싶다.
```
이 문제를 해결하기 위해 Spring Cloud Config를 도입한다.
---
## **2. Spring Cloud Config의 기본 구조**
Spring Cloud Config는 설정을 중앙에서 관리하기 위한 도구다.
구조는 크게 세 가지로 나눌 수 있다.
```text
Config Repository
→ 설정 파일이 저장되는 저장소

Config Server
→ 저장소에서 설정을 읽어 클라이언트에게 제공하는 서버

Config Client
→ Config Server에서 설정을 받아 사용하는 애플리케이션
```
설정 파일은 보통 Git 저장소에 둔다.
예를 들어 다음과 같은 구조가 될 수 있다.
```text
config-repo
├── service-a.yml
├── service-a-prod.yml
├── service-b.yml
└── service-b-prod.yml
```
각 애플리케이션은 자기 이름과 프로파일에 맞는 설정을 Config Server에서 받아온다.
예를 들어 `service-a`가 `prod` 프로파일로 실행되면 Config Server는 `service-a.yml`과 `service-a-prod.yml`을 읽어 설정을 제공한다.
전체 흐름은 다음과 같다.
```text
애플리케이션 시작
→ Config Client가 Config Server에 설정 요청
→ Config Server가 Git 저장소에서 설정 파일 조회
→ Config Server가 설정값 반환
→ 애플리케이션이 설정값을 Environment에 등록
→ 서비스 로직에서 설정값 사용
```
이렇게 하면 애플리케이션 내부에 설정을 고정하지 않고 외부 저장소에서 관리할 수 있다.
---
## **3. 도입 기대 효과**
Spring Cloud Config를 사용하면 설정값을 중앙에서 관리할 수 있다.
기존에는 서비스마다 설정이 흩어져 있을 수 있다.
```text
service-a 서버 환경 변수
service-b application.yml
service-c 운영 서버 설정 파일
service-d 배포 스크립트 내부 값
```
이렇게 설정이 흩어져 있으면 현재 운영 환경에 어떤 설정이 적용되어 있는지 확인하기 어렵고, 변경 이력도 추적하기 어렵다.
반면 Git 기반 Config Repository를 사용하면 설정 변경 이력을 Git으로 관리할 수 있다.
```text
누가 설정을 바꿨는지 확인 가능
언제 설정을 바꿨는지 확인 가능
무엇이 변경되었는지 diff로 확인 가능
문제 발생 시 이전 커밋으로 롤백 가능
```
또 설정값을 코드와 분리할 수 있기 때문에 애플리케이션을 다시 빌드하지 않고도 환경별 설정을 다르게 줄 수 있다.
무엇보다 중요한 장점은 동적 변경이다. 설정을 바꾼 뒤 애플리케이션을 재시작하지 않고 새 값을 반영할 수 있다면, 운영 중 대응 속도가 빨라진다.
---
## **4. 하지만 기본 구조에는 SPOF 문제가 있다**
Spring Cloud Config를 도입하면 편해지는 부분도 있지만, 동시에 새로운 문제가 생긴다.
가장 큰 문제는 **Config Server가 단일 장애 지점이 될 수 있다**는 점이다.
Config Server가 모든 설정 요청의 중심에 있기 때문이다.
```text
service-a ─┐
service-b ─┼→ Config Server → Config Repository
service-c ─┘
```
이 구조에서 Config Server가 장애 나면 여러 문제가 발생할 수 있다.
```text
새로 시작하는 애플리케이션이 설정을 못 가져올 수 있다.
운영 중 설정 변경을 반영하지 못할 수 있다.
Config Client가 fail-fast로 설정되어 있으면 애플리케이션 시작이 실패할 수 있다.
설정 서버 장애가 여러 서비스 장애로 전파될 수 있다.
```
즉 원래는 설정을 편하게 관리하려고 도입했는데, Config Server가 오히려 운영 리스크가 될 수 있다.
그래서 발표의 핵심은 단순히 “Spring Cloud Config를 도입했다”가 아니라, **Spring Cloud Config를 고가용성 구조로 커스터마이징했다**는 데 있다.
---
## **5. HA가 필요한 이유**
HA는 High Availability, 즉 고가용성을 의미한다.
Spring Cloud Config를 운영 환경에서 쓰려면 Config Server가 안정적으로 살아 있어야 한다.
Config Server가 한 대만 있으면 다음과 같은 구조가 된다.
```text
Config Client들
→ Config Server 1대
→ Git 저장소
```
이 구조에서는 Config Server 한 대가 죽는 순간 설정 조회가 막힌다.
그래서 Config Server는 여러 대로 구성해야 한다.
```text
Config Client들
→ Load Balancer
→ Config Server 1
→ Config Server 2
→ Config Server 3
→ Git 저장소
```
이렇게 하면 Config Server 한 대가 장애가 나더라도 다른 인스턴스가 요청을 처리할 수 있다.
하지만 Config Server를 여러 대 띄우는 것만으로 모든 문제가 해결되는 것은 아니다.
---
## **6. Config Server 다중화만으로 부족한 이유**
Config Server 뒤에는 설정 저장소가 있다.
대부분 Git 저장소를 사용한다.
```text
Config Client
→ Config Server
→ Git Repository
```
이때 장애가 날 수 있는 지점은 Config Server만이 아니다.
```text
Config Client와 Config Server 사이 네트워크 장애
Config Server 자체 장애
Config Server와 Git 저장소 사이 네트워크 장애
Git 저장소 장애
Git 인증 실패
잘못된 설정 파일 push
설정 파일 파싱 실패
```
예를 들어 Config Server는 살아 있지만 Git 저장소와 통신하지 못할 수 있다.
또 Config Server가 여러 대 있더라도 각 인스턴스가 서로 다른 시점의 설정을 가지고 있을 수 있다.
```text
Config Server 1 → 최신 설정 반영
Config Server 2 → 이전 설정 유지
Config Server 3 → Git pull 실패
```
이런 상황에서는 클라이언트가 어떤 Config Server에 붙느냐에 따라 다른 설정을 받을 수 있다.
따라서 HA를 적용하려면 단순히 서버 수만 늘리는 게 아니라, Config Server와 Git 저장소 사이의 동기화와 장애 대응도 함께 고려해야 한다.
---
## **7. Git 저장소 접근 문제와 로컬 캐시**
Spring Cloud Config Server는 Git 저장소를 로컬에 clone해서 설정을 읽는다.
이 구조는 장점이 있다. Git 저장소에 잠깐 문제가 생겨도 이미 clone된 로컬 저장소가 있으면 기존 설정을 제공할 수 있다.
하지만 이 방식도 제대로 관리하지 않으면 문제가 된다.
예를 들어 처음 요청이 들어올 때 Git clone을 수행하면 첫 요청이 느려질 수 있다. 또한 Git 저장소 접근 정보가 잘못되어 있어도 서버 시작 시점에는 문제가 드러나지 않고, 첫 요청 때 문제가 터질 수 있다.
그래서 운영 환경에서는 서버 시작 시점에 Git 저장소를 미리 clone하는 방식이 유리하다.
```yaml
spring:
  cloud:
    config:
      server:
        git:
          clone-on-start: true
```
이렇게 하면 Config Server가 뜰 때 Git 저장소 접근 가능 여부를 미리 확인할 수 있다.
또 원격 저장소와 통신이 너무 오래 걸릴 경우를 대비해 timeout도 필요하다.
```yaml
spring:
  cloud:
    config:
      server:
        git:
          timeout: 5
```
로컬 저장소가 꼬이거나 원격과 충돌이 생겼을 때는 원격 기준으로 강제로 다시 가져오는 설정도 사용할 수 있다.
```yaml
spring:
  cloud:
    config:
      server:
        git:
          force-pull: true
```
이런 설정들은 Config Server가 Git 저장소와 안정적으로 동작하도록 돕는다.
---
## **8. 동적 프로퍼티 변경 방식**
Spring Cloud Config의 중요한 기능은 동적 프로퍼티 변경이다.
설정 저장소의 값을 바꾼 뒤 애플리케이션을 재시작하지 않고 새 값을 반영하는 것이다.
기본 흐름은 다음과 같다.
```text
1. Git 저장소의 설정 파일 수정
2. commit/push
3. Config Server가 최신 설정 조회 가능
4. Config Client 애플리케이션에 refresh 요청
5. 애플리케이션이 Config Server에서 새 설정을 다시 가져옴
6. 설정값을 사용하는 빈이 갱신됨
```
일반적으로 Spring Boot Actuator의 refresh 엔드포인트를 사용할 수 있다.
```bash
POST /actuator/refresh
```
이 요청을 보내면 애플리케이션은 Config Server에서 최신 설정을 다시 읽어온다.
이 방식은 설정 변경을 빠르게 반영할 수 있다는 장점이 있다.
---
## **9. ****`/actuator/refresh`**** 방식의 한계**
하지만 `/actuator/refresh`를 직접 호출하는 방식은 인스턴스가 적을 때만 단순하다.
운영 환경에서는 같은 서비스가 여러 인스턴스로 떠 있다.
```text
service-a-1
service-a-2
service-a-3
service-a-4
service-a-5
```
이때 설정을 바꾸려면 모든 인스턴스에 refresh 요청을 보내야 한다.
```text
service-a-1 → refresh
service-a-2 → refresh
service-a-3 → refresh
service-a-4 → refresh
service-a-5 → refresh
```
문제는 일부 인스턴스만 refresh에 성공할 수 있다는 점이다.
```text
service-a-1 → 새 설정 반영
service-a-2 → 새 설정 반영
service-a-3 → refresh 실패
service-a-4 → 새 설정 반영
service-a-5 → 이전 설정 유지
```
이 경우 같은 서비스인데 인스턴스마다 다른 설정으로 동작한다.
이건 운영 환경에서 위험하다.
예를 들어 어떤 인스턴스는 새 timeout 값을 쓰고, 어떤 인스턴스는 이전 timeout 값을 쓰면 요청 처리 결과가 달라질 수 있다.
따라서 동적 설정 변경을 안정적으로 운영하려면 단순히 refresh API를 호출하는 것만으로는 부족하다.
---
## **10. 설정 변경 이벤트 전파**
여러 인스턴스에 설정 변경을 전파하려면 이벤트 기반 구조가 필요하다.
대표적인 방식은 Spring Cloud Bus를 사용하는 것이다.
Spring Cloud Bus는 메시지 브로커를 통해 여러 애플리케이션 인스턴스에 이벤트를 전달한다.
구조는 다음과 같다.
```text
설정 변경
→ refresh 이벤트 발생
→ 메시지 브로커
→ 여러 애플리케이션 인스턴스가 이벤트 수신
→ 각 인스턴스가 Config Server에서 새 설정 조회
→ 설정 반영
```
이때 메시지 브로커로는 Kafka나 RabbitMQ 같은 시스템을 사용할 수 있다.
수동 refresh와 비교하면 다음과 같다.
```text
수동 refresh 방식
→ 인스턴스마다 직접 refresh 호출

Bus 방식
→ 한 번 이벤트 발행
→ 여러 인스턴스가 이벤트 수신
→ 각자 refresh 수행
```
이 방식은 인스턴스 수가 많을 때 훨씬 관리하기 쉽다.
하지만 이 구조에도 새로운 장애 지점이 생긴다.
```text
메시지 브로커 장애
refresh 이벤트 전파 실패
일부 인스턴스 이벤트 수신 실패
Config Server 조회 실패
```
따라서 설정 변경 이벤트를 전파하는 구조도 HA 관점에서 관리해야 한다.
---
## **11. 일부 인스턴스만 반영되는 문제**
운영에서 가장 위험한 상황 중 하나는 설정이 일부 인스턴스에만 반영되는 것이다.
예를 들어 기능 플래그를 켰다고 하자.
```yaml
feature:
  new-order-flow-enabled: true
```
그런데 일부 인스턴스는 새 설정을 받고, 일부 인스턴스는 이전 설정을 유지하면 어떻게 될까?
```text
instance-1: new-order-flow-enabled=true
instance-2: new-order-flow-enabled=true
instance-3: new-order-flow-enabled=false
instance-4: new-order-flow-enabled=true
```
사용자 요청이 어느 인스턴스로 가느냐에 따라 다른 로직이 실행될 수 있다.
이런 상태는 장애를 만들기 쉽다.
따라서 설정 변경 시스템에는 다음 기능이 필요하다.
```text
설정 변경 이벤트 발행 여부 확인
각 인스턴스의 refresh 성공 여부 확인
각 인스턴스가 사용 중인 설정 버전 확인
실패한 인스턴스 재시도
문제 발생 시 롤백
```
즉 동적 설정 변경은 단순히 “값을 바꾼다”가 아니라, “모든 실행 인스턴스에 같은 값이 안정적으로 반영된다”까지 보장해야 한다.
---
## **12. 커스터마이징의 핵심 방향**
발표 제목의 “커스터마이징”은 기본 Spring Cloud Config 기능만으로 부족한 운영 요구사항을 보완했다는 의미로 볼 수 있다.
운영 환경에서 필요한 커스터마이징 방향은 다음과 같다.
```text
Config Server를 여러 대 구성한다.
Config Server가 Git 저장소 장애에 대비하게 한다.
Config Client가 timeout과 retry를 갖도록 한다.
설정 변경 이벤트를 여러 인스턴스에 전파한다.
일부 인스턴스 반영 실패를 감지한다.
설정 변경 이력과 버전을 추적한다.
필요하면 롤백할 수 있어야 한다.
민감한 설정값은 안전하게 보호한다.
```
즉 Spring Cloud Config의 기본 기능을 그대로 쓰는 것에서 끝나는 게 아니라, 서비스 안정성을 위해 주변 운영 구조를 함께 설계하는 것이 핵심이다.
---
## **13. 설정 변경은 배포만큼 조심해야 한다**
설정 변경은 코드 배포보다 가벼워 보일 수 있지만, 실제로는 서비스 동작을 직접 바꾼다.
예를 들어 timeout을 잘못 줄이면 외부 API 호출이 대량으로 실패할 수 있다.
```yaml
payment:
  timeout: 100
```
기능 플래그를 잘못 켜면 아직 검증되지 않은 기능이 운영에 노출될 수 있다.
```yaml
feature:
  beta-payment-enabled: true
```
인증 관련 값을 잘못 바꾸면 전체 로그인 장애가 날 수도 있다.
```yaml
auth:
  token-expiration-seconds: 10
```
따라서 설정 변경은 단순한 파일 수정이 아니라 운영 이벤트로 봐야 한다.
필요한 관리 요소는 다음과 같다.
```text
변경자 기록
변경 시간 기록
변경 내용 diff 확인
승인 절차
반영 대상 확인
반영 결과 확인
실패 시 재시도
문제 시 롤백
```
Spring Cloud Config를 도입할 때도 이런 운영 절차를 함께 만들어야 안정적으로 사용할 수 있다.
---
## **14. 전체 흐름 정리**
서비스 운영 중에는 설정값을 빠르게 바꿔야 하는 경우가 있다. 기존 방식처럼 설정 변경 때마다 애플리케이션을 재배포하면 시간이 오래 걸리고 운영 부담이 크다.
이를 해결하기 위해 Spring Cloud Config를 도입한다. 설정은 Git 저장소에 두고, Config Server가 설정을 읽어 제공한다. 애플리케이션은 Config Client를 통해 설정을 받아 사용한다.
이 구조를 사용하면 설정을 중앙에서 관리할 수 있고, Git을 통해 변경 이력을 남길 수 있으며, 필요하면 런타임에 설정을 새로고침할 수 있다.
하지만 Config Server가 모든 설정 요청의 중심이 되면서 새로운 장애 지점이 된다. Config Server가 죽거나 Git 저장소와 통신하지 못하면 설정 조회가 실패할 수 있다. 따라서 Config Server를 다중화하고, Git 저장소 접근 실패에 대비해야 한다.
또한 동적 설정 변경을 위해 `/actuator/refresh`를 사용할 수 있지만, 인스턴스가 많아지면 모든 인스턴스에 안정적으로 refresh를 전파하는 문제가 생긴다. 일부 인스턴스만 새 설정을 반영하면 같은 서비스가 서로 다른 설정으로 동작할 수 있다.
이를 해결하기 위해 메시지 기반 이벤트 전파 구조를 사용할 수 있고, 각 인스턴스의 반영 성공 여부를 확인하는 체계가 필요하다.
결국 Spring Cloud Config를 실무에서 안정적으로 사용하려면 Config Server HA, Git 저장소 장애 대비, Config Client timeout/retry, 설정 변경 이벤트 전파, 설정 반영 상태 확인, 롤백 전략까지 함께 설계해야 한다.
---
# **마지막 핵심 정리**
이 발표의 핵심은 **Spring Cloud Config를 단순히 도입하는 것이 아니라, 운영 환경에서 안정적으로 사용할 수 있도록 HA 구조로 커스터마이징하는 것**이다.
Spring Cloud Config를 사용하면 설정을 Git 저장소에 모아두고, Config Server를 통해 여러 애플리케이션에 제공할 수 있다. 또한 설정 변경 후 refresh를 통해 애플리케이션 재배포 없이 새 설정을 반영할 수 있다.
하지만 기본 구조에는 위험이 있다.
```text
Config Server가 SPOF가 될 수 있다.
Git 저장소 장애가 설정 조회 실패로 이어질 수 있다.
수동 refresh는 인스턴스가 많을 때 관리하기 어렵다.
일부 인스턴스만 설정이 반영될 수 있다.
설정 변경 실패 여부를 추적하기 어렵다.
```
그래서 HA 적용을 위해서는 다음 요소가 필요하다.
```text
Config Server 다중화
Git 저장소 접근 안정화
로컬 clone/cache 활용
Config Client timeout/retry 설정
Spring Cloud Bus 같은 이벤트 전파 구조
인스턴스별 설정 반영 상태 확인
설정 변경 이력 관리
롤백 전략
```
한 줄로 정리하면 다음과 같다.
**Spring Cloud Config는 배포 없이 프로퍼티를 바꿀 수 있게 해주지만, 실무에서 안정적으로 쓰려면 Config Server와 설정 변경 전파 구조 전체를 고가용성으로 설계해야 한다.**

