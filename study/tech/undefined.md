---
hidden: true
---

# 멱등성에서 시작해 동시성 제어까지 제어해야할 문제를 정의

## 서론: 이 글을 쓰게된 이유

서비스에는 첫 수행 이후 여러 번 반복해도 결과가 변경되지 않는 **멱등함을 보장**해야 하는 상황이 있습니다.

프로젝트 요구사항을 정의하고 시퀀스 다이어그램을 작성하던 중, 멱등성이 필요한 상황을 인지하게 되었습니다. 이를 해결하기 위해 멱등함을 구체화하는 여러 방법을 예제 시나리오로 구현하고 적용해봤습니다.

그 과정에서 멱등함을 공부하다 **동시성 제어**와 유사하다는 느낌을 받게 되었고, 두 개념의 차이를 명확히 하기 위해 이 글을 작성했습니다.

### 글의 구성

이 글에서는

1. **멱등성 기초**: 멱등함이 무엇이고 어떻게 구현하는지
2. **혼동과 발견**: 멱등함과 동시성 제어의 차이점
3. **커머스로 배우기**: 친숙한 예제를 통한 동시성 제어 학습
4. **MES에 적용하기**: 제조업 환경에서의 실제 사례
5. **배운 점 정리**: 도메인별 최적 기법 선택

최소한의 코드 베이스로 두 개념을 명확히 이해하는 과정을 기록하였습니다.



## 1. 멱등함의 기초

멱등성은 수학 개념입니다.

**여러 번 연산을 수행해도 결과가 변경되지 않는 성질**을 의미합니다.

수학적으로 표현하면  `f(f(x)) = f(x)`

#### 예시

**멱등한 동작**:

* **HTTP GET 요청**: 웹 페이지를 조회하기 위해 새로고침을 여러 번 하더라도, 서버는 항상 동일한 데이터를 반환하며 서버의 상태는 변경되지 않음.
* HTTP DELETE 요청: 특정 리소스를 삭제하기 위해 요청을 여러 번 보내면, 첫 번째 요청에서 리소스가 삭제되고, 이후의 요청은 해당 리소스가 이미 없다는 동일한 최종 상태를 반환&#x20;

**비멱등 동작**:

* **HTTP POST 요청**: 일반적으로 새로운 리소스를 생성하는 데 사용됩니다. 동일한 POST 요청을 여러 번 보내면 매번 새로운 리소스가 생성되어 서버의 상태가 요청 횟수만큼 달라짐
* **계좌 이체**: 은행 계좌에서 돈을 인출하거나 입금하는 요청을 반복하면, 매번 잔액이 달라지므로 비멱등 연산입니다.



### 멱등성 해결하기 with java <a href="#id-13" id="id-13"></a>

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

**특징**:

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



#### ReentrantLock - 더 세밀한 제어

**개념 설명**

`ReentrantLock`은 `Lock` 인터페이스를 구현한 클래스입니다.

`synchronized`와 달리 직접 락을 제어할 수 있으며, 다음과 같은 기능을 제공합니다:

* `tryLock(timeout)`: 타임아웃을 설정하고 락 획득 시도
* 공정성 모드: 대기 중인 스레드를 FIFO 순서로 처리
* 조건 변수: 스레드 간 복잡한 조건 처리 가능

**특징**:

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

**개념 설명**

`AtomicInteger`는 **CAS(Compare-And-Swap)** 알고리즘을 사용하여 락 없이 원자적 연산을 보장합니다.

현재 값을 읽고, 예상 값과 비교한 후, 일치하면 새 값으로 교체합니다. 멀티스레드 환경에서도 안전하며 매우 빠릅니다.

(추후 CAS 에 대해서 디테일 한 포스팅을 할 예정입니다.)



**특징**:

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

**개념 설명**

`ConcurrentHashMap`은 내부적으로 버킷별 락을 사용합니다.

일반적으로 많이 사용하는 `HashMap`과 달리 여러 스레드가 서로 다른 버킷에 동시에 접근할 수 있으므로 높은 동시성을 제공합니다. 읽기는 완전히 락 없이 진행되므로 매우 빠릅니다.

**특징**:

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

