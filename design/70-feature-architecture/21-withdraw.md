# 21. 회원 탈퇴·복구

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: 탈퇴를 신청하면 바로 로그아웃되고 글이 모두 감춰지며, 30일 안에는 복구할 수 있습니다. 30일이 지나면 새벽 작업이 모듈마다 등록한 "지우기 단계"를 정해진 순서로 실행해 개인 데이터를 지웁니다.

## 구성: 모듈마다 지우기 단계를 끼워 넣기

```mermaid
flowchart LR
    W["WithdrawPage"] --> WS["WithdrawalService<br/>탈퇴 신청 · 복구"]
    WS --> P[("member.status = WITHDRAWN<br/>withdrawn_at")]
    WS -. "MemberWithdrawn" .-> FR["friend<br/>친구 관계 바로 삭제"]
    WS --> SE["SessionTerminator<br/>모든 기기 로그아웃"]
    J["WithdrawalPurgeJob<br/>매일 04:50"] --> STEPS
    subgraph STEPS["WithdrawalPurgeStep 구현 (order 순)"]
        direction TB
        S1["5 알림 대상"] --> S2["10 글 (휴지통 완전 삭제와 같은 길)"]
        S2 --> S3["15~17 시리즈 · 소개 · 소셜 링크"]
        S3 --> S4["20 댓글 · 30 좋아요 · 40 사진"]
        S4 --> S5["44~47 디스코드 · 텔레그램 · 토큰 · AI 메모"]
        S5 --> S6["50 로그인 수단 · 60 친구 · 65 팔로우"]
        S6 --> S7["70 받은 알림 · 75 문의 · 80 신고"]
        S7 --> S8["90 회원 행"]
    end
```

## 처리 흐름

```mermaid
sequenceDiagram
    autonumber
    participant U as 회원
    participant W as WithdrawalService
    participant P as PostgreSQL
    participant R as Redis
    participant J as WithdrawalPurgeJob
    U->>W: POST /api/me/withdrawal (이메일 회원은 비밀번호, 소셜은 "탈퇴" 입력)
    W->>P: status WITHDRAWN, withdrawn_at, MemberWithdrawn 사건
    W->>R: 모든 세션 삭제
    W-->>U: 안내 메일
    Note over U,P: 30일 안: 로그인하면 복구 화면, [복구] 를 눌러야 복구 (로그인만으로는 안 됨)
    J->>J: 매일 04:50 JobLock withdraw-purge
    J->>P: 30일 지난 회원 50명씩, 회원마다 트랜잭션 + FOR UPDATE SKIP LOCKED
    J->>P: 지우기 단계들을 order 순서로 실행
    P-->>J: 커밋
    J->>R: 남은 세션·토큰·실패 횟수·제한 키 정리
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 유예 | 30일. 그동안 글·댓글·블로그가 모두 감춰짐(`withdrawn_at IS NULL` 조건) |
| 확인 | 비밀번호 5번 틀리면 잠금. 관리자는 탈퇴 불가 |
| 친구 관계 | 신청 즉시 삭제, 복구해도 되살리지 않음(V33) |
| 복구 | 명시적으로 `POST /api/me/restore` |
| 단계 순서 | 다른 행을 참조하는 것부터 지우고 회원 행은 맨 마지막(90) |
| 즉시 삭제 | 관리자가 `POST /api/admin/members/{handle}/purge`로 바로 실행 가능 |

## 코드 위치

- 백엔드: `account/application/{WithdrawalService, WithdrawalPurgeJob, WithdrawalPurgeStep}`, 각 모듈의 `*WithdrawalPurgeStep`, `friend/application/FriendshipWithdrawalListener`, `shared/security/SessionTerminator`
- 화면: `pages/{WithdrawPage, WithdrawnPage, RestorePage}`, `lib/{withdraw.ts, restore.ts}`, `App.tsx`의 `WithdrawnGate`

## 예상 질문

- **account 모듈이 모든 테이블을 알아야 하지 않나요?** 아니요. 각 모듈이 자기 데이터 지우는 법을 `WithdrawalPurgeStep`으로 등록하고, account는 순서대로 부르기만 합니다. 새 모듈을 만들면 단계 하나만 추가하면 됩니다.
- **중간에 실패하면?** 회원 한 명이 한 트랜잭션이라 통째로 취소되고 다음 날 다시 시도합니다.
