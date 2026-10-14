# 16. 알림 (앱·텔레그램·디스코드)

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 각 모듈이 낸 도메인 사건을 알림 리스너가 커밋 뒤 비동기로 받아 앱 알림을 만들고, 새 알림이 생기면 다시 사건(`NotificationCreated`)을 내어 텔레그램·디스코드로 퍼뜨립니다. 텔레그램 봇은 메모를 받아 임시글로 바꾸기도 합니다.

## 구성: 사건이 알림이 되기까지

```mermaid
flowchart LR
    subgraph Sources["사건을 내는 모듈"]
        CM["comment<br/>CommentCreated · Deleted"]
        LK["like<br/>PostLiked · Unliked"]
        FW["follow<br/>Followed · Unfollowed"]
        FR["friend<br/>FriendRequested · Accepted"]
        PO["post<br/>PostFirstPublic"]
        MO["moderation<br/>ReportsResolved · ContentHidden"]
    end
    DIR["inquiry · mcp<br/>직접 호출"]
    NL["NotificationEventListener<br/>@Async 2스레드 · 큐 1000<br/>커밋 뒤"]
    NS["NotificationService<br/>받을 사람 판정 · 묶기"]
    P[("PostgreSQL<br/>notification + 대상 테이블들")]
    NC["NotificationCreated"]
    TN["TelegramNotifier"]
    DN["DiscordNotifier"]
    TG["Telegram Bot API"]
    DS["Discord 웹훅"]
    BELL["NotificationBell<br/>30초마다 안 읽은 수"]

    Sources -. "사건" .-> NL --> NS --> P
    DIR --> NS
    NS --> NC
    NC -. "커밋 뒤 비동기" .-> TN --> TG
    NC -. "커밋 뒤 비동기" .-> DN --> DS
    BELL --> P
```

## 처리 흐름: 좋아요 알림이 텔레그램까지

```mermaid
sequenceDiagram
    autonumber
    participant L as LikeService
    participant N as 알림 리스너
    participant S as NotificationService
    participant P as PostgreSQL
    participant T as TelegramNotifier
    L->>L: 좋아요 저장, PostLiked 사건
    L-->>N: 커밋 뒤 다른 스레드로 전달
    N->>S: postLiked
    S->>S: 내 글에 내가? 탈퇴? 끈 알림? 글을 볼 수 있나?
    alt 같은 글에 안 읽은 좋아요 알림이 있음
        S->>P: 그 알림에 누른 사람만 추가 (새 사건 없음)
    else
        S->>P: 새 LIKE 알림
        S-->>T: NotificationCreated (커밋 뒤)
        T->>T: 텔레그램 연결·알림 켬 확인
        T->>T: Bot API 로 보냄, 봇이 차단됐으면 연결 해제
    end
```

## 처리 흐름: 텔레그램 메모 → 임시글

```mermaid
sequenceDiagram
    autonumber
    participant U as 텔레그램 사용자
    participant B as TelegramBot
    participant R as Redis
    participant M as MemoDraftService
    participant P as PostgreSQL
    B->>B: 1초 간격 롱 폴링 getUpdates (JobLock 으로 한 파드만)
    U->>B: "Redis 락 만료 시간 정리하기..."
    B->>R: 하루 20개 메모 한도
    B->>M: AI 동의·한도가 있으면 Ollama/Gemini 로 제목·본문 다듬기
    M-->>B: 실패하면 메모 원문 그대로
    B->>P: 임시글 저장
    B-->>U: "임시글로 저장했어요" + /write/번호 링크
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 종류 | 11가지. 사용자가 끌 수 있는 것: 댓글·답글·좋아요·팔로우·새 글·AI 글감 제안·친구 요청·친구 수락. 운영 알림(신고 처리·숨김·문의 답변)은 못 끔 |
| 묶기 | 좋아요·팔로우는 안 읽은 같은 알림에 사람만 추가. 팔로우·친구는 같은 사람에게 7일에 1번, 좋아요는 글마다 1번 |
| 새 글 알림 | 팔로워 ∪ 시리즈 구독자, 중복 없이 |
| 비동기 | 알림 2스레드·큐 1000, 넘치면 버리고 경고. 텔레그램·디스코드 각 2스레드·큐 500 |
| 보관 | 90일 지난 알림 삭제, 받는 사람당 최신 1,000개(매일 04:30) |
| 종 아이콘 | 30초마다 안 읽은 수, 탭이 가려지면 멈춤, `99+` |
| 텔레그램 연결 | 설정에서 코드 발급(10분, Redis) → 봇에 `/start 코드` → `member_telegram` 저장 |
| 디스코드 | 웹훅 주소를 받아 확인 후 저장(V27). 호스트는 늘 `discord.com`만 부르고 토큰은 화면에 돌려주지 않음. 연결·테스트 시간당 10회 |

## 코드 위치

- 백엔드: `notification/{application/NotificationEventListener, NotificationService, NotificationAsyncConfig, NotificationCleanupJob, NotificationType}`, `telegram/{application/TelegramBot, TelegramNotifier, TelegramLinks, infra/HttpTelegramApi}`, `discord/*`, `ai/application/MemoDraftService`
- 화면: `components/{NotificationBell, NotificationEntry}`, `pages/NotificationsPage`, `lib/{notifications.ts, telegram.ts, discord.ts}`
- 테이블: `notification`, `notification_actor`·`notification_post`·`notification_comment` 등 대상 테이블, `notification_mute`, `member_telegram`(V7), `member_discord`(V27)

## 예상 질문

- **알림 만들기가 실패하면 좋아요도 실패하나요?** 아니요. 커밋이 끝난 뒤 다른 스레드에서 만들기 때문에 본래 요청과 분리돼 있습니다. 대신 서버가 그 순간 죽으면 알림 하나를 잃을 수 있고, 이는 받아들인 절충입니다.
- **텔레그램은 왜 웹훅이 아니라 폴링인가요?** 롱 폴링은 서버에서 바깥으로 나가는 연결만 쓰므로, 텔레그램이 부를 공개 웹훅 주소 없이도 동작합니다(코드 기준 추정 이유). 여러 파드 중 한 곳만 받도록 `JobLock`을 잡고, 어디까지 읽었는지는 Redis `tg:offset`에 둡니다.
