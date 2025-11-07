---
hidden: true
---

# 멱등성에서 시작해 동시성 제어까지 제어해야할 문제를 정의

## 서론 이 글을 쓰게된 이유

서비스에는 첫 수행 이후 여러 번 반복해도 결과가 변경되지 않는 **멱등함을 보장**해야 하는 상황이 있습니다.

프로젝트 요구사항을 정의하고 시퀀스 다이어그램을 작성하던 중, 멱등성이 필요한 상황을 인지하게 되었습니다. 이를 해결하기 위해 멱등함을 구체화하는 여러 방법을 예제 시나리오로 구현하고 적용해봤습니다.

그 과정에서 멱등함을 공부하다 **동시성 제어**와 유사하다는 느낌을 받게 되었고, \
두 개념의 차이를 공부하는 과정에서 도달한 최종 결론에 대해서 공유 하고자 글을 작성하였습니다.

## 멱등함의 기초

멱등성은 수학 개념입니다.\
**여러 번 연산을 수행해도 결과가 변경되지 않는 성질**을 의미합니다.\
수학적으로 표현하면  `f(f(x)) = f(x)`

#### 예시

**멱등한 동작**

* **HTTP GET 요청**: 웹 페이지를 조회하기 위해 새로고침을 여러 번 하더라도, 서버는 항상 동일한 데이터를 반환하며 서버의 상태는 변경되지 않음.
* **HTTP DELETE 요청**: 특정 리소스를 삭제하기 위해 요청을 여러 번 보내면, 첫 번째 요청에서 리소스가 삭제되고, 이후의 요청은 해당 리소스가 이미 없다는 동일한 최종 상태를 반환&#x20;

**비멱등 동작**:

* **HTTP POST 요청**: 일반적으로 새로운 리소스를 생성하는 데 사용됩니다. 동일한 POST 요청을 여러 번 보내면 매번 새로운 리소스가 생성되어 서버의 상태가 요청 횟수만큼 달라짐
* **계좌 이체**: 은행 계좌에서 돈을 인출하거나 입금하는 요청을 반복하면, 매번 잔액이 달라지므로 비멱등 연산입니다.



### 멱등성 해결하기 with Java <a href="#id-13" id="id-13"></a>

멱등성을 해결하는 가장 기본적인 방법은 멱등성 키(Idempotency Key) 를 도입하는 것입니다.

각 요청에 고유한 멱등성 키를 포함하여 보내면, 서버는 멱등성 키를 저장했다가, 동일한 키로 온 요청을 이미 처리된 것으로 판단하고, 이전 결과를 반환합니다.

```java
@Service
public class PaymentService {
    
    // 처리된 요청 캐시
    private final Map<String, PaymentResponse> processedRequests 
        = new ConcurrentHashMap<>();
    
    private final PaymentRepository paymentRepository;
    
    public PaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }
    
    public PaymentResponse pay(PaymentRequest request, String idempotencyKey) {
        
        // 1. 이미 처리된 요청인지 확인
        if (processedRequests.containsKey(idempotencyKey)) {
            return processedRequests.get(idempotencyKey);
        }
        
        // 2. 신규 요청 처리
        Payment payment = Payment.create(
            request.getAmount(),
            request.getUserId()
        );
        
        paymentRepository.save(payment);
        
        // 3. 응답 생성 및 캐싱
        PaymentResponse response = PaymentResponse.from(payment);
        processedRequests.put(idempotencyKey, response);
        
        return response;
    }
}

```

멱등성 키를 이용한 테스트코드는 다음과 같습니다.

```java
@Test
@DisplayName("사용자가 결제를 2번 요청해도 1번만 처리되어야 함")
void 결제_중복_요청_테스트() {
    // Given
    PaymentRepository repository = new PaymentRepository();
    PaymentService service = new PaymentService(repository);

    PaymentRequest request = new PaymentRequest(10000, "user_123");

    // When: 동일한 요청을 2번 전송
    PaymentResponse response1 = service.pay(request, "req_001");
    PaymentResponse response2 = service.pay(request, "req_001"); // 중복 요청

    // Then: 결제는 1번 처리됨 동일한 멱등키를 가지고 2번 접근
    assertEquals(1, repository.countByUserId("user_123"));
}
```

물론 실제 비즈니스 로직에서 다뤄지는 멱등성 키를 이용한 방식은 더 복잡하기 때문에 지금 이 예시 코드는 단순히 참고 해서 이런식으로 멱등성을 하는구나 흐름만 파악하는 코드로 만들었습니다.

#### Synchronized 를 이용한 멱등성 처리

`synchronized`는 Java에서 제공하는 가장 기본적인 동기화 메커니즘입니다.

메서드나 코드 블록에 `synchronized` 키워드를 붙이면, 한 번에 하나의 스레드만 해당 메서드나 블록에 접근할 수 있습니다. 다른 스레드는 앞의 스레드가 완료될 때까지 대기합니다.&#x20;

**특징**

* 구현이 가장 간단
* 한 번에 하나의 스레드만 실행 (안전함)
* 경합이 높으면 성능 저하
* 타임아웃 설정 불가
* 단일 인스턴스에서만 적용이 가능하고, 다중 인스턴스에서는 사용 불가능

