# 14. 트렌딩

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 10분마다 최근 7일 공개 글의 점수를 SQL 한 번으로 계산해 상위 100개를 Redis 스냅숏으로 저장하고, 화면은 그 스냅숏을 커서로 넘겨 봅니다. 스냅숏 덕분에 순위가 바뀌어도 다음 쪽에서 글이 겹치거나 빠지지 않습니다.

## 구성

```mermaid
flowchart LR
    F["Feed.tsx 트렌딩 탭"]
    subgraph Trending["trending 모듈"]
        TC["TrendingController<br/>GET /api/posts/trending"]
        TS["TrendingService<br/>10분마다 · JobLock"]
        TR["TrendingRanker<br/>점수 SQL"]
    end
    R[("Redis<br/>trending:snapshot:시각<br/>trending:current 30분")]
    P[("PostgreSQL<br/>post_like · comment · post_view")]

    TS --> TR --> P
    TS --> R
    F --> TC --> R
    TC --> P
```

## 점수

```mermaid
flowchart LR
    L["좋아요 × 3"] --> S["합"]
    C["댓글 단 사람 수 × 2<br/>(글쓴이 빼고, 한 사람 1번)"] --> S
    V["조회수 × 0.1"] --> S
    S --> D["÷ (처음 공개 뒤 지난 시간 + 2)^1.5"]
    D --> T["상위 100개<br/>글쓴이당 3개까지"]
```

## 처리 흐름

```mermaid
sequenceDiagram
    autonumber
    participant J as TrendingService
    participant P as PostgreSQL
    participant R as Redis
    participant F as Feed.tsx
    J->>J: 10분마다 JobLock trending
    J->>P: 점수 SQL 한 번 (7일 안, 좋아요나 댓글이 있는 공개 글)
    J->>R: trending:snapshot:시각 = 글 번호 목록, trending:current 가 가리킴 (30분)
    F->>J: GET /api/posts/trending
    J->>R: current 스냅숏 읽기
    J->>P: 그중 지금 볼 수 있는 글 카드
    J-->>F: 9개 + 커서(스냅숏, 위치)
    F->>J: 다음 쪽 (같은 스냅숏)
    alt 스냅숏이 만료됨
        J-->>F: 410 TRENDING_EXPIRED → 처음부터 다시
    end
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 후보 | 공개 목록 조건, 처음 공개 7일 안, 좋아요나 댓글이 1개 이상 |
| 동점 | 처음 공개 최신순, 그다음 번호 큰 순 |
| 캐시 | `Cache-Control: public, max-age=60` |
| Redis 장애 | 첫 쪽만 바로 계산해 `temporary=true`로 주고, 다음 쪽 없음 |
| 볼 수 없는 글 | 스냅숏 뒤에 숨겨지거나 지워진 글은 쪽을 채울 때 건너뜀 |

## 코드 위치

- 백엔드: `trending/{web/TrendingController, application/TrendingService, TrendingRanker}`
- 화면: `components/Feed`, `lib/trending.ts`

## 예상 질문

- **왜 시간으로 나누나요?** 오래된 인기 글이 계속 위에 있지 않게, 새 글이 반응을 받으면 빨리 오르도록 하는 감쇠입니다(Hacker News 방식).
- **글쓴이당 3개 제한은?** 한 사람의 글이 순위를 독차지하지 않게 하려는 것입니다.
