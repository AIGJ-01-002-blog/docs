# devlog 문서

개발자 블로그 서비스 **devlog**의 요구사항·설계 문서 저장소입니다. 소스코드, 기능 명세(Spec Kit `specs/`), ERD SQL, 배포 설정은 코드 저장소 **[AIGJ-01-002-blog/devlog](https://github.com/AIGJ-01-002-blog/devlog)** 에 있습니다.

| 저장소 | 담는 것 |
|---|---|
| [devlog](https://github.com/AIGJ-01-002-blog/devlog) | 앱(Spring Boot·React), 기능 명세 `specs/`, ERD SQL `erd/`, 배포 `deploy/`, CI·릴리스 |
| docs (여기) | 사람이 읽는 요구사항·설계 문서, 설계 검증 기록 |

코드 주석의 `docs/10 §2` 같은 표기는 이 저장소의 `design/10-…md` 2절을 뜻합니다.

## 설계 문서 (`design/`)

번호대: 01–13 공통 기준과 핵심 흐름 · 20–25 소통·알림 · 30–34 반응·발견 · 40–45 화면·권한 · 51–53 통합 명세·품질 · 60– 다음 방향

| 번호 | 문서 |
|---|---|
| [01](design/01-common-requirements.md) | 팀 공통 최소 요구사항 (초안) |
| [02](design/02-architecture.md) | 팀 공통 아키텍처 (초안) |
| [03](design/03-erd.md) | 팀 공통 ERD (초안) |
| [04](design/04-draft-and-image.md) | 임시저장·자동 저장과 사진 업로드 설계 |
| [05](design/05-publish.md) | 발행·수정 설계 |
| [06](design/06-visibility.md) | 공개 범위 설계 |
| [07](design/07-auth.md) | 로그인·로그아웃 설계 |
| [08](design/08-blog-address.md) | 블로그 주소(아이디) 설계 |
| [09](design/09-nickname.md) | 닉네임 설계 |
| [10](design/10-post-list.md) | 전체 글 목록과 개인 블로그 페이지 설계 |
| [11](design/11-profile.md) | 프로필 수정·계정 설정 설계 |
| [12](design/12-content-sanitize.md) | 글 작성·본문 정화 설계 |
| [13](design/13-delete-withdraw.md) | 글 삭제·회원 탈퇴 데이터 처리 설계 |
| [20](design/20-domain-events.md) | 도메인 이벤트 목록 (알림 연결용) |
| [21](design/21-comment.md) | 댓글·답글 설계 |
| [22](design/22-tag.md) | 태그 설계 |
| [23](design/23-image.md) | 이미지 업로드: 남은 결정 |
| [24](design/24-follow-feed.md) | 팔로우·팔로잉 피드 설계 |
| [25](design/25-notification.md) | 인앱 알림 설계 |
| [30](design/30-like.md) | 글 좋아요 설계 |
| [31](design/31-view-count.md) | 조회수 설계 |
| [32](design/32-trending.md) | 트렌딩 설계 |
| [33](design/33-search.md) | 검색 설계 |
| [34](design/34-ai-tag-suggest.md) | AI 태그 추천 설계 |
| [40](design/40-post-detail.md) | 글 상세 설계 |
| [41](design/41-manage-posts.md) | 내 글 관리 설계 |
| [42](design/42-permission-matrix.md) | 권한 매트릭스 (C-OWN-1 최종) |
| [43](design/43-report-hide.md) | 신고·관리자 숨김 설계 |
| [44](design/44-withdraw.md) | 회원 탈퇴·복구 화면 설계 |
| [45](design/45-dark-mode.md) | 다크 모드 설계 |
| [51](design/51-erd-unified.md) | 팀 공통 ERD 통합 명세 |
| [52](design/52-feature-summary.md) | 팀 블로그 플랫폼 기능 총정리 |
| [53](design/53-code-quality.md) | 코드 품질 분석 (SonarQube) |
| [60](design/60-direction-ai-mcp-premium.md) | devlog 다음 방향: UI · AI · MCP · 유료 콘텐츠 |
| [61](design/61-home-ai-ollama.md) | 집 PC AI(Ollama) 연결하기 |
| [62](design/62-devlog-extensions-053-067.md) | devlog 확장 설계 (spec 052~067, V13~V21) |
| [63](design/63-devlog-extensions-068-076.md) | devlog 확장 설계 (spec 068~076, V22~V26) |

## 설계 검증 (`verification/`)

요구사항 문서를 합친 뒤 경계의 모순·빠진 것을 찾은 검증 지시문과 보고서입니다. [verification/README.md](verification/README.md)

## 기록

2026-10-08에 코드 저장소에서 문서만 git 기록째 옮겨 왔습니다(`docs/` → `design/`). 옮기기 전 기록은 각 파일의 `git log`로 볼 수 있습니다.
