---
hidden: true
tags:
  - wil
  - monitoring
  - grafana
  - loki
---

# 로키 그라파나 구축 여정기

> TL;DR: Prometheus + Loki + Grafana 스택을 Docker Compose와 AKS에 도입하면서 management 포트 분리의 함정, 이벤트 기반 메트릭 분리, ThreadLocal 타이밍 문제까지 — 삽질의 기록을 남깁니다.

### 서론 : 이 글을 작성하게 된 이유

서비스가 돌아가고 있는데, 지금 이 순간 어떤 상태인지 확인할 방법이 없었습니다.

에러가 발생했을 때 제일 먼저 한 일은 서버에 SSH로 접속해서 로그 파일을 뒤지는 것이었습니다. 서버가 느려졌다는 보고를 받아도 JVM이 어떤 상태인지, DB 커넥션이 얼마나 열려 있는지 확인할 방법이 없었습니다. 장애가 터지고 나서야 원인을 추적하는 방식으로는 한계가 분명했습니다.

그래서 이번에 모니터링 시스템을 도입하기로 했습니다. Prometheus로 메트릭을 수집하고, Loki로 로그를 수집하고, Grafana로 시각화하는 구성입니다. 개발 서버는 Docker Compose로, 운영 서버는 AKS(Azure Kubernetes Service) + Helm으로 각각 구성했습니다.

글은 다음과 같이 구성했습니다.

* Management 포트를 분리하다가 Spring Security와 충돌한 이야기
* Prometheus + Loki + Grafana 스택을 구성한 방법
* 비즈니스 로직에서 메트릭 수집을 분리한 이야기
* HTTP 메트릭에 테넌트 태그를 추가하면서 ThreadLocal 타이밍 문제를 만난 이야기

"왜 이렇게 판단했고, 어디서 막혔고, 어떻게 풀었는지" 그 과정을 최대한 공유하고자 합니다.

**환경 정보**

* Spring Boot 3.x / Java 17
* Micrometer + Spring Boot Actuator
* Prometheus, Loki, Grafana, Promtail
* Docker Compose (개발) / AKS + Helm (운영)

***

### 1장. Management 포트 분리의 함정

#### 처음에 왜 포트를 분리하려 했나

Spring Boot Actuator는 `/actuator/**` 경로로 메트릭, 헬스체크 등을 노출합니다. 이 엔드포인트는 외부에 노출되면 안 되는데, 메인 포트(예: 8080)를 그대로 쓰면 Nginx에서 경로를 차단해야 합니다.

`management.server.port`를 별도로 설정하면 Actuator가 다른 포트에서 뜨기 때문에, 방화벽 레벨에서 해당 포트만 닫으면 된다는 장점이 있습니다. 처음에는 이 방식이 깔끔하다고 생각했습니다.

```yaml
management:
  server:
    port: 8099  # 메인 포트(8080)와 분리
```

그런데 설정 직후부터 에러가 발생했습니다.

#### 문제: DispatcherServlet이 없다

```
java.lang.IllegalArgumentException: Failed to find servlet [dispatcherServletRegistration] in the servlet context
    org.springframework.security.config.annotation.web.AbstractRequestMatcherRegistry
        $DispatcherServletDelegatingRequestMatcher.matcher(...)
```

`management.server.port`를 분리하면 Spring Boot는 **별도의 child application context**를 생성해 management 전용 웹 서버를 띄웁니다. 이 child context에는 `DispatcherServlet`이 존재하지 않습니다.

```
메인 컨텍스트 (8080)
  └── DispatcherServlet ✅
  └── SecurityFilterChain (MvcRequestMatcher 기반)

Management 컨텍스트 (8099)  ← child context
  └── DispatcherServlet ❌ (없음)
  └── 같은 SecurityFilterChain이 적용됨 → 에러 발생
```

Spring Security 6.x의 `MvcRequestMatcher`는 내부적으로 `DispatcherServlet`을 참조합니다. management child context에서 이 matcher가 동작하려 하면 `DispatcherServlet`을 찾지 못해 에러가 납니다.

#### 첫 번째 시도: @Order(0) management 전용 SecurityFilterChain

