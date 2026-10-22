---
type: refined
slug: jwt-structure-signature
tags: [JWT, json-web-token, 제이더블유티, 토큰구조, header, payload, signature, 서명, 시그니처, HMAC, HS256, secret, 비밀키, Base64Url, 인코딩, encoding, 위조방지, tamper-proof, 무결성, integrity, 열람, 기밀성, confidentiality, 클레임, claim, sub, exp, role, 민감정보, jwt.io, 재계산, 면접답변]
topic: "JWT 서명은 위조를 막지만 열람은 막지 못한다 — payload 는 암호화가 아니라 Base64Url 인코딩이다"
summary: "JWT 는 header.payload.signature 이고 signature 는 HMAC(header.payload, 서버만 아는 secret) 이다. 서버는 받은 header·payload 로 서명을 다시 계산해 붙어 온 값과 비교하므로, payload 를 고친 공격자는 secret 이 없어 맞는 서명을 만들 수 없다. 반면 payload 는 Base64Url 인코딩일 뿐이라 누구나 디코드해 읽는다. 그래서 식별자(sub)·권한·만료는 넣되 비밀번호·연락처 같은 민감정보는 넣지 않는다."
contributors: [dongju]
source_refs:
  - https://datatracker.ietf.org/doc/html/rfc7519
  - https://datatracker.ietf.org/doc/html/rfc7515
updated: 2026-10-22
---

# JWT 구조와 서명 — 위조는 막고, 열람은 못 막는다

## 개념 정의

```
header.payload.signature

header    = base64url({"alg":"HS256","typ":"JWT"})
payload   = base64url({"sub":"42","role":"USER","exp":1761100000})
signature = HMAC-SHA256( header + "." + payload, 서버만 아는 secret )
```

## 작동 원리 — 서버는 서명을 "다시 계산"한다

1. 공격자가 payload 를 `role: ADMIN` 으로 고친다.
2. 서버는 받은 header·payload 로 **secret 을 써서 서명을 재계산**한다.
3. 재계산 값 ≠ 붙어 온 signature → 거부.
4. 공격자가 서명까지 새로 만들려면 **secret 이 필요**한데, 없다.

"시그니처 값을 비교한다"만으로는 반쪽이다. 핵심은 **서버만 아는 secret 으로 재계산**한다는 점이다.

## 서명이 막지 못하는 것 — 열람

- payload 는 **Base64Url 인코딩**일 뿐이다. 키 없이 누구나 디코드한다 (jwt.io 에 붙여 넣으면 보인다).
- 서명은 **무결성(위조 방지)** 을 주지, **기밀성(숨김)** 을 주지 않는다 ([[encoding-encryption-hashing]]).

| 넣어도 되는 것 | 넣으면 안 되는 것 |
|---|---|
| 식별자(`sub` — 없으면 토큰이 누구 것인지 몰라 기능을 못 함), 권한, 만료(`exp`) | 비밀번호, 주민번호, 연락처 등 **민감정보** |

판단 기준은 **"남에게 보여도 되는가"**.

## 트레이드오프 / 한계

- 서명이 유효하면 서버는 저장소 조회 없이 신뢰한다 → **발급 후 즉시 무효화가 어렵다**. 짧은 만료 + 리프레시 토큰으로 보완 ([[refresh-token-rotation]], [[session-vs-token-auth]]).
- HS256 은 대칭키라 **secret 이 곧 위조 능력**이다. 짧거나 추측 가능한 secret 은 오프라인 대입으로 뚫린다 ([[secret-entropy-and-search-space]]). secret 미설정 시 기동 실패(fail fast) 처리 ([[externalized-configuration]]).

> [!WARNING]
> **오답 코너**
> - **"JWT payload 는 암호화되어 있다"** — **Base64Url 인코딩**이다. 누구나 읽는다.
> - **"위조 방지 = 시그니처 값을 비교"** — 서버가 `HMAC(header.payload, secret)` 을 **재계산**해 비교한다. 공격자는 secret 이 없어 새 서명을 못 만든다.
> - **"payload 에 식별 가능한 값을 넣으면 안 된다"** — 식별자(`sub`)는 **필요**하다. 빼야 하는 건 **민감정보**.

## 면접 답변

> "JWT 는 header, payload, signature 세 부분이고, 서명은 header 와 payload 를 서버만 아는 secret 으로 HMAC 한 값입니다. 사용자가 payload 를 고치면 서버가 서명을 다시 계산했을 때 맞지 않아 거부되고, 공격자는 secret 이 없어서 맞는 서명을 만들 수 없습니다. 다만 payload 는 암호화가 아니라 Base64Url 인코딩이라 누구나 읽을 수 있어서, 사용자 ID 나 권한은 넣지만 민감정보는 넣지 않습니다."

## 복습 체크

- [ ] 사용자가 payload 의 role 을 바꾸면 서버에서 정확히 어떤 일이 일어나는가?
- [ ] 서명이 주는 것(무결성)과 주지 않는 것(기밀성)을 구분할 수 있는가?
- [ ] payload 에 넣어도 되는 것과 안 되는 것의 판단 기준은?
- [ ] JWT 를 즉시 무효화하기 어려운 이유와 보완책은?

## 관련

[[session-vs-token-auth]] · [[encoding-encryption-hashing]] · [[refresh-token-rotation]] · [[secret-entropy-and-search-space]] · [[token-storage-httponly-cookie]]
