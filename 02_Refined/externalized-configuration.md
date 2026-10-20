---
type: refined
slug: externalized-configuration
tags: [외부설정, externalized-configuration, 설정우선순위, property-precedence, PropertySource, 커맨드라인인자, command-line-arguments, 시스템프로퍼티, system-properties, 환경변수, environment-variables, application.yml, application-profile.yml, 프로필, spring-profile, spring.profiles.default, 완화된바인딩, relaxed-binding, 환경변수이름규칙, 플레이스홀더, placeholder, 기본값, default-value, Could-not-resolve-placeholder, fail-fast, 안전한기본값, secure-by-default, 시크릿관리, secret-management, Dockerfile, docker-compose, 같은jar, build-once-deploy-anywhere, 면접답변]
topic: "외부 설정은 실행 시점에 가까울수록 이기고, 환경변수 이름을 ${이름:기본값} 으로 명시하는 건 완화된 바인딩 규칙과 사람의 직관 사이의 틈을 없애고 기본값 정책을 드러내기 위해서다"
summary: "설정을 코드·yml 에 박으면 값마다 재빌드가 필요해 같은 jar 를 모든 환경에서 쓸 수 없고 시크릿이 git 에 남는다. 스프링은 커맨드라인 > 시스템 프로퍼티 > OS 환경변수 > jar 밖 프로필/기본 yml > jar 안 프로필/기본 yml 순으로 덮어써, jar 안 yml 은 기본값이 되고 배포 환경이 재빌드 없이 덮어쓴다. 환경변수는 완화된 바인딩(점→밑줄, 대시 제거, 대문자)으로 연결되는데 file.upload-dir 은 FILE_UPLOADDIR 이 되어 직관과 어긋나므로, ${FILE_UPLOAD_DIR:...} 처럼 이름을 직접 정해 배포와의 약속이자 목록으로 삼는다. 기본값은 깜빡했을 때 안전하고 티가 나는 경우에만 두고, 기본값 자체가 위험하면 없애 부팅을 실패시킨다."
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-boot/reference/features/external-config.html
  - https://12factor.net/config
updated: 2026-10-20
---

# 외부 설정 (Externalized Configuration)

## 개념 정의

jar 를 **한 번** 빌드해 로컬·컨테이너·운영에서 돌리려면 환경마다 다른 값(DB 주소, 시크릿, 공개 URL)을 jar **밖**에서 넣어야 한다. 코드·yml 에 박으면:

1. 값 하나 바꾸려면 **다시 빌드·배포** → "같은 jar 를 모든 환경에서" 전제가 깨지고 테스트한 jar 와 운영 jar 가 달라진다.
2. 시크릿이 **git 에 남는다** → 공개 저장소의 JWT 시크릿이면 누구나 아무 회원의 토큰을 위조할 수 있다.

## 작동 원리

### 우선순위: 실행 시점에 가까울수록 이긴다

```
커맨드라인 인자          --app.base-url=...
 > 시스템 프로퍼티       -Dapp.base-url=...
 > OS 환경변수          APP_BASEURL=...
 > 설정 파일: jar 밖 application-{profile}.yml
            > jar 밖 application.yml
            > jar 안 application-{profile}.yml
            > jar 안 application.yml
```

- jar 안의 yml 은 **기본값**이고, 배포 환경이 **재빌드 없이** 덮어쓴다. 그래서 위 1번 문제를 이 순서가 해결한다.
- 프로필 파일이 기본 파일을 덮는다. 예: `spring.profiles.default: local` 로 프로필 미지정 시 `application-local.yml` 이 켜지고, 그 안의 개발용 `jwt.secret` 이 `application.yml` 의 `${JWT_SECRET}` 을 덮는다. 다른 프로필을 지정하는 순간 로컬 기본값이 사라진다.

### 완화된 바인딩(relaxed binding): 환경변수 → 프로퍼티

```
점(.) → 밑줄(_),  대시(-) → 제거,  전부 대문자

spring.datasource.url → SPRING_DATASOURCE_URL
file.upload-dir       → FILE_UPLOADDIR      ← 사람이 쓰는 FILE_UPLOAD_DIR 과 다르다
app.base-url          → APP_BASEURL
```

대시를 밑줄로 쓴 형태를 하위 호환으로 받아 줄 수도 있지만, 문서화된 규칙은 "대시 제거"다. 동작 여부를 프레임워크 세부에 맡기게 된다.

### 환경변수 이름을 명시하는 이유

