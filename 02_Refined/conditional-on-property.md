---
type: refined
slug: conditional-on-property
tags: [ConditionalOnProperty, 조건부빈, conditional-bean, Conditional, Profile, 프로필, spring-profile, 빈등록, bean-registration, 빈정의, bean-definition, 부팅시점, startup, 재시작, H2, H2콘솔, h2-console, spring.h2.console.enabled, SecurityFilterChain, securityMatcher, PathRequest.toH2Console, 다중필터체인, multiple-filter-chains, CSRF, 클릭재킹, clickjacking, frameOptions, sameOrigin, permitAll, 인증면제, 보안예외, 수명일치, 스위치하나, 안전한기본값, secure-by-default, ObjectProvider, 면접답변]
topic: "@ConditionalOnProperty 로 기능의 보안 예외를 그 기능을 켜는 프로퍼티에 묶는다 — 스위치를 하나로 만들어 수명을 같게 하고, @Profile 이면 두 스위치가 어긋난다"
summary: "@ConditionalOnProperty 는 앱이 뜰 때 한 번 프로퍼티를 보고 빈을 등록할지 말지를 정한다(거짓이면 @Bean 메서드가 호출조차 안 됨). H2 콘솔처럼 CSRF 해제·프레임 허용·인증 면제를 푸는 전용 SecurityFilterChain 은 콘솔을 켜는 바로 그 프로퍼티(spring.h2.console.enabled)에 묶어야 한다. @Profile 로 묶으면 콘솔은 프로퍼티가, 체인은 프로필이 결정해 콘솔 없이 인증 면제 규칙만 남거나 콘솔은 켰는데 메인 체인에 막혀 쓸 수 없는 어긋남이 생긴다. 운영 안전은 어노테이션이 아니라 안전한 기본값(false)이 맡는다."
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html#features.developing-auto-configuration.condition-annotations.property-conditions
  - https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.h2-web-console
updated: 2026-10-20
---

# @ConditionalOnProperty

## 개념 정의

- 프로퍼티 값을 보고 **이 빈을 컨테이너에 등록할지 말지**를 정한다. 켜고 끄는 게 아니라 **있느냐 없느냐**. 조건이 거짓이면 `@Bean` 메서드는 **호출조차 안 되고** 빈이 존재하지 않는다.
- 결정 시점은 **앱이 뜰 때 한 번**, 빈 정의를 등록하는 단계(인스턴스 생성보다도 앞). 요청마다가 아니다 — 싱글톤 빈을 요청마다 만들 수는 없다 ([[singleton-bean-stateful]]). 운영 중 값을 바꿔도 **재시작 전엔 그대로**.

## 작동 원리 — H2 콘솔 보안 체인

```java
@Bean
@Order(0)
@ConditionalOnProperty(name = "spring.h2.console.enabled", havingValue = "true")
public SecurityFilterChain h2ConsoleFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher(PathRequest.toH2Console())                   // 적용 대상: 콘솔 경로만
        .csrf(AbstractHttpConfigurer::disable)                        // CSRF 방어 해제
        .headers(h -> h.frameOptions(f -> f.sameOrigin()))            // 클릭재킹 방어 완화 (콘솔이 프레임을 씀)
        .authorizeHttpRequests(a -> a.anyRequest().permitAll());      // 인증 면제 (익명 접근)
    return http.build();
}
```
```yaml
spring.h2.console.enabled: ${H2_CONSOLE_ENABLED:false}
```

- `@Order(0)` + `securityMatcher` 로 콘솔 경로만 이 체인이 먼저 가져가고, 나머지는 메인 체인으로.
- `PathRequest.toH2Console()` 는 설정의 콘솔 경로를 따라가므로 경로를 바꿔도 예외 범위가 같이 움직인다.
- 이 체인이 **항상** 등록되면, 콘솔을 꺼 둔 환경에도 콘솔 경로에 세 가지 보안 해제 규칙이 남는다. 경로가 다시 열리면 구멍이 이미 뚫려 있는 상태.

