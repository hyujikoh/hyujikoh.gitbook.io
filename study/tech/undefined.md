---
hidden: true
---

# DDD, 이제는 쉽게: 책임과 관계로 풀어낸 도메인 설계 원칙

> TL:DR  DDD든 헥사고날이든, 결국 '이 놈은 뭘 해야 하고 누구와 대화해야 하나'만 명확하면 된다.

## 서론. 이 글을 작성하게 된 이유 <a href="#undefined" id="undefined"></a>

**좋은 도메인 설계는 무엇일까? 도메인의 본질은 무엇일까? 엔티티를 나누고 관계를 정의하는 것이 도메인의 근간일까?**

서버 개발을 하면서 신규 개발이든 유지 보수이든 기능을 구현할 때 마다 "이 로직은 어디에 둬야 하지?" 라는 질문 앞에서 머뭇거리곤 했습니다. 서비스 에다 넣을까? , 아니면 도메인 엔티티에 넣을까?&#x20;

이런 고민을 덜어내고자 DDD 관련 자료 (기술 블로그, 책) 를 읽을수록 혼란만 더 늘어났습니다.

애그리거트, 바운디드 컨텍스트, 헥사고날 아키텍처... 각 용어의 정의는 이해했지만, 실제 코드로 옮길 때는 막막했습니&#xB2E4;**.** "이게 맞나?" 싶은 의심이 항상 따라왔습니다.

멘토링 시간에 도메인에 대해서 많은 내용을 배웠지만 가장 기억에 남는 말은 다음과 같습니다.&#x20;

**"수학 잘하는 사람한테 영어 시키지 마세요. 그게 설계의 전부예요."**

이 이야기를 도메인에 대한 복잡한 용어들 뒤에 숨어있던 건 결국 단순한 두 가지 질문입니&#xB2E4;**.**

**이 도메인은 어떤걸 해야하고, 누구와 상호작용을 하는것인가?**

이 글에서는 도메인 주도 개발을 시도하고 이를 위한 아키텍처를, 실제 코드로 작성하며 제가 고민했던 것들을 공유하고자 합니다.

* 좋아요 기능을 어디에 둘 것인가
* 주문과 주문항목은 어떻게 연결해야 하는가
* 도메인 서비스와 애플리케이션 서비스는 어떻게 나눌 것인가
* 애그리거트가 너무 크면 어떻게 할 것인가

정답을 제시하기보다는, 어떤 기준으로 판단했는지 그 **고민의 과정**을 담으려 했습니다..

결국 좋은 설계는 복잡한 용어를 많이 아는 게 아니라, **책임과 관계를 명확히 정의하는 것**이라는 걸 경험으로 배웠기 때문입니다.

## 1. 우리가 선택한 아키텍처: 레이어드 + DIP <a href="#id-1------dip" id="id-1------dip"></a>

### 아키텍처를 고민하게 된 계기

프로젝트 초기, 코드가 점점 복잡해지면서 문제가 생겼습니다.

* Controller에 비즈니스 로직이 섞임
* Service에서 직접 JPA Repository 구현체를 호출
* 테스트 작성 시 실제 DB가 필요함
* 한 곳을 수정하면 여러 곳이 깨짐
* 여러곳에서 사용하는 도메인 서비스

"구조가 필요하다"는 건 알았지만, 유지보수가 용이한 아키텍처를 구성해야 할지 고민이었습니다.

헥사고날, 클린 아키텍처, 레이어드 아키텍처... 각 아키텍처마다 장단점이 있었습니다.

이번에 **레이어드 아키텍처에 DIP를 적용**하는 방식을 선택했고 아래와 같이 구성을 했습니다.

```
[ Interfaces Layer ]
→ 사용자 요청/응답 처리 (Controller)
→ package: /interfaces/api/xx

[ Application Layer ]
→ 유스케이스 실행, 흐름 조율 (Facade, Service)
→ package: /application/xx

[ Domain Layer ]
→ 비즈니스 로직의 핵심 (Entity, VO, DomainService)
→ package: /domain/xx

[ Infrastructure Layer ]
→ 외부 기술 의존 (JPA, Redis, Kafka)
→ package: /infrastructure/xx
```

각 계층의 책임을 명확히 분리했고 각각 레이어는 다음을 담당합니다..

