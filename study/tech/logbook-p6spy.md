# 어플리케이션 로그, 줄여야 보인다: Logbook과 P6Spy 리팩토링으로 로그 정리

> TL;DR 로그는 많다고 좋은게 아니라, 내가 모니터링 하고 관리하는데 필수적인 로그들만 구성하고 그걸 판단하는 명확한 기준을 세워야 한다 이해했습니다.

## 서론 : 이 글을 작성하게된 이유

서비스의 로깅 정책운 처음 개발을 시작할 때부터 지금까지 명확한 해답을 찾지 못했습니다. \
그나마 경험이 쌓이고 이리저리 좋은 오픈소스를 하나하나 붙여가면서 모니터링을 용이하게 구축하도록 노력했습니다.

오늘 소개할 서비스의 HTTP 요청/응답은 LogBook 을 이용해 관리하고, \
어플리케이션에서 실행되는 쿼리 로깅은 P6Spy 가 각각 책임을 갖도록 구성 하였고, \
각 ThreadLocal 에 상관관계 id 를 부여함으로 스레드 별로 로그를 추척 할 수 있게 구성하였습니다.

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

대부분의 레퍼런스에서도 Hibernate SQL은 운영에서 사용하지 말라고 권장하였기 때문에. P6Spy 하나로 통합하고, Hibernate SQL 로깅은 전체 프로필에서 `warn`으로 올려 비활성화했습니다. \
실제 프로필별 로깅 전략을 어떻게 나눴는지는 다음장에서 소개 하겠습니다.
