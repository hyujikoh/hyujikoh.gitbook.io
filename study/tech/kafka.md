# Kafka 이벤트 파이프라인 구축: 로컬 이벤트에서 글로벌로

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

***

## 1. Kafka 보장 원칙: At-Least-Once + At-Most-Once <a href="#id-1-kafka---at-least-once--at-most-once" id="id-1-kafka---at-least-once--at-most-once"></a>

Kafka를 도입하면서 가장 먼저 고민했던 건 **보장 수준**이었습니다.\
Kafka는 기본적으로 **At-Least-Once**를 제공하는데, 모놀로직에서는 신경 안 썼던 게 갑자기 중요해졌습니다.

### Kafka의 고려해야할 기준

1. **At-Least-Once** (기본): 메시지 유실 절대 NO, 중복 가능 \
   → Producer 문제: DB 변경 OK + Kafka 실패 = 이벤트 유실 😱
2. **At-Most-Once**: 중복 절대 NO, 유실 가능 \
   → Consumer 문제: Kafka 재전달 = 같은 이벤트 2번 처리 😱
3. **Exactly-Once**: 유실도 중복도 NO (Transactions 필요) \
   → 현실: 조합으로 구현 (Producer 최소1번 + Consumer 최대1번)

**기존 모놀로직 구조**에서는 이런 보장 수준을 고려하지 않았습니다.\
`PaymentEntity.completeWithEvent()`에서 `registerEvent()` 처리하면 간단했습니다.

```java
public void completeWithEvent() {
    complete();
    registerEvent(new PaymentCompletedEvent(/*...*/));  // 같은 트랜잭션
}
```

하지만 **격리된 서비스**로 관심사를 나누면서 신경써야할 사항이 생겼습니다.

&#x20;카프카 활용의 핵심은 메세지를 잃지 않고, 단 한번만 처리되게 보장할 수 있는가 입니다.



#### **Producer → Broker**

* **어떻게든 발행 (At Least Once)**
* `Producer` 는 네트워크 지연, 장애가 있어도 메세지를 최소 한 번은 `Broker` 에 기록되도록 보장.
* 즉 무슨일어나도 데이터가 전송 되지 못하는 일은 있어서는 안되는 기준을 가집니다.



#### **Consumer ← Broker**

* **어떻게든 한 번만 처리 (At Most Once)**
* `Consumer` 는 같은 메세지가 여러 번 오더라도, 멱등하게 처리하여 최종 결과는 단 한번만 반영되도록 보장해야 합니다.
* 해당 컨텍스트에서는 여러번 들어와도 한번만 처리되게 하는것이 목표 입니다.&#x20;



이렇게 했을때 다음과 같이 가정을 하였습니다.

1. `Producer`  에서 발행을 할때는 여러번 발행하는건 중요하지 않고, 못보냈을 경우가 어떤것이 있는가?
   1. 브로커의 장애가 났을때 전송 자체 실패 (외부 요인으로 인한 전송 실패)
   2. 내부 어플리케이션이 장애가 나서 이벤트 자체가 호출이 안된다.
      1. 만약 비즈니스에서 발생되는 로직이라고 하면 카프카 발행 오류가 아니기 때문에 책임 X
      2. 카프카 를 발행하는 내부 로직에서 발생에 오류가 발생 (RTE, NPE  등 내부 비즈니스 오류) 경우 책임 O
2. `Consumer` 의 경우 데이터를 받기 위해 가장 먼저 해야하는것은 브로커에 데이터를 받았을때가 첫번째 스텝 입니다.
   1. 제 3의 관점에서 보면 가장 마지막 후처리 이기때문에 앞선에서 처리가 잘못되도 책임 분명이 명확하게 구성이 됩니다.
   2. 즉 컨슈머의 가장 중요한건 데이터를 잘 받아와서 저장이 되어있는지? 또한 동일한 이벤트를 네트워크 이슈로 여동시에 왔을때도 1건만 처리를 하였는지 책임이 잘 정리가 되어있는지?



### 처음 시도: 의식의 흐름대로 구성

프로듀서: 기존 커머스 플랫폼 API (사용자 API)

* PaymentCompletedEvent, LikeChangedEvent, ProductViewEvent 발행

컨슈머: 백오피스 서비스 (통계/관리용)

* 상품의 구매, 좋아요, 조회에 대한 상품 Metrics 집계

토픽 구성:

* catalog-events (상품 관련): 파티션 1개
* order-events (주문 관련): 파티션 1개



***

