# WIL: 이번주 공부 (2월 4주차)

### TL; WR

수동 배포에서 GitHub Actions 기반 CI/CD 파이프라인으로 전환하면서, Docker/SSH/셸 스크립트/GitHub 인프라에 대한 실전 경험을 쌓았다.

***

### 좋았던 점 & 배웠던 점

#### 1. CI/CD 파이프라인의 설계 원칙

처음으로 CI/CD 파이프라인을 설계부터 구현까지 해봤다. 핵심은 **CI(빌드)는 공통, CD(배포)는 환경별 분리**라는 원칙이다. 같은 소스에서 이미지를 빌드하지만, 개발서버(Docker Compose)와 운영서버(Kubernetes)는 배포 방식이 다르기 때문에 CD만 분기하는 구조가 자연스럽다.

이미지 태그 전략도 중요했다. `:latest`로 덮어쓰면 이전 버전이 유실되어 롤백이 불가능하지만, 커밋 SHA 기반 태그(`dev-a1b2c3d`)를 사용하면 모든 버전이 보존되고 특정 커밋으로 즉시 롤백할 수 있다.

#### 2. Docker 생태계에 대한 이해

**Docker Compose V1과 V2의 차이**를 실전에서 부딪히며 배웠다. V1(`docker-compose`, Python 기반)은 2023년 7월 EOL이고, V2(`docker compose`, Go 기반)는 Docker CLI 플러그인으로 동작한다. 패키지 매니저가 없는 서버에서도 `~/.docker/cli-plugins/`에 바이너리만 넣으면 V2를 사용할 수 있다는 것도 알게 되었다.

**Docker API 버전 호환성**도 배웠다. 최신 Compose V2를 설치했더니 `client version 1.53 is too new. Maximum supported API version is 1.41` 오류가 발생했다. Docker Engine 20.10.x는 API 1.41까지만 지원하므로, Compose도 호환 버전(v2.20.3)을 설치해야 했다. **도구를 설치할 때는 서버의 기존 런타임 버전과의 호환성을 반드시 확인해야 한다.**

**Docker Compose 프로젝트명 메커니즘**도 처음 알았다. Docker Compose는 디렉토리명을 프로젝트명으로 사용하기 때문에, 같은 이름의 디렉토리에서 서로 다른 서비스를 실행하면 충돌이 발생한다. `-p` 옵션이나 `COMPOSE_PROJECT_NAME` 환경변수로 명시적으로 구분해야 한다.

#### 3. GitHub Actions & GHCR 권한 체계

GHCR(GitHub Container Registry)에 이미지를 push하려면 **두 계층의 권한**이 필요하다는 것을 배웠다:

1. Organization 레벨: Actions의 Workflow permissions
2. 패키지 레벨: 각 GHCR 패키지의 Manage Actions access

워크플로우 YAML에 `permissions: packages: write`를 지정해도 Organization 정책이 우선한다. 또한 `GITHUB_TOKEN`은 **워크플로우 실행 시작 시점에 발급**되기 때문에, 권한을 변경한 뒤 기존 실행을 Re-run해도 이전 토큰을 재사용한다. 새 push로 워크플로우를 다시 트리거해야 변경된 권한이 적용된다.

#### 4. SSH 키 인증의 원리

SSH 키 쌍의 동작 원리를 제대로 이해했다. 공개키는 서버의 `~/.ssh/authorized_keys`에 등록하고(자물쇠), 개인키는 접속하는 쪽(GitHub Secrets)이 보관한다(열쇠). SSH는 보안 정책상 `~/.ssh`(700)과 `authorized_keys`(600)의 권한이 느슨하면 키 인증 자체를 거부한다. 권한 설정을 빠뜨리면 "왜 안 되지?"로 시간을 낭비하게 된다.

ed25519가 RSA보다 키 길이가 짧으면서도 보안성이 높다는 것, 자동화 용도의 키는 passphrase를 비워야 한다는 것도 배웠다.

#### 5. 셸 스크립트 안전성

배포 스크립트를 작성하면서 셸 스크립트의 방어적 프로그래밍을 배웠다:

