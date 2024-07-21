# equals , hashCode 란?

## 들어가기전에..



3줄 요약

1. equals는 객체의 주소를 비교 하는 메소드이다. 하지만 다르게 사용 가능하다.
2. hashCode는 객체의 메모리 주소를 이용해서 정수값 리턴한다.
3. equals와 hashCode를 재정의 해서 객체의 활용도를 높일수 있다.

> 클래스를 선언할 때 extends 키워드로 다른 클래스를 상속하지 않으면 암시적으로 java.lang.Object 클래스를 상속하게 된다. 자바의 모든 클래스는 Object 의 자식이거나 자손 클래스이다

따라서 모든 객체는 Object가 가진 메소드를 사용할 수 있다. 아래는 Object가 가진 주요 메소드이다.

| method                     | desc               |
| -------------------------- | ------------------ |
| boolean equals(Object obj) | 객체 번지를 비교하고 결과를 리턴 |
| int hashCode()             | 객체의 해시코드를 리턴       |
| String toString()          | 객체의 문자 정보를 리턴      |

## equals 란?

### 객체 동등 비교

Object의 equals() 메소드는 객체의 번지를 비교하고 boolean 값을 리턴한다.

객체의 내부의 내용이 같다고 하더라고 각기 다른 객체이기 때문에 다른 번지수를 갖는다.&#x20;

두 객체가 동일한 객체라면 true를 리턴, 그렇지 않을경우 false 를 리턴한다.

하지만 아래 코드 처럼 번지수를 비교하는것이면, `==` 연산자를 사용해도 동일한 결과가 나온다.



```java
@Test
public void test1(){
    Object obj1 = new Object();
    Object obj2 = obj1;
    Assertions.assertThat(obj1.equals(obj2)).isEqualTo(obj1==obj2);
}
```

했을때 결과는 다음과 같이 나온다.&#x20;

<div align="left">

<figure><img src="../../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

</div>

### 주소값만 비교하는것이 아닌 경우

하지만 equals() 메소드는 객체가 달라도 내부의 데이터가 같은지 비교하는 동등비교로 사용이 된다.&#x20;

String 같은 경우엔 equals 메소드를 재정의 해서 내부 문자열이 같은지 비교를 한다.&#x20;

```java
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



결과적으로 @Override 를 통해서 equals 를 재정의함으로서 원하는 방식을 사용 가능하게 한다.



## hashCode 란?&#x20;

### 메모리번지를 이용한 정수값 리턴

객체 해시코드란 객체를 식별하는 정수를 말한다.

Object 클래스의 hashCode() 메소드는 객체의 메모리 번지를 이용해서 해시코드를 생성하기 때문에 객체마다 다른 정수값을 리턴한다.&#x20;

각각의  객체는  독립적인 요소이기 때문에 객체마다 다른 정수값을 리턴한다.&#x20;

결국엔 hashCode() 메소드의 용도는 equals() 메소드와 동일한 맥락이다. 즉 두 객체가 동등한지 비교할때 주로 사용하다.

```java
public int hashCode()
```



### 코드 예시

이를 잘 활용하는 것이 Collections 의 `HashSet` 이다.\
`HashSet` 은 동등 객체를 저장하지 않는 특성을 가지는데, 이때 hashCode() 랑 equals() 메소드를 이용해서 동등 객체여부를 판단한다.&#x20;

```java
@Test
public void test3(){
    HashSet<Member> hashSet = new HashSet();

    Member member = new Member(1,"min");

    hashSet.add(member);
    Assertions.assertThat(hashSet.size()).isEqualTo(1);

    Member member2 = new Member(1,"min");

    hashSet.add(member2);
    Assertions.assertThat(hashSet.size()).isEqualTo(2);

    Member member3 = new Member(112,"min");

    hashSet.add(member3);
    Assertions.assertThat(hashSet.size()).isEqualTo(3);
}

public class Member{
    int num;
    String name;

    Member(int num, String name){
        this.num = num;
        this.name = name;
    }
}
```

다음과 같이 테스트 케이스를 만들어서 사용한 결과 총 3개의 객체가 들어간것으로 확인이 된다.&#x20;

결국엔 안에 내용물이 똑같더라도, 객체의 주소값이 다르게 배정되기때문에 총 3개가 들어갔다.



## hashCode 와 equals 재정의&#x20;

그러면 이번엔 hashCode() equals() 메소드를 재정의 해서 사용해보자

```java
@Test
public void test4(){
    HashSet<MemberA> hashSet = new HashSet();

    MemberA member = new MemberA(1,"min");

    hashSet.add(member);
    Assertions.assertThat(hashSet.size()).isEqualTo(1);

    MemberA member2 = new MemberA(1,"min");

    hashSet.add(member2);
    Assertions.assertThat(hashSet.size()).isEqualTo(1);

    MemberA member3 = new MemberA(112,"min");

    hashSet.add(member3);
    Assertions.assertThat(hashSet.size()).isEqualTo(2);
}

public class MemberA{
    int num;
    String name;

    MemberA(int num, String name){
        this.num = num;
        this.name = name;

    }
    @Override
    public int hashCode(){
        final int PRIME = 31;
        int result = 1;
        result = PRIME * result + this.num;
        return result;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        MemberA memberA = (MemberA) o;

        if (num != memberA.num) return false;
        if (!Objects.equals(name, memberA.name)) return false;

        return true;
    }
}
```



테스트 결과 객체의 내부값과, hashCode 재정의를 통해 동일한 값을 가진경우로 처리 되었다.\
![](<../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png>)
