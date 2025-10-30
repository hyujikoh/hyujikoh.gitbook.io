# JWT with Java Spring 프로젝트 만들기(1) 토큰 생성

간단한 프로젝트로 진행할 예정이기 때문에 현재 spec 은 다음과 같이 구축하였다.

### 프로젝트 설정

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



### 테스트 기반 프로젝트 구성

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

<figure><img src="../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

`build.gradle` 에 추가하였던 라이브러리 레퍼런스를 확인해보면 토큰을 생성하는 로직이 자세하게 명시가 되어있다. ([JWT 토큰 생성](https://github.com/jwtk/jjwt?tab=readme-ov-file#creating-a-jwt))

#### header 설정

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

<figure><img src="../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

`.` 를 기준으로 하나밖에 없는것은 Payload 와 Signiture 를 등록 안했기 때문이다.



#### payload 구성

다음은 JWT Payload 구간을 설정하는 코드다.

```java
// payload 에 대한 내용
builder.claims()
        // 여기 부터 기본 클레임을 설정하는 구간이다.
        .issuer("hyujikoh") // 토큰을 발행하는 발행자에 대해 명시,
        .subject("jwt Project ver 1") // 토큰의 주제에 대해 명시
        .audience()// 토큰에 수신자를 명시, 여러 정보들을 넣을수 있다.
            .add("수신자 1")
            .add("수신자 2")
            .and()
        .expiration(new Date(System.currentTimeMillis() + (1000 * 60))) // 토큰의 만료시간 현재는 테스트이기 때문에 1초 로 설정
        .notBefore(new Date(System.currentTimeMillis() + (1000))) // 해당 시간 이전에는 토큰이 처리되서는 안되는걸 설정하기 위한 명시, 역시 필요가 없으면 사용 안해도 무방
        .issuedAt(new Date()) // 토큰이 발행한 시각 명시
        .id(UUID.randomUUID().toString()) // JWT id 명시 역시 필수는 아니다.
        // 여기까지 기본 클레임을 설정하고 이후부터 커스텀 클레임을 지정하는 구간이다.
        .add("custom-claim-key1","custom-claim-value-1") // 커스텀 클레임을 적용이 가능하다. 운영자가 사용자 토큰을 보다 명시적으로 구분하기 위해 사용해도 좋다.
        .add("custom-claim-key2","custom-claim-value-2")
        .and();
```



이렇게 설정한 builder 를 생성후에 뜯어내면 다음과 같이 나온다.

<figure><img src="../../../../.gitbook/assets/image (2) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

아직 설정을 안한 Signiture 를 제외하고는 정상적으로 토큰에 정보다 상세하게 담겨져 나온다.



#### Signiture

서명 부분을 구성할려면 인코딩된 header, payload, secrect, header 의 알고리즘을 가져와 서명해야한다.&#x20;

서명은 메시지가 도중에 변경되지 않았는지 확인하는 데 사용되며, private 키로 서명된 토큰의 경우 JWT 발신자가 누구인지 확인할 수도 있다.

서명을위해 필요한 메서드는 다음과 같이 구성을 하였다.

```java
private static final String SECRET_KEY_32 = "RBWXHJlYXtmfL5j4+ObYL3L20wns5e/h4uYvT45UxPI=";

// sha 512 시크릿 키
private static final String SECRET_KEY_64 = "SntJsbqFMtFSC0GFXRpOb9OZR64V0Ztv/qRexuZkh4Dpp3TExTLVMBsu4WXkjZQb5UFdo9SL73z5ebYYmisb4w==";

/**
 * 시크릿 키 생성 메서드
 * @return
 */
private SecretKey getSigningKey() {
    byte[] keyBytes = Decoders.BASE64.decode(this.SECRET_KEY_64);
    return Keys.hmacShaKeyFor(keyBytes);
}
```

여기서 별도의 키값을 구분을 했는데, 이는 `hmacShaKeyFor` 와 `byte[]` 타입을 받는 메소드와 매개변수 때문이다. 해당 메소드를 자세히 확인해 보면 다음과 같이 구성되어있다.

<figure><img src="../../../../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

해당 코드를 확인해보면, 매개변수로 받은 `byte[]` 의 size에 따라 암호화 알고리즘이 다르게 res 된다. \
\
각각 다른 시크릿 키를 받음으로서 alg 가 제대로 구성이 되었는지는 현재 가지고 있는 테스트 코드로는 확인이 어렵다. 이를 확인하는 테스트는 추후 작성이 필요하다.



이렇게 작성된 utils 의 createJwt 메소드는 다음과 같이 셋팅이 되었다.



```java
public class JwtUtils {

    // sha 256 시크릿 키
    private static final String SECRET_KEY_32 = "RBWXHJlYXtmfL5j4+ObYL3L20wns5e/h4uYvT45UxPI=";

    // sha 512 시크릿 키
    private static final String SECRET_KEY_64 = "SntJsbqFMtFSC0GFXRpOb9OZR64V0Ztv/qRexuZkh4Dpp3TExTLVMBsu4WXkjZQb5UFdo9SL73z5ebYYmisb4w==";
    
    public String createJwt() {
        // 보다 명시적으로 분리하기 위해 builder 선언
        JwtBuilder builder = Jwts.builder();

        //header 부분,  alg, enc or zip 은 헤더에 추가 안해도 자동으로 생성된다.
        builder.header()
                .keyId("key id") // 옵션, 해당 JWT의 서명하는데 키의 식별자 역할로 사용가능
                .type("jwt") // 토큰의 타입에 대해 명시
                .add("custom-key","custom-value") // 커스텀하게 header 값을 부여할 수 있다. 역시 필요할때 말고는 사용안해도 된다. 또한 Map 형태로 필드 값 부여도 가능하다.
                .contentType("해석본 명시")// 옵션, JWT 가 중첩되어있는 경우 내부 JWT 를 어떻게 해석해야하는지에 대해 명시, 보통은 사용할 필요없음
                .critical() // 무조건적으로 검증해야하는 필드, 없을 경우 아예 검증조차 못하게 수행 가능, 여러개 추가 가능
                    .add("exp")
                    .add("sub")
                .and()
                .and();

        // payload 에 대한 내용
        builder.claims()
                // 여기 부터 기본 클레임을 설정하는 구간이다.
                .issuer("hyujikoh") // 토큰을 발행하는 발행자에 대해 명시,
                .subject("jwt Project ver 1") // 토큰의 주제에 대해 명시
                .audience()// 토큰에 수신자를 명시, 여러 정보들을 넣을수 있다.
                    .add("수신자 1")
                    .add("수신자 2")
                    .and()
                .expiration(new Date(System.currentTimeMillis() + (1000 * 60))) // 토큰의 만료시간 현재는 테스트이기 때문에 1초 로 설정
                .notBefore(new Date(System.currentTimeMillis() + (1000))) // 해당 시간 이전에는 토큰이 처리되서는 안되는걸 설정하기 위한 명시, 역시 필요가 없으면 사용 안해도 무방
                .issuedAt(new Date()) // 토큰이 발행한 시각 명시
                .id(UUID.randomUUID().toString()) // JWT id 명시 역시 필수는 아니다.
                // 여기까지 기본 클레임을 설정하고 이후부터 커스텀 클레임을 지정하는 구간이다.
                .add("custom-claim-key1","custom-claim-value-1") // 커스텀 클레임을 적용이 가능하다. 운영자가 사용자 토큰을 보다 명시적으로 구분하기 위해 사용해도 좋다.
                .add("custom-claim-key2","custom-claim-value-2")
                .and();

        // 서명에 대한 내용
        builder.signWith(getSigningKey());
        return builder.compact();
    }

    /**
     * 시크릿 키 생성 메서드
     * @return
     */
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(this.SECRET_KEY_64);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}

```

테스트를 통해 발급된 JWT를 파싱하면 다음과 같이 나온다.

<figure><img src="../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

이렇게 JWT 를 생성하는 로직을 완료하고 다음으로 토큰을 읽고 정상 JWT 이 아닌경우 에러 처리하는 것을 진행하겠다.&#x20;



현재까지 만든 프로젝트 Repo

{% embed url="https://github.com/hyujikoh/jwt_with_java_spring/tree/feature/issue-1" %}

