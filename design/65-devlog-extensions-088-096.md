# devlog 확장 설계 (spec 088~096, V32~V35)

작성 2026-10-11, 고침 2026-10-14 · 매일 저장소·다국어 점검. [64 devlog 확장 설계](./64-devlog-extensions-077-083.md)(spec 077~087)에 이어, 그 뒤 devlog에서 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 화면 움직임(spec 095)은 [46 화면 움직임 설계](./46-ui-motion.md)에 따로 있습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다. v1.68.0 이후(spec 092·097~100, V36~V46)는 [67](./67-devlog-extensions-092-100.md)에 이어집니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/CHANGELOG.md) v1.65.0~v1.67.0, 마이그레이션 V32~V35. v1.67.0은 목표 단위 Git Flow로 낸 첫 릴리스입니다([목표 #169](https://github.com/AIGJ-01-002-blog/devlog/issues/169)).

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 088 | AI 일기를 검색 수집에서 빼기 | 1.65.0 | `post.ai_diary` (V32) | 없음 | 일기 글 상세는 공개여도 `noindex`, 광고 코드 없음, 사이트맵에서 뺌. 블로그·목록·RSS에는 그대로 보임 |
| 089 | 전체 태그 둘러보기 | 1.66.0 | 없음 | `GET /api/tags/browse?q=&sort=popular\|name\|recent&cursor=` | 공개 목록 조건의 글만 셈. 태그마다 인기 글 최대 10개(트렌딩과 같은 무게). 검색어가 있는 `/tags?q=`는 `noindex` |
| 090 | Search Console 색인 경고 대응 | 1.66.1 | 없음 | 없음 | 공개 글이 없는 태그 주소의 첫 화면은 404(예전 200 + 빈 화면). `/mcp`·`/releases`를 사이트맵에 넣음. 사이트맵이 robots.txt Disallow와 겹치지 않는지 테스트로 지킴 |
| 091 | 빈 임시글 바로 지우기 | 1.66.2 | 없음 | `DELETE /api/posts/{id}/if-empty` → `{deleted}` | 본인 임시글만. 서버가 자동 저장분까지 확인한 뒤 제목·본문·소개·태그·사진·첨부·썸네일·시리즈가 모두 없을 때만 휴지통 없이 지움. 새벽 정리·[삭제]·[새 글] 이어 쓰기도 같은 기준 |
| 020 갱신 | 탈퇴 신청 즉시 친구 관계 삭제 | 1.67.0 | `friendship` 정리 (V33) | 없음 | 처리방침의 "바로 지운다"에 맞춤. 30일 안에 복구해도 친구 관계는 되살리지 않음 |
| 093 | 릴리스 노트에 화면 사진·PR·비교 링크 | 1.67.0 | 없음 | 정적 경로 `/release-assets/<폴더>/<파일>`(30일 캐시) | 구조: `release.yml`이 이전 태그 뒤 머지된 PR 표와 `/compare/` 링크를 GitHub Release에 붙임. 빌드가 `assets/releases/` 사진을 jar에 넣고, 사이트 릴리스 노트는 GitHub 원본 사진 주소를 사이트 사본 주소로 바꿔 그림 |
| 096 | 네이버 로그인 | 1.67.0 | 로그인 수단 `NAVER`, 블로그 주소 `nv-` (V34) | OAuth2 `naver` 등록(scope 없음, 사용자 정보는 `response` 안) | `NAVER_CLIENT_ID`·`NAVER_CLIENT_SECRET`이 있을 때만 켜짐. 네이버 이메일은 인증된 이메일로 보지 않음. 사진은 `phinf.pstatic.net`만 받음 |
| 011 FR-011a | 대댓글을 Git 브랜치처럼 선으로 잇기 | 1.67.0 | `comment.reply_to_comment_id` (V35) | 댓글 응답에 답한 답글 번호 | 저장 구조는 그대로 깊이 1(`parent_id`는 늘 최상위). 화면만 3단계까지 들여 써 선으로 이음. 삭제·탈퇴·댓글 수 규칙은 바뀌지 않음 |

v1.65.1~v1.65.6(화면·글꼴·썸네일 다듬기)과 v1.67.0의 첫 화면 움직임·GitHub 바로가기는 공통 설계에 영향이 없습니다. 구성 비교 측정(#172)은 [66 가벼운 구성 실측 비교](./66-infra-light-benchmark.md)에 있습니다.

## 2. ERD에 더해진 컬럼·규칙 (V32~V35)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `post.ai_diary` | V32 | boolean, 기본 false. 자정 AI 일기가 만들 때 정하고 바꾸지 않음. 기존 일기는 제목 `YYYY-MM-DD 개발 일기`로 찾아 true |
| `friendship` | V33 | 이미 탈퇴 신청(`member.status = 'WITHDRAWN'`)한 회원의 남은 친구 관계를 한 번 지움. 이후에는 탈퇴 신청 처리에서 바로 지움 |
| `auth_identity.provider` CHECK, `member.handle` CHECK | V34 | 로그인 수단에 `NAVER`, 소셜 접두어에 `nv-`. 이메일 가입 주소가 `nv_`로 시작하지 못하는 규칙은 카카오·Facebook처럼 애플리케이션(`HandlePolicy`)에서 막음 |
| `comment.reply_to_comment_id` | V35 | bigint NULL, `comment` FK(ON DELETE SET NULL). 답글(`parent_id`가 있는 행)에만 값이 올 수 있음(CHECK). 부분 인덱스 |

## 3. 개발·릴리스 구조

2026-10-12부터 목표 단위 Git Flow입니다(devlog `CONTRIBUTING.md`). 기능은 `develop`에서 딴 `feature/*`에서 만들어 `develop`으로 합치고 버전을 올리지 않습니다. 목표(마일스톤) 이슈가 모두 닫히면 `develop`을 `main`에 합치며 다음 마이너 버전으로 릴리스하고, 운영 배포는 main에서 백엔드·화면 테스트가 통과한 뒤에만 합니다. 급한 장애·보안만 `hotfix/*`로 패치 릴리스합니다.

## 4. 다국어 범위 (spec 081 기준, 2026-10-14 점검)

| 대상 | 언어 | 비고 |
|---|---|---|
| 화면 문장(관리자 화면 제외) | 한국어·English·日本語·中文 | 한국어 문장을 키로 쓰는 사전 `locales/en·ja·zh.ts`, 키 1,354개 이상(기능이 늘 때마다 늘어남). `i18n.test.ts`가 빠진 키·자리표시 불일치를 잡음 |
| 관리자 화면 | 한국어 | 운영자만 써서 번역하지 않음 |
| 약관·처리방침 본문 | 한국어 | 법적 검토 뒤 번역본(일일 보고 질문) |
| 서버 오류 문장 | 한국어 | 다음 단계: `Accept-Language`로 고르기 |
| 글 본문·제목·태그·닉네임 | 쓴 그대로 | 번역하지 않음 |
| 코드 저장소 README | 한·영·일·중 4판 | `scripts/check-readmes.py`가 구조·릴리스 표·버전을 맞춤, 조직 프로필(.github)도 같은 내용 |
| 설계 문서(이 저장소)·기능 명세 `specs/` | 한국어 | 팀 문서라 한국어판만 둠 |

> 이후 확장(spec 092·097~100, V36~V46)은 [67 devlog 확장 설계](./67-devlog-extensions-092-100.md)에 이어집니다.