**Interfaces 계층**

* Application Layer의 유스케이스 호출만 담당
* 요청 검증, 응답 매핑 수행
* 비즈니스 로직은 포함하지 않음

**Application(응용) 계층**&#x20;

* 비즈니스 기능의 흐름 조율
* 여러 도메인을 조합하여 유스케이스 완성
* 실질적인 비즈니스 로직은 도메인으로 위임

**Domain 계층**

* 비즈니스 로직의 핵심
* 다른 계층에 의존하지 않음
* 모든 의존 방향은 도메인을 향함

**Infrastructure 계층**

* 구체적인 외부 기술에 의존
* Domain이 정의한 인터페이스를 구현

#### DIP 적용: 의존성 방향 뒤집기

일반적인 제가 작성한 의존성 구조는 위에서 아래로 의존합니다.

```
Application → Infrastructure (Repository 구현체)
```

문제는 이렇게 하면 Application이 JPA 같은 구체적인 기술에 의존하게 된다는 문제가 있습니다. \
마치 연극을 할때 배역을 바라보는게 아니라 그 배역을 맡은 배우에게 의존하게 되는 상황입니다. \
배우가 없을경우에 언제든지 배우를 바꿔서 연극이 진행되는데 문제가 없는것처럼 말입니다.

위와 같은 예시를 실제 DIP(Dependency Inversion Principle)를 적용해 의존 방향을 뒤집었습니다.

```
Domain (인터페이스 정의) ← Infrastructure (구현체)
```



**Domain Layer에 인터페이스 정의**

```java
// domain/order/OrderRepository.java
package com.example.domain.order;

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long orderId);
}
```



**Infrastructure Layer에서 구현**

```java
// infrastructure/order/OrderRepositoryImpl.java
package com.example.infrastructure.order;


@Repository
public class OrderRepositoryImpl implements OrderRepository {
    
    private final OrderJpaRepository jpaRepository;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderEntity.from(order);
        OrderEntity saved = jpaRepository.save(entity);
        return saved.toDomain();
    }
    
    @Override
    public Optional<Order> findById(Long orderId) {
        return jpaRepository.findById(orderId)
            .map(OrderEntity::toDomain);
    }
}
```



**Application Layer에서 인터페이스만 의존**

```java
// application/order/OrderApplicationService.jav
package com.example.application.order;

import com.example.domain.order.OrderRepository; // 인터페이스만 의존

@Service
public class OrderService {
    
    private final OrderRepository orderRepository; // 구현체가 아닌 인터페이스
    
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    @Transactional
    public Long createOrder(OrderRequest request) {
        Order order = Order.create(request.getUserId(), request.getItems());
        Order saved = orderRepository.save(order);
        return saved.getId();
    }
}
```

​

#### DIP의 실질적 이점

**1. 테스트 용이성**

```java
// 테스트 시 Fake 구현체 사용 가능
public class FakeOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    private Long sequence = 0L;
    
    @Override
    public Order save(Order order) {
        order.setId(++sequence);
        store.put(order.getId(), order);
        return order;
    }
    
    @Override
    public Optional<Order> findById(Long orderId) {
        return Optional.ofNullable(store.get(orderId));
    }
}

@Test
void 주문_생성_테스트() {
    // Given: DB 없이 테스트 가능
    OrderRepository fakeRepository = new FakeOrderRepository();
    OrderApplicationService service = new OrderApplicationService(fakeRepository);
    
    // When
    Long orderId = service.createOrder(request);
    
    // Then
    assertNotNull(orderId);
}
```

만약 서비스 DB를 정하기 전에 테스트를 진행하는 상황이라고 했을때. Mock을 쓸 수도 있지만, Fake 구현체를 사용해서 영속성만 처리하고 테스트를 진행할 수도 있습니다. 물론 실제 db 의 조건과 완전히 만족할수는 없지만 결국 fack 이기 때문에 바꿔야할 상황이 온다고 하면 언제든지 바꾸는것이 가능합니다.



**2. 구현 교체 용이성**

JPA에서 QueryDSL로, 또는 다른 DB로 변경해도 Application 코드는 변경할 필요가 없습니다. 다시한번 이야기 하지만 우리는 연극을 볼때 배우를 보지 않고 연극의 등장인물을 바라보면 되기 때문입니다.&#x20;

