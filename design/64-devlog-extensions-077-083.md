# devlog 확장 설계 (spec 077~087, V27~V31)

작성 2026-10-11 · 매일 문서 점검. [63 devlog 확장 설계](./63-devlog-extensions-068-076.md)에 이어, 그 뒤 devlog에서 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 팀 공통 설계(01~53, 통합 ERD V1)는 그대로 두었습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/main/CHANGELOG.md) v1.53.0~v1.64.2, [마이그레이션](https://github.com/AIGJ-01-002-blog/devlog/tree/main/app/backend/src/main/resources/db/migration) V27~V31, 컨트롤러·`SecurityConfig`·`ContentSecurityPolicy`.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 077 | 사이트 소개, 처리방침 광고 항목 재동의 | 1.53.0 | 없음 | 화면 `/about`, `GET /api/site/about` | 서버가 본문을 먼저 그려 검색 엔진이 읽음, sitemap 포함. 처리방침 버전이 올라 기존 회원은 다음 로그인 때 다시 동의 |
| 052 갱신 | ChatGPT 커넥터 OAuth 보강 | 1.53.1 | 없음 | 토큰 요청의 `Authorization: Basic` client_id 허용 | 비밀값은 보지 않음(공개 클라이언트 그대로) |
| 078 | 디스코드 웹훅 알림, 텔레그램 회원 누구나 | 1.54.0 | `member_discord` (V27) | `GET`·`PUT`·`PATCH`·`DELETE /api/me/discord`, `POST /api/me/discord/test` | 본인만. 디스코드 공식 웹훅 주소만 받고 요청은 discord.com으로만. 토큰은 화면·로그에 내보내지 않음. 연결·시험은 한 시간에 10번. `TELEGRAM_AUDIENCE` 기본 `MEMBERS` |
| 015 A-1 | 친구 요청·수락 알림 | 1.55.0 | 알림 종류 `FRIEND_REQUEST`·`FRIEND_ACCEPTED` (V28) | 기존 알림 API | 같은 사람은 7일에 한 번. 알림 설정에서 따로 끔 |
| 079 | 팔로워·팔로잉 목록 공개 설정 | 1.56.0 | `member.follow_list_public` (V29) | `PATCH /api/me/settings`의 `followListPublic`, 목록 API가 비공개면 목록 대신 "비공개" 응답 | 아래 2절 |
| 080 | 카카오 로그인 | 1.57.0, 1.59.3 | 로그인 수단 `KAKAO`, 블로그 주소 `ka-` (V30) | OAuth2 `kakao` 등록, `GET /api/auth/signup/avatar` | `KAKAO_CLIENT_ID`가 있을 때만 켜짐. 사진 대신 받기는 가입 대기 중인 본인의 소셜 사진 주소만 |
| 081 | 화면 다국어 (영어·일본어·중국어) | 1.58.0~1.59.1 | 없음 | 없음 | 화면만. 브라우저 언어나 `?lang=`으로 고르고 브라우저가 기억. 관리자 화면·약관 본문·서버 오류 문장은 한국어 |
| 082 | 휴대폰 줄바꿈 정리 | 1.59.2, 1.61.1 | 없음 | 없음 | 화면만 |
| 020 FR-039 | 탈퇴 회원 바로 정리 | 1.60.0 | 없음 | `POST /api/admin/members/{handle}/purge` | 관리자만(매니저 403), 탈퇴 유예 회원만(아니면 409). 30일 정리와 같은 단계, 되돌릴 수 없음 |
| 083 | Facebook 로그인 | 1.61.0 | 로그인 수단 `FACEBOOK`, 블로그 주소 `fb-` (V31) | OAuth2 `facebook` 등록 (Graph API v26.0) | `FACEBOOK_CLIENT_ID`가 있을 때만 켜짐 |
| 084 | 화면 코드의 `.js` 사본 지우기 | 1.61.2 | 없음 | 없음 | 구조: 화면 코드는 `.ts`·`.tsx`만 두고 `tsconfig` `noEmit`, `src/**/*.js`는 git에서 뺌(빌드가 옛 `.js`를 먼저 읽던 문제) |
| 085 | 애드센스 반려 대응 | 1.62.0 | 없음 | `GET /api/site/policies/{kind}`, 화면 `/terms`·`/privacy`를 서버가 본문까지 그림 | 약관 문안은 `policy/terms.html`·`policy/privacy.html` 한 곳. 공개 글이 하나뿐인 태그 화면은 noindex, 광고 없음 |
| 086 | AI가 쓴 글의 비밀값 막기 | 1.63.0 | 없음 | MCP `publish_post`·`add_note`·일기 자동 발행 동작 변경 | 아래 2절. 사람이 읽지 않고 공개되는 길에서만 막음 |
| 087 | 설정 › AI 연결 앱별 안내 | 1.64.0~1.64.1 | 없음 | 없음 | 화면만 |

v1.53.2(팔로우 알림 중복 고침)와 v1.64.2(다국어 빈틈·명세 목록 정리)는 알림 처리 규칙만 바뀌어 공통 설계에 영향이 없습니다.

## 2. 권한 매트릭스에 더해진 것

[42 권한 매트릭스](./42-permission-matrix.md), [62 §2](./62-devlog-extensions-053-067.md#2-권한-매트릭스에-더해진-것), [63 §2](./63-devlog-extensions-068-076.md#2-권한-매트릭스에-더해진-것)에 더해:

| 행동 | 비회원 | 회원 | 본인 | 매니저 | 관리자 |
|---|---|---|---|---|---|
| 비공개로 둔 팔로워·팔로잉 목록 보기 | ✗ ("비공개") | ✗ ("비공개") | ✓ | ✓ | ✓ |
| 팔로워·팔로잉 수 보기 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 디스코드 웹훅 연결·끊기·시험 | ✗ | 본인 것만 | ✓ | 본인 것만 | 본인 것만 |
| 탈퇴 회원 바로 정리 | ✗ | ✗ | ✗ | ✗ (403) | ✓ |

AI(MCP)가 쓴 글의 비밀값 검사(086): 토큰·API 키·개인 키·JWT·주소 속 비밀번호·주민등록번호 형식이 보이면 `publish_post`는 발행하지 않고, 일기 자동 발행은 임시글로 두며, `add_note`는 메모를 받지 않습니다. 임시글 만들기·고치기는 저장하되 주의 문장만 돌려줍니다. 찾은 값은 응답·로그에 남기지 않고 종류만 알립니다. 웹에서 사람이 직접 발행하는 글은 검사하지 않습니다.

## 3. ERD에 더해진 테이블·규칙 (V27~V31)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `member_discord` | V27 | PK(회원), 회원 FK(CASCADE), 웹훅 번호·토큰, 채널 이름, 알림 켜기(기본 true) |
| 알림 종류 `FRIEND_REQUEST`·`FRIEND_ACCEPTED` | V28 | `notification`·`notification_mute` 종류 CHECK에 추가. 대상 표 없이 `notification_actor`에 사람을 적음 |
| `member.follow_list_public` | V29 | 기본 true |
| `auth_identity.provider` CHECK | V30·V31 | `LOCAL`·`GITHUB`·`GOOGLE`·`KAKAO`·`FACEBOOK` |
| `member.handle` CHECK | V30·V31 | 소셜 접두어 `go-`·`gi-`·`ka-`·`fb-`. 이메일 가입 주소가 `ka_`·`fb_`로 시작하지 못하는 규칙은 기존 회원 때문에 DB가 아니라 애플리케이션(`HandlePolicy`)에서 막음 |

## 4. 구조(아키텍처)에 더해진 것

- **디스코드(discord) 모듈**: 화면 알림이 생기면 텔레그램과 같은 문구(`NotificationText`, 알림 모듈로 옮김)로 디스코드 웹훅에 보냅니다. `wait=true`로 저장을 확인하고, 429면 5초 이하로 기다렸다 한 번 더 보냅니다. 웹훅이 지워졌으면 연결을 지웁니다. 탈퇴 정리 단계에도 들어갑니다.
- **소셜 로그인 확장**: Spring OAuth2 Client 등록에 카카오·Facebook을 더했습니다. 키가 없으면 등록과 버튼이 모두 빠집니다. 가입 마무리 화면 CSP의 `img-src`에 카카오·Facebook 사진 호스트를 더했고, 카카오처럼 브라우저가 가공할 수 없는 사진은 서버가 가입 대기 세션의 주소만 대신 받아 옵니다(저장하지 않음).
- **화면 다국어**: React 화면에 언어별 사전(한·영·일·중)과 숫자·날짜 표기를 두고, 화면 문장이 세 언어 사전에 모두 있는지 테스트가 확인합니다. 서버 API는 바뀌지 않았습니다.
- **서버가 그리는 공개 화면**: `/about`, `/terms`, `/privacy`는 다른 화면과 달리 서버가 본문까지 그립니다. 스크립트를 돌리지 않는 검토 도구·검색 엔진도 읽고, React 화면은 같은 문안을 API로 받습니다.
- **화면 코드 빌드**: 화면 코드는 TypeScript 원본만 저장소에 두고 `.js`는 빌드 때만 만듭니다(084).

> 이후 확장(spec 088~091)은 [65 devlog 확장 설계](./65-devlog-extensions-088-091.md)에 이어집니다.
