# JVM Metaspace, 우리 Spring Boot 서비스에서

> TL;DR Metaspace는 클래스 메타데이터 저장소이며, Spring Boot 앱에서는 엔티티 메타모델, CGLIB/JDK 프록시, 생성 코드가 주요 점유 요소. `jstat`은 사용량만, `jcmd`는 내용물까지 확인할 수 있다.

서론 : 이 글을 작성하게 된 이유

운영 중인 멀티테넌트 MES 서비스의 JVM 설정을 점검하던 중, `MaxMetaspaceSize=200m`이라는 숫자가 눈에 들어왔습니다. "200MB면 충분한가?", "거기엔 뭐가 들어있는 거지?"라는 질문에서 출발해 실제 코드베이스를 기반으로 Metaspace 구성을 분석했습니다.

이 글은 다음 순서로 진행됩니다.

* 1장: Metaspace란 무엇이고 뭘 저장하는가
* 2장: 우리 서비스 구조 기반으로 Metaspace 점유 요소 분석
* 3장: Spring 빈이 프록시로 감싸져 Metaspace에 올라가는 과정
* 4장: Metaspace를 모니터링하는 방법 (jstat vs jcmd)
* 5장: 컨테이너 환경에서 Metaspace의 함정 (K8s 1Gi → 2Gi 튜닝 사례)
* 6장: Metaspace는 왜 단방향으로 증가하는가 — 클래스 vs 인스턴스
* 7장: MetaspaceSize vs MaxMetaspaceSize — 시작 시 불필요한 Full GC

**환경 정보**

* Java 17 (JDK 17.0.9), Spring Boot 3.2.5
* Hibernate 6.6.1, QueryDSL 5.1.0, MapStruct 1.6.2
* 컨테이너 기반 배포 (1GB / K8s 동적 할당)

***

### 1장. Metaspace란 무엇인가

#### Java 8 이전: PermGen

Java 7 이하에서는 클래스 메타데이터를 \*\*PermGen(Permanent Generation)\*\*이라는 힙 내부의 고정 크기 영역에 저장했습니다. 크기가 고정이다 보니 클래스를 많이 로드하는 프레임워크(Spring, Hibernate)에서 `java.lang.OutOfMemoryError: PermGen space`가 빈번하게 발생했습니다.

#### Java 8 이후: Metaspace

Java 8부터 PermGen이 제거되고 **Metaspace**로 대체되었습니다. 핵심 차이는 Metaspace가 **힙이 아닌 네이티브 메모리**에 위치한다는 것입니다.

```
JVM 메모리 구조 (Java 17)

┌─────────────────────────────────────────┐
│                JVM Process              │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │           Heap (힙)               │   │
│  │  ┌──────────┐  ┌──────────────┐  │   │
│  │  │  Young   │  │     Old      │  │   │
│  │  │  Gen     │  │     Gen      │  │   │
│  │  └──────────┘  └──────────────┘  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │     Metaspace (네이티브 메모리)     │   │  ← 힙 바깥
│  │  - 클래스 메타데이터               │   │
│  │  - 메서드 바이트코드               │   │
│  │  - 상수 풀 (Constant Pool)        │   │
│  │  - 어노테이션 정보                 │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌───────────────┐ ┌────────────────┐   │
│  │  Code Cache   │ │ Direct Memory  │   │
│  │  (JIT 컴파일)  │ │  (NIO 버퍼)    │   │
│  └───────────────┘ └────────────────┘   │
└─────────────────────────────────────────┘
```

#### Metaspace에 저장되는 것

| 항목                   | 설명                          |
| -------------------- | --------------------------- |
| 클래스 메타데이터            | 클래스 구조, 필드/메서드 시그니처, 접근 제어자 |
| 클래스 로더 정보            | 어떤 클래스 로더가 어떤 클래스를 로드했는지    |
| 메서드 바이트코드            | JIT 컴파일 전 원본 바이트코드          |
| 상수 풀 (Constant Pool) | 클래스 내 리터럴, 심볼릭 참조           |
| 어노테이션 정보             | 런타임 보존 어노테이션 메타데이터          |

중요한 점은 `MaxMetaspaceSize`를 설정하지 않으면 **OS 물리 메모리까지 무한히 증가**할 수 있다는 것입니다. 컨테이너 환경에서는 반드시 상한을 지정해야 합니다.

***

### 2장. 우리 서비스에서 Metaspace에 뭐가 올라가는가

실제 코드베이스를 분석하여 Metaspace 점유 요소를 분류했습니다.

#### 2-1. Hibernate 엔티티 메타모델 (가장 큼)

| 항목                  | 수량   | 설명                               |
| ------------------- | ---- | -------------------------------- |
| `@Entity` 클래스       | 77개  | 15개 도메인에 분포                      |
| `@MappedSuperclass` | 2개   | `BaseEntity`, `BaseCustomColumn` |
| `@Embeddable`       | 13개  | CustomColumn 계열, 검사 상세           |
| LAZY 관계             | 113개 | ByteBuddy 런타임 프록시 생성             |

