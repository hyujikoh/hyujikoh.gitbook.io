---
hidden: true
---

# 프란츠 카프카

## 이 글을 작성하게된 이유: <a href="#undefined" id="undefined"></a>

지난주 스프링 로컬 이벤트로 주문 생성 메서드에서 관심사를 분리해봤습니다.\
결제 도메인에서 각각의 도메인의 행위를(재고 차감, 쿠폰 사용, 행동 추적) 이벤트 기반 분리를 하고나니\
응용 메서드가 한결 가벼워졌고, 각 도메인이 자기 관심사만 보는 구조가 자연스럽게 나왔습니다.

`PaymentEntity`에서 직접 `PaymentCompletedEvent`를 발행하고,\
`OrderEventHandler`와 `DataPlatformEventHandler`에서 각각 관심 있는 부분만 처리하도록 했습니다.



{% tabs %}
{% tab title="PaymentEntity" %}
```java
public class PaymentEntity extends BaseEntity {
        /**
     * 결제 완료 처리 (도메인 이벤트 발행)
     */
    public void completeWithEvent() {
        complete();

        // 도메인 이벤트 발행 (이벤트 핸들러에서 데이터 플랫폼 연동 처리)
        registerEvent(new PaymentCompletedEvent(
                this.transactionKey,
                this.orderNumber,
                this.userId,
                this.amount,
                this.cardType
        ));
    }


    /**
     * 결제 실패 처리 (도메인 이벤트 발행)
     */
    public void fail(String reason) {

        if (this.paymentStatus != PaymentStatus.PENDING) {
            throw new IllegalStateException(
                    String.format("PENDING 상태의 결제만 실패 처리할 수 있습니다. (현재 상태: %s)", this.paymentStatus)
            );
        }
        this.failureReason = reason;
        this.paymentStatus = PaymentStatus.FAILED;

        // 도메인 이벤트 발행 (이벤트 핸들러에서 데이터 플랫폼 연동 처리)
        registerEvent(new PaymentFailedEvent(
                this.transactionKey,
                this.orderNumber,
                this.userId,
                this.amount,
                this.cardType,
                reason
        ));
    }
    
}
```
{% endtab %}

{% tab title="OrderEventHandler" %}
```java
// 주문에 대한 이벤트 핸들러
@Component
@Slf4j
@RequiredArgsConstructor
public class OrderEventHandler {
@Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        Long orderId = event.orderNumber();
        Long userId = event.userId();

        // 1. 주문 확정 처리
        executeSafely("PAYMENT_COMPLETED", orderId, userId,
                () -> orderFacade.confirmOrderByPayment(orderId, userId));

        // 2. Kafka 이벤트용 Outbox 저장
        try {
            savePaymentSuccessToOutbox(event);
        } catch (Exception e) {
            log.error("결제 완료 이벤트 Outbox 저장 실패 - orderNumber={}, userId={}",
                    orderId, userId, e);
        }
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Long orderId = event.orderId();
        Long userId = event.userId();
        executeSafely("PAYMENT_FAILED", orderId, userId,
                () -> orderFacade.cancelOrderByPaymentFailure(orderId, userId));
    }

}
```
{% endtab %}

{% tab title="DataPlatformEventHandler" %}
```java
// 데이터 플랫폼 도메인에 대한 이벤트 핸들
@Component
@Slf4j
@RequiredArgsConstructor
public class DataPlatformEventHandler {

    /**
     * 주문 확정 데이터 플랫폼 전송 이벤트 처리
     */
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        try {
            log.debug("주문 확정 데이터 플랫폼 전송 시작 - orderNumber: {}", event.orderNumber());

            OrderDataDto orderData = new OrderDataDto(
                    event.orderId(),
                    event.orderNumber(),
                    event.userId(),
                    OrderStatus.CONFIRMED,
                    event.originalTotalAmount(),
                    event.discountAmount(),
                    event.finalTotalAmount(),
                    ZonedDateTime.now(),
                    "ORDER_CONFIRMED"
            );

            boolean success = dataPlatformClient.sendOrderData(orderData);

            if (success) {
                log.info("주문 확정 데이터 플랫폼 전송 성공 - orderNumber: {}", event.orderNumber());
            } else {
                log.warn("주문 확정 데이터 플랫폼 전송 실패 - orderNumber: {}", event.orderNumber());
            }

        } catch (Exception e) {
            log.error("주문 확정 데이터 플랫폼 전송 중 예외 발생 - orderNumber: {}", event.orderNumber(), e);
        }
    }

    /**
     * 주문 취소 데이터 플랫폼 전송 이벤트 처리
     */
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCancelled(OrderCancelledEvent event) {
        try {
            log.debug("주문 취소 데이터 플랫폼 전송 시작 - orderNumber: {}", event.orderId());

            OrderDataDto orderData = new OrderDataDto(
                    event.orderId(),
                    event.orderNumber(),
                    event.userId(),
                    OrderStatus.CANCELLED,
                    event.originalTotalAmount(),
                    event.discountAmount(),
                    event.finalTotalAmount(),
                    ZonedDateTime.now(),
                    "ORDER_CANCELLED"
            );

            boolean success = dataPlatformClient.sendOrderData(orderData);

            if (success) {
                log.info("주문 취소 데이터 플랫폼 전송 성공 - orderNumber: {}", event.orderId());
            } else {
                log.warn("주문 취소 데이터 플랫폼 전송 실패 - orderNumber: {}", event.orderId());
            }

        } catch (Exception e) {
            log.error("주문 취소 데이터 플랫폼 전송 중 예외 발생 - orderNumber: {}", event.orderId(), e);
        }
    }
    

```
{% endtab %}
{% endtabs %}

