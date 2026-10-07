# ERD·스키마 검증 보고서
기준 커밋: `f5cc270` (docs 01~45, 작업 트리 `dev`) · 통합 ERD: `erd/blog` = `0318059` (검증은 `15124ed`에서 했고, 두 커밋의 51·erd 파일 5개는 blob이 같다) · 모델: Claude Opus 5.5 · 검증한 문서: docs 01~13·20~25·30~34·40~45 (30개, 모두 끝까지), `erd/blog:docs/51-erd-unified.md`, `erd/blog:erd/` 4개 · 작성일 2026-10-06

> **인용 형식:** docs 01~45는 `dev`와 `erd/blog`가 같아서 `docs/xx.md:줄`로 적는다. 51과 erd 파일은 `erd/blog:경로:줄`로 적는다 (V1 줄 + 802 = 51 줄. 51:803-1501의 SQL 블록이 V1과 같다).

## 0. 작업 중에 바뀐 저장소 상태 (먼저 읽어 주세요)

| 시각 | 변화 | 내가 한 일 |
|---|---|---|
| 15:07:24 | `erd/blog`에 `15124ed "erd"` 커밋이 생겼다 (51·erd·검증 파일·`~/.cmuxterm` 포함) | — |
| 15:07:29 | 작업 트리가 `dev`(f5cc270)로 바뀌었다. 51과 `erd/`가 작업 트리에서 사라졌고, 읽는 도중 `verification/prompts/common.md`가 보강 절이 없는 HEAD판으로 바뀌었다 | 브랜치는 바꾸지 않았다. `git archive 15124ed`로 /tmp에 꺼내 검증했다. common.md는 15124ed판(보강 절 포함)을 따랐다 (총괄 지시와 같음) |
| 15:12 | 검증 지시문·보고서가 별도 작업 트리 `/Users/atgg_03_008/AIP-1/teamblog-verify`(브랜치 `verify/2026-10-06`)에 놓였다 | 총괄 지시대로 이 보고서는 원래 작업 트리 경로에 썼다. 다른 보고서와 한곳에 두려면 옮겨 주세요 |
| 15:15~15:16 | `erd/blog`가 `e9447fd` → `0318059`로 다시 쓰였다 ("검증 보고서와 ~/ 제거") | 51·erd 5개 파일의 blob이 15124ed와 같음을 확인했다 → 이 보고서의 결과는 그대로 유효하다. 단 `0318059:scripts/lib/extract.py`는 옛 정규식으로 돌아갔다. 제안 추출에는 고친 판(15124ed판 = `teamblog-verify` 작업 트리판)을 썼다 |

51·erd 파일 자체가 읽는 도중 바뀐 일은 없다 (blob: 51 `0b4a319`, V1 `8deaa55`, OPTION `7612b33`, import `a4ee072`, 체크리스트 `0e1c979`).

## 요약
- **높음 1건 / 중간 4건 / 낮음 6건**
- 한 줄 총평: **V1 자체는 건강하다** — 빈 PostgreSQL 18.6에 오류 없이 적용되고, 51이 말한 17개 테이블·139개 컬럼·33개 FK와 표·관계선·ERD Cloud 가져오기 파일이 카탈로그와 모두 맞으며, 담당자 제안도 빠짐없이 들어갔다. **문제는 V1과 문서가 만나는 곳에 있다.** ① V1의 공개 목록 부분 인덱스는 `hidden_at IS NULL`을 요구하는데 문서의 목록·검색·트렌딩 SQL과 06 R-2a에는 그 조건이 없어서, 문서대로 짜면 숨긴 글이 노출되고 인덱스를 못 쓴다 (홈 46ms → 0.17ms, 트렌딩 262ms → 5ms). ② 51이 새로 정한 것(friendship 기본 포함, `report.closed_at`, 생략된 ON DELETE = RESTRICT, Tier C 테이블의 V1 통합)은 01~45에 결정 기록이 없고, friendship 포함은 06 §6 마이그레이션을 깨뜨린다. ③ 사진 재연결과 숨긴 댓글 삭제·해제 두 경로에서 데이터가 틀어진다.

| 총괄이 물은 것 | 답 |
|---|---|
| V1 적용 | **성공.** PostgreSQL 18.6, `--single-transaction`, 오류 0 (CREATE TABLE 17, CREATE INDEX 33, ALTER 1, COMMENT 156). DB 소유자(비superuser) 계정이나 DB에 CREATE 권한이 있는 계정으로도 성공, 둘 다 아닌 계정은 `pg_trgm` 권한 오류 (부록 B, J-4) |
| 51의 17/139/33 | **모두 맞다.** PK 17·UNIQUE 9·CHECK 48·별도 인덱스 33(UNIQUE 인덱스 2)·GIN 4·timestamptz 40·`CURRENT_TIMESTAMP` 19·주석 누락 0도 51:776-777과 같다 |
| `check-ddl.sh` (V1 대입) | **PASS 49 / FAIL 4** — 테스트가 옛 스키마를 가정 2 (§5 태그·글-태그 중복), V1이 문서 규칙(06 §6)과 충돌 1 (§8 → M1), V1에 제안이 이미 들어 있어 생긴 예상된 실패 1 (§9). 실행된 "거부 기대" 검사 31개는 모두 의도한 제약으로 거부됨 (부록 C) |
| 보강 확인 | V1 새 규칙 보강 테스트 **66건 PASS** (문서는 막으라는데 DB가 막지 않는 5가지 확인 → L1), 탈퇴 30일 처리 재현, 목록·검색·배치 쿼리 **43개 EXPLAIN** (부록 D·E·F) |
| 이미 알려진 §9 실패 (00-scripts.txt 재실행) | **21은 교체형**(`CREATE TABLE comment`)이라 실패한 것이 맞다. 22도 교체형(`CREATE TABLE tag`·`post_tag`)이라 21이 없어도 같은 방식으로 실패하고, 21을 교체로 해석해 적용하면 43의 `ALTER TABLE comment ADD COLUMN hidden_at`이 21의 `hidden_at`과 겹쳐 실패한다 (부록 G) |

## 발견 사항

### [높음] H1. V1의 공개 목록 부분 인덱스는 `hidden_at IS NULL`을 전제하는데, 문서의 목록·검색·트렌딩 SQL과 06 R-2a 공용 조건에는 그 조건이 없다 — 문서대로 구현하면 숨긴 글이 노출되고 인덱스를 못 쓴다
- **위치:** erd/blog:erd/V1__common_schema.sql:255, :257 (`ix_post_feed`·`ix_post_blog` "… AND deleted_at IS NULL AND hidden_at IS NULL"), erd/blog:docs/51-erd-unified.md:749 ("공개 목록 인덱스 WHERE에 `hidden_at IS NULL`") ↔ docs/06-visibility.md:190 (R-2a "`deleted_at IS NULL`(휴지통 글 제외) + 작성자의 `withdrawn_at IS NULL`"), docs/03-erd.md:370, docs/10-post-list.md:198, docs/22-tag.md:161·202, docs/24-follow-feed.md:141, docs/32-trending.md:59, docs/33-search.md:89; docs/43-report-hide.md:117·245·269-271·285 ("전원 | 공용 조건(06 R-2a)에 `hidden_at IS NULL`을 넣는 것에 동의 … | 화요일")
- **문제:** 43은 공용 조건과 인덱스를 **함께** 바꾸자고 했는데(43:269 "06 §7 R-2a 공용 조건과 `canRead`에 '숨기지 않은 글(`hidden_at IS NULL`, 작성자 본인 제외)' 추가", 43:271 "03 `ix_post_feed`, `ix_post_blog` 조건에 `hidden_at IS NULL` 추가"), V1은 인덱스만 바꿨다. 06 R-2a와 위 7개 SQL은 여전히 `status·visibility·deleted_at`(+`withdrawn_at`)만 건다. PostgreSQL은 쿼리의 WHERE가 부분 인덱스의 조건을 함축할 때만 그 인덱스를 쓴다.
- **측정 (V1 + 글 5만·댓글 4만·좋아요 16만, 부록 F):** 문서 SQL 그대로 → 홈 Q1 **46.5ms** (Seq Scan post + Sort), 커서 없는 첫 페이지 39.4ms, 블로그 Q2 2.5ms(`ix_post_manage` + Sort), 태그 Q3 21.8ms, 피드 Q7 16.2ms, 트렌딩 Q20 **261.9ms**, 검색 최근창 Q22 24.6ms. 같은 쿼리에 `AND p.hidden_at IS NULL`만 더하면 각각 **0.17** / 0.19 / 0.05 / 0.73 / 0.22 / **5.2** / 1.6ms (`ix_post_feed`·`ix_post_blog` 사용). 같은 데이터에서 문서 SQL로 홈 목록에 나오는 **숨긴 글은 373개**(최신 100개 안에 2개).
- **영향:** (1) 관리자가 숨긴 글이 홈·블로그·태그·피드·검색·트렌딩에 그대로 나온다 — 43 H-6 "작성자 외에는 비공개 글과 똑같이 취급"(43:19)이 지켜지지 않아 운영 조치가 무력해진다. (2) 공개 목록이 모두 전체 스캔 + 정렬이 되어 글 수에 비례해 느려진다 (02 §6 기준 300ms는 글 1만 건 기준인데, 5만 건에서 트렌딩이 262ms). (3) 06 §6 친구 규격의 `ix_post_blog_friends`(06:148-149)에도 hidden 조건이 없어 친구 공개 적용자에게 같은 일이 생긴다. (4) 43:117의 "작성자 본인 제외"를 목록 조건에도 넣으면, 작성자가 보는 자기 블로그 목록에 숨긴 글이 나와 06 V-8("블로그 페이지는 남에게 보이는 모습 그대로", 06:19)과 어긋나고 그 쿼리는 `ix_post_blog`를 쓰지 못한다.
- **제안:** ① 06 R-2a를 "`deleted_at IS NULL` + `hidden_at IS NULL` + 작성자 `withdrawn_at IS NULL`"로 고친다. "작성자 본인 제외"는 `canRead`(상세)에만 두고 목록 조건에는 두지 않는다 (V-8 유지, 숨긴 글은 내 글 관리 배지로만 — 41 §3-2). ② 03 §4·10 §7·22 §5·§6·24 §5·32 §3-1·33 §4 SQL에 `AND p.hidden_at IS NULL`을 넣는다 (32는 43:284의 숨긴 댓글 제외 `c.hidden_at IS NULL`도). ③ 06 §6의 `ix_post_blog_friends` WHERE에 `hidden_at IS NULL`. ④ 공개 목록 통합 테스트에서 EXPLAIN으로 두 인덱스 사용을 확인한다 (아래 "문서 어디에도 없는 것" 1). 대안으로 V1 인덱스에서 hidden 조건을 빼면 SQL을 안 고쳐도 인덱스는 쓰지만 숨긴 글 노출은 그대로라 권하지 않는다.
- **담당:** 공통(03·06·10) · 강성찬(22·24) · 김민서(32·33) · 나민서(43, 결정 확정)
- **확신:** 확실. (01 보고서 [중간] 5번이 문서 쪽 불일치를 다뤘다. 여기서는 V1이 인덱스로 이 조건을 강제한 뒤의 실제 영향 — 노출 + 인덱스 미사용 — 을 측정해서 높음으로 올렸다.)

### [중간] M1. V1이 `friendship`을 공통 기준선에 넣고 Tier C 테이블을 V1 하나로 합쳤는데, 03·06·24·25는 여전히 "공통에 넣지 않음 / V2·V{n} 마이그레이션"을 지시한다 — 06 §6 규격은 V1 위에 그대로 적용되지 않는다
- **위치:** erd/blog:erd/V1__common_schema.sql:3, :504-520; erd/blog:docs/51-erd-unified.md:744; erd/blog:erd/ERDCLOUD-CHECKLIST.md:20·95 ↔ docs/03-erd.md:153 (E-18 "공통 스키마에는 넣지 않는다. 구현하는 사람은 06 문서 §6의 마이그레이션을 그대로 적용"), docs/06-visibility.md:12 (V-1), :119 ("공통 스키마 위에 마이그레이션 하나(`V{n}__friends.sql`)로 추가"), :132 (`CREATE TABLE friendship`); docs/01-common-requirements.md:131; docs/03-erd.md:391 ("Tier C에서 공통으로 확정되면 V2 마이그레이션으로 추가"); docs/24-follow-feed.md:208·211 (`-- V2__follow.sql`); docs/25-notification.md:239·242 (`-- V2__notification.sql`); docs/13-delete-withdraw.md:140 ("(친구 공개 규격 적용자)") · docs/44-withdraw.md:114 ("(친구 공개 적용자)")
- **문제:** V1은 `friendship`·`ix_friendship_b`를 넣고 FRIENDS CHECK·`ix_post_blog_friends`는 뺐다 (51:744). 06 §6 블록을 "그대로" V1 위에 실행하면 CHECK 교체 2개를 마친 뒤 `CREATE TABLE friendship`에서 `ERROR: relation "friendship" already exists`로 멈춘다 (부록 C §8, check-ddl의 FAIL 중 실제 문제). follow·notification·report도 V1에 들어갔으므로, 24·25가 적은 `V2__follow.sql`·`V2__notification.sql`을 따로 만들면 같은 테이블을 두 번 만든다. 이 통합 결정은 01 결정 기록과 03 어디에도 없다 (51:3 "회의 채택 변경 + 이번 통일 지시").
- **영향:** 친구 공개를 켜려는 사람이 06 문서대로 마이그레이션을 쓰면 실패한다 (Flyway는 한 트랜잭션이라 CHECK 교체까지 취소되고, 트랜잭션 없이 돌리면 CHECK만 바뀐 채 반쯤 적용된다). 13 §3-3 6번·44 order 60은 "(적용자)" 조건이 붙어 있지만 이제 모든 서비스에 테이블이 있다. (24·25가 둘 다 `V2`를 쓰는 번호 충돌은 05 보고서 M5가 다뤘다.)
- **제안:** ① 01 결정 기록에 "통합 V1 = 03 + Tier C(24·25·31·43) + `friendship` 테이블. `FRIENDS` 값은 선택 활성"을 적고, 03 E-18·§5·§6 표를 맞춘다. ② 06 §6-1을 "V1에 `friendship`이 있으므로 FRIENDS 적용자는 CHECK 교체 + `ix_post_blog_friends`(hidden 조건 포함)만 `V{n}__friends.sql`로"로 바꾼다. ③ 24 §10·25 §10의 `V2__…` 주석을 "V1에 포함"으로. ④ 13 §3-3 6번·44 order 60의 "(적용자)"를 지우고 항상 실행. 대안: `friendship`을 V1에서 빼고 06 §6을 그대로 둔다 (03 E-18 원안).
- **담당:** 공통(01·03·06·13) · 강성찬(24·25) · 나민서(44)
- **확신:** 확실

### [중간] M2. `post_image.image_id`가 `ON DELETE CASCADE`라서 아직 쓰이는 사진이 정리 배치에 지워져도 DB가 막지 못한다 — 05 §7 ⑥에는 "다시 연결하면 `detached_at`을 비운다", "다른 글에서 쓰면 기록하지 않는다"는 규칙이 없다
- **위치:** erd/blog:erd/V1__common_schema.sql:410 (`fk_post_image_image … ON DELETE CASCADE`), :117-118 (`fk_member_profile_image … ON DELETE RESTRICT`); docs/03-erd.md:341, :159 (E-13 "어떤 글에도 연결되지 않은 사진 … 을 정리 배치가 삭제"); docs/05-publish.md:137 ("⑥ post_image 동기화, 연결된 사진 status = ATTACHED, 빠진 사진 detached_at 기록"); docs/04-draft-and-image.md:324-325 ("ATTACHED였다가 연결이 끊긴 지(`detached_at`) 7일이 지난 사진 → … DB 행 삭제"); docs/13-delete-withdraw.md:80-82 (완전 삭제 때만 "다른 글에서도 쓰면 유지")
- **문제:** 같은 사진을 글 A·B에 쓰다가 A에서 빼면 ⑥이 `detached_at`을 기록하고(B가 쓰는지 확인하는 규칙이 없다), 7일 뒤 배치가 행을 지우면 CASCADE가 B의 `post_image`까지 지운다. 뺐다가 7일 안에 다시 넣은 사진도 `detached_at`이 남아 같은 일이 생긴다. 프로필 사진은 RESTRICT라서 같은 상황에서 삭제가 거부된다. 재현 (부록 J-1): 글 B가 쓰는 사진 → `deleted`, B의 연결 0행 / 프로필 사진 → `violates RESTRICT setting of foreign key constraint "fk_member_profile_image"`.
- **영향:** 발행된 글의 사진 파일이 저장소에서 지워져 깨진 이미지가 되고, 연결 행도 조용히 사라져 썸네일·정리 판단이 틀어진다.
- **제안:** ① 05 §7 ⑥·04 §4-4에 "연결할 때 `detached_at = NULL`", "다른 글(`post_image`)·프로필(`member.profile_image_id`)에서 쓰면 `detached_at`을 기록하지 않는다"를 적는다 (13 §2-5와 같은 조건). ② 정리 배치 조건에 `NOT EXISTS (SELECT 1 FROM post_image WHERE image_id = i.id)`를 넣는다. ③ 안전장치로 `fk_post_image_image`를 RESTRICT로 바꾼다 — 사진 행은 정리 배치만 지우므로 CASCADE가 필요한 경로가 없다 (글 완전 삭제는 `fk_post_image_post` CASCADE가 맡는다).
- **담당:** 공통(03·04·05) · 강성찬(23)
- **확신:** 확실 (재현)

### [중간] M3. 숨긴 댓글이 "삭제된 자리"가 되거나 그 반대일 때 `comment_count`가 실제 행 수와 어긋난다 (21 §8·§9와 43 §4-3을 문서대로 적용해 재현)
- **위치:** docs/21-comment.md:249-251 (삭제 "−1 (숨김 상태였으면 그대로)"), :270 ("숨기면 −1, 해제하면 +1"), :22 (CM-11 "삭제되지 않고 숨겨지지 않은 댓글"), :363 (§14-7 "항상 … 행 수와 같다"); docs/43-report-hide.md:138 ("`hidden_at` 등을 비우고 원래대로. 댓글 수 다시 더함"); docs/30-like.md:91-105 (좋아요에만 보정 배치)
- **문제:** 경우 1 — 답글 있는 최상위 댓글을 숨김(−1) → 작성자가 삭제(자리만, "숨김 상태였으면 그대로") → 관리자가 해제(+1): `comment_count` 2, 실제 1. 경우 2 — 작성자가 먼저 지워 자리만 남은 댓글을 (그 전에 접수된 신고로) 관리자가 숨김(−1): `comment_count` 0, 실제 1 (부록 J-2). 숨김·해제의 ±1이 `deleted_at`을 보지 않기 때문이다. V1의 `ck_post_counts`는 음수만 막는다.
- **영향:** 카드·글 상세의 "댓글 N"과 트렌딩 재료가 틀어지고 21 §14 7번 완료 기준이 깨진다. 좋아요와 달리 댓글 수에는 보정 배치가 없어 틀어진 값이 계속 남는다.
- **제안:** 21 §9·43 §4-3에 "숨김 −1 / 해제 +1은 `deleted_at IS NULL`인 댓글에만"을 적거나, "삭제된 자리는 숨김·해제 대상이 아니다 (43 §3 처리 화면에서 '대상 없음'으로 종료)"로 정한다. 30 §4-1처럼 `comment_count` 보정 배치(삭제·숨김이 아닌 행 수로 맞춤)를 21에 더한다.
- **담당:** 강성찬(21) · 나민서(43)
- **확신:** 확실 (재현)

