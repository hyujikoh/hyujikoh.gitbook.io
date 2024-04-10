---
description: Map 인터페이스 상속 받는 클래스 구현체들을 공부 및 정리
---

# Map의 성능 차이

## 서론

### 글을 들어가기 전

이 글을 통해, Java의 Map 인터페이스와 그 하위 구현체들에 대한 깊이 있는 이해를 도모하고자 한다. 여러 Map 구현체들의 특성을 비교하며, 어떤 상황에서 각각을 최적으로 활용할 수 있는지 탐색할 것이다. 또한, 실제 코드 예제와 함께 각 구현체의 성능을 직접 테스트하고 분석함으로써, 이론과 실제의 연결고리를 마련하려 한다.



## 본론

### Map 이란?

Map Interface는 Key 와 Value 를 연결하여 객체를 Mapping 하는 구조로, 중복된 키를 허용하지 않으며 하나의 값에만 매핑한다.

&#x20;보통 많이 사용하는 HashMap 과 Hashtable를 비교 및 정리 하면 다음과 같다.

| Feature         | HashMap                                              | Hashtable                                                                        |
| --------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------- |
| Thread-satety   | X                                                    | O                                                                                |
| Synchronization | X                                                    | O                                                                                |
| K/V null 허용     | O                                                    | X                                                                                |
| 성능              | 단일 스레드 환경에서는 HashMap 이 성능 우수                         | 동기화를 진행하기 때문에 성능저하가 발생                                                           |
| iteration       | fail-fast; 즉, 반복중에 HashMap 이 구조적으로 수정되면 exception 발생 | <p>Enumerator와 Iteration 둘다 사용 가능함. </p><p>Enumerator 는 Fail-safe 이지만, 구식이다.</p> |
| Inheritance     | AbstractMap class 상속                                 | Dictionary class 상속                                                              |
| Legacy          | 1.2                                                  | 1.0                                                                              |

간단하게 정리를 하자면, 단일 스레드에서는 `HashMap` 을 사용하고, 멀티스레드에서는 `HashTable` 이 적합하다.&#x20;

그러면 단일 스레드에서 `HashMap` `HashTable`  의 성능이 어떤지 테스트 코드에서 한번 확인을 해봤다.

```java
@Test
@Name("싱글 스레드 테스트")
public void mapTest2() throws InterruptedException {
    final int operations = 10_000_000; // 1천만 개의 작업 수로 증가
    testHashMap(operations);
    testHashtable(operations);
}

private static void testHashMap(int operations) {
    Map<String, Integer> hashMap = new HashMap<>();
    long startTime = System.currentTimeMillis();
    for (int i = 0; i < operations; i++) {
        hashMap.put("Key" + i, i);
    }
    long endTime = System.currentTimeMillis();
    System.out.println("HashMap (Single Thread) - Time: " + (endTime - startTime) + " ms");
}

private static void testHashtable(int operations) {
    Map<String, Integer> hashtable = new Hashtable<>();
    long startTime = System.currentTimeMillis();
    for (int i = 0; i < operations; i++) {
        hashtable.put("Key" + i, i);
    }
    long endTime = System.currentTimeMillis();
    System.out.println("Hashtable (Single Thread) - Time: " + (endTime - startTime) + " ms");
}
```

테스트 결과는 다음과 같다.

<figure><img src="../../../.gitbook/assets/image (21).png" alt=""><figcaption><p>테스트 결과</p></figcaption></figure>

## 참고 레퍼런스 및 사이트

* JAVA Map Docs from \
  [https://docs.oracle.com/javase/8/docs/api/index.html](https://docs.oracle.com/javase/8/docs/api/index.html)&#x20;
