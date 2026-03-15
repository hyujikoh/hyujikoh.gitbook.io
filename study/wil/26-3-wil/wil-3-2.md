# WIL: 이번주 공부 (3월 2주차)

> TL;WR 쿼리 모니터링 통합, 2계층 캐싱 PR 리뷰 반영, MySQL EXPLAIN 분석, 에러 응답 구조 리팩토링까지 -- 코드 품질과 성능 양쪽을 동시에 개선한 한 주였다.

### 좋았던 점 & 배웠던 점

#### 1. P6Spy 통합과 필터 순서 제어로 쿼리 모니터링 기반을 다졌다

Hibernate StatementInspector와 P6Spy가 동시에 SQL을 가로채면서 충돌이 발생했는데, 같은 목적의 메커니즘이 두 개 있으면 하나로 통합해야 한다는 원칙을 체감했다. P6Spy의 `SimpleJdbcEventListener`로 카운팅을 옮기니 로깅과 카운팅이 한 경로로 깔끔하게 정리됐다.

추가로 `QueryCountFilter`에 `@Order(HIGHEST_PRECEDENCE + 1)`을 명시해서 Spring Security 필터보다 앞에 배치한 것도 의미 있었다. `@Order`가 빈 생성 순서가 아닌 필터 실행 순서를 제어한다는 점, 그리고 인증 쿼리까지 포함해서 전체 쿼리를 집계하려면 필터 순서가 중요하다는 것을 배웠다.

#### 2. MySQL EXPLAIN을 읽고 옵티마이저의 판단을 추적했다

LOT 추적 쿼리에서 Full Scan이 발생해 인덱스를 추가했는데, 옵티마이저가 조인 순서를 바꿔버리면서 인덱스를 무시하는 상황이 생겼다. `key = null`이면 인덱스 존재 여부가 아니라 옵티마이저가 선택하지 않은 것이라는 점, 그리고 테스트 환경의 적은 데이터(rows=2)가 실행 계획을 왜곡할 수 있다는 점이 인상적이었다.

결국 커버링 인덱스(`lot_master_id, is_deleted, shipping_order_id`)로 테이블 접근 자체를 줄이는 방향으로 해결했다. JPA가 SQL 생성까지만 관여하고 실행 계획은 MySQL이 전담한다는 구조를 명확히 이해하게 됐다.

#### 3. 2계층 캐싱 PR 리뷰에서 숨겨진 버그들을 발견했다

Gemini와 CodeRabbit에서 23건의 리뷰가 달렸는데, 오탐과 실제 이슈를 분류하는 과정 자체가 학습이었다. 특히:

* CachedUserInfo에 password 해시가 포함된 보안 이슈
* soft delete 사용자가 `findByUsernameAndState`로 재로드되는 기존 버그
* Caffeine loader에서 null 반환 시 NPE 발생

이런 것들은 캐시를 도입하지 않았으면 계속 숨겨져 있었을 문제다. 새로운 계층을 추가하면 기존 코드의 잠재 버그가 드러난다는 것을 경험했다. AI 리뷰 도구가 멀티테넌시 같은 도메인 맥락을 이해하지 못해 오탐을 내는 것도 확인했고, 결국 직접 검증이 필수라는 점을 느꼈다.

#### 4. 에러 응답을 BUSINESS/SYSTEM으로 구조화했다

백엔드 BaseResponse에 `errorType` 필드를 추가하고, 프론트엔드에서 `ApiError` 커스텀 클래스로 메타데이터를 보존하는 구조를 만들었다. 25개 페이지의 하드코딩된 에러 메시지를 `errorHandler`로 통합하는 작업이었는데, 에러 처리는 백엔드와 프론트엔드가 함께 설계해야 한다는 점을 다시 확인했다.

리팩토링 과정에서 `@ExceptionHandler` 어노테이션과 메서드 파라미터 타입 불일치라는 기존 잠재 버그도 발견했다. 리팩토링이 버그 탐지의 기회가 된다는 것을 실감했다.

#### 5. 4계층 아키텍처 원칙을 반복 적용하며 체득했다

Controller에서 비즈니스 로직을 분리하는 작업(PermissionGroup)과 알림 API 날짜 범위 필터 구현 모두 4계층 아키텍처 기준으로 진행했다. AppService가 여러 도메인 데이터를 반환해야 할 때 결과 record로 묶는 패턴, 날짜 변환(LocalDate -> LocalDateTime)은 Controller에서 처리하고 DomainService 이하는 LocalDateTime만 다루는 패턴 등이 자연스러워졌다.

***

### 아쉬운 점

#### 1. 테스트 환경과 운영 환경의 괴리

MySQL EXPLAIN을 로컬(rows=2)에서만 확인하고, k6 부하 테스트도 로컬 MySQL 대상으로만 돌렸다. 캐시 전/후 응답시간 차이가 미미하게 나왔는데, 이건 로컬 DB가 이미 충분히 빨라서 캐시 효과가 상쇄된 것이었다. 운영과 유사한 데이터 규모에서 검증하지 못한 점이 아쉽다.

#### 2. 트랜잭션과 캐시 eviction 타이밍 문제를 TODO로 남겼다

트랜잭션 커밋 전 eviction으로 인한 race condition을 인지하면서도 `TransactionSynchronization#afterCommit` 적용을 후속 PR로 미뤘다. 발생 확률이 낮다는 판단이었지만, 알고 있는 문제를 미루는 것이 올바른 선택이었는지 고민이 된다.

#### 3. Spring @Transactional vs Jakarta @Transactional 같은 기본기

IDE auto-import로 Jakarta 것이 들어온 것을 뒤늦게 발견한 것이 좀 부끄러웠다. `readOnly = true`가 Hibernate flush mode를 MANUAL로 바꾼다는 것도 이제야 정리했다. 기본적인 부분을 놓치지 않도록 주의해야겠다.

***

### 종합 정리

이번 주는 "기존 코드를 개선하면서 동시에 배우는" 흐름이었다. P6Spy 통합, 캐싱 PR 리뷰, 에러 응답 구조화, EXPLAIN 분석 등 대부분의 작업이 새 기능 개발보다는 기존 시스템의 품질을 높이는 방향이었다.

특히 PR 리뷰 과정에서 배운 것이 많았다. AI 리뷰 도구의 23건 코멘트를 하나씩 검증하면서 오탐과 실제 이슈를 분리하는 능력이 생겼고, 캐시 도입이 기존 코드의 숨겨진 버그(soft delete 미체크, password 캐싱)를 드러내는 경험도 했다.

다음 주에는 `TransactionSynchronization#afterCommit`으로 캐시 eviction 타이밍 문제를 해결하고, 운영 환경에 가까운 데이터로 EXPLAIN과 부하 테스트를 다시 돌려봐야 한다.
