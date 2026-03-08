# WIL: 이번주 공부 (3월 1주차)

> TL;WR Prometheus + Loki + Grafana 모니터링 스택을 Docker Compose와 AKS에 도입하면서, Spring Security child context 문제, ThreadLocal 타이밍 문제, 이벤트 기반 메트릭 분리까지 — 보이지 않던 서비스에 눈을 달아주는 한 주였다.

### 좋았던 점 & 배웠던 점

#### 1. Management 포트 분리의 함정

Spring Boot Actuator를 별도 포트에서 운영하면 방화벽 레벨에서 차단할 수 있어서 깔끔하다고 생각했다. `management.server.port`를 설정했더니 바로 에러가 터졌다.

```
java.lang.IllegalArgumentException: Failed to find servlet [dispatcherServletRegistration] in the servlet context
```

`management.server.port`를 분리하면 Spring Boot가 **별도의 child application context**를 생성해 management 전용 서버를 띄운다. 이 child context에는 `DispatcherServlet`이 없다. 그런데 Spring Security 6.x의 `MvcRequestMatcher`는 내부적으로 `DispatcherServlet`을 참조하므로, child context에서는 문자열 기반 matcher 전체가 실패한다.

`@Order(0)` management 전용 `SecurityFilterChain`을 `EndpointRequest`로 구성해서 Actuator는 해결했지만, 이번엔 일반 API 요청에서도 같은 에러가 발생했다. `MvcRequestMatcher` 기반 체인 자체가 child context에서 동작할 수 없는 구조였다.

결국 `management.server.port` 설정을 완전히 제거하고, Actuator를 메인 포트에서 운영하기로 결정했다. 보안은 Nginx에서 `/actuator/**` 경로를 차단하는 방식으로 대체했다.

```nginx
location /actuator {
    deny all;
    return 403;
}
```

Prometheus는 Docker 내부 네트워크에서 직접 스크래핑하기 때문에 Nginx를 거치지 않아서 이 방식으로 충분했다. 복잡한 workaround보다 근본 원인을 제거하는 게 낫다는 걸 다시 한 번 체감했다.

#### 2. Prometheus + Loki + Grafana 스택 구성

```
Spring App ──pull──► Prometheus
                          │
로그 파일 ──push──► Loki ──►  Grafana
(Promtail 경유)
```

메트릭은 Prometheus가 `/actuator/prometheus`를 15초마다 스크래핑하고, 로그는 Promtail이 파일을 감시해서 Loki에 push한다.

로그 수집 범위를 전체 레벨이 아닌 WARN/ERROR만으로 제한했다. 대시보드에서 확인할 로그는 경고/에러가 대부분이고, INFO는 99%가 정상 요청 로그라 Loki에 넣을 가치가 낮다고 판단했다. 디스크 비용과 Loki 쿼리 성능을 고려하면 수집 범위를 처음부터 제한하는 게 맞다.

개발 서버(Docker Compose)와 운영 서버(AKS)의 구성이 꽤 달랐다.

| 항목         | 개발 (Docker Compose)      | 운영 (AKS)                      |
| ---------- | ------------------------ | ----------------------------- |
| 설치 방식      | docker-compose           | Helm 차트                       |
| 로그 수집      | 파일 (warn.log, error.log) | 컨테이너 stdout/stderr            |
| 로그 필터      | `filename=~".*error.*"`  | `namespace`, `app` 라벨         |
| Grafana 접근 | SSH 터널                   | SSH 터널 + kubectl port-forward |

K8s에서는 Promtail이 파일이 아닌 컨테이너 로그를 수집하므로 LogQL 쿼리도 달라진다. 개발 환경에서 검증한 쿼리가 운영 환경에서 그대로 쓰일 거라고 기대했다가 당황했다.

#### 3. Promtail multiline + timestamp 스테이지

logback JSON 포맷터를 `prettyPrint: true`로 설정하면 로그 1건이 여러 줄로 출력된다. Promtail은 기본적으로 줄 단위로 읽으므로 Loki에서 1건이 수십 개 엔트리로 쪼개진다.

