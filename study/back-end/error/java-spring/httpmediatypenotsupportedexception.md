---
description: >-
  (2024-02-15) "Content-Type 'application/x-www-form-urlencoded;charset=UTF-8'
  is not supported"
---

# HttpMediaTypeNotSupportedException 오류 해결

### 문제&#x20;

특정 엔드포인트api 를 통해서 진행된 요청에 대한 결과를 타 url 로 리다이렉트 하는 로직을 진행하였다. 로컬 및 각각 별도의 api 에서는 정상 통과가 되어서 서버에 배포를 하였지만, 오류가 다음과 같이 발생하였다.

```javastacktrace
org.springframework.web.HttpMediaTypeNotSupportedException: Content-Type 'application/x-www-form-urlencoded;charset=UTF-8' is not supported
	at org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters(AbstractMessageConverterMethodArgumentResolver.java:209)
	at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.readWithMessageConverters(RequestResponseBodyMethodProcessor.java:163)
	at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.resolveArgument(RequestResponseBodyMethodProcessor.java:136)
	at org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.resolveArgument(HandlerMethodArgumentResolverComposite.java:122)
	at org.springframework.web.method.support.InvocableHandlerMethod.getMethodArgumentValues(InvocableHandlerMethod.java:181)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:148)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:118)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:884)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:797)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1081)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:974)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1011)
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:914)
```

내가 딱히 content-type 을 지정하지 않았지만, 왜 다음과 같은 요청에 대해서 서버가 받지 못한건지 이해가 안되었다.



### 원인

오류가 발생한 api 에 대해서는 다음과 같이 작성하였다.

```java
@ResponseBody
@PostMapping(value = "/auth//login")
@Operation(summary = "어드민 로그인 API",description = " 로그인 API ")
public BaseResponse<AuthenticationInfo> createAdminLogin(@RequestBody PostLoginReq postLoginReq) {
    oAuthService.oAuthLoginOrJoin( postLoginReq);

    return new BaseResponse<>(new AuthenticationInfo(getSocialOAuthRes.getAccessToken(), getSocialOAuthRes.getRefreshToken() , getSocialOAuthRes.getStatus()));
}
```

json 데이터로 RESTFUL 하게 서버와 클라이언트가 서로 통신하길 바랬고, 그래서 요청받는 Dto 에 `@RequstBody` 어노테이션을 통해 dto 를 받을려 했다.&#x20;



`@RequstBody` 어노테이션의 특징은 다음과 같다.&#x20;

<figure><img src="../../../../.gitbook/assets/image (12) (1).png" alt=""><figcaption><p>english.. english</p></figcaption></figure>

자세히 내용을 들여다보면 요청 메시지를 자바 객체로 convertor 해준다고 한다. 좀더 자세히 확인하기 위해 관련된 레퍼런스를 확인해보았다.

{% hint style="info" %}
Simply put, **the&#x20;**_**@RequestBody**_**&#x20;annotation maps the&#x20;**_**HttpRequest**_**&#x20;body to a transfer or domain object, enabling automatic deserialization** of the inbound _HttpRequest_ body onto a Java object.

First, let’s have a look at a Spring controller method:

{% code title="sampleCode.class" overflow="wrap" lineNumbers="true" fullWidth="false" %}
```java
@PostMapping("/request")
public ResponseEntity postController(
  @RequestBody LoginForm loginForm) {
 
    exampleService.fakeAuthenticate(loginForm);
    return ResponseEntity.ok(HttpStatus.OK);
}
```
{% endcode %}

Spring automatically deserializes the JSON into a Java type, assuming an appropriate one is specified.

By default, **the type we annotate with the&#x20;**_**@RequestBody**_**&#x20;annotation must correspond to the JSON sent from our client-side controller:**
{% endhint %}

결국 Spring 에서 `@RequstBody` 는 json 데이터를 java 객체로 역직렬화 해주도록 하는 어노테이션이다.



다시한번 에러로 넘어가자. 이 에러는 `Content-Type 'application/x-www-form-urlencoded;charset=UTF-8' is not supported` 즉 `application/x-www-form-urlencoded` 는 요청한 API가 지원을 안한다는 소리다. Content-Type 이 json 형태로 구성이 되어있지 않은 상태로 json형태의 DTO 를 처리를 Spring 에서 처리를 할수없어서 생긴 에러였다.



### 해결

해결방법은 다음과 같이 선택할수 있었다.

#### 클라이언트에서 요청 을 json 으로 수정

1. 간단한 해결방법이다. 그냥 `content type = application/json`  으로 타입 변경을 함으로서 서버에 공수를 안들게 할 수 있다.
2. 만약 별다른 연계 api 가 없는 상황에서는 이 방법으로 하는것이 정답이지만, 해당 기능은 외부 API 와 연계되는 부분이라서 내가 자체적으로 클라이언트 요청을 수정할 수 없는 상황이다.

#### 서버의 dto 를 json 대신 다른 방식으로 수정

* 이 문제 자체가 서로의 content-type 이 달라서 생긴 이슈이기 때문에 dto json 형태의 body 로 담는것이 아닌 queryParams 방식으로 보내면 요청 자체도 받을수 있고, 크게 공수가 안들것이다.
* 단점 : 이 api 는 회원 인증/인가를 수행하는 api 이기 때문에 dto 를 body 대신 queryParams 방식으로 보낼 경우에 보안에 굉장히 취약해지는 큰 단점이 발생한다.
* 크게 보안을 고려하는 기능이 아닐 경우엔 이렇게 수행해도 크게 문제가 없다.



#### 선택한 방법

* 여러  타입의  사용자와 관리자 인증/인가를 하나의 api 로 발생되는 상황이었기 때문에, 처음엔 공수가 가장 적게 드는 dto 를 queryParams 방식으로 수정을 하였다.&#x20;

```java
@ResponseBody
@PostMapping(value = "/auth//login")
@Operation(summary = "어드민 로그인 API",description = " 로그인 API ")
public BaseResponse<AuthenticationInfo> createAdminLogin(PostLoginReq postLoginReq) {
    oAuthService.oAuthLoginOrJoin( postLoginReq);

    return new BaseResponse<>(new AuthenticationInfo(getSocialOAuthRes.getAccessToken(), getSocialOAuthRes.getRefreshToken() , getSocialOAuthRes.getStatus()));
}
```

* 이후에 문제가 되었던 부분이 queryParams 으로 할 경우에 특수문자(ex. +, \* ,$ , ! 등등) 가 제대로 encoding 이 안되는 문제가 발생하였다. 또한 보안과 관련된 부분에 영향을 미쳤기때문에, 해당 방식으로 진행하는것은 포기하였다.
* 결국 하나의 통합 api를 json dto 를 받는 인증/인가 api 와 content-type 에 크게 영향을 받지않는 api 로 분리함으로서 오류 케이스를 해결하였다.



### 참고 레퍼런스

* [https://www.baeldung.com/spring-request-response-body](https://www.baeldung.com/spring-request-response-body)