Hibernate는 `SessionFactory` 초기화 시 엔티티 메타모델, 타입 레지스트리, 필드 매핑 정보를 전부 Metaspace에 올립니다. 도메인별로 보면 information(24개)이 가장 많고, production(9개), inspection(6개) 순입니다.

특히 LAZY 관계 113개에 대해 Hibernate 6.x는 **ByteBuddy**를 사용해 프록시 서브클래스를 동적 생성합니다. 이 프록시들은 앱 시작이 아닌 **처음 접근 시점**에 만들어지므로, 운영 시간이 지날수록 Metaspace 사용량이 점진적으로 증가할 수 있습니다.

#### 2-2. Spring AOP 프록시

| 어노테이션             | 파일 수 | 프록시 방식              |
| ----------------- | ---- | ------------------- |
| `@Service`        | 124  | CGLIB 서브클래스         |
| `@RestController` | 62   | CGLIB 서브클래스         |
| `@Component`      | 53   | CGLIB 서브클래스         |
| `@Repository`     | 41   | JDK 동적 프록시          |
| `@Transactional`  | 119  | AOP 인터셉터 적용         |
| `@PreAuthorize`   | 53   | Security AOP 적용     |
| `@Configuration`  | 18   | CGLIB (Bean 싱글톤 보장) |

280개 Spring 빈 중 AOP가 적용되는 빈은 **원본 클래스 + 프록시 클래스** 2개가 Metaspace에 올라갑니다. 이 과정은 3장에서 자세히 다룹니다.

#### 2-3. 어노테이션 프로세서 생성 코드

| 생성기             | 생성 클래스 수 | 시점     |
| --------------- | -------- | ------ |
| QueryDSL Q-Type | 83개      | 컴파일 타임 |
| MapStruct Impl  | 26개      | 컴파일 타임 |

컴파일 타임에 생성되므로 런타임에 추가 증가는 없지만, 앱 시작 시 109개 클래스가 한꺼번에 로드됩니다. Q-Type은 엔티티 필드 수만큼 `StringPath`, `NumberPath` 등의 정적 필드를 보유하므로 필드가 많은 엔티티일수록 Q-Type의 메타데이터도 큽니다.

#### 2-4. 서드파티 라이브러리

| 라이브러리              | Metaspace 기여 | 이유                       |
| ------------------ | ------------ | ------------------------ |
| Hibernate 6.6.1    | 매우 높음        | 메타모델, ByteBuddy, 타입 시스템  |
| Spring Security    | 높음           | 필터 체인, SpEL 컴파일, AOP     |
| Firebase Admin SDK | 중간\~높음       | gRPC, Netty, Guava 동반 로드 |
| SpringDoc OpenAPI  | 중간           | 리플렉션 기반 API 스펙 생성        |
| Apache POI         | 중간           | 엑셀 처리 내부 클래스 다수          |

#### 2-5. 멀티테넌시 — Metaspace에 영향 없음

`SCHEMA` 전략은 단일 `SessionFactory`를 공유하고 커넥션 수준에서 `setCatalog(tenantId)`로 스키마를 전환합니다. `DATABASE` 전략처럼 테넌트마다 별도 SessionFactory를 만들지 않으므로, **테넌트가 늘어도 Metaspace 사용량은 증가하지 않습니다**.

#### 2-6. 종합 비율 (추정)

```
Metaspace 200MB
├── Hibernate 메타모델 + ByteBuddy 프록시  ██████████░  ~40%
│   (77 Entity + 113 LAZY 프록시 + 타입 레지스트리)
├── Spring CGLIB/JDK 프록시              ████████░░░  ~25%
│   (280 빈 + @Transactional 119 + @PreAuthorize 53)
├── 서드파티 라이브러리                    ██████░░░░░  ~20%
│   (Firebase+gRPC, Security, SpringDoc, POI)
├── 생성 코드 (QueryDSL + MapStruct)      ██░░░░░░░░░  ~8%
│   (83 Q-Type + 26 MapperImpl)
└── 나머지 (Config, 커스텀 코드 등)         █░░░░░░░░░░  ~7%
```

***

### 3장. Spring 빈이 프록시로 Metaspace에 올라가는 과정

Spring에서 AOP가 적용되는 빈은 단순히 클래스를 로드하는 것으로 끝나지 않습니다. 어떤 어노테이션이 붙었느냐에 따라 Metaspace 적재 방식이 달라집니다.

#### 3-1. 순수 @Component (프록시 없음)

AOP 대상이 아닌 빈은 원본 클래스 1개만 Metaspace에 올라갑니다.

```
클래스로더: MyComponent.class 로드
                ↓
Metaspace: [MyComponent]  ← 1개
```

#### 3-2. @Service + @Transactional (CGLIB 프록시)

```java
@Service
@RequiredArgsConstructor
public class ProductionOrderAppService {

    @Transactional
    public ProductionOrder create(...) { ... }
}
```

Spring 컨테이너 초기화 과정:

```
① 클래스로더가 ProductionOrderAppService.class 로드
   → Metaspace: [ProductionOrderAppService]

② BeanPostProcessor가 @Transactional 감지
   → CGLIB이 런타임에 서브클래스를 바이트코드로 생성

③ 생성된 프록시:
   ProductionOrderAppService$$SpringCGLIB$$0

   → Metaspace: [ProductionOrderAppService$$SpringCGLIB$$0]
```