```java
// JPA 구현체
public class OrderJpaRepositoryImpl implements OrderRepository { ... }

// QueryDSL 구현체
public class OrderQueryDslRepositoryImpl implements OrderRepository { ... }

// Redis 구현체
public class OrderRedisRepositoryImpl implements OrderRepository { ... }
```



**3. 도메인 중심 설계**

모든 의존 방향이 Domain을 향하므로, 자연스럽게 도메인 중심으로 설계하게 된다.

```
Application → Domain ← Infrastructure
```

### 레이어드 + DIP 설계 는 정말 사용하기 좋았나?

처음에는 "인터페이스 만드는 게 번거롭지 않을까?"라는 생각도 했습니다.

하지만 실제로 적용하니 테스트 작성 시간이 절반으로 줄었습니다. DB 설정 없이 Fake 구현체로 빠르게 검증할 수 있었습니다. 기능 변경 시 영향 범위도 명확해졌고, 코드 리뷰에서 "domain 패키지가 infrastructure를 import하면 안 된다"는 규칙이 명확해서 계층 위반을 쉽게 발견할 수 있었습니다. \
(이 부분은 checkstyle 을 통해 위배여부를 확실히 체크 할 수 있었습니다.)

헥사고날 과 같은 복잡한 아키텍처 이론을 설명하지 않아도, 이번 구조는 **각 계층의 책임과 의존 방향**만 명확하면 \
유지보수하는데 정말 용이했습니다.

이번에 실무에 도입했을때도 레이어드 아키텍처 + DIP는 이해하기 쉽고, 테스트하기 쉽고, 변경하기 용이했습니다.&#x20;

**이렇게 아키텍처를 구성하고 나서 중요했던건 "각 도메인은 어떻게 상호작용을 해야하는건가?" 였습니다.**&#x20;





## 2. 좋아요 도메인: 관계만 관리하는 엔티티 <a href="#id-2" id="id-2"></a>

### 좋아요는 독립적일 수 없다

좋아요 기능을 구현하면서 고민이 생겼습니다.

좋아요(Like)는 User와 Product가 있어야만 존재할 수 있습니다. 사용자도 없고 상품도 없는데 좋아요만 있을 수는 없습니다. 이런 도메인을 어떻게 설계해야 할까요?

이리저리 돌고 돌아서 내린 결론은 좋아요는 User의 일도, Product의 일도 아니고 오로지. 좋아요만의 책임이 있다. 다만 독립적인 도메인이 아니라 상호작용이 절대적으로 필요한 도메인이라는 결론을 내렸습니다. \
**User와 Product의 관계를 표현하는 것이 좋아요의 본질이라는게 제 결론 이었습니다**

```java
public class LikeEntity extends BaseEntity {

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Long productId;

    public static LikeEntity createEntity(Long userId, Long productId) {
        return new LikeEntity(userId, productId);
    }
}

```

Like는 User와 Product를 객체로 참조하지 않고 ID로만 참조했습니다. 인스턴스 대신  ID로만 참조한 이유는

객체로 직접 참조하면 Like 조회 시 User, Product도 함께 조회되고, User나 Product가 변경되면 Like도 영향받을 수 있습니다. 반면 ID로만 참조하면 Like는 "누가 어떤 상품을 좋아했다"는 관계 정보만 관리하고, User, Product와 독립적으로 동작하도록 하도록 의도하여 설계하였습니다.

#### 도메인 서비스: Like만의 책임

```java
@Component
@RequiredArgsConstructor
public class LikeService {
    private final LikeRepository likeRepository;
    
    @Transactional(readOnly = true)
    public Optional<LikeEntity> findLike(Long userId, Long productId) {
        return likeRepository.findByUserIdAndProductId(userId, productId);
    }
    
    @Transactional
    public LikeEntity upsertLike(Long userId, Long productId) {
        return likeRepository.findByUserIdAndProductId(userId, productId)
                .map(like -> {
                    if (like.getDeletedAt() != null) {
                        like.restore();
                    }
                    return like;
                })
                .orElseGet(() -> likeRepository.save(
                    LikeEntity.createEntity(userId, productId)
                ));
    }
}
```