### [중간] M4. V1의 `report.closed_at`은 01~45에 근거가 없고 기록 시점·대상 상태가 정해지지 않아, 처리된 신고 스냅샷 문제를 스키마로도 풀지 못한다 (탈퇴 30일 처리 재현 결과 포함)
- **위치:** erd/blog:erd/V1__common_schema.sql:4, :550; erd/blog:docs/51-erd-unified.md:426 ("닫힌 시각 (스냅샷 삭제 기준); 종료 시 기록, 스냅샷 30일 보관 | 이번 지시(H4)"), :750 ↔ docs/43-report-hide.md:148 ("완전 삭제 시 `report`는 `status = CLOSED_NO_TARGET`로 바뀌고 스냅샷은 30일 뒤 지운다"), :57, :94 ("대상이 이미 사라졌으면 '대상 없음'으로 자동 종료"); docs/44-withdraw.md:117 (order 80 "대기 신고 `CLOSED_NO_TARGET`"); docs/13-delete-withdraw.md:110 · docs/44-withdraw.md:35 ("30일이 지나면 글·…·완전히 삭제"); docs/25-notification.md:283-289 (`notification_mute`)
- **문제:** ① "H4"는 01~45·검증 지시문 어디에도 없는 지시다. "종료"가 `CLOSED_NO_TARGET`만인지 `HIDDEN`·`REJECTED`(처리됨)도 포함하는지, "스냅샷을 지운다"가 행 삭제인지 `snapshot_*`만 비우는 것인지 정해져 있지 않다 (행 삭제라면 V1의 `notification.report_id … ON DELETE SET NULL`이 쓰인다). 43 §3은 대상이 사라진 신고를 **관리자가 열 때** 자동 종료하므로, 아무도 열지 않으면 `closed_at`이 비어 있는 채로 남는다. ② 44 §4 순서대로 탈퇴 30일 처리를 V1에서 재현하면 (부록 E) 대기 신고는 `CLOSED_NO_TARGET` + `closed_at`이 되지만, 처리된(`HIDDEN`) 신고는 `closed_at` NULL로 탈퇴 회원의 본문 스냅샷("W 본문: 전화번호 010-…")을 그대로 가진다. 같은 재현에서 `notification_mute` 2행도 남는다 (정리 단계 없음). ③ `closed_at`·`target_author_id`에는 인덱스가 없어 두 정리 작업은 전체 스캔이다 (부록 F Q39·Q40).
- **영향:** 탈퇴 안내(13:110)와 달리 처리된 신고의 스냅샷(PRIVACY 신고라면 노출된 개인정보 자체)이 무기한 남는다. 03 보고서 [높음](처리된 신고 스냅샷)·05 보고서 M9와 같은 문제이고, V1의 `closed_at`만으로는 그 정리 배치를 만들 수 없다.
- **제안:** ① 43 §5에 `closed_at` 정의: "`HIDDEN`·`REJECTED`·`CLOSED_NO_TARGET`이 될 때 기록", "`closed_at` + 30일(팀이 정한 기간) 뒤 `snapshot_*`·`detail`을 NULL로 (행은 유지)". ② 13 §2-5 완전 삭제 트랜잭션과 44 order 80에서 대상 신고를 즉시 `CLOSED_NO_TARGET` + `closed_at`으로 (관리자 열람에 기대지 않음), 탈퇴 회원 콘텐츠의 처리된 신고도 스냅샷을 비운다. ③ 01 결정 기록에 H4(`closed_at`)를 남긴다. ④ 44 §4에 `notification_mute` 삭제 단계를 더한다 (03 보고서 [낮음]과 같음). ⑤ `CREATE INDEX ix_report_closed ON report (closed_at) WHERE closed_at IS NOT NULL`, `CREATE INDEX ix_report_target_author ON report (target_author_id) WHERE status = 'PENDING'`.
- **담당:** 나민서(43·44) · 공통(13)
- **확신:** 확실 (재현)

### [낮음] L1. 문서가 "필수·금지"라고 한 규칙 중 V1 CHECK가 막지 않는 것 (보강 테스트로 확인)
- **위치·근거 (부록 D):** (a) erd/blog:erd/V1__common_schema.sql:559 `ck_report_detail CHECK (reason <> 'OTHER' OR length(btrim(detail)) > 0)` ↔ docs/43-report-hide.md:15 ("기타만 200자 설명 필수") — `OTHER` + `detail NULL`이 통과한다 (51:758도 인정). (b) docs/43-report-hide.md:21·159 ("사유 필수") ↔ 정지 사유·기간 CHECK 없음 — `SUSPENDED` + 사유 NULL, `ACTIVE` + 옛 사유가 저장된다. (c) docs/43-report-hide.md:97 (`hidden_by` 기록) ↔ `hidden_at`·`hidden_by`·`hidden_reason`을 함께 두는 CHECK·사유 값 목록이 없다 — `hidden_by`만 있는 행이 저장된다. (d) docs/43-report-hide.md:25 (H-12 관리자 정지 금지)·docs/44-withdraw.md:19 (W-6 관리자 탈퇴 금지) ↔ DB 규칙 없음 — 관리자 `SUSPENDED`가 저장된다. (e) docs/05-publish.md:62 ("본문 필수") ↔ `ck_post_published`는 제목만 검사한다 (V1:247).
- **영향:** Service 버그나 운영자의 직접 SQL로 규칙을 어긴 행이 들어가도 DB가 막지 않는다. (a)는 CHECK 이름이 규칙을 지키는 것처럼 보여 오해를 부른다.
- **제안:** (a) `reason <> 'OTHER' OR (detail IS NOT NULL AND length(btrim(detail)) > 0)`. (b) `CHECK (status = 'SUSPENDED' OR (suspended_until IS NULL AND suspended_reason IS NULL))` + `CHECK (status <> 'SUSPENDED' OR suspended_reason IS NOT NULL)`. (c) `CHECK ((hidden_at IS NULL) = (hidden_by IS NULL) AND (hidden_at IS NULL) = (hidden_reason IS NULL))` + 사유 코드 CHECK (43에서 목록 확정, `post`·`comment` 둘 다). (d) `CHECK (NOT (role = 'ADMIN' AND status <> 'ACTIVE'))`. (e) 필요하면 `ck_post_published`에 `length(btrim(content_md)) > 0`. 모두 E-10 방식(CHECK 추가)이라 V1 확정 전에 넣기 쉽다.
- **담당:** 나민서(43·44) · 공통(05)
- **확신:** 확실

### [낮음] L2. 배치·정리 쿼리가 찾는 컬럼에 인덱스가 없다
- **위치:** docs/25-notification.md:213 (③ "`last_actor_id = 나`") ↔ V1에는 `fk_notification_last_actor`(V1:615)만 있고 인덱스 없음; docs/43-report-hide.md:242 ("`target_author_id` … 작성자 탈퇴 처리 때 찾기 쉽게") ↔ 인덱스 없음 (V1:554); docs/41-manage-posts.md:120 (탭별 개수 "`GROUP BY`"), docs/44-withdraw.md:49·109 (휴지통 포함 내 글 전부) ↔ `post`의 `author_id` 인덱스는 부분 인덱스 두 개(`ix_post_manage` WHERE deleted_at IS NULL, `ix_post_trash` WHERE deleted_at IS NOT NULL)뿐
- **측정 (부록 F):** Q18 Seq Scan notification 8.6ms(10만 행), Q39 Seq Scan report, Q27(41 탭별 개수) 30ms·Q28(44 내 글 전부) 28ms Seq Scan post(5만 행) — 부분 인덱스 두 개를 합쳐 쓰지 못한다.
- **영향:** 지금 규모에서는 작지만 글·알림 수에 비례해 커진다. 41 개수 쿼리는 내 글 관리 첫 화면마다 실행된다.
- **제안:** `CREATE INDEX ix_notification_last_actor ON notification (last_actor_id) WHERE group_key IS NULL`, `ix_report_target_author`(M4 ⑤), `CREATE INDEX ix_post_author ON post (author_id)` — 또는 41의 개수를 `deleted_at IS NULL`/`IS NOT NULL` 두 쿼리로 나눠 부분 인덱스를 쓰게 한다.
- **담당:** 강성찬(25) · 나민서(41·43·44) · 공통(03)
- **확신:** 확실

### [낮음] L3. "생략된 ON DELETE = RESTRICT"는 01·03에 기록이 없는 결정이다 (재현한 흐름에서는 동작 차이 없음)
- **위치:** erd/blog:docs/51-erd-unified.md:6 ("원문에서 생략된 삭제 동작은 팀 결정에 따라 `RESTRICT`로 쓴다"), :751; erd/blog:erd/ERDCLOUD-CHECKLIST.md:10 ↔ docs/03-erd.md:206·228 등 (옵션 없음 = PostgreSQL 기본 NO ACTION), docs/01-common-requirements.md 결정 기록(115-165) — 해당 행 없음
- **문제:** 문서 기준 스키마와 V1을 카탈로그로 비교하면 FK 21개가 NO ACTION → RESTRICT로 바뀌었다 (부록 H-2). RESTRICT는 문장 끝까지 미루지 않고 즉시 검사한다. 재현한 13 §2-5 완전 삭제·44 §4 탈퇴 처리·check-ddl·보강 테스트에서는 차이가 없었다 (회원·태그 행을 지우는 흐름이 없기 때문).
- **영향:** 지금은 없다. 나중에 회원·태그를 지우는 개인 확장이나 한 문장에서 부모와 자식을 함께 지우는 SQL을 쓰면, NO ACTION일 때 되던 것이 실패할 수 있다.
- **제안:** 01 결정 기록에 "FK 삭제 동작 미지정 = RESTRICT (이유)"를 남기거나, 이유가 없으면 PostgreSQL 기본(NO ACTION)으로 둔다.
- **담당:** 공통(01·03)
- **확신:** 확실

### [낮음] L4. `OPTION-post-split.sql`은 01 §3 원칙 1·03 E-6과 충돌하고, 적어 둔 "수정 필요 문서"가 일부뿐이다
- **위치:** erd/blog:erd/OPTION-post-split.sql:1-5 ("05·10·30·31 문서 수정 필요"), :74-87 (`post_content`로 `content_md`·`excerpt`·`thumbnail_url`·`edit_version`·`render_version` 이동), :105-113 (`post_stat`로 카운터 이동) ↔ docs/01-common-requirements.md:89 ("공통 ERD는 바꾸지 않고, 개인 확장은 테이블·컬럼을 추가만 한다"), docs/03-erd.md:142 (E-6 "목록 9개를 그릴 때 …"), docs/10-post-list.md:188
- **문제:** 공통 `post`의 컬럼을 다른 테이블로 옮기는 것은 "추가만" 원칙의 변경이다. 영향 문서도 05·10·30·31 외에 04 §2-4(`UPDATE post SET title, content_md, edit_version … WHERE edit_version < :version`이 두 테이블에 걸침), 12 §7-7(`render_version`과 `status`가 다른 테이블), 13 §2-5·§3-3(카운터 감소), 21 §5 ③·§11(`comment_count`), 22 §5·§6, 24 §5, 32 §3-1, 33 §4(`content_md ILIKE` → `post_content`), 41 §5, 03 §4가 있다. 파일 자체는 `member`만 있는 빈 DB에 오류 없이 적용된다 (부록 J-3).
- **영향:** 채택하면 위 문서의 SQL이 모두 바뀌고, 04 §2-4의 "버전 비교 + 덮어쓰기"를 한 행 UPDATE로 원자적으로 할 수 없다.
- **제안:** 파일 머리말의 영향 문서 목록을 위처럼 채우고 "채택하려면 01 §3 원칙 1 변경 결정이 먼저 필요"를 적는다. 51:754·체크리스트:95의 "적용하지 않는다"는 그대로 둔다.
- **담당:** 51·erd 작성자
- **확신:** 확실

### [낮음] L5. 자동 검증이 V1을 보지 않는다 — check-ddl은 여전히 03을 적용하고, V1을 넣으면 테스트 2개가 옛 스키마를 가정한다
- **위치:** scripts/check-ddl.sh:9 (`$EXTRACT ddl "$DOCS/03-erd.md" | qf`), :56·58 (`insert into post_tag select $P, id from tag` — `position` 없음), :85 (06 §6 블록), :92-96 (§9 제안); erd/blog:docs/51-erd-unified.md:790 ("`extract.py proposals`는 … 번호 없는 `## ERD 변경 제안`만 탐색 … 21~25의 번호 붙은 절도 현 스크립트에서는 누락")
- **문제:** 팀이 구현할 것은 V1인데 check-ddl은 03을 검증한다. V1으로 바꿔 돌리면 (부록 C) §5 "태그"는 `post_tag.position` NOT NULL 때문에, "글-태그 중복"은 같은 `psql -c` 안의 태그 INSERT가 함께 롤백돼 0행 INSERT가 성공해서 FAIL — 둘 다 테스트가 03의 2컬럼 `post_tag`를 가정한 것이다. §8은 M1의 실제 충돌, §9는 V1에 제안이 이미 들어 있어서 생긴 예상된 실패다. 51:790은 15:04에 고친 extract.py(번호 붙은 절도 읽음)와 다르다 — 그 수정은 지금 `verify/2026-10-06` 작업 트리에만 있고 `erd/blog`(0318059)의 extract.py는 옛 판이다.
- **영향:** V1을 고쳐도 회귀를 잡지 못하고, 그대로 V1에 돌리면 거짓 FAIL 2개가 섞인다.
- **제안:** check-ddl에 V1 모드(부록 C의 한 줄 교체)를 더하고, §5의 `post_tag` INSERT에 `position`을 넣고, §8을 "V1에서는 FRIENDS CHECK 교체만" 검사로, §9를 "V1 위에서는 ALTER형 제안만"으로 바꾼다. 51:790 문장을 갱신한다. (scripts 수정은 검증 팀 범위 밖이라 제안만)
- **담당:** scripts 작성자 · 51 작성자
- **확신:** 확실

### [낮음] L6. 51·체크리스트의 표기 오류와 그림 차이
- **위치:** erd/blog:docs/51-erd-unified.md:514 (`uq_member_nickname`의 컬럼 칸 "`lower(nickname`" — 닫는 괄호 없음); erd/blog:docs/51-erd-unified.md:14·222, erd/blog:erd/ERDCLOUD-CHECKLIST.md:24·94 (프로필 이미지 1:N, "근거 없는 UNIQUE나 1:1을 추가하지 않는다") ↔ docs/03-erd.md:25 (`IMAGE |o--o| MEMBER : "프로필 이미지"`, 1:1)
- **문제:** 03 그림은 1:1, 51·체크리스트는 1:N이다. DDL은 03·V1 모두 `profile_image_id`에 UNIQUE가 없고, 11 §5(11:145 "본인이 올렸고, `purpose = PROFILE`")와 업로더가 한 명뿐인 점 때문에 실제로는 1:1이 된다.
- **제안:** 오타를 고친다. 03 그림을 DB 기준(1:N)으로 고치거나, 1:1을 DB에서 보장하려면 `UNIQUE (profile_image_id)`를 더하고 체크리스트 94줄을 바꾼다.
- **담당:** 51 작성자 · 공통(03)
- **확신:** 확실

## 팀이 특히 알고 싶은 것 — ERD·스키마 관점의 답

| # | 질문 | 답 (근거) |
|---|---|---|
| 1 | 확장성 | 글이 늘 때 가장 큰 위험은 **쿼리 조건과 부분 인덱스 조건의 불일치**다 (H1: 5만 건에서 트렌딩 262ms). 그 밖에 개수·정리 쿼리의 전체 스캔(L2). 태그 목록은 hidden 조건이 있으면 `ix_post_feed`를 따라가며 `post_tag` PK를 찍어 0.7ms라 22 §5가 말한 비정규화는 아직 필요 없다. 개인 확장: V1이 모든 CHECK에 이름을 붙여 E-10(CHECK 교체)은 쉬워졌다. 하지만 "새 테이블을 만드는" 규격(06 §6)은 V1 포함 여부에 따라 깨지고(M1), 수직 분할안은 "추가만" 원칙과 충돌한다(L4) |
| 2 | 기능 충돌 | H1 (43 ↔ 06·03·10·22·24·32·33), M1 (06 ↔ V1), M3 (21 ↔ 43), 20:111 `ReportResolved.targetType`의 `MEMBER` ↔ V1 `ck_report_target`(POST·COMMENT, 43:279에서 이미 요청) |
| 3 | 배포 | Flyway 계정은 **DB 소유자이거나 DB에 CREATE 권한**이 있어야 `CREATE EXTENSION pg_trgm`이 된다 (J-4: 소유자 OK, 소유자 아님 + DB CREATE 권한 OK, 둘 다 없음 → `permission denied to create extension "pg_trgm"` — 33:204의 "권한 확인"에 대한 답). 24·25의 V2 파일을 따로 만들면 V1과 중복 생성(M1). DB 시간대가 정해져 있지 않다 (아래 4) |
| 4 | 인증·인가 | 판정에 필요한 컬럼(`role`·`status`·`email_verified_at`·`deleted_at`·`withdrawn_at`·`hidden_at`·`suspended_until`)은 V1에 모두 있다. 정지·관리자 규칙(사유 필수, 관리자 정지·탈퇴 금지)은 DB가 막지 않는다 (L1 b·d) |
| 5 | 고려하지 못한 것 | 아래 표 |

