---
description: >-
  (2023-10-26) org.springframework.transaction.UnexpectedRollbackException:
  Transaction silently rolled back because it has been marked as rollback-only
---

# UnexpectedRollbackException

문제&#x20;

JPA 를 통해 데이터를 생성 API 를 진행하는 도중에 아래와 같은 에러가 발생하였다.&#x20;

<figure><img src="../../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

지금 까지 CRUD 작업을 하면서 rollback 관련한 이슈가 나온건 처음이었다.&#x20;

상황은 다음과 같다.

1. 서비스 로직을 처리하는 `Bean A` 가 데이터 저장을 위해 `Bean B`  을 실행을 하면 위와 같은 에러가 발생&#x20;
2. 두개의 Bean 클래스에는 동일하게 `@Transctional` 이 걸려있다.



원인&#x20;

Bean B 클래스에 `@Transctional` 을 적용했는데, 메소드에 또다시 `@Transctional` 을 적용함으로서 중첩 트랜잭션이 걸려서 생긴 이슈 였다.
