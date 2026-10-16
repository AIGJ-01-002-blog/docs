# devlog 확장 설계 (spec 101~102, V47~V52)

작성 2026-10-16, 고침 2026-10-16(v2.2.1) · 매일 저장소·다국어 점검. [67 devlog 확장 설계](./67-devlog-extensions-092-100.md)(spec 092·097~100, V36~V46)에 이어, v2.1.0부터 공통 설계(테이블·API·구조·권한)를 바꾼 기능만 모았습니다. 기능의 세부 규칙과 화면은 코드 저장소의 기능 명세 `specs/NNN-*`이 기준입니다.

근거: [devlog CHANGELOG](https://github.com/AIGJ-01-002-blog/devlog/blob/develop/CHANGELOG.md) v2.1.0~v2.2.1(목표 v2.2.0 [#282](https://github.com/AIGJ-01-002-blog/devlog/issues/282)), 마이그레이션 V47~V52. v2.2.0 릴리스로 V52까지 main에 들어갔고, v2.2.1 뒤 새 마이그레이션은 없습니다.

## 1. 기능별로 바뀐 공통 설계

| spec | 기능 | 버전 | 테이블 | API | 권한·구조 |
|---|---|---|---|---|---|
| 099 갱신 | 매매 재현 숏폼 | 2.1.0 | `shorts.replay` (V47) | `POST /api/me/shorts/replay`, `POST /api/me/trades/{date}/shorts` | 매매 일지 체결(같은 거래소·종목 1~6개)로 대본을 만들고, 렌더 워커가 실시간 매매처럼 그림 |
| 101 | 자동매매 모의 화면 `/trade` | 2.1.0 | `trade_bot_status`, `trade_bot_account`, `trade_bot_event` (V48) | `GET /api/me/trade-bot`, `GET /api/me/trade-bot/charts`, MCP `report_trade_bot` | 봇이 알린 모의 계좌·체결·Gemini 판단·하루 요약을 보기만 함. 화면에 주문 버튼 없음. 모의 기록은 운영 매매 일지(`trade`)와 나눔 |
| 102 | 서버가 돌리는 모의 봇 | 2.1.1, 2.2.0 | `trade_paper_bot` (V49) | `GET·PUT /api/me/trade-bot/paper` `{enabled}` | 켠 회원만 5분마다 돎(JobLock으로 서버 한 대). 거래소 공개 시세만 쓰고 키를 맡지 않음. 시세는 (거래소, 봉)마다 한 번 받아 함께 씀. 결과는 101 표에 계좌 `server-paper-<거래소>-spot-1h`로 남김. v2.2.0부터 회원마다 기준(코인·봉·손절·익절·한 번 금액)을 고르고, 원화·바이낸스 계좌 모두 회원의 기본 크레딧(1크레딧 = 1원)으로 사고팜(상점 크레딧은 쓰지 않음) |
| 099 갱신 | 쇼츠 캐릭터·끝 광고 | 2.1.1, 2.2.0 | `shorts.mascot` (V50), `shorts.outro` (V51) | MCP `create_shorts_script`·`update_shorts_script`의 캐릭터 값 | 대본마다 캐릭터를 고름. 끝 3초 블로그 광고는 기본 켬 |
| 100 갱신 | 크레딧 모의투자 지정가 주문 | 2.2.0, 2.2.1 | `paper_limit_order` (V52) | `POST /api/me/paper/limit-orders`, `DELETE /api/me/paper/limit-orders/{id}`, `GET /api/me/paper/chart` | 주문할 때 금액을 지갑별(`locked_invest`·`locked_shop`)로 묶어 두고, 체결·취소 때 풂. 같은 회원의 `client_order_id`는 겹치지 않음(중복 주문 방지). v2.2.1부터 수량×가격·수량이 1조를 넘으면 거절하고 금액 계산이 넘치면 실패 |

v2.1.2(쇼츠 영상 공개 경로 `shorts/*`)와 v2.1.3(모의투자 시세를 코빗 공개 시세로 받기)은 테이블을 바꾸지 않았습니다. 시세 출처는 업비트에서 코빗(`/v2/tickers`, `/v2/candles`, 키 없음)으로 바뀌었습니다.

## 2. ERD에 더해진 테이블·컬럼 (V47~V52)

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

## 3. 다국어 범위 (2026-10-16 점검)

[65 §4](./65-devlog-extensions-088-096.md#4-다국어-범위-spec-081-기준-2026-10-14-점검)의 범위 그대로입니다. `/trade` 화면과 코인 이름(비트코인·이더리움·리플·솔라나·도지코인)도 4개 언어 사전을 거칩니다. 봇이 남긴 이유·Gemini 판단 문장은 봇이 쓴 그대로 보여 주고 번역하지 않습니다.
