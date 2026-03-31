---
hidden: true
---

# WIL: 이번주 공부(3월 4주차)

## WIL: 이번주 공부 (3월 4주차)

TL;WR HikariCP 풀 사이징부터 JVM Non-Heap OOM 사건, Prometheus+Loki 통합 Observability 구축, N+1 쿼리 24건 일괄 수정, QueryDSL StackOverflow까지 — 추측이 아닌 데이터 기반으로 장애를 분석하고 근본 원인을 코드 레벨에서 해결한 주였다.

### 좋았던 점 & 배웠던 점

#### 1. HikariCP 풀 사이징에 과학적 근거가 있다는 깨달음

기존에는 "10개 정도면 괜찮을 거야"라는 막연한 추측으로 풀 사이즈를 정했다. 이번에 HikariCP 저자(brettwooldridge)의 공식과 우리 애플리케이션의 실제 커넥션 소비 주체를 정밀하게 분석하며, 각 스레드 풀별로 몇 개의 커넥션을 얼마나 동시에 잡을 수 있는지 파악했다.

공식 `pool size = (CPU core × 2) + effective_spindle_count`은 기본이지만, 멀티테넌시, 비동기 작업, 배치 스레드풀의 조합으로 실제 필요한 크기를 산정했다:

* HTTP 요청: Tomcat 스레드
* 비동기 작업: notificationTaskExecutor, taskExecutor
* 스케줄러: notification-scheduler, TenantSchemaCache

각각 몇 개의 커넥션을 동시에 필요로 하는지 명확히 했고, 피크의 70% 수준으로 풀을 설계하는 게 왜 타당한지 이해했다. 또한 MySQL의 `max_connections`를 고려하여 멀티 Pod 배포 시 안전 범위를 검증했다.

#### 2. Spring Boot Profile YML 우선순위 함정

`application.yml`의 `on-profile: azure` 블록에서 설정을 수정했는데, 실제로는 `application-azure.yml` 파일이 그것을 덮어쓰고 있었다는 사실을 Grafana 메트릭으로 발견했다.

Spring Boot에서 profile-specific 파일(`application-{profile}.yml`)이 `application.yml` 내의 같은 profile 블록보다 우선순위가 높다는 점을 명확히 학습했다. 설정을 수정할 때 어느 파일을 건드릴지 먼저 파악해야 한다는 원칙을 몸으로 익혔다.

#### 3. HikariCP 주요 파라미터별 설정 전략

단순히 "값을 크게 하면 된다"고 생각했던 각 설정이 고유한 목적을 가지고 있음을 이해했다:

* `connection-timeout`: 개발은 3초(빨리 실패해야 원인을 빨리 캐치), 운영은 5초(순간 피크 대응)
* `maximum-pool-size = minimum-idle` (고정 풀): idle→active 전환 비용 제거로 트래픽 급증 시 응답 지연 방지
* `idle-timeout`: 고정 풀에서는 거의 무의미하지만 나중의 유연성을 위해 설정
* `max-lifetime`: **반드시** MySQL `wait_timeout`보다 짧아야 함 (DB가 먼저 끊으면 좀비 커넥션 발생)

#### 4. OOMKilled ≠ Heap OOM — Non-Heap 메모리의 중요성

가장 큰 충격이었던 부분이다. Grafana에서 Heap 사용률이 512MB 중 448MB로 정상 범위였는데, K8s Pod가 `OOMKilled`로 강제 종료되었다.

이는 **Heap이 아닌 Non-Heap 메모리 영역이 1Gi 컨테이너 제한을 초과했기 때문**이었다:

* **CodeCache** (JIT 컴파일 코드): 기본 240MB, 상한 없음
* **Direct Memory** (NIO/Netty 버퍼): 기본값 = Heap 크기(512MB), 상한 없음
* **Metaspace** (클래스 메타데이터): 상한 설정됨
* **Thread Stacks**: 기본 1MB × 스레드 수

Prometheus 메트릭(jvm\_memory\_used\_bytes)은 Heap과 Metaspace 위주로 구성되어 있어서, CodeCache와 Direct Memory의 증가를 사전에 감지할 수 없었다. **컨테이너 RSS(실제 사용 메모리)와 JVM 메트릭 사이의 갭**이 존재한다는 것을 체득했다.