## 1장. Kafka 보장 원칙: At-Least-Once + At-Most-Once <a href="#id-1-kafka---at-least-once--at-most-once" id="id-1-kafka---at-least-once--at-most-once"></a>

Kafka를 도입하면서 가장 먼저 고민했던 건 **보장 수준**이었습니다. 외부 데이터 파이프라인을 구성하면서, \
단일 서비스에서 고려하지 않았던게 갑자기 중요해졌습니다.

### Kafka의 고려해야할 기준

```
textAt-Least-Once (기본): 메시지 유실 절대 NO, 중복 가능
→ Producer 문제: DB 변경 OK + Kafka 실패 = 이벤트 유실 

At-Most-Once: 중복 절대 NO, 유실 가능
→ Consumer 문제: Kafka 재전달 = 같은 이벤트 2번 처리 

Exactly-Once: 유실도 중복도 NO (Transactions 필요)
→ 현실: 조합으로 구현 (Producer 최소1번 + Consumer 최대1번)
```

단일 서비스 구조에서는 이런 보장 수준을 고려하지 않았습니다. `PaymentEntity.completeWithEvent()`에서 `registerEvent()` 처리하면 간단했습니다.

```java
public void completeWithEvent() {
    complete();
    registerEvent(new PaymentCompletedEvent(/*...*/)); // 같은 트랜잭션
}
```

하지만 격리된 서비스로 관심사를 나누면서 신경써야할 사항이 생겼습니다. **카프카 활용의 핵심은 메시지를 잃지 않고, 단 한번만 처리되게 보장할 수 있는가**입니다.

#### Producer → Broker: 어떻게든 발행 (At Least Once)

Producer는 네트워크 지연, 장애가 있어도 메시지를 **최소 한 번은 Broker에 기록되도록 보장**해야 합니다.

**즉 무슨 일이 있어도 데이터가 전송되지 못하는 일은 절대 안 되는 기준**을 가져야 합니다.

**Producer에서 못 보낸 경우는 어떤 것인가?**

1. **브로커의 장애가 났을 때 전송 자체 실패** (외부 요인으로 인한 전송 실패)
2. **내부 애플리케이션이 장애가 나서 이벤트 자체가 호출이 안 됨**
   * 만약 비즈니스 로직에서 발생되는 것이라면 카프카 발행 오류가 아니므로 **책임 X**
3. **카프카를 발행하는 내부 로직에서 오류 발생** (RTE, NPE 등 내부 비즈니스 오류)
   * 이 경우 **책임 O**

**그래서 이번 카프카 적용 및 구성을 할떄 Producer에서 발행할 때 여러번 발행하는 건 우선사항이 아니였습니다.** \
중요한 건 **절대 못 보낸 경우를 방지**하는 것입니다.

#### Consumer ← Broker: 어떻게든 한 번만 처리 (At Most Once)

Consumer는 같은 메시지가 여러 번 오더라도, **멱등하게 처리하여 최종 결과는 단 한 번만 반영되도록 보장**해야 합니다.

**해당 컨텍스트에서는 여러번 들어와도 한 번만 처리되게 하는 것이 목표**입니다.

**Consumer의 가장 중요한 책임**:

1. **브로커에서 데이터를 잘 받아와서 저장이 되어 있는지?**
2. **동일한 이벤트를 네트워크 이슈로 동시에 왔을 때도 1건만 처리했는지?**

**제3의 관점에서 보면 Consumer는 가장 마지막 후처리**이기 때문에 앞 단계에서 잘못돼도 책임이 명확하게 구분됩니다.

{% hint style="info" %}
이렇게 했을때 만약 컨슈머, 프로듀서 둘중 중요도를 따지고 신경써야한다면 컨슈머가 더 중요하다고 생각합니다.

결국 데이터를 받아서 처리하고, 가공 및 저장하는, 중복 처리 방지의 책임은 컨슈머에 있기 때문입니다.&#x20;
{% endhint %}

### 실제 구현에서 겪은 문제

**처음 시도: 의식의 흐름대로 kafkaTemplate 직통**

```
프로듀서: 기존 커머스 플랫폼 API
- PaymentCompletedEvent, LikeChangedEvent, ProductViewEvent 발행

컨슈머: 백오피스 서비스
- 상품의 구매, 좋아요, 조회에 대한 Metrics 집계

토픽: catalog-events(파티션 1개), order-events(파티션 1개)
```

```java
@EventListener
public void handlePaymentCompleted(PaymentCompletedEvent event) {
    orderFacade.confirmOrder(); // DB 변경
    kafkaTemplate.send("catalog-events", toJson(event)); // 바로 발행
}
```