실제 메서드 호출 흐름:

```
appService.create(...)
        │
        ▼
ProductionOrderAppService$$SpringCGLIB$$0  ← 프록시
   │
   │  1) TransactionInterceptor.invoke()
   │     → 트랜잭션 시작 (BEGIN)
   │
   │  2) 원본 메서드 위임
   │     → ProductionOrderAppService.create()
   │
   │  3) 정상 완료 시 COMMIT / 예외 시 ROLLBACK
   │
   ▼
결과 반환
```

**Metaspace에는 2개가 올라갑니다:**

```
Metaspace
├── ProductionOrderAppService                    ← 원본
└── ProductionOrderAppService$$SpringCGLIB$$0    ← 프록시
```

#### 3-3. @Transactional + @PreAuthorize 겹침

```java
@Service
public class ProductionOrderAppService {

    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public void delete(...) { ... }
}
```

AOP 어노테이션이 2개 이상이라도 프록시 클래스는 **1개**만 생성됩니다. 프록시 내부에 인터셉터 체인이 쌓이는 구조입니다.

```
Metaspace (여전히 2개)
├── ProductionOrderAppService
└── ProductionOrderAppService$$SpringCGLIB$$0

프록시 내부 인터셉터 체인:
   ┌─ MethodSecurityInterceptor  (@PreAuthorize)
   ├─ TransactionInterceptor     (@Transactional)
   └─ 원본 메서드 호출
```

따라서 `@Transactional`(119개) + `@PreAuthorize`(53개)가 상당 부분 같은 클래스에 붙어있으므로, 실제 프록시 수는 어노테이션 합계(172)가 아니라 **빈 수(\~280)에 가깝습니다**.

#### 3-4. @Repository (JPA 인터페이스 → JDK 동적 프록시)

```java
public interface ProductionOrderRepository
    extends JpaRepository<ProductionOrder, Long> {

    List<ProductionOrder> findByStatus(String status);
}
```

인터페이스만 존재하고 구현체가 없으므로 Spring Data JPA가 **JDK 동적 프록시**를 생성합니다.

```
Metaspace
├── ProductionOrderRepository   ← 인터페이스 메타데이터
├── SimpleJpaRepository         ← 공통 구현체 (모든 JPA Repository가 공유, 1개)
└── $Proxy123                   ← JDK 동적 프록시 (런타임 생성)
```

CGLIB 프록시와 JDK 동적 프록시의 차이:

|           | CGLIB 프록시                | JDK 동적 프록시                |
| --------- | ------------------------ | ------------------------- |
| 대상        | 구체 클래스                   | 인터페이스                     |
| 방식        | 서브클래스 바이트코드 생성           | `java.lang.reflect.Proxy` |
| 사용처       | `@Service`, `@Component` | `@Repository` (JPA)       |
| Metaspace | 원본 + `$$SpringCGLIB$$`   | 인터페이스 + `$Proxy`          |

#### 3-5. 요약 테이블

| 케이스                                             | Metaspace에 올라가는 클래스 수         |
| ----------------------------------------------- | ----------------------------- |
| `@Component` (AOP 없음)                           | 1개 (원본만)                      |
| `@Service` + `@Transactional`                   | 2개 (원본 + CGLIB 프록시)           |
| `@Service` + `@Transactional` + `@PreAuthorize` | 2개 (프록시 1개에 인터셉터 체인)          |
| `@Repository` (JPA 인터페이스)                       | 2\~3개 (인터페이스 + 구현체 + JDK 프록시) |
| `@Configuration`                                | 2개 (원본 + CGLIB, Bean 싱글톤 보장용) |

***

### 4장. Metaspace 모니터링: jstat vs jcmd

#### jstat — "얼마나 쓰고 있는가"

`jstat`은 Metaspace의 **크기와 사용량**만 보여줍니다.

```bash
# PID 확인
jps -l

# Metaspace 사용량 확인
jstat -gc <PID>
```

주요 출력 컬럼:

| 컬럼              | 의미                                 |
| --------------- | ---------------------------------- |
| `MC`            | Metaspace Committed (OS에서 할당받은 크기) |
| `MU`            | Metaspace Used (실제 사용 중인 크기)       |
| `MCMN` / `MCMX` | 최소 / 최대 용량                         |

"지금 120MB 쓰고 있다"는 알 수 있지만, "그 120MB 안에 뭐가 있는지"는 알 수 없습니다.

#### jcmd — "무엇이 들어있는가"

실제 로드된 클래스와 인스턴스 정보를 보려면 `jcmd`를 사용합니다.

```bash
# 로드된 클래스별 인스턴스 수/메모리 크기
jcmd <PID> GC.class_histogram | head -30

# 출력 예시:
#  num   #instances  #bytes  class name
#    1:      45032  5765120  [B  (byte[])
#    2:      12847  1234512  java.lang.String
#    3:       3024   967680  o.h.persister.entity.SingleTableEntityPersister
#    4:        893   214320  ...$$SpringCGLIB$$0
#    ...
```

