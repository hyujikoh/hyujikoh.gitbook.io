---
hidden: true
---

# 단위 테스트가 많을수록 좋을까? 🤔 피라미드 vs 트로피, 우리 팀에 맞는 전략 찾기

### **잘 실패하는 테스트는 무엇일까요?**

테스트 코드의 본질은 무엇일까? 단순히 버그를 찾아내거나 코드 커버리지를 높이는 것일까?

테스트의 본질은 이걸 통해 검증하려던 것이 정말 검증이 되는가. 또 이것이 얼마나 민감하게 영향을 받느냐\
중요하다 생각합니다 .

만약 원본 기능이 변했는데 테스트가 실패하지 않는다면, 해당 테스트는 제 역할을 하지 못한것 입니다.\
변경에 민감하게 반응하는 테스트는 저희 서비스의 안정성을 보장하는 안정망이 될것입니다.

그렇다면, '잘 실패하는' 좋은 테스트를 만들려면 단위 테스트와 통합 테스트를 어떻게 배분해야 할까요?

해당 글에서는 좋은 테스트를 작성하기 `단위 테스트`와 `통합 테스트`를 어떻게 전략적으로 분배해야 하는지에 대해 이야기해 보려 합니다.

### **1부: 기초부터 시작하기 - 단위와 통합, 뭐가 다른 거야?** <a href="#undefined" id="undefined"></a>

사전 지식: 두 테스트의 차이점

몇가지 단위 테스트를 작성했고 커버리지도 100% 달성 하였지만, 서로 연관된 도메인 끼리 상호작용을 하는 과정에서 에러가 터지는 경험을 간혹 겪곤 했습니다.\
\
(이미지 넣을까 고민중)

**단위 테스트(Unit Test)**

* 애플리케이션의 가장 작은 부분(함수, 클래스)을 **격리**해서 테스트
* 외부 의존성은 Mock으로 대체
* ⚡ **빠르고** 간단하고 문제 지점을 정확히 찾아냄

**통합 테스트(Integration Test)**

* 여러 컴포넌트가 **함께 동작**하는 것을 테스트
* 실제 DB, API 등과 연동 (필요시 일부만 대체)
* 🎯 **실제 환경**과 유사하여 신뢰도가 높음

***

#### 실제 코드로 이해하기

**시나리오: 주문 시나리오**

```java
// 단위 테스트가 적합한 경우: 순수한 계산 로직
public class PriceCalculator {
    public Money calculateDiscount(Money price, int discountRate) {
        return price.times((100 - discountRate) / 100.0);
    }
}

@Test
@DisplayName("등록된_할인율에_따라_주문가격이_할인된다.")
void 등록된_할인율에_따라_주문가격이_할인된다.() {
    // Arrange
    PriceCalculator calculator = new PriceCalculator();
    Money price = Money.wons(10000);
    
    // Act
    Money result = calculator.calculateDiscount(price, 20);
    
    // Assert
    assertThat(result).isEqualTo(Money.wons(8000));
}

```

외부 의존성이 없고, 순수한 계산 로직을 검증하기 때문에 빠르고 명확하게 테스트가 가능합니다.