#### 5. JVM 메모리 예산을 명시적으로 설계해야 한다

컨테이너 환경에서는 Heap만 설정하면 안 되고, 모든 Non-Heap 영역에 상한을 명시적으로 설정해야 한다:

```dockerfile
# Pod 1Gi 기준
-Xmx512m                       # Heap 512MB
-XX:MaxMetaspaceSize=200m      # 클래스 메타데이터 200MB
-XX:ReservedCodeCacheSize=128m # JIT 코드 128MB (기본 240MB → 축소)
-XX:MaxDirectMemorySize=64m    # NIO 버퍼 64MB (기본 무제한 → 제한)
-Xss512k                       # 스레드 스택 512KB (기본 1MB → 축소)
```

이렇게 설정하면 Heap 512MB + Non-Heap 약 467MB = 총 약 979MB로 1Gi 컨테이너 안에 안전하게 들어간다.

특히 `-Xmx`와 `mem_limit`이 같으면 Non-Heap을 위한 공간이 0이 되어 **무조건 OOM이 발생하는 구조**임을 명확히 했다.

#### 6. G1GC 선택과 GC 튜닝은 근본 해결책이 아니다

개발 서버에서 Serial GC를 G1GC로 변경하고 Heap을 1GB로 늘렸더니 수일간 안정적이었다. 하지만 Non-Heap 상한이 없으면 시간이 지나면서 CodeCache와 Direct Memory가 계속 증가하고, 결국 OOM이 반복된다.

GC 튜닝은 **Heap 내부 관리만 담당**한다. Non-Heap 영역의 무제한 증가는 GC와 무관하다. 따라서 "GC를 적절히 조정했으니 안전하다"는 착각에서 벗어나 모든 메모리 영역을 아우르는 전체 설계가 필요하다는 점을 학습했다.

#### 7. Prometheus 메트릭 장기 추이가 조기 경고 신호다

운영과 개발 모두에서 `jvm_memory_usage_after_gc_percent`가 재시작 후 12%에서 매일 약 2%씩 상승하는 패턴을 관찰했다. 이 추이가 보이면 약 4\~5일 후 OOM을 **정확히 예측할 수 있다**.

또한 `kubectl describe pod`에서 Last State와 RESTARTS 횟수를 확인하는 것이 중요함을 알았다. 재시작이 빈번한 Pod는 그냥 "재시작하고 복구되니 괜찮네"로 넘어가기 쉽지만, 실제로는 반복되는 장애의 신호다.

#### 8. Dockerfile에 JVM 옵션을 문서화하면 나중의 혼동을 방지할 수 있다

각 JVM 플래그가 왜 필요한지, 어떤 메모리 영역을 제어하는지, 목표 컨테이너 환경이 무엇인지를 주석으로 명시하면, 나중에 "이 플래그는 왜 있지?"라는 의문이나 무분별한 수정을 줄일 수 있다.

#### 9. Prometheus + Loki 통합으로 "왜"를 볼 수 있게 됐다

Prometheus 메트릭만으로는 "무슨 일이 있었는지"는 알 수 있지만 "왜 발생했는지"는 알 수 없었다. Loki 앱 로그와 결합한 통합 Observability 대시보드(5개 Row, 15개 패널)를 구축하여, GC 스파이크 시점에 어떤 API가 호출됐는지, N+1 경고가 응답 시간에 어떤 영향을 줬는지 상관관계를 한 화면에서 확인할 수 있게 됐다.

Loki에서 `level` 라벨의 대소문자 차이(`ERROR` vs `error`)로 전체 패널이 No data가 된 함정도 있었다. `(?i)` 플래그로 대소문자 무관 매칭하는 것이 안전하다는 교훈을 얻었다.

#### 10. N+1은 단순 쿼리 문제가 아니라 GC/커넥션/CPU 삼중 문제다

