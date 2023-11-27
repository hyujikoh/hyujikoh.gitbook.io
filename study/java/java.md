# Java 구글 드라이브 파일 링크를 통해 파일 다운로드

{% hint style="info" %}
이번에 프로젝트에서 구글  드라이브링크를 통해  파일을 다운하는 법에 대해 정리
{% endhint %}

## 개요

이번에 진행한 프로젝트에서 기능한 구현중 파일 처리를 해야하는 부분이 포함 되어서 기능을 구현한 과정을 작성해봤다.

기능에 대한 기준은 다음과 같다.&#x20;

* 파일  Url는 `drive.google.com` 로 시작되어야 한다.
* 파일 링크를 다운 받을때 전체 공유가 가능해야하고, 아닐경우엔 error 를 처리한다.

## 테스트를 통해서 구현해보기

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

<summary>구글 링크 접근 가능여부 테스트</summary>

### Access 전체 허용 URL 일 경우

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


    MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();

  
    URI url     = URI.create(googleFileUrl);


    RestTemplate           rt     = new RestTemplate();
    ResponseEntity<byte[]> res    = rt.getForEntity(url, byte[].class);

    // 응답 코드가 200 인 경우
    assertEquals(HttpStatusCode.valueOf(200),res.getStatusCode());
}
```

그렇게  테스트 를 하면 아래와 같이 브라우저로 접근이 가능한 파일 링크를 위와 같은 코드를 통해 접근이 가능해진다.

<img src="../../.gitbook/assets/image (4).png" alt="test file link" data-size="original">

<img src="../../.gitbook/assets/image (5).png" alt="테스트 통과" data-size="original">



### &#x20;Private URL 인 경우(접근불가)

앞서 예시를 든 URL은 전체 허용 URL 이다. 그러면 반대로 제한된 사용자만 허용된 URL 일 경우 어떻게 될까? 당연히 접근이 가능한 URL 이 status 가 `200 OK` 가 나왔으니까, `403` 에러가 나오지 않을까 싶다.

역시 이를 위해 접근 제한 파일을 생성했다.

<img src="../../.gitbook/assets/image (6).png" alt="" data-size="original">

이 파일 URL 을 테스트코드를 작성하여 결과값을 403 으로 예상해보자

```
@Test
public void fileAccessIs404(){
    String privateFileUrl = "https://drive.google.com/drive/folders/13X0o5nwn6AQZrIKH1RDODQsfLOs9Zvu4";

    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.MULTIPART_FORM_DATA);

  
    MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();

 
    URI url     = URI.create(privateFileUrl);

  
    RestTemplate           rt     = new RestTemplate();
    ResponseEntity<byte[]> res    = rt.getForEntity(url, byte[].class);


    assertEquals(HttpStatusCode.valueOf(403),res.getStatusCode());
}
```

<img src="../../.gitbook/assets/image (7).png" alt="" data-size="original">

테스트 코드의 결과값은 200 이 나왔다. 그러면 전체 허용과, 제한된 허용 URL 에 대해서 res 가 어떻게 오는지 확인을 해보자

```
 // 정상적으로 허용 됐을떄 나오는 res
 <200 OK OK,[B@5bb8f9e2,[Content-Type:"text/html; 
 charset=utf-8", 
 X-Robots-Tag:"noindex, 
 nofollow, nosnippet", 
 Cache-Control:"no-cache, 
 no-store, max-age=0, 
 must-revalidate", 
 Pragma:"no-cache", 
 Expires:"Mon, 01 Jan 1990 00:00:00 GMT", 
 Date:"Tue, 14 Nov 2023 16:43:28 GMT", 
 P3P:"CP="This is not a P3P policy! See g.co/p3phelp for more info."", 
 Content-Security-Policy:"require-trusted-types-for 'script';report-uri https://csp.withgoogle.com/csp/docs-tt", 
 Referrer-Policy:"origin", 
 X-Content-Type-Options:"nosniff", 
 X-Frame-Options:"SAMEORIGIN", 
 X-XSS-Protection:"1; mode=block", 
 Server:"GSE", 
 Set-Cookie:"NID=511=NDGT2a_-W7MeztF7siZOTPE5KPniaiBnlowzhUPHT5QxzCFE8eEZ9fgTt7J-IpnLytotlsHoUgpMkJyvMpjNbozZQWmb8OMgy0pxWhW5rNpKfiO0EDgmuj9t24I2VuOAPINvNFzy3lnae-oe5-kZJKGPZuGl5xKQU8H32W-HjQg; expires=Wed, 
 15-May-2024 16:43:28 GMT; path=/; domain=.google.com; HttpOnly", Alt-Svc:"h3=":443"; ma=2592000,h3-29=":443"; ma=2592000", Accept-Ranges:"none", Vary:"Sec-Fetch-Dest, 
 Sec-Fetch-Mode, Sec-Fetch-Site,Accept-Encoding", Transfer-Encoding:"chunked"
 ]>
 
 // 제한된 URL 접근시 나오는 res
 <200 OK OK,[B@7c8326a4,[Content-Type:"text/html; 
 charset=utf-8", 
 X-Frame-Options:"DENY", 
 Set-Cookie:"__Host-GAPS=1:FMaHaUN8FfUi6KULkFHVMbbpyefSjw:OKDv91b43OeHpU6M; 
 Expires=Thu, 13-Nov-2025 16:45:59 GMT; 
 Path=/; Secure; HttpOnly; 
 Priority=HIGH", x-auto-login:"realm=com.google&args=service%3Dwise%26continue%3Dhttps://drive.google.com/drive/folders/13X0o5nwn6AQZrIKH1RDODQsfLOs9Zvu4", 
 Link:"<https://www.google.com/intl/en-US/drive/>; rel="canonical"", 
 x-ua-compatible:"IE=edge", 
 Cache-Control:"no-cache, 
 no-store, max-age=0, 
 must-revalidate", 
 Pragma:"no-cache", 
 Expires:"Mon, 01 Jan 1990 00:00:00 GMT", Date:"Tue, 14 Nov 2023 16:45:59 GMT", Strict-Transport-Security:"max-age=31536000; includeSubDomains", Cross-Origin-Resource-Policy:"same-site", Report-To:"{"group":"AccountsSignInUi","max_age":2592000,"endpoints":[{"url":"https://csp.withgoogle.com/csp/report-to/AccountsSignInUi"}]}", 
 Content-Security-Policy:"script-src 'report-sample' 'nonce-SXrtyaUCr9TPV8DafHMdCw' 'unsafe-inline';object-src 'none';base-uri 'self';report-uri /v3/signin/_/AccountsSignInUi/cspreport;worker-src 'self'", "require-trusted-types-for 'script';report-uri /v3/signin/_/AccountsSignInUi/cspreport", 
 Accept-CH:"Sec-CH-UA-Arch, Sec-CH-UA-Bitness, Sec-CH-UA-Full-Version, Sec-CH-UA-Full-Version-List, Sec-CH-UA-Model, Sec-CH-UA-WoW64, Sec-CH-UA-Form-Factor, Sec-CH-UA-Platform, Sec-CH-UA-Platform-Version", Permissions-Policy:"ch-ua-arch=*, ch-ua-bitness=*, ch-ua-full-version=*, ch-ua-full-version-list=*, ch-ua-model=*, ch-ua-wow64=*, ch-ua-form-factor=*, ch-ua-platform=*, ch-ua-platform-version=*", 
 Cross-Origin-Opener-Policy-Report-Only:"same-origin; report-to="AccountsSignInUi"", Server:"ESF", X-XSS-Protection:"0", X-Content-Type-Options:"nosniff", Alt-Svc:"h3=":443"; ma=2592000,h3-29=":443"; ma=2592000", Accept-Ranges:"none", Vary:"Sec-Fetch-Dest, 
 Sec-Fetch-Mode, Sec-Fetch-Site,Accept-Encoding", Transfer-Encoding:"chunked"]
 >
```

뭐... 결과만 보면 `X-Frame-Options` 이 `DENY` 일때 접근 제한자 URL 이라는걸 알수가 있다.&#x20;

지금 서비스 로직을 변경 하여 충분히 구현을 할수 있지만, 이건 추후 리팩토링을 통해 구현할 것이다. \
우선은 해당 res 의 `X-Frame-Options` 여부를 체크하여 URL 접근 여부를 확인하자

```
@Test
public void fileAccessIs403(){
    String privateFileUrl = "https://drive.google.com/drive/folders/13X0o5nwn6AQZrIKH1RDODQsfLOs9Zvu4";

    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.MULTIPART_FORM_DATA);

    // 요청 바디 데이터 설정 (env 및 file)
    MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();

    // 원격 파일 다운로드 URL
    URI url     = URI.create(privateFileUrl);

    // 원격 파일 다운로드
    RestTemplate           rt     = new RestTemplate();
    ResponseEntity<byte[]> res    = rt.getForEntity(url, byte[].class);

 
    Assertions.assertTrue(res.toString().contains("X-Frame-Options:\"DENY\""));
}
```

테스트 결과는 다음과 같이 나왔다!\


<img src="../../.gitbook/assets/image (9).png" alt="" data-size="original">



</details>

<details>

<summary>즉시 다운이 가능한 링크로 변환 및 다운로드</summary>

이렇게 접근 가능한 링크는 즉시 다운로드가 가능한 링크가 아니다. 이 링크를 다운로드 링크로 변경 할 필요가 있다.&#x20;

다음과 같은 서비스 로직을 통해 파일 다운로드 링크로 변환을 해봤다. 여기서 필요한건 드라이브 내 파일의 id 를 이용해 만드는 것이다.

<pre><code><strong>public String fileUrlToDownloadUrl(String imageUrls) {
</strong>        String[] splitLink = imageUrls.split("/");

        // 이 부분은 파일의 ID를 추출하는 부분이므로 파일에 따라 인덱스가 다를 수 있습니다.
        String fileId = splitLink[splitLink.length-1].contains("view") ? splitLink[splitLink.length-2]:splitLink[splitLink.length-1];
        return "https://docs.google.com/u/0/uc?export=download&#x26;confirm=t&#x26;id=" + fileId;
}
</code></pre>

이렇게 만든 URL은 `confirm=t` 를 포함하는데, 구글의 대용량 파일에 대해서 재차 확인을 요구하기 때문에 이 URL 에 대해서 신뢰하겠다라는 일종의 동의 같은것이라 보면 된다.

이렇게 만든 URL 을 이용해 다운로드 및 저장하는 로직을 만들어 보겠다.

```
@Test
public void finalFileDownload(){
    String googleFileUrl = "https://drive.google.com/file/d/1HSrfZhvIHgdizKVnji8D7gJqT8n3ApGe/view";

    String downloadURl = this.fileUrlToDownloadUrl(googleFileUrl);

    URI url     = URI.create(downloadURl);

    // 원격 파일 다운로드
    RestTemplate           rt     = new RestTemplate();
    ResponseEntity<byte[]> res    = rt.getForEntity(url, byte[].class);

    // 접근 오류 발생시 오류 throw
    if(res.toString().contains("X-Frame-Options:\"DENY\"")){
        
    }

    byte[] buffer = res.getBody();

    // 로컬 서버에 저장
    String fileName = UUID.randomUUID().toString();                    // 파일명 (랜덤생성)
    String ext      = ".zip"; // 파일을 확장자를 추출, 나는 zip 파일만 사용하기 때문에 하드코딩으로 넣었다.
    Path target   = Paths.get("", fileName + ext);    // 파일 저장 경로

    try {
        FileCopyUtils.copy(buffer, target.toFile());

        Files.delete(target);


    } catch (IOException e) {
        e.printStackTrace();
    }
}

public String fileUrlToDownloadUrl(String imageUrls) {
    String[] splitLink = imageUrls.split("/");

    // 이 부분은 파일의 ID를 추출하는 부분이므로 파일에 따라 인덱스가 다를 수 있습니다.
    String fileId = splitLink[splitLink.length-1].contains("view") ? splitLink[splitLink.length-2]:splitLink[splitLink.length-1];
    return "https://docs.google.com/u/0/uc?export=download&confirm=t&id=" + fileId;
}
```





</details>
