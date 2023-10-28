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

