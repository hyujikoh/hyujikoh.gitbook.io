---
description: JWT 에 대해 이해하고 공부를 위한 POST
---

# JWT🎫

## 서론

### 목적

이 POST를 통해, 사용자 인증 방법중 하나인 JWT 를 이해하고 간단한 프로젝트를 통해 인증/인가를 구현하는 서비스를 구현해보고자 한다.



## 본론

### WHAT is JWT&#x20;

> JSON Web Token (JWT) is an open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with the **HMAC** algorithm) or a public/private key pair using **RSA** or **ECDSA**.
>
> Although JWTs can be encrypted to also provide secrecy between parties, we will focus on _signed_ tokens. Signed tokens can verify the _integrity_ of the claims contained within it, while encrypted tokens _hide_ those claims from other parties. When tokens are signed using public/private key pairs, the signature also certifies that only the party holding the private key is the one that signed it.
>
> -jwt.io 소개란 발췌

정리하자면 당사자 간의 정보를 안전하게 전송하기 위한 방법으로 JSON 객체로 정의하는 표준 방법( RFC 7519) 을 JWT 라고 한다. JWT 는 비밀키 또는 공개/개인 키(RSA or ECDSA)를 사용하여 서명할 수 있다.

#### RFC 와 RFC 7519

앞서 jwt.io 에서 소개한 내용중 RFC 7519 가 어떤것인지 알아보기 위해 이리저리 한번 찾아봤다.

RFC 자체는 인터넷과 관련된 다양한 기술, 프로토콜 절차등을 설명하고 정의하는 정의하는 일련의 문서들을 의미한다. RFC 문서들은 제안 및 검토과정을 거쳐서 특정 기술과 프로토콜에 대한 합의가 이루어지면 공식적인 문서로 발행된다. RFC 문서는 다음과 같이 여러 카테고리로 분류가 된다.

* **표준(Stantards)** : 웹에서 수용되고 사용되는 기준이 되는 기술이나 프로토콜에 대한 명세
* **정보(Infomation)** : 일반적인 정보 제공을 목적으로 하며 표준화 과정을 거치지 않은 기술이나 정책에 대한 설명을 포함될수 있음.
* **실험적(Experimental)** : 실험적인 기술이나 아이디어에 대해 설명하며, 정보 및 표준 문서가 되기전에 테스트와 검토가 필요한 경우에 해당.
* **최선의 현재 관행(Best Current Practices)**: 웹에서 인정하는 최선의 운영 관행에 대한 가이드라인 제공

정리를 하자면 RFC 는 일련의 기술들을 정리한 문서들이고, 그중에서 우리가 사용하는 JWT는 RFC 7519 라는 명칭으로 RFC 문서로 관리되고 있다.

실제 RFC 에서 관리되고 있는 목록을 보면 RFC 7519 의 status 는 `Proposed Standard` 표준으로 제안이 되어있는 상태고, `Best Current Practices` 는 RFC 8725 로 되어있다는것을 참고하면 좋을것 같다.

<figure><img src="../../../../.gitbook/assets/image (29).png" alt=""><figcaption><p>RFC 8725</p></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (30).png" alt=""><figcaption><p>RFC 7519</p></figcaption></figure>



### JWT structure

JWT 의 구성은 간단하다. 특별하게 구성된 경우를 제외하고 `.` 를 기준으로 토큰은 3개의 파트로 나누어져있는데, `header` , `payload` , `Siginture` 로 구성되어있다.  따라서 JWT 정보를 보면 다음과 같이 구성되어있다.

`xxxxx.yyyyy.zzzzz`

#### header

헤더( header) 는 보통 2개의 부분으로 구성이 되어있는데, 토큰의 타입(ex. JWT) 과 어떤 알고리즘으로 서명으로 사용하였는지(ex. HMAC SHA256 or RSA.) 로 구성되어있다.

다음과 같은 예시처럼 구성되어있다.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### Payload

payload는 토큰에 담을 클레임(claim) 정보를 포함하고 있다. 페이로드에 담는 정보의 한 조각을 클레임이라 부르고 클레임은 엔티티에 관한 진술과 추가적인 데이터를 말한다. 이는 name/value 쌍으로 이루어지고 있다. 클레임은 registered claim, public claim, private claim 세 종류로 분류된다.

* **Registered Claim** : 상호 운용 가능하고 필수는 아니지만 권장되는 미리 정의된 클레임 세트. 해당 클레임으로는 **iss** (issuer), **exp** (expiration time), **sub** (subject), **aud** (audience), **nbf**(Not Before), **iat** (Issued At) 이 있다.
* **Public Claim** : JWT 를 사용하는 사람들이 마음대로 정의할 수 있는 클레임이다.&#x20;
* **Private Claim** : 사용자 정의 클레임으로 사용에 동의하는 당사자 간 정보를 공유하기 위한 클레임

페이로드는 다음과 같은 예시로 되어있다.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

{% hint style="warning" %}
서명된 토큰과 같은 경우 변조로부터 보호되지만, 해당 정보는 누구나 읽을 수 있습니다. JWT 가 암호화되지 않은 한 비밀 정보를 페이로드나 헤어요소에 넣지 말아야한다.
{% endhint %}

#### Siginture&#x20;

서명 부분을 생성할려면 인코딩된 헤더, 페이로드, 헤더에 지정된 알고리즘을 가져와 서명해야한다. 서명에는 secrect key 를 포함하여 암호화되어 있다. 서명은 메시지가 중간에 변경되지 않았는지 확인하는 데 사용되면, 개인 키로 서명된 토큰의 경우 JWT 발신자가 누구인지 확인이 가능하다.



## 마무리

이렇게 JWT 에 대해서 알아보는 POST 를 작성하였는데, JWT 를 활용하여 인증/인가를 구현한 간단한 프로젝트를 만듬으로써 보다 더 이해를하는 시간을 가져볼 것이다.



## 참고 레퍼런스 및 사이트

* RFC editor\
  [https://www.rfc-editor.org/search/rfc\_search\_detail.php](https://www.rfc-editor.org/search/rfc_search_detail.php)
* JWT INFO\
  [https://jwt.io](https://jwt.io/)
*   JWT (JSON Web Token) 이해와 활용

    [http://www.opennaru.com/opennaru-blog/jwt-json-web-token/](http://www.opennaru.com/opennaru-blog/jwt-json-web-token/)
