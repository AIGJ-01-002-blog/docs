# 11. 주제 가지와 비슷한 글

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 시리즈에 넣지 않은 공개 글을 태그와 임베딩으로 10분마다 묶어 "주제"를 만들고, 홈 피드에서 같은 시리즈·주제의 글을 Git 브랜치처럼 선으로 잇습니다. 글 아래에는 비슷한 글 4개를, 발행 때에는 넣을 만한 시리즈·주제를 제안합니다.

## 구성

```mermaid
flowchart LR
    subgraph Topic["topic 모듈"]
        TJ["TopicIndexJob<br/>10분마다 · JobLock"]
        TC["TopicClusterer<br/>태그 IDF + 임베딩 → 묶기"]
        TQ["TopicQuery<br/>글의 주제 · 인기 주제"]
        SP["SimilarPosts<br/>비슷한 글 4개"]
        BS["BranchSuggester<br/>발행 때 시리즈·주제 제안"]
    end
    SEM["search SemanticSearch<br/>similarPairs"]
    P[("PostgreSQL<br/>post_topic · post_topic_optout<br/>post_embedding")]
    subgraph UI["화면"]
        HF["HomePage 피드<br/>branchLanes 최대 3줄"]
        BB["BranchBox · BranchList"]
        SI["SimilarPosts"]
        BSG["BranchSuggest"]
    end

    TJ --> TC
    TJ --> SEM --> P
    TJ --> P
    HF --> TQ --> P
    BB --> TQ
    SI --> SP --> P
    BSG --> BS --> P
```

## 처리 흐름: 주제 묶기

```mermaid
sequenceDiagram
    autonumber
    participant J as TopicIndexJob
    participant P as PostgreSQL
    participant S as SemanticSearch
    participant C as TopicClusterer
    J->>J: 10분마다 JobLock topic-index
    J->>P: 시리즈 밖, 빼기 안 한 공개 글 최대 2,000개
    J->>C: 태그가 겹치는 정도로 선 잇기 (흔한 태그는 무시)
    J->>S: 임베딩이 가까운 글 짝 (이웃 5개)
    J->>C: 선을 강한 순으로 합치기 (주제당 12개까지)
    C-->>J: 주제 이름 = 가장 특징적인 태그, 없으면 오래된 글 제목 앞부분
    J->>P: 한 트랜잭션에서 post_topic 전부 갈아 끼우기
    Note over J,P: 실패하면 예전 묶음이 그대로 남음
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 태그 연결 | IDF 가중 겹침 0.5 이상, 60개 넘는 글에 쓰인 흔한 태그는 제외 |
| 임베딩 연결 | 거리 ≤ 검색 기준 × 0.6, 이웃 5개 |
| 묶음 | Kruskal 방식으로 강한 선부터 합침, 최대 12개. 키 = 가장 작은 글 번호, 방법 TAG·EMBEDDING·MIXED |
| 피드 가지 | 시리즈가 있으면 시리즈, 없으면 주제. 화면은 최대 3줄로 그림. `?branch=s{id}`·`t{key}`로 가지만 보기 |
| 비슷한 글 | 임베딩 이웃에서 같은 시리즈·주제를 빼고, 모자라면 겹치는 태그 수로 채워 4개 |
| 발행 제안 | 내 시리즈·주제 중 점수(겹치는 태그 IDF ÷ 이 글 태그 IDF 합) 0.4 이상 |
| 빼기 | 글쓴이가 주제 묶기에서 뺄 수 있음(`PUT /topic-optout`) |

## 코드 위치

- 백엔드: `topic/{web/TopicController, application/TopicIndexJob, TopicClusterer, TopicQuery, SimilarPosts, BranchSuggester, TopicProperties}`
- 화면: `components/{BranchBox, BranchList, BranchSuggest, SimilarPosts}`, `lib/{branch.ts, branchLanes.ts}`
- 테이블: `post_topic`(V24), `post_topic_optout`(V25)

## 예상 질문

- **AI(LLM)로 주제를 정하나요?** 아니요. 태그 통계와 이미 만든 임베딩만 써서 계산합니다. 추가 비용이 없고 결과가 늘 같습니다.
- **글이 많아지면?** 한 번에 최대 2,000개만 보고, 별도 작업으로 10분마다 돌아 사용자 요청 속도와 무관합니다.
