---
hidden: true
---

# 우리 서비스는 Lock이 정말 필요한가? - 동시성 제어의 선택과 트레이드오프

> TL:DR : Lock은 만능이 아니며, 도메인 특성을 이해하고 최소한의 복잡도로 해결하는 것이 중요하다.

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

정리 하자면 다음과 같습니다.

**핵심 판단 근거:**

* 쿠폰은 이미 특정 사용자에게 발급된 것 (처리 주체가 한 명)
* 같은 사용자의 동시 요청 확률이 극히 낮음 (충돌 빈도 낮음)

## 2장. 사용자 포인트: 데드락과의 싸움 <a href="#id-2" id="id-2"></a>

#### 문제의 발견 <a href="#undefined" id="undefined"></a>

쿠폰에 낙관적 락을 적용하고 만족스럽게 테스트를 마쳤습니다. 다음은 사용자 포인트 차감입니다. 사용자가 주문하면 포인트에서 금액을 차감하는 로직인데, 처음엔 간단해 보였습니다. 포인트도 낙관적 락으로 처리하면 되지 않을까?

그런데 동시성 테스트를 돌리는 순간, 이상한 일이 발생했습니다.

```
Deadlock found when trying to get lock; try restarting transaction
```

로그를 찾아보니 데드락이 발생한것이었습니다.

데드락이 발생한 케이스는 다음과 같습니다.&#x20;

**사용자는 각기 다른 주문을 동시에 주문한다고 했을때 포인트 차감이 제대로 이루어져야한다.**

테스트 코드는 다음과 같습니다.&#x20;

<details>

<summary>포인트 차감 테스트 코드</summary>

```java
@Test
@DisplayName("동일한 유저가 서로 다른 주문을 동시에 수행해도 포인트가 정상적으로 차감된다")
void should_deduct_points_correctly_with_concurrent_orders_by_same_user() throws InterruptedException {
    // Given: 브랜드 생성
    BrandEntity brand = brandService.registerBrand(
            BrandTestFixture.createRequest("테스트브랜드", "브랜드 설명")
    );
    // Given: 사용자 생성 및 충분한 포인트 충전
    UserRegisterCommand userCommand = UserTestFixture.createDefaultUserCommand();
    UserInfo userInfo = userFacade.registerUser(userCommand);
    BigDecimal initialPoints = new BigDecimal("500000.00");
    pointService.charge(userInfo.username(), initialPoints);
    // Given: 여러 상품 생성
    List<ProductEntity> products = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        ProductEntity product = productService.registerProduct(
                ProductTestFixture.createRequest(
                        brand.getId(),
                        "상품" + i,
                        "설명" + i,
                        new BigDecimal("10000"),
                        100
                )
        );
        products.add(product);
    }
    // Given: 동시 주문 설정
    int threadCount = 10;
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
    CountDownLatch latch = new CountDownLatch(threadCount);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failCount = new AtomicInteger(0);
    // When: 동일 사용자가 동시에 10개의 서로 다른 주문 시도
    for (int i = 0; i < threadCount; i++) {
        final int productIndex = i;
        executorService.submit(() -> {
            try {
                OrderCreateCommand orderCommand = OrderCreateCommand.builder()
                        .username(userInfo.username())
                        .orderItems(List.of(
                                OrderItemCommand.builder()
                                        .productId(products.get(productIndex).getId())
                                        .quantity(2)
                                        .build()
                        ))
                        .build();
                orderFacade.createOrder(orderCommand);
                successCount.incrementAndGet();
            } catch (Exception e) {
                failCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });
    }
    // Then: 모든 스레드 완료 대기
    latch.await(30, TimeUnit.SECONDS);
    executorService.shutdown();
    // Then: 포인트가 충분하므로 모든 주문이 성공해야 함
    assertThat(successCount.get()).isEqualTo(threadCount);
    assertThat(failCount.get()).isZero();
    // Then: 최종 포인트가 정확히 차감되었는지 검증
    UserEntity finalUser = userRepository.findByUsername(userInfo.username())
            .orElseThrow();
    BigDecimal expectedFinalPoints = initialPoints.subtract(
            new BigDecimal("10000").multiply(new BigDecimal("2")).multiply(new BigDecimal(threadCount))
    );
    assertThat(finalUser.getPointAmount())
            .as("동시성 제어로 포인트는 정확히 차감되어야 함")
            .isEqualTo(expectedFinalPoints);
    // Then: 포인트가 음수가 되지 않았는지 확인
    assertThat(finalUser.getPointAmount())
            .as("포인트는 절대 음수가 될 수 없음")
            .isGreaterThanOrEqualTo(BigDecimal.ZERO);
}
```

