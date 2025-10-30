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

**시나리오: 주문 가격 계산하기**

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
    
    @Test
    void 주문_생성_시_재고가_차감되고_결제가_기록된다() {
        // Given: 테스트 데이터 준비
        Long userId = 1L;
        Long productId = 100L;
        int orderQuantity = 2;
        
        // 초기 재고 설정
        Inventory inventory = new Inventory(productId, 10);
        inventoryRepository.save(inventory);
        
        // When: 주문 생성 (여러 도메인이 협력)
        Order order = orderService.createOrder(userId, productId, orderQuantity);
        
        // Then: 주문 도메인 검증
        assertThat(order.getStatus()).isEqualTo(OrderStatus.COMPLETED);
        assertThat(order.getQuantity()).isEqualTo(2);
        
        // Then: 재고 도메인이 정상적으로 차감되었는지 검증
        Inventory updatedInventory = inventoryRepository.findByProductId(productId).get();
        assertThat(updatedInventory.getQuantity()).isEqualTo(8);  // 10 - 2
        
        // Then: 결제 도메인에 기록이 생성되었는지 검증
        Payment payment = paymentRepository.findByOrderId(order.getId()).get();
        assertThat(payment.getAmount()).isEqualTo(order.getTotalPrice());
        assertThat(payment.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
    }
}
```

두번쨰는 주문, 재고, 결제 세개의 도메인의&#x20;



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

