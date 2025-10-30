---
description: for 문과 Stream 의 각기 장단점에 대해 이해하기 위해 작성한 글
---

# for문과 Stream의 성능

## 들어가기 전에...

### 3줄 요약

1. 무조건 어느 한쪽이 더 성능이 좋다고 말한순 없다.
2. 순회 비용과 계산비용을 고려하여 로직을 구현해야한다.
3. 어느걸 써도 코드가 직관적이여야하고, 성능에 영향을 끼쳐서는 안된다.



### 글을 작성하게된 계기

> 처음 개발을 시작할때는 단순히 for 문을 통해 기능을 구현하였다. 공부를 하면서 Java 8 에서 나온 stream api 를 알게 되었지만, 뭔가 어렵게 느껴졌을뿐만 아니라 for 문을 사용하는게 편해서 멀리하게 되었다. 이후 로직을 구현하는데 있어 for 문으로 개발하는데 한계가 있었을 뿐만 아니라 개발 이후 리팩토링을 하는데 있어 번잡하게 작성된 코드로 인해 가독성이 떨어지는 문제를 겪었다. 물론 단순히 for 문을 사용했다는 이유 때문에 이런 문제가 발생되지는 않았지만, 원인이 되었던적이 빈번하였다. 이때부터 코드의 가독성과 간결함을 유지하기위해 Stream API 를 사용하였다. 사용하다 보니 익숙해졌을 뿐만 아니라 맘대로 Stream API 가 좋다고 인식이 박히게 되었다. 이번 글을 통해 두 기능에 대해 이해할 수 있으면 좋겠다.

***

## 본문

### 고정 관념을 부수기 위한 성능테스트

테스트 하기전 나는 Stream API(이하 스트림)가 for 문보다 엄청나게 성능이 좋고 코드를 간결하게 만들어 주는 기능이라고 생각을 하였다.&#x20;

실제로 맞는 내용인지 간단한 테스트 코드를 통해 성능 측정을 하였다.

#### 첫번째 테스트 코드

```java
public class apiTest {
    List<Long> data =  LongStream.rangeClosed(1, 10000)
            .boxed()
            .collect(Collectors.toList());

    @Test
    public void forTest(){
        long beforeTime = System.nanoTime(); //코드 실행 전에 시간 받아오기
        long sum = 0;
        for (long i:
             data) {
            sum+=i;
        }
        long afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
        long secDiffTime = afterTime - beforeTime; //두 시간에 차 계산
        System.out.println("테스트 A 시간차이(m) : "+secDiffTime);
    }

    @Test
    public void forStream(){
        long beforeTime = System.nanoTime(); //코드 실행 전에 시간 받아오기
        long sum = data.stream().mapToLong(
                s -> s
        ).sum();
        long afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기

        long secDiffTime = afterTime - beforeTime; //두 시간에 차 계산
        System.out.println("테스트 B 시간차이(m) : "+secDiffTime);
    }
}

```

list 의 길이를 `10000` 으로 했을때 다음과 같은 결과가 나온다.&#x20;

<div align="left"><figure><img src="../../../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption><p>list size가 <code>10000</code> 일때 각각 테스트 결과</p></figcaption></figure></div>

list 의 길이를 `10000000` 으로 했을때는 성능 결과는 다음과 같다.

<div align="left"><figure><img src="../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption><p>list size가 <code>10000000</code> 일때 각각 테스트 결과</p></figcaption></figure></div>

결과만 봤을때는 for 문이 스트림보다 성능이 더 좋다. 그렇다면 왜 스트림을 사용하는 걸까?



### 스트림이란?

#### 반복문의 처리

Java 8 이전에는 컬렉션 및 배열에 저장된 요소를 반복 처리하기 위해서는 for 문을 이용하거나 Iterator 를 이용을 해야했다.

```java
//예시 코드
class A {
    public void methodA(){
        List<Integer> list = new ArrayList<>();
     
        list.add(1);
        list.add(2);
        list.add(3);
        
        for(int i = 0 ; i< list.size(); i++){
           System.out.println(list.get(0));
        }
     
    }
}
```

Java 8 부터는 또 다른 방법으로 컬렉션 및 배열의 요소를 반복 처리하기 위해 스트림을 사용할 수 있다.

스트림은 for 문 과 Iterator 와 비슷한 반복자이이지만 다음과 같은 차이점을 가진다.