`EndpointRequest`는 Actuator 전용 matcher로 `DispatcherServlet`에 의존하지 않습니다. management 포트 전용 체인을 최우선 순위로 등록하면 되지 않을까?

```java
@Bean
@Order(0)
public SecurityFilterChain managementSecurityFilterChain(HttpSecurity http) throws Exception {
    http.securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(registry -> registry.anyRequest().permitAll())
            .csrf(AbstractHttpConfigurer::disable);
    return http.build();
}

@Bean
@Order(1)
public SecurityFilterChain mainSecurityFilterChain(HttpSecurity http, ...) throws Exception {
    // 기존 메인 체인
}
```

Actuator 엔드포인트는 정상 동작했습니다. 그런데 이번엔 일반 API 호출에서도 같은 에러가 발생했습니다.

Spring Security 6.x의 `.requestMatchers(String...)`도 내부에서 `DispatcherServletDelegatingRequestMatcher`를 사용합니다. management child context에서는 **문자열 기반 matcher 전체**가 실패합니다. `@Order`로 순서를 조정해도 메인 체인 자체가 child context에서 동작할 수 없는 구조였습니다.

#### 결론: 포트 분리를 철회

복잡한 workaround보다 근본 원인을 제거하기로 했습니다. `management.server.port` 설정을 완전히 제거하고, Actuator를 메인 포트에서 운영하기로 결정했습니다.

```yaml
# Before
management:
  server:
    port: 8099

# After
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,info
  # server.port 없음 → 메인 포트에서 동작 → child context 미생성
```

Actuator 보안은 리버스 프록시에서 처리하는 방식으로 대체했습니다.

```nginx
# Nginx에서 /actuator 외부 접근 차단
location /actuator {
    deny all;
    return 403;
}
```

Prometheus는 Docker 내부 네트워크에서 직접 스크래핑하기 때문에 Nginx를 거치지 않습니다. 외부는 차단하되 내부 수집은 정상 동작합니다.

**배운 점**

* `management.server.port`를 분리하면 child context가 생성되고, 이 context에는 `DispatcherServlet`이 없다
* Spring Security 6.x의 문자열 기반 matcher(`MvcRequestMatcher`)는 `DispatcherServlet`에 의존하므로 child context에서 사용할 수 없다
* management 포트 분리는 Security 설정이 단순한 프로젝트에 적합하다. 복잡한 `MvcRequestMatcher` 기반 Security 설정과는 충돌 가능성이 높다
* 복잡한 workaround보다 근본 원인 제거가 낫다

***

### 2장. Prometheus + Loki + Grafana 스택 구성

#### 전체 아키텍처

```
┌─────────────┐    pull (15s)   ┌─────────────┐
│  Spring App │ ◄─────────────  │ Prometheus  │  메트릭 저장
│  /actuator/ │                 │             │
│  prometheus │                 └──────┬──────┘
└──────┬──────┘                        │
       │ 로그 파일                      ▼
       │ (warn.log, error.log)  ┌─────────────┐
       ▼                        │             │
┌─────────────┐    push         │   Grafana   │
│  Promtail   │ ────────────►   │  Dashboard  │
│             │    Loki         │             │
└─────────────┘                 └─────────────┘
```

구성 요소별 역할을 정리하면 다음과 같습니다.

| 구성 요소      | 역할        | 수집 방식               |
| ---------- | --------- | ------------------- |
| Prometheus | 메트릭 수집/저장 | Pull (15초마다 스크래핑)   |
| Loki       | 로그 수집/저장  | Push (Promtail이 전송) |
| Promtail   | 로그 파일 감시  | 파일 tail + Loki push |
| Grafana    | 시각화       | PromQL / LogQL 조회   |

#### 로그 수집 범위 결정

처음에는 모든 레벨의 로그를 수집하는 appender를 추가했습니다. 하지만 부하와 저장 비용을 고려해 WARN/ERROR만 수집하기로 결정했습니다.

| 방식                      | 장점            | 단점                    |
| ----------------------- | ------------- | --------------------- |
| 전체 로그 수집                | 모든 로그 검색 가능   | 디스크 폭증, Loki 쿼리 성능 저하 |
| **WARN/ERROR만 수집** (채택) | 저장 효율, 핵심 로그만 | INFO는 서버 직접 확인 필요     |