</details>

데드락이 발생한 코드를 다시한번 확인해봤습니다.&#x20;

```java
// OrderFacade.createOrder
public Order createOrder(Long userId, List<OrderItem> items, Long couponId) {
    // 1. Product에 lock 획득
    for (OrderItem item : items) {
        Product product = productRepository.findByIdWithLock(item.getProductId());
        product.decreaseStock(item.getQuantity());
    }
    
    // 2. Coupon 조회
    Coupon coupon = couponRepository.findById(couponId);
    
    // 3. User의 포인트 차감 (UPDATE 발생!)
    User user = userRepository.findById(userId);
    user.usePoint(totalAmount);
    
    // 4. Order 생성
    return orderRepository.save(order);
}
```

문제는 **락 획득 순서**에 있었습니다.

```java
Thread A: Product1 lock 획득 → User UPDATE 대기
Thread B: Product2 lock 획득 → User UPDATE 대기
```

두 스레드 모두 서로 다른 Product를 잠갔지만, **같은 User row를 UPDATE하려는 순간** 충돌이 발생합니다. MySQL InnoDB는 UPDATE 실행 시 해당 row에 X-Lock(배타적 락)을 걸기 때문입니다. 두 스레드가 동시에 같은 User row에 X-Lock을 걸려고 하면서 교착 상태가 발생한 것입니다.

핵심은 이것입니다. **서로 다른 상품이어도, 같은 사용자의 포인트를 차감하므로 User row에서 충돌합니다.**

#### 해결: 락 획득 순서 통일 <a href="#undefined" id="undefined"></a>

데드락을 방지하는 가장 확실한 방법은 **락 획득 순서를 통일**하는 것입니다. 모든 트랜잭션이 같은 순서로 자원을 잠그면, 순환 대기가 발생하지 않습니다.

```java
// UserRepository에 추가
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT u FROM UserEntity u WHERE u.username = :username")
Optional<UserEntity> findByUsernameWithLock(@Param("username") String username);



// OrderFacade.createOrder (수정 후)
public Order createOrder(Long userId, List<OrderItem> items, Long couponId) {
    // 1. User에 먼저 lock 획득!
    User user = userRepository.findByUsernameWithLock(username);
 
    
   // 이하 로직   
}
```

이렇게 변경 함으로서 주문 등록 플로우에서 User - > Product 순서로 락을 획득했습니다. \
Thread A와 Thread B 모두 먼저 User를 잠그려 하고, 먼저 획득한 쪽이 작업을 완료할 때까지 나머지는 대기하는 구조가 되었고. 이로인해 발생되었던 데드락도 해결하였습니다.

#### 왜 비관적 락을 선택했는가 <a href="#undefined" id="undefined"></a>

쿠폰과 달리, 사용자 포인트에는 비관적 락을 선택했습니다. 이유는 다음과 같습니다.

**핵심 판단 근거:**

* 포인트는 금액과 직결되므로 Lost Update가 절대 허용되지 않습니다.
* 모든 주문에서 User에 접근하므로 충돌 빈도가 높음 (낙관적 락 부적합)
* 데이터 무결성이 성능 저하 보다 중요하다 생각했습니다

이렇게 나름 적절하다고 생각한 기법을 적용해보면서 기능 구현을 했지만, \
역시나 마음은 편하지 못했습니다. "정말 lock 을 적용하는것 말고는 방법이 없을까? " 였습니다



## 3장. 좋아요: Lock 없이 해결하기

### 문제 상황 <a href="#undefined" id="undefined"></a>

먼저 엔티티 구조를 설명하겠습니다.

* `LikeEntity` : 사용자 가 어떤 상품을 좋아요를 했는지 판단하는 테이블 매핑 엔티티
* `ProductEntity.likeCount` : 해당 상품의 좋아요 갯수가 몇개인지 판단하는 필드 likeCount