```bash
# 클래스 로더 계층 구조
jcmd <PID> VM.class_hierarchy
```

#### 비교 정리

| 목적        | 도구     | 명령어                             |
| --------- | ------ | ------------------------------- |
| 사용량 (얼마나) | jstat  | `jstat -gc <PID>`               |
| 내용물 (무엇이) | jcmd   | `jcmd <PID> GC.class_histogram` |
| 클래스 수 실시간 | jstat  | `jstat -class <PID>`            |
| 클래스 로딩 로그 | JVM 옵션 | `-verbose:class`                |
| 시각적 분석    | GUI 도구 | VisualVM, JConsole              |

운영 환경에서 Metaspace가 `MaxMetaspaceSize`에 근접하고 있다면, 먼저 `jstat -gc`로 사용량 추이를 확인하고, `jcmd GC.class_histogram`으로 어떤 클래스가 점유하고 있는지 파악하는 순서로 접근하면 됩니다.

***

### 5장. 컨테이너 환경에서 Metaspace의 함정

#### 핵심: 컨테이너는 힙이 아니라 총 RSS를 본다

JVM의 `-Xmx`는 **힙만** 제한합니다. 하지만 컨테이너의 `mem_limit`(Docker)이나 `resources.limits.memory`(K8s)는 \*\*프로세스 전체 RSS(Resident Set Size)\*\*를 감시합니다. Metaspace, Code Cache, Direct Memory, 스레드 스택 전부 포함입니다.

```
컨테이너 메모리 한도
│
├── Heap (-Xmx)                 ← JVM이 관리
├── Metaspace                   ← 힙 바깥, 하지만 컨테이너 메모리에 포함
├── Code Cache                  ← 힙 바깥, 하지만 컨테이너 메모리에 포함
├── Direct Memory               ← 힙 바깥, 하지만 컨테이너 메모리에 포함
├── Thread Stacks               ← 힙 바깥, 하지만 컨테이너 메모리에 포함
├── Native Libraries            ← 힙 바깥, 하지만 컨테이너 메모리에 포함
├── GC 내부 구조                 ← 힙 바깥, 하지만 컨테이너 메모리에 포함
└── OS/커널 오버헤드
    ─────────────────────
    합계가 한도를 넘으면 → OOMKilled (137)
```

중요한 차이가 있습니다. JVM의 `OutOfMemoryError`는 힙덤프도 남기고 스택 트레이스도 출력합니다. 하지만 컨테이너 OOMKilled는 **커널이 프로세스를 강제 종료**하는 것이므로, 로그도 힙덤프도 없이 Pod가 죽습니다. `Dockerfile-Azure`에 `-XX:+HeapDumpOnOutOfMemoryError`를 설정해두었지만, K8s OOMKilled는 JVM 바깥에서 일어나는 일이라 이 옵션으로 캡처되지 않습니다.

#### Dev 환경: 1G 컨테이너의 메모리 예산

Dev 환경은 Docker `mem_limit: 1G`에 고정 힙을 사용합니다.

```
컨테이너 한도: 1024MB

Heap          512MB  (-Xmx512m)          ██████████████████████████░░░░░░░░
Metaspace     200MB  (MaxMetaspaceSize)  ██████████░░░░░░░░░░░░░░░░░░░░░░░
Code Cache    128MB  (ReservedCodeCache)  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░
Direct Mem     64MB  (MaxDirectMemory)    ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
─────────────────────────────
소계 (고정)   904MB
남은 여유     120MB  ← 스레드 스택 + GC + 네이티브 전부 여기서
```

스레드 스택(`-Xss512k`)은 스레드 수에 비례합니다. Tomcat 기본 스레드 200개 + GC 스레드 + 비동기 스레드 등을 합치면 피크 시 100개 이상이 될 수 있습니다. `100 × 512KB = 50MB`입니다.

거기에 Firebase SDK가 내부적으로 사용하는 gRPC/Netty의 네이티브 버퍼, G1GC의 Card Table과 Remembered Set 등을 더하면 **120MB 여유로는 빠듯합니다**. 부하가 몰리면 OOMKilled가 발생할 수 있는 구조입니다.

#### 운영 K8s: 1Gi → 2Gi 업그레이드 후

운영 환경은 `MaxRAMPercentage=50.0`으로 힙을 컨테이너 메모리의 절반으로 자동 설정합니다.

**튜닝 전 (Pod 1Gi)**

```
Pod 한도: 1024MB

Heap          512MB  (1024 × 50%)        ██████████████████████████░░░░░░░░
Metaspace     200MB                       ██████████░░░░░░░░░░░░░░░░░░░░░░░
Code Cache    128MB                       ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░
Direct Mem     64MB                       ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
─────────────────────────────
소계          904MB
남은 여유     120MB  ← 위험
```

Dev와 사실상 동일한 메모리 예산입니다. 운영 환경은 트래픽이 더 많고, 동시 접속 테넌트도 많으므로 스레드 수가 피크 시 더 높아집니다. OOMKilled 위험이 있었습니다.

**튜닝 후 (Pod 2Gi)**