이렇게 기존 이벤트에서 그대로 카프카를 발행한다 했을때 데이터는 잘 가는것은 확인이 되었지만.

브로커를 끄고 이벤트를 발행했을때 문제가 확인되었습니다.



이렇게 된다면 kafkaTemplate.send() = 네트워크 실패, 또는 서비스 내 오류 = 유실 위험 으로 직결됩니다.\
또한 카프카의 발행의 책임을 기존 도메인 이벤트에서 바로 연결해서 발행하는 방식이 과도한 책임을 가진것\
같다는 생각이 들었어서 실패에 대해서 재시도하고 최소 1번의 은 발행 보장을 해야하기 위해 \
아웃박스 패턴을 추가하였습니다.

#### Outbox 패턴 적용

즉시 발행하는 로직에서 아웃박스 테이블을 따로 만들어서 이벤트가 발생하는 즉시 데이터를 저장을 하였습니다.

```java
@Entity
@Table(name = "outbox_event")
public class OutboxEventEntity {
    @Id
    private String eventId;
    
    @Column(name = "topic_name")
    private String topicName;
    
    @Column(name = "payload_json")
    private String payloadJson;
    
    @Enumerated(EnumType.STRING)
    private EventStatus status; // READY, SENT, FAILED
}
```

**OrderEventHandler에서는 아래와 같은 로직을 구성하였습니다.**

```java
@Component 
@Slf4j 
@RequiredArgsConstructor
public class OrderEventHandler {
    
    @Async 
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        Long orderId = event.orderNumber();
        
        // 1. 주문 확정 처리
        executeSafely("PAYMENT_COMPLETED", orderId,
            () -> orderFacade.confirmOrderByPayment(orderId));
        
        // 2. Kafka 이벤트용 Outbox 저장
        try {
            savePaymentSuccessToOutbox(event);
        } catch (Exception e) {
            log.error("결제 완료 이벤트 Outbox 저장 실패 - orderNumber={}, userId={}", 
                orderId, event.userId(), e);
            // 별도 실패했을떄 저장 
        }
    }
}
```

이것들을 OutboxRelayScheduler 스케줄러를 통해 배치 단위로 처리 하는 로직을 구성했습니다.

```java
@Scheduled(fixedDelay = 1000)
public void relayEvents() {
    List<OutboxEventEntity> readyEvents = 
        outboxRepository.findByStatus(READY, 1000);
    
    for (OutboxEventEntity event : readyEvents) {
        kafkaTemplate.send(event.topicName(), event.payload());
        event.markAsSent(); // 상태 업데이트
    }
}
```

Outbox 저장 실패를 했을때 별도 스케줄러를 통한 재시도를 통해 실패한 부분을 **최소1번 보장 하도록 하였습니다.**

**지금까지 만든것과 브로커 구성은 다음과 같이 구성하였습니다**&#x20;

```
프로듀서: 커머스 API → Outbox → Kafka
토픽: catalog-events, order-events (파티션 1개)
```

이제 이렇게 프로듀서에서 발행하는 부분을 브로커에서 멱등성을 보장한 과정을 구현해야했습니다.



## 2장. Consumer 멱등성: 중복 처리 방지 <a href="#id-2-consumer" id="id-2-consumer"></a>

Producer에서 Outbox 패턴으로 최소 1번 보장을 확보했지만, Consumer 쪽에서 새로운 문제가 생겼습니다.

Kafka의 At-Least-Once 특성상 **같은 메시지가 여러 번 전달될 수 있습니다**. 네트워크 재전달, Consumer 재시작 등으로 인해 동일 이벤트가 중복 수신되는 상황이 발생합니다.

**상황 예시**:

```
결제 완료 이벤트 1개 발행 → Kafka 재전달로 2번 수신
→ 상품 메트릭이 2배 증가 (판매량, 조회수 등)
→ 캐시가 잘못 업데이트
→ 분석 데이터 이슈
```

**앞에서 이야기 했던것 처럼 Consumer의 가장 중요한 책임은:**

1. 브로커에서 데이터를 잘 받아와서 저장
2. **동일한 이벤트를 네트워크 이슈로 동시에 왔을 때도 1건만 처리**

#### 첫 시도: 예외 기반 중복 체크 (안티패턴)

가장 직관적인 방법으로 `event_handled` 테이블에 이벤트 ID를 저장하고 중복 체크를 했습니다.