```java
// 통합 테스트가 적합한 경우: 여러 도메인의 협력
@SpringBootTest
@Transactional
class OrderIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @MockBean  // 외부 결제 시스템은 Mock으로 대체 💳
    private PaymentGateway paymentGateway;
    
    @MockBean  // 이메일 발송도 Mock으로 대체 📧
    private EmailService emailService;
    
    @Test
    void 주문_생성_시_재고가_차감되고_결제가_처리된다() {
        // Given: 테스트 데이터 준비
        Long userId = 1L;
        Long productId = 100L;
        int orderQuantity = 2;
        
        // 초기 재고 설정
        Inventory inventory = new Inventory(productId, 10);
        inventoryRepository.save(inventory);
        
        // 외부 결제 API는 Stub으로 성공 응답 설정
        when(paymentGateway.processPayment(any()))
            .thenReturn(new PaymentResult(SUCCESS, "txn-12345"));
        
        // When: 주문 생성 (여러 도메인이 협력)
        Order order = orderService.createOrderWithPayment(userId, productId, orderQuantity);
        
        // Then: 주문 도메인 검증
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
        assertThat(order.getQuantity()).isEqualTo(2);
        
        // Then: 재고 도메인이 정상적으로 차감되었는지 검증 (실제 DB)
        Inventory updatedInventory = inventoryRepository.findByProductId(productId).get();
        assertThat(updatedInventory.getQuantity()).isEqualTo(8);  // 10 - 2
        
        // Then: 결제 도메인에 기록이 생성되었는지 검증 (실제 DB)
        Payment payment = paymentRepository.findByOrderId(order.getId()).get();
        assertThat(payment.getTransactionId()).isEqualTo("txn-12345");
        assertThat(payment.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
        
        // Then: 외부 시스템 호출 검증 (Mock)
        verify(paymentGateway).processPayment(any());
        verify(emailService).sendOrderConfirmation(order.getId());
    }
}

```

두번쨰 테스트는 **주문(Order), 재고(Inventory), 결제(Payment)** 세 도메인의 협력을 검증하면서도, **외부 결제 API와 이메일 발송**은 Mock으로 대체합니다. 이유는 다음과 같습니다.

제어 가능한 것은 실제로, 제어가 불가능한 것만은  **Mock**:

* ✅ **실제 사용**: 우리가 제어하는 도메인들 (Order, Inventory, Payment)
* 👻 **Mock 사용**: 외부 시스템 (토스페이 API, 이메일 발송)

```java
// E2E 테스트: 실제 사용자 관점의 전체 흐름
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class OrderE2ETest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void 사용자가_주문_API를_호출하면_전체_프로세스가_완료된다() throws Exception {
        // Given
        String requestBody = """
            {
                "userId": 1,
                "productId": 100,
                "quantity": 2
            }
            """;
        
        // When & Then: HTTP 요청부터 응답까지 전체 스택 테스트
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.orderId").exists())
                .andExpect(jsonPath("$.status").value("COMPLETED"));
        
        // DB 최종 상태까지 검증
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).hasSize(1);
    }
}
```

마지막으로 **HTTP 요청 → Controller → Service → Repository → DB → HTTP 응답**까지 전체 흐름을 검증합니다. 실제 사용자의 관점에서 시작되는 테스트 입니다.

> 물론 `끝(End)` 의 정의가 테스트 대상 시스템의 어느 부분에 속해 있느냐에 따라 완전히 달라지지만 저는 서버 개발자의 입장에서 http 클라이언드와 종착지인 DB 까지의 여정을 각각 End 라 생각하고 작성하였습니다



***

앞서 테스트 에서 `@Mock` , `@MockBean` 이 등장 하였는데,&#x20;

영화 촬영에서 위험한 장면을 대신하는 '스턴트 더블'처럼, 실제 객체를 대신해줄수 있는 객체를 테스트 더블이라 합니다. 아래는 주로 많이 사용하는 테스트 더블의 종류를 소개합니다.

**주요 테스트 더블 종류:**

* **Mock**: 호출 여부와 호출 방식을 검증 (`verify()`)
* **Stub**: 미리 정해진 응답을 반환 (`when().thenReturn()`)
* **Spy**: 실제 객체를 감싸서 일부 동작만 감시
* **Fake**: 단순화된 실제 구현 (예: In-Memory DB)



결국 정리를 하자면 다음과 같습니다.

**단위 테스트**: "이 함수가 올바른 값을 반환하나요?"

* 모든 의존성을 Mock으로 격리
* Repository, 외부 API 모두 Mock

**통합 테스트**: "여러 도메인이 함께 잘 작동하나요?"

* 내부 도메인은 실제로 협력
* 외부 시스템만 Mock으로 대체

