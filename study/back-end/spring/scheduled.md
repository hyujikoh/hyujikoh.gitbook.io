# @Scheduled 스레드 테스트

## 작성하게 된 이유..

나름 운영한지 5년이 넘은 솔루션은 Spring Boot 기반 서버로 만들어졌고, 이 솔루션에는 알림 관리, 보고서 생성, 데이터 업데이트 등 다양한 단위 작업을 위한 스케줄러가 다양한 주기로 운영되고 있었습니다. 평화롭다면 평화롭게? 운영되는 이 솔루션에서 어느순간 스케줄러가 일부 동작하지 않게 되었습니다.&#x20;

#### 문제 현상

1. **전체 스케줄러 중단**: 특정 시점 이후부터 모든 스케줄러가 동작하지 않음
   1. 특정 날짜에서 부터 스케줄링 기반 결과물이 쌓이지 않는것이 확인 되었습니다.
2. **재부팅 후 일회성 실행**: 서버 재시작 후 1회만 실행되고 반복 실행 안됨 (1번 현상 재 반복)
   1. 문제는 재부팅을 했을때 최초 1 회는 되었지만, 동일하게 스케줄링이 안되는 현상이 재 반복 되었습니다.&#x20;

이리저리 분석하여 원인을 찾아본 결과 특정 스케줄러가 blocking 이 되면서 다른 스케줄러가 실행이 안되는 결과에 도달하게 되었습니다. (블로킹이 되었던 솔루션은 외부 솔루션과 연계되어있던 상황이라 무한정 대기가 되어있던 상황이었습니다. 즉각적으로 로직은 수정을 하였습니다.)

#### 원인 분석: @Scheduled의 단일 스레드 특성

**Spring @Scheduled의 기본 동작**이 문제의 원인이었습니다:

* **default 설정으로 scheduler 는  단일 스레드**로 모든 스케줄러를 실행 및 운영한다.
* 하나의 작업이 지연되면 전체 스케줄러 대기

이 문제를 실증적으로 검증하기 위해 테스트 환경을 구축 및 검증하기로 결정했습니다.

### 🛠️ 기술 스택 선택 <a href="#undefined" id="undefined"></a>

#### TinyLog - 경량화된 로깅

기존 Spring Boot의 Logback 대신 **TinyLog**를 선택한 이유:

* **경량화**: 더 적은 메모리 사용량
* **성능**: 빠른 로그 처리 속도
* **단순함**: 간단한 설정으로 원하는 로그 포맷 구성

예전부터 알고있던 오픈소스 기반 로깅 방식이지만, 이번에 테스트 기반으로 빠른 로깅을 하기 위해 한번 사용을 해봤습니다. 사용해본 후기로는 기존에 많이 사용하던 slf4j 보다 빠르다는 체감이 든거 같았습니다..

```
[tinylog.properties]
writer        = console # 로그를 기록 및 노출하는 방식 설정, 원하면 파일로도 설정할 수 있다.
writer.format = {date: HH:mm:ss.SSS} [{thread}] {level}: {message} # 로그 포멧팅의 방식
level         = debug # 로그를 최소 디버스 레벨 이하인 들만 출력함 
```

#### Awaitility - 비동기 테스트

스케줄러와 같은 **시간 기반 비동기 작업** 테스트를 위해 **Awaitility** 도입:

* **조건 기반 대기**: 고정 시간 대신 조건 만족까지 대기
* **테스트 안정성**: 타이밍 문제로 인한 테스트 실패 방지
* **가독성**: DSL 방식으로 직관적인 테스트 코드

이후에 멀티스레드 기반으로 테스트를 할때, 비동기 작업에 대해서 테스트의 정합성을 높이기 위해 이리저리 찾아보다 해당 라이브러리를 확인 및 발견하여 사용하게 되었습니다.

```gradle
testImplementation 'org.awaitility:awaitility:4.2.0'
```

## 📝 테스트 시나리오 설계 <a href="#undefined" id="undefined"></a>

#### 목적

**Spring @Scheduled의 단일 스레드 동작 특성을 실증적으로 확인**

#### 테스트 조건

