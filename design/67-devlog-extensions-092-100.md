# devlog 확장 설계 (spec 092·097~100, V36~V46)

작성 2026-10-15 · 매일 저장소·다국어 점검. [65 devlog 확장 설계](./65-devlog-extensions-088-096.md)(spec 088~096, V32~V35)에 이어, v1.68.0부터 devlog에서 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다. 기능별 그림은 [70 기능별 아키텍처](./70-feature-architecture.md)(v1.69.0 기준)에 있습니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/CHANGELOG.md) v1.68.0~v2.1.0(목표 v2.1.0 [#253](https://github.com/AIGJ-01-002-blog/devlog/issues/253)), 마이그레이션 V36~V46. v2.1.0 릴리스로 V46까지 main에 들어갔습니다. 그 뒤(spec 101~102, V47~V53)는 [68](./68-devlog-extensions-101-102.md)에 이어집니다.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 092 | develop 시험 환경 | 1.68.0 | 없음 | 없음 | develop에 머지하면 테스트·이미지를 거쳐 같은 클러스터의 네임스페이스 `blog-dev`(dev.devlog.life)에 배포. DB·Redis·MinIO·비밀값을 운영과 나눔. `STAGING_ENABLED=true`일 때만 돎 |
| 097 | 글 읽기 경험 다듬기 | 1.69.0 | 없음 | 없음(앱 404 화면은 화면 경로에만) | 최근 읽은 글·최근 검색어는 이 기기 저장소에만 둠. 공개 글에 `twitter:card`·JSON-LD BlogPosting. 렌더링 규칙 버전 4 |
| 098 | 첫 로그인 AI 연결 안내 | 1.70.0 | 없음 | 없음 | 닫았음은 브라우저에 회원별 키 `devlog.ai-onboarding.done.{회원번호}`로만 기억(서버 저장 없음) |
| 099 | 쇼츠 스튜디오 | 2.0.0 | `shorts`, `shorts_render_request` (V36), `shorts_upload` (V37), `shorts_batch` (V42) | `/api/me/shorts`(목록·만들기·고치기·지우기), `POST /api/me/shorts/{id}/check`(양산형 검사), `POST /api/me/shorts/{id}/render`, `PUT·DELETE /api/me/shorts/{id}/uploads/{platform}`, `/api/me/shorts/batches`, 렌더 워커 `POST /api/render/jobs/claim`·`PUT /api/render/jobs/{id}/video`·`POST /api/render/jobs/{id}/fail` | `SHORTS_ENABLED`(기본 꺼짐). 영상은 집 컴퓨터의 렌더 워커가 대기열에서 가져가 만들고(`claim_token`), SNS에는 사람이 직접 올림(자동 업로드 없음). MCP `create_shorts_script`·`update_shorts_script`·`list_shorts`·`plan_shorts` |
| 매매 일지 | 코인 매매 기록 | 2.0.0 (V39), 2.1.0 (V40·V41) | `trade`, `trade_day` (V39), 거래소·통화 (V40), 선물 (V41) | `GET /api/me/trades`, `GET /api/me/trades/{date}`, `PUT …/{date}/memo`, `POST …/{date}/draft`, `POST …/{date}/shorts`, `DELETE /api/me/trades/fills/{id}` | `TRADES_ENABLED`(기본 꺼짐). 매매 봇이 쓰기 토큰으로 MCP `record_trade`를 부름. 거래소 키는 devlog에 두지 않음. 통화가 다르면 합치지 않고 따로 보여 줌 |
| AI 메모 근거 | 일지·메모 links | 2.0.0 | `ai_note.links` (V38) | MCP `write_devlog`·`create_draft`·`add_note`의 `links` | 일지는 본문 끝 `## 근거`, 일기는 주제 문단 아래에 붙임 |
| 100 | 블로그 포인트 | 2.1.0 | `point_ledger`·`point_attendance`·`point_badge`·`point_rule` (V43), `shop_item`·`member_item`·`member_decoration` (V44), `paper_position`·`paper_order` (V45), `point_outbox` (V46) | `GET /api/me/points`·`/history`·`/today`, `POST /api/me/points/attendance`, `GET /api/shop`, `POST /api/shop/items/{id}/purchase`, `PUT·DELETE /api/me/decorations/…`, `GET /api/members/{handle}/badges`·`/decorations`, `GET /api/me/paper`, `POST /api/me/paper/orders`, 관리자 `/api/admin/points/rules`·`/api/admin/shop/items` | `POINTS_ENABLED`(기본 꺼짐). 지갑 두 개: 첫 지급은 투자 전용(`INVEST_ONLY`), 출석·글·모의투자 수익은 상점(`SHOP`, 화면 이름 "크레딧"). 포인트는 현금으로 바꾸지 않음. 글 보상은 outbox로 나중에 적립 |

v1.70.0의 하이브리드 검색 되살리기·닉네임 바로 확인, v2.0.0의 내 대시보드·집 컴퓨터로 옮기기(`deploy/home/`)는 테이블을 바꾸지 않았습니다.

## 2. ERD에 더해진 테이블·컬럼 (V36~V46)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `shorts` | V36 | 회원의 쇼츠 대본과 영상. 상태 `DRAFT`·`QUEUED`·`RENDERING`·`READY`·`FAILED`(CHECK). `READY`면 `video_key` 필수, `RENDERING`이면 `claim_token` 필수. 대기열·렌더 중 부분 인덱스 |
| `shorts_render_request` | V36 | 회원별 렌더 요청 기록(하루 한도 계산용) |
| `shorts_upload` | V37 | 직접 올린 뒤 남긴 영상 주소. 플랫폼 `YOUTUBE`·`INSTAGRAM`·`TIKTOK` |
| `ai_note.links` | V38 | varchar(1000), 기본 빈 문자열 |
| `trade`, `trade_day` | V39 | 체결 한 건(모드 `PAPER`·`LIVE`, `BUY`·`SELL`, 수량 > 0, 가격·수수료 ≥ 0)과 날짜별 메모 |
| `trade.exchange`, `trade.currency` | V40 | 거래소(대문자·숫자) 와 통화(`KRW` 기본, 대문자 2~10자). 금액 자릿수를 넓혀 소수 수량을 그대로 둠 |
| `trade.market`, `position_side`, `leverage`, `timeframe` | V41 | `SPOT`·`FUTURES`, 롱·숏, 레버리지, 봉 `1m`~`1d` |
| `shorts_batch`, `shorts.batch_id`·`topic`·`source_text` | V42 | 글·맥락 하나를 주제 1~5개로 나눈 묶음. 묶음 안 주제 번호는 겹치지 않음(부분 유일 인덱스) |
| `point_ledger`, `point_attendance`, `point_badge`, `point_rule` | V43 | 원장(지갑 `INVEST_ONLY`·`SHOP`, 금액 0 금지), 출석(연속 일수 ≥ 1), 휘장(20·50·100·150·200·300·365일), 관리자 규칙 값 |
| `shop_item`, `member_item`, `member_decoration` | V44 | 꾸미기 아이템(`THEME`·`FRAME`·`BANNER`·`EFFECT`, 티어 1~9), 산 아이템, 종류별로 하나 끼운 꾸미기 |
| `paper_position`, `paper_order` | V45 | 포인트 모의투자 보유와 주문. 원가를 지갑별(`cost_invest`·`cost_shop`)로 나눠 수익 적립 지갑을 정함 |
| `point_outbox` | V46 | 글 발행 보상 등 나중에 적립할 일, `next_at` 순으로 처리 |

## 3. 개발·릴리스 구조

목표 단위 Git Flow는 [65 §3](./65-devlog-extensions-088-096.md#3-개발릴리스-구조)과 같습니다. v1.68.0부터 develop은 시험 환경(spec 092)에서 먼저 확인합니다. v2.0.0 뒤 목표는 "devlog 2.0 완성"([#253](https://github.com/AIGJ-01-002-blog/devlog/issues/253))이며 버전 번호는 v2.1.0입니다. 매매 봇은 별도 저장소에서 자체 버전을 씁니다.

## 4. 다국어 범위 (2026-10-15 점검)

[65 §4](./65-devlog-extensions-088-096.md#4-다국어-범위-spec-081-기준-2026-10-14-점검)의 범위 그대로입니다. 쇼츠 스튜디오·매매 일지·포인트·상점 화면도 4개 언어 사전을 거치며, 사전 키는 약 1,900개입니다. 변수로 넘기는 이름(장면 연출 이름, 상점 티어 이름)도 사전에 있습니다. 쇼츠 대본과 매매 메모는 쓴 그대로 두고 번역하지 않습니다.
