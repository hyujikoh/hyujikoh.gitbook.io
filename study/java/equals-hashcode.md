# equals , hashCode 란?

> 클래스를 선언할 때 extends 키워드로 다른 클래스를 상속하지 않으면 암시적으로 java.lang.Object 클래스를 상속하게 된다. 자바의 모든 클래스는 Object 의 자식이거나 자손 클래스이다

따라서 모든 객체는 Object가 가진 메소드를 사용할 수 있다. 아래는 Object가 가진 주요 메소드이다.

| method                     | desc               |
| -------------------------- | ------------------ |
| boolean equals(Object obj) | 객체 번지를 비교하고 결과를 리턴 |
| int hashCode()             | 객체의 해시코드를 리턴       |
| String toString()          | 객체의 문자 정보를 리턴      |



### 객체 동등 비교

Object의 equals() 메소드는 객체의 번지를 비교하고 boolean 값을 리턴한다.

객체의 내부의 내용이 같다고 하더라고 각기 다른 객체이기 때문에 다른 번지수를 갖는다.&#x20;

두 객체가 동일한 객체라면 true를 리턴, 그렇지 않을경우 false 를 리턴한다.

하지만 아래 코드 처럼 번지수를 비교하는것이면, `==` 연산자를 사용해도 동일한 결과가 나온다.



```
@Test
public void test1(){
    Object obj1 = new Object();
    Object obj2 = obj1;
    Assertions.assertThat(obj1.equals(obj2)).isEqualTo(obj1==obj2);
}
```

했을때 결과는 다음과 같이 나온다.&#x20;

<div align="left">

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

</div>

하지만 equals() 메소드는 객체가 달라도 내부의 데이터가 같은지 비교하는 동등비교로 사용이 된다.&#x20;

String 같은 경우엔 equals 메소드를 재정의 해서 내부 문자열이 같은지 비교를 한다.&#x20;

```
//Object의 equals 메소드
public boolean equals(Object obj) {
        return (this == obj);
}


// String 의 equals 메소드 재정의 =
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        return (anObject instanceof String aString)
                && (!COMPACT_STRINGS || this.coder == aString.coder)
                && StringLatin1.equals(value, aString.value);
}
```