synchronized 를 이용한 테스트 코드는 아래와 작성하였습니다.&#x20;

```java
@Service
public class SynchronizedPaymentService {

    private Map<String, PaymentResponse> processedRequests = new HashMap<>();
    private final PaymentRepository paymentRepository;

    public SynchronizedPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    // synchronized로 메서드 전체 보호
    public synchronized PaymentResponse pay(
            PaymentRequest request,
            String idempotencyKey
    ) {
        if (processedRequests.containsKey(idempotencyKey)) {
            return processedRequests.get(idempotencyKey);
        }

        Payment payment = Payment.create(
                request.getAmount(),
                request.getUserId()
        );

        paymentRepository.save(payment);

        PaymentResponse response = PaymentResponse.from(payment);
        processedRequests.put(idempotencyKey, response);

        return response;
    }
}

// 테스트 코드
@Test
void synchronized_멱등성_동시성_테스트() throws InterruptedException {
    PaymentRepository repository = new PaymentRepository();
    SynchronizedPaymentService service = new SynchronizedPaymentService(repository);

    String idempotencyKey = "pay_001";
    PaymentRequest request = new PaymentRequest(5000, "user_sync");

    // 10개 스레드가 동시에 같은 키로 요청
    AtomicInteger successCount = new AtomicInteger(0);

    ConcurrencyTestHelper.executeConcurrently(10, () -> {
        try {
            service.pay(request, idempotencyKey);
            successCount.incrementAndGet();
        } catch (Exception e) {
            // 예외 처리 (필요시)
        }
    });

    // Then: 결제는 1번만 처리
    assertEquals(1, repository.countByUserId("user_sync"));
    assertEquals(10, successCount.get());
}
```

테스트를 용이하게 진행하기 위한 helper 메소드는 다음과 같이 구성하였습니다.

```java
public class ConcurrencyTestHelper {

    /**
     * 지정된 작업을 여러 스레드에서 동시에 실행합니다.
     *
     * @param threadCount 동시에 실행할 스레드의 수
     * @param task 각 스레드에서 실행할 작업(Runnable)
     * @throws InterruptedException 스레드가 인터럽트될 경우 발생
     */
    public static void executeConcurrently(int threadCount, Runnable task)
            throws InterruptedException {

        // 각각의 CountDownLatch는 스레드 동기화를 위해 사용됩니다
        CountDownLatch readyLatch = new CountDownLatch(threadCount); // 모든 스레드의 준비 상태를 추적
        CountDownLatch startLatch = new CountDownLatch(1);          // 모든 스레드의 동시 시작을 제어
        CountDownLatch doneLatch = new CountDownLatch(threadCount); // 모든 스레드의 작업 완료를 추적

        // 고정 크기의 스레드 풀을 생성하여 작업을 관리
        try (ExecutorService executor = Executors.newFixedThreadPool(threadCount)) {

            // 지정된 수만큼 스레드를 생성하고 작업을 제출
            for (int i = 0; i < threadCount; i++) {
                executor.submit(() -> {
                    try {
                        readyLatch.countDown(); // 이 스레드의 준비 상태를 알림
                        startLatch.await();     // 다른 모든 스레드가 준비될 때까지 대기
                        task.run();             // 모든 스레드가 동시에 작업 실행
                    } catch (InterruptedException e) {
                        // 인터럽트 발생 시 현재 스레드의 인터럽트 상태를 재설정
                        Thread.currentThread().interrupt();
                    } finally {
                        doneLatch.countDown();  // 이 스레드의 작업 완료를 알림
                    }
                });
            }

            // 모든 스레드가 준비될 때까지 메인 스레드가 대기
            readyLatch.await();
            // 모든 스레드에게 동시 시작 신호를 보냄
            startLatch.countDown();
            // 모든 스레드의 작업이 완료될 때까지 메인 스레드가 대기
            doneLatch.await();

            // 스레드 풀을 정상적으로 종료
            executor.shutdown();
        }
    }
}
```

#### ReentrantLock 를 이용한 더 세밀한 제어

`ReentrantLock`은 `Lock` 인터페이스를 구현한 클래스입니다.

`synchronized`와 달리 직접 락을 제어할 수 있으며, 다음과 같은 기능을 제공합니다

* `tryLock(timeout)`: 타임아웃을 설정하고 락 획득 시도
* 공정성 모드: 대기 중인 스레드를 FIFO 순서로 처리
* 조건 변수: 스레드 간 복잡한 조건 처리 가능

**특징**

* 타임아웃 설정 가능 (무한 대기 방지)
* 공정성 모드로 FIFO 보장
* 코드가 복잡함 (try-finally 필수)
* synchronized 대비 세밀한 제어

예시 코드와 테스트 코드는 아래와 같이 작성하였습니다.

