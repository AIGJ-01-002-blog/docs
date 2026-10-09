# devlog 확장 설계 (spec 068~076, V22~V26)

작성 2026-10-10 · 매일 문서 점검. [62 devlog 확장 설계](./62-devlog-extensions-053-067.md)에 이어, 그 뒤 devlog에서 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 팀 공통 설계(01~53, 통합 ERD V1)는 그대로 두었습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/main/CHANGELOG.md) v1.42.0~v1.52.0, [마이그레이션](https://github.com/AIGJ-01-002-blog/devlog/tree/main/app/backend/src/main/resources/db/migration) V22~V26, 컨트롤러.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API·MCP | 권한·구조 |
|---|---|---|---|---|---|
| 068 | UX/UI 다시 점검 (초점·터치·확인 창) | 1.42.0 | 없음 | 없음 | 화면만 |
| 064 갱신 | 방문자 수에 운영진 방문도 셈 | 1.42.1 | 없음 | 없음 | 집계 규칙만 바뀜 |
| 069 | 목록 무한 스크롤 | 1.43.0 | 없음 | 기존 쪽 나눔 API | 화면만 |
| 070 | 유입 경로·많이 본 화면 | 1.44.0 | `visit_source_day`, `page_view_day` (V22) | 관리자 대시보드 | 원래 주소·검색어는 남기지 않고 하루 단위 수만. 400일 뒤 삭제. 관리자·글쓰기 화면은 세지 않음 |
| 071 | AI 글 제안 알림, 일기 시각·자동 발행, Mermaid, 주제별 나눠 쓰기 | 1.45.0 | `member.ai_diary_hour`, 알림 종류 `AI_PROPOSAL`, `notification_ai_proposal` (V23) | MCP 일기·제안 도구 동작 변경 | 일기 자동 발행은 "AI 발행·삭제 허용"을 켠 회원만. 일기 묶기 예약 작업이 회원별 시각으로 돎 |
| 072 | 새 디자인: 브랜치 그래프, 주제 자동 묶음, 시리즈 구독, 포트폴리오 모드 | 1.46.0~1.48.0 | `post_topic` (V24), `series_subscription`·`post_topic_optout` (V25), `series` 포트폴리오 칸 (V26) | `GET /api/posts/{id}/topic`·`similar`, `GET /api/topics/popular`, `POST /api/posts/{id}/branch-suggestion`, `PUT /api/posts/{id}/topic-optout`, `PUT`·`DELETE /api/series/{id}/subscription`, `GET /api/members/{handle}/portfolio`, `GET`·`PUT /api/me/series/{id}/project`, 화면 `/@{handle}/portfolio` | 주제 묶기 예약 작업(10분마다, 공개 글만, 브랜치당 최대 12편). 시리즈 구독은 로그인 회원. 프로젝트 칸은 시리즈 주인만 |
| 073 | MCP 시리즈·포트폴리오 도구 | 1.49.0 | 없음 | MCP `list_series`, `create_series`, `add_to_series`, `set_series_project` | `set_series_project`는 누구나 보는 화면을 바꾸므로 "AI 발행·삭제 허용"을 켠 회원만 |
| 074 | 관리자 대시보드 지표 묶음 | 1.50.0 | 없음 | 관리자 대시보드 | 화면만 |
| 075 | MCP 글감 추천 | 1.51.0 | 없음 | MCP `suggest_topics`(읽기) | 저장하지 않음. 근거 조각의 IP·메일 주소·비밀값은 가려서 돌려줌 |
| 076 | 구글 애드센스 | 1.52.0 | 없음 | `GET /ads.txt` | 공개 화면에만 광고 코드. 광고를 싣는 화면은 요청마다 새 nonce + `strict-dynamic` CSP, 다른 화면 CSP는 그대로. `ADSENSE_CLIENT`를 비우면 모두 꺼짐 |

v1.43.0~v1.52.0 사이의 나머지 릴리스(1.48.1, 1.48.2, 1.49.1, 1.50.1, 1.50.2, 1.51.1~1.51.3)는 화면·문구·배포 설정 수정이라 공통 설계에 영향이 없습니다.

## 2. 권한 매트릭스에 더해진 것

[42 권한 매트릭스](./42-permission-matrix.md)와 [62 §2](./62-devlog-extensions-053-067.md#2-권한-매트릭스에-더해진-것)에 더해:

| 행동 | 비회원 | 회원 | 시리즈 주인 |
|---|---|---|---|
| 포트폴리오 화면·주제·비슷한 글 보기 | ✓ (공개 글만) | ✓ | ✓ |
| 시리즈 새 글 알림 구독 | ✗ | ✓ | ✓ |
| 시리즈를 포트폴리오 프로젝트로 켜기·칸 쓰기 | ✗ | ✗ | ✓ |
| 내 글을 주제 브랜치에서 빼기 | ✗ | 글 작성자만 (남의 글 404) | 글 작성자만 |
| MCP로 프로젝트 칸 쓰기, 일기 자동 발행 | ✗ | ✗ | "AI 발행·삭제 허용"을 켠 경우만 |

## 3. ERD에 더해진 테이블 (V22~V26)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `visit_source_day` | V22 | PK(날짜, 경로 종류, 사이트), 방문 수 > 0 |
| `page_view_day` | V22 | PK(날짜, 화면 경로), 조회 수 > 0 |
| `member.ai_diary_hour` | V23 | 0~23 CHECK, 기본 0(자정) |
| 알림 종류 `AI_PROPOSAL` | V23 | `notification`·`notification_mute` 종류 CHECK에 추가 |
| `notification_ai_proposal` | V23 | 알림 FK(알림 번호·종류, CASCADE), 제안 FK(CASCADE), 종류 = `AI_PROPOSAL` |
| `post_topic` | V24 | PK(글), 글 FK(CASCADE), 묶은 방법 `EMBEDDING`·`TAG`·`MIXED`, 이름은 빈칸 금지 |
| `series_subscription` | V25 | PK(회원, 시리즈), 회원·시리즈 FK(CASCADE) |
| `post_topic_optout` | V25 | PK(글), 글 FK(CASCADE) |
| `series` 포트폴리오 칸 | V26 | `portfolio`(기본 false), 기간·한 줄 설명·쓴 기술·우리 팀이 한 일·제 역할 |

## 4. 구조(아키텍처)에 더해진 것

- **주제 브랜치(topic)**: 시리즈에 없는 공개 글을 10분마다 묶는 예약 작업. 임베딩이 있으면 임베딩 거리로, 없으면 드문 태그가 겹치는지로 봅니다.
- **포트폴리오 모드**: 블로그와 같은 글·시리즈 데이터를 `/@{handle}/portfolio`에서 프로젝트 단위로 보여 줍니다. 별도 저장소는 없습니다.
- **방문 집계**: 하루 단위 수만 남기는 유입 경로·화면별 집계가 방문자 수 집계에 더해졌습니다.
- **광고**: 공개 화면에만 광고 스크립트를 싣고, 그 화면만 nonce 기반 CSP를 씁니다.
