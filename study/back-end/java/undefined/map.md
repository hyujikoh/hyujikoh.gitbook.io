---
description: Map 인터페이스 상속 받는 클래스 구현체들을 공부 및 정리
---

# Map의 특성 및 성능

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

### HashMap VS HashTable

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

<figure><img src="../../../../.gitbook/assets/image (21).png" alt=""><figcaption><p>테스트 결과</p></figcaption></figure>

그렇다면 멀티 스레드 환경에서 둘의 성능을 한번 비교해보기 위해 테스트 코드를 작성해봤다.&#x20;

테스트 방식은 이전에 진행한 싱글 스레드 형태와 비슷하게 진행하려 한다.

GPT 를 이용해 작성한 테스트 코드는 다음과 같다.

```java
@Test
@DisplayName("멀티 스레드 테스트")
public void mapTestMultiThread() throws InterruptedException {
    final int operations = 10_000_000; // 1천만 개의 작업 수
    final int threadCount = 10; // 사용할 스레드의 수

    testHashMapMultiThread(operations, threadCount);
    testHashtableMultiThread(operations, threadCount);
}

private void testHashMapMultiThread(int operations, int threadCount) throws InterruptedException {
    Map<String, Integer> hashMap = new HashMap<>();
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                hashMap.put("Key" + i + Thread.currentThread().getId(), i);
            }
        });
    }

    executorService.shutdown();
    executorService.awaitTermination(1, TimeUnit.HOURS);

    long endTime = System.currentTimeMillis();
    System.out.println("HashMap (Multi Thread) - Time: " + (endTime - startTime) + " ms");
}

private void testHashtableMultiThread(int operations, int threadCount) throws InterruptedException {
    Map<String, Integer> hashtable = new Hashtable<>();
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                hashtable.put("Key" + i + Thread.currentThread().getId(), i);
            }
        });
    }

    executorService.shutdown();
    executorService.awaitTermination(1, TimeUnit.HOURS);

    long endTime = System.currentTimeMillis();
    System.out.println("Hashtable (Multi Thread) - Time: " + (endTime - startTime) + " ms");
}
```

1천만개의 작업을 하는데 조금 지연이 될거라 생각했지만, 이렇게 까지 지연이 될거라곤 예측을 못했다.

아예 해시맵 1차 작업이 끝나는게 확인 자체가 안된다.

그래서 작업이 충돌이 일어났나 해서 여러번 다시 테스트 했을때 다음과 같이 결과가 나왔다.

<figure><img src="../../../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

그래서 다시 한번 돌려봤다.

<figure><img src="../../../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

이번에는 HashMap 이 더 빠르다고 나오는 상황까지 발생했다.

아예 작업수를 1억을 할려니 메모리 초과를 오류가 발생되었다...

<figure><img src="../../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

### 그렇다면 HashMap 만 쓰면 되는것으로 마무리?

다시한번 HashMap 과 HashTable 의 장단점을 정리해볼 필요가 있다.

Oracle 에서 제공하는 java 8 Docs 에서 HashMap의 내용을 보면 볼드체로 강조하는 문구가 있다.

> **Note that this implementation is not synchronized.**

아무리 성능에서 큰 차이가 없다 해도 동기화가 안됨으로 정합성이 보장이 안된다고 하면 문제가 된다.\
보다 더 디테일한 테스트를 한번 작성해보기로 했다.

이전과 테스트는 동일하지만, 성능 결과와 각각의 map 과 table 에서 제대로 값이 들어 갔는지 테스트를 해보기로 했다.

```java
@Test
@DisplayName("멀티 스레드 상세 테스트")
public void mapTestMultiThreadDetail() throws InterruptedException {
    final int operations = 1000; // 작업 수를 1000개로 설정
    final int threadCount = 10; // 사용할 스레드의 수



    testHashMapMultiThreadDetail(operations, threadCount);
    testHashtableMultiThreadDetail(operations, threadCount);

}

private void testHashMapMultiThreadDetail(int operations, int threadCount) throws InterruptedException {
    Map<String, Integer> hashMap = new HashMap<>();
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        int finalT = t;
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                hashMap.put("Key" + finalT + "-" + i, i);
            }
        });
    }

    executorService.shutdown();
    boolean finished = executorService.awaitTermination(1, TimeUnit.MINUTES);
    assertTrue(finished);

    long endTime = System.currentTimeMillis();

    // 검증 과정
    assertFalse(hashMap.size() == operations);

    System.out.println("HashMap (Multi Thread) - Time: " + (endTime - startTime) + " ms, Size: " + hashMap.size());
}

private void testHashtableMultiThreadDetail(int operations, int threadCount) throws InterruptedException {
    Map<String, Integer> hashtable = new Hashtable<>();
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        int finalT = t;
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                hashtable.put("Key" + finalT + "-" + i, i);
            }
        });
    }

    executorService.shutdown();
    boolean finished = executorService.awaitTermination(1, TimeUnit.MINUTES);
    assertTrue(finished);

    long endTime = System.currentTimeMillis();

    // 검증 과정
    assertTrue(hashtable.size() == operations);

    System.out.println("Hashtable (Multi Thread) - Time: " + (endTime - startTime) + " ms, Size: " + hashtable.size());
}
```