```java
// ReentrantLock 서비스 코드
@Service
public class LockPaymentService {
    
    private Map<String, PaymentResponse> processedRequests = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock(true); // 공정성 모드
    private final PaymentRepository paymentRepository;
    
    public LockPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }
    
    public PaymentResponse pay(PaymentRequest request, String idempotencyKey) 
            throws InterruptedException {
        
        // 2초 타임아웃으로 락 획득 시도
        if (!lock.tryLock(2, TimeUnit.SECONDS)) {
            throw new TimeoutException("결제 처리 대기 시간 초과");
        }
        
        try {
            if (processedRequests.containsKey(idempotencyKey)) {
                return processedRequests.get(idempotencyKey);
            }
            
            Payment payment = Payment.create(
                request.getAmount(),
                request.getUserId()
            );
            
            paymentRepository.save(payment);
            
            PaymentResponse response = PaymentResponse.from(payment);
            processedRequests.put(idempotencyKey, response);
            
            return response;
        } finally {
            lock.unlock(); // 반드시 해제
        }
    }
}



// 테스트 코드
@Test
void reentrantLock_멱등성_테스트() throws InterruptedException {
    PaymentRepository repository = new PaymentRepository();
    LockPaymentService service = new LockPaymentService(repository);
    
    String idempotencyKey = "lock_pay_001";
    PaymentRequest request = new PaymentRequest(15000, "user_lock");
    
    ConcurrencyTestHelper.executeConcurrently(10, () -> {
        try {
            service.pay(request, idempotencyKey);
        } catch (Exception e) {
            // 무시
        }
    });
    
    assertEquals(1, repository.countByUserId("user_lock"));
}


```

#### AtomicInteger - 락 없는 연산

`AtomicInteger`는 **CAS(Compare-And-Swap)** 알고리즘을 사용하여 락 없이 원자적 연산을 보장합니다.

현재 값을 읽고, 예상 값과 비교한 후, 일치하면 새 값으로 교체합니다. 멀티스레드 환경에서도 안전하며 매우 빠릅니다.

(추후 CAS 에 대해서 디테일 한 포스팅을 할 예정입니다.)



**특징**

* 매우 빠름 (락 없음)
* CAS 재시도 오버헤드 있음
* 단순 숫자 연산에만 사용 가능
* 데드락 불가능

샘플 코드는 아래와 같이 작성하였습니다.

```java
@Service
public class AtomicPaymentService {

    private final Map<String, PaymentResponse> processedRequests
            = new ConcurrentHashMap<>();

    // 처리된 결제 카운트 (원자적 연산)
    private final AtomicInteger paymentCount = new AtomicInteger(0);
    private final PaymentRepository paymentRepository;

    public AtomicPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    public PaymentResponse pay(PaymentRequest request, String idempotencyKey) {

        PaymentResponse existingResponse = processedRequests.computeIfAbsent(
                idempotencyKey,
                key -> {
                    Payment payment = Payment.create(
                            request.getAmount(),
                            request.getUserId()
                    );
                    paymentRepository.save(payment);

                    PaymentResponse response = PaymentResponse.from(payment);
                    paymentCount.incrementAndGet(); // 원자적 증가

                    return response;
                }
        );


        if (existingResponse != null) {
            return existingResponse;
        }

        return processedRequests.get(idempotencyKey);
    }

    public int getProcessedCount() {
        return paymentCount.get();
    }
}


// 테스트 코드
@Test
void atomic_멱등성_테스트() throws InterruptedException {
    PaymentRepository repository = new PaymentRepository();
    AtomicPaymentService service = new AtomicPaymentService(repository);

    String idempotencyKey = "atomic_pay_001";
    PaymentRequest request = new PaymentRequest(12000, "user_atomic");

    ConcurrencyTestHelper.executeConcurrently(10, () -> {
        service.pay(request, idempotencyKey);
    });

    assertEquals(1, repository.countByUserId("user_atomic"));
    assertEquals(1, service.getProcessedCount());
}
```

#### ConcurrentHashMap - 스레드 안전한 맵

`ConcurrentHashMap`은 내부적으로 버킷별 락을 사용합니다.

일반적으로 많이 사용하는 `HashMap`과 달리 여러 스레드가 서로 다른 버킷에 동시에 접근할 수 있으므로 높은 동시성을 제공합니다. 읽기는 완전히 락 없이 진행되므로 매우 빠릅니다.

**특징**

* 읽기는 완전 락프리
* 쓰기는 버킷별 락 사용 (높은 동시성)
* 약한 일관성 (size() 등은 근사치)
* 가장 실용적

아래는 ConcurrentHashMap 기반 멱등성을 테스트한 코드 입니다.

