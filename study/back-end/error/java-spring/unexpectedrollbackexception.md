---
description: >-
  (2023-10-26) org.springframework.transaction.UnexpectedRollbackException:
  Transaction silently rolled back because it has been marked as rollback-only
---

# UnexpectedRollbackException 오류 해결

문제&#x20;

JPA 를 통해 데이터를 생성 API 를 진행하는 도중에 아래와 같은 에러가 발생하였다.&#x20;

<figure><img src="../../../../.gitbook/assets/image (3) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

지금 까지 CRUD 작업을 하면서 rollback 관련한 이슈가 나온건 처음이었다.&#x20;

상황은 다음과 같다.

1. 서비스 로직을 처리하는 `Bean A` 가 데이터 저장을 위해 `Bean B`  을 실행을 하면 위와 같은 에러가 발생&#x20;
2. 두개의 Bean 클래스에는 동일하게 `@Transctional` 이 걸려있다.



원인 및 해결

1. Bean B 클래스에 `@Transctional` 을 적용했는데, 메소드에 또다시 `@Transctional` 을 적용함으로서 중첩 트랜잭션이 걸려서 생긴 이슈 였다. 이후에 `Bean A` 에 적용한 트랜잭션을 해제하고 Bean B 에만 남겨놓고 적용을 하니 정상적으로 처리가 되었다. 좀만 더 이 부분에서 찾아보고 작성하겠다.
2. 이후에 동일한 상황이 닥쳤을때, 여러 트랜잭션이 겹쳐있었다. 해당 특정 트랜잭션 내 실패가 발생하였지만, 해당 트랜잭션에 예외 처리를 안하고 continue 하여 강제로 다른 트랜잭션으로 넘어가게 진행을 하였다. 결과적으로 해당 로직이 정상적으로 수행이 되었지만, 여러 트랜잭션 중 하나가 실패로 끝났기 때문에 rollback 예외가 발생을 하였다. 이를 위해서 했던 방법은 해당 트랜잭션을 제거 하든지, 오류가 발생하지 않기 위해 서비스 코드를 리팩토링 하는 방법이 있었다. (2024-02-15)

