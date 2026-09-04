# INDEX — 02_Refined 개념노트 색인

## 02_Refined

- [기능별 패키지 구성](02_Refined/package-by-feature.md) — package-by-feature vs 레이어별, 응집도, "같이 바뀌는 것은 같이"
- [Spring 애플리케이션 이벤트](02_Refined/spring-application-events.md) — ApplicationEventPublisher/@EventListener, 모듈 결합 차단, 명령 vs 사실 통보, OCP
- [SSE (Server-Sent Events)](02_Refined/sse-server-sent-events.md) — SseEmitter, 연결 유지 스트리밍, 스레드 반납, ConcurrentHashMap, 메모리 누수, 스케일 아웃 pub/sub
- [@TransactionalEventListener](02_Refined/transactional-event-listener.md) — TransactionPhase, AFTER_COMMIT 조용한 유실 함정, REQUIRES_NEW, 원자성 상실
- [Transactional Outbox 패턴](02_Refined/transactional-outbox-pattern.md) — outbox 테이블, 반드시 전달, 릴레이 폴링/CDC, at-least-once·멱등성
- [Open Session In View (open-in-view)](02_Refined/open-in-view-osiv.md) — OSIV, 영속성 컨텍스트 생명주기, 커넥션 점유 ③~⑦, 풀 고갈, 의도치 않은 UPDATE
- [지연 로딩과 프록시](02_Refined/lazy-loading-proxy.md) — LAZY/EAGER, 프록시 초기화, LazyInitializationException "no Session", fetch 기본값
- [N+1 문제와 fetch join](02_Refined/n-plus-1-problem.md) — 1+N 쿼리, join fetch, @EntityGraph, default_batch_fetch_size, 컬렉션 페이징 불가
- [변경 감지 (Dirty Checking)](02_Refined/dirty-checking.md) — 스냅샷, flush 시점, 영속 vs 준영속, readOnly 최적화, 쓰기 지연
- [DB 수준 페이징과 Page vs Slice](02_Refined/db-pagination.md) — LIMIT/OFFSET, ORDER BY 전제조건, count 쿼리 비용, Slice limit+1
- [키셋(커서) 페이징](02_Refined/keyset-pagination.md) — OFFSET 성능 절벽, WHERE 마지막값, 무한 스크롤, 임의 페이지 점프 포기
- [LIKE '%kw%' 와 B-tree 인덱스](02_Refined/like-search-btree-index.md) — 앞 와일드카드 풀 스캔, 탐색 시작점, Containing vs StartingWith, full-text
- [리포지토리 동적 프록시](02_Refined/spring-data-repository-proxy.md) — 구현 없는 인터페이스, 구동 시 1회 생성 싱글턴, SimpleJpaRepository, @Transactional AOP, self-invocation
- [파생 쿼리 메서드](02_Refined/derived-query-method.md) — 메서드 이름이 쿼리, 엔티티 필드명, 구동 시점 파싱 fail-fast, PropertyReferenceException
- [HandlerMethodArgumentResolver](02_Refined/handler-method-argument-resolver.md) — 컨트롤러 파라미터 바인딩, PageableHandlerMethodArgumentResolver, 0-based page, 커스텀 리졸버
- [JVM 실행 파이프라인과 WORA](02_Refined/jvm-execution-pipeline.md) — javac→바이트코드→JVM, 플랫폼 독립성, 클래스로더·런타임데이터영역·실행엔진
- [인터프리터와 JIT 컴파일러](02_Refined/interpreter-and-jit.md) — 하이브리드 전략, 호출 카운터·임계값, 핫스팟, 워밍업, 역최적화
- [JVM 메모리 구조 — 스택과 힙](02_Refined/jvm-stack-and-heap.md) — 스택은 스레드별·힙은 공유 1개, 지역변수 스레드 안전, 참조 변수, StackOverflowError
- [GC 와 도달 가능성](02_Refined/garbage-collection-reachability.md) — GC Root, reachability, OOM 이 나는 이유, 메모리 누수 정의, Stop-the-World, 세대별은 별개 축
- [비밀값 해시 알고리즘 선택 기준](02_Refined/secret-hashing-algorithm-choice.md) — BCrypt vs SHA-256, 비밀번호 저장, 리프레시 토큰 저장, 왜 알고리즘이 다른가, 해시 선택 기준
- [비밀값의 엔트로피와 탐색 공간](02_Refined/secret-entropy-and-search-space.md) — 엔트로피, 탐색 공간, 무차별 대입, 비트는 자릿수, SecureRandom, 사전 공격
- [결정적 해시와 DB 조회](02_Refined/deterministic-hash-for-lookup.md) — salt 랜덤성, 인덱스를 못 타는 이유, 토큰 갱신 조회, 전체 스캔, 찾기와 검증의 분리
- [오프라인 공격 vs 온라인 공격](02_Refined/offline-vs-online-attack.md) — 레이트 리밋, 초대 코드, 인증번호, 시도 횟수 통제, 느린 해시의 DoS 역효과
