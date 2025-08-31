---
description: 구글의 오픈소스 도구 중 하나인 lighthouse 를 이용해 화면에서 발생하는 랜더링 속도를 저하하는 요소들을 개선한 사례 입니다.
---

# Lighthouse  를 이용한 화면 응답 속도 개선

## 들어가기 앞서 작성하게 된 사유

회사에 입사하여 첫 업무가 특정 페이지에서 발생하는 랜더링 및 지연이 일으키는 부분을 개선하라는 지시가 내려왔습니다.&#x20;

서버 api 응답이 지연 되면 그 부분을 집중적으로 파고 들면 되지만,  Spring mvc 로 구성된 이 플랫폼은 뷰 영역까지 체크를 해야했었습니다. 프론트에 대한 지식이 미천하기 때문에 아는 프론트 지인에게 프론트 페이지 랜더링 지연을 일으키는 문제를 분석하는 방법을 물어봤을때 추천 받았던게 바로 LightHouse 였습니다.

해당 도구를 통해 랜더링 지연 되는 영역을 분석하고, 해결하는 과정을 기억하기 위해 이 포스트를 작성하게 되었습니다.

## 라이트하우스란 무엇인가 <a href="#undefined" id="undefined"></a>

### 소개 <a href="#undefined" id="undefined"></a>

**Google Lighthouse**는 웹페이지의 품질을 개선하는 데 도움이 되는 **오픈소스 자동화 도구**입니다. 구글에서 개발한 이 도구는 공개 웹페이지나 인증이 필요한 모든 웹페이지에서 실행할 수 있으며, 웹사이트의 성능, 접근성, SEO 등에 대한 종합적인 감사 기능을 제공합니다.



### 주요 측정 지표 <a href="#undefined" id="undefined"></a>

### Core Web Vitals

라이트하우스는 Google이 정의한 **Core Web Vitals** 지표들을 중심으로 성능을 평가합니다:

| 지표                                 | 설명                                                | 기준                                        |
| ---------------------------------- | ------------------------------------------------- | ----------------------------------------- |
| **FCP (First Contentful Paint)**   | 사용자가 페이지로 이동한 후 브라우저에서 첫번째 DOM 콘텐츠를 렌더링하는데 걸리는 시간 | 0-1.8s: 좋음, 1.8-3.0s: 개선 필요, 3.0s+: 나쁨    |
| **LCP (Largest Contentful Paint)** | 표시 영역에 표시되는 가장 큰 이미지 또는 텍스트 블록 렌더링 시간             | 0-2.5s: 좋음, 2.5-4.0s: 개선 필요, 4.0s+: 나쁨    |
| **SI (Speed Index)**               | 페이지 로드 중 콘텐츠가 시각적으로 표시되는 속도를 측정                   | 0-3.4s: 좋음, 3.4-5.8s: 개선 필요, 5.8s+: 나쁨    |
| **TBT (Total Blocking Time)**      | 사용자 입력에 응답하지 못하도록 차단된 시간                          | 0-200ms: 좋음, 200-600ms: 개선 필요, 600ms+: 나쁨 |

### 라이트하우스만의 추가 지표

Core Web Vitals 외에도 라이트하우스는 다음과 같은 추가 지표들을 제공합니다:

* **TTI (Time to Interactive)**: 페이지가 완전히 상호작용 가능한 상태가 되는 시간
* **CLS (Cumulative Layout Shift)**: 시각적 안정성을 측정하는 지표

### 지표 활용 방향

#### LCP , FCP 는 우선순위에서 뒤처졌습니다.

사실 저희 서비스에서 성능 측정을 위한 베이스 데이터가 많든 적든 랜더링 하여 보여줬을때 FCP, LCP 는 일정한 수치를 넘어가지 않았습니다. 즉 저희의 첫 content 가 랜더링 되는 부분과 large content 은 저희 서비스 랜더링 속도에 큰 영향을 주지 않았습니다.

#### TBT , Speed Index

앞서 지표중 TBT 는 사용자 입력이 응답하지 못하도록 차단된 시간의 지표였습니다. row 데이터가 많아지면 많아질수록 점수에 영향이 직접적이었습니다.&#x20;

SI(8.4s), TBT(3.53) 라고 가정했을 경우, 조회 페이지에서 시각화 도구 기반 컨텐츠가 나오고, 사용자 입력이 가능하기 까지 총 12s 시간이 발생됨

많은 row 데이터를 chart, grid 형태로 랜더링 하는 과정에  FCP 와 상호작용 시간 사이에 모든 장기 작업(50ms 이상)의 차단 부분(Grid, Chart 등)을 더하여 계산했습니다.

<figure><img src="../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

SI 도 같은 맥락으로 row 데이터를 기반으로 chart , grid content 로 노출이 되기 까지 많은 시간이 소요가 되었습니다.

이 두 지표가 서비스에 측정에 가장 직관적으로 확인이 가능한 부분이여서 이 두 지표를 개선하는 방향으로 성능 개선 작업을 진행하였습니다.

아래는 성능 개선 작업 이전에 성능 점수를 row 데이터를 기준으로 측정한 결과값 입니다. 어느 특정 갯수 이상 부터는 아예 측정 불가한 지표들이 일부 존재하여 평균 10점대의 성능점수가 나오는것으로 확인 되었습니다.

<figure><img src="../../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

다음은 각각 content 에 따른 점수 구간을 테이블 표로 작성한것 입니다. 테이블 보다 chart 를 그릴때 랜더링 속도가 현저히 저하되는 현상을 확인할수 있었습니다.

