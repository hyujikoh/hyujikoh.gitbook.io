# 어플리케이션 로그, 줄여야 보인다: Logbook과 P6Spy 리팩토링으로 로그 정리

> TL;DR 로그는 많다고 좋은게 아니라, 내가 모니터링 하고 관리하는데 필수적인 로그들만 구성하고 그걸 판단하는 명확한 기준을 세워야 한다 이해했습니다.

## 서론 : 이 글을 작성하게된 이유

서비스의 로깅 정책운 처음 개발을 시작할 때부터 지금까지 명확한 해답을 찾지 못했습니다. \
그나마 경험이 쌓이고 이리저리 좋은 오픈소스를 하나하나 붙여가면서 모니터링을 용이하게 구축하도록 노력했습니다.

오늘 소개할 서비스의 HTTP 요청/응답은 LogBook 을 이용해 관리하고, \
어플리케이션에서 실행되는 쿼리 로깅은 P6Spy 가 각각 책임을 갖도록 구성 하였고, \
각 ThreadLocal 에 상관관계 ID 를 부여함으로 스레드 별로 로그를 추척 할 수 있게 구성하였습니다.

**정보는 최대한 많이 남겨두는 게 좋다**고 생각했습니다.\
장애가 발생했을 때 로그가 부족해서 원인을 못 찾는 것보다는, \
많이 남겨두고 나중에 필요한 걸 찾아보는 편이 낫다고 판단했기 때문입니다.

그런데 서비스 사용자가 늘어나고 트래픽이 비대해지면서, 불필요한 로그가 눈에 들어오기 시작했습니다. \
하물며 굳이 기록 하지 않아도 되는 Swagger UI 리소스 요청/응답 로그가 수십 줄씩 쌓이고, \
단순 조회 API의 SELECT 쿼리가 로그의 대부분을 차지했습니다. \
아무리 상관관계 ID로 스레드를 구분할 수 있다 해도, 정작 봐야 할 로그가 잡음에 파묻혀 보이지 않았습니다.&#x20;

**로그는 많을수록 좋은 게 아니라, 필요한 것만 남아 있어도 충분한 것이였습니다.**

이번 글은 기술 부채로 남겨놨던 로그 정책을 개선하고 \
그 과정에서 해결했던 문제들을 공유하고자 합니다.

글은 다음과 같이 구성하였습니다.

* Hibernate SQL과 P6Spy가 중복 출력되는 구조를 정리한 과정
* 프로필별(개발/운영) 로깅 전략을 어떻게 나눴는가
* Logbook에서 URL 패턴과 HTTP 메서드를 필터링한 방법
* P6Spy에서 불필요한 쿼리를 제외하려다 4번 실패하고, 결국 성공한 과정

**최종 목적은 "왜 이렇게 판단했고, 어디서 막혔고, 어떻게 풀었는지"** 그 고민의 과정을 담으려 했습니다.



## 중복 쿼리: Hibernate SQL과 P6Spy

운영 서버에서  일부 설정 오류로 Hibernate SQL 과 P6Spy 둘다 활성화가 되어있어서 \
중복 쿼리가 2번이나 발생하는 상황이 발생하였습니다.

로그는 아래와 같은 형식으로 연속으로 발생하였습니다.&#x20;

<pre class="language-log"><code class="lang-log"><strong>// Hibernate SQL (org.hibernate.SQL=DEBUG)
</strong>Hibernate:
    select p.id, p.name, p.price
    from product p
    where p.id = ?

// P6Spy
[statement] | 3 ms | select p.id, p.name, p.price from product p where p.id = 42

</code></pre>

&#x20;같은 쿼리이지만 실제 다른 출력행태를 가지고 있었습니다.&#x20;