```java
@Service
public class ConcurrentHashMapPaymentService {

    // 자동으로 스레드 안전함
    private final ConcurrentHashMap<String, PaymentResponse> processedRequests
            = new ConcurrentHashMap<>();

    private final PaymentRepository paymentRepository;

    public ConcurrentHashMapPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    public PaymentResponse pay(PaymentRequest request, String idempotencyKey) {

        return processedRequests.compute(idempotencyKey, (key, existingValue) -> {

            // 이미 처리된 요청
            if (existingValue != null) {
                return existingValue;
            }

            // 신규 요청 처리
            Payment payment = Payment.create(
                    request.getAmount(),
                    request.getUserId()
            );

            paymentRepository.save(payment);
            return PaymentResponse.from(payment);
        });
    }
}

// 테스트 코드
@Test
void concurrentHashMap_멱등성_테스트() throws InterruptedException {
    PaymentRepository repository = new PaymentRepository();
    ConcurrentHashMapPaymentService service =
            new ConcurrentHashMapPaymentService(repository);
    // 여러 키로 동시 요청
    List<String> keys = Arrays.asList("pay_1", "pay_2", "pay_3");
    ConcurrencyTestHelper.executeConcurrently(30, () -> {
        String key = keys.get((int) (Math.random() * keys.size()));
        PaymentRequest request = new PaymentRequest(5000, "user_" + key);
        service.pay(request, key);
    });
    // 각 키당 1번씩만 처리 (총 3건)
    int total = repository.countByUserId("user_pay_1") +
            repository.countByUserId("user_pay_2") +
            repository.countByUserId("user_pay_3");
    assertEquals(3, total);
}
```



### 비교 및 정리

앞서 테스트 코드 기반으로 멱등성을 처리하는 방법에 대해 여러 방법을 적용 및 시도를 해봤습니다.&#x20;

| 기법                | 구현 복잡도 | 성능    | 사용 시기        |
| ----------------- | ------ | ----- | ------------ |
| synchronized      | 간단     | 낮음    | 단순 케이스       |
| ReentrantLock     | 복잡     | 중간    | 타임아웃, 공정성 필요 |
| AtomicInteger     | 간단     | 매우 높음 | 카운터, 간단한 연산  |
| ConcurrentHashMap | 간단     | 높음    | 캐시, 맵        |

이렇게 각 기법을 적용하고 코드를 만들던 도중 의구심이 드는것이 하나 있습니다.

**"결국 멱등성이랑 동시성이랑 비슷한 맥락이 아닌가?"**&#x20;

## 멱등성, 동시성?&#x20;

### 멱등성에서 나온 고민 <a href="#id-21" id="id-21"></a>

앞 문단에서는 멱등성을 구현하기 위해 여러 방법들을 테스트 코드를 통해 적용 및 검증을 해봤습니다.

`synchronized`, `ReentrantLock`, `AtomicInteger`, `ConcurrentHashMap`...

이 모든 기법들을 멱등성을 보장하는 데 사용해봤습니다. 그런데 이 기법들은 반대로 **동시성 제어**를 다루는 곳에서도 사용되는  키워드들입니다.

**"어? 이건 멱등성인가, 동시성인가?"**

멱등성 공부를 할수록 동시성이라는 개념이 계속 등장했습니다. 나중엔 뭐가 어떤건지 혼란스러웠습니다.

### 두 개념의 정의 <a href="#id-22" id="id-22"></a>

#### 멱등성 (Idempotency)

**정의**: 같은 요청을 여러 번 수행해도 결과가 같음을 보장하는 성질

**발생 상황**

* 네트워크 재시도
* 사용자 실수 (버튼 중복 클릭)
* 타임아웃 후 재요청

**예시**

```
14:25:10 - 요청 1: 결제 버튼 클릭
14:25:12 - 네트워크 타임아웃
14:25:15 - 요청 2: 다시 클릭 (5초 후)
```

#### 동시성 제어 (Concurrency Control)

**정의**: 여러 스레드/프로세스가 동시에 같은 자원에 접근할 때 데이터 정합성을 보장하는 방식

**발생 상황**

* 멀티스레드 환경
* 병렬 처리
* 여러 CPU 코어 활용

**예시**

```
14:25:10.000 - 스레드 A: 재고 100 읽음
14:25:10.001 - 스레드 B: 재고 100 읽음 (A 스레드와 거의 동시 시도)
14:25:10.002 - 스레드 A: 재고 99 업데이트
14:25:10.003 - 스레드 B: 재고 99 업데이트 (덮어씀)
```

### 왜 두 개념을 혼동 하였는가? <a href="#id-23" id="id-23"></a>

제가 두 개념을 헷갈렸던 이유는 다음과 같았습니다.

**멱등성 구현 시 동시성 제어가 필요합니다.**

예시 코드 기반 멱등성을 구현할 때 멱등성 키를 저장하는 맵(Map)을 사용을하였습니다.

만약 두 스레드가 동시에 멱등성 키를 확인한다면 어떻게 될까요?

```java
// 문제 상황
if (processedRequests.containsKey(key)) { // 확인
    return processedRequests.get(key);
}

// 다른 스레드가 여기서 끼어들 수 있음!
processedRequests.put(key, result); // 저장

```

두 스레드가 동시에 같은 키를 확인하고 둘 다 처리할 수 있습니다.

따라서 멱등성 키 자체는 **`ConcurrentHashMap`, `synchronized`** 등의 동시성 제어 기법이 사용가능합니다.

또한 두 개념의 명확한 차이점을 정리하면 다음과 같습니다.

