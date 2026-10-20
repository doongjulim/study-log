---
date: 2026-10-20
type: raw
author: dongju
tags: [외부설정, externalized-configuration, 설정우선순위, property-precedence, PropertySource, 커맨드라인인자, command-line-arguments, 시스템프로퍼티, system-properties, 환경변수, environment-variables, application.yml, application-profile.yml, 프로필, spring-profile, spring.profiles.default, 완화된바인딩, relaxed-binding, 환경변수이름규칙, 플레이스홀더, placeholder, 기본값, default-value, Could-not-resolve-placeholder, fail-fast, 안전한기본값, secure-by-default, 시크릿관리, secret-management, JWT_SECRET, H2_CONSOLE_ENABLED, FILE_UPLOAD_DIR, APP_BASE_URL, SPRING_DATASOURCE_URL, Dockerfile, docker-compose, 같은jar, build-once-deploy-anywhere, study-board, 면접답변]
topic: "외부 설정은 실행 시점에 가까울수록 이기고(커맨드라인 > 시스템 프로퍼티 > 환경변수 > 프로필 yml > 기본 yml), 환경변수 이름을 ${이름:기본값} 으로 명시하는 건 완화된 바인딩 규칙과 사람의 직관 사이의 틈을 없애고 기본값 정책을 한 줄에 드러내기 위해서다"
summary: "설정을 코드·yml 에 박으면 값마다 재빌드가 필요해 같은 jar 를 모든 환경에서 쓸 수 없고 시크릿이 git 에 남는다. 스프링은 실행 시점에 가까운 소스가 이기도록 커맨드라인 > -D 시스템 프로퍼티 > OS 환경변수 > jar 밖 프로필/기본 yml > jar 안 프로필/기본 yml 순으로 덮어쓴다. 환경변수는 완화된 바인딩(점→밑줄, 대시 제거, 대문자)으로 프로퍼티에 연결되는데 file.upload-dir 은 FILE_UPLOADDIR 이 되어 사람이 쓰는 FILE_UPLOAD_DIR 과 어긋난다. 그래서 study-board 는 ${FILE_UPLOAD_DIR:...} 처럼 이름을 직접 정해 배포와의 약속으로 삼고, yml 이 받는 환경변수 목록이자 기본값 정책 문서가 되게 했다. 기본값은 깜빡했을 때 안전한 쪽이면 두고(H2_CONSOLE_ENABLED:false), 기본값 자체가 위험하면 없애 부팅을 실패시킨다(JWT_SECRET)."
source: session
distilled: false
---

## 배운 개념

### 1. 왜 설정을 밖으로 빼나

- study-board 는 `Dockerfile` 로 jar 를 **한 번** 빌드해 로컬·compose·운영에서 돌린다.
- 값을 코드·yml 에 박으면: ① 값 하나 바꾸려면 **다시 빌드·배포** → "같은 jar 를 모든 환경에서" 전제가 깨지고 테스트한 jar 와 운영 jar 가 달라짐. ② 시크릿이 **git 에 남음** → 저장소를 읽는 누구나 아무 회원의 JWT 를 위조.

### 2. 우선순위: 실행 시점에 가까울수록 이긴다

```
커맨드라인 인자          --app.base-url=...
 > 시스템 프로퍼티       -Dapp.base-url=...
 > OS 환경변수          APP_BASEURL=...
 > 설정 파일: jar 밖 application-{profile}.yml
            > jar 밖 application.yml
            > jar 안 application-{profile}.yml
            > jar 안 application.yml
```

- jar 안의 yml 은 **기본값**, 배포 환경이 **재빌드 없이** 덮어쓴다 → 1번 문제를 이 순서가 해결.
- study-board: `spring.profiles.default: local` → 프로필 미지정 시 `application-local.yml` 이 켜지고, 그 안의 `jwt.secret` 이 `application.yml` 의 `${JWT_SECRET}` 을 덮어씀. `SPRING_PROFILES_ACTIVE` 로 다른 프로필을 지정하는 순간 로컬 기본값이 사라짐.

### 3. 완화된 바인딩(relaxed binding): 환경변수 → 프로퍼티