<figure><img src="../../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>



## 해결 및 조치 과정

위에 나온 수치를 토대로 TBT , SI 를 개선 하기 위한 작업을 위해 라이트 하우스에 나온 리포트 결과를 좀더 면밀히 \
분석하였습니다.

### JS 성능 저하

다음 지표는 row 데이터 3000개 기준으로 성능 측정을 하였을때 피드백 입니다. js 실행시간을 단축하라는 것이 주 내용입니다.&#x20;

<figure><img src="../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>



그러나 row 데이터가 늘어나면 늘어날수록 실행 시간이 기하급수적으로 증가하는 사실이 확인 되었습니다.&#x20;

보통 외부 라이브러리로 사용되는 [AMCHART](https://www.amcharts.com/) 스크립트에서 많은 소모시간이 발생되었습니다.&#x20;

<figure><img src="../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

이렇게 실행시간이 많이 발생하는 부분이 무엇인지 분석해본 결과 가장 큰 원인이 되는 부분이 있었습니다.&#x20;

#### Bullets 으로 인한 성능 지연

AMCHART 에서 차트를 그리는 기능중 bullets 이란 것이 있습니다. 우리가 흔히 차트에 변곡점등 포인트가  Bullets 입니다.  아래 같은 포인트들이 각각 하나의 bullet 입니다.

{% embed url="https://codepen.io/team/amcharts/pen/WJjybm" %}

이  Bullets은 문제가 하나 있는데 직접적으로 성능에 큰 영향을 끼친다는 것입니다. ([공식문서](https://www.amcharts.com/docs/v4/concepts/bullets/#Bullet_performance))

> ### Bullet performance <a href="#bullet_performance" id="bullet_performance"></a>
>
> Since bullets usually come in packs, they might reduce chart performance.
>
> Please check "[Bullets](https://www.amcharts.com/docs/v4/concepts/performance/#Bullets)" section in our "Performance" article for tips on how to set up bullets so they don't bog down your website or application.

저희 서비스는 이전에 나름 운영하던 서비스 중 오래된 서비스에 속했고, 처음에 핸들링 하는 데이터 수도 적었고 그래서 Bullets 으로 차트를 표현했을 해왔습니다. 하지만 서비스가 오래 되면서 활용해야하는 row 데이터는 많아지고 그러다 보니 Bullets 자체가 성능에 영향을 끼치는 범위를 넘어서던 것이었습니다.&#x20;

불필요한 bullets 을 제거하는 것을 목표로 하여 레거시 코드를 리팩토링을 하였습니다.



#### Chart 픽셀 간격 처리

저희 서비스에서 차트를 그릴때 모든 row 데이터를 차트 포인트로 적용하는 방식을 진행하였습니다. 이것도 앞서 설명했던 대로 데이터 가 많아지면서 발생했던 문제였습니다. 차트의 가시성을 위해서 라도 js 개선 작업에 chart  pixel 간격을 처리를 하기로 하였습니다.

AMCHARTS 에서는 차트에 대해서 각 로우 데이터의 차트를 그릴떄  pixel 간격을 지정할수있는 옵션이 있습니다. ([공식문서](https://www.amcharts.com/docs/v4/reference/options/#minPolylineStep_property)) 이를 통해서도 모든 포인트를 그리는 대신 10 pixel 간격으로 차트 그래프를 그리는 방식을 적용하였습니다.

```javascript
am4core.options.minPolylineStep = 10;
```



### 리소스 응답 데이터 압축

라이트 하우스 에서 지속적으로 제시한 보편적인 피드백 중 하나가 텍스트 기반 리소스 압축이었습니다. ([관련 포스트](https://developer.chrome.com/docs/lighthouse/performance/uses-text-compression?hl=ko))

서버에서 정적 리소스 제공 시 따로 압축 알고리즘을 통해 제공되는것이 없이 그대로 제공 되었습니다. 하지만 네트워크 사용 총량을 최소화 하는것이 추후 서비스를 운영하였을때 안정적일것이라 생각하여 텍스트 기반 리소스 들을 [GZIP](https://www.gnu.org/software/gzip/)  또는 Brotli 알고리즘을 통해 리소스를 압축 및 제공하는 방식으로 수정하였습니다.

다음과 같은 작업은 사실 엄청난 절감을 기대하고 한것은 아니였습니다. 다만 이후에 할 작업이라고 하면 지금 당장 하자는 마음으로 진행을 하였고, 성능을 측정했을때 크게 신경을 쓰지 않은 지표인 FCP 와 LCP 지표가 개선이 된 반사이익을 얻었습니다. 😅



<figure><img src="../../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>



### 결과

라이트 하우스를 통해 받은 보고서 및 피드백을 통해 우선순위를 산정하여 하나씩 수행을 하였고 이전 대비 아래와 같은 결과를 얻었습니다.

<figure><img src="../../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

기존 대비 확실히 성능은 개선이 되었던건 사실이었지만, 여전히 많은 데이터를 content 형태로 제공할경우 성능이 저하 되는 부분은 있었습니다.&#x20;

이런 지표는 이후 운영 관점에서 최소한의 데이터 기준을 잡았고, 최대 데이터 적용 기준을 삼는데 많은 참고가 되었습니다.





## 회고

라이트 하우스가 실제 성능 개선을 하는데 중요한 기준이 되었고 많은 도움이 되었고,&#x20;

도구를 활용하고 실제 수치와 결과를 통해 팀원들을 설득 하는 경험도 덩달아 얻었습니다.
