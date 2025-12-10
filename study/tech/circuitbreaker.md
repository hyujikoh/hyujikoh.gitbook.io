# Resilience4j 서킷 브레이커, 정말 내 서비스에서 제대로 동작하는가

좋습니다. 전체 흐름은 이미 너무 잘 잡으셨어요.\
문장만 조금 다듬으면 그대로 써도 될 정도입니다.\
아래는 말투/호흡만 정리한 버전입니다.

***

## 서론: 이 글을 작성하게 된 이유 <a href="#undefined" id="undefined"></a>

결제 도메인을 다루다 보면 불가피하게 외부 PG사 모듈을 쓰게 됩니다.\
사실 PG 모듈뿐 아니라, 자체 서비스가 아닌 외부 서비스를 사용하는 일은 흔합니다.

이때 언제 닥칠지 모르는 외부 장애의 영향 범위를 최소화하기 위해 Resilience4j 서킷 브레이커를 적용해서, 더 안정적인 서비스를 운영하려고 합니다.\
Resilience4j로 서킷 브레이커를 적용하는 것 자체는 레퍼런스가 많아서 난이도가 아주 높지는 않습니다.\
물론 아키텍처 관점에서 깊이 들어가면 충분히 어려운 패턴이긴 합니다.

그런데 막상 운영 환경을 떠올려 보면 고민이 하나 생깁니다.

> “지금 내가 설정해 둔 서킷 브레이커가\
> 실제 우리 서비스 흐름에서 정말 의도한 대로 동작하고 있는 걸까?”

PG는 외부 시스템이라 응답 시간도 들쭉날쭉하고, 장애 패턴도 예측하기 어렵습니다.\
일부러 장애를 내보기도 애매하고, 테스트 환경이라고 해서 항상 같은 식으로 실패해 주지도 않습니다.\
그러다 보니 “서킷 브레이커 설정은 넣어 놨는데, 이게 맞는지 틀린지 애매한 상태”가 되기 쉽습니다.

이 글은 그 애매함에서 글로 쓰면 좋을것 같아서 작성하게 되었습니다. Resilience4j를 결제/주문 흐름에 붙여 놓고,\
&#xNAN;**"정말 내 서비스에서 제대로 동작하는지 테스트로 어떻게 확인할 수 있을까?”** 를 정리해 보려고 합니다.

이 글에서 다루는 내용은 다음과 같습니다.

* 실제 PG를 붙여 놓고 테스트하기 어려운 이유
* PG를 전부 Mock으로 두고, 우리 서비스의 “반응”만 테스트하기로 한 이유
* 서킷 브레이커가 제대로 열리고/닫히는지, fallback이 기대대로 타는지 검증한 방법
* PG 콜백을 가정해 결제·주문 상태 전이를 테스트한 방법
* Postman으로 두드리는 대신, 단위/통합 테스트를 선택한 이유



## 1장. 실제 PG 없이 서킷 브레이커를 어떻게 테스트할까 <a href="#id-1--pg" id="id-1--pg"></a>

이번 작업에서 가장 먼저 부딪힌 고민은 다음과 같습니다.

> “서킷 브레이커 설정은 넣어 놨다.\
> 그럼 이제 이게 **제대로 동작하는지**를 어떻게 검증하지?”

서킷 브레이커는 결국 “외부가 이상하게 동작할 때, 우리 쪽이 어떻게 반응해야 하는가”에 대한 정책입니다.\
그러려면 당연히 다음 같은 상황들을 고려해 테스트를 통해 다음과 같은 것들을 검증합니다.

* 실패율이 커지면 `CLOSED → OPEN` 으로 전환되는지
* 일정 시간 후 `OPEN → HALF_OPEN → CLOSED` 복구가 잘 되는지
* `OPEN` 상태에서 실제로 외부 호출이 막히는지, `fallback` 이 타는지
* 성공/실패/notPermitted 같은 메트릭이 기대한 값으로 쌓이는지

문제는, 이걸 운영 **PG**를 붙여 놓고 테스트하기가 굉장히 애매하다는 점이었습니다.

