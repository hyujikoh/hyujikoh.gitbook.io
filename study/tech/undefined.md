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