`multiline` stage로 `{`로 시작하는 줄을 새 로그의 시작으로 인식하게 설정해서 해결했다. prettyPrint JSON에서 `{`가 줄 맨 앞에 나타나는 건 최상위 객체 시작뿐이라 이 방식이 딱 맞았다.

그리고 `json` stage는 JSON 필드를 추출만 한다. 추출한 `timestamp`를 Loki 엔트리 타임스탬프로 반영하려면 별도의 `timestamp` stage가 필요하다. 이걸 빠뜨리면 Loki에 **수집 시점**이 기록되어서 실제 로그 발생 시각과 대시보드 시간축이 틀어진다. PR 리뷰에서 지적받고 나서야 알았는데, 처음부터 알았다면 훨씬 좋았을 것 같다.

Promtail의 timestamp `format`은 Go의 reference time layout을 사용한다. logback `yyyy-MM-dd'T'HH:mm:ss.SSSX`는 Go layout `2006-01-02T15:04:05.000-07`에 대응한다. 파이썬/Java 날짜 포맷과 달라서 처음엔 낯설었다.

#### 4. 비즈니스 로직에서 메트릭 수집을 분리하다

로그인 횟수를 측정하려고 서비스 클래스에 `MeterRegistry`를 직접 주입했다. 동작은 했지만 비즈니스 로직과 메트릭 수집이 섞이는 게 계속 거슬렸다.

프로젝트에서 이미 알림 기능에 `@TransactionalEventListener` 패턴을 쓰고 있었다. 같은 방식을 메트릭에 적용했다.

```
UserAppService → LoginEvent 발행
                     ↓ (AFTER_COMMIT)
MonitoringMetricsEventListener → Counter.increment()
```

`AFTER_COMMIT`을 선택한 이유가 있다. 로그인 트랜잭션이 롤백되는 경우, `AFTER_COMMIT`이 아니면 실제 로그인은 실패했는데 카운터가 올라갈 수 있다. 커밋 성공 이후에만 메트릭을 기록하면 롤백된 요청이 집계에 포함되지 않는다.

서비스 클래스에서 `MeterRegistry` 의존을 완전히 제거할 수 있었고, 메트릭 수집 방식이 바뀌어도 비즈니스 코드를 수정할 필요가 없어졌다.

#### 5. HTTP 메트릭에 테넌트 태그를 추가하면서 만난 ThreadLocal 타이밍 문제

`http.server.requests` 메트릭에 테넌트 태그를 추가하고 싶었다. JWT 필터에서 테넌트 정보를 `ThreadLocal`에 저장하고 있으니, 거기서 읽으면 되겠다고 생각했다.

결과는 항상 `"unknown"`. 왜?

```
JwtAuthenticationFilter    : ThreadLocal.set(tenant)
인터셉터 afterCompletion() : ThreadLocal.clear()   ← 여기서 비워짐
ServerHttpObservationFilter.stop()                 ← 메트릭 기록
  → ServerRequestObservationConvention 호출
      → ThreadLocal.get() = null                  ← 이미 비워진 상태
```

Micrometer의 `Observation.stop()`은 필터 체인 완료 이후에 호출된다. 인터셉터의 `afterCompletion()`에서 ThreadLocal을 정리하고 나면 메트릭 기록 시점엔 이미 값이 없다.

`HttpServletRequest`의 attribute는 응답이 완료될 때까지 유지된다는 걸 이용했다. JWT 필터에서 ThreadLocal에 저장할 때 `request.setAttribute("tenant", value)`도 함께 하고, `ServerRequestObservationConvention`에서는 request attribute를 읽도록 했다.

`@Component`로 등록한 커스텀 `ServerRequestObservationConvention`을 Spring Boot 3.x가 자동으로 `ServerHttpObservationFilter`에 주입해준다는 것도 이번에 처음 알았다.

테넌트 태그를 추가하면 Prometheus 시계열이 테넌트 수만큼 증가한다. 수십 개 테넌트 규모에서는 실제 시계열이 5만\~10만 수준으로 예상되고, Prometheus는 수백만까지 처리 가능하니 당장은 문제없지만 나중에 규모가 커지면 카디널리티를 신경써야 할 것 같다.