* 내부 반복자이므로 처리 속도가 빠르고 병렬 처리에 효율적이다.
* 람다식으로 다양한 요소 처리를 정의할 수 있다.
* 중간 처리와 최종처리를 수행하도록 파이프 라인을 형성할 수 있다.&#x20;



#### 내부 반복자 와 외부 반복자

for 문과 Iterator는 컬렉션의 요소를 컬렉션 바깥쪽으로 가져와 처리하는데, 이런 방식을 외부 반복자라고 한다. 반면 스트림은 요소 처리 방법을 컬렉션 내부로 주입시켜서 요소를 반복 처리하는데, 이것을 내부 반복라고 한다.&#x20;

외부 반복자일 경우는 컬렉션의 요소를 외부로 가져오는 코드와 처리하는 코드를 모두 개발자 코드가 가지고 있어야 한다. 반대로 내부 반복자일 경우는 개발자 코드에서 제공한 데이터 처리 코드를 가지고 컬렉션 내부에서 요소를 반복 처리한다.&#x20;

내부 반복자는 멀티코어 CPU를 최대한 활용하기 위해 요소들을 분배시켜 병렬 작업을 수행하여 하나씩 처리하는 순차적 외부 반복자보다 효율적으로 요소를 반복시킬 수 있는 장점이 있다.

#### 중간 처리와 최종처리

스트림은 하나 이상 연결될수 있다. 즉 본래 스트림 이후에 중간 스트림이 연결될 수 있고, 그 이후에 매핑 중간 스트림이 연결될 수 있다. 이처럼 스트림이 연결되어 있는 것을 스트림 파이프라인이라 한다.

파이프 라인을 도식화 하면 다음과 같이 설명될 수 있다.



<img src="../../../../.gitbook/assets/file.excalidraw (1).svg" alt="중간처리와 최종처리 파이프라인" class="gitbook-drawing">

해당 파이프라인을 코드로 표현한게 위에 작성한 테스트 코드이다. 오로지 스트림 내장 메소드를 이용해 람다식으로 보다 간편하게 작성된 코드이다. 중간 처리와 상관없이, 가장 중요한것은 최종 처리를 해야한다는 것이다.

### 그러면 왜 앞선 테스트에서는 for 문이 성능이 더 좋게 나올까?

앞서 작성된 테스트 코드에서 는 원시 타입을 기준으로 테스트를 작성하였는데, 반대로 참조 타입을 통해서 테스트 성능을 측정해보겠다.

#### 두번째 테스트

```java
@Test
public void forTest_ver2(){
    Long beforeTime = System.nanoTime(); //코드 실행 전에 시간 받아오기
    Long sum = 0L;
    for (Long i:
            data) {
        sum+=i;
    }
    Long afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
    Long secDiffTime = afterTime - beforeTime; //두 시간에 차 계산
    System.out.println("테스트 A 시간차이(m) : "+secDiffTime);
}

@Test
public void forStream_ver2(){
    Long beforeTime = System.nanoTime(); //코드 실행 전에 시간 받아오기
    Long sum = data.stream().mapToLong(
            s -> s
    ).sum();
    Long afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기

    Long secDiffTime = afterTime - beforeTime; //두 시간에 차 계산
    System.out.println("테스트 B 시간차이(m) : "+secDiffTime);
}
```

테스트 결과는 다음과 같이 나온다!\
![](<../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png>)

이전 테스트 에서 분명 성능이 더 좋았던 for-loop 가 변수 타입을 참조 타입으로 변경했을뿐인데, 이전 결과가 역전이 되어버린것이었다.

다음은 JVM 의 대략적은 구조를 설명하는 자료이다.&#x20;

<figure><img src="../../../../.gitbook/assets/image (13).png" alt=""><figcaption><p>출처 : <a href="https://www.devkuma.com/docs/jvm/memory-structure/">https://www.devkuma.com/docs/jvm/memory-structure/</a></p></figcaption></figure>