E2E 테스트까지는 잘 돌았지만, 실제 운영 환경을 생각해보니 한계가 뚜렷했습니다.

```
🤔 백오피스/분석/추천 시스템도 같은 이벤트 소비도 동일한 어플리케이션에서 해야하나?
🤔 많은 트래픽을 감당 한다고 했을때도 잘 돌아가는가?
🤔 다른 서비스와의 느슨한 결합은 어떤 방식으로 해야할까?
```

**관심사 분리는 됐지만, 결국 여러 서비스가 같은 사건을 공유해야 합니다.**\
그래서 이번에는 **Kafka 이벤트 파이프라인**으로 확장해보기로 했습니다.

이전 글에서는 로컬 이벤트에서 배운 "**누군가 관심 있는 과거의 사건**" 철학을,

좀더 확장성있게 했을때도 그대로 유지가 가능하도록 하는 과정을 글로 작성해봤습니다.



## 1. Kafka 보장 원칙: At-Least-Once + At-Most-Once <a href="#id-1-kafka---at-least-once--at-most-once" id="id-1-kafka---at-least-once--at-most-once"></a>

Kafka를 도입하면서 가장 먼저 고민했던 건 **보장 수준**이었습니다.\
Kafka는 기본적으로 **At-Least-Once**를 제공하는데, 이게 운영에서 어떤 의미인지 생각해보니 중요한 문제가 생겼습니다.

### Kafka의 고려해야할 기준

1. **At-Least-Once** (기본): 메시지 유실 절대 NO, 중복 가능
2. **At-Most-Once**: 중복 절대 NO, 유실 가능
3. **Exactly-Once**: 유실도 중복도 NO (Transactions 필요)

**기존 모놀로직 구조**에서는 이런 보장 수준을 고려하지 않았습니다.\
`PaymentEntity.completeWithEvent()`에서 `registerEvent()` 처리하면 간단했습니다.

하지만 **격리된 서비스**로 관심사를 나누면서 신경써야할 사항이 생겼습니다.



### 단순 무식하게 시작한 첫 구성

그래서 처음에는 **최대한 단순하게** 프로듀서와 컨슈머를 나눠봤습니다.

```markdown
프로듀서: 기존 커머스 플랫폼 API (사용자 API)
- PaymentCompletedEvent, LikeChangedEvent 발행

컨슈머: 백오피스 서비스 (통계/관리용)
- Metrics 집계, 사용자 패턴 통게 

토픽 구성:
- catalog-events (상품 관련): 파티션 1개
- order-events (주문 관련): 파티션 1개
```



```java
// 프로듀서: 기존 OrderEventHandler에서 확장
@EventListener
public void handlePaymentCompleted(PaymentCompletedEvent event) {
    // 1. 주문 확정 (기존 로직)
    orderFacade.confirmOrderByPayment(orderId, userId);
    
    // 2. Kafka용 Outbox 저장 (신규)
    savePaymentSuccessToOutbox(event);  // catalog-events 토픽
}

// 컨슈머: 백오피스에서 수신
@KafkaListener(topics = "catalog-events")
public void handleCatalogEvents(String payload) {
    // 상품 메트릭 집계, 캐시 업데이트
    metricsService.processCatalogEvent(payload);
}
```

**첫 목표**:

1. **로컬 이벤트** → **Kafka 글로벌 이벤트**로 확장 확인
2. 진짜 데이터가 수신이 되는지..?&#x20;