**E2E 테스트**: "사용자가 실제로 사용할 수 있나요?"

* HTTP 요청부터 전체 스택 검증
* 외부 시스템은 여전히 Mock



***

### **2부: 전통의 강자, '테스트 피라미드' 🔺**

### 테스트를 몇 개나 써야 할까? 🤔

1부에서 단위 테스트와 통합 테스트의 차이를 배웠고, 이제 고민이 하나 생겼습니다.

> "그래서... 단위 테스트 100개, 통합 테스트 5개 쓰는 게 맞아? 아니면 반대?"

이 질문에 답하기 위해 2001년 Mike Cohn이 제안한 **테스트 피라미드**가 등장합니다.

테스트 피라미드는 간단합니다. 이집트 피라미드처럼 **아래가 넓고 위로 갈수록 좁아지는** 구조입니다.

<div data-with-frame="true"><figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>이미지 출처: <a href="https://martinfowler.com/articles/microservice-testing/#conclusion-test-pyramid">https://martinfowler.com/articles/microservice-testing/#conclusion-test-pyramid</a></p></figcaption></figure></div>

단위, 통합 , E2E 기준으로

* 단위 테스트 : 70% (빠르고 비용이 저렴하지만 단위의 범위는 좁게)
* 통합 테스트 : 20% (모듈 간 연동 확인, 적당한 속도와 신뢰도)
* E2E 테스드 : 10% (핵심 사용자 시나리오를 기준으로 하는것이기 때문에 높은 신뢰도 보장)

#### 왜 피라미드 모양일까? 💰

테스트에는 **트레이드오프(Trade-off)** 가 있기 때문입니다.

| 테스트 유형 | 신뢰도   | 유지보수 비용   | 권장 개수     |
| ------ | ----- | --------- | --------- |
| E2E    | ⭐⭐⭐⭐⭐ | 💰💰💰 높음 | 5\~10개    |
| 통합     | ⭐⭐⭐⭐  | 💰💰 보통   | 20\~50개   |
| 단위     | ⭐⭐⭐   | 💰 낮음     | 100\~500개 |

피라미드는 이 트레이드오프를 최적화하여:

⚡ **빠른 피드백**: 대부분의 버그를 단위 테스트에서 초 단위로 잡음

💰 **비용 효율**: 느린 E2E는 최소화

🎯 **집중**: 각 레이어가 명확한 역할

***

### **2부: 전통의 강자, '테스트 피라미드'**  <a href="#undefined" id="undefined"></a>

### 실전 예시: 복잡한 할인 정책 계산 🏷️

전자상거래 사이트에서 주문을 생성할 때 복잡한 할인 정책이 적용된다고 가정해 보겠습니다.

아래에는 할인에 대한 가상의 요구사항을 정의 해보도록하겠습니다.

* VIP 회원: 20% 할인
* 첫 구매 고객: 5,000원 할인
* 쿠폰 적용
* 중복 할인 제한

이런 복잡한 로직을 테스트 피라미드 기준으로 테스트 비중을 아래와 같이 맞추었습니다.&#x20;

***

### 70% - 단위 테스트: 비즈니스 로직에 집중 ⚡

