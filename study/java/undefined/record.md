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

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption><p>테스트 결과!</p></figcaption></figure>

</div>

/참고 자료

[https://openjdk.org/jeps/359](https://openjdk.org/jeps/359)(java record info)

[^1]: DTO와 VO, DAO 와 같은건 다음에 다루겠다.
