---
hidden: true
---

# JWT with Java Spring 프로젝트 만들기(2) 토큰 유효성 검증

이전 포스트에서 진행한 JWT 생성 로직을 테스트 했으니 실제 이 토큰이 정상적인지 테스트 코드를 통해 검증하는 방식으로 진행할 것이다.&#x20;

테스트는 다음과 같이 진행할것이다.&#x20;

1. 발급된 토큰이 제대로 파싱이 되었는지 테스트
2. 유효기간이 만료된 토큰일경우 에러 처리가 정상적으로 되는지 테스트
3. 실제 토큰을 만들때 사용한 서명과 검증 시에 사용한 서명이 일치하지 않을 경우 에러 처리 테스트



### 토큰 유효성 정상 테스트

기능을 만들기 위해서 테스트코드를 다음과 같이 만들었다.

```java
@Test
@DisplayName("토큰을 읽는 테스트 코드 입니다.")
public void readJwt(){
    String jwt = this.jwtUtils.createJwt();

    Assertions.assertTrue(this.jwtUtils.isValidJwt(jwt));
}
```

테스트 코드는 토큰을 생성하고, 해당 토큰 파싱을 통해 서명 및 유효성이 제대로 통과를 하는것을 목적으로 두는 테스트다.



처음이 테스트는 아마 빌드부터 안될것이다. `isValidJwt` 라는 메소드 자체가 없기 때문이다.&#x20;

메소드를 만든다면 다음과 같이 구성되어있을것이다.<br>

```java
public boolean isValidJwt(String token) {
        return token.getClass() == String.class;
}
```

이렇게 하면 테스트는  통과는 될것이다.&#x20;

<figure><img src="../../../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

이제 이 코드를 정상 코드로 리팩토링이 필요하다.



```java
public boolean isValidJwt(String token) {
    if (token == null || token.isBlank() || token.isEmpty()) {
        throw new NullPointerException();
    }
    return true;
}
```

우선 처음으로 토큰 자체가 빈값이나, null , 공백 등 자체 문제 일 경우를 에러를 처리할 필요가 있다.



```java
public boolean isValidJwt(String token) {
    if (token == null || token.isBlank() || token.isEmpty()) {
        throw new NullPointerException();
    }
    String jwt = token.replaceAll("^Bearer( )*", "");
    try {
        Jwt<?, ?> parse = Jwts.parser()
                .setSigningKey(SECRET_KEY_64)
                .build()
                .parse(jwt);
    } catch (Exception e) {
        throw e;
    }
    return true;
}
```

그 다음 토큰을 담을때, 불필요한 텍스트 정보를 제거 후, 파싱 작업을 거친다.&#x20;

파싱 로직은 다음과 같이

서명을 통해 검사를 하려는 토큰에 대해 유효성에 문제 없다면 parse 객체에 넣는 작업을 한다.



이렇게 만든 로직 기반으로 테스트를 해보았다.

<figure><img src="../../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

다행히 지금 만든 코드는 정상적으로 통과가 되었다.&#x20;



### 만료된 토큰일경우

이경우엔 기존에 만든 생성 메소드로는 만료에 대한 케이스를 확인하는데 어렵다. 물론 테스트를 위해 테스트 메소드를 만들수 있지만, 생성 메소드에 시간을 설정하는 매개변수를 추가함으로 유효시간을 조절해보기로 했다.



```java
public String createJwt(Long time) {

    builder.claims()
    .expiration(new Date(System.currentTimeMillis() + time))

}
```

이렇게 수정한 매개변수를 테스트 코드에도 매개변수를 추가한다음 테스트를 다시한번 돌려봤다.&#x20;



<figure><img src="../../../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>
