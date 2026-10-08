# 53. 코드 품질 분석 (SonarQube)

학교 SonarQube가 백엔드(`app/backend`)를 분석한다. 이 문서는 분석 결과를 받는 방법과, 보안 핫스팟을 검토한 결과를 적는다.

## 1. 분석과 결과 받기

- main에 백엔드 변경이 들어오면 `backend-ci`의 "SonarQube 분석" 단계가 결과를 올린다. 분석 서버가 내려가 있어도 CI는 막지 않는다.
- 결과 목록은 수동 실행 워크플로 "SonarQube 이슈 내보내기"(`.github/workflows/sonar-export.yml`)로 받는다. 실행 로그에 서버 판과 이슈가 한 줄씩 찍히고, 아티팩트 `sonar-issues`에 JSON이 남는다.
- 핫스팟 목록은 토큰에 프로젝트 Browse 권한이 있어야 받을 수 있다. 분석 전용 토큰이면 이슈만 받는다.
- 학교 서버와 같은 판(25.1 Community)을 로컬 Docker로 띄우면 고친 결과를 머지 전에 확인할 수 있다.

## 2. 고친 것 (1.23.1)

2026-10-08 첫 분석의 보안 1건, 신뢰성 60건을 모두 고쳤다. 주요 원인은 다음과 같다.

| 규칙 | 건수 | 원인과 처리 |
|------|------|-------------|
| S6437 | 1 | 없는 이메일의 로그인 시간을 맞추는 가짜 해시를 고정 문자열로 만들었다. 실행마다 무작위 값으로 만든다 |
| S2259 | 34 | 대부분은 Spring 7이 `queryForList`의 반환 원소에 붙인 `@Nullable`을 분석기가 "목록이 null"로 읽은 것이다. 한 열 조회를 `shared/jdbc/Columns`(행 매퍼)로 바꿨다. 이메일 다루는 두 곳은 null 처리 순서를 바로잡고, `RETURNING id`·OAuth 사용자 정보는 `Objects.requireNonNull`로 확인한다 |
| S2583 | 5 | `tx.execute` 결과의 null 검사를 `Objects.requireNonNullElse`로 바꿨다 |
| S5998, S5850 | 6 | 반복 그룹이 긴 입력에서 스택을 넘칠 수 있는 정규식은 소유 수량자로, 우선순위가 모호한 정규식은 그룹으로 묶었다 |
| S6218 | 3 | 배열을 가진 record는 내용으로 비교하도록 `equals`·`hashCode`·`toString`을 직접 둔다 |
| S6906 | 1 | 메일 발송을 가상 스레드에서 작은 플랫폼 스레드 풀로 옮겼다. JavaMail이 `synchronized` 안에서 소켓을 기다려 운반 스레드를 붙잡기 때문이다 |
| 그 밖 | 11 | `PreparedStatement` 매개변수(S2695), `Optional` 확인(S3655), 같은 분기(S3923), 정수 넘침(S2184), 경로 변수(S6856), 생성자 `@Autowired`(S6829), 테스트의 클래스 이름 비교(S1872) |

`LocalMediaController`의 `/media/{*key}`는 나머지 경로 전체를 `key`로 받는 Spring 문법인데 분석기가 이름을 `*key`로 읽는다(S6856 오탐). 그 줄에 `// NOSONAR`와 이유를 남겼다.

## 3. 보안 핫스팟 검토

첫 분석의 핫스팟 31건 중 11건은 코드를 고쳐 없앴다.

- 정규식 역추적(S5852) 6건: 주소 끝 `/`·앞뒤 `-` 떼기는 정규식 대신 `TextCleaner.trim`·`trimEnd`로, 코드 울타리 정규식은 소유 수량자로 바꿨다.
- 양방향 문자(S6389) 2건: 테스트가 일부러 넣은 보이지 않는 문자를 `‮`처럼 이스케이프로 적었다.
- SQL 조립(S2077) 3건: 한 열 조회를 `Columns`로 바꾸면서 없어졌다.

남은 20건은 모두 SQL 조립(S2077)이고, 검토 결과 안전하다. 사용자 입력은 모두 `?` 매개변수로 넘기고, 이어 붙이는 조각은 다음 셋뿐이다.

| 붙이는 조각 | 어디서 오나 | 위치 |
|-------------|-------------|------|
| 클래스 상수 (`PostAccessPolicy.PUBLIC_LIST_CONDITION`, `PostSql.*`, `DUE`, `COLUMNS`, `PAGE_SIZE` 등) | 소스 코드에 고정 | WithdrawalPurgeJob, CommentQuery, AdjacentPostQuery, FeedQuery, RssFeed, FollowQuery, NotificationQuery, SeriesQuery, AdminReportQuery |
| 코드 안에서 고르는 고정 문자열 (커서가 있을 때의 `" AND (…) > (?, ?)"`, 신고 대상에 따른 `post_id`/`comment_id`, 숨김 대상의 테이블 이름 enum, 검색 단계 조건) | 분기마다 리터럴 | CommentQuery, NotificationQuery, AdminReportQuery, ReportService, ModerationService, SearchQuery |
| 설정 값에서 고르는 고정 문자열 (`TelegramProperties.audienceSql()`: `TRUE` 또는 `m.role = 'ADMIN'`), `Duration` 상수의 일수 | enum·상수 | TelegramLinks, ReportPurgeJob |

SonarQube 화면에서 이 20건을 "Safe"로 표시하려면 프로젝트 관리 권한이 있는 계정이 필요하다.

## 4. 남은 것

유지보수성(Code Smell) 이슈는 우선순위가 낮아 나중에 정리한다.