## 문서 어디에도 없는 것 (우리가 고려하지 못한 것)
| # | 주제 | 왜 필요한가 (없으면 생기는 일) | 제안 (어느 문서에 무엇을 추가) | 심각도 |
|---|---|---|---|---|
| 1 | 목록 SQL과 부분 인덱스 조건을 맞추는 규칙·검사 | 인덱스의 WHERE(V1:255·257)와 쿼리 조건이 한 조건이라도 다르면 PostgreSQL이 인덱스를 버리고 전체 스캔 + 정렬을 한다 (H1: 46ms → 0.17ms). 지금은 어떤 테스트도 이것을 보지 않아서 조건을 하나 바꾸면 조용히 느려진다 | 06 §7 또는 02 §6에 "공개 목록 조건은 `VisibilityFilter` 하나가 만들고, 통합 테스트(Testcontainers)에서 대표 목록 쿼리의 EXPLAIN에 `ix_post_feed`/`ix_post_blog`가 나오는지 확인한다" 추가 | 중간 |
| 2 | V1 이후 스키마 변경 제안의 형식 | 21·22는 `CREATE TABLE`(교체형)로 제안해 03 위에서 `relation already exists`로 실패했다 (부록 G). V1이 운영 DB에 들어간 뒤에는 데이터가 있는 테이블을 다시 만들 수 없으므로 교체형 제안은 적용할 길이 없다 | scripts/README "담당자 문서에 SQL을 쓸 때"와 03 §6에 "V1 확정 뒤 제안은 `ALTER`·`CREATE INDEX`(필요하면 `CONCURRENTLY`), 새 테이블은 새 마이그레이션 파일" 규칙 추가 (01 보고서도 21·22 형식 결정을 제안) | 중간 |
| 3 | FK·배치 조건 컬럼의 인덱스 원칙 | FK 33개 중 참조하는 쪽 인덱스가 없는 것(`last_actor_id`·`target_author_id`·`hidden_by`·`handled_by`·`reply_to_member_id`)이 있고, 탈퇴·정리 배치가 그 컬럼으로 찾는다 (L2). 원칙이 없어 새 테이블마다 빠질 수 있다 | 03 §2에 "FK 컬럼과 배치 WHERE 컬럼은 인덱스를 둔다. 두지 않으면 이유를 적는다" | 낮음 |
| 4 | DB 시간대 | `post_view_daily.view_date`는 "한국 시간 기준 날짜"(31:224)인데 postgres:18 기본 시간대는 `Etc/UTC`다 (부록 B). SQL에서 `current_date`로 날짜를 만들면(예: 31 §6 보관 배치) 한국 시간 0~9시에는 하루 어긋난다. 검증 스크립트는 `TZ=Asia/Seoul`로 띄워(scripts/lib/common.sh:19) 이 차이를 가린다 | 02 §2에 "DB·JDBC 시간대(예: UTC 고정)와, 한국 날짜는 앱이 계산해 넘긴다" 명시 | 낮음 |
| 5 | `comment_count` 보정 | 좋아요에는 매일 보정 배치가 있지만(30:91-105) 댓글 수에는 없어서, M3처럼 한 번 틀어지면 계속 틀린 채로 남는다 | 21에 30 §4-1과 같은 보정 배치(삭제·숨김이 아닌 행 수로 맞춤, 0건이 아니면 경고) 추가 | 낮음 |
| 6 | 처리된 신고 뒤 같은 사람의 재신고 | `uq_report (reporter_id, target_type, target_id)`(V1:556)는 상태와 상관없이 걸린다. `REJECTED` 처리 뒤 작성자가 글을 고쳐 문제 내용을 넣어도 같은 사람의 새 신고는 "이미 신고함 → 200"(42:157)으로 사라진다 | 43에서 결정: 재신고를 허용하면 `CREATE UNIQUE INDEX … WHERE status = 'PENDING'`(부분 UNIQUE)로 바꾸고, 허용하지 않으면 이유를 결정 기록에 적는다 | 낮음 |

## 확인했지만 문제 없던 것
- **V1 적용:** 빈 DB에 `--single-transaction`으로 성공 (PostgreSQL 18.6). DB 소유자(비superuser) 계정으로도 성공 — `pg_trgm` 1.6은 trusted 확장이다.
- **51의 수:** 17/139/33과 PK 17·UNIQUE 9·CHECK 48·별도 인덱스 33(UNIQUE 인덱스 2)·GIN 4·timestamptz 40·`CURRENT_TIMESTAMP` 19·`pg_trgm` 1.6·주석 누락 0 — 모두 맞다 (부록 B).
- **51 표 ↔ V1:** §2 컬럼표 139행(순서·타입·NULL·기본값·논리명·FK·ON DELETE), §3 제약·인덱스표 140개(이름·정의), §1 관계선 33개가 카탈로그와 모두 일치한다. 차이로 나온 것은 PostgreSQL이 `IN (…)`을 `= ANY (ARRAY[…])`로, `position`을 `"position"`으로 표시하는 것뿐이다 (부록 H-1).
- 51의 SQL 블록(51:803-1501)과 V1 파일은 바이트가 같고, 51:795의 `extract.py block` 명령 출력도 V1과 같다 (끝 줄바꿈 1개 차이).
- **`erdcloud-import.sql` ↔ V1:** 컬럼 139·DATETIME 40·FK 33(CONSTRAINT 32 + ALTER 1)·KEY 11. CHECK와 부분·표현식·GIN·INCLUDE 인덱스가 모두 메모나 KEY로 들어 있고 컬럼 순서·타입·NULL·기본값·주석이 같다 (부록 J-7). ERD Cloud에 실제로 가져오는 것은 **미실행** — 특히 MySQL식 문법에서 `text NOT NULL DEFAULT ''`(erd/blog:erd/erdcloud-import.sql:174·175·202)를 받아 주는지는 확인 못 함.
- **`ERDCLOUD-CHECKLIST.md`:** 체크박스 59 (A 13, B 12, C 31, 공동 3), 관계선 R01~R33 = V1 FK 33.
- **담당자 제안 반영:** "03 + `friendship` 테이블 + 담당자 제안 SQL(21·22는 교체로 해석) + 글로만 적힌 제안(33 GIN 4, 34 `ai_consent_at`, 43 인덱스 조건·`report` FK SET NULL·`comment` 숨김 컬럼)"으로 만든 문서 기준 스키마와 V1을 카탈로그로 비교하면 컬럼·제약 정의·인덱스 59개가 모두 같다. 차이는 `report.closed_at`(M4), FK 21개 RESTRICT(L3), 제약 이름 33개(자동 이름 → `fk_`/`ck_`), `now()` → `CURRENT_TIMESTAMP` 19개, 컬럼 순서(member·image·report)뿐이다 (부록 H-2).
- **check-ddl (V1):** PASS 49. 실행된 "거부 기대" 검사 31개(스크립트의 33개 중 §8에서 건너뛴 1개와 FAIL 1개 제외)가 모두 의도한 제약(`ck_member_handle`, `uq_member_nickname`, `ck_post_published`, `ck_tag_name`, `fk_member_profile_image` RESTRICT 등)으로 거부됐다 (부록 C 진단 실행).
- **보강 테스트 66건 (부록 D):** 태그 허용 7·거부 10, `post_tag` position UNIQUE·CHECK·PK, 22 §4 "DELETE 후 INSERT" 교체 성공 (UPDATE로 순서를 맞바꾸면 즉시 UNIQUE 검사로 실패하지만 22는 교체 방식이라 문제없음), 복합 부모 FK(다른 글 댓글을 부모로 → 거부), `ck_comment_reply_to`, 2단계 깊이는 DB가 허용 (21:411 "Service에서 검사"와 같음), `ck_comment_edited`, 썸네일 크기, 팔로우 자기·중복, 알림 group·result·type CHECK, 25 §4-2 묶음 upsert(두 번 → 같은 id), 안 읽은 묶음 UNIQUE, 운영 알림 끄기 거부, 신고 UNIQUE·자기 신고·`MEMBER` 거부·공백 설명 거부, `views > 0`·31 §5-2 누적 upsert, FRIENDS 저장 거부, 13 §2-5 완전 삭제 연쇄(댓글·좋아요·태그·사진 연결·작업본·일별 조회·알림·알림 행위자 모두 0), 신고 삭제 → 알림 `report_id` SET NULL.
- **탈퇴 30일 처리 (부록 E):** 13 §3-3을 44 §4 순서로, 13·21·30·25의 문서 SQL로 V1에서 재현했다. RESTRICT에 걸리는 단계가 없고, 처리 뒤 `comment_count`·`like_count`가 실제 행 수와 같으며, 남의 답글이 달린 댓글만 자리로 남고, 남의 답글의 `reply_to_member_id`는 껍데기 회원을 가리키고, 묶음 알림에서 탈퇴 회원만 빠지며 `actor_count`·`last_actor_id`가 다시 계산되고, 로그인 수단·팔로우·친구·좋아요가 0이 되고, 회원 행이 익명화된다(`handle`만 유지). 구현 주의: 25 §8 ②를 `WITH … DELETE … UPDATE` 한 문장으로 쓰면 UPDATE가 DELETE 전 스냅샷을 봐서 재계산이 안 된다 (내 첫 구현이 그랬다 — 문서 결함은 아님).
- **값 목록 (CHECK):** `post.status`·`visibility`, `member.status`·`role`·`default_visibility`, `image.status`·`purpose`·`content_type`, `auth_identity.provider`, `report.target_type`·`reason`(43 H-2의 6종)·`status`(43 §7 대응), `notification.type` 7종·`result` 2종, `notification_mute.type` 5종, `friendship.status`가 문서에 나온 값을 모두 담는다. (20:111의 `MEMBER`는 43:279에서 정리 요청 중.)
- **NULL 규칙:** 03·05·07·09·11·13·21~43의 필수/선택과 V1의 NOT NULL·기본값이 맞다 (예외는 L1).
- **이름 규칙:** V1의 UNIQUE·CHECK·FK는 모두 `uq_`/`ck_`/`fk_`, 인덱스는 `ix_`(UNIQUE 인덱스 2개는 `uq_`), PK는 `{테이블}_pkey`, 이름 없는 제약 0, 테이블 17개 모두 단수형. 03에서 이름이 없던 FK 14개와 `post_draft`의 CHECK에 이름이 붙었다 (부록 J-6).
- **비정규화 카운터 경로:** `like_count`(30 §4 증감·13 §3-3 ③·30 §4-1 보정), `comment_count`(21 CM-11: 작성·삭제·숨김·해제·탈퇴), `view_count`(31 §5-2, 더하기만), `notification.actor_count`(25 §4-2·§4-3·§8)가 모두 문서에 있다 (예외: M3).
- **Redis와 테이블의 경계:** 07(토큰·실패 횟수), 04(자동 저장 버퍼), 05(Idempotency), 21(중복 방지), 22(인기 태그 캐시), 23(하루 업로드 수), 30·43(요청 제한), 31(중복 판정·대기 집계), 32(트렌딩 스냅샷), 34(AI 캐시·공급자 상태)는 Redis에만 있고 V1에 대응 테이블이 없다. 테이블로 둔 것(`post_draft`, 날짜별 합계만 담는 `post_view_daily` — 03 E-11과 맞음, `notification`, `member.ai_consent_at`)도 문서와 같다. 참고: 25 NT-8·NT-9의 중복 방지는 `notification_actor` 행에 기대므로 사용자가 알림을 [×]로 지우면(25:66, CASCADE) 그 기록도 사라진다 — 작은 동작 차이.
- **인덱스가 의도대로 쓰인 쿼리:** 댓글 목록(`ix_comment_root`·`ix_comment_reply`), 32 댓글 작성자 하위 쿼리(`uq_comment_post_id` — 21이 `ix_comment_post`를 지웠지만 `UNIQUE (post_id, id)`가 대신함), 알림 목록·안 읽은 수, 팔로워 목록·수, 태그 자동완성(`varchar_pattern_ops`, DB 정렬 규칙 `en_US.utf8`에서 사용됨), 검색 인덱스 단계(trgm 3개), 사람 검색(trgm 2개), 내 글 관리 3탭, 휴지통·탈퇴 대상·사진 정리 배치, 일별 조회수 보관·합계. Q17·Q38의 Seq Scan은 샘플 분포 때문이다 (알림이 모두 120일 이내, 신고의 70%가 대기).

---

## 부록

### A. 실행 환경과 순서
- 저장소는 읽기만 했다. `git archive 15124ed docs erd scripts verification | tar -x -C /tmp/teamblog-verify-02/snap` 으로 꺼냈고, 이후 `erd/blog`(0318059)와 blob이 같음을 `git show erd/blog:… | cmp`로 확인했다.
- 컨테이너: `teamblog-verify-02-pg` (`docker run -d --name teamblog-verify-02-pg -e POSTGRES_PASSWORD=pw postgres:18`), check-ddl 복사본이 띄운 `teamblog-verify-02-ddl`, 진단 실행의 `teamblog-verify-02-ddl-diag` (둘 다 스크립트 종료 때 `docker rm -f -v`로 자동 삭제).
- `teamblog-verify-02-pg` 안에서 DB를 나눠 실험했다: `a1`~`a4`(부록 G), `v1`(카탈로그), `docbase`·`db03`(부록 H), `v1t`(부록 D), `v1purge`(부록 E), `v1perf`(부록 F), `v1img`·`v1cnt`·`opt`·`v1owner`·`v1noowner`(부록 J).
- 쓴 도구: 15124ed판 `scripts/lib/extract.py`(번호 붙은 절도 읽는 판), `scripts/check-ddl.sh`·`scripts/lib/common.sh`의 /tmp 복사본, 비교용 Python 스크립트(카탈로그 JSON 덤프·대조).

### B. V1 적용과 카탈로그 수
- 파일: `erd/blog:erd/V1__common_schema.sql` — blob `8deaa552a2d366c08b6df1d1fcb81b9824a3af36` (15124ed와 0318059에서 같음), sha256 `7f4fe3be31de4c7ef551323aab96073ea19c7d59c3cfbeb223ef71b944665a9e`, 699줄, 29486바이트
- 명령: `docker exec -i teamblog-verify-02-pg psql -X -U postgres -d v1 -v ON_ERROR_STOP=1 --single-transaction < V1__common_schema.sql` → exit 0, `ERROR` 0줄

```text
$ docker run -d --name teamblog-verify-02-pg -e POSTGRES_PASSWORD=pw postgres:18
$ docker exec teamblog-verify-02-pg psql -tAc 'select version()'
PostgreSQL 18.6 (Debian 18.6-1.pgdg13+2) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
$ docker exec -i teamblog-verify-02-pg psql -X -U postgres -d v1 -v ON_ERROR_STOP=1 --single-transaction < V1__common_schema.sql
exit code: 0 / 출력 줄 종류:
 156 COMMENT
  33 CREATE INDEX
  17 CREATE TABLE
   1 CREATE EXTENSION
   1 ALTER TABLE

-- 카탈로그 수
tables: 17
columns: 139
constraints (contype | count):
c | 48
f | 33
p | 17
u | 9
indexes total: 59  / 제약이 아닌 별도 인덱스: 33 (그중 UNIQUE 2, GIN 4)
timestamptz columns: 40 / CURRENT_TIMESTAMP 기본값: 19 / pg_trgm: 1.6
주석 없는 테이블: 0 / 주석 없는 컬럼: 0
테이블별 컬럼 수:
auth_identity | 9
comment | 12
follow | 3
friendship | 6
image | 14
member | 19
notification | 13
notification_actor | 3
notification_mute | 3
post | 23
post_draft | 6
post_image | 2
post_like | 3
post_tag | 3
post_view_daily | 3
report | 14
tag | 3
DB 시간대(기본 이미지): Etc/UTC
```

### C. check-ddl.sh — V1 대입 실행
복사본에서 바꾼 것은 두 가지다: 지시대로 03 DDL을 꺼내 적용하던 줄을 V1 파일 적용으로, 그리고 컨테이너 이름 규칙(`teamblog-verify-02-*`)을 지키려고 `common.sh` 복사본의 접두어를 바꿨다. 출력 첫 줄의 "(docs/03-erd.md)"는 스크립트의 안내 문구가 그대로 남은 것이고 실제로는 V1을 적용했다 ("테이블 17개").

```diff
--- snap/scripts/check-ddl.sh	2026-10-06 15:07:24
+++ run/scripts/check-ddl.sh	2026-10-06 15:18:11
@@ -6,7 +6,7 @@
 
 echo "== 1. 공통 스키마 적용 (docs/03-erd.md)"
 start_pg ddl
-$EXTRACT ddl "$DOCS/03-erd.md" | qf || { echo "스키마 적용 실패"; exit 1; }
+qf < "$ROOT/erd/V1__common_schema.sql" || { echo "스키마 적용 실패"; exit 1; }
 echo "   테이블 $(q "select count(*) from information_schema.tables where table_schema='public'")개"
 
 M="insert into member(handle,nickname,terms_agreed_at,privacy_agreed_at) values"
--- snap/scripts/lib/common.sh	2026-10-06 15:07:24
+++ run/scripts/lib/common.sh	2026-10-06 15:18:11
@@ -15,14 +15,14 @@
 trap cleanup EXIT
 
 start_pg() {   # start_pg <이름>
-  local name="teamblog-check-$1"; docker rm -f -v "$name" >/dev/null 2>&1
+  local name="teamblog-verify-02-$1"; docker rm -f -v "$name" >/dev/null 2>&1
   docker run -d --name "$name" -e POSTGRES_PASSWORD=pw -e TZ=Asia/Seoul "$PG_IMAGE" >/dev/null || { echo "PostgreSQL 컨테이너를 시작하지 못했습니다"; exit 2; }
   CONTAINERS+=("$name"); PG="$name"
   for _ in $(seq 1 60); do docker exec "$PG" pg_isready -U postgres >/dev/null 2>&1 && break; sleep 1; done
   sleep 1
 }
 start_redis() {
-  local name="teamblog-check-$1"; docker rm -f -v "$name" >/dev/null 2>&1
+  local name="teamblog-verify-02-$1"; docker rm -f -v "$name" >/dev/null 2>&1
   docker run -d --name "$name" "$REDIS_IMAGE" >/dev/null || { echo "Redis 컨테이너를 시작하지 못했습니다"; exit 2; }
   CONTAINERS+=("$name"); RD="$name"
   for _ in $(seq 1 30); do docker exec "$RD" redis-cli ping >/dev/null 2>&1 && break; sleep 1; done
```

실행: `cd /tmp/teamblog-verify-02/run && bash scripts/check-ddl.sh` (7초, exit 1)

