---
hidden: true
---

# 우리 서비스는 Lock이 정말 필요한가? - 동시성 제어의 선택과 트레이드오프

> TL:DR&#x20;

## 서론. 이 글을 작성하게 된 이유 <a href="#undefined" id="undefined"></a>



**"정말 우리 서비스에 동시에 1000명이 같은 쿠폰을 쓸까? 이 복잡도는 과연 필요한 것일까?"**

주문 결제, 주문 취소 등 주문 서비스를 구현하면서 재고, 포인트, 쿠폰까지 여러 도메인의 정합성을 보장해야 했습니다. 동시성 문제를 해결하려면 Lock을 걸면 되지 않을까? 처음엔 그렇게 생각했습니다.

하지만 구현과 테스트를 반복하면서 계속 하나의 질문이 머릿속에 맴돌았습니다.

**"이 문제를 해결하는데, 이 방법이 정말 필요한가?"**

쿠폰에 비관적 락을 걸었더니 작동은 했지만 과한 것 같았고, 사용자 포인트 차감 과정에서는 데드락이 발생했으며, 좋아요 기능은 아예 락 없이 해결할 수 있었습니다. 세 가지 도메인에 대해서 의사결정을 하면서 Lock을 절대반지처럼 사용하지 않고자 했습니다.

이 글에서는 동시성 제어를 위해 제가 고민했던 것들을 공유하고자 합니다.

* 쿠폰: 비관적 락에서 낙관적 락으로 전환한 이유
* 사용자 포인트: 데드락과의 싸움, 락 획득 순서 통일
* 좋아요: Lock 없이 원자적 UPDATE
* 각 선택의 트레이드오프와 판단 기준

각 상황에 대해 이것이 맞다 라기 보다는, **어떤 기준으로 판단했는지 그 고민의 과정**을 담으려 했습니다. 결국 좋은 동시성 제어는 복잡한 락 기법을 많이 아는 게 아니라, 각 도메인의 특성과 요구사항을 명확히 이해하고 적절한 전략을 선택하는 것이라는 게 중요하다 생각합니다.

## 1장. 쿠폰: 비관적 락은 과했다 <a href="#id-1" id="id-1"></a>

### 문제 상황 <a href="#undefined" id="undefined"></a>

사용자가 주문 시 쿠폰을 사용하면, 해당 쿠폰은 더 이상 사용할 수 없어야 합니다. 만약 같은 사용자가 거의 오차가 없는 쿠폰 사용을 한다고 하면 중복으로 사용되지 않게 방지해야합니다.

### 처음의 선택: 비관적 락 <a href="#undefined" id="undefined"></a>

가장 먼저 떠오른 해결책은 비관적 락이었습니다. 쿠폰 자체가 금전적인 부분에 영향을 주는 도메인이기 때문에 읽는 순간 락을 잠그는 비용정도는 지출해도 되겠다 생각하고, 중복 사용을 확실히 막을 수 있을 것 같았습니다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT c FROM CouponEntity c WHERE c.id = :id AND c.userId = :userId")
Optional<CouponEntity> findByIdWithLock(Long id, Long userId);

```

아래와 같은 테스트 코드를 작성하여 동시성을 제어를 검증해보기도 했습니다.

<details>

<summary>동시성 테스트 코드</summary>

```java
@Test
@DisplayName("동일한 쿠폰을 동시에 사용하려고 하면 한 번만 성공한다")
void should_allow_only_one_order_when_using_same_coupon_concurrently() throws InterruptedException {
     // 테스트 를 위한 초기 값 설정
    
                // When - 두 사용자가 동시에 같은 쿠폰으로 주문 시도
            ExecutorService executor = Executors.newFixedThreadPool(2);
            
            // 사용자1 주문 스레드
            executor.submit(() -> {
                try {
                    startLatch.await(); // 동시 시작을 위한 대기
                    
                    OrderCreateCommand orderCommand = OrderCreateCommand.builder()
                        .username(users.get(0).username())
                        .orderItems(List.of(
                            OrderItemCommand.builder()
                                .productId(product.getId())
                                .quantity(1)
                                .couponId(sharedCoupon.getId())
                                .build()
                        ))
                        .build();
                    
                    OrderInfo result = orderFacade.createOrder(orderCommand);
                    successfulOrders.add(result);
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    failureCount.incrementAndGet();
                    exceptions.add(e);
                } finally {
                    endLatch.countDown();
                }
            });
            
            // 사용자1이 동일 쿠폰으로 두 번째 주문 시도
            executor.submit(() -> {
                try {
                    startLatch.await(); // 동시 시작을 위한 대기
                    
                    OrderCreateCommand orderCommand = OrderCreateCommand.builder()
                        .username(users.get(0).username())
                        .orderItems(List.of(
                            OrderItemCommand.builder()
                                .productId(product.getId())
                                .quantity(1)
                                .couponId(sharedCoupon.getId())
                                .build()
                        ))
                        .build();
                    
                    OrderInfo result = orderFacade.createOrder(orderCommand);
                    successfulOrders.add(result);
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    failureCount.incrementAndGet();
                    exceptions.add(e);
                } finally {
                    endLatch.countDown();
                }
            });
            
            // 동시 실행 시작
            startLatch.countDown();
            
            // 모든 스레드 완료 대기 (최대 10초)
            boolean completed = endLatch.await(10, TimeUnit.SECONDS);
            executor.shutdown();
            
            // Then
            assertThat(completed).isTrue(); // 타임아웃 없이 완료되어야 함
            assertThat(successCount.get()).isEqualTo(1); // 정확히 하나만 성공
            assertThat(failureCount.get()).isEqualTo(1); // 정확히 하나만 실패
}
```

</details>

이렇게 완성하고 끝내려고 했지만, 뭔가 신경쓰이는 부분이 있었습니다.

처음 제가 작성한 쿠폰 엔티티 로직 입니다.

```java