LikeService는 오직 Like Repository에만 의존합니다. User를 모르고, Product를 모르고, Like의 생성, 조회, 삭제만 담당합니다. 이게 도메인 서비스의 책임입니다.

#### 응용 서비스: 여러 도메인 조율

실제 좋아요 등록은 Like만으로 끝나지 않습니다. 사용자 검증, 상품 검증, 좋아요 등록, 좋아요 수 증가까지 필요합니다. 이 전체 흐름을 응용 서비스(Facade)가 담당합니다.

```java
@Component
@RequiredArgsConstructor
public class LikeFacade {
    private final ProductService productService;
    private final UserService userService;
    private final LikeService likeService;
    @Transactional
    public LikeInfo upsertLike(String username, Long productId) {
        UserEntity user = userService.getUserByUsername(username);
        ProductEntity product = productService.getProductDetail(productId);
        LikeEntity likeEntity = likeService.upsertLike(user.getId(), product.getId());
        productService.increaseLikeCount(product.getId());
        return LikeInfo.of(likeEntity, product, user);
    }
}
```

#### 책임 분리

* **LikeEntity**: 좋아요 관계 데이터 관리
* **LikeService**: Like 도메인 로직, LikeRepository만 의존
* **LikeFacade**: User, Product, Like 세 도메인 조율

결국 파사드에서 도메인을 활용하는 방법은 뷔페 에서 음식을 골라 먹는것과 같았습니다. 뷔페의 각 음식(도메인)은 서로를 모릅니다. 손님(응용 서비스)이 음식을 조합해서 한 끼 식사를 완성하는 과정이었습니다.

#### 좋아요 도메인을 통해 알게된건?

좋아요 도메인을 통해 배운 것들입니다.

관계만 관리하는 엔티티도 독립된 도메인입니다. Like는 매핑 테이블 성격이 강하지만 자신만의 책임이 명확하고, ID 참조로 느슨한 결합을 유지합니다.

도메인 서비스는 한 도메인만 알고, 응용 서비스는 여러 도메인을 조율합니다.

반대로 강하게 결합된 도메인을 어떻게 설계했는지, Order와 OrderItem 예제를 통해 확인해봤습니다.



***



## 3. 주문 도메인: 강한 결합, 하지만 독립적인 테스트 <a href="#id-3" id="id-3"></a>

### Order와 OrderItem의 관계

좋아요 도메인과는 다른 고민이 생겼습니다.

주문(Order)과 주문항목(OrderItem)은 분명 강한 관계입니다. 주문 없이 주문항목은 존재할 수 없고, 주문이 삭제되면 주문항목도 함께 삭제되어야 합니다. 생명주기를 함께하는 관계입니다.

그런데 코드를 작성하다 보니 Like와는 다른 선택을 하게 됐습니다.

#### 왜 ID 참조를 선택했는가



```java
@Entity
@Table(name = "orders")
public class OrderEntity extends BaseEntity {
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;
    
    public static OrderEntity createOrder(OrderDomainCreateRequest request) {
        return new OrderEntity(request);
    }
    
    public void confirmOrder() {
        if (this.status != OrderStatus.PENDING) {
            throw new CoreException(ErrorType.INVALID_ORDER_STATUS);
        }
        this.status = OrderStatus.CONFIRMED;
    }
}


---

// 주문 항목
@Entity
@Table(name = "order_items")
public class OrderItemEntity extends BaseEntity {
    
    @Column(name = "order_id", nullable = false)
    private Long orderId;
    
    @Column(name = "product_id", nullable = false)
    private Long productId;
    
    @Column(name = "quantity", nullable = false)
    private Integer quantity;
    
    @Column(name = "unit_price", nullable = false)
    private BigDecimal unitPrice;
    
    @Column(name = "total_price", nullable = false)
    private BigDecimal totalPrice;
    
    public static OrderItemEntity createOrderItem(OrderItemDomainCreateRequest request) {
        return new OrderItemEntity(request);
    }
    
    public BigDecimal calculateItemTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```



일반적으로 생각 하면 `Order` 에 `orderitem` 을 가져야 한다고 생각할수 있지만,&#x20;

저는 `OrderItem`  에 id 참조 방식을 적용했는데,  이유는 **각자의 책임을 명확히 분리** 하기 위해서였습니다.

**Order와 OrderItem은 강한 관계입니다**

