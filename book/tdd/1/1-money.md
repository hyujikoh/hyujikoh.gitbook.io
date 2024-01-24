# 1장 다중 통화를 지원하는 Money 객체

다음과 같은 조건이 추가 되었다고 가정해보자

{% hint style="info" %}
기존의 달러만 다루던 채권을 다른 화폐로 채권을 다룰 상황에 대비해 프로그램을 수정하라
{% endhint %}

즉 현 보고서를&#x20;

<table data-full-width="true"><thead><tr><th>종목</th><th>주</th><th>가격</th><th>합계</th></tr></thead><tbody><tr><td>Company A </td><td>1000</td><td>30</td><td>30000</td></tr><tr><td>Company B</td><td>400</td><td>100</td><td>40000</td></tr><tr><td></td><td></td><td>합계</td><td>70000</td></tr></tbody></table>

다중통화를 지원하는 보고서로 만들어야한다.&#x20;

<table data-full-width="true"><thead><tr><th>종목</th><th>주</th><th>가격</th><th>합계</th></tr></thead><tbody><tr><td>Company A </td><td>1000</td><td>30USD</td><td>30000</td></tr><tr><td>Company B</td><td>400</td><td>100<a data-footnote-ref href="#user-content-fn-1">CHF</a></td><td>40000</td></tr><tr><td></td><td></td><td>합계</td><td>70000</td></tr></tbody></table>



이를 위해서는 환율 정보도 명시해야한다.

| 기준  | 변환  | 환율  |
| --- | --- | --- |
| CHF | USD | 1.5 |



이와 같은 상황이 발생했을때 나는 아마 이렇게 생각했을거다.

> 기존의 환율을 달러로만 사용했던 방식을 각기 다른 환율을 위해서 각각의 정보를 DB 테이블로 관리를 하는게 좋을거고... 또 테이블을 만들기 위해서 ERD를 작성해야하고...  또 환율 데이터를 Access 하는`repository` 클래스를 만들어야겠다... 아 이왕 만드는거 다형성과 상속성을 이용해서.. 객체지향적으로 짜면 좋을거 같다! 그럴러면 인터페이스랑 실제 구현체 클래스로 나누어서 만들어야겠다. 또 환율 대비 연산 수행처리하는 메소드를분리해서 만들어야겠다.

이러다 잘린다.

우선은 책에서 진행하는 방법대로 할일목록을 만들고, 작업이 끝나면 ~~작업1~~ 으로 처리하기로 했다.

테스트를 작성할때는 오퍼레이션의 완벽한 인터페이스에 대해 상상해보는 것이 좋다고 한다. 그러면 오퍼레이션은 뭘 의미하는걸까? 책에서는 보통 Method 와 비슷한 의미로 씌이고 객체가 수행할 수 있는 연산을 의미한다. 다형성을 지원하는 언어에서는 한 오퍼레이션은 여러 메소드를 가질수있다고 한다고 한다.



그러니까 Java 로 풀이를 하면 이렇게 표현할 수 있을거 같다.&#x20;

```
public interface Operation {
    void operationA();
}


public class OperationA implements Operation{
    @Override
    public void operationA() {

    }
}


public class OperationB implements Operation{
    @Override
    public void operationA() {

    }
}

```

위 와 같은 코드로 설명하면 `Operation` 이라는 인터페이스는 여러 메소드를 가지고 있다.



다시 화폐로 넘어가서 곱셈에 대해 테스트  코드를 적어보자

```
@Test
public void testMul(){
        Dollar five = new Dollar(5);
        five.times(2);
        Assertions.assertThat(10,five.amount);
 }
```

이 테스트에 대해서 아래와 같은 할일 목록을 하나하나 클리어 해보겠다.

* [ ] $5 + 10CHF = 10$(환율이 2:1  일 경우)
* [ ] **$5\*2 = $10**
* [ ] amount를 private로 만들기(지금처럼 필드에 바로 접근하게 하면 지옥간다)
* [ ] Dollar 부작용
* [ ] Money 반올림(int 연산자로 세세한 환율을 계산하면 지옥간다. )

우선 이 테스트에 대해서 없는것들이 너무 많다. 우선 Dollar 클래스도 없는데 Dollar 를 선언 하다니... 바로 클래스 부터 만들어 보자

```
public class Dollar {
}
```

클래스 생성 에러가 하나 없어졌으니 다음은 클래스에 대한 int 매개변수를 받는 생성자를 만들어보자

```
public class Dollar {
    Dollar(int amount){
        
    }
}
```

생성자 에러를 제거 하였으니 다음은 메서드에 대해서 최소한 통과가 되게 만들어보자.

```
public class Dollar {
    Dollar(int amount){

    }
    
    void times (int mul){
        
    }
}
```

메서드도 만들었으니 이제는 amount 필드를 만들자

```
public class Dollar {
    int amount;
    Dollar(int amount){

    }

    void times (int mul){

    }
}

```

우선적으로 테스트가 돌아가게 만든 이 케이스를 실행시키면 아마 실패됐다고 나올것이다. 이제는 이 테스트를 빠르게 통과하는 코드를 만들어보자.

```
public class Dollar {
    int amount=10;
    Dollar(int amount){

    }

    void times (int mul){

    }
}
```

<figure><img src="../../../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>멋지게 통과했다.</p></figcaption></figure>

이렇게 만든 테스트는 야매로 통과 되었으니까 테스트 코드를 정석대로 중복을 제거해보자. 초기에  필드`amount` 를 10 으로  선언한것은 `5*2` 가 빠르게 머리속에 스쳐지나갔으니까 사용한 메서드 `times` 를 이용해보자.&#x20;

<pre><code><strong>public class Dollar {
</strong>    int amount;
    Dollar(int amount){

    }

    void times (int mul){
        amount = 5*mul;
    }
}
</code></pre>

다시 코드를 리팩토링을 해보자!

```
public class Dollar {
    int amount;
    Dollar(int amount){
        this.amount = amount;
    }

    void times (int mul){
        amount *= mul;
    }
}
```

이렇게 함으로서 테스트에 대해 완료를 수행했고 다음과 같이 할일 목록이 업데이트 되었다.

* [ ] $5 + 10CHF = 10$(환율이 2:1  일 경우)
* [x] ~~**$5\*2 = $10**~~
* [ ] amount를 private로 만들기(지금처럼 필드에 바로 접근하게 하면 지옥간다)
* [ ] Dollar 부작용
* [ ] Money 반올림(int 연산자로 세세한 환율을 계산하면 지옥간다. )

이 다음 장에는 Dollar의 부작용에 대해서 공부한것을 정리 해보겠다.

[^1]: 스위스 프랑