```
점(.) → 밑줄(_),  대시(-) → 제거,  전부 대문자

spring.datasource.url → SPRING_DATASOURCE_URL
file.upload-dir       → FILE_UPLOADDIR      ← 사람이 쓰는 FILE_UPLOAD_DIR 과 다르다
app.base-url          → APP_BASEURL
```

- 대시를 밑줄로 쓴 형태도 하위 호환으로 받아 줄 수 있지만 문서화된 규칙은 "대시 제거"다 → 동작 여부를 프레임워크 세부에 맡기게 됨.
- study-board 의 datasource 는 이 방식(이름 미명시). 대신 주석에 "운영에서는 SPRING_DATASOURCE_URL/USERNAME/PASSWORD 환경변수로 덮어쓴다"라고 따로 적어 둠.

### 4. study-board 의 두 방식

```yaml
# ① 환경변수 이름 명시
spring.h2.console.enabled: ${H2_CONSOLE_ENABLED:false}
file.upload-dir:           ${FILE_UPLOAD_DIR:${user.home}/board-uploads/}
app.base-url:              ${APP_BASE_URL:http://localhost:8080}
app.cookie.secure:         ${APP_COOKIE_SECURE:false}
jwt.secret:                ${JWT_SECRET}            # 기본값 없음

# ② 이름 미명시 (완화된 바인딩에 맡김)
spring.datasource.url: jdbc:h2:file:./data/boarddb  # SPRING_DATASOURCE_URL 로 덮어씀
```

- `compose.yaml` 은 두 방식을 섞어 넘김 (`JWT_SECRET`, `APP_BASE_URL` / `SPRING_DATASOURCE_*`). `Dockerfile` 은 `ENV FILE_UPLOAD_DIR=/app/uploads/`.

### 5. 환경변수 이름을 명시한 이유

| 이유 | 설명 |
|---|---|
| ① 이름을 우리가 정한다 | 대시 제거 규칙에 휘둘리지 않음(`FILE_UPLOAD_DIR`). 프로퍼티를 리팩터링해도 compose·Dockerfile·서버의 이름은 그대로 → **배포와의 약속**. 짧은 프로젝트 이름도 가능(`H2_CONSOLE_ENABLED` vs `SPRING_H2_CONSOLE_ENABLED`) |
| ② yml 이 곧 목록 | `${` 만 찾으면 앱이 받는 환경변수가 전부 보임. 미명시 방식은 주석에 따로 써야 하고 주석은 코드와 어긋나기 쉬움 |
| ③ 기본값을 바로 옆에 | `${이름:기본값}` 한 줄에 "무슨 환경변수가, 없으면 무엇으로" |

### 6. 기본값을 줄지 말지의 기준

| 환경변수 | 기본값 | 깜빡하면 | 판단 |
|---|---|---|---|
| `H2_CONSOLE_ENABLED` | `false` | 콘솔 꺼진 채 뜸 | 안전한 기본값 → 둔다 |
| `JWT_SECRET` | 없음 | `Could not resolve placeholder 'JWT_SECRET'` 로 부팅 실패 | 어떤 기본값이든 공개 저장소에 있음 = 위험 → 없앤다 (로컬 편의는 `application-local.yml` 이 맡음) |
| `APP_BASE_URL` | `http://localhost:8080` | 조용히 뜨고 OG 이미지 주소가 전부 localhost | 안전하지만 **티가 안 남** (`SiteMetaAdvice` 주석: 발견 자리가 남의 메신저) → 로컬 전용으로 옮길 **개선 후보** |

> 기준: **깜빡했을 때 기본값이 안전한가**, 그리고 **바로 티가 나는가.** 위험하거나 조용히 틀리면 기본값을 없애 fail fast.

### 면접 답변

> "외부 설정은 실행 시점에 가까울수록 우선합니다. 커맨드라인 인자, 시스템 프로퍼티, 환경변수, 프로필별 yml, 기본 yml 순이라, jar 안의 yml 은 기본값이고 배포 환경이 재빌드 없이 덮어씁니다. 환경변수 이름을 `${이름:기본값}` 으로 명시한 건, 스프링의 완화된 바인딩이 대시를 제거해서 `file.upload-dir` 이 `FILE_UPLOADDIR` 이 되는 것처럼 사람의 직관과 어긋나기 때문입니다. 이름을 직접 정하면 배포 설정과의 약속이 되고, yml 만 봐도 받는 환경변수와 기본값이 다 보입니다. 기본값은 깜빡했을 때 안전한 경우에만 두고, JWT 시크릿처럼 기본값 자체가 위험하면 없애서 부팅을 실패시켰습니다."

