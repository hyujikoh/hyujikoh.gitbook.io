---
hidden: true
---

# WIL: 이번주 공부(3월 3주차)

> TL;WR Java/JPA의 동작 원리를 깊이 파고든 과정부터, HikariCP 커넥션 누수 장애를 겪고 모니터링 + JVM 튜닝까지 이어진 과저까지. "알기만 했던" 개념들이 실전에서 중요하다고 느껴진 한주 였다.

### 좋았던 점 & 배웠던 점

#### 1. Java HashSet의 동작 원리 — DB collation과 Java 메모리의 차이

이전에도 java 에 대한 equals, hashcode 에 대해서 공부해봤었지만,&#x20;

`HashSet.contains()`가 `hashCode()` + `equals()` 두 단계를 거친다는 것을 직접 체험했다. 예를 들어  `"250G"`와 `"250g"`는 해시값부터 다르기 때문에 버킷 자체가 달라 매칭에 실패한다. DB의 `utf8mb4_general_ci` collation은 대소문자를 무시하지만 Java String은 바이트 레벨 비교가 기본이다. DB에서 조회한 데이터를 Java에서 Set으로 비교할 때 이 차이를 인식하지 못하면 매칭 실패가 발생한다.

#### 2. JPA IDENTITY vs SEQUENCE — Batch Insert가 안 되는 이유

`GenerationType.IDENTITY`(MySQL AUTO\_INCREMENT)는 INSERT 실행 후에야 ID를 알 수 있어서, Hibernate의 영속성 컨텍스트(Map\<ID, Entity>) 저장을 위해 매 persist()마다 즉시 INSERT를 실행한다. 이 때문에 batch insert가 원천 차단된다. 반면 `SEQUENCE` 전략은 INSERT 전에 ID를 확보할 수 있어 쓰기 지연 + batch가 가능하다.

MySQL에서 batch insert가 필요하면 JPA를 우회하여 `JdbcTemplate.batchUpdate()`를 사용해야 한다. 단, 불필요한 최적화보다는 실제 성능 이슈가 관측되었을 때 적용하는 게 맞다.

#### 3. @Transactional과 flush()는 완전히 다르다

`flush()`는 영속성 컨텍스트의 변경을 DB에 SQL로 전송하지만 트랜잭션은 열린 상태이고, `commit()`은 트랜잭션을 확정한다. Hibernate는 기본적으로 INSERT → UPDATE → DELETE 순서로 SQL을 실행하기 때문에, DELETE → INSERT 순서가 필요한 경우(유니크 제약 위반 방지 등) 중간에 `entityManager.flush()`를 명시적으로 호출해야 한다.

#### 4. Prometheus + Grafana 모니터링 대시보드 설계

장애 원인을 추적하면서 31개 패널의 대시보드를 설계했다. Micrometer의 HikariCP 메트릭이 histogram이 아닌 summary 타입이라 `histogram_quantile()`을 사용할 수 없고, `rate(_sum) / rate(_count)`로 구간 평균을 구해야 한다는 것을 삽질 끝에 알게 되었다.

상관관계 패널(커넥션 풀 사용률 + GC Pause + 5xx 에러를 하나의 차트에 표시)이 장애 원인 분석에 가장 효과적이었다. 어떤 지표가 먼저 튀었는지 한눈에 보인다.

#### 5. JVM 튜닝 실전 포인트

컨테이너 환경에서 `-Xmx` 미설정 시 JVM이 예상의 1/4만 힙을 할당한다는 사실, Serial GC가 서버 앱에 부적합하다는 것을 확인했다. 가장 뼈아팠던 것은 Dockerfile에서 JVM 플래그를 `-jar` 뒤에 넣으면 `main(String[] args)`로 전달되어 튜닝이 전혀 적용되지 않는다는 것. 코드 리뷰에서 발견되지 않았으면 "튜닝했는데 왜 안 먹히지?"가 될 뻔했다.

***

### 아쉬운 점

#### 1. 배포 직후 메트릭으로 성능 개선을 판단한 것

GC 튜닝 후 "Heap 사용률 85.9% → 13.9%"라는 수치를 봤지만, 수정 전은 16.8시간 운영 상태이고 수정 후는 재시작 1.1시간 상태였다. 공정한 비교가 아니다. GC 튜닝 효과는 동일 부하, 12시간+ 운영 후 장기 추이로 판단해야 한다.

***

### 종합 정리

3월 3주차는 Java/JPA의 내부 동작 원리를 파고들면서 시작해서, 실제 장애 대응과 모니터링 구축으로 마무리된 한 주였다.

주 초반에 HashSet의 hashCode/equals 동작, JPA IDENTITY 전략의 batch insert 제한, flush와 commit의 차이를 공부했는데, 이런 기초 지식이 주 후반의 HikariCP 장애 대응에서 "커넥션이 왜 반환되지 않는가"를 이해하는 토대가 되었다. DB 커넥션도 결국 "빌려 쓰고 반납하는" 리소스라는 점에서, try-catch의 catch 블록에서 리소스를 정리하지 않는 실수가 얼마나 치명적인지 직접 체감했다.

Prometheus + Grafana 대시보드를 설계하면서 PromQL 쿼리와 메트릭 타입(histogram vs summary)의 차이를 배웠고, JVM 튜닝에서는 Dockerfile 플래그 순서 같은 사소해 보이는 실수가 튜닝 전체를 무효화할 수 있다는 것을 알게 되었다.

개념적으로 당연한 이야기들이, 실제 장애와 연결되었을 때 완전히 다른 무게로 다가온다. 이번 주의 경험들이 앞으로의 코드 품질을 높여줄 것이라 확신한다.
