---
type: refined
slug: encoding-encryption-hashing
tags: [인코딩, encoding, 암호화, encryption, 복호화, decryption, 해시, hash, hashing, 단방향, one-way, 양방향, 가역성, reversibility, Base64, Base64Url, 기밀성, confidentiality, 무결성, integrity, 비밀번호저장, password-storage, bcrypt, 키, key, 대칭키, JWT, 면접답변]
topic: "인코딩·암호화·해시는 수단이 아니라 '되돌릴 수 있느냐, 누가 되돌리느냐'로 가른다"
summary: "인코딩은 누구나 되돌리는 형식 변환(키 없음), 암호화는 키가 있는 사람만 되돌리는 기밀성 장치, 해시는 아무도 되돌리지 못하는 단방향 변환이다. 비밀번호를 암호화로 저장하면 DB 와 키가 함께 털렸을 때 전부 복호화되므로, 원문이 다시 필요 없는 비밀번호는 해시로 저장하고 비교한다."
contributors: [dongju]
source_refs:
  - https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
  - https://datatracker.ietf.org/doc/html/rfc4648
updated: 2026-10-22
---

# 인코딩 · 암호화 · 해시

## 개념 정의

| | 되돌릴 수 있나 | 키 필요 | 목적 | 예 |
|---|---|---|---|---|
| **인코딩** | ⭕ **누구나** | ❌ | 형식 변환 (전송·표현) | Base64, Base64Url(JWT payload), URL 인코딩 |
| **암호화** | ⭕ **키가 있으면** | ⭕ | 기밀성 | AES, RSA |
| **해시** | ❌ **아무도** (단방향) | ❌ | 비교·무결성 | SHA-256, bcrypt |

가르는 축은 수단(키를 쓰나, 어떤 수식인가)이 아니라 **되돌릴 수 있느냐, 누가 되돌리느냐**다.

## 작동 원리 — 비밀번호는 왜 해시인가

- 비밀번호는 서버가 **원문을 다시 알 필요가 없다**. 로그인 때 입력값을 같은 방식으로 해시해 **저장된 해시와 비교**하면 된다.
- 암호화로 저장하면 **DB 와 키가 함께 털렸을 때 전부 복호화**된다. 키를 아는 내부자도 원문을 볼 수 있다.
- 해시는 되돌릴 방법이 없으므로, 유출돼도 공격자에게 남는 길은 **후보를 하나씩 해시해 맞춰 보는 대입**뿐이다 → 그 비용을 올리는 것이 bcrypt 의 느림이다 ([[offline-vs-online-attack]], [[secret-hashing-algorithm-choice]]).

## 트레이드오프 / 한계

- 해시는 되돌릴 수 없어서 **원문이 다시 필요한 값**(외부 API 키를 꺼내 써야 하는 경우 등)에는 못 쓴다 → 암호화.
- 인코딩은 보안 장치가 **아니다**. Base64 로 감싼 값은 숨겨진 게 아니다 ([[jwt-structure-signature]]).

> [!WARNING]
> **오답 코너**
> - **"비밀번호를 암호화해서 비교한다"** — **해시**다. 핵심 차이는 되돌릴 수 있느냐. 암호화 저장은 DB + 키 유출 시 전부 복호화된다.
> - **"JWT payload 는 암호화되어 있다"** — **인코딩**이다. 키 없이 누구나 되돌린다.

## 면접 답변

> "인코딩은 누구나 되돌릴 수 있는 형식 변환이고, 암호화는 키가 있는 사람만 되돌릴 수 있고, 해시는 누구도 되돌릴 수 없는 단방향 변환입니다. 비밀번호는 서버가 원문을 다시 알 필요가 없으니 해시로 저장하고 비교합니다. 암호화로 저장하면 DB 와 키가 같이 유출됐을 때 전부 복호화되기 때문입니다."

## 복습 체크

- [ ] 셋을 "되돌릴 수 있나 / 키가 필요한가" 두 축으로 구분할 수 있는가?
- [ ] 비밀번호를 암호화가 아니라 해시로 저장하는 이유는?
- [ ] 해시를 쓸 수 없는 값은 어떤 성질을 가진 값인가?

## 관련

[[jwt-structure-signature]] · [[secret-hashing-algorithm-choice]] · [[offline-vs-online-attack]] · [[deterministic-hash-for-lookup]]