<table><thead><tr><th width="149.5">구분</th><th width="207.5">멱등성</th><th width="259">동시성 제어</th></tr></thead><tbody><tr><td>해결하는 문제</td><td>중복 요청 재처리</td><td>동시 접근 데이터 충돌</td></tr><tr><td>발생 타이밍</td><td>시간 간격 있음</td><td>밀리초 단위 동시</td></tr><tr><td>원인</td><td>네트워크, 사용자 실수</td><td>멀티스레드 경쟁</td></tr><tr><td>구현 방식</td><td>멱등성 키, 상태 체크</td><td>Lock, synchronized, CAS</td></tr><tr><td>예방 대상</td><td>반복된 요청</td><td>동시 실행</td></tr></tbody></table>

#### 구체적 예시

* **멱등성만 필요한 경우**

```
// 단일 스레드 환경
public PaymentResponse pay(String idempotencyKey) {
    if (alreadyProcessed(idempotencyKey)) {
        return getCachedResult(idempotencyKey); // 이전 결과 반환
    }
    processPayment();
    cacheResult(idempotencyKey);
}
```



* **동시성 제어만 필요한 경우**

```
// 같은 요청이 안 오지만 여러 스레드가 다른 작업을 동시에 함
public synchronized void decreaseStock(int quantity) {
    if (stock >= quantity) {
        stock -= quantity; // 동시 접근 시 보호 필요
    }
}
```



* **둘 다 필요한 경우**

```
// 멱등성 키를 안전하게 관리하려면 동시성 제어도 필요
public synchronized PaymentResponse pay(String idempotencyKey) {
    if (processedRequests.containsKey(idempotencyKey)) {
        return processedRequests.get(idempotencyKey); // 멱등성
    }
    processPayment();
    processedRequests.put(idempotencyKey, response); // 동시성 안전
}
```

***

정리를 하자면 멱등성 ≠ 동시성, 즉 같은 의미도 아니고 비슷한 맥락도 아니였습니다.\
멱등성 자체는 동시성 제어가 아니고, 동시성 제어가 멱등성을 제공하는 것도 아닙니다.

* **멱등성**: "같은 요청"을 여러 번 받아도 한 번만 처리
* **동시성 제어**: "다른 스레드"가 동시에 접근해도 데이터 정합성 보장

그렇다면 동시성 제어를 쓰는 케이스에 대해서 확인해보겠습니다.

## 커머스로 배우는 동시성 제어 <a href="#part-3" id="part-3"></a>

### 동시성 문제란 <a href="#id-31" id="id-31"></a>

멱등성은 "반복된 요청"을 다루는 개념입니다.

동시성 문제는 다릅니다. **같은 자원에 여러 스레드가 동시에 접근**할 때 발생합니다.

#### Race Condition (경쟁 상태)

가장 흔한 동시성 문제는 **Race Condition**입니다.

두 스레드가 동시에 같은 값을 읽고 각각 수정한 후 저장할 때, 한 스레드의 수정이 다른 스레드의 수정을 덮어쓰는 상황입니다.

#### 동시성 충돌 상황: 상품 재고 차감

온라인 쇼핑몰에서 한정판 상품 100개가 오픈되었습니다.

정확히 12시에 1,000명이 동시에 "구매하기" 버튼을 클릭합니다.

**이상적인 상황**

* 100명은 구매 성공
* 900명은 "재고 없음" 메시지

**Race Condition 발생 시**

* 110명이 구매 성공
* 890명은 구매 성공

***

동시성 제어가 없을 경우에 대해 동시에 100 건의 구매에 대한 재고의 정합성을 테스트를 해봤습니다.

```java
public class Product {
    private Long id;
    private String name;
    private int stock;
    
    public Product(Long id, String name, int initialStock) {
        this.id = id;
        this.name = name;
        this.stock = initialStock;
    }
    
    // 동시성 제어 없음
    public void decreaseStockUnsafe(int quantity) {
        if (stock >= quantity) {
            // 시뮬레이션: 실제 처리 시간
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            stock -= quantity; // Race Condition 발생 지점
        }
    }
    
    public int getStock() {
        return stock;
    }
}


@Test
@DisplayName("동시성 제어 없으면 재고가 초과 판매됨")
void race_condition_재현_테스트() throws InterruptedException {
    // Given: 한정판 상품 100개
    Product product = new Product(1L, "한정판 운동화", 100);
    // When: 100명이 동시에 1개씩 구매
    ConcurrencyTestHelper.executeConcurrently(100, () -> {
        product.decreaseStockUnsafe(1);
    });
    
    System.out.printf("남은 재고: %d%n", product.getStock());


    // 재고는 0 동시성 이슈 떄문에 0 이 안나옴
    assertNotEquals(0, product.getStock());
}


```



결과는 기대했던대로 나왔습니다.

<div align="left"><figure><img src="../../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure></div>

100명이 1개씩 100번 구매했는데 재고가 4개가 남았고. 96**개만 차감되었습니다.**

> 시간 T1: 스레드 A가 재고 100 읽음\
> 시간 T2: 스레드 B가 재고 100 읽음 (동시!)\
> 시간 T3: 스레드 C가 재고 100 읽음 (동시!)\
> ...\
> 시간 T51: 스레드 A가 재고 99로 업데이트\
> 시간 T52: 스레드 B가 재고 99로 업데이트 (덮어씀!)\
> 시간 T53: 스레드 C가 재고 99로 업데이트 (덮어씀!)