@Entity
@Table(name = "coupons")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CouponEntity extends BaseEntity {

    /**
     * 사용자 ID (users.id 참조)
     */
    @Column(name = "user_id", nullable = false)
    private Long userId;

    /**
     * 쿠폰 타입 (FIXED_AMOUNT: 정액 할인, PERCENTAGE: 배율 할인)
     */
    @Enumerated(EnumType.STRING)
    @Column(name = "coupon_type", nullable = false, length = 20)
    private CouponType couponType;

    /**
     * 정액 할인 금액 (정액 쿠폰용)
     */
    @Column(name = "fixed_amount", precision = 10, scale = 2)
    private BigDecimal fixedAmount;

    /**
     * 할인 비율 (배율 쿠폰용, 0-100)
     */
    @Column(name = "percentage")
    private Integer percentage;

    /**
     * 쿠폰 상태 (UNUSED: 미사용, USED: 사용됨)
     */
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private CouponStatus status = CouponStatus.UNUSED;
}
```

쿠폰 도메인 특징상 이미 **특정 사용자에게 발급된 쿠폰**이기도 했습니다.. `userId`가 쿠폰에 명시되어 있고, 그 사용자만 사용할 수 있습니다. 즉, 쿠폰을 처리하는 주체는 오직 그 사용자 한 명뿐입니다.

같은 사용자가 정확히 동시에 두 번 주문할 확률이 얼마나 될까? 라고 고민했을때 모든 쿠폰 조회를 직렬화하고, 락을 획득하고 해제하는 오버헤드를 감수해야 할까 라는 스스로의 물음에 비관적 락은 과하다는 결론을 내렸습니다.

### 낙관적 락으로 전환 <a href="#undefined" id="undefined"></a>

쿠폰은 단순히 상태만 바꾸는 게 아니라 할인 금액 계산, 유효성 검증 등 여러 로직이 필요했습니다. 엔티티의 데이터 원자성을 보장하면서도 가벼운 방법이 필요했습니다. 그래서 상대적으로 충돌이 오는걸 낙관적으로 생각해도 되겠다는 생각으로 낙관적 lock 을 구현을 하게 되었습니다.

```java
@Entity
@Table(name = "coupons")
public class CouponEntity extends BaseEntity {
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private CouponStatus status = CouponStatus.UNUSED;
    
    @Version  // 낙관적 락 추가
    private Long version;
    
    public void use() {
        if (this.status == CouponStatus.USED) {
            throw new IllegalStateException("이미 사용된 쿠폰입니다.");
        }
        this.status = CouponStatus.USED;
    }
}

```

#### 낙관적 락의 동작 원리 <a href="#undefined" id="undefined"></a>

`@Version`을 추가하면 JPA가 자동으로 버전 필드를 관리합니다. 동작 방식은 간단합니다.

**1. 엔티티를 조회할 때:**

```sql
SELECT id, user_id, status, version FROM coupons WHERE id = 1
-- 결과: version = 5
```

**2. 엔티티를 수정하고 커밋할 때:**

```sql
UPDATE coupons 
SET status = 'USED', version = 6
WHERE id = 1 AND version = 5  -- 조회했던 버전과 일치하는지 확인
```

만약 두 트랜잭션이 동시에 같은 쿠폰을 수정하려 한다면, 먼저 커밋된 트랜잭션이 version을 5에서 6으로 올립니다. 나중에 커밋하려는 트랜잭션은 "WHERE version = 5"로 찾으려 하지만, 이미 6으로 바뀌었기 때문에 UPDATE가 0건 실행됩니다. 이때 JPA는 OptimisticLockException을 던집니다.

이 방식의 핵심은 "최초 커밋만 인정하기" 입니다. 락을 잡고 기다리는 게 아니라, 충돌이 발생하면 나중 트랜잭션이 실패하고 재시도하는 방식입니다.

쿠폰은 이미 특정 사용자에게 발급된 1회용 도메인이므로, 동시 충돌 확률이 매우 낮습니다. 따라서 비관적 락의 오버헤드를 감수하기보다는, 낙관적 락으로 성능을 확보하고 만에 하나의 충돌은 예외 처리로 대응하는 것이 합리적이라 생각하여 결정 했습니다.

물론 사용자 포인트를 갱신하는 과정에서는 낙관적 락으로는 해결할수 없는 문제들이 발생하였습니다.&#x20;
