# devlog 확장 설계 (spec 052~067, V13~V21)

작성 2026-10-09 · 매일 문서 점검. 팀 공통 설계(01~53, 통합 ERD V1)는 그대로 두고, 김민서 개인 확장인 devlog에서 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/main/CHANGELOG.md) v1.27.0~v1.41.0, [마이그레이션](https://github.com/AIGJ-01-002-blog/devlog/tree/main/app/backend/src/main/resources/db/migration) V13~V21, `SecurityConfig`.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API·MCP | 권한·구조 |
|---|---|---|---|---|---|
| 052 | MCP 서버, ChatGPT·Codex 연결 | 1.27.0 | `oauth_client`, `personal_access_token` (V13) | `/mcp`, OAuth 동적 등록 | 접근 토큰(READ·WRITE) 주인 = 회원 |
| 053 | AI 발행·삭제 허용 | 1.28.0 | `member.ai_publish_allowed` (V14) | `GET`·`PUT /api/me/ai-publish`, MCP `publish_post`·`delete_post` | 설정은 웹 세션 + CSRF에서만. 토큰·OAuth로는 못 바꿈 |
| 054 | 문의·신고, AI 버그 신고 | 1.29.0 | `inquiry` (V15) | `POST /api/inquiries`, `GET /api/me/inquiries`, `GET`·`PATCH /api/admin/inquiries/**`, `GET /api/release-notes`, MCP `report_bug`, 관리자 전용 `list_inquiries`·`get_inquiry`·`update_inquiry` | 문의는 접수한 회원과 운영진만. 관리자 도구는 일반 토큰에서 "없는 도구" |
| 055 | 툴팁·모바일 아래 탭·서식 도구 | 1.30.0 | 없음 | 없음 | 화면만 |
| 056 | 하이브리드 검색 | 1.32.0 | `post_embedding` (V16, pgvector 있을 때만) | 기존 검색 API | 공개 글만 임베딩. 1분마다 20개씩 처리하는 예약 작업. 모델이 응답 없으면 키워드만 |
| 057 | 발행 전 점검 | 1.33.0 | 없음 | 없음 | 발행 창 화면만 |
| 058 | 글 수정 이력 | 1.33.0 | `post_revision` (V17, 글마다 최근 50판) | `GET /api/posts/{id}/revisions`, `GET /api/posts/{id}/revisions/{no}` | 작성자만. 남의 글은 404 |
| 059 | 내 글 내보내기 | 1.33.0 | 없음 | `GET /api/me/export`, `GET /api/me/export.zip` | 본인만. 회원마다 10분에 5번 |
| 060 | MCP 도구 강화 1차 | 1.31.0 | 없음 | MCP `upload_image`, `create_image_upload_link` 등 | 웹과 같은 사진 저장소·한도 |
| 061 | AI 글 제안·자정 일기 | 1.34.0 | `ai_post_proposal`, `ai_note`, `member.ai_diary_enabled` (V18) | MCP `propose_post`·`list_post_proposals`·`add_note`, `create_draft`의 `proposal_id` | 일기 설정은 웹에서만. 매일 00:00(KST) 예약 작업 |
| 062 | 관리자 페이지 | 1.35.0 | `member.role`에 `MANAGER` 추가 (V19) | `/api/admin/dashboard`·`members`·`members/{handle}/stats`·`members/{handle}/role`·`posts`·`posts/{id}/hide`·`unhide` | 아래 2절 |
| 063 | 머리말·내 메뉴·내 설정 | 1.37.0 | 없음 | 없음 | 화면만 (운영진에게만 관리자 메뉴) |
| 064 | 사이트 방문자 수 | 1.38.0 | `site_visit` (V20) | `POST /api/visits`, 관리자 대시보드 | 원래 IP는 남기지 않음. 400일 뒤 삭제 |
| 065 | 가입 화면 아이디 확인·이메일 인증번호 | 1.39.0 | 없음 | `POST /api/auth/signup/email-code`, `POST /api/auth/signup/email-code/verify` | 번호 10분·마지막 것만 유효, 5번 틀리면 폐기. `SIGNUP_EMAIL_CODE=on\|off` |
| 066 | 날짜별 활동한 회원 | 1.40.0 | `member_active_day` (V21) | 관리자 대시보드 | 400일 뒤 삭제 |
| 067 | 글 카드 태그·조회·댓글 수 | 1.41.0 | 없음 | 기존 목록 API 응답 | 화면만 |

## 2. 권한 매트릭스에 더해진 것

[42 권한 매트릭스](./42-permission-matrix.md)의 행위자(비회원·회원·작성자·관리자)에 **매니저**가 더해졌습니다.

| 행동 | 회원 | 매니저 | 관리자 |
|---|---|---|---|
| 관리자 페이지(`/api/admin/**`) 보기·글 숨김·문의 처리 | ✗ (403) | ✓ | ✓ |
| 회원 권한 바꾸기(매니저로 올리기·내리기) | ✗ | ✗ | ✓ |
| 매니저 정지 | ✗ | ✗ | 권한을 먼저 거둔 뒤에만 |
| 비공개 글 제목 보기(관리자 글 목록) | ✗ | ✗ | ✗ |
| 문의 내용 보기 | 본인 것만 | ✓ | ✓ |
| 글 수정 이력 보기 | 작성자만 (남의 글 404) | 같음 | 같음 |
| AI 발행·삭제 허용, 자정 일기 켜기 | 본인, 웹 세션에서만 | 같음 | 같음 |

- 권한을 바꾸면 그 회원은 다시 로그인해야 합니다.
- 운영자 계정은 실행 설정 `OWNER_HANDLE`·`OWNER_GITHUB_ID`로 앱이 뜰 때 관리자가 됩니다.
- MCP 접근 토큰으로 부를 때도 토큰 주인의 역할을 요청마다 새로 읽습니다.

## 3. ERD에 더해진 테이블 (V13~V21)

[51 통합 ERD](./51-erd-unified.md)는 팀 공통 V1(20테이블)입니다. devlog는 V2~V12 뒤에 아래를 더했습니다.

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `oauth_client` | V13 | AI 앱 동적 등록. 비밀값 없음(PKCE) |
| `personal_access_token` | V13 | 회원 FK(CASCADE), 토큰은 SHA-256만 저장, `scope IN ('READ','WRITE')` |
| `member.ai_publish_allowed` | V14 | 기본 false |
| `inquiry` | V15 | 회원 FK(CASCADE), 처리자 FK(SET NULL), 종류·경로·상태 CHECK, 답변과 답변 일시는 함께 있거나 함께 없음 |
| `post_embedding` | V16 | pgvector가 있는 DB에서만 생김. 글 FK(CASCADE) |
| `post_revision` | V17 | PK(글, 판 번호), 판 번호 > 0. 기존 발행 글을 1판으로 채움 |
| `ai_post_proposal`, `ai_note`, `member.ai_diary_enabled` | V18 | 제안 상태 `OPEN`·`DRAFTED`·`DISMISSED` |
| `member.role` CHECK | V19 | `USER`·`MANAGER`·`ADMIN`, 운영진 부분 인덱스 |
| `site_visit` | V20 | PK(날짜, 방문자 해시), 회원 여부, 방문 수 > 0 |
| `member_active_day` | V21 | PK(날짜, 회원), 회원 FK(CASCADE) |

## 4. 구조(아키텍처)에 더해진 것

[02 아키텍처](./02-architecture.md)의 모듈러 모놀리스에 아래 모듈·작업이 더해졌습니다.

- **MCP·OAuth**: `/mcp` 엔드포인트와 접근 토큰 인증. 사용자 쪽 AI(Claude·ChatGPT 등)가 웹과 같은 서비스를 부릅니다.
- **문의(inquiry)**, **관리자 콘솔(admin)** 모듈.
- **예약 작업**: 글 임베딩(1분마다), 자정 일기 묶기(00:00 KST), 방문·활동 기록 정리(400일).
- **의미 검색**: PostgreSQL pgvector + 임베딩 모델. 없거나 응답이 없으면 키워드 검색만 씁니다.

> 이후 확장(spec 068~076, V22~V26)은 [63 devlog 확장 설계](./63-devlog-extensions-068-076.md)에 이어집니다.