이런 식으로 여러 스레드가 원자성 있는 초기화 값을 읽지 않았으므로, \
결과적으로 남는 재고 데이터가 존재하는 상황이 발생합니다.

앞서 멱등성에 적용한 기법을 동일하게 한번 동시성 제어에도 적용하여 테스트 해봤습니다.

***

#### Synchronized <a href="#id-33---synchronized" id="id-33---synchronized"></a>

```java
public class SynchronizedProduct {
    private Long id;
    private String name;
    private int stock;
    
    public SynchronizedProduct(Long id, String name, int initialStock) {
        this.id = id;
        this.name = name;
        this.stock = initialStock;
    }
    
    // synchronized로 보호
    public synchronized boolean decreaseStock(int quantity) {
        if (stock >= quantity) {
            // 시뮬레이션
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            stock -= quantity;
            return true;
        }
        return false; // 재고 부족
    }
    
    public int getStock() {
        return stock;
    }
}

@Test
@DisplayName("synchronized로 재고를 정확히 관리")
void synchronized_재고_관리_테스트() throws InterruptedException {
    // Given: 한정판 상품 100개
    SynchronizedProduct product = new SynchronizedProduct(1L, "한정판 운동화", 100);
    
    // When: 100명이 동시에 1개씩 구매
    AtomicInteger successCount = new AtomicInteger(0);
    
    ConcurrencyTestHelper.executeConcurrently(100, () -> {
        if (product.decreaseStock(1)) {
            successCount.incrementAndGet();
        }
    });
    
    // Then: 100명 모두 성공, 재고 0
    assertEquals(100, successCount.get());
    assertEquals(0, product.getStock());
}
```

***

#### ReentrantLock <a href="#id-34---reentrantlock" id="id-34---reentrantlock"></a>

```java
public class LockProduct {
    private Long id;
    private String name;
    private int stock;
    private final ReentrantLock lock = new ReentrantLock(true); // 공정성 모드
    
    public LockProduct(Long id, String name, int initialStock) {
        this.id = id;
        this.name = name;
        this.stock = initialStock;
    }
    
    public boolean decreaseStock(int quantity) throws InterruptedException {
        // 2초 타임아웃으로 락 획득 시도
        if (!lock.tryLock(2, TimeUnit.SECONDS)) {
            throw new TimeoutException("재고 처리 타임아웃");
        }
        
        try {
            if (stock >= quantity) {
                // 시뮬레이션
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                stock -= quantity;
                return true;
            }
            return false;
        } finally {
            lock.unlock(); // 반드시 해제
        }
    }
    
    public int getStock() {
        return stock;
    }
}

@Test
@DisplayName("ReentrantLock으로 재고를 정확히 관리")
void reentrantLock_재고_관리_테스트() throws InterruptedException {
    // Given: 한정판 상품 100개
    LockProduct product = new LockProduct(1L, "한정판 운동화", 100);
    
    // When: 100명이 동시에 1개씩 구매
    AtomicInteger successCount = new AtomicInteger(0);
    
    ConcurrencyTestHelper.executeConcurrently(100, () -> {
        try {
            if (product.decreaseStock(1)) {
                successCount.incrementAndGet();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
    
    // Then
    assertEquals(100, successCount.get());
    assertEquals(0, product.getStock());
}
```

***

#### AtomicInteger (CAS) <a href="#id-36---atomicinteger-cas" id="id-36---atomicinteger-cas"></a>

```java
public class AtomicProduct {
    private Long id;
    private String name;
    private final AtomicInteger stock;
    
    public AtomicProduct(Long id, String name, int initialStock) {
        this.id = id;
        this.name = name;
        this.stock = new AtomicInteger(initialStock);
    }
    
    public boolean decreaseStock(int quantity) {
        while (true) {
            int currentStock = stock.get();
            
            if (currentStock < quantity) {
                return false; // 재고 부족
            }
            
            // CAS: 현재값이 currentStock이면 (currentStock - quantity)로 업데이트
            if (stock.compareAndSet(currentStock, currentStock - quantity)) {
                return true; // 성공
            }
            // 실패하면 루프 반복 (다시 시도)
        }
    }
    
    public int getStock() {
        return stock.get();
    }
}


@Test
@DisplayName("AtomicInteger로 재고 관리")
void atomic_재고_관리_테스트() throws InterruptedException {
    // Given
    AtomicProduct product = new AtomicProduct(1L, "한정판 운동화", 100);
    
    // When: 100명이 동시에 1개씩 구매
    AtomicInteger successCount = new AtomicInteger(0);
    
    ConcurrencyTestHelper.executeConcurrently(100, () -> {
        if (product.decreaseStock(1)) {
            successCount.incrementAndGet();
        }
    });
    
    // Then
    assertEquals(100, successCount.get());
    assertEquals(0, product.getStock());
}
```

***

#### ConcurrentHashMap <a href="#id-37---concurrenthashmap" id="id-37---concurrenthashmap"></a>