```
Pod 한도: 2048MB

Heap         1024MB  (2048 × 50%)        ██████████████████████████░░░░░░░░
Metaspace     200MB                       █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Code Cache    128MB                       ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Direct Mem     64MB                       ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
─────────────────────────────
소계         1416MB
남은 여유     632MB  ← 충분
```

| 비교 항목         | 1Gi Pod   | 2Gi Pod   | 변화     |
| ------------- | --------- | --------- | ------ |
| Heap          | 512MB     | 1024MB    | 2배     |
| Metaspace     | 200MB     | 200MB     | 동일     |
| Code Cache    | 128MB     | 128MB     | 동일     |
| Direct Memory | 64MB      | 64MB      | 동일     |
| **남은 여유**     | **120MB** | **632MB** | **5배** |

`MaxRAMPercentage=50.0`의 장점이 여기서 드러납니다. Pod 크기를 올리면 힙은 자동으로 비례 증가하고, Non-Heap(Metaspace, Code Cache 등)은 고정이므로 **여유 공간이 비례 이상으로 늘어납니다**.

#### Non-Heap이 고정인 이유

Metaspace(200MB), Code Cache(128MB), Direct Memory(64MB)는 Pod 크기와 관계없이 동일합니다. 이 영역들의 크기를 결정하는 것은 컨테이너 크기가 아니라 **애플리케이션 구조**이기 때문입니다.

* Metaspace: 엔티티 77개, Spring 빈 280개 → Pod를 키워도 클래스 수는 그대로
* Code Cache: JIT 컴파일 대상 메서드 수 → 코드가 같으면 동일
* Direct Memory: Redis(Lettuce/Netty) 버퍼 → 커넥션 풀 설정에 의존

Pod 메모리를 늘려도 Non-Heap은 변하지 않습니다. 달라지는 건 힙 크기와 여유 공간뿐입니다. 이것이 `MaxRAMPercentage` 방식이 고정 `-Xmx`보다 컨테이너 환경에 유리한 이유입니다.

***

### 6장. Metaspace는 왜 단방향으로 증가하는가

#### 클래스(Metaspace)와 인스턴스(Heap)의 차이

```
클래스 = 붕어빵 틀        → Metaspace에 1개, 절대 복제되지 않음
인스턴스 = 구워진 붕어빵    → Heap에 N개, 요청마다 생기고 GC가 정리
```

이 구분을 이해하면 "Metaspace가 단방향으로 증가한다"는 말이 왜 "무한히 증가한다"는 뜻이 아닌지 알 수 있습니다.

#### LAZY 프록시가 Metaspace를 늘리는 실제 과정

다음과 같은 엔티티가 있다고 가정합니다.

```java
@Entity
public class ProductionOrder {

    @ManyToOne(fetch = FetchType.LAZY)
    private Product product;        // 아직 접근 안 함

    @ManyToOne(fetch = FetchType.LAZY)
    private User createdBy;         // 아직 접근 안 함
}
```

**앱 시작 직후** — LAZY 프록시는 아직 없습니다.

```
Metaspace
├── ProductionOrder  (엔티티 메타데이터)
├── Product          (엔티티 메타데이터)
├── User             (엔티티 메타데이터)
└── (프록시 클래스 없음)
```

**첫 번째 요청: 생산지시 조회**

```java
// Fetch Join이 아닌 일반 조회
ProductionOrder order = orderRepository.findById(1L);
```

이 쿼리가 실행되면 Hibernate는 `product` 필드에 실제 Product 객체가 아닌 **프록시**를 넣어둡니다.

```
Hibernate 내부 동작:

1. SELECT po.* FROM production_order po WHERE po.id = 1
   → ProductionOrder 엔티티 생성

2. product 필드가 LAZY
   → "지금 로드하지 말자"
   → ByteBuddy로 Product의 프록시 서브클래스 생성

3. 이 프록시 클래스가 처음 만들어지는 순간:
   Product$HibernateProxy$abc123.class ← Metaspace에 올라감
```

```
Metaspace (프록시 1개 추가)
├── ProductionOrder
├── Product
├── Product$HibernateProxy$abc123   ← 새로 생성됨
├── User
└── (User 프록시는 아직 없음)
```

**프록시 객체의 내부 구조:**

```
order.getProduct() 가 반환하는 것:

Product$HibernateProxy$abc123 인스턴스
├── id = 42                      ← FK에서 가져온 값 (쿼리 불필요)
├── name = null                  ← 아직 로드 안 됨
├── code = null
├── interceptor = LazyInitializer
│   └── "getName() 호출되면 그때 SELECT 실행"
└── loaded = false

        │  order.getProduct().getName() 호출 시
        ▼

        SELECT * FROM product WHERE id = 42

        │
        ▼

├── id = 42
├── name = "반도체 웨이퍼"        ← 이제 채워짐
├── code = "PRD-001"
└── loaded = true
```

프록시 **클래스** 자체는 `getName()` 호출 전, 엔티티를 로드하는 시점에 이미 Metaspace에 올라가 있습니다. 실제 데이터 로딩은 필드에 접근할 때 별도 쿼리로 수행됩니다.

**이후 다른 API에서 User에 처음 LAZY 접근:**

