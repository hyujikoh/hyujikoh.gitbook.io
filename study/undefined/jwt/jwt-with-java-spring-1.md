# JWT with Java Spring 프로젝트 만들기(1)

간단한 프로젝트로 진행할 예정이기 때문에 현재 spec 은 다음과 같이 구축하였다.

* Java 21
* Spring boot 3.2.5
* DB : 프로젝트 인메모리 형식으로 진행 : 추후 변경될 여지 있음

우선 프로젝트에 JWT 를 적용하기 위해 `build.gradle` 에 JWT 라이브러리를 추가했다.

```gradle
dependencies {
    /**/
    implementation 'io.jsonwebtoken:jjwt-api:0.12.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.5'
    /**/
}
```



해당 프로젝트를 진행할때는 되도록이면 테스트를 수행하면서 진행하기로마음을 먹었기 때문에, 다음과 같이 먼저 Jwt(이하 토큰) 을 생성하는 테스트를 작성하였다.

```java
@SpringBootTest
public class JwtTest {
    @Autowired
    JwtUtils jwtUtils;
    
    @Test
    @DisplayName("jwt 토큰을 생성하는 테스트 코드 입니다.")
    public void createJwt(){
        String jwt = this.jwtUtils.createJwt();

        Assertions.assertNotEquals(jwt,null);
    }
}
```



물론 지금과 같이 할경우 빌드 하는 과정에서 오류가 발생할 것이다. 이를 위해서 Jwt 관련된 기능을 수행하는 JwtUtils 라는 클래스와 토큰을 생성하는 메소드가 필요하다.&#x20;



다음과 같이 구현을 하였다.

<pre class="language-java"><code class="lang-java"><strong>@Component
</strong>public class JwtUtils {
    public String createJwt() {
        return "jwts";
    }
}
</code></pre>



이렇게 수행을할 경우 빌드도 문제없이 되고 초록색 막대가 보이도록 통과가 될것이다. 하지만 이제 해당 메소드를 원래 취지에 맞게 구현을 해야한다.

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

`build.gradle` 에 추가하였던 라이브러리 레퍼런스를 확인해보면 토큰을 생성하는 로직이 자세하게 명시가 되어있다. ([JWT 토큰 생성](https://github.com/jwtk/jjwt?tab=readme-ov-file#creating-a-jwt))



우선 Header 를 구성해보겠다.

```java
@Component
public class JwtUtils {

    public String createJwt() {
        // 보다 명시적으로 분리하기 위해 builder 선언
        JwtBuilder builder = Jwts.builder();

        //header 부분,  alg, enc or zip 은 헤더에 추가 안해도 자동으로 생성된다.
        String jwt = builder.header()
                .keyId("key id") // 옵션, 해당 JWT의 서명하는데 키의 식별자 역할로 사용가능
                .type("jwt") // 토큰의 타입에 대해 명시
                .contentType("해석본 명시")// 옵션, JWT 가 중첩되어있는 경우 내부 JWT 를 어떻게 해석해야하는지에 대해 명시, 보통은 사용할 필요없음
                .critical() // 무조건적으로 검증해야하는 필드, 없을 경우 아예 검증조차 못하게 수행 가능, 여러개 추가 가능, 보통은 사용할 필요없음
                    .add("exp")
                .and()
                .and()
                .compact();
        return jwt;
    }
}
```

이렇게 헤더까지 작업하면 나오는 토큰은 다음과 같이 출력이 된다.

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

`.` 를 기준으로 하나밖에 없는것은 payload 랑 signiture 를 수행을 안했기 때문에 그렇다.