### 1-1. PG를 그대로 두고 테스트하기가 애매했던 이유

PG는 저희가 제어할 수 없는 외부 시스템입니다.

* 응답이 어떤 타이밍에 느려질지 알 수 없고&#x20;
* 언제 오류를 낼지, 어떤 패턴으로 실패할지 예측하기 어렵고
* 테스트 환경이라고 해서 항상 같은 식으로 망가져 주지도 않습니다.

예를 들어, 이런 걸 테스트하고 싶다고 가정해 보겠습니다.

* “10번 중 6번 실패했을 때, failureRateThreshold 50%를 넘어서 OPEN으로 전환되는가?”
* “OPEN 상태에서 호출했을 때, 실제 PG는 한 번도 안 불리고 바로 notPermitted로 떨어지는가?”
* “HALF\_OPEN 상태에서 허용된 몇 건이 모두 성공하면 CLOSED로 돌아가는가?”

이걸 실제 PG에 붙여 놓고 만들려고 하면, 거의 불가능에 가깝습니다.\
실패를 “대략 몇 번” 내보는 수준은 가능하겠지만,\
**정확한 패턴과 횟수, 타이밍을 반복해서 재현하는 것**은 현실적이지 않습니다.

그래서 테스트 주도 개발에서 많이 나오는 개념인 Mock 을 생각했습니다.

> PG 자체를 테스트하려고 하지 말자.\
> 우리가 원하는 응답을 주는 PG라는 이름에 **가짜**를 이용해,\
> 외부가 이렇게 행동했을 때 **우리 쪽이 어떻게 반응하는지**만 테스트.

여기서부터 Mock을 쓰는 쪽으로 자연스럽게 결론이 났습니다.

### 1-2. 테스트에서 세운 기준: “PG는 전부 Mock, 우리는 반응만 본다”

이번에 테스트를 설계하면서 기준을 아주 단순하게 잡았습니다.

* 실제 PG 모듈은 **테스트 대상이 아니다.**
* 테스트에서는 PG를 전부 **Mock**으로 둔다.
* PG가 어떤 응답/예외/지연을 보인다고 **가정**했을 때,
  * 서킷 브레이커 설정이 의도대로 동작하는지,
  * 결제/주문 도메인이 의도한 상태로 전이되는지.

테스트는 크게 두 케이스로 나눴습니다.

1. **서킷 브레이커 동작 검증**
   * PG 클라이언트 인터페이스를 Mock으로 대체
   * 실패/성공 시나리오를 마음대로 조합해서 원하는 상황에 대해 검증.
     * 상태 전이(CLOSED/OPEN/HALF\_OPEN)
     * fallback 동작
     * 메트릭
2. **PG 콜백 이후 흐름 검증**
   * PG에서 콜백이 온 것처럼 HTTP 요청 전송
   * 결제 상태(PENDING/COMPLETED/FAILED)와\
     주문 상태(CONFIRMED/CANCELLED)가\
     기대하는 대로 변하는지 확인.
   * 중복 콜백, 뒤늦은 콜백 같은 케이스도 고려.

둘 다 공통점은 같았습니다.

> 외부는 전부 가짜(Mock)이고,\
> **우리가 설계한 “반응”만 테스트한다.**

### 1-3. 서킷 브레이커 쪽에서 집중한 포인트

서킷 브레이커 테스트에서는 다음 포인트들을 실제 코드로 고려하였습니다.

* `minimumNumberOfCalls`를 채우기 전에는\
  실패가 나도 failureRate가 계산되지 않는지
* 실패율이 `failureRateThreshold` 이상일 때만\
  `CLOSED → OPEN` 으로 전환되는지
* `OPEN` 상태에서 호출하면\
  실제 PG Mock은 전혀 호출되지 않고 notPermitted로 처리되는지
* fallback 메서드가 정상적으로 호출되고,\
  비즈니스 로직이 우리가 기대한 상태로 정리되는지
* `OPEN → HALF_OPEN → CLOSED` 복구 플로우가\
  성공/실패 조합에 따라 의도대로 움직이는지