```java
ProductionOrder order = orderRepository.findById(2L);
order.getCreatedBy();  // User에 처음 LAZY 접근
```

```
Metaspace (프록시 2개로 증가)
├── ProductionOrder
├── Product
├── Product$HibernateProxy$abc123
├── User
├── User$HibernateProxy$def456      ← 이제 새로 생성됨
└── ...
```

#### 핵심: 프록시 클래스는 엔티티 타입당 1개

10개 테넌트(공장)에서 동시에 생산지시를 조회하는 상황을 봅니다.

```java
ProductionOrder orderA = repository.findById(1L);  // 테넌트 A
ProductionOrder orderB = repository.findById(2L);  // 테넌트 B
ProductionOrder orderC = repository.findById(3L);  // 테넌트 C
// ... 테넌트 J까지 10개
```

```
Metaspace (클래스 = 틀)
├── Product                              ← 원본 1개
├── Product$HibernateProxy$abc123        ← 프록시 틀 1개
└── 끝. 테넌트가 100개여도 여기는 2개.

Heap (인스턴스 = 구워진 붕어빵)
├── Product$HibernateProxy 인스턴스 #1   ← 테넌트 A의 product (id=42)
├── Product$HibernateProxy 인스턴스 #2   ← 테넌트 B의 product (id=77)
├── Product$HibernateProxy 인스턴스 #3   ← 테넌트 C의 product (id=103)
│   ...
└── Product$HibernateProxy 인스턴스 #10  ← 테넌트 J의 product (id=501)
```

`Product` LAZY 접근이 1000번 일어나도 `Product$HibernateProxy$abc123`는 **1개만** 만들어집니다. 1000개의 **인스턴스**(Heap)가 생기지만, **클래스**(Metaspace)는 1개입니다.

같은 원리를 String으로 보면 더 명확합니다.

```java
String a = "안녕";
String b = "하세요";
String c = "반갑습니다";
// ... 10000개
```

```
Metaspace: String.class              × 1개    (클래스 정의는 하나)
Heap:      String 인스턴스            × 10000개 (값이 다른 객체들)
```

#### EAGER vs LAZY — Metaspace 관점에서의 차이

```java
@Entity
public class ProductionOrder {

    @ManyToOne(fetch = FetchType.EAGER)  // 즉시 로딩
    private Product product;

    @ManyToOne(fetch = FetchType.LAZY)   // 지연 로딩
    private User createdBy;
}
```

```java
ProductionOrder order = orderRepository.findById(1L);
```

```
EAGER (product):
  → JOIN으로 즉시 가져옴
  → Product 실제 객체 반환 (프록시 아님)
  → Metaspace 추가 없음

LAZY (createdBy):
  → 쿼리 안 날림 (접근 시 별도 SELECT)
  → User$HibernateProxy 반환
  → Metaspace에 프록시 클래스 추가 (최초 1회만)
```

|           | EAGER     | LAZY               |
| --------- | --------- | ------------------ |
| SQL       | JOIN으로 즉시 | 접근 시 별도 SELECT     |
| 반환 타입     | 실제 객체     | 프록시                |
| Metaspace | 추가 없음     | 프록시 클래스 1개 (최초 1회) |

직접 조회(`productRepository.findById`)도 EAGER와 동일합니다. 실제 Product 객체를 반환하므로 프록시가 생기지 않고, Metaspace에 영향이 없습니다.

모든 관계를 EAGER로 바꾸면 프록시 클래스가 안 생기니까 Metaspace 사용량은 줄어듭니다. 하지만 매 쿼리마다 불필요한 JOIN이 수십 개 붙으므로 현실적이지 않습니다. LAZY + 필요 시 Fetch Join이, Metaspace를 수십 KB 더 쓰는 대신 쿼리 성능을 지키는 올바른 트레이드오프입니다.

#### 시간에 따른 Metaspace 증가 패턴

```
Metaspace
 사용량
  │
  │                                ┌─── 안정화 (plateau)
  │                          ╭─────
  │                     ╭────╯
  │                ╭────╯  ← 다양한 LAZY 관계가 처음 접근될 때마다 계단식 증가
  │           ╭────╯
  │      ╭────╯
  │   ╭──╯  ← 앱 시작: 엔티티 + Spring 빈 + CGLIB 프록시 대량 로드
  │  ╭╯
  ├───────────────────────────────────── 시간
  시작    수분      수시간       수일
```

```
[09:00] 앱 시작
        → 엔티티 77개 + Spring 빈 프록시 로드 → 약 150MB

[09:05] 생산지시 조회 → product LAZY
        → Product 프록시 클래스 최초 생성 → Metaspace +1

[09:05] 생산지시 조회 2건째 → 또 product LAZY
        → 프록시 클래스 이미 있음 → Metaspace 변화 없음

[09:10] 입고검사 조회 → InspectionForm.product LAZY
        → Product 프록시 이미 있음 (같은 타입) → 변화 없음

[09:15] 출하 조회 → ShippingOrder.vendor LAZY
        → Vendor 프록시 최초 생성 → Metaspace +1

[09:30] 설비 이력 → EquipmentHistory.equipment LAZY
        → Equipment 프록시 최초 생성 → Metaspace +1

[10:00] LOT 추적 → Lot.product LAZY
        → Product 프록시 이미 있음 → 변화 없음

        ...

[며칠 후] 모든 엔티티 타입이 한 번 이상 LAZY 접근됨
        → 안정화 (더 이상 새로운 타입 없음)
```

