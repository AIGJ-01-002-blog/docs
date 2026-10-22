# devlog 확장 설계 (spec 103~107, V56~V61)

작성 2026-10-19, 고침 2026-10-20(v2.7.1), 2026-10-21(v2.10.0), 2026-10-22(v2.12.0, v2.13.0) · 매일 저장소·다국어 점검. [68 devlog 확장 설계](./68-devlog-extensions-101-102.md)(spec 101~102, V47~V55)에 이어, v2.6.0부터 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 [`specs/103-shorts-channel`](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/specs/103-shorts-channel/spec.md)·[`specs/104-adsense-ready`](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/specs/104-adsense-ready/spec.md)·[`specs/105-adsense-recheck`](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/specs/105-adsense-recheck/spec.md)·[`specs/106-adsense-ads-on-articles`](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/specs/106-adsense-ads-on-articles/spec.md)·[`specs/107-portfolio-seo`](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/specs/107-portfolio-seo/spec.md)이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/CHANGELOG.md) v2.6.0~v2.13.0(목표 [#397](https://github.com/AIGJ-01-002-blog/devlog/issues/397), [#408](https://github.com/AIGJ-01-002-blog/devlog/issues/408), [#439](https://github.com/AIGJ-01-002-blog/devlog/issues/439), [#449](https://github.com/AIGJ-01-002-blog/devlog/issues/449), [#443](https://github.com/AIGJ-01-002-blog/devlog/issues/443)), 마이그레이션 V56~V61. v2.8.0 릴리스로 V58이, v2.12.0 릴리스로 V59가, v2.13.0 릴리스로 V60·V61이 main에 들어갔고, v2.9.0~v2.11.0은 새 마이그레이션이 없습니다.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 103 | 쇼츠 채널을 블로그에 잇기 | 2.6.0 | `member_social_link.kind`에 `YOUTUBE`·`INSTAGRAM`·`TIKTOK` (V56), `shorts.on_blog` (V57) | `PUT /api/me/shorts/{id}/blog` `{visible}`, `GET /api/members/{handle}/shorts`(최근 60편), `GET /api/posts/{id}/shorts`(6편), 화면 경로 `/@handle/shorts` | 플랫폼 로그인(OAuth)·자동 업로드는 하지 않음(099의 "수동으로" 결정 유지). 채널은 소셜 정보(043)의 칸으로 두고 새 테이블을 만들지 않음. 블로그에 보이는 쇼츠는 회원이 직접 남긴 영상 주소로만 만들고, 바탕 글이 공개·발행·숨김 아님일 때만 보임. 쇼츠가 꺼진 서버는 404. 개수 상한은 설정 `blog.shorts.blog-max`·`post-max`. YouTube 재생은 `youtube-nocookie.com` 임베드(CSP `img-src https://i.ytimg.com`, `frame-src https://www.youtube-nocookie.com`), Instagram·TikTok 임베드 스크립트는 싣지 않음 |
| 104 | 애드센스 승인 준비 | 2.7.0, 2.7.1 | 없음 | 없음(서버 HTML·수집 규칙만) | 글 없는 블로그(AI 일기만 있는 블로그 포함)·시리즈 없는 시리즈 목록은 `noindex`이고 광고 코드가 없음. 태그 수집 기준(085)은 AI 일기가 아닌 공개 글 수로 셈. 트렌딩 탭의 정규 주소는 `/`. 수집하는 화면의 서버 HTML에 소개·이용약관·개인정보 처리방침·문의 바닥글. `/about`·`/support`에 로그인 없는 연락 방법(설정 `blog.site.contact-email`·`contact-url`). v2.7.1부터 홈 피드와 트렌딩에서 AI 일기를 빼고 트렌딩 순위 키를 바꿈 |
| 105 | 애드센스 재심사 준비 | 2.8.0 | `post.body_chars` (V58) | 없음(서버 HTML·수집 규칙만) | 수집 대상 글은 "AI 일기가 아니고 본문 1,000자 이상인 공개 글"(`PostAccessPolicy.INDEXABLE_ARTICLE_CONDITION`). 기준 미만 글은 `noindex`·광고 없음·사이트맵 제외(글은 그대로 공개). 블로그·태그 화면의 수집 기준(104)도 이 글 수로 셈. 홈 서버 HTML에 사이트 소개 문단과 `/about` 링크 |
| 106 | 광고는 글 화면에만 | 2.9.0 | 없음 | 없음 | 광고 코드(`adsbygoogle.js`)와 nonce는 수집하는 글 화면(`og:type=article`)에만. 다른 공개 화면은 CSP가 nonce 없이 `'self'`만 믿고, 수집 화면에는 `google-adsense-account` 메타만 둠. 사이트맵에서 블로그 첫 화면(`/@아이디`)을 빼고 글 본문만 남김 |
| (목표 #449, spec 없음) | 쇼츠 재생목록 묶음 | 2.12.0 | `shorts.playlist` (V59) | `PUT /api/me/shorts/{id}/playlist` `{playlist}`, MCP `update_shorts_script`의 `playlist`(빈 문자열이면 지움) | 블로그 주인 결정 "하루 1개, 7주"(2026-10-21). 재생목록은 회원이 YouTube에서 직접 만들고 devlog는 넣을 묶음만 기억함(자동 업로드 없음 유지). 묶음만 고르면 대본과 만든 영상은 그대로. 올리기 안내에 재생목록 이름 "[백엔드 면접] 묶음", 글 쇼츠 제목 앞머리 "[백엔드 면접]", 매일 20:00(한국 시간) 예약 공개 |
| 107 | 포트폴리오 블로그와 검색 색인 정리 | 2.13.0 | `post.index_approved_at` (V60), `post.ai_assisted`·`post.retro_week` (V61) | `PUT /api/admin/posts/{postId}/index-approval`, 화면 경로 `/portfolio`·`/contact`, 사이트맵 색인 `/sitemap.xml` → `/sitemaps/posts.xml`·`series.xml`·`pages.xml`. MCP `update_draft`·`publish_post`는 주간 회고를 거절 | 외부 회원(USER) 글은 운영진(관리자·매니저)이 수집을 허락해야 검색 수집·사이트맵 대상. 태그 화면은 늘 `noindex, follow`, 독자가 읽는 noindex 화면은 `noindex, follow`. robots.txt에 로그인 전용 화면(`/studio`·`/trade`·`/paper`·`/points` 등) 차단. 주간 회고 초안은 매주 일요일 22시(KST) `WeeklyRetroJob`이 만들고, 도입·배운 점에 사람이 100자 이상 써야 발행. 광고 스위치 `ADS_PROVIDER`(기본 none), 검색 엔진 확인 메타는 `GOOGLE_SITE_VERIFICATION`·`NAVER_SITE_VERIFICATION` 설정값. 새 모듈·의존성·인프라 없음 |

v2.6.0의 쇼츠 캐릭터 동작·표정과 설명 그림 짜임(#396), v2.10.0의 쇼츠 첫 화면·훅·길이(30~45초), v2.11.0의 세 편씩 같은 색 계열 첫 화면(#439, 서버가 계열을 정해 워커에 넘김)은 테이블을 바꾸지 않았습니다. 시리즈 이름 "에러로그 30초", v2.12.0의 제목 앞머리 "[백엔드 면접]"과 재생목록 이름(배포·DB·보안·성능·AI협업)은 회원이 YouTube 채널에 그대로 쓰는 이름이라 번역하지 않습니다.

## 2. ERD에 더해진 테이블·컬럼 (V56~V61)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `member_social_link.kind` | V56 | CHECK `ck_member_social_link_kind`가 `EMAIL`·`GITHUB`·`X`·`FACEBOOK`·`HOMEPAGE`·`YOUTUBE`·`INSTAGRAM`·`TIKTOK`. 값은 정리한 형태(YouTube는 `@핸들` 또는 `channel/UC…`, Instagram·TikTok은 아이디만) |
| `shorts.on_blog` | V57 | boolean, 기본 true. 남긴 영상을 블로그·글 화면에 보일지. 글마다 쇼츠를 찾는 `ix_shorts_post` 부분 인덱스(`post_id IS NOT NULL`) |
| `post.body_chars` | V58 | integer, 생성 열(STORED). 발행 본문(`content_md`)에서 코드 블록·그림·링크 주소를 빼고 공백을 하나로 줄인 글자 수. 글을 쓰고 고치는 코드는 바꾸지 않음 |
| `shorts.playlist` | V59 | varchar(12), NULL이면 아직 안 고름. CHECK `ck_shorts_playlist`가 `DEPLOY`·`DB`·`SECURITY`·`PERFORMANCE`·`AI` |
| `post.index_approved_at` | V60 | timestamptz, NULL이면 외부 회원 글은 `noindex`·사이트맵 제외. 운영진 글은 이 열과 상관없이 수집 기준(일기 아님·본문 1,000자 이상)만 봄 |
| `post.ai_assisted`·`post.retro_week` | V61 | `ai_assisted` boolean 기본 false, 한 번 붙으면 떼지 않음(글 아래 "AI 보조 작성"). `retro_week` date, 주간 회고 초안이면 그 주 월요일(KST). 부분 유니크 인덱스 `uq_post_retro_week (author_id, retro_week) WHERE retro_week IS NOT NULL AND deleted_at IS NULL`로 한 사람에 한 주 하나. 마이그레이션이 기존 AI 일기·AI 제안 글·`## 근거`가 있는 글에 `ai_assisted`를 채움 |

## 3. 다국어 범위 (2026-10-19 점검)

[65 §4](./65-devlog-extensions-088-096.md#4-다국어-범위-spec-081-기준-2026-10-14-점검)의 범위 그대로입니다. 블로그 [쇼츠] 탭·"이 글의 쇼츠"·채널 칸과 툴팁도 4개 언어 사전을 거칩니다. 회원이 남긴 쇼츠 제목은 쓴 그대로 보여 주고 번역하지 않습니다. v2.13.0의 `/about`(AI 사용 범위 포함)·`/contact`·대본 큐도 4개 언어 사전을 거칩니다(2026-10-22 점검). `/portfolio`의 사례 문안은 운영자 글과 같은 콘텐츠라 서버 HTML(`site/portfolio.html`) 한 곳에 한국어로만 두고, 화면 틀(제목·버튼)만 번역합니다.