| 기법                | 구현 복잡도 | 성능    | 사용 시기          |
| ----------------- | ------ | ----- | -------------- |
| synchronized      | 간단     | 낮음    | 단순 케이스         |
| ReentrantLock     | 복잡     | 중간    | 타임아웃, 공정성 필요   |
| AtomicInteger     | 간단     | 매우 높음 | 카운터, 간단한 연산    |
| ConcurrentHashMap | 간단     | 높음    | 캐시, 맵 (가장 실용적) |

이렇게 각 기법을 적용하고 코드를 만들던 도중 의구심이 드는것이 하나 있습니다.

**"결국 멱등성이랑 동시성이랑 비슷한 맥락이 아닌가?"**&#x20;



## 2. 멱등성, 동시성?&#x20;

### 멱등성에서 나온 고민 <a href="#id-21" id="id-21"></a>

앞 문단에서는 멱등성을 구현하기 위해 여러 방법들을 테스트 코드를 통해 적용 및 검증을 해봤습니다.

`synchronized`, `ReentrantLock`, `AtomicInteger`, `ConcurrentHashMap`...

이 모든 기법들을 멱등성을 보장하는 데 사용해봤습니다. 그런데 이 기법들은 반대로 **동시성 제어**를 다루는 곳에서도 사용되는  키워드들입니다.

**"어? 이건 멱등성인가, 동시성인가?"**

멱등성 공부를 할수록 동시성이라는 개념이 계속 등장했습니다. 둘의 관계가 뭘까요?

### 두 개념의 정의 정리 <a href="#id-22" id="id-22"></a>

#### 멱등성 (Idempotency)

**정의**: 같은 요청을 여러 번 수행해도 결과가 같음을 보장하는 성질

**발생 상황**:

* 네트워크 재시도
* 사용자 실수 (버튼 중복 클릭)
* 타임아웃 후 재요청

**예시**:

```
14:25:10 - 요청 1: 결제 버튼 클릭
14:25:12 - 네트워크 타임아웃
14:25:15 - 요청 2: 다시 클릭 (5초 후)
```

#### 동시성 제어 (Concurrency Control)

**정의**: 여러 스레드/프로세스가 동시에 같은 자원에 접근할 때 데이터 정합성을 보장하는 방식

**발생 상황**:

* 멀티스레드 환경
* 병렬 처리
* 여러 CPU 코어 활용

**예시**:

```
14:25:10.000 - 스레드 A: 재고 100 읽음
14:25:10.001 - 스레드 B: 재고 100 읽음 (A 스레드와 거의 동시 시도)
14:25:10.002 - 스레드 A: 재고 99 업데이트
14:25:10.003 - 스레드 B: 재고 99 업데이트 (덮어씀)
```

### 왜 두 개념을 혼동 하였는가? <a href="#id-23" id="id-23"></a>

제가 두 개념을 헷갈렸던 이유는 다음과 같았습니다.

**멱등성 구현 시 동시성 제어가 필요합니다.**

멱등성을 구현할 때 멱등성 키를 저장하는 맵(Map)을 사용합니다.

만약 두 스레드가 동시에 멱등성 키를 확인한다면?

```java
// 문제 상황
if (processedRequests.containsKey(key)) { // 확인
    return processedRequests.get(key);
}

// 다른 스레드가 여기서 끼어들 수 있음!
processedRequests.put(key, result); // 저장

```

두 스레드가 "동시에" 같은 키를 확인하고 둘 다 처리할 수 있습니다.

따라서 멱등성 키 자체는 **`ConcurrentHashMap`, `synchronized`** 등의 동시성 제어 기법을 사용해야 합니다.

두 개념의 명확한 차이점을 정리하면 다음과 같습니다.

<table><thead><tr><th width="149.5">구분</th><th width="207.5">멱등성</th><th width="259">동시성 제어</th></tr></thead><tbody><tr><td>해결하는 문제</td><td>중복 요청 재처리</td><td>동시 접근 데이터 충돌</td></tr><tr><td>발생 타이밍</td><td>시간 간격 있음</td><td>밀리초 단위 동시</td></tr><tr><td>원인</td><td>네트워크, 사용자 실수</td><td>멀티스레드 경쟁</td></tr><tr><td>구현 방식</td><td>멱등성 키, 상태 체크</td><td>Lock, synchronized, CAS</td></tr><tr><td>예방 대상</td><td>반복된 요청</td><td>동시 실행</td></tr></tbody></table>

결국 두개를 나누어져야 하는게 아니라 케이스 별로 구분하여 코드로 구체화를 해야합니다.

####

* 멱등성만 필요한 경우
