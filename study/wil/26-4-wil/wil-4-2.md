# WIL: 이번주 공부(4월 2주차)

TL;WR JVM Non-Heap 4개 영역(Metaspace, Direct Memory, Code Cache, Thread Stacks)을 우리 서비스 코드베이스 기반으로 전수 분석하고 블로그 4편을 작성했다. 동시에 개인 권한(UserPermission) 기능의 백엔드 5 Phase 구현 + Figma 디자인 기반 프론트엔드 구현을 진행했다.

### 좋았던 점 & 배웠던 점

#### 1. JVM Non-Heap 4개 영역을 시리즈로 완주한 것

지난주(W1) Metaspace에서 시작해서 이번주에 Direct Memory, Code Cache, Thread Stacks까지 전부 다뤘다. 각 영역을 "우리 서비스 코드에서 구체적으로 뭐가 올라가는가"로 분석하니 숫자가 단순한 설정값이 아니라 애플리케이션 구조의 반영이라는 게 느껴졌다.

| 영역                    | 크기 결정 요인 | 우리 서비스 기준                               |
| --------------------- | -------- | --------------------------------------- |
| Metaspace 200MB       | 클래스 수    | 50개 이상 Entity + 300개 이상 Bean + 기타 생성 코드 |
| Direct Memory 64MB    | NIO 채널 수 | Redis + gRPC 기반 코드 + Tomcat만 사용         |
| Code Cache 128MB      | 핫 메서드 수  | 수만 메서드 JIT 대상                           |
| Thread Stacks 512KB/개 | 동시 스레드 수 | 47\~267개 (트래픽에 따라 동적)                   |

특히 "왜 이 값인가?"를 각 영역마다 코드 근거로 답할 수 있게 된 것이 가장 큰 수확이다.

#### 2. Direct Memory에서 균형 잡힌 시각을 얻은 것

처음에 "Heap Buffer도 결국 Direct로 복사하는데, 전부 Direct에 넣으면 되지 않나?"라는 혼란이 있었다. 대부분의 NIO 블로그가 Direct Buffer의 장점만 강조해서 생긴 혼란이었다.

파고 들어가니 "Direct Memory 자체는 느리다. Netty 풀링이 결합될 때만 빠르다. 그리고 I/O 빈도가 높아야 의미 있다"라는 조건이 있었다. 기술의 장점만 보지 말고 **적용 조건**을 따져야 한다는 걸 배웠다.

MySQL이 블로킹 Socket을 유지하는 이유도 "JDBC 스펙 자체가 블로킹 API"라는 근본적 제약 때문이었고, OpenJDK 소스코드(`IOUtil.java`)에서 Heap → Direct 복사가 실제로 일어나는 코드를 확인한 것이 인상적이었다.

#### 3. "From Packet To Object" 슬라이드와의 교차 검증

동료가 발표한 "전기 신호 → DMA → 프로토콜 스택 → JVM" 슬라이드와 우리 블로그 내용을 매칭했다. 슬라이드가 전체 파이프라인을 넓게, 우리 블로그가 JVM 내부를 깊게 다뤄서 상호 보완적이었고, 상충하는 내용이 없었다. mmap, io\_uring 같은 고급 최적화 기술도 "뭘 제거하는가" 기준으로 4단계로 분류하니 이해가 수월했다.

***

### 아쉬운 점

#### 1. 블로그 글이 너무 길어졌다

Direct Memory 블로그가 9장까지 갔다. 처음에는 "Direct Memory란 무엇인가"로 시작했는데, NIO, Netty, 풀링, MySQL 근거, Java 21 변화, mmap/io\_uring, OOM 시나리오까지 확장되면서 하나의 글이 감당할 범위를 넘었다. 주제가 2개 이상이면 글을 나누라는 블로그 컨벤션을 좀 더 엄격하게 지켰어야 했다.

#### 2. 실측 데이터가 없다

4개 영역을 전부 코드 분석 + 추정치로 다뤘다. 실제 운영 Pod에서 `jcmd VM.native_memory summary`를 돌려서 Metaspace, Code Cache, Direct Memory, Thread Stack의 실측값을 확인했으면 추정과 현실의 차이를 검증할 수 있었을 것이다. `-XX:NativeMemoryTracking=detail`이 이미 설정되어 있으니, 다음에 운영 서버에 접근할 기회가 되면 반드시 확인해야 한다.

#### 3. Code Cache + Warmup 블로그가 상대적으로 얕다

Metaspace(7장), Direct Memory(9장)에 비해 Code Cache(4장)는 깊이가 부족했다. 카카오페이 아티클 기반으로 웜업 전략을 정리했지만, 우리 서비스에 실제로 ApplicationRunner 웜업을 구현해보지는 못했다. 코드 예시까지 넣었으니 다음 스프린트에서 실제 적용을 시도해볼 만하다.

***

### 종합 정리

이번 주는 "JVM 설정값의 의미를 코드 수준에서 이해하기"를 테마로 진행했다. `MaxMetaspaceSize=200m`이 왜 200인지, `MaxDirectMemorySize=64m`이 왜 64인지, `ReservedCodeCacheSize=128m`이 왜 128인지, `-Xss512k`가 왜 512인지를 각각 우리 서비스의 클래스 수, 통신 경로, 핫 메서드 수, 스레드 수로 설명할 수 있게 되었다.

4개 영역을 관통하는 공통점도 발견했다. "앱 시작 직후에는 비어있고, 운영 중에 채워지고, 가득 차면 문제가 생긴다"는 것이다. 다만 문제의 형태가 다르다. Metaspace/Direct는 OOM, Code Cache는 성능 저하(죽지 않고 느려짐), Thread Stacks는 트래픽에 따라 동적 변동. 이 차이를 알아야 장애 시 어디를 볼지 판단할 수 있다.

기회가 되면  JVM 학습 내용을 실측 데이터로 검증하고, 가능하면 ApplicationRunner 웜업을 실제로 적용해보고 싶다.
