# record 란?

3줄 요약

1. java 14부터 도입된 기술이다.
2. 반복적인 코드 작성을 줄일수있다.
3. hashCode 와 equals를 이용한 동등 비교를 쉽게 적용이 가능하다.

데이터 전달을 위한 DTO[^1]를 작성할 때 반복적으로 사용되는 코드를 줄이기 위해 java 14 부터는 record 를 도입하였다.

<pre class="language-java"><code class="lang-java"><strong>public class Book{
</strong>    private int id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Book book = (Book) o;
        return id == book.id &#x26;&#x26; Objects.equals(name, book.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }
    
    @Override
    public String toString() {
        return "Book{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
</code></pre>

다음과 같이 Book 클래스의 데이터 접근은 getter 를 통해서 가능하게 구성되어있고, hashCode() 와 equals() 메소드를 재정의 함으로서 동등 비교가 가능하게 설정되어있다. 그 외 여러 메소드를 통해서 클래스를 구성하고 있다. 이렇게 구성되어있는 DTO 를 반복적으로 copy & paste 를 한다 했을때, 코드의 길이가 늘어난다는 단점이 있다.

record 는 이런 단점을 개선 시킨 방법이다. 위와 같은 코드를 단순히 아래와 같은 선언을 통해서 동일한 역할을 수행한다.

```java
public record Book(String name, int id){}
```

이렇게 선언된 레코드 소스를 컴파일 하면 매개변수의 타입과 이름을 이용해서, getter method 와 hashCode() 와 equals(), toString() 메소드하는 코드를 추가해준다. 다음은 테스트 코드를 통해 확인해봤다. 테스트 코드는 다음과 같다.

```java
public class test {
    @Test
    public void classBookTest(){
        Book book1 = new Book(1, "nameA");
        Book book2 = new Book(1, "nameA");
        Assertions.assertEquals(book1.hashCode(),book2.hashCode());
        Assertions.assertEquals(book1,book2);
    }

    @Test
    public void recordBookTest(){
        RecordBook book1 = new RecordBook(1, "nameA");
        RecordBook book2 = new RecordBook(1, "nameA");
        Assertions.assertEquals(book1.hashCode(),book2.hashCode());
        Assertions.assertEquals(book1,book2);
    }
}

// 일반 클래스
class Book{
    private int id;
    private String name;

    public Book(int id, String name){
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Book book = (Book) o;
        return id == book.id && Objects.equals(name, book.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    @Override
    public String toString() {
        return "Book{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}

// record 클래스
record RecordBook(int id, String name){
}

```

<div align="left">

<figure><img src="../../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption><p>테스트 결과!</p></figcaption></figure>

</div>



#### record 의 불편함..?

레코드를 사용하고 나서 코드를 간편하게 줄일수 있고, 직관적으로 작성할 수 있었지만, 반대로 단점도 있었다.&#x20;

1. setter 가 없고, 아예 수정이 불가능하다.

특정 api 에서 운영환경(dev, prod, test 등)에 따라 url 경로를 수정해야 했다. 당연히 dto 를 가공하면 된다라고 생각했는데, 안되는 거였다... 왜 안되는지 찾아보니 각각의 인스턴스 멤버변수들은 final 선언이 자동으로 되어있던 것이다. 각각의 멤버변수들을 수정한다는것 자체가 불가능한것이였다. 따라서 이와 같은 기능을 수행하기 위해서 record 로 된 dto 를 클래스 형태로 수정을 하였다.&#x20;

#### 참고 자료

[https://openjdk.org/jeps/359](https://openjdk.org/jeps/359)(java record info)

[^1]: DTO와 VO, DAO 와 같은건 다음에 다루겠다.
