---
description: docker network 를 이용해 구성한 ELK 클러스터 입니다.
---

# ELK  구축 (1) - 단일 네트워크

## 들어가기 앞서

우선 모의 환경으로 로컬 PC 안에 ELK 를 구축하고, 그 안에 APM 에이전트를 붙인 webapp 을 모니터링을 하는 시나리오로 구축할 예정 입니다.

구성는 다음과 같이 하였습니다.

### 구성 환경

ELK version : 8.16

ELK Repo : [https://github.com/hyujikoh/elastic-stack-docker-part-two](https://github.com/hyujikoh/elastic-stack-docker-part-two)

Java version : 17

Java webapp Repo : [https://github.com/hyujikoh/UrlToKeyProject](https://github.com/hyujikoh/UrlToKeyProject)

Spring version : 3.06



### 구성도



## 본문

### ELK 클러스터 생성

생성 자체는 매우 간단합니다.

ELK repo 에 존재하는 `docker-compose.yml` 를

```bash
docker compose up
```

명령어를 통해 elastic 네트워크로 구성된 elk 클러스터 데모 버전이 실행이 됩니다.

ELK repo 에서 가장 좋았던 부분은 다음과 같았습니다.

1.  인증서 자동 생성

    ELK 8.0 이상 부터는 APM 기능을 구성할려면 ELK 자체에 CA 와 인증서를 기반으로 하는 통신이 필수적으로 되어야 합니다. 이 부분을 setup 서비스가 CA 와 인증서를 자동으로 생성하는 함으로서 초기 설정이 간다하게 설정할수 있었습니다.
2.  독립적인 네트워크 및 서비스 관리

    elastic 라는 docker 네트워크를 설정함으로서 논리적으로 외부와 격리된 환경을 구축할 수 있고, 이를 통해 추후 외부 app 을 모니터링 하는데 일종의 사전 연습을할 수가 있었습니다.





## 참고자료

{% embed url="https://www.elastic.co/kr/blog/getting-started-with-the-elastic-stack-and-docker-compose" %}

{% embed url="https://www.elastic.co/kr/blog/getting-started-with-the-elastic-stack-and-docker-compose-part-2" %}

