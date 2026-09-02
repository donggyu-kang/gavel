# CLAUDE.md — Gavel 프로젝트 작업 지침

## 프로젝트 개요
Gavel = 실시간 경매 플랫폼. Spring Boot 백엔드. **포트폴리오 프로젝트** — 프로덕션 배포보다 백엔드 역량 시연이 목적.

핵심 학습 목표: 동시성 제어, 실시간 기능(WebSocket), Redis 캐싱, Elasticsearch 검색. 부수 목표: 면접 대비 설명 능력, 기술 블로그 소재 확보.

프론트엔드 협업자와 함께 진행하나 **백엔드는 단독으로 완결되는 구조** 유지.

## 구현 파이프라인 (합의된 순서)
회원 인증 → 상품/경매 등록 → 동시성 안전한 입찰 → 실시간 반영 → 캐싱/비동기 처리(JMeter 부하 테스트, 캐싱 전/후 성능 비교) → 검색 최적화

## 기술 스택
- Backend: Spring Boot, Java 21, Gradle-Groovy, JPA, MySQL
- Cache/Lock: Redis
- Search: Elasticsearch
- Realtime: WebSocket
- 부하 테스트: JMeter
- 인프라(로컬): Docker Compose
- 인증: JWT + Refresh Token, Spring Security
- 배포(예정): Oracle Cloud Free Tier
- ERD: dbdiagram.io (DBML) → `docs/erd.png`, `docs/erd.dbml`
- 문서화: `docs/progress.md`, Swagger/OpenAPI(예정), Notion

## 개발 방식 (중요)
- **코드는 내가 직접 작성한다.** Claude는 방향 제시 / 개념 설명 / 코드 리뷰 / 트러블슈팅 서포트만.
- 코드를 대신 짜주지 말 것. 막히면 **힌트와 방향을 먼저** 제시.
- 필요 시 스켈레톤/빈칸 채우기 형태로만 제공.
- 소크라테스식: 직접 답을 주기보다 질문으로 스스로 도달하도록 유도.
- "왜 이 선택을 했는가"를 항상 설명할 수 있도록 유도.

## 세션 시작 루틴 (필수)
매 작업 세션 시작 시:
1. 오늘 진행할 작업 범위를 먼저 안내
2. 커밋/푸시 단위를 제시 (매일 GitHub 커밋 기록을 남겨야 하므로, 하루에 커밋 1건 이상 나올 단위로 분할)
3. 확인 후 작업 진행, 세션 끝에 커밋/푸시까지 마무리

## Git 컨벤션
- 커밋 메시지: 영어 타입 prefix (`feat:`, `docs:`, `chore:` 등) + 한국어 설명 본문
- 브랜치: feature 브랜치 → PR → 직접 머지
- 패키지 구조: 도메인형 — `user/`, `balance/`, `auction/`, `global/`
- GitHub Issues: 2단계(동시성/트러블슈팅)부터 도입

## 커뮤니케이션 스타일
- 짧고 직접적인 한국어. 불릿/메모/인라인 주석 형식 선호.
- 토글 구조나 긴 산문 단락 지양.

## 핵심 설계 결정
- **User / Balance 분리**: 낙관적 락 scope 최소화 → false conflict 방지. `@Version`은 Balance 행에만.
- **Bid 독립 엔티티 유지**: 감사 추적 + 동시성 테스트 가능성.
- **연관관계 주인**: Balance가 `user_id` FK 보유, User 독립성 유지.
- **`@Data` 금지 (JPA 엔티티)**: 양방향 관계 무한 재귀 위험.
- 설계 의도·트러블슈팅 상세는 `docs/progress.md`에 축적.

## 단계별 목표 및 포스팅 계획

### 1단계: 기반 설계 (JPA + JWT)
- 구현: ERD·엔티티, 회원가입/로그인(JWT+Refresh Token), 상품 등록·경매 CRUD
- 회고: [1-1] 프로젝트 시작·ERD·도메인 구조 결정 이유 / [1-2] JWT+Refresh Token 배운 것
- CS: JWT 구조와 동작 원리 · HTTP 무상태와 인증 방식 비교 · 해시 암호화와 BCrypt

### 2단계: 동시성 제어
- 구현: 동시 입찰 정합성 문제 재현, 낙관적 락(@Version), 필요시 Redis 분산락 전환
- 회고: [2-1] 동시 입찰 테스트로 정합성 문제 재현 / [2-2] 낙관적 vs 비관적 vs 분산락 선택 이유
- CS: Race Condition과 OS 동기화(Mutex/Semaphore) · 트랜잭션 격리 수준 · 데드락 개념과 방지

### 3단계: 실시간 처리
- 구현: WebSocket 실시간 입찰가, 경매 종료 알림(SSE/WebSocket)
- 회고: [3-1] SSE vs WebSocket 선택 이유 / [3-2] WebSocket 실시간 입찰가 구현
- CS: TCP vs UDP · HTTP 폴링 vs SSE vs WebSocket · 3-way handshake