```text
== 1. 공통 스키마 적용 (docs/03-erd.md)
   테이블 17개

== 2. 회원·로그인 (07·08·09)
PASS  약관 동의 없이 가입
PASS  회원 생성
PASS  대문자 블로그 주소
PASS  정하지 않은 접두어(xx-)
PASS  접두어가 아닌 하이픈
PASS  끝이 밑줄
PASS  블로그 주소 중복
PASS  닉네임 Kim
PASS  닉네임 대소문자만 다름 (kim)
PASS  닉네임 KIM2는 다른 닉네임
PASS  닉네임 숫자만
PASS  닉네임 자음만
PASS  닉네임 11자
PASS  이메일 가입
PASS  같은 이메일의 Google → 별도 계정
PASS  같은 이메일로 이메일 가입 두 번
PASS  한 계정에 로그인 수단 2개
PASS  이메일 가입인데 비밀번호 없음
PASS  이메일 가입 대문자 이메일
PASS  소개 201자

== 3. 글·발행·공개 범위 (05·06)
PASS  제목 없는 임시글
PASS  published_at 없이 발행
PASS  공개 발행인데 first_public_at 없음
PASS  공백뿐인 제목으로 발행
PASS  수정 시각이 발행보다 이름
PASS  본문 100,000자 초과
PASS  공통 스키마에 FRIENDS 공개 범위
PASS  공개 발행 글

== 4. 자동 저장 작업본 (04)
PASS  늦게 온 옛 버전이 작업본을 덮어쓰지 않음 (v5)
PASS  작업본이 있어도 독자는 발행본 (본문)

== 5. 태그·댓글·좋아요 (C-TAG·C-CMT·30)
FAIL  태그
      ERROR:  null value in column "position" of relation "post_tag" violates not-null constraint
DETAIL:  Failing row contains (8, 1, null).
PASS  정규화 안 된 태그
FAIL  글-태그 중복
      
PASS  댓글 + 답글
PASS  삭제되지 않은 빈 댓글
PASS  삭제된 댓글은 내용을 비울 수 있음
PASS  같은 사람 좋아요 두 번 → 1 (1)

== 6. 이미지 (04·10·11)
PASS  글 사진 + 썸네일
PASS  10MB 초과
PASS  SVG
PASS  알 수 없는 purpose
PASS  프로필 이미지 연결
PASS  연결된 프로필 이미지 삭제

== 7. 휴지통·탈퇴 (13)
PASS  활동 중인 회원의 닉네임 비우기
PASS  탈퇴 시각 없이 WITHDRAWN
PASS  활동 중인 회원에 deleted_at
PASS  탈퇴 신청 → 익명 처리
PASS  해제된 닉네임을 다른 사람이 사용
PASS  탈퇴한 사람의 블로그 주소 재사용
PASS  글 완전 삭제 → 연결 데이터 CASCADE
PASS  남은 댓글·좋아요·글-태그·작업본 (0/0/0/0)

== 8. 친구 공개 규격 (06 §6, 선택 구현)
ERROR:  relation "friendship" already exists
FAIL  친구 공개 규격 적용

== 9. 담당자 문서의 ERD 변경 제안 (20~49)
   대상: 21-comment.md
   대상: 22-tag.md
   대상: 23-image.md
   대상: 24-follow-feed.md
   대상: 25-notification.md
   대상: 31-view-count.md
   대상: 43-report-hide.md
ERROR:  relation "comment" already exists
FAIL  제안 SQL 적용

결과: PASS 49 / FAIL 4
```

**FAIL 4건 분류**

| FAIL | 원인 | 분류 |
|---|---|---|
| §5 태그 | `post_tag.position` NOT NULL (22 제안 반영) — 테스트가 03의 2컬럼 `post_tag`를 가정 | 테스트가 옛 스키마 가정 (L5) |
| §5 글-태그 중복 | 위 문장이 같은 `psql -c` 안에서 실패해 태그 INSERT도 롤백 → `select … where name='jpa'`가 0행 → INSERT 성공 | 테스트가 옛 스키마 가정 (L5) |
| §8 친구 공개 규격 적용 | `relation "friendship" already exists` — 06 §6 블록이 V1과 충돌 | **통합 ERD가 문서 규칙(06 §6·03 E-18)과 어긋남** (M1) |
| §9 제안 SQL 적용 | `relation "comment" already exists` — 21이 교체형이고, V1에 제안이 이미 들어 있음 | 예상된 결과 (V1은 제안을 이미 포함, 부록 H-2) |

**진단 실행** (같은 복사본에서 `check()`가 거부 사유도 찍게 바꾼 것, 거부를 기대한 검사만):

```text
PASS  약관 동의 없이 가입
      (거부 사유) ERROR:  null value in column "terms_agreed_at" of relation "member" violates not-null constraint
PASS  대문자 블로그 주소
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_handle"
PASS  정하지 않은 접두어(xx-)
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_handle"
PASS  접두어가 아닌 하이픈
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_handle"
PASS  끝이 밑줄
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_handle"
PASS  블로그 주소 중복
      (거부 사유) ERROR:  duplicate key value violates unique constraint "uq_member_handle"
PASS  닉네임 대소문자만 다름 (kim)
      (거부 사유) ERROR:  duplicate key value violates unique constraint "uq_member_nickname"
PASS  닉네임 숫자만
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_nickname"
PASS  닉네임 자음만
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_nickname"
PASS  닉네임 11자
      (거부 사유) ERROR:  value too long for type character varying(10)
PASS  같은 이메일로 이메일 가입 두 번
      (거부 사유) ERROR:  duplicate key value violates unique constraint "uq_auth_identity"
PASS  한 계정에 로그인 수단 2개
      (거부 사유) ERROR:  duplicate key value violates unique constraint "uq_auth_identity_member"
PASS  이메일 가입인데 비밀번호 없음
      (거부 사유) ERROR:  new row for relation "auth_identity" violates check constraint "ck_auth_password"
PASS  이메일 가입 대문자 이메일
      (거부 사유) ERROR:  new row for relation "auth_identity" violates check constraint "ck_auth_local_email"
PASS  소개 201자
      (거부 사유) ERROR:  value too long for type character varying(200)
PASS  published_at 없이 발행
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_published"
PASS  공개 발행인데 first_public_at 없음
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_public_at"
PASS  공백뿐인 제목으로 발행
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_published"
PASS  수정 시각이 발행보다 이름
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_edited_at"
PASS  본문 100,000자 초과
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_content"
PASS  공통 스키마에 FRIENDS 공개 범위
      (거부 사유) ERROR:  new row for relation "post" violates check constraint "ck_post_visibility"
PASS  정규화 안 된 태그
      (거부 사유) ERROR:  new row for relation "tag" violates check constraint "ck_tag_name"
PASS  삭제되지 않은 빈 댓글
      (거부 사유) ERROR:  new row for relation "comment" violates check constraint "ck_comment_content"
PASS  10MB 초과
      (거부 사유) ERROR:  new row for relation "image" violates check constraint "ck_image_size"
PASS  SVG
      (거부 사유) ERROR:  new row for relation "image" violates check constraint "ck_image_type"
PASS  알 수 없는 purpose
      (거부 사유) ERROR:  new row for relation "image" violates check constraint "ck_image_purpose"
PASS  연결된 프로필 이미지 삭제
      (거부 사유) ERROR:  update or delete on table "image" violates RESTRICT setting of foreign key constraint "fk_member_profile_image" on table "member"
PASS  활동 중인 회원의 닉네임 비우기
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_nickname_null"
PASS  탈퇴 시각 없이 WITHDRAWN
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_withdrawn"
PASS  활동 중인 회원에 deleted_at
      (거부 사유) ERROR:  new row for relation "member" violates check constraint "ck_member_deleted"
PASS  탈퇴한 사람의 블로그 주소 재사용
      (거부 사유) ERROR:  duplicate key value violates unique constraint "uq_member_handle"
```

### D. 보강 테스트 — V1에 새로 들어온 규칙 (66건)
스크립트: `/tmp/teamblog-verify-02/v1_extra.sh` (빈 DB `v1t`에 V1 적용 후). "ok"인데 규칙 위반을 저장한 줄이 L1의 근거다.

```text
== 22 태그 형식·순서
PASS  허용: c++                                    
PASS  허용: c#                                     
PASS  허용: node.js                                
PASS  허용: .net                                   
PASS  허용: 스프링-부트                       
PASS  허용: spring-boot                            
PASS  허용: 자바_기초                          
PASS  거부: ...                                    new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: ---                                    new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: #                                      new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: ㅋㅋ                                 new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: 🔥hot                                new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: a/b                                    new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: c@d                                    new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: Spring                                 new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: spring boot                            new row for relation "tag" violates check constraint "ck_tag_name"
PASS  거부: aaaaaaaaaaaaaaaaaaaa                   value too long for type character varying(30)
PASS  글-태그 position 0,1                        
PASS  같은 글 같은 position                     duplicate key value violates unique constraint "uq_post_tag_position"
PASS  position 100                                   new row for relation "post_tag" violates check constraint "ck_post_tag_position"
PASS  같은 글 같은 태그 두 번 (PK)          duplicate key value violates unique constraint "post_tag_pkey"
PASS  22 §4 교체 방식 (DELETE 후 INSERT로 순서 뒤집기) 
PASS  UPDATE로 순서 맞바꾸기 (UNIQUE 즉시 검사) duplicate key value violates unique constraint "uq_post_tag_position"
== 21 댓글 구조
PASS  같은 글 최상위에 답글                 
PASS  다른 글의 댓글을 부모로 (복합 FK)  insert or update on table "comment" violates foreign key constraint "fk_comment_parent"
PASS  부모 없이 reply_to_member_id               new row for relation "comment" violates check constraint "ck_comment_reply_to"
PASS  답글의 답글 대상 표시 (parent=최상위) 
PASS  2단계 깊이(답글을 부모로)는 DB가 허용 (21: Service 검사) 
PASS  updated_at < created_at                        new row for relation "comment" violates check constraint "ck_comment_edited"
PASS  숨김 + 숨긴 관리자                      
== 23 이미지 썸네일 크기
PASS  thumb_size_bytes 1MiB                          
PASS  thumb_size_bytes 1MiB+1                        new row for relation "image" violates check constraint "ck_image_thumb_size"
PASS  thumb_size_bytes 0                             new row for relation "image" violates check constraint "ck_image_thumb_size"
== 24 팔로우
PASS  자기 자신 팔로우                        new row for relation "follow" violates check constraint "ck_follow_self"
PASS  같은 팔로우 두 번 → RETURNING 1행+0행 실제 [1] / 기대 [1]
== 25 알림
PASS  LIKE인데 group_key 없음                    new row for relation "notification" violates check constraint "ck_notification_group"
PASS  COMMENT인데 group_key 있음                 new row for relation "notification" violates check constraint "ck_notification_group"
PASS  REPORT_RESOLVED인데 result 없음            new row for relation "notification" violates check constraint "ck_notification_result"
PASS  COMMENT에 result                              new row for relation "notification" violates check constraint "ck_notification_result"
PASS  친구 알림 종류(선택 구현, 비활성) new row for relation "notification" violates check constraint "ck_notification_type"
PASS  25 §4-2 묶음 찾기/만들기 두 번 → 같은 id 실제 [same] / 기대 [same]
PASS  읽은 뒤 새 묶음                          
PASS  안 읽은 같은 묶음 두 개               duplicate key value violates unique constraint "uq_notification_unread_group"
PASS  운영 알림은 끌 수 없음 (mute CHECK)   new row for relation "notification_mute" violates check constraint "ck_notification_mute_type"
== 43 신고·정지
PASS  글 신고                                     
PASS  같은 대상 다시 신고 (uq_report)        duplicate key value violates unique constraint "uq_report"
PASS  같은 번호의 댓글 신고는 별개       
PASS  자기 것 신고                              new row for relation "report" violates check constraint "ck_report_self"
PASS  MEMBER 신고 (20의 targetType과 다름)     new row for relation "report" violates check constraint "ck_report_target"
PASS  OTHER + 공백 설명                          new row for relation "report" violates check constraint "ck_report_detail"
PASS  OTHER + 설명 NULL (43은 '필수'라 했지만 DB는 통과) 
PASS  SUSPENDED인데 사유·기간 없음 (43 H-8 '사유 필수') 
PASS  ACTIVE인데 정지 사유가 남음           
PASS  hidden_by만 있고 hidden_at 없음           
PASS  관리자 정지 (43 H-12 금지 규칙은 DB에 없음) 
== 31 일별 조회수 / friendship
PASS  views 0                                        new row for relation "post_view_daily" violates check constraint "ck_post_view_daily_views"
PASS  31 §5-2 누적 upsert                         
PASS  누적 결과                                  실제 [5] / 기대 [5]
PASS  friendship 행 저장 (테이블은 V1에 있음) 
PASS  그러나 FRIENDS 글은 저장 불가         new row for relation "post" violates check constraint "ck_post_visibility"
== 삭제 연쇄 (13 §2-5 SQL 그대로)
PASS  완전 삭제 (13 §2-5 두 문장)            
PASS  남은 댓글·좋아요·태그·이미지연결·작업본·일별·알림·알림행위자 실제 [0/0/0/0/0/0/0/0] / 기대 [0/0/0/0/0/0/0/0]
PASS  그 글에 대한 신고 행 (FK 없음 → 남음, 43 §5는 Service가 닫음) 실제 [1건 status=PENDING] / 기대 [1건 status=PENDING]
PASS  사진 연결 해제 기록                    실제 [true] / 기대 [true]
PASS  신고 행 삭제 → 알림 report_id SET NULL 
PASS  REPORT_RESOLVED 알림은 남고 report_id NULL 실제 [1/null] / 기대 [1/null]

결과: PASS 66 / FAIL 0
```

### E. 탈퇴 30일 처리 재현 (13 §3-3 / 44 §4 순서, 문서 SQL)
준비 데이터: 탈퇴 회원 W의 글 2개(하나는 관리자 숨김)·사진·프로필, W가 남의 글에 쓴 댓글 5종(남의 답글 달린 최상위, 내 답글만 달린 최상위, 남의 최상위에 단 답글, 숨겨진 댓글, 대상 표시 답글), 좋아요, 팔로우 양방향, 친구, 받은 알림·묶음 알림·하나짜리 알림, `notification_mute`, W가 한 신고·W 글에 대한 대기 신고·처리된(HIDDEN) 신고.

실행한 SQL (`$W` = `(select id from member where handle='wdraw')`):

```sql
\set ON_ERROR_STOP 1
BEGIN;
-- 10 PostWithdrawalPurgeStep (13 §2-5를 내 글 전부에)
UPDATE image SET detached_at = now()
 WHERE id IN (SELECT image_id FROM post_image WHERE post_id IN (SELECT id FROM post WHERE author_id = (select id from member where handle='wdraw'))) AND detached_at IS NULL
   AND id NOT IN (SELECT image_id FROM post_image WHERE post_id NOT IN (SELECT id FROM post WHERE author_id = (select id from member where handle='wdraw')));
DELETE FROM post WHERE author_id = (select id from member where handle='wdraw');
-- 20 CommentWithdrawalPurgeStep (21 §11 SQL 그대로)
UPDATE post p SET comment_count = p.comment_count - x.n
FROM (SELECT post_id, count(*) AS n FROM comment
      WHERE author_id = (select id from member where handle='wdraw') AND deleted_at IS NULL AND hidden_at IS NULL GROUP BY post_id) x
WHERE p.id = x.post_id;
UPDATE comment c SET content = '', deleted_at = now()
WHERE c.author_id = (select id from member where handle='wdraw') AND c.parent_id IS NULL
  AND EXISTS (SELECT 1 FROM comment r WHERE r.parent_id = c.id AND r.author_id <> (select id from member where handle='wdraw'));
DELETE FROM comment WHERE author_id = (select id from member where handle='wdraw') AND deleted_at IS NULL;
DELETE FROM comment c WHERE c.parent_id IS NULL AND c.deleted_at IS NOT NULL
  AND NOT EXISTS (SELECT 1 FROM comment r WHERE r.parent_id = c.id);
-- 30 LikeWithdrawalPurgeStep (13 §3-3 ③: like_count 감소 후 삭제)
WITH del AS (DELETE FROM post_like WHERE member_id = (select id from member where handle='wdraw') RETURNING post_id)
UPDATE post p SET like_count = p.like_count - x.n FROM (SELECT post_id, count(*) n FROM del GROUP BY post_id) x WHERE p.id = x.post_id;
-- 40 ImageWithdrawalPurgeStep
UPDATE member SET profile_image_id = NULL WHERE id = (select id from member where handle='wdraw');
UPDATE image SET detached_at = now() WHERE uploader_id = (select id from member where handle='wdraw') AND detached_at IS NULL;
-- 50 AuthIdentity
DELETE FROM auth_identity WHERE member_id = (select id from member where handle='wdraw');
-- 60 Friendship
DELETE FROM friendship WHERE member_a_id = (select id from member where handle='wdraw') OR member_b_id = (select id from member where handle='wdraw');
-- 65 Follow (24 §6)
DELETE FROM follow WHERE follower_id = (select id from member where handle='wdraw') OR followee_id = (select id from member where handle='wdraw');
-- 70 Notification (25 §8 ①②③)
DELETE FROM notification WHERE receiver_id = (select id from member where handle='wdraw');
CREATE TEMP TABLE gone ON COMMIT DROP AS SELECT notification_id FROM notification_actor WHERE actor_id = (select id from member where handle='wdraw');
DELETE FROM notification_actor WHERE actor_id = (select id from member where handle='wdraw');
UPDATE notification n SET actor_count = (SELECT count(*) FROM notification_actor a WHERE a.notification_id = n.id),
       last_actor_id = (SELECT a.actor_id FROM notification_actor a WHERE a.notification_id = n.id ORDER BY a.created_at DESC LIMIT 1)
 WHERE n.id IN (SELECT notification_id FROM gone);
DELETE FROM notification WHERE group_key IS NOT NULL AND actor_count = 0;
DELETE FROM notification WHERE last_actor_id = (select id from member where handle='wdraw') AND group_key IS NULL;   -- ③ 하나짜리 알림만
-- 80 Report (43 §5: 내 콘텐츠의 대기 신고 → CLOSED_NO_TARGET)
UPDATE report SET status = 'CLOSED_NO_TARGET', closed_at = now() WHERE target_author_id = (select id from member where handle='wdraw') AND status = 'PENDING';
-- 90 Member 익명화 (13 §3-3 7 + 44: ai_consent_at, suspended_*)
UPDATE member SET nickname = NULL, bio = NULL, profile_image_url = NULL, nickname_changed_at = NULL,
       ai_consent_at = NULL, suspended_until = NULL, suspended_reason = NULL, deleted_at = now() WHERE id = (select id from member where handle='wdraw');
COMMIT;
```

출력:

```text
== 처리 전 알림
1 | wdraw | COMMENT | 1 | xx_1 |  | 
2 | xx_1 | COMMENT | 1 | wdraw |  | 
3 | xx_1 | LIKE | 2 | wdraw | LIKE:post:px1 | yy_1,wdraw
4 | xx_1 | FOLLOW | 1 | wdraw | FOLLOW | wdraw
== 탈퇴 처리 전: 글별 comment_count / 실제(삭제·숨김 아닌 행) / like_count / 실제
PW1 2 2 2 2
PW2 0 0 0 0
PX1 6 6 2 2
PZ1 2 2 1 1
== 44 §4 순서대로 한 트랜잭션 (문서 SQL)
   트랜잭션 성공 (FK RESTRICT에 걸린 단계 없음)
== 처리 후: 글별 comment_count / 실제 / like_count / 실제
PX1 2 2 1 1
PZ1 1 1 0 0
== 남은 댓글 (PX1·PZ1)
3 | (빈 내용) | t | wdraw | -
4 | Y가 CW1에(대상W) | f | yy_1 | wdraw
7 | CX1 | f | xx_1 | -
10 | CY1 | f | yy_1 | -
== 탈퇴 회원을 아직 가리키는 행
comment.author_id : 1
comment.reply_to_member_id : 1
report.reporter_id : 1
report.target_author_id : 2
notification_mute.member_id : 2
notification.last_actor_id : 0
notification_actor.actor_id : 0
image.uploader_id (detached) : 2
follow/friendship/post_like/auth : 0
== 남은 신고 스냅샷 (탈퇴 회원 콘텐츠)
CLOSED_NO_TARGET | t | PW1 | W 본문 (개인정보 포함 가능)
HIDDEN | f | PW2 | W 본문: 전화번호 010-…
== 남은 이미지 원래 파일 이름 (정리 배치 전)
wpost | 김민서_신분증.jpg | t
wprof | me.png | t
== 회원 행
wdraw | ∅ | WITHDRAWN | t | ∅ | ∅
== 알림 (X가 받은 것)
LIKE | 1 | yy_1 | LIKE:post:px1
```