분명히, OrderItem은 Order 없이 생성될 수 없습니다. 생명주기를 함께하는 강한 참조 관계입니다. 주문이 삭제되면 주문항목도 삭제되어야 합니다.

그런데 강한 관계라고 해서 반드시 객체 참조로 연결해야 할까라는 생각이 들었습니다.

**각자의 책임은 다릅니다**

* **Order의 책임**: 주문 상태 관리, 주문 총액 관리, 주문 확정/취소
* **OrderItem의 책임**: 항목별 수량 관리, 항목별 금액 계산, 가격 검증

Order가 OrderItem 컬렉션을 직접 가지면, Order의 테스트에서 OrderItem의 로직까지 함께 검증해야 합니다. Order의 상태 변경 로직을 테스트하는데 왜 항목의 행위에 대해 까지 고민을 해야하는지 고민을 했습니다. 만약 삭제가 필요하다고 하면, 비즈니스 로직에서 충분히 검증 및 테스트를 통해 풀어낼수 있습니다.

**도메인의 독립적인 역할과 각 도메인의 상호작용을 통해서 각각의 격벽을 만들어내는것이 중요하다 생각했습니다.**

각자의 비즈니스 로직을 독립적으로 검증할 수 있습니다. Order의 상태 관리 로직이 변경되어도 OrderItem 테스트는 영향받지 않고, 그 반대도 마찬가지입니다.

**LazyLoading도 가능하지만**

물론 `@OneToMany(fetch = FetchType.LAZY)`로 지연 로딩을 할 수도 있습니다. 하지만 그건 기술적 해결책일 뿐입니다.

제가 ID 참조를 선택한 진짜 이유는 **도메인의 책임을 명확히 분리**하기 위해서였습니다. Order는 주문의 상태를, OrderItem은 항목의 금액을 책임집니다. 이 책임을 코드 레벨에서도, 테스트 레벨에서도 분리하고 싶었습니다.

Order와 OrderItem의 관계는 분명히 있습니다. 하지만 그 관계를 **어느 계층에서 관리**하느냐가 중요했습니다.



```java
// 도메인 계층: 각자의 책임만
OrderEntity order = OrderEntity.createOrder(...);
OrderItemEntity item = OrderItemEntity.createOrderItem(...);

// 응용 계층: 관계를 조율
@Transactional
public void createOrder(Long userId, List<OrderItemRequest> items) {
    OrderEntity order = orderService.create(userId, totalAmount);
    
    items.forEach(itemRequest -> {
        orderItemService.create(order.getId(), itemRequest);
    });
}
```

도메인 계층에서는 각자의 책임에만 집중하고, 응용 계층에서 관계를 조율하는 방식으로 이번 설계를 해봤습니다.&#x20;



***

## 4. 실무에서 마주한 고민들 <a href="#id-4" id="id-4"></a>

### 값 객체는 언제 만들어야 할까

Order와 OrderItem을 개발하면서 BigDecimal 타입의 금액을 다루게 됐습니다.

totalAmount, unitPrice, totalPrice 모두 `BigDecimal`  이었는데, 개발하다 보니 불안한 순간이 있었습니다. \
가령 quantity와 unitPrice 는 개념이 다르지만,  둘 다 숫자 타입이라 구조상으로는 서로 계산이 되지만, 논리적으로는 말이 안 됩니다.

Price라는 값 객체 (VO) 를 만들까 고민했습니다. 하지만 현재 프로젝트에서는 금액 계산 로직이 복잡하지 않았습니다. 단순히 곱하기, 더하기 정도였습니다.

결국 BigDecimal을 그대로 사용하기로 했습니다. 대신 calculateItemTotal 같은 메서드로 계산 로직을 명확히 캡슐화했습니다.&#x20;

```java
public BigDecimal calculateItemTotal() {
    return unitPrice.multiply(BigDecimal.valueOf(quantity));
}
```

만약 나중에 환율 계산이나 복잡한 할인 정책이 추가된다면, 그때 Price VO를 만들어도 늦지 않다고 판단했습니다.

제가 생각했을때 값 객체를 만들어야 하는 기준을 정리하면 이렇습니다.&#x20;

* 비즈니스 규칙이 있을 때
* 타입 안정성이 중요할 때
* 여러 곳에서 같은 검증이 반복될 때.&#x20;

