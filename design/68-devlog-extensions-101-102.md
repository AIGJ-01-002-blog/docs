# devlog 확장 설계 (spec 101~102, V47~V55)

작성 2026-10-16, 고침 2026-10-16(v2.2.1), 2026-10-17(v2.3.0), 2026-10-18(v2.4.0), 2026-10-19(v2.5.0) · 매일 저장소·다국어 점검. [67 devlog 확장 설계](./67-devlog-extensions-092-100.md)(spec 092·097~100, V36~V46)에 이어, v2.1.0부터 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/CHANGELOG.md) v2.1.0~v2.5.0(목표 v2.2.0 [#282](https://github.com/AIGJ-01-002-blog/devlog/issues/282), v2.3.0 [#344](https://github.com/AIGJ-01-002-blog/devlog/issues/344), v2.4.0 [#372](https://github.com/AIGJ-01-002-blog/devlog/issues/372), v2.5.0 [#394](https://github.com/AIGJ-01-002-blog/devlog/issues/394)), 마이그레이션 V47~V55. v2.5.0 릴리스로 V55까지 main에 들어갔습니다. 그 뒤(spec 103, V56~V57)는 [69](./69-devlog-extensions-103.md)에 이어집니다.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 099 갱신 | 매매 재현 숏폼 | 2.1.0 | `shorts.replay` (V47) | `POST /api/me/shorts/replay`, `POST /api/me/trades/{date}/shorts` | 매매 일지 체결(같은 거래소·종목 1~6개)로 대본을 만들고, 렌더 워커가 실시간 매매처럼 그림 |
| 101 | 자동매매 모의 화면 `/trade` | 2.1.0 | `trade_bot_status`, `trade_bot_account`, `trade_bot_event` (V48) | `GET /api/me/trade-bot`, `GET /api/me/trade-bot/charts`, MCP `report_trade_bot` | 봇이 알린 모의 계좌·체결·Gemini 판단·하루 요약을 보기만 함. 화면에 주문 버튼 없음. 모의 기록은 운영 매매 일지(`trade`)와 나눔 |
| 102 | 서버가 돌리는 모의 봇 | 2.1.1, 2.2.0 | `trade_paper_bot` (V49) | `GET·PUT /api/me/trade-bot/paper` `{enabled}` | 켠 회원만 5분마다 돎(JobLock으로 서버 한 대). 거래소 공개 시세만 쓰고 키를 맡지 않음. 시세는 (거래소, 봉)마다 한 번 받아 함께 씀. 결과는 101 표에 계좌 `server-paper-<거래소>-spot-1h`로 남김. v2.2.0부터 회원마다 기준(코인·봉·손절·익절·한 번 금액)을 고르고, 원화·바이낸스 계좌 모두 회원의 기본 크레딧(1크레딧 = 1원)으로 사고팜(상점 크레딧은 쓰지 않음). v2.3.0부터 바이낸스 계좌는 기본 크레딧을 그때 환율(코빗 USDT-KRW)로 USDT로 바꿔 현물과 3배 선물(롱·숏)을 돌리고, 정리할 때 되돌림(원금은 기본 크레딧, 넘는 수익은 상점 크레딧). 한 번 금액 범위는 설정 `blog.trades.bot-budget` |
| 099 갱신 | 쇼츠 캐릭터·끝 광고 | 2.1.1, 2.2.0 | `shorts.mascot` (V50), `shorts.outro` (V51) | MCP `create_shorts_script`·`update_shorts_script`의 캐릭터 값 | 대본마다 캐릭터를 고름. 끝 3초 블로그 광고는 기본 켬 |
| 100 갱신 | 크레딧 모의투자 지정가 주문 | 2.2.0, 2.2.1 | `paper_limit_order` (V52) | `POST /api/me/paper/limit-orders`, `DELETE /api/me/paper/limit-orders/{id}`, `GET /api/me/paper/chart` | 주문할 때 금액을 지갑별(`locked_invest`·`locked_shop`)로 묶어 두고, 체결·취소 때 풂. 같은 회원의 `client_order_id`는 겹치지 않음(중복 주문 방지). v2.2.1부터 수량×가격·수량이 1조를 넘으면 거절하고 금액 계산이 넘치면 실패 |
| 매매 일지 갱신 | 봇 판단 흔적과 복기 | 2.3.0 | `trade` 판단 흔적 열 (V53) | MCP `record_trade`의 판단 흔적 값(모두 선택), `GET /api/me/trades`·`/{date}`와 MCP `get_trade_day`에 체결별 흔적과 기법 버전·셋업별 성적 | 같은 판단 키(`decision_id`)의 진입·정리 체결을 한 포지션으로 묶음. 이유는 2000자까지, 비밀값이 들어 있으면 남기지 않음. 뉴스 원문·AI 입력 같은 판단 원본은 봇 쪽 로컬 DB에만 둠 |
| 보안 후속 | MCP 관리자 도구 토큰 | 2.4.0 | `personal_access_token.admin_tools` (V54) | 토큰 만들기·앱 연결 동의에 "관리자 도구" 값 | MCP 문의 관리 도구(`list_inquiries`·`get_inquiry`·`update_inquiry`)는 관리자가 따로 켠 토큰·연결에서만 보임. 옛 토큰·연결은 꺼진 채로 남음. 같은 릴리스의 요청 본문 상한(기본 256KB, 글 1MB, MCP 4MB), 세션 30일 절대 만료, 이메일 가입 IP당 1시간 10계정, 네임스페이스 나가는 연결 기본 차단은 테이블을 바꾸지 않음 |
| 099 갱신 | 쇼츠 연출 다양화 | 2.5.0 | `shorts.look` (V55) | 렌더 워커 작업에 같은 회원의 최근 5편 연출을 함께 줌, `list_shorts`에 첫 장면 | 워커가 영상마다 룩·배경음을 골라 `룩:배경음`으로 적고, 최근 3편의 룩과 최근 조합을 피함 |

v2.1.2(쇼츠 영상 공개 경로 `shorts/*`)와 v2.1.3(모의투자 시세를 코빗 공개 시세로 받기)은 테이블을 바꾸지 않았습니다. 시세 출처는 업비트에서 코빗(`/v2/tickers`, `/v2/candles`, 키 없음)으로 바뀌었습니다.

## 2. ERD에 더해진 테이블·컬럼 (V47~V55)

| 테이블·컬럼 | 마이그레이션 | 핵심 |
|---|---|---|
| `shorts.replay` | V47 | jsonb NULL. 매매 재현 영상이면 고른 체결과 차트 구간 |
| `trade_bot_status` | V48 | 회원당 한 행. AI 필터 켜짐·모델·오늘 호출 수(≥ 0)·봇 버전·마지막 알림 시각 |
| `trade_bot_account` | V48 | 기본 키 (회원, 계좌). 거래소·시장(`SPOT`·`FUTURES`)·통화와 현금·포지션 |
| `trade_bot_event` | V48 | 종류 `FILL`·`AI`·`SUMMARY`, 방향 `BUY`·`SELL`·`LONG`·`SHORT`·`CLOSE`, 판단 `ALLOW`·`BLOCK`. (회원, `external_id`) 유일로 같은 알림을 두 번 쓰지 않음 |
| `trade_paper_bot` | V49 | 회원당 한 행, 켜짐(부분 인덱스), 규칙·종목별 마지막 처리 봉·하루 수익. 탈퇴 정리 때 지움 |
| `shorts.mascot` | V50 | varchar(20) NULL. 비우면 기본 캐릭터 |
| `shorts.outro` | V51 | boolean, 기본 true |
| `paper_limit_order` | V52 | 상태 `OPEN`·`FILLED`·`CANCELED`, `BUY`·`SELL`, 가격·수량 > 0, 묶은 금액 ≥ 0. 체결되면 `trade_id`를 남김 |
| `trade.strategy_version`·`decision_id`·`basket`·`regime`·`setup`·`trigger_name`·`news_score`·`news_flags`·`risk_pct`·`kelly_raw`·`kelly_used`·`ai_conf`·`equity_at_entry`·`stop_price`·`tp1_price`·`tp2_price`·`liquidation_price`·`ai_model`·`ai_cost`·`exit_reason`·`r_multiple`·`funding` | V53 | 모두 선택 값(옛 체결은 비어 있음). 바구니 `CORE`·`TRADING`, 정리 이유 `STOP`·`TP1`·`TP2`·`TRAIL`·`TIME`·`NEWS`·`MANUAL`, 비율 값 0~1, 청산가·펀딩비는 선물에만(CHECK). `reason`을 varchar(2000)으로 넓힘. (회원, `decision_id`) 부분 인덱스 |
| `personal_access_token.admin_tools` | V54 | boolean, 기본 false. 켠 토큰만 MCP 관리자 도구를 씀 |
| `shorts.look` | V55 | varchar(32) NULL. 워커가 고른 `룩:배경음`(예 `grape:2`) |

## 3. 다국어 범위 (2026-10-16 점검)

[65 §4](./65-devlog-extensions-088-096.md#4-다국어-범위-spec-081-기준-2026-10-14-점검)의 범위 그대로입니다. `/trade` 화면과 코인 이름(비트코인·이더리움·리플·솔라나·도지코인)도 4개 언어 사전을 거칩니다. 봇이 남긴 이유·Gemini 판단 문장은 봇이 쓴 그대로 보여 주고 번역하지 않습니다.