### F. 목록·검색·배치 쿼리 EXPLAIN (V1 + 샘플 데이터)
데이터: 회원 3,000 (탈퇴 신청 40·익명 처리 20), 글 50,000 (공개 발행 34,762·숨김 443·휴지통 1,564·임시 7,525), 댓글 40,000 (최상위 20,000 + 답글 20,000), 좋아요 159,861, 글-태그 124,942, 태그 500, 사진 30,000, 팔로우 32,103, 알림 100,000 + 행위자 28,690, 신고 3,197, 일별 조회수 240,000, 알림 끄기 600. `ANALYZE` 후 `EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF, SUMMARY ON)`. "문서 그대로"는 문서의 SQL을 바꾸지 않은 것, "h"가 붙은 것은 `AND p.hidden_at IS NULL`만 더한 것이다. 문서에 SQL이 없는 쿼리는 문서의 조건·정렬로 만들었다 (표에 표시). 생성 SQL은 부록 I-4.

| # | 문서 위치 · 쿼리 | 실행 시간 | 사용한 인덱스 | Seq Scan | 정렬 노드 |
|---|---|---|---|---|---|
| Q1 | 03 §4·10 §7 홈 최신 글 (문서 그대로, 커서) | 46.537 ms | — | `member`, `post` | 있음 |
| Q1h | Q1 + p.hidden_at IS NULL (43이 공용 조건에 추가 요청) | 0.165 ms | `ix_post_feed`, `member_pkey` | — | — |
| Q1f | Q1 첫 페이지 (커서 없음, 문서 그대로) | 39.411 ms | — | `member`, `post` | 있음 |
| Q1fh | Q1f + hidden_at IS NULL | 0.189 ms | `ix_post_feed`, `member_pkey` | — | — |
| Q2 | 10 §5 개인 블로그 목록 (03 ix_post_blog 조건, 문서 그대로) | 2.503 ms | `ix_post_manage` | — | 있음 |
| Q2h | Q2 + hidden_at IS NULL | 0.052 ms | `ix_post_blog` | — | — |
| Q2c | 10 §5 블로그 공개 글 수 + hidden | 0.232 ms | `ix_post_blog` | — | — |
| Q3 | 22 §5 태그별 글 목록 (문서 그대로) | 21.830 ms | `ix_post_tag_tag`, `post_pkey` | `member` | 있음 |
| Q3h | Q3 + hidden_at IS NULL | 0.725 ms | `ix_post_feed`, `member_pkey`, `post_tag_pkey` | — | — |
| Q4 | 22 §6 전체 태그 상위 100 (문서 그대로) | 75.733 ms | — | `member`, `post`, `post_tag`, `tag` | 있음 |
| Q5 | 22 §7 자동완성 앞부분 일치 (varchar_pattern_ops) | 0.033 ms | `ix_tag_name_prefix` | — | 있음 |
| Q6 | 22 §8 블로그 안 태그와 글 수 | 4.513 ms | `ix_post_blog`, `post_tag_pkey` | `tag` | 있음 |
| Q7 | 24 §5 팔로잉 피드 (문서 그대로) | 16.244 ms | `follow_pkey`, `ix_post_manage`, `member_pkey` | — | 있음 |
| Q7h | Q7 + hidden_at IS NULL | 0.222 ms | `follow_pkey`, `ix_post_feed`, `member_pkey` | — | — |
| Q8 | 24 §2-2 팔로워 목록 (created_at DESC, 회원 ID DESC) | 0.192 ms | `ix_follow_followee`, `member_pkey` | — | 있음 |
| Q9 | 24 §5 팔로워 수 (문서 그대로) | 1.259 ms | `ix_follow_followee` | `member` | — |
| Q10 | 21 §6 ① 최상위 21개 (문서 그대로, 첫 페이지) | 0.052 ms | `ix_comment_root`, `member_pkey` | — | — |
| Q11 | 21 §6 ② 처음 답글 3개 + 답글 수 (문서 그대로) | 0.198 ms | `ix_comment_reply`, `ix_comment_root`, `member_pkey` | — | 있음 |
| Q12 | 21 §11 2-a 탈퇴 회원 댓글 수 집계 (author_id) | 0.225 ms | `ix_comment_author` | — | 있음 |
| Q13 | 32 §3-1 댓글 작성자 수 하위 쿼리의 접근 경로 (글 하나) | 0.105 ms | `uq_comment_post_id` | — | 있음 |
| Q14 | 25 §5 알림 목록 (updated_at DESC, id DESC) | 0.183 ms | `ix_notification_list` | — | — |
| Q15 | 25 §5 안 읽은 수 | 0.179 ms | `ix_notification_unread` | — | — |
| Q16 | 25 §4-1 NEW_POST 팔로워 전원 (SELECT 부분, 문서 그대로) | 0.698 ms | `ix_follow_followee` | `member`, `notification_mute` | — |
| Q17 | 25 §6 90일 정리 | 1.131 ms | — | `notification` | — |
| Q18 | 25 §8 ③ 탈퇴 회원이 행동한 하나짜리 알림 (last_actor_id) | 8.552 ms | — | `notification` | — |
| Q19 | 25 §8 ② 묶음에서 탈퇴 회원 행 (actor_id) | 0.229 ms | `ix_notification_actor_actor` | — | — |
| Q20 | 32 §3-1 트렌딩 집계 (문서 그대로) | 261.853 ms | `ix_post_manage`, `member_pkey`, `uq_comment_post_id` | — | 있음 |
| Q20h | Q20 + hidden_at IS NULL | 5.242 ms | `ix_post_feed`, `member_pkey`, `uq_comment_post_id` | — | 있음 |
| Q21 | 33 §4 ② 인덱스 단계 (문서 그대로, 드문 단어) | 1.120 ms | `ix_post_content_trgm`, `ix_post_tag_tag`, `ix_post_title_trgm`, `member_pkey`, `post_pkey` | `tag` | 있음 |
| Q22 | 33 §4 ① 최근창 3,000개 (문서에 SQL 없음 — 공용 조건 + 최신 3,000에서 찾기) | 24.558 ms | — | `member`, `post` | 있음 |
| Q22h | Q22 + hidden_at IS NULL | 1.597 ms | `ix_post_feed`, `member_pkey` | — | — |
| Q23 | 33 §5 사람 검색 (trigram) | 0.076 ms | `ix_member_handle_trgm`, `ix_member_nickname_trgm` | — | — |
| Q24 | 41 §5 임시글 탭 (updated_at DESC, id DESC, 21개) | 0.144 ms | `ix_post_manage` | — | 있음 |
| Q25 | 41 §5 발행 글 탭 + 비공개 필터 | 0.481 ms | `ix_post_manage` | — | 있음 |
| Q26 | 41 §5 휴지통 탭 | 0.158 ms | `ix_post_trash` | — | 있음 |
| Q27 | 41 §5 탭별 개수 (GROUP BY) | 30.063 ms | — | `post` | 있음 |
| Q28 | 44 §4 order 10 / 44 §2 내 글 전부(휴지통 포함) | 27.877 ms | — | `post` | — |
| Q29 | 13 §2-5 휴지통 30일 비우기 배치 | 0.415 ms | `ix_post_trash` | — | — |
| Q30 | 13 §3-3 탈퇴 30일 대상 | 0.041 ms | `ix_member_withdraw_purge` | — | — |
| Q31 | 04 §2-5 빈 임시글 정리 배치 | 65.729 ms | `ix_post_title_trgm` | — | — |
| Q32 | 12 §7-7 다시 렌더링 배치 (render_version < 현재) | 0.279 ms | `post_pkey` | — | — |
| Q33 | 04 §4-4 TEMP 사진 정리 | 2.300 ms | `ix_image_cleanup_temp` | — | — |
| Q34 | 04 §4-4 연결 끊긴 사진 정리 | 0.662 ms | `ix_image_cleanup_detached` | — | — |
| Q35 | 23 §3 사용량 합계 | 0.172 ms | `ix_image_uploader` | — | — |
| Q36 | 31 §6 일별 조회수 90일 보관 | 4.947 ms | `ix_post_view_daily_date` | — | — |
| Q37 | 31 §6 최근 7일 글별 합계 (점수식 B·통계) | 2.334 ms | `ix_post_view_daily_date` | — | 있음 |
| Q38 | 43 §3 관리자 대기 신고 목록 (대상별 묶음) | 2.354 ms | — | `report` | 있음 |
| Q39 | 43 §5·44 order 80 탈퇴 회원 콘텐츠 신고 닫기 (target_author_id) | 0.673 ms | — | `report` | — |
| Q40 | 43 §5 스냅샷 30일 뒤 삭제 (closed_at) | 0.839 ms | — | `report` | — |
| Q41 | 30 §4-1 좋아요 수 보정 배치 (문서 그대로) | 80.638 ms | `post_pkey` | `post`, `post_like` | — |
| Q42 | 30 §4·13 §3-3 ③ 탈퇴 회원 좋아요 (member_id) | 0.257 ms | `ix_post_like_member` | — | — |
| Q43 | 24 §6 탈퇴 회원 팔로우 양방향 | 0.846 ms | `ix_follow_followee`, `ix_follow_follower` | — | — |

파라미터: {"cursor": ["2026-10-06 06:25:19.10227+00", "35573"], "author": "1", "tag": "1", "post": "39224", "receiver": "1", "followee": "1"}

주요 실행 계획:

```text
##### Q1 — 03 §4·10 §7 홈 최신 글 (문서 그대로, 커서)
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=13053 read=17789
  ->  Sort (actual rows=10.00 loops=1)
        Sort Key: p.first_public_at DESC, p.id DESC
        Sort Method: top-N heapsort  Memory: 40kB
        Buffers: shared hit=13053 read=17789
        ->  Hash Join (actual rows=34778.00 loops=1)
              Hash Cond: (p.author_id = m.id)
              Buffers: shared hit=13047 read=17789
              ->  Seq Scan on post p (actual rows=35131.00 loops=1)
                    Filter: ((deleted_at IS NULL) AND ((status)::text = 'PUBLISHED'::text) AND ((visibility)::text = 'PUBLIC'::text) AND (ROW(first_public_at, id) < ROW('2026-10-06 06:25:19.10227+00'::timestamp with time zone, 35573)))
                    Rows Removed by Filter: 14869
                    Buffers: shared hit=12996 read=17789
              ->  Hash (actual rows=2940.00 loops=1)
                    Buckets: 4096  Batches: 1  Memory Usage: 216kB
                    Buffers: shared hit=51
                    ->  Seq Scan on member m (actual rows=2940.00 loops=1)
                          Filter: (withdrawn_at IS NULL)
                          Rows Removed by Filter: 60
                          Buffers: shared hit=51
Planning:
  Buffers: shared hit=390 read=1
Planning Time: 0.496 ms
Execution Time: 46.537 ms

##### Q1h — Q1 + p.hidden_at IS NULL (43이 공용 조건에 추가 요청)
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=35 read=6
  ->  Nested Loop (actual rows=10.00 loops=1)
        Buffers: shared hit=35 read=6
        ->  Index Scan using ix_post_feed on post p (actual rows=10.00 loops=1)
              Index Cond: (ROW(first_public_at, id) < ROW('2026-10-06 06:25:19.10227+00'::timestamp with time zone, 35573))
              Index Searches: 1
              Buffers: shared hit=11
        ->  Memoize (actual rows=1.00 loops=10)
              Cache Key: p.author_id
              Cache Mode: logical
              Hits: 0  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
              Buffers: shared hit=24 read=6
              ->  Index Scan using member_pkey on member m (actual rows=1.00 loops=10)
                    Index Cond: (id = p.author_id)
                    Filter: (withdrawn_at IS NULL)
                    Index Searches: 10
                    Buffers: shared hit=24 read=6
Planning:
  Buffers: shared hit=394
Planning Time: 0.453 ms
Execution Time: 0.165 ms

##### Q2 — 10 §5 개인 블로그 목록 (03 ix_post_blog 조건, 문서 그대로)
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=570 read=220
  ->  Sort (actual rows=10.00 loops=1)
        Sort Key: first_public_at DESC, id DESC
        Sort Method: top-N heapsort  Memory: 37kB
        Buffers: shared hit=570 read=220
        ->  Bitmap Heap Scan on post p (actual rows=660.00 loops=1)
              Recheck Cond: ((author_id = 1) AND ((status)::text = 'PUBLISHED'::text) AND (deleted_at IS NULL))
              Filter: ((visibility)::text = 'PUBLIC'::text)
              Rows Removed by Filter: 126
              Heap Blocks: exact=768
              Buffers: shared hit=564 read=220
              ->  Bitmap Index Scan on ix_post_manage (actual rows=788.00 loops=1)
                    Index Cond: ((author_id = 1) AND ((status)::text = 'PUBLISHED'::text))
                    Index Searches: 1
                    Buffers: shared hit=6 read=10
Planning:
  Buffers: shared hit=265
Planning Time: 0.320 ms
Execution Time: 2.503 ms

##### Q2h — Q2 + hidden_at IS NULL
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=17
  ->  Index Scan using ix_post_blog on post p (actual rows=10.00 loops=1)
        Index Cond: (author_id = 1)
        Index Searches: 1
        Buffers: shared hit=17
Planning:
  Buffers: shared hit=268
Planning Time: 0.285 ms
Execution Time: 0.052 ms

##### Q7 — 24 §5 팔로잉 피드 (문서 그대로)
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=7103 read=1664 dirtied=7
  ->  Sort (actual rows=10.00 loops=1)
        Sort Key: p.first_public_at DESC, p.id DESC
        Sort Method: top-N heapsort  Memory: 34kB
        Buffers: shared hit=7103 read=1664 dirtied=7
        ->  Nested Loop (actual rows=6659.00 loops=1)
              Join Filter: (f.followee_id = p.author_id)
              Buffers: shared hit=7100 read=1664 dirtied=7
              ->  Merge Join (actual rows=305.00 loops=1)
                    Merge Cond: (f.followee_id = m.id)
                    Buffers: shared hit=37 read=4
                    ->  Index Only Scan using follow_pkey on follow f (actual rows=305.00 loops=1)
                          Index Cond: (follower_id = 7)
                          Heap Fetches: 0
                          Index Searches: 1
                          Buffers: shared hit=1 read=4
                    ->  Index Scan using member_pkey on member m (actual rows=1577.00 loops=1)
                          Filter: (withdrawn_at IS NULL)
                          Index Searches: 1
                          Buffers: shared hit=36
              ->  Index Scan using ix_post_manage on post p (actual rows=21.83 loops=305)
                    Index Cond: ((author_id = m.id) AND ((status)::text = 'PUBLISHED'::text))
                    Filter: (((visibility)::text = 'PUBLIC'::text) AND (ROW(first_public_at, id) < ROW('2026-10-06 06:25:19.10227+00'::timestamp with time zone, 35573)))
                    Rows Removed by Filter: 4
                    Index Searches: 305
                    Buffers: shared hit=7063 read=1660 dirtied=7
Planning:
  Buffers: shared hit=463 read=9
Planning Time: 0.764 ms
Execution Time: 16.244 ms

##### Q7h — Q7 + hidden_at IS NULL
Limit (actual rows=10.00 loops=1)
  Buffers: shared hit=193 read=2
  ->  Nested Loop (actual rows=10.00 loops=1)
        Buffers: shared hit=193 read=2
        ->  Nested Loop (actual rows=10.00 loops=1)
              Buffers: shared hit=163 read=2
              ->  Index Scan using ix_post_feed on post p (actual rows=55.00 loops=1)
                    Index Cond: (ROW(first_public_at, id) < ROW('2026-10-06 06:25:19.10227+00'::timestamp with time zone, 35573))
                    Index Searches: 1
                    Buffers: shared hit=54 read=2
              ->  Memoize (actual rows=0.18 loops=55)
                    Cache Key: p.author_id
                    Cache Mode: logical
                    Hits: 1  Misses: 54  Evictions: 0  Overflows: 0  Memory Usage: 5kB
                    Buffers: shared hit=109
                    ->  Index Only Scan using follow_pkey on follow f (actual rows=0.19 loops=54)
                          Index Cond: ((follower_id = 7) AND (followee_id = p.author_id))
                          Heap Fetches: 0
                          Index Searches: 54
                          Buffers: shared hit=109
        ->  Memoize (actual rows=1.00 loops=10)
              Cache Key: p.author_id
              Cache Mode: logical
              Hits: 0  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
              Buffers: shared hit=30
              ->  Index Scan using member_pkey on member m (actual rows=1.00 loops=10)
                    Index Cond: (id = p.author_id)
                    Filter: (withdrawn_at IS NULL)
                    Index Searches: 10
                    Buffers: shared hit=30
Planning:
  Buffers: shared hit=475
Planning Time: 0.633 ms
Execution Time: 0.222 ms

##### Q20 — 32 §3-1 트렌딩 집계 (문서 그대로)
Limit (actual rows=100.00 loops=1)
  Buffers: shared hit=5741 read=15092 written=9
  ->  Sort (actual rows=100.00 loops=1)
        Sort Key: capped.score DESC, capped.first_public_at DESC, capped.id DESC
        Sort Method: top-N heapsort  Memory: 37kB
        Buffers: shared hit=5741 read=15092 written=9
        ->  Subquery Scan on capped (actual rows=359.00 loops=1)
              Buffers: shared hit=5735 read=15092 written=9
              ->  WindowAgg (actual rows=359.00 loops=1)
                    Window: w1 AS (PARTITION BY p.author_id ORDER BY ((((((3 * p.like_count) + (2 * (SubPlan 1))))::numeric + (0.1 * (p.view_count)::numeric)) / power(((EXTRACT(epoch FROM (now() - p.first_public_at)) / '3600'::numeric) + '2'::numeric), 1.5))), p.first_public_at, p.id ROWS UNBOUNDED PRECEDING)
                    Run Condition: (row_number() OVER w1 <= 3)
                    Storage: Memory  Maximum Storage: 17kB
                    Buffers: shared hit=5735 read=15092 written=9
                    ->  Incremental Sort (actual rows=364.00 loops=1)
                          Sort Key: p.author_id, ((((((3 * p.like_count) + (2 * (SubPlan 1))))::numeric + (0.1 * (p.view_count)::numeric)) / power(((EXTRACT(epoch FROM (now() - p.first_public_at)) / '3600'::numeric) + '2'::numeric), 1.5))) DESC, p.first_public_at DESC, p.id DESC
                          Presorted Key: p.author_id
                          Full-sort Groups: 12  Sort Method: quicksort  Average Memory: 26kB  Peak Memory: 26kB
                          Buffers: shared hit=5735 read=15092 written=9
                          ->  Merge Join (actual rows=364.00 loops=1)
                                Merge Cond: (m.id = p.author_id)
                                Buffers: shared hit=5735 read=15092 written=9
                                ->  Index Scan using member_pkey on member m (actual rows=2901.00 loops=1)
                                      Filter: (withdrawn_at IS NULL)
                                      Rows Removed by Filter: 60
                                      Index Searches: 1
                                      Buffers: shared hit=67
                                ->  Sort (actual rows=371.00 loops=1)
                                      Sort Key: p.author_id
                                      Sort Method: quicksort  Memory: 45kB
                                      Buffers: shared hit=4996 read=14704 written=9
                                      ->  Bitmap Heap Scan on post p (actual rows=371.00 loops=1)
                                            Recheck Cond: (((status)::text = 'PUBLISHED'::text) AND (deleted_at IS NULL))
                                            Filter: (((visibility)::text = 'PUBLIC'::text) AND (first_public_at >= (now() - '7 days'::interval)) AND ((like_count >= 1) OR ((SubPlan 2) >= 1)))
                                            Rows Removed by Filter: 40775
   … (이하 생략 — 하위 쿼리 계획, 전체는 Q20 실행 시간 줄 참조)
Execution Time: 261.853 ms

##### Q20h — Q20 + hidden_at IS NULL
Limit (actual rows=100.00 loops=1)
  Buffers: shared hit=1488 read=42
  ->  Sort (actual rows=100.00 loops=1)
        Sort Key: capped.score DESC, capped.first_public_at DESC, capped.id DESC
        Sort Method: top-N heapsort  Memory: 37kB
        Buffers: shared hit=1488 read=42
        ->  Subquery Scan on capped (actual rows=351.00 loops=1)
              Buffers: shared hit=1482 read=42
              ->  WindowAgg (actual rows=351.00 loops=1)
                    Window: w1 AS (PARTITION BY p.author_id ORDER BY ((((((3 * p.like_count) + (2 * (SubPlan 1))))::numeric + (0.1 * (p.view_count)::numeric)) / power(((EXTRACT(epoch FROM (now() - p.first_public_at)) / '3600'::numeric) + '2'::numeric), 1.5))), p.first_public_at, p.id ROWS UNBOUNDED PRECEDING)
                    Run Condition: (row_number() OVER w1 <= 3)
                    Storage: Memory  Maximum Storage: 17kB
                    Buffers: shared hit=1482 read=42
                    ->  Incremental Sort (actual rows=356.00 loops=1)
                          Sort Key: p.author_id, ((((((3 * p.like_count) + (2 * (SubPlan 1))))::numeric + (0.1 * (p.view_count)::numeric)) / power(((EXTRACT(epoch FROM (now() - p.first_public_at)) / '3600'::numeric) + '2'::numeric), 1.5))) DESC, p.first_public_at DESC, p.id DESC
                          Presorted Key: p.author_id
                          Full-sort Groups: 12  Sort Method: quicksort  Average Memory: 26kB  Peak Memory: 26kB
                          Buffers: shared hit=1482 read=42
                          ->  Merge Join (actual rows=356.00 loops=1)
                                Merge Cond: (m.id = p.author_id)
                                Buffers: shared hit=1482 read=42
                                ->  Index Scan using member_pkey on member m (actual rows=2901.00 loops=1)
                                      Filter: (withdrawn_at IS NULL)
                                      Rows Removed by Filter: 60
                                      Index Searches: 1
                                      Buffers: shared hit=67
                                ->  Sort (actual rows=363.00 loops=1)
                                      Sort Key: p.author_id
                                      Sort Method: quicksort  Memory: 44kB
                                      Buffers: shared hit=379 read=42
                                      ->  Bitmap Heap Scan on post p (actual rows=363.00 loops=1)
                                            Recheck Cond: ((first_public_at >= (now() - '7 days'::interval)) AND ((status)::text = 'PUBLISHED'::text) AND ((visibility)::text = 'PUBLIC'::text) AND (deleted_at IS NULL) AND (hidden_at IS NULL))
                                            Filter: ((like_count >= 1) OR ((SubPlan 2) >= 1))
                                            Rows Removed by Filter: 13
   … (이하 생략 — 하위 쿼리 계획, 전체는 Q20 실행 시간 줄 참조)
Execution Time: 5.242 ms

##### Q27 — 41 §5 탭별 개수 (GROUP BY)
GroupAggregate (actual rows=3.00 loops=1)
  Group Key: (CASE WHEN (deleted_at IS NOT NULL) THEN 'trash'::character varying ELSE status END)
  Buffers: shared hit=14698 read=16090
  ->  Sort (actual rows=950.00 loops=1)
        Sort Key: (CASE WHEN (deleted_at IS NOT NULL) THEN 'trash'::character varying ELSE status END)
        Sort Method: quicksort  Memory: 25kB
        Buffers: shared hit=14698 read=16090
        ->  Seq Scan on post (actual rows=950.00 loops=1)
              Filter: (author_id = 1)
              Rows Removed by Filter: 49050
              Buffers: shared hit=14695 read=16090
Planning:
  Buffers: shared hit=212
Planning Time: 0.264 ms
Execution Time: 30.063 ms

##### Q18 — 25 §8 ③ 탈퇴 회원이 행동한 하나짜리 알림 (last_actor_id)
Seq Scan on notification (actual rows=10.00 loops=1)
  Filter: ((group_key IS NULL) AND (last_actor_id = 1))
  Rows Removed by Filter: 99990
  Buffers: shared hit=157 read=1212
Planning:
  Buffers: shared hit=160
Planning Time: 0.163 ms
Execution Time: 8.552 ms

##### Q39 — 43 §5·44 order 80 탈퇴 회원 콘텐츠 신고 닫기 (target_author_id)
Seq Scan on report (actual rows=53.00 loops=1)
  Filter: ((target_author_id = 1) AND ((status)::text = 'PENDING'::text))
  Rows Removed by Filter: 3144
  Buffers: shared hit=417
Planning:
  Buffers: shared hit=117
Planning Time: 0.145 ms
Execution Time: 0.673 ms
```