```java
public class Coupon {
    private final String name;
    private final AtomicInteger remainingCount;
    
    public Coupon(String name, int initialCount) {
        this.name = name;
        this.remainingCount = new AtomicInteger(initialCount);
    }
    
    public boolean issue() {
        while (true) {
            int current = remainingCount.get();
            
            if (current <= 0) {
                return false; // 쿠폰 소진
            }
            
            if (remainingCount.compareAndSet(current, current - 1)) {
                return true;
            }
        }
    }
    
    public int getRemainingCount() {
        return remainingCount.get();
    }
}

public class CouponManager {
    private final ConcurrentHashMap<String, Coupon> coupons 
        = new ConcurrentHashMap<>();
    
    public CouponManager() {
        // 여러 쿠폰 등록
        coupons.put("신규가입_10%", new Coupon("신규가입_10%", 100));
        coupons.put("생일_20%", new Coupon("생일_20%", 50));
        coupons.put("추천_5%", new Coupon("추천_5%", 200));
    }
    
    public boolean issueCoupon(String couponName) {
        Coupon coupon = coupons.get(couponName);
        
        if (coupon == null) {
            return false;
        }
        
        return coupon.issue();
    }
    
    public int getRemainingCount(String couponName) {
        Coupon coupon = coupons.get(couponName);
        return coupon != null ? coupon.getRemainingCount() : -1;
    }
}

// 테스트
@Test
@DisplayName("ConcurrentHashMap으로 여러 쿠폰 동시 발급")
void concurrentHashMap_쿠폰_발급_테스트() throws InterruptedException {
    // Given
    CouponManager manager = new CouponManager();
    
    // When: 100명이 동시에 "신규가입_10%" 쿠폰 요청
    AtomicInteger successCount = new AtomicInteger(0);
    
    ConcurrencyTestHelper.executeConcurrently(150, () -> {
        if (manager.issueCoupon("신규가입_10%")) {
            successCount.incrementAndGet();
        }
    });
    
    // Then: 100명만 발급 (초과 요청은 실패)
    assertEquals(100, successCount.get());
    assertEquals(0, manager.getRemainingCount("신규가입_10%"));
}
```

위와 같은 테스트의 결과는 기대한 대로 나왔습니다.&#x20;

<div align="left"><figure><img src="../../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure></div>

***

#### 정리&#x20;

우리가 흔히 접할수 있는 커머스 도메인을 예시로 들어 동시성 제어하는 기법을 적용을 해봤습니다.

물론 지금은 예시 상황이라서 실제 비즈니스 요구사항을 충족을 하는건 절대 아니지만, 최소한의 단위 시나리오를 통해

어떤 방식으로 동시성 제어하는지 이해하고자 작성한 로직였습니다.

그러면 실제 제가 속한 도메인인 제조업에서도 멱등성과 동시성을 제어 해야하는 요구사항을 분석해봤습니다.



## 제조업 에서의 동시성과 멱등성

이전 파트에서 커머스 도메인에서 동시성 제어를 고민해봤습니다.

제가 있는 도메인은 제조업이기 때문에 동시성과 멱등성을 고민해야하는 시점이 조금 다릅니다.&#x20;

<table><thead><tr><th width="155">특성</th><th width="210">커머스</th><th>MES 제조업</th></tr></thead><tbody><tr><td>데이터 발생</td><td>사용자 행동 (불규칙)</td><td>설비 센서 (지속적)</td></tr><tr><td>동시성 수준</td><td>이벤트 시 폭증 (타임딜)</td><td>항상 일정 수준 </td></tr><tr><td>실시간성</td><td>몇 초 지연 허용</td><td>밀리초 단위 중요</td></tr><tr><td>트랜잭션 복잡도</td><td>단순 (주문 1건)</td><td>매우 복잡 (작업지시→자재 할당→설비 배정 )</td></tr><tr><td>롤백 가능성</td><td>높음 (주문 취소)</td><td>낮음 (물리적 작업)</td></tr><tr><td>읽기/쓰기 비율</td><td>70% / 30%</td><td>95% / 5% (모니터링)</td></tr></tbody></table>



### 작업지시 관리 - synchronized 적용 <a href="#id-42-----synchronized" id="id-42-----synchronized"></a>

#### 복합 트랜잭션의 원자성 보장

작업지시 시작 시 **3가지 자원을 동시에 제어**합니다

* 작업지시 상태 변경
* 자재 할당
* 설비 배정

이 중 하나라도 실패하면 전체 트랜잭션이 실패해야 하는데, 여러 스레드가 각각 다른 자원에 접근하면 **원자성이 깨집니다**.

```
스레드 A: 작업지시 시작 OK → 자재 할당 OK → 설비 배정 FAIL
         (이미 상태가 변경되어 롤백 불가능)
```

`synchronized`로 전체 메서드를 보호하면 이 3가지가 모두 완료되거나 모두 실패하도록 보장합니다.

#### 데드락 방지

여러 자원을 제어할 때는 **데드락** 위험이 있습니다.

```
스레드 A: 작업지시 락 → 자재 락 대기 (대기중)
스레드 B: 자재 락 → 작업지시 락 대기 (대기중)
→ 둘 다 영원히 대기 (데드락)
```

`synchronized`를 단일 메서드에 적용하면 **순차적 접근**을 강제하므로 데드락이 발생하지 않습니다.