좋아요 기능을 구현하면서 5명의 사용자가 동시에 같은 상품에 좋아요를 누르는 상황을 가정하여 동시성 테스트를 진행하였습니다.

```
@Test
void 복수의_사용자가_동시에_같은_상품에_좋아요하면_각각_독립적으로_등록된다() {
    // 5명이 동시에 좋아요
    // 예상: LikeEntity 5개 생성, Product.likeCount = 5
}
```

테스트 결과는 흥미로웠습니다.

* `LikeEntity`는 5개가 정상적으로 생성됨 ✅
* `ProductEntity.likeCount`는 1만 증가함 ❌

왜 하나는 성공하고, 하나는 실패했을까요?

#### LikeEntity는 왜 정상이었을까? <a href="#likeentity" id="likeentity"></a>

엔티티의 구조는 아래와 같이 작성하였습니다.

```java
@Entity
@Table(name = "likes", uniqueConstraints = {
    @UniqueConstraint(name = "uc_like_user_product", 
                      columnNames = {"userId", "productId"})
})
public class LikeEntity {
    @Column(nullable = false)
    private Long userId;
    
    @Column(nullable = false)
    private Long productId;
}

```

엔티티 테이블을 만들때 사용자와 상품의 id 를 각각 묶어서 유니크 키로 묶었기 때문에 중복을 원자적으로 방지하기 때문입니다. 동시에 5명이 INSERT를 시도해도, 데이터베이스가 각각을 독립적으로 처리하므로 모두 성공합니다.

#### ProductEntity.likeCount는 왜 문제인가? <a href="#productlikecount" id="productlikecount"></a>

문제는 JPA의 변경 감지 메커니즘에서 발생했습니다.

```java
public LikeEntity upsertLike(UserEntity user, ProductEntity product) {
    product.increaseLikeCount();  // count++
    productRepository.save(product);
    return likeRepository.save(LikeEntity.createEntity(...));
}

```

여러개의 트랜잭션이 동일한 데이터를 동시에 수정하려 할 때, 한 트랜잭션의 변경사항이 다른 트랜잭션에 의해 덮어쓰이는  Lost Update 가 발생하여 데이터가 업데이트가 정상적으로 되지 않았습니다.

<table><thead><tr><th width="87.5">시간</th><th>Thread 1 </th><th>Thread 2</th><th>like count</th></tr></thead><tbody><tr><td>T1</td><td>product 조회 (count=0)</td><td></td><td>0</td></tr><tr><td>T2</td><td></td><td>product 조회 (count=0)</td><td>0</td></tr><tr><td>T3</td><td>count++ → 1</td><td></td><td>0</td></tr><tr><td>T4</td><td></td><td>count++ → 1</td><td>0</td></tr><tr><td>T5</td><td>save → DB에 1 저장</td><td></td><td>1</td></tr><tr><td>T6</td><td></td><td>save → DB에 1 저장</td><td>1 (덮어씀)</td></tr></tbody></table>

두 스레드가 모두 `count=0`을 읽고, 각각 1로 증가시켜 저장하면서 한 번의 증가가 사라졌습니다.

#### 비관적 락을 쓸까? <a href="#undefined" id="undefined"></a>

처음 떠오른 해결책은 비관적 락이었습니다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<ProductEntity> findByIdWithLock(Long id);
```

동작은 하겠지만, 이건 지금 상황에서는 너무 과하다는게 확신이 들었습니다. \
좋아요는 단순히 숫자를 1 증가시키는 것뿐인데, Product 전체를 잠그고 다른 트랜잭션을 대기시켜야 한다 했을때 \
아니라는게 분명했기 때문이었습니다. 게다가 2장에서 경험한 데드락 위험도 신경 쓰였습니다.\
그래서 이번엔 다른 접근방식을 한번 찾아봤습니다.

#### 최종 선택: 원자적 UPDATE 쿼리 <a href="#update" id="update"></a>

락 없이 해결할 방법을 찾다 선택한건, **원자적 UPDATE 쿼리**입니다.

```java
// Repository에 추가
@Modifying
@Query("UPDATE ProductEntity p SET p.likeCount = p.likeCount + 1 " +
       "WHERE p.id = :productId")
int incrementLikeCount(@Param("productId") Long productId);