### G. 제안 분류표와 03 + 06 §6 위 적용 결과

**분류표** (SQL은 15124ed판 `extract.py proposals docs`로 꺼낸 7개 문서 + 글로만 적힌 제안)

| 문서 | 제안 | 형태 | 분류 | 03 + 06 §6 위 적용 (부록 G 출력) | V1 반영 |
|---|---|---|---|---|---|
| 20 §8 | 없음 (24·25에서 제안) | — | — | — | — |
| 21 §15 | `comment` 재정의: `reply_to_member_id`, `hidden_at`, `UNIQUE (post_id, id)` + 복합 부모 FK, `ck_comment_reply_to`·`ck_comment_edited`, 인덱스 3개로 교체 | SQL `CREATE TABLE comment` | **교체형** | `relation "comment" already exists` | 반영 (`hidden_at` 1번만) |
| 21 §15 #5 | `updated_at`은 내용 수정 때만 (규칙) | 글 | 규칙 | — | 해당 없음 (`ck_comment_edited`) |
| 22 §11 | `tag` CHECK 강화, `ix_tag_name_prefix`, `post_tag.position` + `UNIQUE (post_id, position)` + CHECK | SQL `CREATE TABLE tag`·`post_tag` | **교체형** | `relation "tag" already exists` | 반영 |
| 23 §7 | `image.thumb_size_bytes` + CHECK | SQL `ALTER TABLE` | 추가형 | OK | 반영 |
| 24 §10 | `follow` + 인덱스 2 (`V2__follow.sql`) | SQL `CREATE TABLE` | 추가형 (새 테이블) | OK | 반영 (V2가 아니라 V1) |
| 25 §10 | `notification`·`notification_actor`·`notification_mute` + 인덱스 (`V2__notification.sql`) | SQL `CREATE TABLE` | 추가형 (새 테이블) | OK | 반영 (V1) |
| 25 §10 글 | 친구 규격 적용자는 알림 종류 CHECK에 `FRIEND_*` 추가 | 글 | CHECK 교체 (선택) | — | 의도적으로 미반영 |
| 25 §12·43 ERD 표 | `notification.report_id → report(id) ON DELETE SET NULL` | 글 | 추가형 | — | 반영 |
| 30 | 스키마 변경 없음 (03 §4에 취소 쿼리, E-6 문구 추가) | 글 | 문서 수정 | — | 해당 없음 |
| 31 | `post_view_daily` + 인덱스 | SQL `CREATE TABLE` | 추가형 | OK | 반영 |
| 31 글 | 03에 E-24 추가 | 글 | 문서 수정 | — | 03 미수정 |
| 32 | 스키마 변경 없음, `ix_comment_post_author`는 "지금은 넣지 않음" | 글 | 보류 | — | 의도적으로 미반영 |
| 33 | `pg_trgm` + GIN 4개 (SQL은 §6, 제안 절은 참조만) | 글(SQL은 절 밖) | 추가형 | 추출되지 않음 | 반영 |
| 34 | `member.ai_consent_at` | 글 | 추가형 | 추출되지 않음 | 반영 |
| 40·41·42·44·45 | 없음 (42는 43으로 미룸, 44는 익명화 때 `ai_consent_at`·`suspended_*` NULL) | — | — | — | — |
| 43 | `report` 테이블 + `ix_report_pending` | SQL `CREATE TABLE` | 추가형 | OK | 반영 (+ `closed_at`, M4) |
| 43 | `post`·`comment`에 `hidden_at`·`hidden_by`·`hidden_reason` | SQL `ALTER TABLE` | 추가형 (`comment.hidden_at`은 21과 중복) | 03 위 OK / 21 교체 후 `column "hidden_at" … already exists` → 같은 문장의 `hidden_by`·`hidden_reason`도 못 들어감 | 반영 (1번만) |
| 43 | `member.suspended_until`·`suspended_reason` | SQL `ALTER TABLE` | 추가형 | OK | 반영 |
| 43 글 | `ix_post_feed`·`ix_post_blog` WHERE에 `hidden_at IS NULL` | 글 | 인덱스 교체 (기존 결정 변경) | — | 반영 |
| 43 글 | 06 R-2a 공용 조건·`canRead`에 `hidden_at IS NULL` | 글 | 규칙 변경 (화요일 합의) | — | 인덱스에만 반영 → **H1** |
| 06 §6 | FRIENDS CHECK 교체 + `friendship` + `ix_friendship_b` + `ix_post_blog_friends` | SQL | CHECK 교체 + 추가형 | OK (03 위) | `friendship`·`ix_friendship_b`만 → **M1** |

**03 + 06 §6 위에 적용했을 때의 충돌** (실행 출력은 아래)
1. 21·22는 교체형이라 03 위에서 바로 실패한다 (A2: 각각 단독으로도 실패). 00-scripts.txt 재실행은 21에서 멈춰 22의 실패가 보이지 않았을 뿐이다.
2. 21을 "지우고 다시 만들기"로 해석하면 43의 `ALTER TABLE comment ADD COLUMN hidden_at …`이 `already exists`로 실패하고, 같은 문장이라 `hidden_by`·`hidden_reason`도 들어가지 않는다 (A4).
3. 25의 `notification.comment_id → comment(id)` 때문에 21의 교체는 25보다 먼저여야 한다 (뒤면 `DROP TABLE comment`가 `cannot drop table comment because other objects depend on it` / `constraint notification_comment_id_fkey on table notification depends on table comment`로 막힌다 — DB `a3`에서 확인). 문서 번호 순서(21 → 25)로는 문제없다.
4. 33·34·43 일부(인덱스 조건·FK)·25 report FK는 글로만 있어 기계적 통합에서 빠진다. V1은 이것들을 손으로 넣었다.
5. 24·25가 둘 다 `V2__…`라 따로 만들면 Flyway 번호가 겹친다 (05 보고서 M5).

```text
== A1. 03 DDL → 06 §6 → 제안 전부(파일 순서, check-ddl §9와 같은 방식: 비트랜잭션, 첫 오류에서 중단)
   03: OK
   06 §6: OK
   제안 전부: ERROR(rc=3): ERROR:  relation "comment" already exists 

== A2. 제안 하나씩 (각각 새 DB: 03 + 06 §6 위에 그 문서 제안만, 단일 트랜잭션)
   21: ERROR(rc=3): ERROR:  relation "comment" already exists 
   22: ERROR(rc=3): ERROR:  relation "tag" already exists 
   23: OK
   24: OK
   25: OK
   31: OK
   43: OK

== A3. 누적 적용 (03 + 06 §6, 이어서 문서 순서대로 각 문서를 단일 트랜잭션으로; 실패해도 다음 문서 계속)
   21: ERROR(rc=3): ERROR:  relation "comment" already exists 
   22: ERROR(rc=3): ERROR:  relation "tag" already exists 
   23: OK
   24: OK
   25: OK
   31: OK
   43: OK
   a3 테이블 수: 17

== A4. 교체형(21·22)을 '기존 테이블을 지우고 새로 만든다'로 해석해 누적 적용
   21_replace: OK
   22_replace: OK
   23: OK
   24: OK
   25: OK
   31: OK
   43: ERROR(rc=3): ERROR:  column "hidden_at" of relation "comment" already exists 
   43을 문장별로 나눠 적용해 어느 문장이 충돌하는지:
     [CREATE TABLE report (     id               bigint GENERATED …] OK
     [CREATE INDEX ix_report_pending ON report (target_type, targe…] OK
     [ALTER TABLE post    ADD COLUMN hidden_at timestamptz, ADD CO…] OK
     [ALTER TABLE comment ADD COLUMN hidden_at timestamptz, ADD CO…] ERROR(rc=3): ERROR:  column "hidden_at" of relation "comment" already exists 
     [ALTER TABLE member  ADD COLUMN suspended_until timestamptz, …] OK
   a4 comment 컬럼: id,post_id,author_id,parent_id,reply_to_member_id,content,created_at,updated_at,deleted_at,hidden_at
```

### H. 51 ↔ V1 ↔ 03 대조
**H-1. 51의 표·관계선 ↔ V1 카탈로그** (`cmp51.py`: 51 §2 컬럼표, §3 제약·인덱스표, §1 mermaid 관계선을 파싱해 카탈로그와 비교. 남은 "차이"는 모두 PostgreSQL의 표기 차이)

```text
[컬럼표] 51 행 139개, V1 컬럼 139개, 51에 없는 V1 컬럼: []
[제약표] 51 행 141개(확장 1 포함), V1 제약+인덱스 이름 140개
  V1에만 있는 이름: []
  51에만 있는 이름: []
[ERD 관계선] mermaid 33개, V1 FK 33개

[불일치]
 - 51:507 제약 ck_member_role 정의 차이
      51: CHECK (role IN ('USER', 'ADMIN'))
      V1: CHECK (((role) = ANY ((ARRAY['USER', 'ADMIN'])[])))
 - 51:508 제약 ck_member_status 정의 차이
      51: CHECK (status IN ('ACTIVE', 'SUSPENDED', 'WITHDRAWN'))
      V1: CHECK (((status) = ANY ((ARRAY['ACTIVE', 'SUSPENDED', 'WITHDRAWN'])[])))
 - 51:512 제약 ck_member_default_visibility 정의 차이
      51: CHECK (default_visibility IN ('PUBLIC', 'PRIVATE'))
      V1: CHECK (((default_visibility) = ANY ((ARRAY['PUBLIC', 'PRIVATE'])[])))
 - 51:528 제약 ck_image_type 정의 차이
      51: CHECK (content_type IN ('image/jpeg', 'image/png', 'image/gif', 'image/webp'))
      V1: CHECK (((content_type) = ANY ((ARRAY['image/jpeg', 'image/png', 'image/gif', 'image/webp'])[])))
 - 51:530 제약 ck_image_status 정의 차이
      51: CHECK (status IN ('TEMP', 'ATTACHED'))
      V1: CHECK (((status) = ANY ((ARRAY['TEMP', 'ATTACHED'])[])))
 - 51:531 제약 ck_image_purpose 정의 차이
      51: CHECK (purpose IN ('POST', 'PROFILE'))
      V1: CHECK (((purpose) = ANY ((ARRAY['POST', 'PROFILE'])[])))
 - 51:547 제약 ck_auth_provider 정의 차이
      51: CHECK (provider IN ('LOCAL', 'GITHUB', 'GOOGLE'))
      V1: CHECK (((provider) = ANY ((ARRAY['LOCAL', 'GITHUB', 'GOOGLE'])[])))
 - 51:569 제약 ck_post_status 정의 차이
      51: CHECK (status IN ('DRAFT', 'PUBLISHED'))
      V1: CHECK (((status) = ANY ((ARRAY['DRAFT', 'PUBLISHED'])[])))
 - 51:570 제약 ck_post_visibility 정의 차이
      51: CHECK (visibility IN ('PUBLIC', 'PRIVATE'))
      V1: CHECK (((visibility) = ANY ((ARRAY['PUBLIC', 'PRIVATE'])[])))
 - 51:621 제약 uq_post_tag_position 정의 차이
      51: UNIQUE (post_id, position)
      V1: UNIQUE (post_id, "position")
 - 51:622 제약 ck_post_tag_position 정의 차이
      51: CHECK (position >= 0 AND position < 100)
      V1: CHECK ((("position" >= 0) AND ("position" < 100)))
 - 51:676 제약 ck_friendship_requester 정의 차이
      51: CHECK (requested_by IN (member_a_id, member_b_id))
      V1: CHECK (((requested_by = member_a_id) OR (requested_by = member_b_id)))
 - 51:677 제약 ck_friendship_status 정의 차이
      51: CHECK (status IN ('PENDING', 'ACCEPTED'))
      V1: CHECK (((status) = ANY ((ARRAY['PENDING', 'ACCEPTED'])[])))
 - 51:691 제약 ck_report_target 정의 차이
      51: CHECK (target_type IN ('POST', 'COMMENT'))
      V1: CHECK (((target_type) = ANY ((ARRAY['POST', 'COMMENT'])[])))
 - 51:692 제약 ck_report_reason 정의 차이
      51: CHECK (reason IN ('SPAM', 'ABUSE', 'SEXUAL', 'PRIVACY', 'COPYRIGHT', 'OTHER'))
      V1: CHECK (((reason) = ANY ((ARRAY['SPAM', 'ABUSE', 'SEXUAL', 'PRIVACY', 'COPYRIGHT', 'OTHER'])[])))
 - 51:694 제약 ck_report_status 정의 차이
      51: CHECK (status IN ('PENDING', 'HIDDEN', 'REJECTED', 'CLOSED_NO_TARGET'))
      V1: CHECK (((status) = ANY ((ARRAY['PENDING', 'HIDDEN', 'REJECTED', 'CLOSED_NO_TARGET'])[])))
 - 51:708 제약 ck_notification_type 정의 차이
      51: CHECK (type IN ('COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST', 'REPORT_RESOLVED', 'CONTENT_HIDDEN'))
      V1: CHECK (((type) = ANY ((ARRAY['COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST', 'REPORT_RESOLVED', 'CONTENT_HIDDEN'])[])))
 - 51:709 제약 ck_notification_group 정의 차이
      51: CHECK ((type IN ('LIKE', 'FOLLOW')) = (group_key IS NOT NULL))
      V1: CHECK ((((type) = ANY ((ARRAY['LIKE', 'FOLLOW'])[])) = (group_key IS NOT NULL)))
 - 51:710 제약 ck_notification_result 정의 차이
      51: CHECK ((type = 'REPORT_RESOLVED') = (result IS NOT NULL) AND (result IS NULL OR result IN ('ACTION_TAKEN', 'NO_VIOLATION')))
      V1: CHECK (((((type) = 'REPORT_RESOLVED') = (result IS NOT NULL)) AND ((result IS NULL) OR ((result) = ANY ((ARRAY['ACTION_TAKEN', 'NO_VIOLATION'])[])))))
 - 51:737 제약 ck_notification_mute_type 정의 차이
      51: CHECK (type IN ('COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST'))
      V1: CHECK (((type) = ANY ((ARRAY['COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST'])[])))
```