* 성공/실패/notPermitted 호출 수, failureRate 같은 메트릭이\
  설정과 시나리오에 맞게 쌓이는지

실패/성공 패턴은 전부 Mock으로 만들었습니다.

* 어떤 테스트에서는 PG Mock이 **항상 예외를 던지도록** 만들고
* 어떤 테스트에서는 **처음 N번은 실패, 이후에는 성공**하도록 바꿔서
* 특정 실패율과 상태 전이를 정확히 만들어 냅니다.

실제 PG 를 이용하는 대신 PG 라는 Mock 을 이용한 테스트 코드 안에서는 원하는 만큼 반복해서 재현할 수 있습니다.

### 1-4. 콜백/이벤트 흐름에서도 같은 전략을 썼다

PG 콜백 이후 흐름도 마찬가지였습니다.

* PG가 “성공 콜백”을 준다고 가정했을 때
  * 결제가 PENDING → COMPLETED 로 바뀌는지
  * 비동기 이벤트를 통해 주문이 CONFIRMED 로 바뀌는지
* PG가 “실패 콜백”을 준다고 가정했을 때
  * 결제가 PENDING → FAILED 로 바뀌는지
  * 주문이 CANCELLED 로 정리되는지
* 이미 COMPLETED 상태인 결제에\
  성공 콜백이 또 와도 아무 변화가 없는지
* 이미 FAILED 상태인 결제에\
  실패 콜백이 또 와도 무시되는지

테스트 코드에서 콜백 요청을 직접 날리고,\
그 결과로 도메인 상태만 확인합니다.

서킷 브레이커 테스트가 **“외부 실패에 대한 시스템의 반응”을 고려해야한다면,**

콜백 테스트는 **“외부 신호에 대한 도메인 상태 전이와 멱등성” 의 관점에서 고려해야합니다.**

### 1-5. 왜 테스트 코드였는지

이런 테스트는 Postman으로도 어느 정도 흉내는 낼 수 있습니다.\
다만 실제로 해보면 한계가 분명한 접근방법 입니다.

* 실패율, 상태 전이 같은 시나리오를\
  손으로 맞추기가 어렵다 (돌발 상황 발생)
* 같은 시나리오를 여러 번 반복해서 돌리기 힘들고 \
  (동일한 결과 X)
* 서킷 브레이커 메트릭과 DB 상태를\
  한 번에 검증할 수 없는 애매함.

반대로, 테스트 코드 + Mock을 사용하면

* 실패/성공/예외/지연 패턴을 코드로 컨트롤할 수 있음
* 서킷 브레이커 상태, 메트릭, 결제/주문 도메인 상태를\
  하나의 테스트 안에서 같이 확인 가능
* CI에 올려서, 설정이 바뀌거나 코드가 수정되어도\
  계속 자동으로 검증할 수 있습니다.

정리하면, 테스트 기법을 이용해 제가 검증하고 싶은걸 검증하는 최선의 방법을 적용했습니다.

&#x20;좋습니다, 구조가 명확해서 2장 쓰기 딱 좋은 상태입니다.\
지금 코드 기준으로 2장을 “설정 + 실제 흐름”에 맞춰 정리해 볼게요.\
(테스트 코드는 3장에서 다루면 좋을 것 같고, 여기선 “테스트 대상이 되는 흐름”까지 잡아두는 느낌으로 가겠습니다.)

***

## 2장. Resilience4j 서킷 브레이커를 어디에, 어떻게 걸었는가 <a href="#id-2-resilience4j" id="id-2-resilience4j"></a>

1장에서 기준을 통해 2장에서는 실제 코드 기준으로 **서킷 브레이커를 어디에 적용했고, 어떤 역할을 맡겼는지**를 \
먼저 정리하겠습니다. 이게 나중에 테스트 코드를 이해할 때 기준선이 됩니다.

### 2-1. 서킷 브레이커는 `PaymentFacade` 에 적용