### 왜 `@Profile("local")` 이 아닌가 — 스위치의 개수

비유: 콘솔 = DB 로 가는 **뒷문**, 체인 = 그 문 앞의 "검사 없이 통과" **지시서**. 문이 있을 때만 지시서도 있어야 한다.

| | 콘솔을 켜는 스위치 | 체인을 켜는 스위치 | 어긋날 수 있나 |
|---|---|---|---|
| `@Profile("local")` | 프로퍼티 | **프로필** | **있음** (스위치 2개) |
| `@ConditionalOnProperty` ✅ | 프로퍼티 | **같은 프로퍼티** | **없음** (스위치 1개) |

`@Profile` 일 때 어긋나는 두 경우:

| 상황 | 콘솔 | 체인 | 결과 |
|---|---|---|---|
| `local`, 콘솔 꺼짐(기본값) | 없음 | **있음** | 콘솔 경로에 해제 규칙만 남음. 기본값이 `false` 라 **로컬에서조차 대부분 이 상태** |
| `local` 아님, 콘솔 켬 | 있음 | **없음** | 콘솔 요청이 메인 체인으로 → 인증 요구로 로그인 페이지, 프레임 `DENY` 로 빈 화면, CSRF 로 403. **켰는데 쓸 수 없음** |

→ **기능과 그 기능의 보안 예외를 같은 스위치로** 묶어 수명을 같게 한다.

## 트레이드오프 / 한계

- 운영에 `H2_CONSOLE_ENABLED=true` 가 들어가면 콘솔과 해제 체인이 **같이** 켜진다. 이 어노테이션은 둘을 **한 몸으로 묶을 뿐**, 운영을 지켜 주지 않는다. 운영 안전은 **안전한 기본값 `false`** 가 맡는다 ([[externalized-configuration]]).
- 고르는 기준: **"이 빈은 무엇과 수명이 같아야 하나?"** 환경(로컬/운영)과 같아야 하면 `@Profile`, 특정 기능과 같아야 하면 그 기능을 켜는 **프로퍼티**.
- 같은 철학의 다른 수단: 설정이 없을 때 기능을 아예 안 붙이도록 `ObjectProvider` 로 선택적 의존을 받는 방식(예: OAuth 자격증명이 없으면 `oauth2Login` 생략). 공통점은 **사람이 기억해야 하는 규칙을 구조가 강제하는 규칙으로** 바꾸는 것.

> [!WARNING]
> **오답 코너**
> - **"yml 설정값에 따라 결정된다"** → 그건 결정의 **근거**. 결정하는 건 **빈을 등록할지 말지**.
> - **"요청이 들어올 때마다 정한다"** → **앱이 뜰 때 한 번.** 값을 바꾸려면 재시작.
> - **"항상 등록하면 보안에 취약하다"** → 무엇이 취약한지 이름으로: CSRF 방어 해제, 클릭재킹 방어 완화, 인증 면제.
> - **"@Profile 이 아닌 이유 = 앱이 뜰 때 한 번 읽어서"** → `@Profile` 도 마찬가지라 차별점이 아니다. 진짜 이유는 **스위치가 하나**라는 것.

## 복습 체크

- [ ] `@ConditionalOnProperty` 가 무엇을, 언제 결정하는지 말할 수 있다
- [ ] H2 콘솔 체인이 푸는 세 가지 보안 보호를 이름으로 말할 수 있다
- [ ] `@Profile("local")` 로 바꿨을 때 어긋나는 두 경우를 콘솔/체인 표로 그릴 수 있다
- [ ] 이 어노테이션이 운영을 지켜 주지 않는 이유와 실제로 지키는 장치를 말할 수 있다
- [ ] `@Profile` 과 `@ConditionalOnProperty` 중 무엇을 고를지의 기준을 한 문장으로 말할 수 있다

## 관련

[[externalized-configuration]] · [[filter-interceptor-aop]] · [[singleton-bean-stateful]] · [[token-storage-httponly-cookie]] · [[csrf-conditions-and-defenses]]