대시보드에서 확인할 로그는 경고/에러 수준이 대부분입니다. INFO는 99%가 정상 요청 로그이고, 이를 전부 Loki에 넣을 이유가 없었습니다.

#### Promtail multiline — prettyPrint JSON 로그 묶기

logback의 JSON 포맷터를 `prettyPrint: true`로 설정하면, 로그 한 건이 여러 줄로 출력됩니다. Promtail은 기본적으로 줄 단위로 읽으므로, Loki에서 로그 1건이 수십 개 엔트리로 쪼개져 보이는 문제가 생깁니다.

```
{                           ← Loki 엔트리 1
  "timestamp" : "...",      ← Loki 엔트리 2
  "level" : "ERROR",        ← Loki 엔트리 3
  "message" : "...",        ← ...
}
```

Promtail의 `multiline` stage로 해결했습니다. `{`로 시작하는 줄을 새 로그의 시작으로 인식하도록 설정합니다.

```yaml
pipeline_stages:
  - multiline:
      firstline: '^\{'    # { 로 시작하는 줄 = 새 로그의 시작
      max_wait_time: 3s
      max_lines: 500
```

prettyPrint JSON에서 `{`가 줄 맨 앞에 나타나는 경우는 최상위 객체 시작뿐입니다. 중첩 객체는 들여쓰기되어 있으므로 `prettyPrint`를 유지하면서 Loki에서 1건 = 1엔트리로 정상 인식됩니다.

#### Promtail timestamp 스테이지

Promtail의 `json` 스테이지는 JSON 필드를 추출만 합니다. 추출한 `timestamp`를 Loki 엔트리 타임스탬프로 설정하려면 별도의 `timestamp` 스테이지가 필요합니다.

```yaml
pipeline_stages:
  - multiline: ...
  - json:
      expressions:
        timestamp: timestamp
        level: level
  - timestamp:
      source: timestamp
      format: '2006-01-02T15:04:05.000-07'  # Go time layout
  - labels:
      level:
```

Promtail의 `format`은 Go의 reference time layout을 사용합니다. logback의 `yyyy-MM-dd'T'HH:mm:ss.SSSX` 형식은 Go layout `2006-01-02T15:04:05.000-07`에 대응합니다.

`timestamp` 스테이지 없이는 Loki에 **Promtail 수집 시점**이 기록됩니다. 로그가 실제로 발생한 시각과 수집된 시각 사이에 지연이 있으면 Grafana 대시보드에서 시간축이 틀어집니다.

#### 개발 환경 vs AKS 환경

운영 환경(AKS)에서는 Docker Compose 대신 Helm 차트를 사용하고, 로그 수집 방식도 다릅니다.

| 항목         | 개발 (Docker Compose)      | 운영 (AKS)                      |
| ---------- | ------------------------ | ----------------------------- |
| 설치 방식      | docker-compose           | Helm 차트                       |
| 로그 수집      | 파일 (warn.log, error.log) | 컨테이너 stdout/stderr            |
| 로그 필터      | `filename=~".*error.*"`  | `namespace`, `app` 라벨         |
| Grafana 접근 | SSH 터널                   | SSH 터널 + kubectl port-forward |

K8s에서 Promtail은 파일이 아닌 컨테이너 로그를 수집하므로, LogQL 쿼리도 달라집니다.

```logql
# 개발 환경
{job="backend", filename=~".*error.*"}

# K8s 환경
{namespace="backend"} | json | level="ERROR"
```

**배운 점**

* `json` 스테이지는 필드 추출만 하고, Loki 타임스탬프 반영은 `timestamp` 스테이지가 별도로 필요하다
* Promtail의 `format`은 Go reference time layout 방식이다
* K8s 환경에서는 로그 라벨 체계가 파일 기반에서 namespace/app 기반으로 완전히 달라진다
* Promtail positions 파일과 로그 파일 크기를 비교하면 새 데이터 유무를 바로 알 수 있다

***

### 3장. 비즈니스 로직에서 메트릭 수집을 분리하다

#### 문제: 관심사가 섞인 코드

