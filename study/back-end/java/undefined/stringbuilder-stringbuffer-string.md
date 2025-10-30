---
description: java에서 문자열을 처리하기 위해 사용되는 3개의 특징과 차이점에 대해 공부
---

# StringBuilder, StringBuffer, String 특성

## 서론

### 글을 들어가기 전

문자열을 처리할때 String 을 사용하고 처리를 했지만, 이전에 String으로 문자열을 처리하는것보다 StringBuilder 나 StringBuffer 를 통해 처리하면 좋다는 코드리뷰를 받아본적이 있다. 이후에는 당연하게 문자열 처리를 다음 2개로 진행했지만, 이번글을 통해 공부를 하고 싶어서 작성하였다.



## 본론

### 불변성과 가변성

우선 가장 큰 차이점은 String 클래스는 불변(immutable) 이라는 점이다. 즉 String 으로 생성된 객체의 내용은 변경이 될수가 없다. 문자열에서 어떤 변경을 수행하려고 하면, 사실상 새로운 String 객체가 생성되어 메모리가 할당이 되는것이다. 이는 메모리 사용 측면에서 비효율적일 수 있고, 특히 빈번한 문자열 수정이 가해진다고 하면 GC 가 사용하지 않는 객체를 삭제하는등 성능에 영향을 줄만한 작업을 수행하게 된다.

반면 StringBuilder 와 StringBuffer 는 가변(mutable) 이다. 이들을 사용하면 문자열을 직접 변경할 수 있으므로 빈번한 문자열 변경이 일어나는 상황에서 String 보다 효율적일수 있다. String 처럼 새로운 객체를 생성하는 대신 기존의 데이터를 직접 수정할 수 있기 때문이다.



### 스레드 안전성

StringBuffer 는 thread-safe 기반으로 설계가 되어있다. 이는 StringBuffer 의 메서드들이 여러 스레드 에서 동시 호출되더라도 안전하게 실행될수 있음을 의미한다. 따라서 멀티스레드 기반으로 안전하게 문자열 데이터를 처리 할 때 StringBuffer 를 사용하는 것이 좋다.

반면 StringBuilder 는 thread-safe 와 동기화를 지원하지 않는다. 이는 단일 스레드 환경에서 좋은 성능이 나타난다는 이야기다.



### 성능

앞서 언급한 내용과 같이, StringBuiler 는 동기화를 고려하지 않기 때문에 StringBuffer보다 빠른 성능을 제공한다. 반대로 문자열을 읽기 작업이 많고 문자열 수행 작업이 적을 경우엔 String 을 쓰는것이 적합할수있다.

각각의 클래스를 비교 및 정리 하자면 다음과 같다.

<details>

<summary>String</summary>

* `String` 클래스는 불변(immutable)이다. 즉, 한 번 생성된 `String` 객체의 내용을 변경할 수 없음. 문자열을 수정할 때마다 새로운 `String` 객체가 생성되며, 이전 객체는 가비지 컬렉션 대상이 된다.
* 문자열 연산이 적고, 읽기 작업이 많을 때 적합.
* 멀티쓰레드 환경에서 안전(thread-safe)합니다.

</details>

<details>

<summary>StringBuffer</summary>

* `StringBuffer`는 가변(mutable)하며, 문자열을 추가, 수정, 삭제할 때 기존 객체를 재사용합니다. 따라서 `String`에 비해 메모리 사용이 효율적이고 빠름.
* `StringBuffer`는 동기화(synchronized)되어 있어 멀티쓰레드 환경에서 안전(thread-safe)하지만, 단일 쓰레드 환경에서는 `StringBuilder`에 비해 성능이 떨어질 수 있다.
* 문자열 연산(추가, 수정, 삭제 등)이 자주 발생하고, 멀티쓰레드 환경에서 사용할 때 적합.

</details>

<details>

<summary>StringBuilder</summary>

* `StringBuilder`도 가변(mutable)하며, `StringBuffer`와 유사하게 작동하지만, `StringBuilder`는 동기화되어 있지 않는다.(non-synchronized).
* 이로 인해 `StringBuilder`는 `StringBuffer`보다 더 빠르며, 단일 쓰레드 환경에서 문자열 연산이 자주 발생할 때 적합.
* 멀티쓰레드 환경에서는 안전하지 않을 수 있으므로 주의가 필요.

</details>
