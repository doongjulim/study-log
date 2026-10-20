---
date: 2026-10-19
type: raw
author: dongju
tags: [ConditionalOnProperty, 조건부빈, conditional-bean, Conditional, Profile, 프로필, spring-profile, 빈등록, bean-registration, 빈정의, bean-definition, 부팅시점, startup, 재시작, H2, H2콘솔, h2-console, spring.h2.console.enabled, SecurityFilterChain, securityMatcher, PathRequest.toH2Console, 다중필터체인, multiple-filter-chains, Order, CSRF, 클릭재킹, clickjacking, frameOptions, sameOrigin, X-Frame-Options, permitAll, 인증면제, 보안예외, 수명일치, 스위치하나, 기본값false, secure-by-default, 환경변수, ObjectProvider, jwt.secret, 잊으면뜨지않는구조, study-board, 면접답변]
topic: "@ConditionalOnProperty 로 H2 콘솔 보안 예외 체인을 콘솔과 같은 프로퍼티에 묶은 이유 — 스위치를 하나로 만들어 콘솔과 예외의 수명을 같게 하고, @Profile 이면 두 스위치가 어긋난다"
summary: "@ConditionalOnProperty 는 앱이 뜰 때 한 번 프로퍼티를 보고 빈을 등록할지 말지를 정한다(조건이 거짓이면 @Bean 메서드가 호출조차 안 됨). study-board 는 CSRF 해제·프레임 허용·인증 면제를 푸는 H2 콘솔 전용 SecurityFilterChain 을 콘솔을 켜는 바로 그 프로퍼티(spring.h2.console.enabled)에 묶었다. @Profile(\"local\") 로 묶으면 콘솔은 프로퍼티가, 체인은 프로필이 결정해서 콘솔 없이 인증 면제 규칙만 남거나, 콘솔은 켰는데 메인 체인에 막혀 쓸 수 없는 어긋남이 생긴다. 운영 안전은 이 어노테이션이 아니라 기본값 false 가 맡는다."
source: session
distilled: false
---

## 배운 개념

### 1. 무엇을, 언제 결정하나

- 프로퍼티 값을 보고 **이 빈을 컨테이너에 등록할지 말지**를 정한다. 켜고 끄는 게 아니라 **있느냐 없느냐**. 조건이 거짓이면 `@Bean` 메서드는 **호출조차 안 되고** 빈이 존재하지 않는다.
- 결정 시점은 **앱이 뜰 때 한 번**, 빈 정의를 등록하는 단계(빈 생성 ①보다도 앞). 요청마다가 아니다(싱글톤 빈을 요청마다 만들 수 없다). 운영 중 값을 바꿔도 **재시작 전엔 그대로**.

### 2. study-board 에서 쓴 곳 (유일한 사용처)

```java
/**
 * H2 콘솔 전용 체인.
 * 콘솔은 DB 를 브라우저에서 그대로 열 수 있는 창구라, 운영에 열려 있으면 그것으로 끝이다.
 * 그래서 "콘솔이 켜져 있을 때만" 이 체인이 만들어지도록 묶었다 -
 * 예외(csrf 무시·프레임 허용·인증 면제)와 콘솔의 수명이 같아져,
 * 콘솔을 끄면 뚫어 둔 구멍도 함께 사라진다.
 */
@Bean
@Order(0)
@ConditionalOnProperty(name = "spring.h2.console.enabled", havingValue = "true")
public SecurityFilterChain h2ConsoleFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher(PathRequest.toH2Console())                   // 적용 대상: /h2-console/**
        .csrf(AbstractHttpConfigurer::disable)                        // ① CSRF 방어 해제 (다른 사이트발 요청 못 막음)
        .headers(h -> h.frameOptions(f -> f.sameOrigin()))            // ② 클릭재킹 방어 완화 (콘솔이 프레임을 씀)
        .authorizeHttpRequests(a -> a.anyRequest().permitAll());      // ③ 인증 면제 (익명 접근 허용)
    return http.build();
}
```
```yaml
spring:
  h2:
    console:
      # 켜져 있으면 DB 를 브라우저에서 그대로 열 수 있으므로,
      # 운영에서 끄는 것을 "잊지 않아야" 하는 구조는 언젠가 잊는다.
      enabled: ${H2_CONSOLE_ENABLED:false}
      path: /h2-console
```