이번 구조에서 서킷 브레이커는 **응용 계층의 유즈케이스**에 붙어 있습니다.

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentFacade {

    private final PaymentService paymentService;
    private final PaymentProcessor paymentProcessor;
    private final PaymentValidator paymentValidator;
    private final UserService userService;
    private final OrderService orderService;

    @CircuitBreaker(name = "pgClient", fallbackMethod = "processPaymentFallback")
    @Transactional
    public PaymentInfo processPayment(PaymentCommand command) {
        UserEntity user = userService.getUserByUsername(command.username());

        OrderEntity order = orderService.getOrderByIdAndUserId(command.orderId(), user.getId());
        paymentValidator.validateOrderForPayment(order, command.amount());

        // PG 결제 처리 (여기에서 PG 호출 + PENDING 생성)
        PaymentEntity payment = paymentProcessor.processPgPayment(user, command);

        return PaymentInfo.from(payment);
    }

    @SuppressWarnings("unused")
    private PaymentInfo processPaymentFallback(PaymentCommand command, Throwable t) {
        log.error("PG 서비스 장애 또는 타임아웃, 결제 요청 실패 처리 - exception: {}, message: {}",
                t.getClass().getSimpleName(), t.getMessage(), t);

        UserEntity user = userService.getUserByUsername(command.username());

        PaymentEntity failed = paymentService.createFailedPayment(
                user,
                command,
                "결제 시스템 응답 지연으로 처리되지 않았습니다. 다시 시도해 주세요."
        );

        return PaymentInfo.from(failed);
    }
}
```

여기서 포인트는:

* **서킷 브레이커는 PG 호출 자체가 아니라 “유스케이스” 단에 붙여 둠**
  * `processPgPayment()` 내부에서 실제 PG 호출이 일어님,
  * 실패/타임아웃에 대한 최종 정책은 Facade 단에서 결정.
* Fallback에서는 **PG 호출 자체를 포기하고, FAILED 결제 건을 생성**.
  * 이때 결제 상태는 바로 `FAILED`로 떨어짐,
  * 실제 결제 여부는 나중 PG 콜백/조회로 재검증

테스트에서는 이 `processPayment()`와 `processPaymentFallback()`이\
서킷 브레이커 설정에 맞춰 어떻게 호출되는지를 검증하게 됩니다.

### 2-2. 실제 PG 호출은 도메인 서비스 + Gateway로 캡슐화

PG 호출은 `PaymentProcessor`와 `PgGateway`를 통해 감쌌습니다.

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentProcessor {

    private final PaymentService paymentService;
    private final PaymentValidator paymentValidator;
    private final PgGateway pgGateway;

    public PaymentEntity processPgPayment(UserEntity user, PaymentCommand command) {
        log.info("PG 결제 처리 시작 - orderId: {}, username: {}, amount: {}",
                command.orderId(), user.getUsername(), command.amount());

        // 1. PENDING 상태 결제 생성
        PaymentEntity pendingPayment = paymentService.createPending(user, command);

        // 2. PG 결제 요청
        PgPaymentResponse pgResponse = pgGateway.requestPayment(
                user.getUsername(),
                PgPaymentRequest.of(command)
        );

        // 3. PG 응답 검증
        paymentValidator.validatePgResponse(pgResponse);

        // 4. transactionKey 업데이트
        pendingPayment.updateTransactionKey(pgResponse.transactionKey());

        log.info("PG 결제 요청 완료 - orderId: {}, transactionKey: {}, status: {}, 콜백 대기 중",
                command.orderId(), pgResponse.transactionKey(), pgResponse.status());

        return pendingPayment;
    }
}
```

그리고 실제 외부 HTTP 호출은 `PgGatewayAdapter` + `PgClient(Feign)` 조합으로 구현 했습니다.

```java
public interface PgGateway {
    PgPaymentResponse requestPayment(String username, PgPaymentRequest request);
    PgPaymentResponse getPayment(String username, String transactionKey);
}

@Component
@RequiredArgsConstructor
public class PgGatewayAdapter implements PgGateway {
    private final PgClient pgClient;

    @Override
    public PgPaymentResponse requestPayment(String username, PgPaymentRequest request) {
        return pgClient.requestPayment(username, request);
    }

    @Override
    public PgPaymentResponse getPayment(String username, String transactionKey) {
        return pgClient.getPayment(username, transactionKey);
    }
}
```