만약 문서에서 나온대로 HashMap 이 동기화를 제공을 안하면 각 스레드에서 진행한 작업이 제대로 수행되지 않아 작업수 만큼 map 의 사이즈가 나오지 않을것이고, HashTable 에서 동기화를 기본적으로 제공해주면 size 는 작업수 만큼 나올것이다.&#x20;

결과는 다음과 같이 나왔다.

<figure><img src="../../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

기대했던 대로 테스트가 제대로 통과가 되었다!

### ConcurrentHashMap , Collections.synchronizedMap

&#x20;이런 HashMap의 동시성을 추가한게 ConcurrentHashMap 과 Collections.synchronizedMap 이다.&#x20;

굳이 synchronizedMap 앞에다 Collections 를 붙인 이유는 ConcurrentHashMap 과 다른 라이브러리를 상속 받기 때문이다. ConcurrentHashMap 은 `util.concurrent` 을 상속 받고,   synchronizedMap 은 `Collections` 를 상속 받는다.

결과적으로 둘다 thread-safe 를 지원해주지만 두 클래스의 차이점 중 두드러진 차이라고 하면 null support 여부 일것이다. ConcurrentHashMap 은 null을 허용하지 않지만, Collections.synchronizedMap 은 null을 허용해준다. null 를 허용해줌으로 생기는 경우의 수를 고려하면 되도록이면 ConcurrentHashMap 이 좋은 선택일수 있다.

#### 성능 비교

테스트 는 이전에 했던걸 이용하기로 했다. \


```java
@Test
@DisplayName("Concurrent , Sync HashMap 성능 비교")
public void differenceMapTestMultiThread() throws InterruptedException {
    final int operations = 1_000_000; // 작업 수를 1000개로 설정
    final int threadCount = 100; // 사용할 스레드의 수

    testConcurrentHashMapMultiThread(operations, threadCount);
    testSyncHashMapMultiThread(operations, threadCount);

}

private void testSyncHashMapMultiThread(int operations, int threadCount) throws InterruptedException {
    Map<String, Integer> hashMap = Collections.synchronizedMap(new HashMap<>());
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        int finalT = t;
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                hashMap.put("Key" + finalT + "-" + i, i);
            }
        });
    }

    executorService.shutdown();
    boolean finished = executorService.awaitTermination(1, TimeUnit.MINUTES);
    assertTrue(finished);

    long endTime = System.currentTimeMillis();

    // 검증 과정
    assertTrue(hashMap.size() == operations, "Hashtable에 모든 요소가 정상적으로 삽입되지 않았습니다.");

    System.out.println("synchronizedMap (Multi Thread) - Time: " + (endTime - startTime) + " ms, Size: " + hashMap.size());
}

private void testConcurrentHashMapMultiThread(int operations, int threadCount) throws InterruptedException {
    ConcurrentHashMap <String, Integer>concurrentHashMap= new ConcurrentHashMap<>();
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    long startTime = System.currentTimeMillis();
    for (int t = 0; t < threadCount; t++) {
        int finalT = t;
        executorService.submit(() -> {
            for (int i = 0; i < operations / threadCount; i++) {
                concurrentHashMap.put("Key" + finalT + "-" + i, i);
            }
        });
    }

    executorService.shutdown();
    boolean finished = executorService.awaitTermination(1, TimeUnit.MINUTES);
    assertTrue(finished);

    long endTime = System.currentTimeMillis();

    // 검증 과정
    assertTrue(concurrentHashMap.size() == operations, "concurrentHashMap 에 모든 요소가 정상적으로 삽입되지 않았습니다.");

    System.out.println("concurrentHashMap (Multi Thread) - Time: " + (endTime - startTime) + " ms, Size: " + concurrentHashMap.size());
}
```

<figure><img src="../../../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

결과가 이렇게 나오긴 한데, 사실 성능에 대해서 여러 레퍼런스를 찾아보기도 했지만, ConcurrentHashMap 이 좋다고 나온 것도 있어서, 지금 진행한 테스트가 환경(HW)에서는 저런 결과가 나오긴 한다. 보다 더 확실한 결과를 보기 위해서 연산을 1억 단위로 진행할려고 하니 아예 메모리 에러가 나오기도 한다.&#x20;

확실한건 thread-safe 와 같은 동시성 이슈를 잡기위한 여러 방법이 있다는것을 확인했다.

## 참고 레퍼런스 및 사이트

* JAVA Map Docs from \
  [https://docs.oracle.com/javase/8/docs/api/index.html](https://docs.oracle.com/javase/8/docs/api/index.html)&#x20;
* Java HashMap Docs from \
  [https://docs.oracle.com/javase/8/docs/api/index.html](https://docs.oracle.com/javase/8/docs/api/index.html)
* Collections.synchronizedMap vs. ConcurrentHashMap  from\
  [https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap](https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap)