하루에 API 요청이 10만 건이라 해도:

```
                  Metaspace              Heap
                  ─────────              ────
앱 시작           +1 (프록시 클래스)       0
첫 요청           변화 없음               +1 (인스턴스)
1000번째 요청      변화 없음               +1 (인스턴스)
GC 발생           변화 없음               -900 (인스턴스 정리)
10000번째 요청     변화 없음               +1 (인스턴스)
```

"단방향 증가"란 매 요청마다 늘어난다는 뜻이 아닙니다. **새로운 엔티티 타입을 처음 LAZY 접근할 때만** 계단식으로 올라가고, 모든 타입이 접근되면 멈춥니다. 우리 서비스에서 LAZY 대상 엔티티가 약 40~~50개 타입이라면, 프록시 클래스도 최대 40~~50개가 추가되고 그 이후로는 안정화됩니다.

***

### 7장. MetaspaceSize vs MaxMetaspaceSize

#### Metaspace인데 왜 GC가 발생하는가

"Metaspace는 힙 바깥인데 왜 GC가 관여하지?"라는 의문이 자연스럽습니다.

Metaspace가 힙 바깥에 있는 건 맞지만, JVM은 Metaspace 공간이 부족할 때 **Full GC를 트리거해서 클래스 언로딩을 시도**합니다. 클래스를 언로딩하려면 그 클래스를 로드한 ClassLoader가 GC 대상인지 확인해야 하고, 그 판별은 힙의 GC 과정에서만 가능하기 때문입니다.

```
Metaspace 임계값 도달
      │
      ▼
"공간이 부족하다, 클래스를 정리하자"
      │
      ▼
Full GC 트리거 (Stop-The-World)
      │
      ├── 힙 GC 수행 (Young + Old)
      │
      └── ClassLoader가 GC 대상인지 확인
            │
            ├── ClassLoader가 아직 살아있음 → 해당 클래스 언로딩 불가
            └── ClassLoader가 GC됨 → 해당 클래스 전부 언로딩
```

#### 현재 설정에 빠진 것

현재 Dockerfile에 `MaxMetaspaceSize=200m`만 있고 `MetaspaceSize`는 설정되어 있지 않습니다. 이 둘은 전혀 다른 역할입니다.

| 옵션                 | 역할                             | 기본값    |
| ------------------ | ------------------------------ | ------ |
| `MetaspaceSize`    | Metaspace GC가 **처음 트리거되는 임계값** | \~20MB |
| `MaxMetaspaceSize` | Metaspace **절대 상한**            | 무제한    |

#### 미설정 시 시작 과정에서 벌어지는 일

`MetaspaceSize` 기본값 약 20MB. 앱 시작 시 클래스를 막 로드하는데 20MB만 채워도 Full GC가 발동합니다. GC해봤자 언로딩할 클래스가 없으니 임계값을 올리고, 다시 채우고, 또 GC하고를 반복합니다.

```
MetaspaceSize 미설정 시:

[0.0초] Spring Boot 시작, 클래스 로드 중...

[0.5초] Metaspace 20MB 도달 (임계값)
        → Full GC 실행 (Stop-The-World)
        → 언로딩 0개 (ApplicationClassLoader 살아있음)
        → 임계값 ~40MB로 상향

[1.2초] Metaspace 40MB 도달
        → Full GC 실행
        → 언로딩 0개
        → 임계값 ~80MB로 상향

[2.5초] Metaspace 80MB 도달
        → Full GC 실행, 언로딩 0개, 임계값 → ~150MB

[3.5초] Metaspace 150MB 도달
        → Full GC 실행, 언로딩 0개, 임계값 → ~200MB

[4.0초] 앱 시작 완료. Metaspace 안정화 약 160MB.

불필요한 Full GC: 4회, 낭비 시간: 수백 ms ~ 수 초
```

```
MetaspaceSize=200m 설정 시:

[0.0초] Spring Boot 시작, 클래스 로드 중...
[0.5초] Metaspace 20MB... 임계값 200MB니까 여유 → GC 없음
[1.5초] Metaspace 100MB... → GC 없음
[2.5초] Metaspace 160MB... → GC 없음, 안정화
[3.0초] 앱 시작 완료.

불필요한 Full GC: 0회
```

#### 왜 언로딩이 안 되는가

Spring Boot 앱에서 클래스를 로드하는 건 ApplicationClassLoader 하나입니다. 이 ClassLoader는 앱이 살아있는 동안 절대 GC되지 않습니다.

```
ApplicationClassLoader
  └── 280개 빈 전부 이 ClassLoader가 로드
  └── 빈들은 ApplicationContext에 의해 참조됨
  └── ApplicationContext는 앱 종료까지 살아있음
  └── ClassLoader도 앱 종료까지 살아있음
  └── 클래스 언로딩 = 항상 0개
```

