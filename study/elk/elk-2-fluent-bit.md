---
hidden: true
---

# ELK 구축 (2) - fluent bit 를 활용한 로그 모니터링

이전 게시글에서는 docker 를 이용해 ELK 클러스터를 구축하고, 그 네트워크에 APM 에이전트를 붙인 demo application 을 컨테이너기반으로 서비스를 올려서 APM 상에 service 목록에 나오는것 까지 진행하였습니다.



<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

1. logstash 와 , 쿠버네티스 기반 앱 서비스 모니터링을 동시에 하고 싶음,&#x20;
2. 모니터링 구조와 그런걸 구성

<figure><img src="../../.gitbook/assets/image(2).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