단순히 타입을 감싸는 것만으로는 충분하지 않습니다.

모든 필드에 의미를 부여할 필요는 없고, 의미 있는 대상에게만 특별한 이름을 붙이면 됩니다.

### 코드 중복, 언제 공통화할까

Order와 OrderItem 둘 다 검증 로직이 있습니다. null 체크, 0 이하 체크 같은 것들입니다.

처음엔 ValidationUtils 같은 공통 클래스를 만들고 싶었지만. 현상 유지를 했습니다.

**OrderEntity의 검증**

```java
private OrderEntity(OrderDomainCreateRequest request) {
    Objects.requireNonNull(request, "주문 생성 요청은 필수입니다.");
    Objects.requireNonNull(request.userId(), "사용자 ID는 필수입니다.");
    Objects.requireNonNull(request.totalAmount(), "주문 총액은 필수입니다.");
    f (request.totalAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("주문 총액은 0보다 커야 합니다.");
    }
    this.userId = request.userId();
    this.totalAmount = request.totalAmount();
    this.status = OrderStatus.PENDING;
}

public void confirmOrder() {
    if (this.status != OrderStatus.PENDING || this.getDeletedAt() != null) {
        throw new CoreException(ErrorType.INVALID_ORDER_STATUS,
        "주문 확정은 대기 상태 또는 활성화된 주문만 가능합니다.");
    }
    this.status = OrderStatus.CONFIRMED;
}
```

**OrderItemEntity의 검증**

```java
private void validateRequest(OrderItemDomainCreateRequest request) {
    if (request == null) {
        throw new IllegalArgumentException("주문 항목 생성 요청은 필수입니다.");
    }
    if (request.quantity() == null) {
        throw new IllegalArgumentException("주문 수량은 필수입니다.");
    }
    if (request.quantity() <= 0) {
        throw new IllegalArgumentException("주문 수량은 1 이상이어야 합니다.");
    }
    if (request.unitPrice().compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("단가는 0보다 커야 합니다.");
    }
}

@Override
protected void guard() {
    BigDecimal calculatedTotal = unitPrice.multiply(BigDecimal.valueOf(quantity));
    if (this.totalPrice.compareTo(calculatedTotal) != 0) {
        throw new IllegalStateException("총 가격이 단가 × 수량과 일치하지 않습니다.");
    }
}
```

실제로 Order의 검증과 OrderItem의 검증은 미묘하게 달랐습니다. Order는 상태에 따른 검증이 필요했고, OrderItem은 금액 정합성 검증이 추가로 필요했습니다.

만약 ValidationUtils를 만들었다면 이렇게 됐을 것입니다.

```java
class ValidationUtils {
    static void checkNotNull(Object value, String message) {
    if (value == null) throw new IllegalArgumentException(message);
    }
    
    
    static void checkPositive(BigDecimal value, String message) {
        if (value.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException(message);
        }
    }
    
    // Order만 쓰는 검증
    static void checkOrderStatus(OrderStatus status, LocalDateTime deletedAt) { ... }
    
    // OrderItem만 쓰는 검증
    static void checkPriceConsistency(BigDecimal total, BigDecimal unit, int qty) { ... }
    
}
```

이건 더 이상 공통 유틸이 아니고, 도메인 로직이 유틸리티로 새어나간 것입니다.

성급하게 공통화했다면, 조건문이 가득한 복잡한 Validator가 만들어졌을 테고, 여러 도메인을 적용하려고 이리저리 시도했을 것입니다. 결국 정말 필요하다 싶을 때 구현하겠다는 다짐을 하고 오히려 각자의 검증 로직을 명확하게 했습니다.

코드 중복을 봤을 때 제가 정한 기준은 이렇습니다. 중복이 생기면 우선 그대로 두고 그것이 컴파일이나 IDE warning이 도배가 되었을 때만 공통화를 고려합니다. 그리고 공통화할 때도 "정말 같은 것인가?"를 한 번 더 확인합니다.

중복을 피하기 위해 과도한 추상화는 도메인 사이에 경계가 무너질 거라고 생각했습니다. 약간의 중복은 각 도메인의 독립성을 보장하는 장치가 될 수 있습니다.

**언제 공통화할까**