```

실행되는 sql 문은 단순 했습니다.

```sql
UPDATE products 
SET like_count = like_count + 1 
WHERE id = ?
```

이 쿼리는 **읽기-수정-쓰기가 하나의 원자적 연산**으로 실행됩니다. 데이터베이스가 내부적으로 X-Lock을 걸고 처리하므로, 동시에 여러 UPDATE가 들어와도 순차적으로 처리됩니다. 애플리케이션 레벨에서 명시적으로 락을 걸 필요가 없습니다.

다만 해당 방식의 단점도 물론 있었습니다.

1. 영속성 컨텍스트 동기화 문제
   1. 가장 큰 단점은 JPA의 1차 캐시와 DB 간 싱크가 맞지 않는다는 점입니다. `@Modifying` 쿼리는 영속성 컨텍스트를 무시하고 바로 DB에 실행되므로, 같은 트랜잭션에서 해당 엔티티를 다시 조회하면 업데이트되기 전의 값을 반환할 수 있습니다. 이를 해결하려면 `@Modifying(clearAutomatically = true)`를 사용해 영속성 컨텍스트를 강제로 비워야 합니다.
2. JPA 변경 감지(Dirty Checking) 포기
   1. JPA의 더티체킹 로직을 사용할수 없습니다. 엔티티를 수정하면 자동으로 UPDATE가 발생하는 편리함을 포기하고, 직접 쿼리를 작성해야 합니다. 객체지향적이지 않고, JPA를 쓰는 이유가 퇴색됩니다. 현재 좋아요 서비스에서 상품 repositoryt 추상 클래스를 호출하는 방식을 적용함으로서 프로젝트의 DDD 구조가 깨지는 현상이 있습니다.
3. 복잡한 비즈니스 로직 구현의 어려움
   1. 지금은 좋아요 수 증가/감소라는 단순한 증감 연산라서 괜찮지만, 복잡한 비즈니스 로직을 SQL로 표현하기 어렵습니다. 만일 쿠폰처럼 상태 변경, 금액 계산, 검증 로직이 복잡한 요구사항이 있었다면 절대 선택하지 않았을겁니다.
4. 엔티티 레벨 검증 불가
   1. 엔티티의 `@NotNull`, `@Min`, `@Max` 같은 유효성 검증이 적용되지 않습니다. 음수가 되면 안 되는 재고를 SQL의 `WHERE stock > 0` 조건으로 막아야 하는데, 이는 도메인 중심의 개발에서 벗어나지는 것입니다.



위에서 설명했듯이 해당 방식은 단점이 명확했습니다. 원자적 UPDATE 쿼리는 JPA의 영속성 컨텍스트를 우회하므로, 같은 트랜잭션에서 Product를 다시 조회하면 업데이트 이전의 값을 받을 수 있습니다. 또한 복잡한 비즈니스 로직을 SQL로 표현하기 어렵고, 엔티티 레벨의 유효성 검증도 적용되지 않습니다.

하지만 현재의 좋아요는 단순히 숫자를 증감시키는 것이므로, 이런 단점들이 크게 부각되지 않았기 때문에 완벽한 객체지향 보다. 실용적인 해결을 위해 선택한 방법이었습니다.&#x20;

정리를 하자면 다음과 같습니다.

**핵심 판단 근거:**

* 좋아요는 단순 카운트 증감 연산 (복잡한 비즈니스 로직 불필요)
* 원자적 UPDATE로 충분히 해결 가능
* 락 오버헤드 제거



## 4장. 마무리: 우리 서비스에 정말 필요한 기법일까? <a href="#id-4--lock" id="id-4--lock"></a>

### 세 가지 선택, 하나의 원칙 <a href="#undefined" id="undefined"></a>

세 가지 도메인에서 각각 다른 동시성 제어 전략을 선택했습니다. 표로 정리하면 다음과 같습니다.

<table><thead><tr><th width="119.5">도메인</th><th>선택한 전략</th><th>핵심 판단 근거</th><th>트레이드오프</th></tr></thead><tbody><tr><td>쿠폰</td><td>낙관적 락 (@Version)</td><td>- 특정 사용자에게 발급된 1회용<br>- 동시 충돌 확률 극히 낮음<br>- 비관적 락의 오버헤드 불필요</td><td>충돌 시 재시도 필요<br>하지만 실제로 거의 발생 안할것이다</td></tr><tr><td>사용자 포인트</td><td>비관적 락 (PESSIMISTIC_WRITE)</td><td>- 금액 정합성 최우선<br>- 모든 주문에서 접근 (충돌 빈도 높음)<br>- 데드락 방지 위해 락 획득 순서 통일</td><td>락 대기로 성능 저하<br>하지만 무결성이 더 중요</td></tr><tr><td>좋아요</td><td>원자적 UPDATE 쿼리</td><td>- 단순 카운트 증감<br>- 복잡한 비즈니스 로직 없음<br>- 락 오버헤드 완전 제거</td><td>JPA 변경 감지 포기<br>DDD 구조 일부 깨짐</td></tr></tbody></table>

각각 다른 전략이지만, 하나의 공통된 원칙이 있었습니다.

**"이 문제를 해결하는데, 정말 이 복잡도가 필요한가?"**

### 배운 것들 <a href="#undefined" id="undefined"></a>

#### 1. Lock은 마지막 수단이다

처음엔 "동시성 = Lock"이라고 생각했습니다. 하지만 실제로는 유니크 제약조건, 원자적 쿼리, 격리 수준 조정 같은 더 단순한 방법부터  Java 에서 활용 가능한 동시성 처리 기법 까지 다양한 방법들까지 고려되어야 했습니다. \
(관련 포스트 : [멱등성에서 시작해 동시성까지 제어해야할 문제를 정의](https://hyujikoh-blog.gitbook.io/blog/~/revisions/FUCwlDmp676KYv0ZGXZQ/study/tech/concurrency_idempotent))

Lock은 강력하지만, 성능 저하, 데드락, 복잡도 증가라는 비싼 비용을 요구합니다.

#### 2. 도메인마다 최적의 전략이 다르다

"무조건 비관적 락" 또는 "무조건 낙관적 락"은 없다는 전제하에 필요한 기법을 선택하였습니다.

* **충돌 빈도가 낮은가?** → 낙관적 락
* **데이터 무결성이 최우선인가?** → 비관적 락
* **단순한 증감 연산인가?** → 원자적 쿼리 (리팩토링 필요)
* **복잡한 비즈니스 로직이 있는가?** → 엔티티 레벨 락 필요

#### 3. 트레이드오프를 이해하라

모든 선택에는 대가가 있습니다. 낙관적 락을 선택하면 재시도 로직이 필요 여부를 고려해야하고, 포인트에서 비관적 락을 선택하면 성능이 저하됩니다. 좋아요에서 원자적 쿼리를 선택하면 DDD 구조가 일부 깨집니다. 중요한 건 **어떤 것을 포기하고, 무엇을 얻을 것인지 명확히 아는 것**입니다.

#### 4. 데드락은 언제든지 발생 가능하다.

서로 다른 상품을 주문하는데도 데드락이 발생했던 경험은, 동시성 제어가 단순히 "Lock을 걸면 끝"이 아니라는 걸 깨닫게 해주었습니다. 락을 걸더라도 순서를 통일하지 않으면 교착 상태에 빠질 수 있습니다.

#### 여전히 남은 고민들 <a href="#undefined" id="undefined"></a>

이 글을 쓰면서도 계속 질문이 떠올랐습니다.

* **이 선택이 과연 최선이었을까?** 다른 방법은 없었을까?
* **분산 환경으로 확장한다면?** Redis 분산 락이 필요할까?
* **이벤트 기반 방식으로** 처리가 가능하지 않을까?

완벽한 답은 없고, 상황이 바뀌면 그것에 따른 설계를 해야합니다. 중요한 건 "왜 이 선택을 했는가"를 명확히 설명할 수 있고, 필요하다면 언제든 다시 고민할 수 있다는 것입니다.

### 마지막으로 <a href="#undefined" id="undefined"></a>

"정말 우리 서비스에 동시에 1000명이 같은 쿠폰을 쓸까?"라는 질문부터 시작했던 이 글은, 결국 과한 복잡도를 경계하는 것의 중요성을 다시 확인하는 과정이었습니다. 복잡한 기법보다 문제의 본질을 이해하고 적절한 수준의 해결책을 찾는 것이 더 중요했습니다.