테스트 관점에서 보면:

* **서킷 브레이커 테스트에서는 `PgGateway`/`PgClient`를 Mock으로 대체**하고
* `PaymentFacade.processPayment()`를 호출했을 때 아래 3 항목만 고려 하면됩니다.
  * 서킷 브레이커 상태 변화
  * fallback 호출 여부
  * 생성된 결제 상태(PENDING/FAILED).

### 2-3. Resilience4j 설정은 `pgClient` 인스턴스 하나로 집중

설정은 다음과 같이 잡았습니다.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      pgClient:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - feign.FeignException.ServiceUnavailable
          - feign.FeignException.InternalServerError
          - java.util.concurrent.TimeoutException
          - feign.RetryableException

  retry:
    instances:
      pgClient:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - feign.FeignException.ServiceUnavailable
          - feign.FeignException.InternalServerError
          - java.lang.RuntimeException
        # 타임아웃은 재시도하지 않음
```

여기서 테스트에 영향을 주는 핵심 포인트만 짚어보면:

* `minimum-number-of-calls: 5`\
  → 5번 이상 호출해야 실패율 계산 시작
* `sliding-window-size: 10`\
  → 최근 10건 기준으로 실패율 계산
* `failure-rate-threshold: 50`\
  → 실패율이 50%를 넘으면 OPEN
* `wait-duration-in-open-state: 60s`\
  → OPEN 상태는 최소 60초 유지
* `permitted-number-of-calls-in-half-open-state: 3`\
  → HALF\_OPEN 상태에서 3건만 시험 호출
* `record-exceptions`\
  → 어떤 예외를 “실패”로 기록할지 정의

테스트에서는 이 숫자들을 전제로:

* “10번 중 6번 실패 → OPEN이 되는가”
* “호출 5번 이하에서는 실패율이 계산되지 않는가”
* “OPEN 상태에서 호출 시, PG Mock은 아예 안 불리는가”

같은 시나리오를 만들어서 구현을 했습니다.

### 2-4. 콜백 쪽 흐름은 별도의 트랜잭션으로 정리

PG 콜백을 처리하는 쪽은 서킷 브레이커 대상이 아닙니다.\
여기는 “PG가 콜백을 줬다”는 전제하에 도메인 상태 전이를 다룹니다.

```java
@Transactional
public void handlePaymentCallback(PaymentV1Dtos.PgCallbackRequest request) {
    log.info("결제 콜백 처리 시작 - transactionKey: {}, status: {}, orderId: {}",
            request.transactionKey(), request.status(), request.orderId());

    PaymentEntity payment = paymentService.getByTransactionKey(request.transactionKey());

    if (payment.getPaymentStatus() != PaymentStatus.PENDING) {
        log.warn("이미 처리된 결제 - transactionKey: {}, currentStatus: {}",
                request.transactionKey(), payment.getPaymentStatus());
        return;
    }

    UserEntity user = userService.getUserById(payment.getUserId());
    OrderEntity order = orderService.getOrderByIdAndUserId(payment.getOrderId(), payment.getUserId());

    PgPaymentResponse pgData = paymentProcessor.verifyPaymentFromPg(
            request.transactionKey(),
            user.getUsername()
    );

    paymentValidator.validateCallbackData(payment, request, pgData);
    paymentValidator.validateOrderAmount(order, pgData.amount());

    paymentService.processPaymentResult(payment, request);
}
```

테스트에서는 다음을 검증 했습니다.

* PENDING → COMPLETED / FAILED 전이
* 이미 COMPLETED/FAILED 상태에서 콜백이 왔을 때 무시되는지
* `PaymentService.processPaymentResult()`에서 이벤트 발행까지 되는지



## 3장. 서킷 브레이커와 콜백을 테스트로 검증한 핵심 패턴들

테스트 코드로 제가 확인하고 싶었던 건 서킷 브레이커의 역할을 제대로 하고 있는가 입니다.

1. 서킷 브레이커가 **설정대로 열리고(OPEN)**
2. **열린 상태에서 외부 호출을 막고 Fallback만 타고**
3. **HALF\_OPEN → CLOSED 복구**가 정상 동작

아래 코드는 실제 테스트에서 사용한 패턴을 단순화해서 정리한 예시입니다.

***

#### 3-1. 실패율로 OPEN 전환 테스트

```java
@SpringBootTest
class PaymentCircuitIntegrationTest {

