# 9. 태그와 AI 태그 추천

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 태그는 정규화 규칙 하나로 같은 태그를 하나로 모으고, 발행 트랜잭션 안에서 연결합니다. AI 태그 추천은 자체 서버 AI(Ollama)와 Google Gemini를 번갈아 쓰고, 같은·비슷한 글은 저장된 결과로 답해 한도를 아낍니다.

## 구성

```mermaid
flowchart LR
    W["WritePage<br/>TagInput · AiTagSuggest"]
    subgraph Tag["tag 모듈"]
        TN["TagNormalizer<br/>NFKC · 소문자 · 하이픈"]
        PT["PostTagger<br/>발행 때 연결"]
        TQ["TagQuery · TagBrowse<br/>인기 · 자동완성 · 둘러보기"]
    end
    subgraph AI["ai 모듈"]
        AT["AiTagService"]
        SS["SuggestionStore<br/>같은 글 30일 · 비슷한 글 7일"]
        PS["ProviderState<br/>한도 · 쿨다운 · 장애"]
        OL["OllamaModel<br/>qwen2.5:3b"]
        GM["GeminiModel<br/>gemini-2.5-flash-lite"]
    end
    R[("Redis<br/>ai:*")]
    P[("PostgreSQL<br/>tag · post_tag")]

    W --> TQ --> P
    W --> AT
    AT --> SS --> R
    AT --> PS --> R
    AT --> OL
    AT --> GM
    AT --> TN
    PT --> TN
    PT --> P
```

## 처리 흐름: AI 태그 추천

```mermaid
sequenceDiagram
    autonumber
    participant B as AiTagSuggest
    participant S as AiTagService
    participant R as Redis
    participant O as Ollama
    participant G as Gemini
    B->>S: POST /api/posts/42/ai-tags
    S->>S: 내 글인가, AI 동의했나, 100자 이상인가
    S->>R: 같은 내용 결과 있나 (30일)
    S->>R: 같은 글의 비슷한 내용 결과 있나 (3글자 유사도 0.9, 7일)
    alt 저장된 결과
        R-->>S: 태그 (하루 횟수 안 씀)
    else 새로 물어봄
        S->>R: 하루 20회 한도 1 차감
        S->>O: 먼저 자체 AI (자리가 있고 살아 있으면)
        alt Ollama 불가
            S->>G: Gemini (하루 450회)
        end
        S->>S: TagNormalizer 로 거르고 5개까지
        S->>R: 결과 저장
    end
    S-->>B: 이미 단 태그를 뺀 추천
    Note over S: 둘 다 안 되면 "지금은 추천할 수 없어요"만, 글쓰기는 그대로
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 정규화 | NFKC → 앞 `#` 제거 → 소문자 → 공백은 `-` → 허용 글자 `[가-힣a-z0-9._+#-]` → 30자 → 금칙어. DB CHECK `ck_tag_name`과 같은 규칙 |
| 글당 태그 | 10개. 저장은 정렬해서 `INSERT … ON CONFLICT`(교착 방지) |
| 둘러보기 | `/tags` 인기·이름·최근 순, 태그마다 인기 글 10개(트렌딩과 같은 무게 3·2·0.1) |
| 자동완성 | 앞글자 일치 10개, 분당 60회 |
| AI 순서 | 기본 `local`: Ollama 먼저, 안 되면 Gemini. `gemini`로 바꾸면 Gemini 먼저, 한도 초과 때만 Ollama |
| 한도 | 회원 하루 20회(저장된 답은 안 셈, 바쁨 오류는 돌려줌), 분당 30회 |
| 장애 표시 | Gemini 한도 소진·60초 쿨다운, Ollama 60초 장애 표시를 Redis에 둠 |
| 프롬프트 | 사이트 인기 태그 50개를 함께 줌(하루 캐시) |

## 코드 위치

- 백엔드: `tag/{web/TagController, application/TagNormalizer, PostTagger, TagQuery, TagBrowse}`, `ai/{web/AiTagController, application/AiTagService, SuggestionStore, ProviderState, AiInput, TagPrompt, MemoDraftService, infra/OllamaModel, GeminiModel}`
- 화면: `components/{AiTagSuggest, TagInput, TagRow}`, `pages/{TagsPage, TagPage}`, `lib/{aiTags.ts, tags.ts}`
- 테이블: `tag`, `post_tag` (V1). AI 동의는 `member_agreement`의 `AI`

## 예상 질문

- **AI가 이상한 태그를 주면?** AI 답도 사람이 쓴 태그와 같은 `TagNormalizer`·금칙어를 거치고, 사용자가 눌러야 붙습니다.
- **왜 AI를 두 개 쓰나요?** 무료 Gemini는 하루 한도가 있고, 자체 Ollama는 느리고 동시에 하나만 처리합니다. 서로 빈 곳을 메웁니다.