**H-2. 문서 기준 스키마 ↔ V1** (`docbase` = 03 + `friendship` 테이블 + 제안 SQL(21·22 교체 해석, 43의 `comment` ALTER에서 `hidden_at` 제외) + 글로만 적힌 제안(부록 I-2). 이름을 무시하고 정의로 짝지음)

```text
== 컬럼
  V1에만: report.closed_at timestamp with time zone null=YES default=None
  (컬럼 순서가 다른 테이블: {'image': 7, 'member': 15, 'report': 1} )
  now() 기본값(문서) → CURRENT_TIMESTAMP(V1) 개수: 19
== 제약 (정의 기준으로 짝지음, 이름은 무시)
  이름이 바뀐 제약 (문서 → V1):
    auth_identity: auth_identity_member_id_fkey → fk_auth_identity_member
    comment: comment_author_id_fkey → fk_comment_author
    comment: comment_hidden_by_fkey → fk_comment_hidden_by
    comment: comment_post_id_fkey → fk_comment_post
    comment: comment_reply_to_member_id_fkey → fk_comment_reply_to_member
    follow: follow_followee_id_fkey → fk_follow_followee
    follow: follow_follower_id_fkey → fk_follow_follower
    friendship: friendship_member_a_id_fkey → fk_friendship_member_a
    friendship: friendship_member_b_id_fkey → fk_friendship_member_b
    image: image_uploader_id_fkey → fk_image_uploader
    notification: notification_comment_id_fkey → fk_notification_comment
    notification: notification_last_actor_id_fkey → fk_notification_last_actor
    notification: notification_post_id_fkey → fk_notification_post
    notification: notification_receiver_id_fkey → fk_notification_receiver
    notification: notification_report_id_fkey → fk_notification_report
    notification_actor: notification_actor_actor_id_fkey → fk_notification_actor_actor
    notification_actor: notification_actor_notification_id_fkey → fk_notification_actor_notification
    notification_mute: notification_mute_member_id_fkey → fk_notification_mute_member
    post: post_author_id_fkey → fk_post_author
    post: post_hidden_by_fkey → fk_post_hidden_by
    post_draft: post_draft_content_md_check → ck_post_draft_content
    post_draft: post_draft_post_id_fkey → fk_post_draft_post
    post_image: post_image_image_id_fkey → fk_post_image_image
    post_image: post_image_post_id_fkey → fk_post_image_post
    post_like: post_like_member_id_fkey → fk_post_like_member
    post_like: post_like_post_id_fkey → fk_post_like_post
    post_tag: post_tag_post_id_fkey → fk_post_tag_post
    post_tag: post_tag_tag_id_fkey → fk_post_tag_tag
    post_view_daily: post_view_daily_views_check → ck_post_view_daily_views
    post_view_daily: post_view_daily_post_id_fkey → fk_post_view_daily_post
    report: report_handled_by_fkey → fk_report_handled_by
    report: report_reporter_id_fkey → fk_report_reporter
    report: report_target_author_id_fkey → fk_report_target_author
  ON DELETE가 바뀐 FK (문서 → V1):
    auth_identity.fk_auth_identity_member: NO ACTION → RESTRICT
    comment.fk_comment_author: NO ACTION → RESTRICT
    comment.fk_comment_hidden_by: NO ACTION → RESTRICT
    comment.fk_comment_reply_to_member: NO ACTION → RESTRICT
    follow.fk_follow_followee: NO ACTION → RESTRICT
    follow.fk_follow_follower: NO ACTION → RESTRICT
    friendship.fk_friendship_member_a: NO ACTION → RESTRICT
    friendship.fk_friendship_member_b: NO ACTION → RESTRICT
    image.fk_image_uploader: NO ACTION → RESTRICT
    member.fk_member_profile_image: NO ACTION → RESTRICT
    notification.fk_notification_last_actor: NO ACTION → RESTRICT
    notification.fk_notification_receiver: NO ACTION → RESTRICT
    notification_actor.fk_notification_actor_actor: NO ACTION → RESTRICT
    notification_mute.fk_notification_mute_member: NO ACTION → RESTRICT
    post.fk_post_author: NO ACTION → RESTRICT
    post.fk_post_hidden_by: NO ACTION → RESTRICT
    post_like.fk_post_like_member: NO ACTION → RESTRICT
    post_tag.fk_post_tag_tag: NO ACTION → RESTRICT
    report.fk_report_handled_by: NO ACTION → RESTRICT
    report.fk_report_reporter: NO ACTION → RESTRICT
    report.fk_report_target_author: NO ACTION → RESTRICT
== 인덱스 (정의 기준, 이름 무시)
```

**H-3. 03 → V1 테이블별** (`db03` = 03 DDL만, `v1` = V1). `ix_post_feed`·`ix_post_blog`는 이름은 같고 WHERE에 `hidden_at IS NULL`이 더해졌다.

| 테이블 | 03 컬럼 | V1 컬럼 | V1에서 추가된 컬럼 | FK (V1 ON DELETE) | CHECK 03→V1 | 별도 인덱스 03→V1 |
|---|---|---|---|---|---|---|
| `auth_identity` | 9 | 9 | — | member_id→member RESTRICT | 3→3 | 1→1  |
| `comment` | 8 | 12 | reply_to_member_id, hidden_at, hidden_by, hidden_reason | author_id→member RESTRICT, hidden_by→member RESTRICT, post_id, parent_id→comment CASCADE, post_id→post CASCADE, reply_to_member_id→member RESTRICT | 2→4 | 2→3 +ix_comment_author +ix_comment_reply +ix_comment_root −ix_comment_parent −ix_comment_post |
| `follow` | — | 3 | (새 테이블) | followee_id→member RESTRICT, follower_id→member RESTRICT | 0→1 | 0→2 +ix_follow_followee +ix_follow_follower |
| `friendship` | — | 6 | (새 테이블) | member_a_id→member RESTRICT, member_b_id→member RESTRICT | 0→4 | 0→1 +ix_friendship_b |
| `image` | 13 | 14 | thumb_size_bytes | uploader_id→member RESTRICT | 5→6 | 3→3  |
| `member` | 16 | 19 | ai_consent_at, suspended_until, suspended_reason | profile_image_id→image RESTRICT | 9→9 | 2→4 +ix_member_handle_trgm +ix_member_nickname_trgm |
| `notification` | — | 13 | (새 테이블) | comment_id→comment CASCADE, last_actor_id→member RESTRICT, post_id→post CASCADE, receiver_id→member RESTRICT, report_id→report SET NULL | 0→4 | 0→6 +ix_notification_cleanup +ix_notification_comment +ix_notification_list +ix_notification_post +ix_notification_unread +uq_notification_unread_group |
| `notification_actor` | — | 3 | (새 테이블) | actor_id→member RESTRICT, notification_id→notification CASCADE | 0→0 | 0→1 +ix_notification_actor_actor |
| `notification_mute` | — | 3 | (새 테이블) | member_id→member RESTRICT | 0→1 | 0→0  |
| `post` | 20 | 23 | hidden_at, hidden_by, hidden_reason | author_id→member RESTRICT, hidden_by→member RESTRICT | 7→7 | 4→6 +ix_post_content_trgm +ix_post_title_trgm |
| `post_draft` | 6 | 6 | — | post_id→post CASCADE | 1→1 | 0→0  |
| `post_image` | 2 | 2 | — | image_id→image CASCADE, post_id→post CASCADE | 0→0 | 1→1  |
| `post_like` | 3 | 3 | — | member_id→member RESTRICT, post_id→post CASCADE | 0→0 | 1→1  |
| `post_tag` | 2 | 3 | position | post_id→post CASCADE, tag_id→tag RESTRICT | 0→1 | 1→1  |
| `post_view_daily` | — | 3 | (새 테이블) | post_id→post CASCADE | 0→1 | 0→1 +ix_post_view_daily_date |
| `report` | — | 14 | (새 테이블) | handled_by→member RESTRICT, reporter_id→member RESTRICT, target_author_id→member RESTRICT | 0→5 | 0→1 +ix_report_pending |
| `tag` | 3 | 3 | — | — | 1→1 | 0→1 +ix_tag_name_prefix |

03: 테이블 10 / 컬럼 82 / FK 14 / CHECK 28 / 인덱스 31
V1: 테이블 17 / 컬럼 139 / FK 33 / CHECK 48 / 인덱스 59

### I. 적용한 DDL
**I-1. V1:** `erd/blog:erd/V1__common_schema.sql` 그대로 (blob `8deaa552`, sha256 `7f4fe3be…a9e`, 699줄). 51:803-1501과 같다. 여기 다시 싣지 않는다.

**I-2. 문서 기준 스키마(`docbase`)를 만들 때 직접 쓴 DDL** — 03 DDL(`extract.py ddl docs/03-erd.md`) → 06 §6의 `friendship`·`ix_friendship_b` → 21·22 앞에 각각 `DROP TABLE comment;` / `DROP TABLE post_tag; DROP TABLE tag;` → 23·24·25·31 제안 → 43 제안(`ALTER TABLE comment …` 문장 제외) → 아래:

```sql
-- 글로만 적힌 제안 (SQL 블록 밖) 반영 — 문서 원문 그대로
-- 43 §ERD: comment 숨김 컬럼 3개 중 21이 이미 만든 hidden_at을 뺀 나머지
ALTER TABLE comment ADD COLUMN hidden_by bigint REFERENCES member (id), ADD COLUMN hidden_reason varchar(30);
-- 43 §ERD "공통 인덱스 영향": ix_post_feed, ix_post_blog의 WHERE에 hidden_at IS NULL 추가
DROP INDEX ix_post_feed; DROP INDEX ix_post_blog;
CREATE INDEX ix_post_feed ON post (first_public_at DESC, id DESC)
    WHERE status = 'PUBLISHED' AND visibility = 'PUBLIC' AND deleted_at IS NULL AND hidden_at IS NULL;
CREATE INDEX ix_post_blog ON post (author_id, first_public_at DESC, id DESC)
    WHERE status = 'PUBLISHED' AND visibility = 'PUBLIC' AND deleted_at IS NULL AND hidden_at IS NULL;
-- 43 §ERD 표 / 25 §12: notification.report_id → report(id) ON DELETE SET NULL
ALTER TABLE notification ADD FOREIGN KEY (report_id) REFERENCES report (id) ON DELETE SET NULL;
-- 33 §6 (ERD 변경 제안 절은 "§6" 참조만 함)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX ix_post_title_trgm      ON post   USING gin (title gin_trgm_ops);
CREATE INDEX ix_post_content_trgm    ON post   USING gin (content_md gin_trgm_ops);
CREATE INDEX ix_member_nickname_trgm ON member USING gin (nickname gin_trgm_ops);
CREATE INDEX ix_member_handle_trgm   ON member USING gin (handle gin_trgm_ops);
-- 34 §ERD: member.ai_consent_at
ALTER TABLE member ADD COLUMN ai_consent_at timestamptz;
```

**I-3. 추출한 담당자 제안 SQL** (`python3 scripts/lib/extract.py proposals docs`, 15124ed판, 161줄)

<details><summary>펼치기</summary>

```sql
-- from 21-comment.md
CREATE TABLE comment (
    id                 bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id            bigint        NOT NULL REFERENCES post (id) ON DELETE CASCADE,
    author_id          bigint        NOT NULL REFERENCES member (id),
    parent_id          bigint,
    reply_to_member_id bigint        REFERENCES member (id),            -- 추가 (CM-2)
    content            varchar(1000) NOT NULL,
    created_at         timestamptz   NOT NULL DEFAULT now(),
    updated_at         timestamptz   NOT NULL DEFAULT now(),            -- 내용 수정 때만 갱신
    deleted_at         timestamptz,
    hidden_at          timestamptz,                                      -- 추가 (CM-9, 나민서님과 확정)
    CONSTRAINT uq_comment_post_id   UNIQUE (post_id, id),                -- 추가
    CONSTRAINT fk_comment_parent    FOREIGN KEY (post_id, parent_id)     -- 변경: 같은 글 안에서만
        REFERENCES comment (post_id, id) ON DELETE CASCADE,
    CONSTRAINT ck_comment_content   CHECK (deleted_at IS NOT NULL OR length(btrim(content)) > 0),
    CONSTRAINT ck_comment_parent    CHECK (parent_id IS NULL OR parent_id <> id),
    CONSTRAINT ck_comment_reply_to  CHECK (reply_to_member_id IS NULL OR parent_id IS NOT NULL),  -- 추가
    CONSTRAINT ck_comment_edited    CHECK (updated_at >= created_at)                              -- 추가
);
-- 최상위 목록 (오래된 순, 커서)
CREATE INDEX ix_comment_root  ON comment (post_id, created_at, id) WHERE parent_id IS NULL;
-- 답글 목록
CREATE INDEX ix_comment_reply ON comment (parent_id, created_at, id) WHERE parent_id IS NOT NULL;
-- 탈퇴 정리·내 댓글
CREATE INDEX ix_comment_author ON comment (author_id);

-- from 22-tag.md
CREATE TABLE tag (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       varchar(30) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_tag_name UNIQUE (name),
    -- 변경: 허용 문자 + 한글·영문·숫자 1자 이상 (22 §2). 금칙어는 TagNormalizer에서
    CONSTRAINT ck_tag_name CHECK (name ~ '^[가-힣a-z0-9._+#-]{1,30}$' AND name ~ '[가-힣a-z0-9]')
);
CREATE INDEX ix_tag_name_prefix ON tag (name varchar_pattern_ops);   -- 추가

CREATE TABLE post_tag (
    post_id  bigint   NOT NULL REFERENCES post (id) ON DELETE CASCADE,
    tag_id   bigint   NOT NULL REFERENCES tag (id),
    position smallint NOT NULL,                                       -- 추가 (입력 순서, 0부터)
    PRIMARY KEY (post_id, tag_id),
    CONSTRAINT uq_post_tag_position UNIQUE (post_id, position),       -- 추가
    CONSTRAINT ck_post_tag_position CHECK (position >= 0 AND position < 100)
);
CREATE INDEX ix_post_tag_tag ON post_tag (tag_id, post_id);

-- from 23-image.md
ALTER TABLE image ADD COLUMN thumb_size_bytes integer
    CONSTRAINT ck_image_thumb_size CHECK (thumb_size_bytes IS NULL OR (thumb_size_bytes > 0 AND thumb_size_bytes <= 1048576));

-- from 24-follow-feed.md
-- V2__follow.sql
CREATE TABLE follow (
    follower_id bigint      NOT NULL REFERENCES member (id),
    followee_id bigint      NOT NULL REFERENCES member (id),
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id),
    CONSTRAINT ck_follow_self CHECK (follower_id <> followee_id)
);
-- 팔로워 목록·수 (최근 순)
CREATE INDEX ix_follow_followee ON follow (followee_id, created_at DESC);
-- 팔로잉 목록 (최근 순). 피드·팔로우 여부 확인은 PK(follower_id, followee_id)
CREATE INDEX ix_follow_follower ON follow (follower_id, created_at DESC);

-- from 25-notification.md
-- V2__notification.sql
CREATE TABLE notification (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    receiver_id   bigint       NOT NULL REFERENCES member (id),
    type          varchar(30)  NOT NULL,
    post_id       bigint       REFERENCES post (id)    ON DELETE CASCADE,
    comment_id    bigint       REFERENCES comment (id) ON DELETE CASCADE,
    report_id     bigint,                 -- 나민서 report 테이블이 확정되면 FK 추가
    result        varchar(20),            -- REPORT_RESOLVED 결과
    last_actor_id bigint       REFERENCES member (id),
    actor_count   integer      NOT NULL DEFAULT 0,
    group_key     varchar(100),           -- 묶는 알림만 (LIKE, FOLLOW)
    read_at       timestamptz,
    created_at    timestamptz  NOT NULL DEFAULT now(),
    updated_at    timestamptz  NOT NULL DEFAULT now(),
    CONSTRAINT ck_notification_type   CHECK (type IN ('COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST',
                                                      'REPORT_RESOLVED', 'CONTENT_HIDDEN')),
    CONSTRAINT ck_notification_group  CHECK ((type IN ('LIKE', 'FOLLOW')) = (group_key IS NOT NULL)),
    CONSTRAINT ck_notification_result CHECK ((type = 'REPORT_RESOLVED') = (result IS NOT NULL)
                                             AND (result IS NULL OR result IN ('ACTION_TAKEN', 'NO_VIOLATION'))),
    CONSTRAINT ck_notification_count  CHECK (actor_count >= 0)
);
-- 안 읽은 묶음은 받는 사람·종류마다 하나
CREATE UNIQUE INDEX uq_notification_unread_group ON notification (receiver_id, group_key)
    WHERE read_at IS NULL AND group_key IS NOT NULL;
CREATE INDEX ix_notification_list    ON notification (receiver_id, updated_at DESC, id DESC);
CREATE INDEX ix_notification_unread  ON notification (receiver_id) WHERE read_at IS NULL;
CREATE INDEX ix_notification_post    ON notification (post_id)    WHERE post_id IS NOT NULL;     -- CASCADE 속도
CREATE INDEX ix_notification_comment ON notification (comment_id) WHERE comment_id IS NOT NULL;  -- CASCADE 속도
CREATE INDEX ix_notification_cleanup ON notification (updated_at);

-- 묶는 알림에 들어간 사람
CREATE TABLE notification_actor (
    notification_id bigint      NOT NULL REFERENCES notification (id) ON DELETE CASCADE,
    actor_id        bigint      NOT NULL REFERENCES member (id),
    created_at      timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (notification_id, actor_id)
);
CREATE INDEX ix_notification_actor_actor ON notification_actor (actor_id, created_at DESC);

-- 끈 알림 종류 (행이 없으면 켜짐)
CREATE TABLE notification_mute (
    member_id  bigint      NOT NULL REFERENCES member (id),
    type       varchar(30) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (member_id, type),
    CONSTRAINT ck_notification_mute_type CHECK (type IN ('COMMENT', 'REPLY', 'LIKE', 'FOLLOW', 'NEW_POST'))
);

-- from 31-view-count.md
-- 일별 조회수 (통계·트렌딩 점수식 B 대비, 90일 보관) — 31 문서
CREATE TABLE post_view_daily (
    post_id   bigint  NOT NULL REFERENCES post (id) ON DELETE CASCADE,
    view_date date    NOT NULL,                 -- 한국 시간 기준 날짜
    views     integer NOT NULL CHECK (views > 0),
    PRIMARY KEY (post_id, view_date)
);
CREATE INDEX ix_post_view_daily_date ON post_view_daily (view_date, post_id) INCLUDE (views);

-- from 43-report-hide.md
-- 03 §5의 report 자리를 확정 (target_type을 UNIQUE에 포함)
CREATE TABLE report (
    id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    reporter_id      bigint        NOT NULL REFERENCES member (id),
    target_type      varchar(20)   NOT NULL,           -- POST | COMMENT
    target_id        bigint        NOT NULL,           -- 다형 참조라 FK 없음
    target_author_id bigint        NOT NULL REFERENCES member (id),
    reason           varchar(30)   NOT NULL,
    detail           varchar(200),
    snapshot_title   varchar(100),
    snapshot_content varchar(2000),
    status           varchar(20)   NOT NULL DEFAULT 'PENDING',
    handled_by       bigint        REFERENCES member (id),
    handled_at       timestamptz,
    created_at       timestamptz   NOT NULL DEFAULT now(),
    CONSTRAINT uq_report UNIQUE (reporter_id, target_type, target_id),
    CONSTRAINT ck_report_target CHECK (target_type IN ('POST', 'COMMENT')),
    CONSTRAINT ck_report_reason CHECK (reason IN ('SPAM', 'ABUSE', 'SEXUAL', 'PRIVACY', 'COPYRIGHT', 'OTHER')),
    CONSTRAINT ck_report_detail CHECK (reason <> 'OTHER' OR length(btrim(detail)) > 0),
    CONSTRAINT ck_report_status CHECK (status IN ('PENDING', 'HIDDEN', 'REJECTED', 'CLOSED_NO_TARGET')),
    CONSTRAINT ck_report_self   CHECK (reporter_id <> target_author_id)
);
CREATE INDEX ix_report_pending ON report (target_type, target_id) WHERE status = 'PENDING';

-- 숨김 (post, comment에 컬럼 추가)
ALTER TABLE post    ADD COLUMN hidden_at timestamptz, ADD COLUMN hidden_by bigint REFERENCES member (id),
                    ADD COLUMN hidden_reason varchar(30);
ALTER TABLE comment ADD COLUMN hidden_at timestamptz, ADD COLUMN hidden_by bigint REFERENCES member (id),
                    ADD COLUMN hidden_reason varchar(30);

-- 정지 (member에 컬럼 추가, status는 이미 SUSPENDED 있음)
ALTER TABLE member  ADD COLUMN suspended_until timestamptz,      -- 영구 정지는 null
                    ADD COLUMN suspended_reason varchar(200);
```