- `@Order(0)` + `securityMatcher` 로 콘솔 경로만 이 체인이 먼저 가져가고, 나머지는 메인 체인(`@Order(1)`).
- `PathRequest.toH2Console()` 는 설정의 콘솔 경로를 따라가므로 경로를 바꿔도 예외 범위가 같이 움직인다.

### 3. 왜 `@Profile("local")` 이 아닌가: 스위치의 개수

비유: 콘솔 = DB 로 가는 **뒷문**, 체인 = 그 문 앞의 "검사 없이 통과" **경비원 지시서**. 문이 있을 때만 지시서도 있어야 한다.

| | 콘솔을 켜는 스위치 | 체인을 켜는 스위치 | 어긋날 수 있나 |
|---|---|---|---|
| `@Profile("local")` | 프로퍼티 | **프로필** | **있음** (스위치 2개) |
| `@ConditionalOnProperty` ✅ | 프로퍼티 | **같은 프로퍼티** | **없음** (스위치 1개) |

`@Profile` 일 때 어긋나는 두 경우:

| 상황 | 콘솔 | 체인 | 결과 |
|---|---|---|---|
| `local`, 콘솔 꺼짐(기본값) | 없음 | **있음** | `/h2-console/**` 에 CSRF 해제·프레임 허용·인증 면제 규칙만 남음. 경로가 다시 열리면 구멍이 이미 뚫려 있음. 기본값이 `false` 라 **로컬에서조차 대부분 이 상태** |
| `local` 아님, 콘솔 켬 | 있음 | **없음** | 콘솔 요청이 메인 체인으로 → 인증 요구로 로그인 페이지, 프레임 `DENY` 로 빈 화면, CSRF 토큰 없어 403. **켰는데 쓸 수 없음** |

→ 주석의 "**예외와 콘솔의 수명을 같게**" = 스위치를 하나로.

### 4. 한계: 운영을 지키는 건 기본값

- 운영에 `H2_CONSOLE_ENABLED=true` 가 들어가면 콘솔과 "검사 없이 통과" 체인이 **같이** 켜진다. 이 어노테이션의 역할은 둘을 **한 몸으로 묶는 것**까지.
- 운영 안전은 **기본값 `false`**(secure by default)가 맡는다.

### 5. study-board 의 같은 철학의 설계들

| 설계 | 원리 |
|---|---|
| H2 체인 `@ConditionalOnProperty` | 기능과 그 기능의 보안 예외를 **같은 스위치**로 |
| `jwt.secret` 기본값은 `application-local.yml` 에만 | 다른 프로필로 띄우면 기본값이 사라져 환경변수를 빠뜨린 배포는 **부팅 실패**. "잊지 않아야 하는 구조 대신 잊으면 뜨지 않는 구조" |
| OAuth 설정을 `oauth` 프로필로 분리 + `ObjectProvider<ClientRegistrationRepository>` | 자격증명이 없으면 `oauth2Login` 자체를 안 붙여 **설정 없이도 앱이 뜬다** ("설정을 잊으면 부팅이 막힌다"는 구조를 만들지 않음) |

세 가지 모두 **"사람이 기억해야 하는 규칙"을 "구조가 강제하는 규칙"으로** 바꾼 것. 조건부 빈을 고를 때 물을 것: **"이 빈은 무엇과 수명이 같아야 하나?"** 환경과 같아야 하면 `@Profile`, 특정 기능과 같아야 하면 그 기능을 켜는 프로퍼티.

### 면접 답변