로그인 횟수를 측정하기 위해 처음에는 서비스 클래스에 `MeterRegistry`를 직접 주입했습니다.

```java
// Before: 비즈니스 로직과 메트릭 수집이 섞임
@Service
public class UserAppService {
    private final MeterRegistry meterRegistry;

    public TokenResponse login(LoginRequest request) {
        // ... 비즈니스 로직 ...

        Counter.builder("app.login.count")
                .tag("tenant", factoryCode)
                .register(meterRegistry)
                .increment();

        return tokenResponse;
    }
}
```

메트릭 수집은 비즈니스의 관심사가 아닙니다. 로그인이라는 핵심 로직이 Micrometer API에 의존하게 되고, 메트릭 수집 방식이 바뀌면 서비스 코드를 수정해야 합니다.

#### 해결: ApplicationEvent 기반 분리

프로젝트에서 이미 알림 기능에 `@TransactionalEventListener` 패턴을 사용하고 있었습니다. 같은 방식을 메트릭 수집에 적용했습니다.

```
Before:
  UserAppService → MeterRegistry → Counter.increment()

After:
  UserAppService → ApplicationEventPublisher → LoginEvent
                                                    ↓ (AFTER_COMMIT)
                   MonitoringMetricsEventListener → MeterRegistry → Counter.increment()
```

```java
// LoginEvent
public class LoginEvent extends ApplicationEvent {
    private final String username;
    private final String tenant;
}

// MonitoringMetricsEventListener
@Component
public class MonitoringMetricsEventListener {
    private final MeterRegistry meterRegistry;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleLoginEvent(LoginEvent event) {
        Counter.builder("app.login.count")
                .tag("tenant", event.getTenant())
                .register(meterRegistry)
                .increment();
    }
}

// UserAppService — MeterRegistry 의존 완전 제거
eventPublisher.publishEvent(new LoginEvent(this, request.username(), factoryCode));
```

#### AFTER\_COMMIT을 선택한 이유

`@TransactionalEventListener`의 phase를 `AFTER_COMMIT`으로 설정한 데는 이유가 있습니다.

로그인 트랜잭션이 롤백되는 경우를 생각해보면, `BEFORE_COMMIT`이나 기본값(`AFTER_COMMIT`이 아닌)이면 실제로 로그인이 성공하지 않았는데 카운터가 올라갈 수 있습니다. `AFTER_COMMIT`은 트랜잭션이 성공적으로 커밋된 이후에만 이벤트를 처리하므로, 롤백된 요청이 메트릭에 반영되지 않습니다.

또한 try-catch로 감싸서 메트릭 기록 실패가 핵심 로직에 영향을 주지 않도록 격리했습니다.

**배운 점**

* 메트릭/로깅 같은 부가 관심사는 비즈니스 로직에서 분리해야 한다 (SRP)
* `AFTER_COMMIT`을 사용하면 트랜잭션 성공 후에만 메트릭이 기록되어, 롤백 시 잘못된 카운트가 쌓이지 않는다
* 이미 사용 중인 이벤트 패턴을 재활용하면 일관성 있는 코드베이스를 유지할 수 있다

***

### 4장. HTTP 메트릭에 테넌트 태그를 추가하면서 만난 ThreadLocal 타이밍 문제

#### 배경

Prometheus가 수집하는 `http.server.requests` 메트릭에 테넌트(tenancy) 태그를 추가하고 싶었습니다. 어떤 테넌트가 API를 얼마나 사용하는지, 에러율은 어느 정도인지 테넌트 단위로 분석하기 위해서입니다.

#### 문제: ThreadLocal이 비워진 이후에 메트릭이 기록된다

서비스는 멀티 테넌시 구조로, JWT 인증 필터에서 테넌트 정보를 `ThreadLocal`에 저장합니다. 메트릭에 테넌트 태그를 추가하려면 이 `ThreadLocal`에서 값을 읽으면 됩니다. 처음에는 그렇게 생각했습니다.

```java
// 직관적인 구현 — 하지만 동작하지 않음
private KeyValue tenantKeyValue(ServerRequestObservationContext context) {
    String tenant = SchemaContextHolder.getCurrentSchema(); // ThreadLocal에서 읽기
    if (tenant != null) {
        return KeyValue.of("tenant", tenant);
    }
    return KeyValue.of("tenant", "unknown"); // 항상 unknown이 반환됨
}
```