만약 Price 값 객체를 만든다면 얘기가 달라집니다.

```java
class Price {
    private final BigDecimal value;
    
    
    public Price(BigDecimal value) {
        if (value.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("가격은 0보다 커야 합니다.");
        }
        this.value = value;
    }

}

// Order와 OrderItem에서 사용
private Price totalAmount;
private Price unitPrice;
```

이건 중복 제거가 아니라 개념의 추출로 값 객체를 사용할수 있다 생각합니다. 다만 지금은 BigDecimal을 직접 사용하고 있고, 각 도메인마다 금액의 의미가 명확하다 생각이 들지 않기 때문에, 우선은 각각 BigDecimal 를 사용해 운영하고 있었습니다 .&#x20;

***

## 5. 결국 중요한 것 <a href="#id-5" id="id-5"></a>

### 다시 원점으로 돌아가서..

DDD, 헥사고날 아키텍처, 클린 아키텍처, 애그리거트, 바운디드 컨텍스트...

처음 이런 용어들을 마주했을 때는 각각을 이해하는 데도 시간이 걸렸고, 실제 코드로 어떻게 구현해야 할지 막막했습니다.

하지만 실제로 개발하면서 깨달았습니다. 이 모든 복잡한 개념들이 결국 답하려는 질문은 두 가지였습니다.

**이 도메인은 뭘 해야 하나?, 이 도메인은 누구와 대화해야 하나?**

#### 책임이 명확하면 코드가 보인다

좋아요 기능을 만들 때 고민했습니다. User 클래스? Product 클래스? 둘 다 어색했습니다.

Like라는 독립된 개념으로 분리하니 명확해졌습니다. Like의 책임은 "좋아요 관계 관리"입니다. User나 Product의 상세 정보는 알 필요가 없습니다. ID만 알면 됩니다.

Order와 OrderItem도 마찬가지였습니다. Order는 "주문 상태 관리", OrderItem은 "항목 금액 계산"이라는 각자의 책임이 명확했습니다. 강한 관계라고 해서 반드시 객체로 참조할 필요는 없었습니다. ID 참조로도 충분했습니다.

만약 각자 책임의 경계가 모호해지고 경계를 침범한다고 하면 그것을 이어주는 다른 도메인을 통해 서로와 상호작용을 하여 각각의 격벽을 새롭게 구성하면 되는것이었습니다.

#### 관계는 계층에서 조율한다

Like는 User나 Product를 모릅니다. LikeService는 LikeRepository만 의존합니다. 그럼 누가 셋을 연결할까요? LikeFacade입니다.

응용 계층에서 UserService, ProductService, LikeService를 조합해서 "좋아요 등록"이라는 완전한 기능을 만듭니다. 뷔페의 각 음식은 서로를 모르지만, 손님이 조합해서 한 끼 식사를 완성하는 것처럼요.

Order와 OrderItem도 도메인 계층에서는 각자의 책임만 가지고, 응용 계층에서 관계를 조율했습니다.

도메인은 순수하게, 응용은 조율하게. 이 분리가 명확하니 수정도 쉬웠습니다.

#### 원칙과 실용 사이

값 객체를 만들지 않았습니다. 코드 중복을 허용했습니다. 애그리거트의 독립성을 깨고 직접 접근하기도 했습니다.

원칙을 어긴 걸까요? 아니라고 생각합니다.

Price 값 객체가 없어도 calculateItemTotal로 계산 로직을 명확히 캡슐화했습니다. ValidationUtils를 만들지 않았지만 각 도메인의 검증 로직은 더 명확했습니다. 애그리거트를 직접 접근했지만 응용 계층에서 일관성을 보장했습니다.

중요한 건 "왜 그렇게 했는가"를 설명할 수 있는 것이었습니다. 무조건 원칙을 따르는 것보다, 상황에 맞는 실용적인 선택을 하되 그 이유를 명확히 아는 것이 더 중요했습니다.

### 레이어드 아키텍처로 충분했다

헥사고날이나 클린 아키텍처를 선택하지 않았습니다. 레이어드 아키텍처에 DIP를 적용하는 것으로 충분했습니다.

OrderRepository 인터페이스를 domain에 두고 OrderRepositoryImpl을 infrastructure에 뒀습니다. 테스트할 때는 FakeOrderRepository를 만들어서 DB 없이 검증할 수 있었습니다.