Hibernate는 \`?\`로 파라미터를 표시하고, P6Spy는 실제 값 \`42\`가 바인딩된 완성된 SQL을 보여줍니다. \
왜 이런 차이가 발생하는지 이해하기 위해, 두 도구가 SQL을 캡처하는 위치를 파악 했습니다.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

**Hibernate SQL 로깅**은 JPA 내부에서 JPQL을 SQL로 변환한 시점에 기록합니다. \
아직 JDBC 드라이버에 전달하기 전이라, 파라미터는 `?` 플레이스홀더 상태입니다. \
바인딩된 실제 값을 보려면 `org.hibernate.type.descriptor.sql=TRACE` 레벨을 별도로 설정해야 합니다.

**P6Spy**는 JDBC DataSource를 프록시로 래핑하는 방식입니다. \
어플리케이션과 실제 JDBC 드라이버 사이에 끼어들어서, 드라이버에 전달되기 직전의 완성된 SQL을 캡처합니다. \
그렇기 때문에 파라미터가 이미 바인딩된 상태이고, 실행 시간(ms)까지 측정할 수 있습니다.

<table><thead><tr><th width="151">구분</th><th width="241" align="center">Hibernate SQL</th><th width="346" align="center">P6Spy</th></tr></thead><tbody><tr><td>파라미터 바인딩</td><td align="center"><code>?</code> 표시 (별도 TRACE 설정 필요)</td><td align="center">실제 값이 바인딩된 완성 SQL</td></tr><tr><td>대상 범위</td><td align="center">Hibernate가 생성한 SQL만</td><td align="center">MyBatis, Native Query 등 JDBC 전체</td></tr><tr><td>실행 시간</td><td align="center">미제공</td><td align="center">쿼리별 실행 시간(ms) 측정</td></tr><tr><td>커넥션 정보</td><td align="center">미제공</td><td align="center">커넥션 ID, 카테고리 포함</td></tr><tr><td>로그 레벨</td><td align="center"><code>DEBUG</code></td><td align="center"><code>INFO</code></td></tr><tr><td>설정 방법</td><td align="center"><code>logging.level.org.hibernate.SQL</code></td><td align="center">의존성 추가만으로 자동 활성화</td></tr></tbody></table>

여기서 주목할 점은 **로그 레벨 차이**입니다.

Hibernate SQL은 `DEBUG` 레벨로 출력하고, P6Spy는 `INFO` 레벨로 출력합니다. \
운영 환경에서 root 로그 레벨이 `INFO`라면, Hibernate SQL(`DEBUG`)은 자동으로 안 보이지만 \
P6Spy(`INFO`)는 그대로 출력됩니다. \
즉 "운영에서는 Hibernate SQL이 안 보이니까 괜찮다"고 넘어갈 수 있지만, \
**P6Spy는 별도로 꺼주지 않으면 운영에서도 모든 SQL을 로깅**하게 됩니다.

#### P6Spy가 Hibernate SQL의 상위 호환인가?

기능 비교만 놓고 보면 그렇습니다. P6Spy는 Hibernate SQL이 하는 일을 모두 포함하면서, \
바인딩 파라미터와 실행 시간까지 추가로 제공합니다. \
여기에 더해 MyBatis나 Native Query처럼 Hibernate를 거치지 않는 SQL까지 캡처합니다.

다만 트레이드오프가 있습니다. P6Spy는 DataSource를 프록시로 래핑하는 방식이기 때문에, **로그를 출력하지 않더라도 프록시 오버헤드는 존재**합니다. 모든 SQL이 프록시 레이어를 한 번 더 거치게 되는 것입니다. 개발 환경에서는 무시해도 될 수준이지만, 운영 환경에서는 로그 레벨을 높이는 것만으로는 부족하고, 프록시 자체를 비활성화하는 것이 맞다고 판단했습니다.

#### Hibernate SQL은 끄고, P6Spy로 통합

대부분의 레퍼런스에서도 Hibernate SQL은 운영에서 사용하지 말라고 권장하였기 때문에. P6Spy 하나로 통합하고, Hibernate SQL 로깅은 전체 프로필에서 `warn`으로 올려 비활성화했습니다.&#x20;

실제 프로필별 로깅 전략은 다음과 같이 정리하였습니다.

* **Hibernate SQL**: 전체 프로필에서 `warn`으로 비활성화
* **P6Spy**: 개발 환경에서만 활성화, 운영 환경에서는 `enable-logging: false`로 비활성화

```yaml
# application-dev.yml
logging.level:
  org.hibernate:
    SQL: warn    # debug → warn

# application-prod.yml
decorator:
  datasource:
    p6spy:
      enable-logging: false
