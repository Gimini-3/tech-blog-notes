# Ingress Nginx Controller의 Prometheus Metric 병목 현상: 원인 분석과 해결 (1부)

- 읽은 날짜: 2026-02-04
- 회사/작성 주체: Kakao
- 원문: [Ingress Nginx Controller의 Prometheus Metric 병목 현상: 원인 분석과 해결 (1부)](https://tech.kakao.com/posts/683)

---

## 1. 전체 구조

Ingress Nginx Controller Pod 안에는 크게 두 역할이 같이 들어있다.
- **Nginx(worker/master)**: 실제 HTTP 요청을 처리
- **Controller(Go 프로세스)**: 설정 동기화 + **메트릭 수집/가공 + /metrics로 노출**
메트릭 흐름
1. Nginx worker가 요청을 처리하면서 메트릭 데이터를 만든다
2. 루아 스크립트가 그 메트릭을 Unix socket(`/tmp/nginx/prometheus-nginx.socket`)으로 Controller에 보낸다
3. Controller는 소켓을 `Accept()`해서 연결을 받고, 연결마다 goroutine을 띄워 메트릭을 파싱/집계한다
4. Prometheus는 주기적으로 Controller의 `/metrics` HTTP endpoint를 pull 방식으로 스크랩한다
5. Grafana는 Prometheus에 쌓인 시계열을 그린다
---
### 2. 관측된 증상
글에서는 문제 상황이 트래픽 처리 문제(Nginx 문제)가 아니라 메트릭 처리 문제로 보였다고 한다.
- **Request는 정상 처리 중인데 Grafana가 No Data**
	- 실제 트래픽이 살아있는데 지표만 비면, 요청 처리보다 수집/노출 경로를 먼저 의심하게 된다.
- **CPU는 여유인데 Memory, open fds, goroutine만 증가**
	- CPU가 바쁘면 계산/처리가 많다는 뜻인데, 여기서는 CPU가 안 뛰었다.
	- 대신 **대기 중인 작업(goroutine)과 연결(socket)이 쌓이면** CPU는 조용한데 메모리/FD만 계속 늘 수 있다.
---
### 3. 1차 원인: Controller가 소켓 연결마다 goroutine을 무제한으로 늘리는 구조
Controller는 소켓을 Listen 하고, 연결이 들어올 때마다:
- `Accept()`
- **연결마다 goroutine 생성**
- `handleMessages()`로 처리
여기서 중요한 점은, 트래픽이 커지면 소켓으로 들어오는 메트릭도 늘고, 그만큼 goroutine도 늘 수 있는데, **처리 속도가 입력 속도를 못 따라가면 goroutine이 계속 누적된다**.
그래서:
- goroutine 수 증가
- 처리 대기 중인 메트릭 객체 증가
- open fds(소켓 연결) 증가
- 메모리 증가
---
### 4) 처리 속도가 느려진 이유: Prometheus `summary` 타입에서 Mutex 경합이 터짐
프로파일(goroutine profile)로 확인했더니, 많은 goroutine이 아래 상태였다.
- `summary.Observe()` 내부의 `Mutex Lock`을 잡으려다가 대기(Blocking)
즉, 병목의 본체는 Controller의 메트릭 처리 함수(handleMessage)가 아니라, 그 안에서 호출되는:
- `latencyMetric.Observe(stats.Latency)`
- (이게 **Prometheus client_golang의 summary** 구현 내부로 들어가서)
- 버퍼에 값을 append하기 위해 **Mutex를 잡는 구간에서 경합**이 폭발
정리하면:
- upstream latency를 summary로 기록하는 과정은 **동시성 + 높은 입력량** 상황에서
- **단일 잠금(Mutex) 구간을 경쟁하는 goroutine이 폭증**
- 그래서 observe가 느려지고, handleMessages가 밀리고, goroutine이 계속 쌓인다.
여기서 CPU가 안 치솟는 이유
- goroutine들이 계산을 많이 하는 게 아니라 **락을 기다리며 멈춰있는 상태** 가 많으면
	- CPU는 상대적으로 조용하고
	- 메모리/FD만 쌓인다.
---
### 5. No Data는 메트릭이 안 쌓여서가 아니라, Prometheus가 못 가져가서 생김
- Prometheus는 **Pull 방식**
- 대상(Controller)은 시간별 기록을 저장하는 게 아니라, **현재 시점의 최신 메트릭 상태**를 `/metrics` 요청이 오면 그때 만들어서 내보낸다.
그런데 `/metrics` 응답을 만드는 과정에서도 summary는 내부적으로:
- `summary.Write()`에서 **또 같은 Mutex(버퍼 관련 Lock)를 잡아야** 한다.
- 이미 Observe 쪽에서 Lock 경합이 심한 상태라면 Write도 오래 대기한다.
- 결과적으로 `/metrics` HTTP 응답이 늦어지고,
- Prometheus는 스크랩 타임아웃으로 실패하며,
- 그 시점 데이터를 Prometheus가 못 적재해서 Grafana는 **No Data**로 보이게 된다.
즉:
- Controller가 메트릭을 못 만들었다가 아니라
- Prometheus가 메트릭을 가져오는 요청이 타임아웃으로 실패했다
- 그래서 “No Data”가 찍힌 것이다.
---
### 6. 문제 상황 요약
대용량 트래픽에서 ingress-nginx-controller가 소켓으로 받은 메트릭 중 `upstream latency(summary)`를 기록/노출하는 과정에서 Mutex 경합이 폭발했고, 그 여파로 goroutine·socket·메모리가 누적되면서 `/metrics` 응답까지 지연되어 Prometheus 스크랩이 타임아웃 → Grafana No Data가 발생했다.
---
### 7. 해결책
#### 방법 1) Ingress 리소스(Controller/Nginx) Scale-out
- Pod/인스턴스를 늘리면 **한 Pod가 처리해야 하는 요청/메트릭 입력량이 줄어든다.**
- 그러면 summary lock 경합이 발생하더라도 한 Pod 안에서의 경쟁 규모가 작아져 병목이 완화된다.
- 운영적으로는 가장 간단하지만 비용이 증가한다.
#### 방법 2) 병목 메트릭(upstream latency summary) 수집 비활성화
- 코드 상 `exclude-socket-metrics`에 `ingress_upstream_latency_seconds`를 넣으면 summary metric이 nil(아무것도 가리키지 않는 값)이 되어 수집을 건너뛴다.
- 그러면 가장 무거운 경로(summary Observe/Write)가 빠져서 병목 자체가 사라진다.
- 다만 조직 공용 헬름 차트를 건드려야 하고, 다른 서비스의 모니터링 요구에 영향을 줄 수 있어 운영 리스크가 있다.
글에서는 최종적으로 운영 리스크(공용 차트 영향, 조직별 분기 운영 부담) 때문에 **Scale-out**을 선택했다.