```java
// 이벤트 관리 엔티티
@Entity
@Table(name = "event_handled")
@NoArgsConstructor(access = lombok.AccessLevel.PROTECTED)
@Getter
public class EventEntity {
    @Id
    @Column(name = "event_id", length = 36, nullable = false)
    private String eventId;

    @Column(name = "handled_at", nullable = false)
    private ZonedDateTime handledAt;

    private EventEntity(final String eventId, final ZonedDateTime handledAt) {
        this.eventId = eventId;
        this.handledAt = handledAt;
    }

    public static EventEntity create(final String eventId) {
        return new EventEntity(eventId, ZonedDateTime.now());
    }
}

```

```java
@KafkaListener(topics = "catalog-events")
public void handleCatalogEvents(String payload) {
    String eventId = extractEventId(payload);
    
    try {
        // 중복 체크 + 처리
        eventHandledRepository.save(EventHandledEntity.of(eventId));
        metricsService.incrementMetrics(payload); // 실제 메트릭 집계
    } catch (DataIntegrityViolationException e) {
        // 이미 처리된 이벤트 → 무시
        log.debug("중복 이벤트 무시: {}", eventId);
    }
}
```

**이렇게 단순히 처리했을때**&#x20;

* **예외 오버헤드**: 모든 이벤트마다 예외 발생 가능성이 존재
* **DB 부하 증가**: 초당 수천 이벤트 → DB 쓰기가 매번 발생 (20000개 이벤트 매번 db 처리)

#### 2단계 멱등성 처리 (메모리 + DB)

&#x20;DB 에서 바로 처리하던 방식 메모리 캐시(1차) + DB 확인(2차)로 변경했습니다.

```java
@Service
@Slf4j
public class MetricsService {
    
    // 메모리 캐시
    private final ConcurrentHashMap<String, Boolean> processedEvents 
        = new ConcurrentHashMap<>();
    
    private final EventHandledRepository eventHandledRepository;
    
    public boolean tryMarkHandled(String eventId, String payload) {
        // 1단계: 메모리 캐시 확인 (빠른 경로)
        if (processedEvents.containsKey(eventId)) {
            log.debug("이미 처리된 이벤트 (메모리): {}", eventId);
            return false;
        }
        
        // 2단계: DB 확인 (느린 경로)
        if (eventHandledRepository.existsById(eventId)) {
            processedEvents.put(eventId, true); // 캐시 업데이트
            log.debug("이미 처리된 이벤트 (DB): {}", eventId);
            return false;
        }
        
        // 3단계: 새로운 이벤트 처리
        try {
            processMetrics(payload); // 실제 메트릭 집계
            eventHandledRepository.save(EventHandledEntity.of(eventId));
            processedEvents.put(eventId, true); // 캐시 추가
            return true;
        } catch (Exception e) {
            log.error("이벤트 처리 실패: {}", eventId, e);
            return false;
        }
    }
    
    private void processMetrics(String payload) {
        // 상품 메트릭 집계, 캐시 업데이트 등
        // 동시성 제어는 다음 장에서
    }
}
```

#### Consumer 설정: 배치 + 수동 커밋

중복 체크 성능을 극대화하기 위해 Consumer도 배치 처리로 변경했습니다.

```java
@KafkaListener(
    topics = {"catalog-events"}, 
    containerFactory = "batchListenerContainerFactory"
)
public void handleCatalogEventsBatch(List<ConsumerRecord<String, String>> records) {
    for (ConsumerRecord<String, String> record : records) {
        String eventId = extractEventId(record.value());
        if (safelyProcess(eventId, record.value())) {
            log.debug("신규 이벤트 처리 완료: {}", eventId);
        }
    }
    // 배치 단위 수동 커밋
    acknowledgment.acknowledge();
}
```

**KafkaConfig 주요 설정**:

```java
@Bean(name = "batchListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> batchFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    factory.setBatchListener(true);
    factory.setConcurrency(3);
    return factory;
}
```

**Consumer 설정 포인트**:

```yaml
max.poll.records=3000 (배치 크기)
enable.auto-commit=false (수동 커밋)
session.timeout.ms=60s (안정적 재연결)
```

이제 컨슈머에 대한 멱등성 처리를 하였으니, **동시성 제어**에 대해서 고민을 하게되었습니다.



## 3장. 동시성 제어:메모리 락 <a href="#id-3---redis" id="id-3---redis"></a>

멱등성으로 중복 처리는 해결했지만, **동시성 문제**가 새롭게 등장했습니다. \
같은 상품에 대한 좋아요/판매 이벤트가 동시에 들어오면 메트릭이 꼬일 수 있습니다.

