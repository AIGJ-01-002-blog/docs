# 10. 검색 (키워드 + 의미 검색)

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 글자가 맞는 글(키워드, pg_trgm)과 뜻이 가까운 글(의미 검색, pgvector + bge-m3 임베딩)을 따로 찾은 뒤 순위를 섞어(RRF) 보여 줍니다. 의미 검색이 안 되는 환경에서는 키워드 검색만으로 그대로 동작합니다.

## 구성

```mermaid
flowchart LR
    B["SearchBox · SearchPage"]
    subgraph Search["search 모듈"]
        SC["SearchController<br/>분당 30회"]
        ST["SearchTerms<br/>단어 5개 · 50자"]
        SQ["SearchQuery<br/>제목 → 태그 → 본문"]
        SM["SemanticSearch<br/>pgvector 코사인 거리"]
        EC["EmbeddingClient<br/>서버 Ollama → 집 PC → Gemini"]
        EJ["EmbeddingIndexJob<br/>60초마다 20개"]
        SN["Snippet<br/>mark 강조"]
    end
    R[("Redis<br/>검색어 벡터 1일")]
    P[("PostgreSQL<br/>pg_trgm GIN<br/>post_embedding vector")]

    B --> SC --> ST
    SC --> SQ --> P
    SC --> SM --> P
    SM --> EC
    SM --> R
    EJ --> EC
    EJ --> P
    SC --> SN
```

## 처리 흐름: 관련도순 검색

```mermaid
sequenceDiagram
    autonumber
    participant B as SearchBox
    participant C as SearchController
    participant S as SemanticSearch
    participant E as EmbeddingClient
    participant R as Redis
    participant P as PostgreSQL
    B->>C: 검색창에 커서 → GET /api/search/warm (임베딩 모델 깨우기)
    B->>C: GET /api/search/posts?q=redis 동시성
    C->>C: 요청 제한, 검색어 나누기
    C->>S: 뜻이 가까운 글
    S->>R: 같은 검색어 벡터 있나
    alt 없음
        S->>E: 검색어 임베딩
        S->>R: 1일 저장
    end
    S->>P: post_embedding 에서 가까운 글 30개 (거리 기준 이하만)
    C->>P: 키워드 후보 100개 (제목 → 제목·태그 → 본문 단계)
    C->>C: RRF 로 두 순위 섞기
    C-->>B: 카드 + 강조된 문장 조각 + 서명된 커서
```

## 순위 섞기 (RRF)

```mermaid
flowchart LR
    K["키워드 순위 rk"] --> F["점수 = 1/(60+rk+1) + 0.8 × 1/(60+rs+1)"]
    SE["의미 순위 rs"] --> F
    F --> O["점수 높은 순<br/>같으면 키워드 순위가 앞선 것"]
    O --> T["섞은 목록이 끝나면<br/>남은 키워드 결과를 이어서"]
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 검색어 | 50자, 단어 5개, 한 글자 단어 버림. 두 글자는 제목·태그만, 세 글자 이상은 본문까지(트라이그램 때문) |
| 키워드 | 최근 공개 글 3,000개 안에서 먼저, 모자라면 색인 후보로. 단계 안에서는 최신순 |
| 의미 검색 | bge-m3(로컬) 또는 gemini-embedding-001(768차원). 거리 0.55(로컬)/0.4(Gemini) 이하, 3초 제한. 실패하면 60초 동안 키워드만 |
| 색인 | 60초마다 임베딩 없는/낡은 공개 글 20개, 제목+태그+본문 3,000자 |
| 확장 없을 때 | pg_trgm 없으면 색인 없는 LIKE, pgvector 없으면 키워드만 |
| 사람 검색 | 닉네임·주소, 정확히 같은 것 먼저 20명 |
| 개인정보 | 검색어는 서버 로그에 남기지 않고 길이·시간만. 최근 검색어 8개는 브라우저에만 |

## 코드 위치

- 백엔드: `search/web/SearchController`, `search/application/{SearchQuery, SearchTerms, Snippet}`, `search/semantic/{SemanticSearch, EmbeddingClient, EmbeddingIndexJob, SemanticProperties}`
- 화면: `pages/SearchPage`, `components/SearchBox`, `lib/{search.ts, recentSearches.ts}`
- 테이블: pg_trgm 색인(V2), `post_embedding`(V16, pgvector가 있을 때만)

## 예상 질문

- **왜 키워드 검색만으로는 부족했나요?** "동시성"으로 찾을 때 "race condition"을 다룬 글처럼 글자는 다르지만 뜻이 같은 글을 놓쳤습니다.
- **임베딩 서버가 죽으면?** 60초 동안 의미 검색을 쉬고 키워드 결과만 보여 줍니다. 사용자는 검색 실패를 보지 않습니다.