* **`set -euo pipefail`**: 에러 즉시 종료 + 미정의 변수 감지 + 파이프 실패 감지. 다만 `grep`이 결과 없을 때 exit 1을 반환하므로 `|| true` 패턴이 필수
* **`sed -i`의 이식성 문제**: macOS(BSD)와 Linux(GNU)에서 동작이 다르다. `sed ... > tmp && mv tmp original` 방식이 크로스 플랫폼 해법
* **Alpine 이미지에는 curl이 없다**: `wget`은 Alpine에 기본 포함되지만 curl은 아니다. 컨테이너 내부에서 실행되는 명령(healthcheck 등)은 이미지에 포함된 도구를 확인해야 한다

#### 6. 수동 → 자동화의 단계적 진화

처음에 수동 배포 스크립트(Phase 1)를 만들고, 이후 GitHub Actions(Phase 2)로 전환했다. Phase 1에서 고민했던 안전장치(.env 백업, 실패 시 복원, 입력값 검증)가 Phase 2 워크플로우 설계에 그대로 반영되었다. 수동으로 먼저 해보는 경험이 자동화의 품질을 높인다는 걸 체감했다.

***

### 아쉬운 점

#### 1. 서버 환경 사전 조사 부족

서버에 Git 저장소가 없고 docker-compose.yaml만 있다는 사실, Docker Compose V2가 설치되어 있지 않다는 사실, SSH 포트가 기본(22)이 아니라는 사실을 모두 **구현 중에 발견**했다. 설계 전에 서버 환경을 철저히 조사했다면 Phase 1 → Phase 2 전환이나 연쇄 트러블슈팅 없이 한 번에 갈 수 있었을 것이다.

#### 2. Docker/인프라 기초 지식 부족

Docker Compose 프로젝트명 메커니즘, Docker API 버전 호환성, GHCR의 이중 권한 구조 등 기본적인 인프라 지식이 부족해서 트러블슈팅에 시간이 많이 걸렸다. 문제가 발생할 때마다 하나씩 배우는 것도 좋지만, Docker/Container 기초를 체계적으로 공부해두면 문제 원인을 더 빠르게 파악할 수 있을 것 같다.

#### 3. 헬스체크 엔드포인트 설계

Spring Boot Actuator 없이 단순히 `"OK"`를 반환하는 헬스체크 엔드포인트를 만들었는데, DB 연결 상태, 디스크 용량, 외부 서비스 연결 등을 확인하지 않으므로 실질적인 헬스체크로는 부족하다. 이후에 의미 있는 헬스체크로 개선이 필요하다.

#### 4. 롤백 시나리오 테스트 미흡

자동 롤백 로직을 구현했지만, 실제로 헬스체크가 실패하는 상황을 만들어서 롤백이 정상 동작하는지 테스트하지 못했다. 배포 파이프라인에서 롤백은 가장 중요한 안전장치이므로, 의도적으로 실패 시나리오를 만들어 검증해야 한다.

***

### 종합 정리

이번 주는 "인프라는 설계대로 한 번에 되지 않는다" 는 것을 온몸으로 체감한 한 주였다.

CI/CD 파이프라인을 구축하면서 Docker(이미지 빌드, Compose, API 버전), GitHub(Actions, GHCR, Secrets, Organization 권한), Linux(SSH 키 인증, 셸 스크립트 안전성), 배포 전략(커밋 SHA 태깅, 헬스체크, 자동 롤백)까지 넓은 범위의 인프라 지식을 실전에서 익혔다.

특히 인상적이었던 건 **수동으로 먼저 해보는 것의 가치**다. Phase 1에서 배포 스크립트를 직접 작성하면서 `.env 백업 → 배포 → 실패 시 복원`이라는 패턴의 필요성을 체감했고, 이것이 Phase 2의 GitHub Actions 워크플로우에 자연스럽게 녹아들었다. 자동화를 잘 하려면 수동 과정을 먼저 이해해야 한다.

다음 단계로는 운영 배포 파이프라인(main → Cloud Image Registry → Kubernetes)과 GitOps(ArgoCD) 도입이 남아있다. 이번 주에 쌓은 Docker/GitHub Actions 경험이 기반이 될 것이다.