복잡한 아키텍처가 중요한 게 아니라, 각 계층의 책임과 의존 방향이 명확한 게 중요했습니다.

#### 배운 것들

좋아요와 주문 도메인을 개발하면서 배운 것들을 정리하면 이렇습니다.

**각자의 책임을 명확히 하라.** Like는 관계 관리, Order는 상태 관리, OrderItem은 주문 상품의 상세 정보 이 질문에 한 문장으로 답할 수 없다면, 책임이 너무 많거나 애매한 것입니다.

**응용 계층은 지휘자다.** 도메인들은 서로를 모릅니다. 응용 계층이 여러 도메인을 조율해서 완전한 기능을 만듭니다.

**단위와 통합을 분리하라.** 단위 테스트는 각 도메인의 로직만 검증하고, 통합 테스트는 협력을 검증합니다. 이 분리가 테스트의 목적을 명확하게 만듭니다.

**실용적으로 선택하라.** 원칙은 중요하지만 맹목적으로 따를 필요는 없습니다. 상황에 맞게 조정하되, 그 이유를 스스로가 납득할수 있도록 설명할수있어야 합니다.

### 마치며

DDD든 헥사고날이든, 결국 핵심은 "도메인으로 뭘 해야 하고 누구와 대화해야 하나"였습니다.

이 질문에 답할 수 있다면, 애그리거트가 뭔지, 바운디드 컨텍스트가 뭔지 몰라도 괜찮습니다. 용어를 모른다고 해서 좋은 설계를 못 하는 건 아닙니다.

반대로, 모든 용어를 안다고 해서 좋은 설계를 하는 것도 아닙니다. 용어에 매몰되면 오히려 본질을 놓칠 수 있다 생각합니다.

좋은 설계는 복잡한 용어를 많이 아는 게 아니라, 각 객체의 책임과 관계를 명확히 정의하는 것이었습니다. 그게 전부입니다.

오늘도 작성하던 클래스를 열어보고 스스로에게 질문합니다.

"이 이 객체는 뭘 해야 하고, 누구와 대화해야 할까?"

이 질문에 명확히 답할 수 있도록, 리팩토링 하고 또 개발하고 있습니다.





## 참고 문헌

#### DDD 및 도메인 설계

1. **DDD - 도메인 주도 설계의 핵심 개념**
   * [https://eottabom.github.io](https://eottabom.github.io)
2. **도메인 주도 설계의 사실과 오해 (3) 연관 관계와 애그리거트**
   * [https://borntodare.me](https://borntodare.me)
3. **도메인 주도 설계와 서비스 계층의 이해**
   * [https://f-lab.kr](https://f-lab.kr)
4. **카카오페이 여신코어 DDD(Domain Driven Design, 도메인 주도 설계)**
   * [https://tech.kakaopay.com](https://tech.kakaopay.com)
5. **DDD 입문서를 읽고 나서 느낀 DDD란 무엇인가?**
   * [https://strong-park.tistory.com](https://strong-park.tistory.com)

#### 레이어드 아키텍처 및 DIP

6. **Infrastructure Layer 설계하는 방법 (+ DIP 제대로 사용하기)**
   * [https://jaehoney.tistory.com](https://jaehoney.tistory.com)
7. **도메인 주도 설계의 계층화 아키텍처(Layered Architecture)와 DIP**
   * [https://yoonbing9.tistory.com](https://yoonbing9.tistory.com)
8. **\[Spring] Spring 프로젝트에서 레이어드 아키텍처 제대로 이해하기**
   * [https://devhooney.tistory.com](https://devhooney.tistory.com)
9. **객체 지향 개발 방법론과 클린 아키텍처에서 DIP의 중요성**
   * [https://f-lab.kr](https://f-lab.kr)
10. **OCP를 만족하는 Layered Architecture 개발하기**
    * [https://traeper.tistory.com](https://traeper.tistory.com)

#### 책임 주도 설계

11. **\[Object] 책임 주도 설계 내용 정리**
    * [https://microhabitat.tistory.com](https://microhabitat.tistory.com)
12. **자바 객체를 설계하는 요령 - 책임주도설계**
    * [https://velog.io](https://velog.io)