```yaml
spring.h2.console.enabled: ${H2_CONSOLE_ENABLED:false}
file.upload-dir:           ${FILE_UPLOAD_DIR:${user.home}/board-uploads/}
app.base-url:              ${APP_BASE_URL:http://localhost:8080}
jwt.secret:                ${JWT_SECRET}            # 기본값 없음
spring.datasource.url:     jdbc:h2:file:./data/boarddb   # 미명시 — SPRING_DATASOURCE_URL 로 덮어씀
```

| 이유 | 설명 |
|---|---|
| ① 이름을 우리가 정한다 | 대시 제거 규칙에 휘둘리지 않음(`FILE_UPLOAD_DIR`). 프로퍼티를 리팩터링해도 compose·Dockerfile·서버의 이름은 그대로 → **배포와의 약속**. 짧은 이름도 가능(`H2_CONSOLE_ENABLED` vs `SPRING_H2_CONSOLE_ENABLED`) |
| ② yml 이 곧 목록 | `${` 만 찾으면 앱이 받는 환경변수가 전부 보인다. 미명시 방식은 주석에 따로 써야 하고 주석은 코드와 어긋나기 쉽다 |
| ③ 기본값을 바로 옆에 | `${이름:기본값}` 한 줄에 "무슨 환경변수가, 없으면 무엇으로" |

## 트레이드오프 / 한계 — 기본값을 줄지 말지

| 환경변수 | 기본값 | 깜빡하면 | 판단 |
|---|---|---|---|
| `H2_CONSOLE_ENABLED` | `false` | 콘솔 꺼진 채 뜸 | 안전한 기본값 → 둔다 |
| `JWT_SECRET` | 없음 | `Could not resolve placeholder 'JWT_SECRET'` 로 부팅 실패 | 어떤 기본값이든 저장소에 있음 = 위험 → 없앤다 (로컬 편의는 로컬 프로필 파일이 맡음) |
| `APP_BASE_URL` | `localhost` | 조용히 뜨고 공유 미리보기(OG) 주소가 전부 localhost | 안전하지만 **티가 안 남** → 로컬 프로필로 옮겨 로컬 밖에선 필수로 만들 후보 |

> 기준: **깜빡했을 때 기본값이 안전한가**, 그리고 **바로 티가 나는가.** 위험하거나 조용히 틀리면 기본값을 없애 fail fast.

- 환경변수도 완벽한 비밀 저장소는 아니다(프로세스 정보·컨테이너 inspect·로그로 노출 가능). 규모가 커지면 Vault·클라우드 Secrets Manager·k8s Secret 을 검토한다.

> [!WARNING]
> **오답 코너**
> - **"설정을 박으면 환경마다 대처 못 하고 보안에 취약하다"** → 이름으로: **재빌드·재배포** 필요로 같은 jar 전제가 깨짐, 시크릿이 **git 에 남아** 토큰 위조 가능.
> - **"bash 값을 쓴다"** → **OS 환경변수**. 그리고 `-D` 시스템 프로퍼티가 환경변수보다 위.
> - **우선순위만 외우고 "왜"를 빠뜨림** → 실행 시점에 가까울수록 이긴다. jar 는 기본값, 배포가 재빌드 없이 덮어씀.
> - **`app.base-url` 의 환경변수가 당연히 `APP_BASE_URL`** → 문서 규칙상 `APP_BASEURL`. 이 혼동이 바로 이름을 명시하는 이유.
> - **기본값 기준을 시크릿 하나로만 설명("안 뜨는 게 안전")** → 기준은 "깜빡했을 때 기본값이 안전한가". 그래서 H2 콘솔은 기본값을 두고 시크릿은 없앤다.

## 복습 체크

- [ ] 설정을 외부로 빼는 이유 두 가지를 이름으로 말할 수 있다
- [ ] 외부 설정 우선순위를 순서대로 말하고, 그 순서가 맞는 이유를 한 문장으로 말할 수 있다
- [ ] 완화된 바인딩 규칙으로 `file.upload-dir` 의 환경변수 이름을 만들 수 있다
- [ ] `${이름:기본값}` 으로 환경변수 이름을 명시하는 이유 세 가지를 말할 수 있다
- [ ] 기본값을 둘지 말지의 기준을 두 경우(H2 콘솔 vs 시크릿)를 대비시켜 설명할 수 있다

## 관련

[[conditional-on-property]] · [[token-storage-httponly-cookie]] · [[jvm-container-heap-sizing]] · [[filter-interceptor-aop]]