```java
// 주문 서비스 단위 테스트
class OrderServiceTest {
    
    @Mock  // 👻 실제 ProductRepository 대신 Mock 사용
    private ProductRepository productRepository;
    
    @Mock  // 👻 실제 UserRepository 대신 Mock 사용
    private UserRepository userRepository;
    
    @Mock  // 👻 실제 DiscountPolicy 대신 Mock 사용
    private DiscountPolicy discountPolicy;
    
    @Mock  // 👻 실제 OrderRepository 대신 Mock 사용
    private OrderRepository orderRepository;
    
    @InjectMocks  // Mock들을 주입받은 실제 테스트 대상
    private OrderService orderService;
    
    @Test
    @DisplayName("VIP_회원의_주문에_할인이_적용된다")
    void VIP_회원의_주문에_할인이_적용된다() {
        // Given: 모든 의존성을 Stub으로 설정
        Long userId = 1L;
        Long productId = 100L;
        
        User vipUser = User.builder()
            .id(userId)
            .grade(UserGrade.VIP)
            .build();
        
        Product product = new Product(productId, "노트북", Money.wons(1000000));
        
        // 👻 Stub: UserRepository가 VIP 사용자를 반환하도록 설정
        when(userRepository.findById(userId))
            .thenReturn(Optional.of(vipUser));
        
        // 👻 Stub: ProductRepository가 상품 정보를 반환하도록 설정
        when(productRepository.findById(productId))
            .thenReturn(Optional.of(product));
        
        // 👻 Stub: DiscountPolicy가 20% 할인을 반환하도록 설정
        Money expectedDiscount = Money.wons(200000);
        when(discountPolicy.calculate(any(), any(), any()))
            .thenReturn(expectedDiscount);
        
        // 👻 Stub: OrderRepository가 저장 성공을 시뮬레이션
        when(orderRepository.save(any()))
            .thenAnswer(invocation -> invocation.getArgument(0));
        
        // When: OrderService의 로직 실행
        Order order = orderService.createOrder(userId, productId, 1);
        
        // Then: 결과 검증
        assertThat(order.getOriginalPrice()).isEqualTo(Money.wons(1000000));
        assertThat(order.getDiscountAmount()).isEqualTo(expectedDiscount);
        assertThat(order.getFinalPrice()).isEqualTo(Money.wons(800000));
        
        // 👻 Mock 검증: 각 Repository가 올바르게 호출되었는지 확인
        verify(userRepository).findById(userId);
        verify(productRepository).findById(productId);
        verify(discountPolicy).calculate(any(), eq(vipUser), any());
        verify(orderRepository).save(any());
    }
    
    @Test
    @DisplayName("재고가_부족하면_주문이_실패한다")
    void 재고가_부족하면_주문이_실패한다() {
        // Given
        Long userId = 1L;
        Long productId = 100L;
        
        // 👻 Stub: 재고가 부족한 상황 시뮬레이션
        when(productRepository.findById(productId))
            .thenReturn(Optional.of(new Product(productId, "품절상품", Money.wons(10000))));
        when(inventoryRepository.checkStock(productId, 10))
            .thenReturn(false);
        
        // When & Then: 예외 발생 검증
        assertThatThrownBy(() -> orderService.createOrder(userId, productId, 10))
            .isInstanceOf(OutOfStockException.class)
            .hasMessage("재고가 부족합니다");
        
        // 👻 Mock 검증: 재고 확인까지만 호출되고 저장은 안됨
        verify(inventoryRepository).checkStock(productId, 10);
        verify(orderRepository, never()).save(any());
    }
}
```

**단위 테스트에서 모든 의존성을 Mock으로 격리하는 이유:**

⚡ **빠름**: DB나 외부 API 호출 없이 메모리에서만 실행

🎯 **집중**: OrderService의 비즈니스 로직만 검증

🔧 **제어**: 재고 부족, 결제 실패 등 원하는 시나리오를 쉽게 재현

물론 이 테스트들이 얼마나 빠를지는 프로젝트의 복잡도에 따라 다르지만, 일반적으로 통합 테스트나 E2E보다는 훨씬 빠르게 실행됩니다.

***

### 20% - 통합 테스트: 도메인 간 협력 검증 🔗