#### 6. 배포 인프라 잡일들

배포하면서 소소하게 막혔던 것들:

* `wget --spider`는 HEAD 요청이다. Spring Security 필터에서 GET/POST만 인증 제외 처리가 되어 있으면 헬스체크가 401로 실패한다. `-O /dev/null`을 쓰면 GET 요청이 된다.
* Docker 볼륨 마운트 경로가 실제 컨테이너와 Promtail 설정에서 달랐다. 심볼릭 링크로 해결했는데, Docker가 마운트 시 심볼릭 링크를 따라가서 가능한 방법이다. 단, Docker가 빈 디렉토리를 먼저 생성하는 경우가 있으므로 순서를 주의해야 한다.
* AKS에서 컨테이너 이름을 확인하지 않고 임의로 가정해서 작업했다가 실패했다. `kubectl patch`보다 yaml 파일 직접 수정 후 `kubectl apply`가 선언적 관리에 더 맞는다는 것도 이번에 정리됐다.
* Loki 기동 직후 Promtail에서 502가 발생했는데, Loki가 완전히 뜨기 전이라 그랬다. Promtail이 자동 재시도해서 수 분 뒤 204(성공)로 전환됐다. 초기 502는 정상이다.

***

### 아쉬운 점

#### 1. Promtail timestamp 스테이지를 처음부터 빠뜨렸다

`json` stage가 timestamp 필드를 추출하는 것과, 그 값을 Loki 엔트리 타임스탬프로 반영하는 것이 별개라는 걸 몰랐다. PR 리뷰에서 지적받고 수정했는데, 처음 설계할 때 알았다면 불필요한 커밋을 줄일 수 있었을 것이다.

#### 2. ThreadLocal 타이밍 문제를 나중에야 발견했다

메트릭 태그가 계속 `"unknown"`으로 찍혀서 원인을 파악하는 데 시간이 걸렸다. Spring 필터/인터셉터 실행 순서와 Micrometer Observation 라이프사이클을 미리 이해하고 있었다면 바로 request attribute 방식을 떠올렸을 것이다.

#### 3. 배포 전 환경 사전 확인 부족

컨테이너 이름, 볼륨 경로, 네트워크 이름 등을 코드 작성 전에 확인하지 않고 추정으로 작업해서 불필요한 트러블슈팅을 반복했다. 인프라 작업은 실제 환경을 먼저 확인하고 시작해야 한다.

#### 4. 롤백 및 장애 시나리오 테스트 미흡

모니터링 스택을 구성했지만, "실제로 에러가 발생했을 때 Grafana에서 어떻게 보이는가"를 직접 확인하지 않았다. 의도적으로 에러를 발생시켜 알람이 울리는지, 로그가 Loki에 잘 잡히는지 검증해볼 필요가 있다.

***

### 종합 정리

이번 주는 "보이지 않으면 고칠 수 없다"를 실감한 한 주였다.

장애가 터지고 나서야 로그를 뒤지던 방식에서, 메트릭과 로그를 실시간으로 볼 수 있는 환경이 갖춰졌다. 스택을 구성하는 과정이 생각보다 간단하지 않았는데, management 포트 분리 문제, ThreadLocal 타이밍, Promtail pipeline 설정까지 모두 하나씩 부딪히면서 배웠다.

특히 인상적이었던 건 ThreadLocal 타이밍 문제였다. "요청 처리 중에는 ThreadLocal이 살아있다"는 막연한 가정이 틀렸다. 인터셉터, 필터, Observation 각각의 라이프사이클을 이해하지 않으면 놓칠 수밖에 없는 문제였다. 이번에 확실히 정리됐다.

다음 단계는 구성한 대시보드를 실제로 활용하는 것이다. 이상 감지 알람 설정, 의미 있는 비즈니스 메트릭 추가, 장애 시나리오 테스트가 남아있다. 눈을 달았으니 이제 제대로 보는 법을 익혀야 한다.
