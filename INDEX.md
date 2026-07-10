# INDEX — 02_Refined 개념노트 색인

## 02_Refined

- [기능별 패키지 구성](02_Refined/package-by-feature.md) — package-by-feature vs 레이어별, 응집도, "같이 바뀌는 것은 같이"
- [Spring 애플리케이션 이벤트](02_Refined/spring-application-events.md) — ApplicationEventPublisher/@EventListener, 모듈 결합 차단, 명령 vs 사실 통보, OCP
- [SSE (Server-Sent Events)](02_Refined/sse-server-sent-events.md) — SseEmitter, 연결 유지 스트리밍, 스레드 반납, ConcurrentHashMap, 메모리 누수, 스케일 아웃 pub/sub
- [@TransactionalEventListener](02_Refined/transactional-event-listener.md) — TransactionPhase, AFTER_COMMIT 조용한 유실 함정, REQUIRES_NEW, 원자성 상실
- [Transactional Outbox 패턴](02_Refined/transactional-outbox-pattern.md) — outbox 테이블, 반드시 전달, 릴레이 폴링/CDC, at-least-once·멱등성