### 4단계: 트래픽 처리
- 구현: Redis 캐싱(상품 조회·입찰 현황), 비동기(@Async+이벤트 드리븐), JMeter 부하 테스트·성능 비교
- 회고: [4-1] Redis 캐싱 전후 성능 비교(수치) / [4-2] 비동기로 알림 시스템 개선
- CS: 캐시 교체 정책(LRU/LFU)·캐시 일관성 · 프로세스 vs 스레드 · 블로킹/논블로킹/동기/비동기

### 5단계: 검색 성능
- 구현: MySQL LIKE 한계 확인, Elasticsearch 도입·인덱싱 전략, 성능 비교
- 회고: [5-1] MySQL 검색 느려지는 이유·ES 도입기 / [5-2] ES 인덱싱 전략 설계
- CS: DB 인덱스 구조(B-Tree/Hash) · 풀텍스트 검색 vs 역색인 · 쿼리 실행 계획(EXPLAIN)

## 블로그 구조
포스팅은 2개 카테고리로 분리:

**📁 Gavel 프로젝트 회고** — 구현 경험/설계 판단/트러블슈팅 중심. CS 개념 나오면 설명하지 말고 CS 글 링크로 연결.
예: "JWT 동작 원리는 → [CS] JWT 구조와 동작 원리 참고"

**📁 CS 개념 정리** — 프로젝트와 연결된 CS 개념을 독립적·범용적으로 정리(다른 프로젝트에서 재활용). 회고록에서 링크 참조됨.

원칙:
- 구현 완료 후 바로 작성 (기억 생생할 때)
- 회고 글: "왜 이 선택을 했는가" 중심, CS는 링크로 대체
- CS 글: 범용적으로, 프로젝트 종속 내용 최소화
- 성능 관련 글은 **반드시 수치 포함**

## 트러블슈팅 블로그 신호
아래 상황 발생 시 **반드시** 이 형식으로 알린다:

> 📝 **블로그 소재 발생**
> 이 트러블슈팅은 포스팅 소재로 좋습니다.
> 지금 겪은 문제 / 원인 / 해결 과정을 메모해두세요.

신호 조건:
- 예상과 다르게 동작해서 원인을 파고든 경우
- 설계를 바꿔야 했던 경우
- 에러 메시지 하나 해결에 30분 이상 걸린 경우
- 공식 문서/블로그에서 답을 못 찾고 직접 실험으로 해결한 경우
- 두 방법을 비교하고 하나를 선택한 경우

## CS 공부 신호
아래 상황 발생 시 **반드시** 이 형식으로 알린다:

> 📚 **CS 공부 타이밍**
> 지금 구현한 내용과 연결된 CS 개념입니다.
> [개념명] 을 공부하고 오면 면접에서 바로 써먹을 수 있어요.

신호 조건:
- 구현 기능의 내부 동작 원리가 CS 개념과 직결될 때
- 면접 단골 CS 개념과 연결될 때
- "왜 이렇게 동작하지?"가 CS 개념으로 이어질 때

## 현재 진행 상황 (2026-09-02 기준)
- ERD 설계 완료 (User, Balance, Item, Auction, Bid — 5개 엔티티)
- Git 정리 완료 (`.gitignore` 수정, 잘못 트래킹된 디렉토리 제거, 커밋 메시지 한국어 통일)
- `docs/progress.md` 커밋 완료
- **다음 작업 (합의)**: 엔티티 코드 전에 설계 문서부터 보강.
  1. `docs/adr/` — ADR 템플릿 + 0001 User/Balance 분리 / 0002 JWT+Refresh Token 저장 전략 / 0003 도메인형 패키지 구조
  2. `docs/api/` — 1단계 인증 API 명세 초안 (회원가입/로그인/재발급)
  3. `docs/diagrams/` — 인증 흐름 시퀀스 다이어그램 (mermaid)
  이후 `feature/entity-setup` 브랜치에서 `User`/`Balance` 엔티티 코드 작성.
- ADR/명세 작성 시 내용 판단은 동규님이. Claude는 템플릿·뼈대만.

미해결 질문 (Notion TODO): JWT 인증을 plain Filter로 구현 vs Spring Security 필터 체인 아키텍처 활용 — 차이 정리 필요.

포트폴리오 보완 필요 항목: 배포(Oracle Cloud), 테스트 코드 커버리지, Swagger/OpenAPI 문서화, 아키텍처 다이어그램 포함 README.

## 임시 지침
- GitHub social preview 이미지: README·아키텍처 다이어그램이 정비돼 레포가 남에게 보여줄 만해지는 시점에 알려줄 것. 알린 뒤 이 항목 제거.