> "H2 콘솔 자체가 `spring.h2.console.enabled` 로 켜지기 때문에, 보안 예외 체인도 같은 프로퍼티를 조건으로 삼아 스위치를 하나로 만들었습니다. `@Profile` 로 묶으면 콘솔은 프로퍼티가, 체인은 프로필이 결정해서 둘이 어긋날 수 있습니다. 콘솔은 꺼졌는데 인증 면제 규칙만 남거나, 콘솔은 켰는데 메인 체인에 막혀 쓸 수 없게 되죠. 운영에서의 안전은 이 어노테이션이 아니라 기본값 `false` 가 맡습니다."

## 오답·헷갈린 점

- **전**: "yaml 에 적어 둔 설정값에 따라서 결정된다" → **후**: 그건 결정의 **근거(입력)**. 결정하는 건 **빈을 등록할지 말지**.
- **전**: "그 결정은 요청이 들어올 때마다 정한다" → **후**: **앱이 뜰 때 한 번.** 조건이 거짓이면 빈이 아예 없고, 값을 바꾸려면 재시작.
- **전**: 체인이 항상 등록되면 "보안에 취약한 상태" → **후**: 무엇이 취약한지 이름으로 — CSRF 방어 해제, 클릭재킹 방어 완화, 인증 면제. (두 번째 시도에서 이름으로 답함 👍)
- **전**: `@Profile` 비교에서 "1. false 로 적용됨 2. 적용 안 됨" → **후**: 콘솔과 체인을 **따로** 봐야 함. 둘을 결정하는 스위치가 다르다는 게 핵심. (여기서 막혀 설명을 요청함)
- **전**: 최종 답변 "앱이 서버에 올라갈 때 한 번만 설정을 읽고 콘솔과 체인이 함께 켜지고 꺼지기 위해" → **후**: 앞부분은 `@Profile` 도 같아 **차별점이 아님.** "왜 A 가 아니라 B" 질문엔 **A 로 하면 무엇이 깨지는지**(스위치 2개 → 어긋남)까지 말해야 답이 됨.
- ⚠️ 반복 테마: **질문 뒷부분 건너뛰기**(Q1-② "언제"를 두 번 건너뜀), **비교 질문에서 비교 대상 누락.** 👍 모를 때 버티지 않고 바로 설명을 요청한 것.

## Q&A

- **Q. `@ConditionalOnProperty` 는 무엇을 언제 결정하나?** A. 프로퍼티를 보고 빈을 등록할지 말지를, 앱이 뜰 때 한 번 결정한다.
- **Q. 운영 중에 프로퍼티를 바꾸면?** A. 재시작 전까지 반영되지 않는다.
- **Q. H2 콘솔 체인이 항상 등록되면?** A. 콘솔이 꺼져 있어도 `/h2-console/**` 에 CSRF 해제·프레임 허용·인증 면제 규칙이 남는다.
- **Q. 왜 `@Profile("local")` 이 아닌가?** A. 콘솔은 프로퍼티가 켜므로 체인을 프로필로 묶으면 스위치가 둘이 되어 어긋난다. 같은 프로퍼티에 묶어 수명을 같게 했다.
- **Q. 그럼 운영은 이 어노테이션이 지켜 주나?** A. 아니다. 운영에서 켜면 같이 켜진다. 안전은 기본값 `false` 가 맡는다.

## 다음 학습

- `@ConditionalOnProperty` 의 `matchIfMissing`, `havingValue` 생략 시 동작 ("false" 가 아닌 모든 값이 참)
- `@Conditional` 계열(`@ConditionalOnBean`, `@ConditionalOnMissingBean`, `@ConditionalOnClass`)과 Spring Boot 자동 설정이 이를 어떻게 쓰나 — H2 콘솔 자체도 자동 설정
- 다중 `SecurityFilterChain` 선택 규칙: `@Order` 와 `securityMatcher`, 매처 없는 체인이 앞에 오면 생기는 문제
- 클릭재킹과 `X-Frame-Options` / CSP `frame-ancestors`
- 연결: [[2026-10-19-filter-interceptor-aop-auth]], [[2026-10-19-constructor-vs-field-injection]], [[singleton-bean-stateful]], [[token-storage-httponly-cookie]]