항상 `"unknown"`이 반환되었습니다. 왜 그럴까?

Spring의 인터셉터 `afterCompletion()`에서 `ThreadLocal`을 `clear()`합니다. 그런데 Micrometer의 `Observation.stop()`(메트릭이 실제로 기록되는 시점)은 필터 체인 완료 이후입니다.

```
ServerHttpObservationFilter.start()
  → JwtAuthenticationFilter  : ThreadLocal.set(tenantCode)
  → 인터셉터 preHandle()      : ThreadLocal 설정 확인
  → Controller → Service     : 비즈니스 로직 처리
  → 인터셉터 afterCompletion(): ThreadLocal.clear() ← 여기서 비워짐
  → ServerHttpObservationFilter.stop()
      → ServerRequestObservationConvention 호출 ← 메트릭 기록 시점
          → ThreadLocal.get() = null ← 이미 비워진 상태
```

#### 해결: request scope 활용

`ThreadLocal`은 인터셉터 종료 시 사라지지만, `HttpServletRequest`의 attribute는 응답이 완료될 때까지 유지됩니다. JWT 필터에서 ThreadLocal에 저장할 때 request attribute에도 함께 저장하도록 했습니다.

```java
// JwtAuthenticationFilter — 1줄 추가
SchemaContextHolder.setCurrentSchema(tenantCode);
request.setAttribute("tenant", tenantCode);  // request scope에도 저장
```

`IfAbsent` 패턴으로 먼저 설정된 값이 덮어써지지 않도록 보장했습니다.

```java
// 인터셉터 — JWT에서 이미 설정한 값을 덮어쓰지 않음
private void setTenantAttributeIfAbsent(HttpServletRequest request, String tenant) {
    if (request.getAttribute("tenant") == null) {
        request.setAttribute("tenant", tenant);
    }
}
```

`ServerRequestObservationConvention`에서는 ThreadLocal 대신 request attribute를 읽습니다.

```java
@Component
public class TenantServerRequestObservationConvention
        extends DefaultServerRequestObservationConvention {

    @Override
    public KeyValues getLowCardinalityKeyValues(ServerRequestObservationContext context) {
        return super.getLowCardinalityKeyValues(context)
                .and(tenantKeyValue(context));
    }

    private KeyValue tenantKeyValue(ServerRequestObservationContext context) {
        HttpServletRequest request = context.getCarrier();
        if (request == null) {
            return KeyValue.of("tenant", "unknown");
        }
        Object tenant = request.getAttribute("tenant");
        if (tenant instanceof String tenantValue && !tenantValue.isEmpty()) {
            return KeyValue.of("tenant", tenantValue);
        }
        return KeyValue.of("tenant", "unknown");
    }
}
```

`@Component`로 등록하면 Spring Boot 3.x가 자동으로 `ServerHttpObservationFilter`에 주입합니다.

#### 카디널리티 고려

테넌트 태그를 추가하면 시계열 수가 늘어납니다. Prometheus는 **메트릭 이름 + 태그 조합 하나**를 시계열 1개로 관리하기 때문에, 태그를 추가하면 기존 시계열 수가 태그의 고유 값 수만큼 곱해집니다.

```
시계열 수 = tenant × uri × method × status × (histogram bucket + 3)
```

테넌트 수가 수십 개 규모라면 실제 시계열은 5만\~10만 수준으로 예상할 수 있고, Prometheus는 수백만 시계열까지 처리 가능하므로 안전한 범위입니다.

카디널리티는 다음 PromQL로 모니터링할 수 있습니다.

```promql
# 전체 시계열 수
prometheus_tsdb_head_series

# http 메트릭 시계열 수만 확인
count({__name__=~"http_server_requests.*"})
```

**배운 점**

* Spring 필터/인터셉터 실행 순서와 Micrometer Observation 라이프사이클의 타이밍을 이해해야 메트릭 태깅이 정확하다
* `ThreadLocal`은 `clear()` 시점에 따라 값이 사라지므로, 요청 전체 라이프사이클 동안 값을 유지하려면 request scope(`setAttribute`)를 활용해야 한다
* `DefaultServerRequestObservationConvention`을 확장하고 `@Component`로 등록하면 Spring Boot가 자동 주입한다