</details>

**I-4. EXPLAIN용 샘플 데이터 생성 SQL** (V1 적용 뒤 실행. 실행 중 내 스크립트 실수 두 가지를 고쳤다: 집계 인자가 바깥 열만 참조해 바깥 쿼리 집계가 된 것, 댓글의 `updated_at < created_at` — V1의 `ck_comment_edited`가 정상적으로 거부. 또 미래 시각 `first_public_at`이 32 점수식의 `power()`를 음수로 만들어 그 값을 지금 시각으로 잘랐다. 실제 데이터에서는 생기지 않는 값이다.)

<details><summary>펼치기</summary>

```sql
-- V1 스키마용 샘플 데이터 (결정적: setseed)
SELECT setseed(0.42);
-- 회원 3,000명 (탈퇴 신청 40명, 익명 처리 20명, 관리자 2명)
INSERT INTO member(handle,nickname,bio,terms_agreed_at,privacy_agreed_at,created_at)
SELECT 'user' || lpad(i::text,5,'0'), '회원' || i, '소개 ' || i, now(), now(), now() - (random()*700 || ' days')::interval
FROM generate_series(1,3000) i;
UPDATE member SET nickname = '김민서' WHERE id = 7;
UPDATE member SET handle = 'kim755030' WHERE id = 7;
UPDATE member SET role='ADMIN' WHERE id IN (1,2);
UPDATE member SET status='WITHDRAWN', withdrawn_at = now() - (random()*20 || ' days')::interval WHERE id BETWEEN 2901 AND 2940;
UPDATE member SET status='WITHDRAWN', withdrawn_at = now() - interval '40 days', deleted_at = now() - interval '9 days', nickname = NULL, bio = NULL WHERE id BETWEEN 2941 AND 2960;
INSERT INTO auth_identity(member_id,provider,provider_user_id,email,password_hash,email_verified_at)
SELECT id,'LOCAL',handle||'@ex.com',handle||'@ex.com','h',now() FROM member WHERE deleted_at IS NULL;
-- 글 50,000개
WITH w AS (SELECT array['스프링','트랜잭션','정리','JPA','N+1','fetch','join','배포','도커','쿠버네티스','롬복','인덱스','성능','테스트','리팩터링','회고','알고리즘','자바','코틀린','리액트','타입스크립트','캐시','레디스','쿼리','설계'] a)
INSERT INTO post(author_id,title,content_md,content_html,excerpt,status,visibility,published_at,first_public_at,created_at,updated_at,deleted_at,hidden_at,hidden_by,hidden_reason)
SELECT a, t, c, '<p>'||c||'</p>', left(c,200), st, vis, pub,
       CASE WHEN st='PUBLISHED' AND vis='PUBLIC' THEN pub + (CASE WHEN random()<0.05 THEN (random()*30||' days')::interval ELSE interval '0' END) END,
       pub - interval '1 hour', pub + interval '1 hour',
       CASE WHEN random() < 0.03 THEN now() - (random()*40 || ' days')::interval END,
       CASE WHEN st='PUBLISHED' AND random() < 0.01 THEN now() END,
       NULL, NULL
FROM (
  SELECT i, 1 + floor(power(random(),2) * 2960)::int AS a,
         (SELECT string_agg(w.a[1+floor(random()*25)::int + 0*g], ' ') FROM generate_series(1, 2 + (i % 4)) g) || ' ' || i AS t,
         (SELECT string_agg(w.a[1+floor(random()*25)::int + 0*g], ' ') FROM generate_series(1, 40 + (i % 120)) g) AS c,
         CASE WHEN random() < 0.85 THEN 'PUBLISHED' ELSE 'DRAFT' END AS st,
         CASE WHEN random() < 0.85 THEN 'PUBLIC' ELSE 'PRIVATE' END AS vis,
         now() - (random()*730 || ' days')::interval AS pub
  FROM generate_series(1,50000) i, w
) s;
UPDATE post SET published_at = NULL, first_public_at = NULL WHERE status='DRAFT';
UPDATE post SET hidden_by = 1, hidden_reason = 'SPAM' WHERE hidden_at IS NOT NULL;
UPDATE post SET content_md = content_md || ' 희귀어사전 ' WHERE id % 5000 = 1;
UPDATE post SET title = '롬복 ' || title WHERE id % 7000 = 3;
-- 태그 500개 + 글-태그
INSERT INTO tag(name) SELECT unnest(array['spring','spring-boot','spring-security','jpa','java','docker','kotlin','react','redis','스프링','회고','c++','c#','node.js']);
INSERT INTO tag(name) SELECT 'tag' || i FROM generate_series(1,486) i;
INSERT INTO post_tag(post_id,tag_id,position)
SELECT p.id, t.tag_id, t.pos FROM post p
CROSS JOIN LATERAL (SELECT tag_id, (row_number() OVER ()) - 1 AS pos FROM (SELECT DISTINCT 1 + floor(power(random(),3)*500)::int + (p.id*0) AS tag_id FROM generate_series(1,3)) d) t
WHERE p.status='PUBLISHED';
-- 이미지 30,000
INSERT INTO image(uploader_id,storage_key,thumb_storage_key,original_name,content_type,size_bytes,thumb_size_bytes,width,height,status,detached_at,created_at)
SELECT 1+floor(random()*2960)::int, 'images/'||i||'.webp', 'images/'||i||'_thumb.webp', 'p'||i||'.jpg','image/webp', 300000, 40000, 1920,1080,
       CASE WHEN random()<0.9 THEN 'ATTACHED' ELSE 'TEMP' END, CASE WHEN random()<0.05 THEN now()-interval '8 days' END, now()-(random()*700||' days')::interval
FROM generate_series(1,30000) i;
-- 댓글 100,000 (최상위 70,000 + 답글 30,000)
INSERT INTO comment(post_id,author_id,content,created_at,updated_at)
SELECT p.id, 1+floor(random()*2960)::int, '댓글 ' || g, z.ts, z.ts
FROM (SELECT id, published_at FROM post WHERE status='PUBLISHED' ORDER BY random() LIMIT 20000) p, generate_series(1, 1 + floor(random()*6)::int) g,
LATERAL (SELECT p.published_at + ((random()*10)::text||' days')::interval + (g*0 || ' s')::interval AS ts) z
LIMIT 70000;
UPDATE comment SET updated_at = created_at WHERE updated_at < created_at;
INSERT INTO comment(post_id,author_id,parent_id,content,created_at,updated_at)
SELECT c.post_id, 1+floor(random()*2960)::int, c.id, '답글', c.created_at + interval '1 hour', c.created_at + interval '1 hour'
FROM (SELECT id, post_id, created_at FROM comment ORDER BY random() LIMIT 30000) c;
UPDATE comment SET hidden_at = now(), hidden_by = 1, hidden_reason='ABUSE' WHERE id % 500 = 0;
UPDATE comment SET content = '', deleted_at = now() WHERE id % 700 = 0 AND parent_id IS NULL;
-- 좋아요 150,000 (자기 글 제외)
INSERT INTO post_like(post_id,member_id,created_at)
SELECT p, m, now() - (random()*300||' days')::interval FROM (
  SELECT 1+floor(random()*50000)::int p, 1+floor(random()*2960)::int m FROM generate_series(1,160000)) x
WHERE NOT EXISTS (SELECT 1 FROM post WHERE id=x.p AND author_id=x.m) ON CONFLICT DO NOTHING;
-- 카운터 맞추기
UPDATE post p SET like_count = s.n FROM (SELECT post_id, count(*) n FROM post_like GROUP BY 1) s WHERE p.id = s.post_id;
UPDATE post p SET comment_count = s.n FROM (SELECT post_id, count(*) n FROM comment WHERE deleted_at IS NULL AND hidden_at IS NULL GROUP BY 1) s WHERE p.id = s.post_id;
UPDATE post SET view_count = floor(random()*5000) WHERE status='PUBLISHED';
-- 팔로우 30,000
INSERT INTO follow(follower_id,followee_id,created_at)
SELECT a,b, now()-(random()*300||' days')::interval FROM (SELECT 1+floor(random()*2960)::int a, 1+floor(power(random(),2)*2960)::int b FROM generate_series(1,32000)) x WHERE a<>b ON CONFLICT DO NOTHING;
INSERT INTO follow(follower_id,followee_id) SELECT 7, id FROM member WHERE id BETWEEN 100 AND 400 ON CONFLICT DO NOTHING;
-- 알림 100,000 + 행위자
INSERT INTO notification(receiver_id,type,post_id,comment_id,result,last_actor_id,actor_count,group_key,read_at,created_at,updated_at)
SELECT r, ty,
       CASE WHEN ty IN ('COMMENT','REPLY','LIKE','NEW_POST','CONTENT_HIDDEN') THEN 1+floor(random()*50000)::int END,
       NULL,
       CASE WHEN ty='REPORT_RESOLVED' THEN 'ACTION_TAKEN' END,
       CASE WHEN ty IN ('REPORT_RESOLVED','CONTENT_HIDDEN') THEN NULL ELSE 1+floor(random()*2960)::int END,
       1,
       CASE WHEN ty='LIKE' THEN 'LIKE:post:'||(1+floor(random()*50000)::int) WHEN ty='FOLLOW' THEN 'FOLLOW' END,
       CASE WHEN random()<0.8 OR ty='FOLLOW' THEN now() END, ts, ts
FROM (SELECT 1+floor(power(random(),2)*2960)::int r,
             (array['COMMENT','REPLY','LIKE','FOLLOW','NEW_POST','REPORT_RESOLVED','CONTENT_HIDDEN'])[1+floor(random()*7)::int] ty,
             now()-(random()*120||' days')::interval ts
      FROM generate_series(1,100000)) x
WHERE EXISTS (SELECT 1 FROM post WHERE id = 1) 
ON CONFLICT (receiver_id, group_key) WHERE read_at IS NULL AND group_key IS NOT NULL DO NOTHING;
DELETE FROM notification n WHERE post_id IS NOT NULL AND NOT EXISTS (SELECT 1 FROM post p WHERE p.id=n.post_id);
INSERT INTO notification_actor(notification_id,actor_id,created_at)
SELECT n.id, n.last_actor_id, n.created_at FROM notification n WHERE n.group_key IS NOT NULL;
-- 신고 3,000
INSERT INTO report(reporter_id,target_type,target_id,target_author_id,reason,snapshot_title,snapshot_content,status,created_at)
SELECT DISTINCT ON (r, p.id) r, 'POST', p.id, p.author_id, 'SPAM', p.title, left(p.content_md,2000),
       CASE WHEN random()<0.7 THEN 'PENDING' ELSE 'REJECTED' END, now()-(random()*60||' days')::interval
FROM (SELECT 1+floor(random()*2960)::int r, 1+floor(random()*50000)::int pid FROM generate_series(1,3200)) x JOIN post p ON p.id=x.pid
WHERE p.author_id <> x.r ON CONFLICT DO NOTHING;
-- 일별 조회수 (최근 120일, 글 2,000개)
INSERT INTO post_view_daily(post_id,view_date,views)
SELECT p.id, current_date - d, 1+floor(random()*50)::int FROM (SELECT id FROM post WHERE status='PUBLISHED' ORDER BY id LIMIT 2000) p, generate_series(0,119) d;
-- 알림 끄기 일부
INSERT INTO notification_mute(member_id,type) SELECT id,'NEW_POST' FROM member WHERE id % 10 = 0;
ANALYZE;
```

</details>

### J. 기타 확인
**J-1·J-3·J-4·J-5·J-6** (사진 재연결 vs CASCADE, OPTION 적용, Flyway 계정 권한, 51 block 추출, 이름 규칙)

```text
== J-1. 다른 글에서 쓰는 사진 / 프로필 사진이 정리 배치에 지워지는가
-- 준비: 글 A·B가 사진 1을 함께 쓰고, 사진 2는 프로필
insert into post_image values (1,1),(2,1); update member set profile_image_id=2 where id=1;
-- 05 §7 ⑥: 글 A에서 사진 1을 뺌 → "빠진 사진 detached_at 기록" (B가 쓰는지 확인 규칙 없음). 프로필도 detached_at이 기록됐다고 가정
delete from post_image where post_id=1 and image_id=1; update image set detached_at = now() - interval '8 days' where id in (1,2);
-- 04 §4-4 정리 배치: detached_at 7일 경과 → 행 삭제
delete from image where id=1 → deleted  / 글 B의 post_image 남은 행: 0
delete from image where id=2 → ERROR:  update or delete on table "image" violates RESTRICT setting of foreign key constraint "fk_member_profile_image" on table "member"

== J-3. OPTION-post-split.sql (member만 있는 빈 DB)
exit=0 오류 0건, 테이블: member,post,post_content,post_stat

== J-4. Flyway 계정 권한 (pg_trgm)
DB 소유자(비superuser)로 V1 적용: exit=0 오류 0건 / pg_trgm 1.6 trusted=t
소유자 아님(스키마 CREATE만)으로 적용: exit=3 ERROR:  permission denied to create extension "pg_trgm"
소유자 아님 + DB CREATE 권한 + 스키마 CREATE로 적용: exit=0 오류 0건

== J-5. 51:795 block 추출 = V1 ?
같음 (끝 줄바꿈 1개 차이만)

== J-6. 이름 규칙
접두어 규칙에 안 맞는 UNIQUE/CHECK/FK/인덱스: []
제약이 아닌 UNIQUE 인덱스: uq_member_nickname, uq_notification_unread_group
복수형 같은 테이블 이름: []
```

**J-2. `comment_count` — 숨김·삭제·해제 순서** (21 §8·§9, 43 §4-3의 ±1 규칙을 그대로 적용)

```text
== 경우 1 (글 P): 작성 2건 → 숨김(21 §9 −1) → 작성자 삭제(21 §8, 답글 있어 자리만, '숨김 상태였으면 그대로') → 해제(21 §9·43 §4-3 +1)
  작성 2건 → comment_count=2 / 실제(삭제·숨김 아닌 행)=2
  숨김 −1 → comment_count=1 / 실제(삭제·숨김 아닌 행)=1
  삭제(자리, 그대로) → comment_count=1 / 실제(삭제·숨김 아닌 행)=1
  해제 +1 → comment_count=2 / 실제(삭제·숨김 아닌 행)=1
== 경우 2 (글 Q): 작성 2건 → 작성자 삭제(자리, −1) → (삭제 전에 들어온 신고로) 관리자 숨김 −1
  작성 2건 → comment_count=2 / 실제(삭제·숨김 아닌 행)=2
  삭제(자리) −1 → comment_count=1 / 실제(삭제·숨김 아닌 행)=1
  숨김 −1 → comment_count=0 / 실제(삭제·숨김 아닌 행)=1
```

**J-7. `erdcloud-import.sql` ↔ V1** (`cmp_import.py`: CREATE TABLE의 컬럼 줄을 파싱해 카탈로그와 비교. FK 32는 `CONSTRAINT` 줄만 센 것이고 `ALTER TABLE member … fk_member_profile_image` 1개를 더하면 33)

```text
컬럼 139개, DATETIME 40개, FK 32개 (V1 FK 33개), KEY 11개
메모·KEY에 없는 CHECK/인덱스: 없음
불일치: 없음
```

### K. 정리 확인
```text
$ docker rm -f -v teamblog-verify-02-pg
teamblog-verify-02-pg
$ docker ps -a | grep teamblog
(출력 없음, grep exit=1)
$ docker ps -a --format '{{.Names}}' | grep -c teamblog-verify-02
0
$ rm -rf /tmp/teamblog-verify-02
$ ls -d /tmp/teamblog-verify-02
ls: /tmp/teamblog-verify-02: No such file or directory
```
- check-ddl 복사본이 띄운 `teamblog-verify-02-ddl`·`teamblog-verify-02-ddl-diag`는 각 실행이 끝날 때 스크립트의 `trap cleanup`(`docker rm -f -v`)으로 지워졌다 (실행 직후 `docker ps -a`에서 확인).
- /tmp에 남아 있는 `teamblog-check.lua`, `teamblog-docs-baseline.json`, `teamblog-erd-metadata.json`, `teamblog-extracted-v1.sql`, `teamblog-generate-erd.py`, `teamblog-verify-erd.sql`(10:02~13:43 생성)은 내가 만든 파일이 아니라서 건드리지 않았다.
- 저장소에서 쓴 파일은 이 보고서 하나뿐이다. `docs/`·`erd`·`scripts/`·다른 보고서는 고치지 않았고 브랜치·stash도 건드리지 않았다.