운영 7일간 데이터를 대시보드에서 종합 분석하며, N+1 쿼리가 GC 압박(Eden 소진 + Old Gen 승격), 커넥션 점유(트랜잭션 동안 잠김), CPU 사용(라운드트립 반복) 세 가지에 동시에 영향을 준다는 것을 확인했다. 03/21 장애의 인과관계 체인이 명확해졌다: 페이징 없는 대량 조회 + N+1 → Humongous 객체 + Old Gen 축적 → GC Pause 600ms → 커넥션 풀 고갈 → CPU 100% → 서비스 다운.

JVM 플래그 튜닝은 증상 완화이고, N+1 해결 + 페이징 적용이 근본 치료라는 우선순위가 확립됐다.

#### 11. OOM 사전 감지 체계 — JVM 밖에서 발생하는 문제 추적법

OOMKilled는 JVM이 아닌 Linux 커널/K8s kubelet이 발생시키므로 앱 로그에 남지 않고, `-XX:+HeapDumpOnOutOfMemoryError`도 무용지물이다. 이를 추적하기 위해 5계층(K8s 이벤트 → Prometheus 메트릭 → 컨테이너 메모리 → JVM 선행 지표 → Docker 환경) 트래킹 체계를 정리했고, 핵심 지표인 `container_memory_working_set_bytes / container_spec_memory_limit_bytes`를 대시보드 Row 7에 구현했다.

#### 12. MaxRAMPercentage의 유연성 — Pod 사이즈 변경에 자동 적응

운영 Pod를 1Gi에서 2Gi로 상향할 때, `MaxRAMPercentage=50%` 덕분에 Dockerfile 수정 없이 Heap이 512MB → 1024MB로 자동 적응했다. Non-Heap 상한은 전부 고정값이라 Pod 크기와 무관하게 동작한다. `requests.memory`도 같이 올려야 K8s 퇴거 우선순위가 안정적이라는 점도 학습했다.

#### 13. 대시보드 환경 이식에서 라벨 체계 차이의 함정

운영용 Observability 대시보드를 개발 Grafana에 Import했더니 전체 No data. 운영(K8s)은 `namespace`, `pod`로 식별하고, 개발(Docker)은 `job`, `instance`로 식별하는 근본적 차이 때문이었다. 단순 문자열 치환은 메트릭마다 라벨이 다르므로 위험하고, 메트릭 유형별 분리 변환 + 환경별 JSON 분리가 필요했다.

#### 14. N+1 쿼리 24건 일괄 수정 — 병렬 분석의 위력

4개 API 그룹(`lot-masters`, `production-results`, `sales-targets/plans`, `inbounds`)을 researcher 에이전트 4개로 병렬 분석하여, 18건의 N+1/성능 결함을 특정하고 수정했다. 이후 코드 리뷰에서 Page 파생 쿼리 오버로드 6건이 추가 발견되어 총 24건 수정.

발견된 패턴 4가지:

* **패턴 A**: JOIN FETCH 누락 (파생 쿼리 → `@Query` + JOIN FETCH 전환)
* **패턴 B**: 루프 안 단건 조회 → 배치 조회 + Map 그룹핑
* **패턴 C**: 루프 안 단건 저장 → `findAllById` + `saveAll` 배치
* **패턴 D**: Cross Join 버그 (FK 없는 JOIN → 엔티티 연관관계 JOIN으로 수정)

"같은 파일의 다른 메서드"에 이미 올바른 패턴이 있었다는 것, 같은 이름의 `List`/`Page` 오버로드를 놓치기 쉽다는 것이 실전에서 느낀 교훈이다.

#### 15. QueryDSL OR 중첩 StackOverflow — JVM 튜닝의 예기치 못한 부작용

N+1 수정 직후, `-Xss512k`로 줄인 스레드 스택이 QueryDSL `BooleanBuilder.or()` 수백 건 중첩의 재귀 직렬화에서 StackOverflow를 유발했다. 기존 1MB에서는 간신히 동작하던 코드가 512KB에서 한계를 넘은 것이다.

OR 중첩을 IN 쿼리로 변환하여 조건 수를 O(N) → O(1)로 줄여 해결했다. JVM 튜닝 후 전체 API 통합 테스트가 필수라는 점, 에러 자체의 스택 트레이스가 Humongous 객체로 GC 부하를 유발한다는 점도 확인했다.

