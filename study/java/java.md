# Java 구글 드라이브 파일 링크를 통해 파일 다운로드

{% hint style="info" %}
이번에 프로젝트에서 다운로드 링크를 통해&#x20;
{% endhint %}

## 개요

이번에 진행한 프로젝트에서 기능한 구현중 파일 처리를 해야하는 부분이 포함 되어서 기능을 구현한 과정을 작성해봤다.

기능에 대한 기준은 다음과 같다.&#x20;

* 파일  Url는 `drive.google.com` 로 시작되어야 한다.
* 파일 링크를 다운 받을때 전체 공유가 가능해야하고, 아닐경우엔 error 를 처리한다.

## 테스트를 통해서 구현해보기

테스트 코드를  구성해봤다.

```
@Test
public void fileTest(){
    String googleFileUrl;
}
```

<details>

<summary>파일 URL 기준 일치 테스트</summary>

메서드의 매개변수 googleFileUrl 는 구글 드라이브 파일 경로다.&#x20;

기준중에 fileUrl 은 `drive.google.com`  으로 시작을 해야한다.&#x20;

파일 링크는 다음과 같이 가지고 있는 파일로 했다.

테스트를 하면 당연히 실패를 한다.

```
@Test
public void fileTest(){
    String googleFileUrl = "https://drive.google.com/file/d/1HSrfZhvIHgdizKVnji8D7gJqT8n3ApGe/view";

    assertEquals("drive.google.com",googleFileUrl);


}
```

<img src="../../.gitbook/assets/image.png" alt="" data-size="original">

파일은 각각의 id 값을 가지고 경로를 구성하고 있다. 패턴은 거의 일치 하니까 `/` 로 문자열을 분리 해보겠다. 그 안에서 `drive.google.com` 가 있는지 테스트 해보자

```
@Test
public void fileTest(){
    String googleFileUrl = "https://drive.google.com/file/d/1HSrfZhvIHgdizKVnji8D7gJqT8n3ApGe/view";

    String[] urlArrays = googleFileUrl.split("/");

    assertEquals("drive.google.com",urlArrays[2]);
}
```

<img src="../../.gitbook/assets/image (1).png" alt="통과~" data-size="original">



</details>

<details>

<summary>구글 링크 접근 가능여부 (정상  접근)</summary>

이제 제대로된 URL 이란걸 확인을 했으니 파일을 받아야 한다. 하지만, 알다시피 링크가 전체 사용자의 권한이 허용되어야 원활한 다운로드가 가능해진다. 만약 제한된 링크 일경우 정상적인 파일이 다운받아지지 않을것이다.&#x20;

그러기 위해서 우리는 우선 해당 파일 링크가 정상적으로 접근이 가능한 파일인지 확인을하는 테스트를 만들어야할 필요가 있다.

상식적으로 파일이 정상적으로 접근이 가능하면 아무래도 응답 코드는 `200` 이 나올것이다. 그렇지 않을 경우엔 접근 권한 오류인 `403` 에러가 나올것이다.([참고](https://developer.mozilla.org/ko/docs/Web/HTTP/Status/403)) 이 부분에 대해서 한번 테스트코드를 만들어보자.&#x20;

```
@Test
public void fileAccessIsOk(){
    String googleFileUrl = "https://drive.google.com/file/d/1HSrfZhvIHgdizKVnji8D7gJqT8n3ApGe/view";


    //당연히 이렇게 하면 실패가 나올것이다.
    assertEquals("200",googleFileUrl);

}
```

이번엔 실제접근을 `RestTemplate`을 사용하여 접근시도를 해보겠다.



1. build.gradle `implementation` 추가

```
 // build.gradle 에서 다음과 같이 추가
 dependencies {
  implementation('org.springframework.boot:spring-boot-starter-web')
 }
```



2. 테스트 코드에 다음과 같이 `RestTemplate` 을 이용한 요청 코드 작성

```
@Test
public void fileAccessIsOk(){
    String googleFileUrl = "https://drive.google.com/file/d/1HSrfZhvIHgdizKVnji8D7gJqT8n3ApGe/view";

    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.MULTIPART_FORM_DATA);

    // 요청 바디 데이터 설정 (env 및 file)
    MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();

    // 원격 파일 다운로드 URL
    URI url     = URI.create(googleFileUrl);

    // 원격 파일 다운로드
    RestTemplate           rt     = new RestTemplate();
    ResponseEntity<byte[]> res    = rt.getForEntity(url, byte[].class);

    // 응답 코드가 200 인 경우
    assertEquals(HttpStatusCode.valueOf(200),res.getStatusCode());
}
```

그렇게  테스트 를 하면 아래와 같이 브라우저로 접근이 가능한 파일 링크를 위와 같은 코드를 통해 접근이 가능해진다.

<img src="../../.gitbook/assets/image (4).png" alt="test file link" data-size="original">

<img src="../../.gitbook/assets/image (5).png" alt="테스트 통과" data-size="original">



</details>