```java
// 주문-할인-재고 통합 테스트
@SpringBootTest
@Transactional
class OrderDiscountIntegrationTest {
    
    @Autowired  //  실제 OrderService 사용
    private OrderService orderService;
    
    @Autowired  // 실제 Repository들 사용
    private UserRepository userRepository;
    private ProductRepository productRepository;
    private InventoryRepository inventoryRepository;
    private OrderRepository orderRepository;
    
    @MockBean  // 👻 외부 결제 API만 Mock으로 대체
    private PaymentGateway paymentGateway;
    
    @MockBean  // 👻 이메일 발송도 Mock으로 대체
    private EmailService emailService;
    
    @Test
    @DisplayName("VIP_회원_주문_시_전체_플로우가_정상_작동한다")
    void VIP_회원_주문_시_전체_플로우가_정상_작동한다() {
        // Given: 실제 DB에 테스트 데이터 저장
        User vipUser = userRepository.save(
            User.builder()
                .email("vip@test.com")
                .grade(UserGrade.VIP)
                .build()
        );
        
        Product product = productRepository.save(
            new Product("노트북", Money.wons(1000000))
        );
        
        inventoryRepository.save(
            new Inventory(product.getId(), 10)
        );
        
        // 👻 Stub: 외부 결제 API는 성공 응답하도록 설정
        when(paymentGateway.processPayment(any()))
            .thenReturn(new PaymentResult(SUCCESS, "txn-12345"));
        
        // When: 실제 주문 생성 (Service → Repository → DB 전체 흐름)
        Order order = orderService.createOrderWithPayment(
            vipUser.getId(), 
            product.getId(), 
            2
        );
        
        // Then: 주문 결과 검증
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
        assertThat(order.getDiscountAmount()).isEqualTo(Money.wons(400000));
        
        // Then: 실제 DB에서 재고가 차감되었는지 확인
        Inventory updatedInventory = inventoryRepository.findByProductId(product.getId()).get();
        assertThat(updatedInventory.getQuantity()).isEqualTo(8);
        
        // 👻 Mock 검증: 외부 시스템 호출 확인
        verify(paymentGateway).processPayment(any());
        verify(emailService).sendOrderConfirmation(order.getId());
    }
}
```

**통합 테스트에서 Test Double을 선택적으로 사용하는 이유는 다음과 같습니다.**

| 대상                  | 실제 vs Mock | 이유                 |
| ------------------- | ---------- | ------------------ |
| Service, Repository | ✅ 실제       | 우리가 제어 가능한 내부 시스템  |
| DB 연동               | ✅ 실제       | 실제 데이터 저장/조회 검증 필요 |
| PaymentGateway      | 👻 Mock    | 외부 API, 실제 호출 시 과금 |
| EmailService        | 👻 Mock    | 실제 발송 시 느리고 비용 발생  |

***

### 피라미드가 효과적인 상황 ✨

**✅ 이럴 때 피라미드 전략이 유용합니다**

1. **복잡한 비즈니스 로직이 많을 때**
   * 결제 정산, 할인 계산, 복잡한 권한 체계
2. **외부 의존성이 적을 때**
   * 순수 계산 중심 애플리케이션
   * 라이브러리/프레임워크 개발
3. **빠른 피드백이 중요할 때**
   * TDD 방식 개발
   * CI/CD 파이프라인 자주 실행

***

### 하지만 피라미드는 정말 어떤상황에서도 적용이 가능한 기법일까요? 🤔

요즘 우리가 만드는 애플리케이션을 생각해보면:

마이크로서비스로 쪼개져 있고 🔗, 외부 API를 여러 개 호출하고 ☁️, 데이터베이스와 복잡하게 얽혀 있습니다. \
즉 일일히 다 연결을 하여 테스트 하는것은 현실적으로 매우 어렵습니다.&#x20;

그래서 위와 같은 시나리오 기반 테스트를 한다고 했을때,  Mock 을 통해서 의존성을 최소한으로 하였습니다.

이런 환경에서 단위 테스트로 모든 의존성을 Mock으로 대체하면:

* 실제 환경과 괴리가 생긴다
* Mock 설정 코드가 테스트보다 길어진다
* "테스트는 통과하는데 배포하면 터진다"

이 문제를 해결하기 위해 다른 접근법을 소개하겠습니다.  🏆