Full GC를 해봤자 정리할 클래스가 없는데, JVM은 해보기 전에는 모릅니다. "일단 GC 돌려서 확인해보자" → "없네" → "임계값 올리자"를 반복하는 것입니다. `MetaspaceSize=200m`은 "200MB 전까지는 확인할 필요 없다"고 JVM에게 알려주는 것입니다.

#### 우리 서비스에서의 실제 영향도

솔직히 말하면 **체감이 크지 않습니다**.

```
우리 Spring Boot 앱의 시작 시간 구성 (추정):

Spring Context 초기화         ~5-10초
Hibernate SessionFactory      ~3-5초   (77개 엔티티 메타모델)
컴포넌트 스캔 + 빈 생성        ~2-3초   (280개 빈)
Firebase SDK 초기화            ~1-2초
Flyway 마이그레이션 확인        ~1초
Tomcat 시작                    ~1초
MetaspaceSize 미설정 Full GC   ~0.5-2초  ← 전체의 5~10%
────────────────────────────
전체                          약 15-25초
```

그리고 **시작 이후에는 영향이 0**입니다.

```
시작 이후 (앱이 떠 있는 동안):
- Metaspace 160MB 안정화 → 200MB 상한에 도달 안 함
- Metaspace GC 트리거 안 됨
- MetaspaceSize 설정 유무와 관계없이 성능 차이 = 0
```

설정이 의미 있는 경우와 아닌 경우:

| 상황                              | 영향도    |
| ------------------------------- | ------ |
| 한 번 뜨면 오래 떠 있는 서비스 (우리)         | 낮음     |
| 배포 빈도 낮음 - 하루 1-2회 (우리)         | 낮음     |
| K8s start-period=90초 (우리)       | 충분히 여유 |
| Serverless / Lambda (매번 콜드 스타트) | 높음     |
| 수시 재시작하는 서비스                    | 높음     |
| start-period가 짧아 헬스체크 빡빡한 경우    | 높음     |

**결론**: 설정하면 좋지만 우선순위가 높지는 않습니다. 시작 시 Full GC 4번이 0번이 되는 건 맞지만, 그 차이가 0.5\~2초이고, 앱이 한 번 뜨면 그 이후로는 차이가 없습니다. 다음에 Dockerfile 수정할 일이 있을 때 한 줄 추가하면 되는 정도입니다.

***

### 결론. Metaspace를 이해하면 보이는 것들

#### 현재 설정 검토

| 환경           | 컨테이너 | 힙 설정                            | Metaspace | 여유           |
| ------------ | ---- | ------------------------------- | --------- | ------------ |
| Dev (Docker) | 1GB  | `-Xms256m -Xmx512m`             | `200m`    | \~120MB (빠듯) |
| Azure K8s    | 2GB  | `MaxRAMPercentage=50%` (1024MB) | `200m`    | \~632MB (충분) |

77개 엔티티, 280개 Spring 빈, 109개 생성 코드, 서드파티 라이브러리를 고려했을 때 Metaspace 200MB는 현재 규모에서 넉넉한 편입니다. K8s를 2Gi로 올린 덕분에 운영 환경의 전체 메모리 여유도 확보되었습니다.

Dev 환경은 여전히 1G로 여유가 120MB 수준인데, 개발 환경이라 동시 접속이 적어 문제가 되지 않지만, 부하 테스트 시에는 주의가 필요합니다.

#### 추후 검토 영역

* **Compressed Class Space**: Metaspace 내부에서 Klass 포인터를 32bit로 압축하는 전용 영역. `UseCompressedClassPointers`가 Java 17에서 기본 ON이라 별도 설정할 일은 거의 없다.
* **Metaspace 메모리 릭 패턴**: ClassLoader 릭, devtools 핫 리로드 반복 등. 우리 서비스는 SCHEMA 멀티테넌시(단일 SessionFactory)이고 devtools를 운영에서 사용하지 않아 해당 사항 없음.

#### 돌아보며

Metaspace는 평소에는 의식하지 않는 영역입니다. 하지만 그 안에 무엇이 들어가는지 이해하면, JVM 설정값이 단순한 숫자가 아니라 우리 애플리케이션 구조의 반영이라는 것을 알 수 있습니다.

`@Transactional` 하나 붙이면 CGLIB 프록시가 생기고, LAZY 관계 하나 추가하면 ByteBuddy 프록시가 생깁니다. 그리고 컨테이너 환경에서는 이 Metaspace가 힙 바깥에서 조용히 메모리를 차지하고 있다가, 한도를 넘기면 로그 한 줄 없이 Pod가 죽습니다. "왜 200MB인가?", "왜 2Gi가 필요한가?"에 대한 답은 결국 우리 코드와 인프라 구조를 함께 이해해야 나옵니다.

***

### 참고

* Oracle, "Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide" — Metaspace 섹션
* Baeldung, "Metaspace in Java" — PermGen에서 Metaspace로의 전환 설명
* Spring Framework Reference — AOP Proxying Mechanisms (CGLIB vs JDK Dynamic Proxy)
* Hibernate ORM 6.x User Guide — ByteBuddy proxy generation
* JDK Tools Reference — jstat, jcmd 사용법