```

운영 환경에서 로그 레벨만 올리지 않고 `enable-logging: false`를 사용한 이유는 다음과 같습니다.

* 로그 레벨을 올리면 출력은 막을 수 있지만, `p6spy-spring-boot-starter`가 DataSource를 `P6DataSource`로 Wrapping 하는 것까지는 막지 못합니다.&#x20;
* 그렇게 되면 모든 SQL이 프록시 레이어를 거치는 오버헤드가 그대로 남기 때문에, \
  이를 위해 설정한  `enable-logging: false` 설정은 이 래핑 자체를 하지 않도록 해줍니다.

정리하면 다음과 같습니다.

<table><thead><tr><th width="113">프로필</th><th align="center">Hibernate SQL</th><th align="center">P6Spy</th><th>SQL 출력</th></tr></thead><tbody><tr><td> dev</td><td align="center">OFF (<code>warn</code>)</td><td align="center">ON</td><td>P6Spy만 (바인딩된 완성 SQL)</td></tr><tr><td>prod</td><td align="center">OFF (<code>warn</code>)</td><td align="center"><strong>OFF</strong></td><td>SQL 로그 없음, 프록시 오버헤드 없음</td></tr></tbody></table>

## Logbook 기반 HTTP 로깅 개선 — URL 제외가 동작하지 않았던 이유

다음 HTTP 로깅 개선에 대해 설명을 드리기 전 프로젝트에서 사용중인 logbook 에 대해서 설명드리겠습니다.

#### Logbook이란

[Logbook](https://github.com/zalando/logbook)은 Zalando에서 만든 HTTP 요청/응답 로깅 라이브러리입니다. Spring Boot의 기본 HTTP 로깅과는 달리, 요청 본문(body)과 응답 본문을 함께 기록해주고, JSON pretty print, 헤더 마스킹, 조건부 필터링 같은 기능을 제공합니다.

서비스에서는 Logbook을 다음과 같이 구성하고 있었습니다.

```java
@Configuration
public class LogbookConfig {
    public static final String[] NOT_LOGGING_URIS = {
        /**
        허용하지 않을 URI 배열
        **/
    }
    @Bean
    public Logbook logbook() {
        Predicate<HttpRequest> excludeUris =
            exclude(stream(NOT_LOGGING_URIS).map(Conditions::requestTo));

        return Logbook.builder()
                .bodyFilter(new PrettyPrintingJsonBodyFilter())
                .condition(excludeUris)
                .build();
    }
}
```

`NOT_LOGGING_URIS`에 정의된 URI은 로깅에서 제외하고, \
나머지 요청에 대해 JSON body를 포맷팅해서 기록하는 구조입니다. \
앞서 소개한 상관관계 ID와 함께 사용하면, \
특정 요청이 어떤 body로 들어와서 어떤 응답을 내려줬는지 스레드 단위로 추적할 수 있습니다.

문제는 모든 http 요청과 응답을 이미 `NOT_LOGGING_URIS`  로 제외 설정은 해놨다고 생각했지만. \
불필요한 요청과 응답이 계속해서 로깅 되는것이 확인되었습니다.

이 문제를 해결하기 위해 설정 파일을 다시 한번 확인 해봤습니다.

기존 코드를 열어보니 Swagger UI의 리소스를 **개별 파일 단위로 20개** 나열하고 있었습니다.

```java
public static final String[] NOT_LOGGING_URIS = {
    HEALTH_CHECK,
    SWAGGER_UI_ROOT,          // "/swagger-ui"
    SWAGGER_UI,               // "/swagger-ui/swagger-ui.html"
    SWAGGER_INDEX_HTML,       // "/swagger-ui/index.html"
    SWAGGER_INDEX_CSS,        // "/swagger-ui/index.css"
    // ... 총 20개 개별 파일
};
```

문제는 두 가지였습니다.

첫째, Swagger UI는 브라우저에서 로딩할 때 `oauth2-redirect.html`, ES 번들 등 목록에 없는 리소스도 추가로 요청합니다. 개별 나열로는 새로 추가되는 리소스를 매번 따라잡을 수 없었습니다.

둘째, URL 인코딩 불일치 문제가 있었습니다.

```java
public static final String SWAGGER_API_DOCS_GENERATED_API = SWAGGER_API_DOCS + "/Generated API";
// 패턴: "/v3/api-docs/Generated API" (공백 포함)
// 실제 요청: "/v3/api-docs/Generated%20API" (URL 인코딩)
// → 매칭 실패
```

공백이 포함된 경로를 그대로 상수로 정의했기 때문에, 실제 요청의 URL 인코딩된 경로와 매칭되지 않았습니다.

#### Logbook은 glob 패턴을 지원한다

해결 방법을 찾기 위해 Logbook의 `Conditions.requestTo()` 내부 구현을 확인했습니다.\
`javap -c -p`로 바이트코드를 디컴파일한 결과, 내부적으로 `Glob.compile()`을 호출하고 있었고, \
이 `Glob` 클래스는 다음과 같은 패턴 변환 규칙을 갖고 있었습니다.

| Glob 패턴   | 정규식 변환    | 설명             |
| --------- | --------- | -------------- |
| `/**` (끝) | `(/.*)?$` | 하위 모든 경로 (선택적) |
| `/*` (끝)  | `/[^/]*$` | 단일 세그먼트        |
| `*`       | `[^/]*?`  | 임의 문자 (경로 제외)  |

즉 `/swagger-ui/**`로 작성하면 `/swagger-ui` 하위의 모든 경로가 한 줄로 매칭됩니다.

#### 수정: 20개 나열에서 glob 패턴 5개로

```java
// 수정 전 - 20개 개별 파일 나열
public static final String[] NOT_LOGGING_URIS = {
    HEALTH_CHECK, SWAGGER_UI_ROOT, SWAGGER_UI,
    SWAGGER_INDEX_HTML, SWAGGER_INDEX_CSS, ... // 20개
};

// 수정 후 - Glob 패턴 5개
public static final String[] NOT_LOGGING_URIS = {
    HEALTH_CHECK,        // /health/check
    "/swagger-ui/**",    // swagger-ui 하위 모든 리소스
    "/v3/api-docs/**",   // API docs 하위 모든 리소스
    API_DOC_HTML,        // /static/docs/index.html
    FAVICON              // /favicon.ico
    /**
    그 이외 정책적으로 불필요한 플랫폼 api  
    **/
};
```

20개를 나열해도 빠지는 것이 있었는데, glob 패턴 5개로 줄이니 오히려 누락 없이 전부 매칭되었습니다.

#### GET 요청 제외

URL 제외를 정리하고 나니, 다음으로 눈에 띈 것은 GET 요청 로그였습니다.

서비스 특성상 조회 API가 대부분이라 GET 요청의 요청/응답 로그가 로그의 상당 부분을 차지하고 있었습니다. 반면 문제가 발생했을 때 추적이 필요한 것은 데이터를 변경하는 POST, PUT, DELETE 요청이었습니다.

Logbook의 `Conditions` 클래스는 URL 패턴뿐 아니라 HTTP 메서드 기반 필터링도 제공하고 있었습니다.

```java
@Bean
public Logbook logbook() {
    Predicate<HttpRequest> excludeUris =
        exclude(stream(NOT_LOGGING_URIS).map(Conditions::requestTo));
    Predicate<HttpRequest> excludeGet =
        exclude(Conditions.requestWithMethod("GET"));

    return Logbook.builder()
            .bodyFilter(new PrettyPrintingJsonBodyFilter())
            .condition(excludeUris.and(excludeGet))
            .build();
}
```

`excludeUris`와 `excludeGet`을 `and`로 연결해서, 제외 URL이거나 GET 요청이면 로깅을 하지 않도록 구성했습니다.

#### HTTP 필터링과 SQL 필터링은 독립적이다

여기서 한 가지 짚고 넘어갈 점이 있습니다. Logbook에서 GET 요청을 제외해도, 그 GET 요청이 트리거한 SQL 쿼리는 P6Spy에서 그대로 출력됩니다. 둘은 완전히 다른 계층에서 동작하기 때문입니다.

```
GET /api/v2/products
    ↓
[Logbook Filter]  ← HTTP 로깅 (GET 이므로 제외됨)
    ↓
[Controller → Service → Repository]
    ↓
[P6Spy Proxy]     ← SQL 로깅 (HTTP 메서드 개념 없음, SELECT 쿼리 그대로 출력)
    ↓
[JDBC Driver → Database]
```

Logbook은 HTTP 계층, P6Spy는 JDBC 계층입니다. 서로의 필터링 설정은 영향을 주지 않습니다. 이 차이를 인식한 것이 다음 장에서 P6Spy의 SELECT 쿼리를 별도로 필터링하게 된 계기가 되었습니다.
