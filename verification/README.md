# 통합 설계 검증 계획

> 언제: 모든 요구사항 PR을 merge한 직후 (아키텍처·ERD 통합 작성 전)
> 목적: 세 사람이 따로 정리한 문서를 합쳤을 때 생기는 **경계의 모순, 빠진 것, 실제로 동작하지 않는 규칙**을 찾는다.
> 원칙: 검증 팀원은 **문서를 고치지 않고 보고만** 한다. 결정은 사람이 한다.

---

## 1. 구성: 검증 팀원 5명 + 총괄 1명

담당자별(강성찬 문서 / 김민서 문서 …)이 아니라 **관점별**로 나눈다. 문제는 각 문서 안보다 문서와 문서가 만나는 경계에서 생기기 때문이다.

| 역할 | 지시문 | 모델 | 하는 일 |
|---|---|---|---|
| **총괄** | [prompts/00-lead.md](./prompts/00-lead.md) | **Fable** (`claude-fable-5-1`) | 다섯 보고서를 모아 중복 정리, 확실하지 않은 지적 재확인, 심각도순 최종 보고서 |
| 1. 일관성 | [prompts/01-consistency.md](./prompts/01-consistency.md) | **Fable** | 문서끼리 다른 결정, 맞추기로 한 것이 실제로 맞춰졌는지 |
| 2. ERD·스키마 | [prompts/02-erd-schema.md](./prompts/02-erd-schema.md) | Opus 5.5 (effort `high`) | 모든 ERD 변경 제안을 합쳐 실제 PostgreSQL에 적용·검증 |
| 3. 권한·보안·개인정보 | [prompts/03-security-privacy.md](./prompts/03-security-privacy.md) | Opus 5.5 (effort `high`) | 권한 매트릭스 완성, 404 원칙, XSS·CSRF·요청 제한, 외부 전송·탈퇴 |
| 4. 완결성·측정 가능성 | [prompts/04-completeness.md](./prompts/04-completeness.md) | Opus 5.5 (effort `high`) | 모든 요구사항에 문서·완료 기준이 있는지, 완료 기준을 테스트할 수 있는지, 미결 목록 |
| 5. API·운영 | [prompts/05-api-ops.md](./prompts/05-api-ops.md) | Opus 5.5 (effort `high`) | API·오류 형식·커서·Redis 키·배치·설정값을 표로 모아 비교 (#5 API 명세 준비) |

모든 팀원은 먼저 [prompts/common.md](./prompts/common.md)를 읽는다.

**Fable을 총괄·일관성에만 쓰는 이유:** 가장 깊은 추론이 필요한 두 역할이다. 나머지는 기준이 분명한 점검이라 Opus로 충분하고, Fable은 Opus보다 약 2.5배 비싸고 느리다. 또 문서 대부분을 Opus 5.5가 작성했으므로, 핵심 관점은 다른 모델이 보는 편이 같은 사각지대를 피하는 데 유리하다. 비용이 상관없으면 전부 Fable로 해도 된다.

---

## 2. 진행 순서

```
① 모든 PR merge → dev 최신으로 받기
② scripts/check-all.sh 실행 → 결과를 verification/reports/{날짜}/00-scripts.txt 로 저장
③ 검증 팀원 1~5를 동시에 시작 (서로 기다리지 않음)
④ 다섯 명이 끝나면 총괄 시작
⑤ 팀이 최종 보고서(summary.md)를 함께 보고 항목마다 결정: 고침 / 그대로 둠(이유) / 미룸
⑥ 결정한 것만 담당자가 고쳐서 PR
```

| 산출물 | 위치 |
|---|---|
| 스크립트 결과 | `verification/reports/{YYYY-MM-DD}/00-scripts.txt` |
| 검증 팀원 보고서 | `verification/reports/{YYYY-MM-DD}/01-consistency.md` … `05-api-ops.md` |
| 최종 보고서 | `verification/reports/{YYYY-MM-DD}/summary.md` |

검증 팀원이 쓰는 파일은 **자기 보고서 하나뿐**이다. `docs/`와 `scripts/`는 고치지 않는다.

---

## 3. 실행할 때

- 팀 에이전트 기능을 쓰면 팀원마다 해당 지시문 파일의 내용을 그대로 지시로 준다. 모델은 위 표대로 지정한다.
- 팀원이 지시문을 직접 읽게 하려면: "verification/prompts/common.md 와 verification/prompts/0X-….md 를 읽고 그대로 수행해" 한 줄이면 된다.
- 모든 팀원에게 같은 기준 커밋을 쓰게 한다: 시작할 때 `git rev-parse --short HEAD` 값을 보고서 맨 위에 적는다.