***

### 결론. 모니터링을 도입하면서 배운 것들

#### 최종 구성

```
외부 요청
    ↓
[Nginx]               /actuator/** → 외부 차단 (403)
                      그 외        → 앱 서버로 프록시
    ↓
[Spring Boot App]
    ├── /actuator/prometheus  ← Prometheus가 내부 네트워크에서 스크래핑
    └── /actuator/health      ← 헬스체크

[필터 체인]
    ├── JwtAuthenticationFilter  → ThreadLocal + request.setAttribute("tenant", ...)
    ├── ServerHttpObservationFilter
    │       └── TenantServerRequestObservationConvention
    │               └── 메트릭 기록 시 request.getAttribute("tenant") 읽기
    └── 인터셉터 afterCompletion() → ThreadLocal.clear()

[비즈니스 이벤트]
    └── UserAppService → LoginEvent → (AFTER_COMMIT) → Counter.increment()

[로그 파이프라인]
    └── warn.log / error.log
            ↓ Promtail (multiline + timestamp stage)
            ↓ Loki push
            ↓ Grafana LogQL 조회
```

#### 돌아보며

모니터링을 "나중에 붙이면 되는 것"으로 미뤄왔습니다. 그 결과 장애가 터지면 로그를 뒤지는 방식으로 대응했고, 서버 상태를 숫자로 확인할 방법이 없었습니다.

이번에 스택을 구성하면서 느낀 것들을 정리하면 다음과 같습니다.

**management 포트 분리는 security 설정이 단순할 때만 유효하다.** `MvcRequestMatcher` 기반의 복잡한 Security 설정을 쓰고 있다면, 포트 분리는 child context 문제로 이어진다. 이 경우 리버스 프록시에서 경로를 차단하는 방식이 더 현실적이다.

**메트릭 수집은 비즈니스 로직 밖에 있어야 한다.** 서비스 클래스에 `MeterRegistry`를 직접 넣으면 비즈니스 코드가 인프라에 결합된다. 이벤트 기반으로 분리하면 SRP를 지키면서 메트릭 수집 방식을 독립적으로 변경할 수 있다.

**ThreadLocal의 생명주기를 알아야 한다.** "요청 처리 중에는 ThreadLocal이 살아 있다"는 가정은 틀렸다. 인터셉터 `afterCompletion()`에서 정리되고, Micrometer의 Observation stop은 그 이후에 발생한다. 값을 오래 유지해야 한다면 request scope를 활용해야 한다.

**`spy.properties` 같은 네이티브 설정은 Spring Boot 자동 설정보다 우선할 수 있다.** (이전 글에서도 같은 교훈을 얻었지만) 프레임워크 위에서 동작하는 라이브러리는 자체 초기화 순서와 설정 경로를 갖고 있다. 동작하지 않을 때는 바이트코드를 직접 확인하는 것이 가장 확실한 디버깅 방법이다.

모니터링은 붙이는 것이 아니라 설계하는 것이었습니다. 무엇을 보고 싶은지를 먼저 정의하고, 그에 맞는 메트릭과 로그를 선택해야 합니다. 수집할 수 있다고 해서 전부 수집하면, 정작 필요한 정보가 노이즈에 파묻힙니다.

***

### 참고

**Spring Boot Actuator / Micrometer**

1. **Spring Boot Actuator 공식 문서**
   * https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html
2. **Micrometer ServerRequestObservationConvention**
   * https://micrometer.io/docs/observation
3. **Spring Security — Management Server Port 분리 주의사항**
   * https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.monitoring

**Prometheus / Grafana / Loki**

4. **Prometheus 공식 문서**
   * https://prometheus.io/docs/introduction/overview/
5. **Grafana Loki 공식 문서**
   * https://grafana.com/docs/loki/latest/
6. **Promtail pipeline stages — multiline, timestamp**
   * https://grafana.com/docs/loki/latest/send-data/promtail/stages/
7. **kube-prometheus-stack Helm 차트**
   * https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