* **3개의 스케줄러**: 각각 1초, 2초, 3초 작업 시간
* **실행 주기**: 1초, 2초, 3초마다 실행
* **테스트 시간**: 총 10초간 관찰
* **예상 결과**: 단일 스레드로 인한 순차 실행 확인

### 🧪 1단계 : 테스트 환경 구축 및 싱글 스레드 테스트 코드 작성 <a href="#undefined" id="undefined"></a>

#### 프로젝트 설정

```gradle
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.3'
    id 'io.spring.dependency-management' version '1.1.7'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // 경량화된 로깅을 위한 tinylog
    implementation 'org.tinylog:tinylog-api:2.6.2'
    implementation 'org.tinylog:tinylog-impl:2.6.2'
    
    // 비동기 테스트를 위한 Awaitility
    testImplementation 'org.awaitility:awaitility:4.2.0'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

#### 스케줄러 구현

```java
@Component
public class MyScheduler {
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss.SSS");

    @Autowired
    private SchedulerService schedulerService;

    private LocalDateTime startTime;

    @PostConstruct
    public void init() {
        startTime = LocalDateTime.now();
        Logger.info("🚀 [테스트 시작] 시간: {}", startTime.format(formatter));
        Logger.info("📋 [테스트 조건] 단일 스레드, 각 작업 1초/2초/3초, 총 10초간 테스트");

        // 10초 후 요약 출력
        new Thread(() -> {
            try {
                Thread.sleep(10000);
                schedulerService.printSummary();
                Logger.info("🏁 [테스트 종료] 시간: {}", LocalDateTime.now().format(formatter));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }

    // 매 1초마다 실행 (1초 작업)
    @Scheduled(fixedRate = 1000)
    public void scheduler1() {
        if (isTestPeriod()) {
            schedulerService.executeTask1();
        }
    }

    // 매 2초마다 실행 (2초 작업)
    @Scheduled(fixedRate = 2000)
    public void scheduler2() {
        if (isTestPeriod()) {
            schedulerService.executeTask2();
        }
    }

    // 매 3초마다 실행 (3초 작업)
    @Scheduled(fixedRate = 3000)
    public void scheduler3() {
        if (isTestPeriod()) {
            schedulerService.executeTask3();
        }
    }

    private boolean isTestPeriod() {
        return LocalDateTime.now().isBefore(startTime.plusSeconds(10));
    }
}
```

#### 서비스 로직

```java
@Service
public class SchedulerService {
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss.SSS");

    private final AtomicInteger scheduler1Count = new AtomicInteger(0);
    private final AtomicInteger scheduler2Count = new AtomicInteger(0);
    private final AtomicInteger scheduler3Count = new AtomicInteger(0);

    public void executeTask1() {
        int count = scheduler1Count.incrementAndGet();
        String startTime = LocalDateTime.now().format(formatter);
        String threadName = Thread.currentThread().getName();

        Logger.info("🔵 [TASK-1] 시작 - 실행횟수: {}, 시간: {}, 스레드: {}", count, startTime, threadName);

        try {
            Thread.sleep(1000); // 1초 작업 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            Logger.error(e, "Task1 interrupted");
        }

        String endTime = LocalDateTime.now().format(formatter);
        Logger.info("🔵 [TASK-1] 완료 - 실행횟수: {}, 시간: {}, 스레드: {}", count, endTime, threadName);
    }

    public void executeTask2() {
        int count = scheduler2Count.incrementAndGet();
        String startTime = LocalDateTime.now().format(formatter);
        String threadName = Thread.currentThread().getName();

        Logger.info("🟢 [TASK-2] 시작 - 실행횟수: {}, 시간: {}, 스레드: {}", count, startTime, threadName);

        try {
            Thread.sleep(2000); // 2초 작업 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            Logger.error(e, "Task2 interrupted");
        }

        String endTime = LocalDateTime.now().format(formatter);
        Logger.info("🟢 [TASK-2] 완료 - 실행횟수: {}, 시간: {}, 스레드: {}", count, endTime, threadName);
    }

    public void executeTask3() {
        int count = scheduler3Count.incrementAndGet();
        String startTime = LocalDateTime.now().format(formatter);
        String threadName = Thread.currentThread().getName();

        Logger.info("🟡 [TASK-3] 시작 - 실행횟수: {}, 시간: {}, 스레드: {}", count, startTime, threadName);

        try {
            Thread.sleep(3000); // 3초 작업 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            Logger.error(e, "Task3 interrupted");
        }

        String endTime = LocalDateTime.now().format(formatter);
        Logger.info("🟡 [TASK-3] 완료 - 실행횟수: {}, 시간: {}, 스레드: {}", count, endTime, threadName);
    }

    public void printSummary() {
        Logger.info("📊 [요약] Task1: {}회, Task2: {}회, Task3: {}회",
                scheduler1Count.get(), scheduler2Count.get(), scheduler3Count.get());
    }
}
```

#### 🔬 Awaitility를 활용한 테스트 코드 <a href="#awaitility" id="awaitility"></a>

#### 통합 테스트 구성

```java
java@SpringBootTest
@TestPropertySource(properties = {
        "logging.level.com.scheduler.schedulerproject=DEBUG"
})
public class SchedulerTest {

    @Autowired
    private SchedulerService schedulerService;

    @Autowired
    private ApplicationContext applicationContext;

    @Test
    void testScheduledTasks() {
        // 실제 서비스를 스파이로 감싸기
        SchedulerService spyService = Mockito.spy(schedulerService);
        SchedulerService originalService = schedulerService;

        try {
            // 테스트를 위해 스파이 서비스로 교체
            ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                    "schedulerService", spyService);

            // 10초 동안 스케줄러가 실행되는지 검증
            await().atMost(Duration.ofSeconds(11))
                    .untilAsserted(() -> {
                        // 단일 스레드 환경에서 최소 실행 보장 확인
                        verify(spyService, atLeast(1)).executeTask1();
                        verify(spyService, atLeast(1)).executeTask2();
                        verify(spyService, atLeast(1)).executeTask3();
                    });
        } finally {
            // 테스트 후 원래 서비스로 복원
            ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                    "schedulerService", originalService);
        }
    }

    @Test
    void testTaskExecutionDuration() {
        // 각 태스크의 실행 시간을 검증
        SchedulerService realService = new SchedulerService();

        long startTime = System.currentTimeMillis();
        realService.executeTask1();
        long duration = System.currentTimeMillis() - startTime;
        assert duration >= 1000 : "Task1 실행 시간이 1초 미만입니다: " + duration;

        startTime = System.currentTimeMillis();
        realService.executeTask2();
        duration = System.currentTimeMillis() - startTime;
        assert duration >= 2000 : "Task2 실행 시간이 2초 미만입니다: " + duration;

        startTime = System.currentTimeMillis();
        realService.executeTask3();
        duration = System.currentTimeMillis() - startTime;
        assert duration >= 3000 : "Task3 실행 시간이 3초 미만입니다: " + duration;
    }
}
```

#### 테스트 로직 핵심 포인트

#### 1. **Spy 객체 활용**

* **Mock**: 가짜 구현으로 실제 동작 안함
* **Spy**: 실제 메서드 실행하면서 호출 추적 가능
* **선택 이유**: 스케줄러가 실제로 동작하면서 호출 횟수 측정 필요

#### 2. **리플렉션을 통한 필드 교체**

* **목적**: 런타임에 스케줄러의 서비스 필드를 Spy 객체로 교체
* **장점**: 기존 스케줄러 동작 유지하면서 호출 추적만 추가
* **복원**: 테스트 후 원래 상태로 복원하여 테스트 격리성 보장

#### 3. **현실적인 기대값 설정**

```
java// 단일 스레드 환경에서의 계산
// Task1(1초) + Task2(2초) + Task3(3초) = 6초 사이클
// 10초 동안 1.6사이클 정도 실행 가능
verify(spyService, atLeast(1)).executeTask1();  // 최소 1회
verify(spyService, atLeast(1)).executeTask2();  // 최소 1회  
verify(spyService, atLeast(1)).executeTask3();  // 최소 1회
```

#### 🎯 테스트 결과 및 검증 <a href="#undefined" id="undefined"></a>

### 실행 결과 확인

테스트를 통해 **단일 스레드 환경에서 각 스케줄러가 최소 1회씩은 실행됨을 보장**할 수 있었습니다.

### 핵심 발견사항

1. **순차 실행**: 모든 스케줄러가 하나의 스레드에서 순차적으로 실행
2. **블로킹 현상**: 긴 작업이 다른 스케줄러의 실행을 지연시킴
3. **예측 가능한 패턴**: 단일 스레드 특성으로 인한 일정한 실행 패턴

## 🚨 2단계: 블로킹 상황 테스트 - 장시간 실행 스케줄러의 영향 <a href="#id-2" id="id-2"></a>

### 🎯 테스트 목적 <a href="#undefined" id="undefined"></a>

첫 번째 테스트에서 단일 스레드 환경에서의 기본 동작을 확인했다면, 이번에는 **실제 운영 환경에서 발생했던 문제 상황을 재연 하고자 했습니다**.

특정 스케줄러가 **30초 동안 장시간 실행**되어 다른 모든 스케줄러의 실행을 차단하는 기대결과를 얻기 위해 테스트를 작성하였습니다.

테스트를 위한 스케줄러 메소드는 다음과 같이 작성하였습니다.&#x20;

```java
// SchedulerService.java
private final AtomicInteger scheduler4Count = new AtomicInteger(0);

// 30초 동안 스레드를 잡아먹는 작업
public void executeTask4() {
    int count = scheduler4Count.incrementAndGet();
    String startTime = LocalDateTime.now().format(formatter);
    String threadName = Thread.currentThread().getName();

    Logger.info("🟣 [TASK-4] 시작 - 실행횟수: {}, 시간: {}, 스레드: {}", count, startTime, threadName);

    try {
        // 30초 작업 시뮬레이션
        Thread.sleep(30000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        Logger.error(e, "Task4 interrupted");
    }

    String endTime = LocalDateTime.now().format(formatter);
    Logger.info("🟣 [TASK-4] 완료 - 실행횟수: {}, 시간: {}, 스레드: {}", count, endTime, threadName);
}
```

스케줄러 등록

```java
//MyScheduler.java

// 매 0.1초마다 실행 (30초 작업)
@Scheduled(fixedRate = 100)
public void scheduler4() {
    if (isTestPeriod()) {
        schedulerService.executeTask4();
    }
}
```

* **fixedRate = 100ms**: 매우 짧은 간격으로 실행 시도
* **30초 작업**: 한 번 실행되면 30초 동안 스레드 점유
* **결과**: 첫 실행 후 다른 모든 스케줄러가 대기 상태

#### 🧪 블로킹 상황 테스트 코드 <a href="#undefined" id="undefined"></a>

```java
@Test
void testScheduledTasksWith30sTask(){
    // 실제 서비스를 스파이로 감싸기
    SchedulerService spyService = Mockito.spy(schedulerService);

    // 원본 참조 저장
    SchedulerService originalService = schedulerService;

    try {
        // 테스트를 위해 스파이 서비스로 교체
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", spyService);

        // 10초 동안 스케줄러가 실행되는지 검증
        await().atMost(Duration.ofSeconds(11))
                .untilAsserted(() -> {
                    // Task4가 30초 작업으로 스레드를 점유하는 동안
                    // 다른 모든 작업들이 실행되지 않음을 검증
                    verify(spyService, never()).executeTask1();
                    verify(spyService, never()).executeTask2();
                    verify(spyService, never()).executeTask3();
                });
    } finally {
        // 테스트 후 원래 서비스로 복원
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", originalService);
    }
}
```

#### 테스트 중 사용한 주요 방법

1\. **`never()` 검증 사용**

```java
verify(spyService, never()).executeTask1();
verify(spyService, never()).executeTask2();
verify(spyService, never()).executeTask3();
```

* **목적**: 특정 메서드가 **한 번도 호출되지 않았음**을 검증하는 `mockito.never()` 를 이용해  Task4가 스레드를 점유하는 동안 다른 작업들이 실행되지 않음을 보장하는 테스트를 작성하였습니다.&#x20;

2\. **타이밍 설계**

* **테스트 시간**: 10초 (Task4 실행 시간 30초보다 짧음)
* **예상 시나리오**:
  1. Task4가 0.1초 후 시작
  2. Task4가 30초 동안 실행 (테스트 시간 초과)
  3. 다른 작업들은 Task4 완료까지 대기
  4. 10초 테스트 시간 내에는 다른 작업 실행 불가

### 테스트 결과

<figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

**테스트 결과** 다음과 같이 Task4가 스레드를 점유하고 10초 동안 다른 작업들이 한 번도 실행되지 않고,  **단일 스레드 스케줄러의 블로킹 현상을 재연하였습니다.**



## ✅  3단계: 멀티스레드 스케줄러 구현 및 검증 <a href="#id-3" id="id-3"></a>

#### 🔧 멀티스레드 스케줄러 설정 <a href="#undefined" id="undefined"></a>

#### Profile 기반 스케줄러 구성

블로킹 문제를 해결하기 위해 **ThreadPoolTaskScheduler**를 사용한 멀티스레드 스케줄러를 구현했습니다.

```java
@Configuration
@EnableScheduling
@Profile("multi") // 프로파일 지정 
public class SchedulerMultiThreadConfig {
    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(4);  // 4개 스레드 사용
        scheduler.setThreadNamePrefix("scheduler-multi-");
        scheduler.initialize();
        return scheduler;
    }
}
```

#### 설정 특징

#### 1. **Profile 분리**

* `@Profile("multi")`: 멀티스레드 환경을 위한 별도 프로파일
* 단일 스레드와 멀티스레드 환경을 쉽게 전환 가능

#### 2. **스레드 풀 크기**

* `setPoolSize(4)`: 4개의 스레드로 동시 실행 가능
* 기존 스케줄러 개수와 동일한 수준으로 설정

{% hint style="info" %}
사실 적당한 수준에 스레드를 지정하는게 맞습니다. 현재는 테스트 때문에 넉넉하게 스레드로 지정하였지만, 각 프로젝트와 서비스에 맞게 적절한 스레드를 스케줄러 작업에 사용할수 있도록 구성하는게 좋습니다.
{% endhint %}

#### 3. **스레드 이름 지정**

* `setThreadNamePrefix("scheduler-multi-")`: 로그 추적 용이
* 멀티스레드 환경에서 어떤 스레드가 실행 중인지 명확히 구분

### 🧪 멀티스레드 환경 테스트 구성 <a href="#undefined" id="undefined"></a>

#### 테스트 클래스 설정

```java
@SpringBootTest
@TestPropertySource(properties = {
        "logging.level.com.scheduler.schedulerproject=DEBUG"
})
@ActiveProfiles("multi")  // 멀티스레드 프로파일 활성화
class MultiThreadSchedulerTests {
    