    @Autowired
    CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    PaymentFacade paymentFacade;

    @MockitoBean
    PgClient pgClient; // 실제 PG 대신 Mock

    @Test
    void 실패율이_threshold_를_넘으면_OPEN_상태가_된다() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("pgClient");

        // 5번 호출 중 3번 실패 → 실패율 60%
        given(pgClient.requestPayment(anyString(), any()))
                .willReturn(success())          // 1
                .willReturn(success())          // 2
                .willThrow(pgException())       // 3
                .willThrow(pgException())       // 4
                .willThrow(pgException());      // 5

        for (int i = 0; i < 5; i++) {
            try {
                paymentFacade.processPayment(testCommand());
            } catch (Exception ignored) {
            }
        }

        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }
}
```

> 핵심: **실제 PG 없이**, Mock으로 “성공 2 + 실패 3”을 강제로 만들어 `failureRateThreshold`를 넘겼을 때 OPEN 되는지 확인.

***

#### 3-2. OPEN 상태에서 Fallback만 타는지 테스트

```java
@Test
void OPEN_상태에서는_PG_호출_없이_Fallback으로_FAILED_결제가_생성된다() {
    CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("pgClient");
    cb.transitionToOpenState(); // 강제로 OPEN

    PaymentCommand command = testCommand();

    PaymentInfo result = paymentFacade.processPayment(command);

    // PG는 아예 호출되지 않음
    then(pgClient).shouldHaveNoInteractions();

    // Fallback 결과: FAILED 결제
    assertThat(result.status()).isEqualTo(PaymentStatus.FAILED);
}
```

> 핵심: **서킷이 열린 상태에서는 PG는 건드리지 않고**, `processPaymentFallback()`만 타서 FAILED 결제를 만드는지 검증.

***

#### 3-3. HALF\_OPEN → CLOSED 복구 테스트

```java
@Test
void HALF_OPEN_상태에서_3번_연속_성공하면_CLOSED로_복구된다() {
    CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("pgClient");

    // 먼저 OPEN 상태로
    cb.transitionToOpenState();

    // HALF_OPEN 전환
    cb.transitionToHalfOpenState();
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.HALF_OPEN);

    // HALF_OPEN에서 3번 모두 성공
    given(pgClient.requestPayment(anyString(), any()))
            .willReturn(success())
            .willReturn(success())
            .willReturn(success());

    for (int i = 0; i < 3; i++) {
        PaymentInfo info = paymentFacade.processPayment(testCommand());
        assertThat(info.status()).isEqualTo(PaymentStatus.PENDING);
    }

    // CLOSED 복구
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
}
```

> 핵심: `permitted-number-of-calls-in-half-open-state = 3` 기준으로 HALF\_OPEN에서 3번 연속 성공 시 CLOSED로 돌아오는지 테스트.

***

### 정리

실제 PG를 붙이지 않고, Mock만 가지고 다음을 확인했습니다.

* 서킷 브레이커가 **CLOSED → OPEN → HALF\_OPEN → CLOSED**로 설정한 대로 움직이는지
* OPEN/HALF\_OPEN에서 **PG 호출을 막고 Fallback으로만 처리하는지**

결론적으로,테스트를 기반으로 한 검증을 통해 적어도 제가 정의한 실패·복구·콜백 시나리오 안에서는 \
Resilience4j와 결제/주문 로직이 의도대로 움직이고 있다는걸 경험했습니다.