***

### 아쉬운 점

#### 1. 프로덕션 배포 전에 모든 메모리 영역을 설계하지 않았다

처음부터 Non-Heap 상한을 고려한 전체 예산을 세웠다면, 운영에서 OOMKilled 사건을 예방할 수 있었다. 마이크로서비스 환경에서 컨테이너 메모리 제한은 금기가 아니라 **필수 설계 사항**임을 사전에 인식했어야 했다.

#### 2. Spring Boot Profile 우선순위에 대한 이해 부족

profile-specific 파일이 더 높은 우선순위를 가진다는 점을 미리 알았다면, 여러 파일을 수정했을 일이 없었다. Spring Boot 설정 계층을 정확히 파악하는 것이 얼마나 중요한지 교훈 삼았다.

#### 3. Prometheus 대시보드에 Non-Heap 메모리 모니터링을 미리 구성하지 않았다

CodeCache와 Direct Memory 메트릭이 Prometheus에는 수집되었지만 대시보드에 시각화되지 않아서, 사건이 터질 때까지 상태를 파악하지 못했다. 프로덕션 배포 전에 컨테이너 환경에 맞는 메트릭 기획이 필요하다.

#### 4. N+1 쿼리를 모니터링이 갖춰진 후에야 발견했다

N+1 경고가 7일 내내 발생하고 있었지만, Observability 대시보드를 만들고 나서야 비로소 확인했다. QueryCountFilter의 WARN 로그는 이미 출력되고 있었는데, 대시보드에서 시각화하기 전까지는 인지하지 못했다. 코드 레벨 경고 체계는 있었지만 가시성이 부족했던 것이다.

#### 5. 대시보드 환경별 라벨 체계를 사전에 고려하지 않았다

운영 대시보드를 만들 때부터 개발(Docker)과 운영(K8s)의 라벨 체계가 다르다는 것을 인지하고 템플릿 변수를 설계했다면, 나중에 개발용 JSON을 별도로 만드는 수고를 줄일 수 있었다.

#### 6. `-Xss512k` 적용 전 전체 API 경로 테스트를 하지 않았다

TIL에 "Dev에서 충분히 테스트 후 적용"이라고 경고까지 해놓고, 실제로는 일반적인 웹 요청만 테스트하고 배포했다. QueryDSL 대량 OR처럼 특정 코드 경로에서만 발생하는 문제는 단순 스모크 테스트로 잡을 수 없다. JVM 플래그 변경 시 전체 API 엔드포인트 호출 + 경계값 데이터가 포함된 통합 테스트가 필수다.

***

### 종합 정리

이번 주는 **"추측의 개발"에서 "데이터 기반의 개발"로 한 발 나아간 주**였다.

HikariCP 풀 사이징은 단순히 "느껴지는 대로" 하던 것에서, 애플리케이션의 실제 스레드풀 구성과 DB 워크로드를 분석해 과학적으로 산정하게 됐다. 더 중요한 것은, **컨테이너 환경에서 메모리는 단순히 "Heap"이 아니라 여섯 가지 영역의 합**이라는 명확한 이해다.

OOMKilled 사건은 고통스러웠지만, 덕분에 JVM 메모리 구조를 가장 깊게 학습한 경험이 됐다. 그리고 그 장애의 근본 원인이 N+1 쿼리와 페이징 없는 대량 조회였다는 것을 Observability 대시보드로 증명하고, 실제로 24건을 수정하여 코드 레벨에서 해결한 것이 이번 주의 가장 큰 성과다.

앞으로 새로운 서비스를 배포할 때마다 이 주차의 교훈을 떠올릴 것 같다:

1. Heap + Non-Heap 전체 예산을 먼저 세우기
2. 모든 메모리 영역에 명시적 상한 설정하기
3. Prometheus + Loki 통합 대시보드로 "무엇이" + "왜"를 동시에 보기
4. 메트릭의 장기 추이에서 조기 경고 신호 읽기
5. JVM 튜닝은 증상 완화, N+1/페이징 수정이 근본 치료
6. JVM 플래그 변경 후 전체 API 통합 테스트 필수