#### 메모리 락 도입

동시성을 처리할수 있는 방법에 대해서는 이전에 포스트에서 각 기법에 대해서 고민해본 결과\
**ConcurrentHashMap + ReentrantLock** 를 적용하였습니다. \
(관련 포스트 [https://hyujikoh-blog.gitbook.io/blog/study/tech/concurrency\_idempotent](https://hyujikoh-blog.gitbook.io/blog/study/tech/concurrency_idempotent))

```java
@Service
public class MetricsService {
    private final ConcurrentHashMap<Long, ReentrantLock> productLocks 
        = new ConcurrentHashMap<>();
    
    public void applyLikeDelta(Long productId, int delta) {
        ReentrantLock lock = productLocks.computeIfAbsent(productId, 
            k -> new ReentrantLock());
        
        if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                metricsTransactionService.applyLikeDelta(productId, delta);
            } finally {
                lock.unlock();
            }
        } else {
            log.warn("락 획득 실패 - 스킵: {}", productId);
        }
    }
}
```

별도로 lock 누수에 대해서 아래와 같이 방어 메소드를 다음과 같이 추가하였습니다.

```java
public void cleanupLocks() {
    productLocks.entrySet().removeIf(entry -> {
        ReentrantLock lock = entry.getValue();
        return !lock.isLocked() && !lock.hasQueuedThreads();
    });
}
```





## 마무리: Kafka 적용 회고 <a href="#kafka--1" id="kafka--1"></a>

**앞서 이야기 했던걸 다시한번 이야기 해보면**

```
🤔 백오피스/분석/추천 시스템도 같은 이벤트 소비를 동일한 어플리케이션에서 해야 하나?
🤔 많은 트래픽을 감당한다고 했을 때도 잘 돌아가는가?
🤔 다른 서비스와의 느슨한 결합은 어떤 방식으로 해야 할까?
```

이벤트는 누군가 관심 있는 과거의 사건 이라는 이전 글의 철학을 Kafka로 확장하면서, \
가장 먼저 고민했던 건 **보장 수준**이었습니다.

#### 이번 여정에서 얻은 핵심

**1단계**: 직통 `kafkaTemplate.send()` → **유실 + 중복** 문제 발견

* DB 변경 성공 + Kafka 실패 = 이벤트 유실 가능성 존재
* At-Least-Once 재전달 = 메트릭 2배 증가

**2단계**: Outbox 패턴 → **Producer 최소1번 보장**

* orderFacade.confirmOrder() + Outbox 저장 = 같은 트랜잭션
* 브로커 다운되어도 DB에 이벤트 남음 → 배치 기반 재시도 로직을 통해 최소 한번 발행 보장

**3단계**: Consumer 멱등성 → **최대 1번만 처리**

* 메모리 와 DB 를 통해 예외 처리에 대한 오버헤드 제거
* event\_handled 테이블로 추적성 확보
* 수동 커밋 과 배치를 통한 리소스 자원 최소화

#### 현재 아키텍처 흐름

```
커머스 API
  ↓ PaymentEntity.registerEvent()
  ↓ OrderEventHandler (@AFTER_COMMIT)
  ↓ Outbox 저장 (트랜잭션 내)
  ↓ OutboxRelayScheduler (1000개 배치)
  ↓ Kafka (catalog-events, order-events)
  ↓ 백오피스 배치 컨슈머 (3000개)
  ↓ 2단계 멱등성 + 메모리 락
  ↓ Metrics DB + Cache
```

#### 깨달은 교훈

**Kafka 도입 과정에서 가장 중요한 4가지**:

1. **보장 > 성능**: "메시지 유실"이 가장 큰 죄악. Outbox가 답.
2. **트랜잭션 경계**가 모든 걸 결정: DB 변경과 이벤트 저장은 반드시 같은 트랜잭션.
3. **Consumer가 더 중요**: Producer는 복구 가능하지만, Consumer가 최종 결과 책임짐.
4. **점진적 개선이 답**: 직통 → Outbox → 멱등성 → 배치 → 동시성. 한 번에 다 바꾸지 않음.

**특히 Consumer 멱등성에서 배운 것**:

```
text"제3의 관점에서 Consumer는 가장 마지막 후처리"
→ 앞 단계 실패해도 책임 명확
→ 브로커 데이터 잘 받고, 중복 1건만 처리
```

**현재 상황에 가장 적합한 아키텍처**를 지속적으로 검증하고 개선해 나가는 과정이 중요하다는 걸 \
이번 여정을 통해 깨달았습니다.