#### 물리적 작업의 특성

제조업은 커머스와 달리 **물리적 작업**입니다. 한 번 설비를 점유하면 되돌릴 수 없습니다.

```
커머스: 주문 생성 실패 → 다시 주문 가능 (스킵 가능)
MES: 설비 점유 실패 → 물리적으로 이미 설비가 움직임 (롤백 불가)
```

따라서 모든 자원이 확보되거나 아무것도 확보하지 않는 원자성 보장이 필수이며, `synchronized`가 가장 안전합니다.

### 설비 상태 모니터링 - ConcurrentHashMap <a href="#id-44------concurrenthashmap" id="id-44------concurrenthashmap"></a>

#### 읽기/쓰기 비율의 극단적 차이

**실제 MES 대시보드 운영 특성**

```
PLC 업데이트: 초당 50건 (5%)
대시보드 조회: 초당 950건 (95%)
```

​제조업에서 핵심은 현재 우리 공정 프로제스가 어떻게 구성이 되어있고, 모니터링을 할수있는지에 대한

#### synchronized 사용 시 성능 저하

`Hashtable` (synchronized 전체 맵) 사용 시

```
동시 상황:
├─ PLC 업데이트: 락 획득 시도 (1건)
└─ 대시보드 조회: 모두 대기 (950건 대기!)

결과: 950개의 조회 요청이 1개의 업데이트를 기다림
→ 대시보드 응답 지연 (수 초)
```

#### ConcurrentHashMap의 세분화된 보호

ConcurrentHashMap은 **버킷별 독립 락** 사용

```
세그먼트별 락 구조 (16개 세그먼트):
├─ PLC 업데이트 (세그먼트 2): 1개 세그먼트만 보호
├─ 조회 (세그먼트 1): 락 없음, 즉시 반환 ✓
├─ 조회 (세그먼트 3): 락 없음, 즉시 반환 ✓
└─ 조회 (세그먼트 4): 락 없음, 즉시 반환 ✓

95% 조회가 5% 업데이트와 간섭 없음
```

## 정리 및 요약

멱등성 자료를 찾다가 마주친 `synchronized`, `Lock`, `Atomic`  으로 시작해 동시성 까지 고민하면서 쓰게된 이 글은

"나의 요구사항이 충분히 고민하고 검수가 되었는가?" 라는 또 다른 고민을 만들었습니다.&#x20;

결국 앞서 이야기한 기법들은 환경과 조건들로 인해 사용을 못할수도 있고, 인프라 환경에 따라 무용지물이 될수있습니다.&#x20;

기술은 도구이고 더 좋은 도구나 적절한 상황에서 더 유용하게 쓸수있습니다.&#x20;

저희가 할일은 다음과 같다 생각합니다.

* 우리의 서비스를 위해 개발자, 기획, 디자인 등 각 직무의 동료들이 같은 방향을 바라보고 있는지?&#x20;
* 서비스에서 고객의 니즈를 기반으로 한 요구사항이 정의가 되었는지?
* 요구사항을 통해 유의미한 결과를 만들어낼수있는 도구를 우리가 판단할수 있는지?&#x20;

이리저리 돌아서 결국 중요한건 명확한 요구사항 분석 이라고 생각합니다.

요구사항이 명확해야 올바른 기술을 선택할 수 있고, 서비스를 사용하는 사용자에게 가치를 전달할 수 있다 생각합니다.



#### 구현 프로젝트&#x20;

{% embed url="https://github.com/hyujikoh/concurrency_idempotent" %}

## 참고 문서

1. [멱등성이 뭔가요? - 토스페이먼츠 개발자센터](https://docs.tosspayments.com/blog/what-is-idempotency)
2. [멱등성을 이용한 중복 작업 방지 API 구축 - Leapcell](https://leapcell.io/blog/ko/myeokdeungseong-eul-yonghan-jungbok-jag-eob-bangji-API-geuchuk)
3. [API의 멱등성을 고려하여 개발하기 - 티스토리](https://cobinding.tistory.com/279)
4. [Java 동시성 제어 기법 - velog](https://velog.io/@wontaekoh/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%EB%B0%8F-Java%EC%97%90%EC%84%9C%EC%9D%98-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95)
5. [자바에서 동시성 문제를 다루는 n가지 방법들 - 티스토리](https://upcurvewave.tistory.com/649)
6. [Java의 synchronized, Lock Stripping과 Atomic Type - 티스토리](https://320hwany.tistory.com/101)
7. [자바의 동시성 이슈 해결하기 - F-Lab](https://f-lab.kr/insight/solving-concurrency-in-java)
8. [ConcurrentHashMap은 어떻게 동시성을 보장할까? - 티스토리](https://simgee.tistory.com/36)
9. [ConcurrentHashmap 동시성 처리 방법 - 티스토리](https://thewayitwas.tistory.com/608)
10. [ConcurrentHashMap 개념과 동기화 동작 원리 - 티스토리](https://wildeveloperetrain.tistory.com/271)
11. [HashMap의 Thread-safety와 대안들 - 티스토리](https://danuvibe.tistory.com/92)
