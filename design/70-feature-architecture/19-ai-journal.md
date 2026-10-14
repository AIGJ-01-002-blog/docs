# 19. AI 일기·글감 제안

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: AI 도구가 작업 중에 남긴 한두 문장 메모(`add_note`)를 모았다가, 사용자가 고른 시각에 하루치를 묶어 "개발 일기" 글로 만듭니다. 주제가 끝나면 AI가 글감을 제안(`propose_post`)하고, 사용자는 화면에서 임시글로 만들거나 넘깁니다.

## 구성

```mermaid
flowchart LR
    AI["AI 도구"]
    subgraph MCP["mcp 모듈"]
        AJ["AiJournal<br/>메모 · 제안"]
        DJ["AiDiaryJob<br/>매시 정각 KST"]
        TS["TopicSuggester<br/>글감 고르기 (LLM 없음)"]
        SC["SecretScanner"]
    end
    UI["AiProposals · SettingsPage<br/>일기 켜기 · 시각 고르기"]
    PC["post PostCommandService<br/>createAiDiary"]
    P[("PostgreSQL<br/>ai_note · ai_post_proposal<br/>post.ai_diary")]
    N["notification AI_PROPOSAL"]

    AI -->|"add_note"| AJ
    AI -->|"propose_post"| AJ
    AI -->|"suggest_topics"| TS --> P
    AJ --> SC
    AJ --> P
    AJ --> N
    DJ --> P
    DJ --> PC
    UI --> AJ
```

## 처리 흐름: 메모 → 일기

```mermaid
sequenceDiagram
    autonumber
    participant AI as AI 도구
    participant J as AiJournal
    participant P as PostgreSQL
    participant D as AiDiaryJob
    AI->>J: add_note "Redis 잠금이 만료돼서 두 번 돌았어요"
    J->>J: 일기 켬 확인, 비밀값 검사
    J->>P: 회원 행 잠금, 하루 60개 확인, ai_note 저장
    D->>D: 매시 정각 JobLock ai-diary
    D->>P: 지금 시각을 고른 회원 찾기
    D->>P: DELETE ai_note ... RETURNING (한 번만 가져감)
    D->>D: 하루 단위로 묶어 틀에 맞춰 본문 (서버에서 AI 호출 없음)
    D->>P: "YYYY-MM-DD 개발 일기" 글 (ai_diary = true)
    alt AI 발행 허용 + 이메일 인증 + 비밀값 없음
        D->>P: 바로 발행
    else
        D->>P: 임시글로 둠
    end
```

## 처리 흐름: 글감 제안 → 임시글

```mermaid
flowchart LR
    A["AI가 주제 끝남을 판단"] --> B["propose_post<br/>제목 · 다룰 것 · 뺄 것"]
    B --> C["ai_post_proposal OPEN<br/>같은 제목이면 갱신"]
    C --> D["AI_PROPOSAL 알림"]
    D --> E{"사용자 선택"}
    E -->|"쓰기"| F["create_draft proposal_id<br/>또는 화면 버튼 → DRAFTED"]
    E -->|"넘기기"| G["DISMISSED"]
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 메모 | 하루 60개, 1,000자, 주제당 50개. 7일 지난 메모 삭제. 일기를 끄면 메모도 지움 |
| 일기 시각 | 회원이 0~23시 중 고름(기본 0시, V23). 그 시각에 끝나는 24시간을 하루로 묶음 |
| 중복 방지 | `DELETE … RETURNING`으로 메모를 꺼내서 작업이 두 번 돌아도 일기 하나 |
| 검색 노출 | 일기 글은 독자에게 보이지만 noindex·광고 없음·사이트맵 제외(08 참고) |
| 제안 | 회원당 열린 제안 20개, 같은 제목은 갱신. OPEN → DRAFTED / DISMISSED |
| 글감 고르기 | 메모·일기·제안·임시글에서 동시성·장애·보안·성능·접근성·운영·인프라·AI·설계 렌즈로 분류. 이미 발행한 글과 겹치는 정도 계산. 기본 14일·5개. IP·이메일·비밀값은 가림 |

## 코드 위치

- 백엔드: `mcp/application/{AiJournal, AiDiaryJob, TopicSuggester}`, `mcp/web/AiJournalController`
- 화면: `components/{AiProposals, AiConnectSection}`, `pages/{SettingsPage, McpPage}`
- 테이블: `ai_note`·`ai_post_proposal`·`member.ai_diary_enabled`(V18), `ai_diary_hour`·`notification_ai_proposal`(V23), `post.ai_diary`(V32)

## 예상 질문

- **일기는 AI가 서버에서 쓰나요?** 아니요. 메모 자체가 AI가 1인칭으로 쓴 문장이라, 서버는 틀에 맞춰 묶기만 합니다. 서버 AI 비용이 들지 않습니다.
- **글감 추천도 LLM인가요?** 아니요. 키워드 렌즈와 겹침 계산으로 고르는 결정적 규칙이라 같은 입력이면 같은 결과입니다.