[첫번째 테스트 코드](for-stream.md#undefined-2)에서 int 형 자료형은 원시타입이기 때문에 JVM 에서 스택영역으로 저장이 되어서 메모리에 직접 참조를 통해 값이 저장이 되어진다.&#x20;

반대로 [두번째 테스트 코드](for-stream.md#undefined-7)에서는 자료형이 참조타입으로 저장되어 Heap 영역에 저장이 되어진다. 이때 값을 가져올때 간접 참조, 간접적으로 주소를 이용해 값을 가져와야한다.&#x20;

값을 직접적으로 참조하는 것보다. 주소 참조를 통해 값을 가져오는 비용이 더 들기 때문에, 앞선 테스트에서 보여준 for 문 성능이점이 없어지게 된것이다. (이를 순회비용이 간접 참조일때 더 많이 발생한다는 것으로 표현된다.)

### 순회 비용과 계산비용

앞선 테스트에서 순회 비용이 계산비용과 비교해서 상대적으로 비용이 높았다는 결론이 나왔다. 그렇다면 만일 순회 비용에 비해서  압도적으로 계산 비용이 높았을 경우엔 어떤 결과가 나올까?

참고한 자료와 마찬가지로 apache 의 sin 함수를 통해 성능계산을 해보기로 했다.

#### 세번째 테스트

```java
@Test
public void cal_cost_test(){
    Long beforeTime = System.nanoTime(); //코드 실행 전에 시간 받아오기

    // 배열을 이용한 for-loop 테스트
    for (int i = 0; i < ints.length; i++) {
        double d = FastMath.sin(ints[i]);
        if (d > m) m = d;
    }

    Long afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
    Long secDiffTime = afterTime - beforeTime; //두 시간에 차 계산
    System.out.println("배열을 이용한 for-loop 테스트 시간 "+secDiffTime);


    // 배열을 이용한 sequential stream 테스트
    beforeTime =  System.nanoTime();
    double maxFromArrayStream = Arrays.stream(ints)
            .mapToDouble(FastMath::sin)
            .reduce(Double.MIN_VALUE, Math::max);

    afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
    secDiffTime = afterTime - beforeTime; //두 시간에 차 계산

    System.out.println("배열을 이용한 sequential stream 테스트 시간  : "+secDiffTime);

    // arrayList 테스트를 위한 사전 초기화
    m = Double.MIN_VALUE;

    // ArrayList를 이용한 for-loop 테스트
    beforeTime =  System.nanoTime();

    for (int i = 0; i < arrayList.size(); i++) {
        double d = FastMath.sin(arrayList.get(i));
        if (d > m) m = d;
    }
    afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
    secDiffTime = afterTime - beforeTime; //두 시간에 차 계산

    System.out.println("ArrayList를 이용한 for-loop 테스트 시간  : "+secDiffTime);



    // ArrayList를 이용한 sequential stream 테스트
    beforeTime =  System.nanoTime();
    double maxFromArrayListStream = arrayList.stream()
            .mapToDouble(FastMath::sin)
            .reduce(Double.MIN_VALUE, Math::max);
    afterTime = System.nanoTime(); // 코드 실행 후에 시간 받아오기
    secDiffTime = afterTime - beforeTime; //두 시간에 차 계산

    System.out.println("ArrayList를 이용한 sequential stream 테스트 시간  : "+secDiffTime);
}
```

이를 측정한 성능 결과는 다음과 같다.

<div align="left"><figure><img src="../../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure></div>

다음과 같이 계산비용이 높은 테스트에서 for 문이 더이상 성능에 우수함을 나타내지 않는 결과를 확인 할 수 있게 됐다. 즉 내부의 함수 계산 비용과 순회비용을 합친 경우를 고려하여도 stream 으로 구현하여도 성능 저하가 나타나지 않는다는 것이다.

## 마무리 및 느낀점

> 지금껏 for 문과 stream 에 대해서 막 사용을 하였고, 어느 부분에서 성능에 이점이 나타나는지 모르는 상태에서 개발에 임했던것 같다. 이 글을 작성하는 기간동안 여러 포스팅을 보고, 관련 서적을 보면서 내가 사용하는 언어에 대해서 좀더 한발짝 알아가는 계기가 되었던것 같고, 어느 상황에 보다 최적의 성능을 낼수있도록 의사결정을 내릴수 있는 개발자로 성장했으면 싶다.

## 참고 자료

{% embed url="https://www.devkuma.com/docs/jvm/memory-structure/" fullWidth="false" %}

{% embed url="https://sigridjin.medium.com/java-stream-api%EB%8A%94-%EC%99%9C-for-loop%EB%B3%B4%EB%8B%A4-%EB%8A%90%EB%A6%B4%EA%B9%8C-50dec4b9974b" %}

{% embed url="https://jaxlondon.com/wp-content/uploads/2015/10/The-Performance-Model-of-Streams-in-Java-8-Angelika-Langer-1.pdf" %}

## 🦔