    @Autowired
    private SchedulerService schedulerService;

    @Autowired
    private ApplicationContext applicationContext;
    
    // 테스트 메서드들...
}
```

### 핵심 설정 요소

#### 1. **@ActiveProfiles("multi")**

* 멀티스레드 스케줄러 설정 활성화
* SchedulerMultiThreadConfig의 ThreadPoolTaskScheduler 사용

#### 2. **디버그 로깅**

* 스레드별 실행 과정을 상세히 관찰
* 병렬 실행 여부 확인 가능

### 🎯 멀티스레드 검증 테스트 <a href="#undefined" id="undefined"></a>

#### 1. 정상 동작 테스트

```java
@Test
@DisplayName("멀티 스레드 기준 스케줄러가 정상적으로 실행되는지 검증")
void testScheduledTasks() {
    SchedulerService spyService = Mockito.spy(schedulerService);
    SchedulerService originalService = schedulerService;

    try {
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", spyService);

        await().atMost(Duration.ofSeconds(11))
                .untilAsserted(() -> {
                    // 모든 작업이 최소 1회 이상 실행되어야 함
                    verify(spyService, atLeast(1)).executeTask1();
                    verify(spyService, atLeast(1)).executeTask2();
                    verify(spyService, atLeast(1)).executeTask3();
                });
    } finally {
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", originalService);
    }
}
```

#### 2. 블로킹 상황 개선 테스트

```java
@Test
@DisplayName("30초 작업이 10초 동안 실행되지 않는지 검증 또한 이로인해 다른 태스크들이 실행되지 않는지 검증")
void testScheduledTasksWith30sTask(){
    SchedulerService spyService = Mockito.spy(schedulerService);
    SchedulerService originalService = schedulerService;

    try {
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", spyService);

        await().atMost(Duration.ofSeconds(11))
                .untilAsserted(() -> {
                    // Task4는 30초 작업이므로 10초 내에 완료되지 않음
                    verify(spyService, never()).executeTask4();
                    
                    // 하지만 다른 작업들은 병렬로 실행됨 (핵심 개선사항)
                    verify(spyService, atLeast(1)).executeTask1();
                    verify(spyService, atLeast(1)).executeTask2();
                    verify(spyService, atLeast(1)).executeTask3();
                });
    } finally {
        ReflectionTestUtils.setField(applicationContext.getBean(MyScheduler.class),
                "schedulerService", originalService);
    }
}
```

#### 테스트 로직의 핵심 차이점

#### 단일 스레드 환경 (as-is)

```
java// 모든 작업이 실행되지 않음
verify(spyService, never()).executeTask1();
verify(spyService, never()).executeTask2();
verify(spyService, never()).executeTask3();
```

#### 멀티스레드 환경 (to-be)

```java
// Task4는 실행 안되지만, 다른 작업들은 병렬 실행됨
verify(spyService, never()).executeTask4();      // 30초 작업 - 미완료
verify(spyService, atLeast(1)).executeTask1();   // 1초 작업 - 실행됨
verify(spyService, atLeast(1)).executeTask2();   // 2초 작업 - 실행됨
verify(spyService, atLeast(1)).executeTask3();   // 3초 작업 - 실행됨
```



이렇게 작성한 테스트 코드를 명확하게 구분하기 위해 각각 이전 테스트 코드는 다음과 같이 수정하였습니다.&#x20;

```java
// Some code

@Configuration
@EnableScheduling
@Profile("single")
public class SchedulerConfig {
}
```



```java
// Some code
@SpringBootTest
@TestPropertySource(properties = {
        "logging.level.com.scheduler.schedulerproject=DEBUG"
})
@ActiveProfiles("multi")
class MultiThreadSchedulerTests {
}
```



이렇게 하고 각각 통합으로 테스트를 실행했을때 다음과 같은 테스트 결과가 나옵니다.&#x20;

<figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>



## 후기

해당 프로젝트는 다음과 같이 기록 하였다&#x20;

프로젝트 링크 :   [https://github.com/hyujikoh/shcedulder-test-project](https://github.com/hyujikoh/shcedulder-test-project)

어떻게 보면 단순하고 기본적인 내용이지만, 한번 트러블에 대한걸 정리 할겸 작성하고 기록하는 시간을 가져봤다.&#x20;



해당 내용의 3줄 요약은 다음과 같다&#x20;

1\. 스케줄링은 기본적으로 단일 스레드다

2\. 멀티 스레드로 해도 되지만 각 서비스의 스케줄링의 중요도에 따라 조절을 하자.&#x20;

3\. 비동기 작업에 대한 테스트 코드를 통해 정합성을 체크했지만, 비동기 작업의 테스트를 보다 \
면밀하게 할수 있는 기회가 있으면 좋겠다.





## 참고 문헌 및 자료 <a href="#undefined" id="undefined"></a>

* **비동기 코드의 타이밍 테스트 하기 (with Awaitility)** - [https://velog.io/@joosing/test-the-timing-of-asynchronous-code-with-awaitability](https://velog.io/@joosing/test-the-timing-of-asynchronous-code-with-awaitability)
* **Introduction to Awaitility** - [https://www.baeldung.com/awaitility-testing](https://www.baeldung.com/awaitility-testing)
* **awaitility를 사용하여 딜레이 테스트하기** - [https://silvergoni.tistory.com/entry/use-awaitility%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%EB%94%9C%EB%A0%88%EC%9D%B4-%ED%85%8C%EC%8A%A4%ED%8A%B8%ED%95%98%EA%B8%B0](https://silvergoni.tistory.com/entry/use-awaitility%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%EB%94%9C%EB%A0%88%EC%9D%B4-%ED%85%8C%EC%8A%A4%ED%8A%B8%ED%95%98%EA%B8%B0)
* **비지니스 로직에 집중하는, 비동기 테스트 코드 만들기** - [https://velog.io/@joosing/focus-on-business-logic-in-asynchronous-test-code](https://velog.io/@joosing/focus-on-business-logic-in-asynchronous-test-code)