## 오답·헷갈린 점

- **전**: 설정을 코드에 박으면 "환경마다 다르게 대처할 수 없다 / 보안에 취약하다" → **후**: 이름으로 — **재빌드·재배포** 필요로 같은 jar 전제가 깨짐, 시크릿이 **git 에 남아** 토큰 위조 가능.
- **전**: "bash 값을 사용한다" → **후**: **OS 환경변수**. 그리고 `-D` 시스템 프로퍼티가 환경변수보다 위.
- **전**: 우선순위만 답하고 "왜 그 순서가 맞나" 누락 → **후**: 실행 시점에 가까울수록 이긴다. jar 는 기본값, 배포가 재빌드 없이 덮어씀.
- **전**: 기본값 기준을 `JWT_SECRET` 하나로만 답함("안 뜨는 게 안전") → **후**: 기준은 "깜빡했을 때 기본값이 안전한가". 그래서 `H2_CONSOLE_ENABLED` 는 두고 `JWT_SECRET` 은 없앰. 여기에 "티가 나는가"를 더하면 `APP_BASE_URL` 도 검토 대상.
- (인터뷰어 정정) 예시에서 `app.base-url` 을 `APP_BASE_URL` 로 덮어쓴다고 당연하게 제시 → 문서 규칙상 이름은 `APP_BASEURL`. 이 혼동 자체가 이름을 명시하는 이유.
- 👍 개선: 우선순위를 처음부터 거의 정확히, 완화된 바인딩 규칙을 처음 보고 바로 정확히 적용(`SPRING_DATASOURCE_URL`, `FILE_UPLOADDIR`). 답이 짧고 정확해짐.
- ⚠️ 반복 테마: **이유·기준 질문에서 예시 하나로 답을 끝냄.** 기준 질문엔 "A 는 이래서 이렇고, B 는 이래서 이렇다"로 두 경우를 대비시킬 것.

## Q&A

- **Q. 설정을 왜 외부로 빼나?** A. 같은 jar 를 모든 환경에서 재빌드 없이 쓰고, 시크릿을 저장소 밖에 두려고.
- **Q. 외부 설정 우선순위는?** A. 커맨드라인 > 시스템 프로퍼티 > 환경변수 > (jar 밖 > jar 안) 프로필 yml > 기본 yml. 실행 시점에 가까울수록 이긴다.
- **Q. 환경변수는 어떻게 프로퍼티에 연결되나?** A. 완화된 바인딩. 점→밑줄, 대시 제거, 대문자. `file.upload-dir` → `FILE_UPLOADDIR`.
- **Q. 환경변수 이름을 왜 명시했나?** A. 바인딩 규칙과 직관의 틈을 없애 배포와의 약속으로 삼고, yml 이 환경변수 목록·기본값 문서가 되게.
- **Q. 왜 JWT_SECRET 만 기본값이 없나?** A. 기본값이 공개 저장소에 있어 그 자체로 위험. 없애서 빠뜨리면 부팅이 실패하게. H2 콘솔은 기본값 false 가 안전하므로 둔다.

## 다음 학습

- `@ConfigurationProperties` vs `@Value` 의 완화된 바인딩 지원 차이, `@ConfigurationProperties` + `@Validated` 로 필수 설정을 부팅 시 검증
- `spring.config.import`, `optional:`, 설정 파일 위치(`./config/`)와 jar 밖 설정 파일 운영
- 시크릿을 환경변수 대신 Vault·AWS Secrets Manager·k8s Secret 으로 관리할 때의 차이, 환경변수의 노출 경로(`/proc`, `docker inspect`, 로그)
- study-board 개선 후보: `APP_BASE_URL`, `APP_COOKIE_SECURE` 기본값을 `application-local.yml` 로 옮겨 로컬 밖에서 필수화
- 연결: [[2026-10-19-conditional-on-property-h2-console]], [[token-storage-httponly-cookie]], [[jvm-container-heap-sizing]]
