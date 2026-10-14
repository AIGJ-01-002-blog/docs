# 23. 배포·운영

[← 기능별 아키텍처 목차](../70-feature-architecture.md)

**한 줄 요약**: `main`에 합쳐지면 GitHub Actions가 테스트 → 이미지(amd64·arm64) → GHCR → kustomize로 쿠버네티스 배포까지 하고, 새 파드가 준비되지 않으면 직전 버전으로 자동 롤백합니다. `develop`은 스테이징(`dev.devlog.life`)에, CHANGELOG가 바뀌면 릴리스 노트가 자동으로 나갑니다.

## 구성

```mermaid
flowchart LR
    DEV["개발자<br/>feature/* → develop → main"]
    subgraph GHA["GitHub Actions"]
        CI["backend-ci · frontend-ci"]
        DP["deploy.yml<br/>validate · image · deploy"]
        RL["release.yml<br/>태그 · GitHub Release"]
        NT["ci-notify · pr-notify"]
    end
    GHCR[("GHCR 이미지")]
    subgraph K8S["쿠버네티스 (selfhosted 오버레이)"]
        ING["인그레스 · Cloudflare 터널"]
        APP["blog-app Deployment<br/>2개, HPA 2~4, PDB"]
        DEP["in-cluster-deps<br/>PostgreSQL · Redis · MinIO<br/>ollama-embed · pg-backup 03:30"]
        STG["blog-dev 네임스페이스<br/>스테이징 1개"]
    end
    DC["Discord · Telegram 알림"]

    DEV --> CI
    DEV --> DP --> GHCR --> APP
    DP --> STG
    DEV --> RL --> DC
    NT --> DC
    ING --> APP --> DEP
```

## 처리 흐름: main 배포와 자동 롤백

```mermaid
sequenceDiagram
    autonumber
    participant G as GitHub Actions
    participant H as GHCR
    participant K as 쿠버네티스
    participant S as rollout.sh
    G->>G: 백엔드·화면 테스트, 비밀값 검사, kustomize 빌드·스키마 검사
    G->>H: 이미지 빌드 (amd64·arm64) 와 푸시
    G->>K: 비밀값·이미지 받기 비밀 준비
    G->>S: 배포 시작
    S->>K: 지금 리비전 기록, kubectl apply --server-side
    S->>K: rollout status (300초 기다림)
    alt 새 파드가 모두 readiness 통과
        K-->>S: 완료 (옛 파드는 새 파드가 준비된 뒤에 내림)
    else 시간 초과
        S->>K: rollout undo 직전 리비전
    end
    G->>K: Cloudflare 터널 적용
```

## 처리 흐름: 목표 단위 Git Flow 릴리스

```mermaid
flowchart LR
    I["목표 이슈<br/>예: v1.69.0"] --> F["feature/* 브랜치<br/>develop 으로 PR"]
    F --> T["테스트 통과 → develop 머지<br/>버전 안 올림"]
    T --> S["스테이징 배포<br/>dev.devlog.life"]
    T --> C{"목표 이슈가 다 닫힘?"}
    C -->|아니오| F
    C -->|예| R["릴리스 PR develop → main<br/>다음 마이너 버전 · CHANGELOG"]
    R --> D["운영 배포"]
    R --> N["release.yml<br/>태그 · 릴리스 노트 · PR 표 · 화면 사진"]
    H["급한 장애·보안"] --> HF["hotfix/* → main<br/>패치 버전"] --> D
```

## 핵심 규칙

| 항목 | 규칙 |
|---|---|
| 이미지 | 3단계 빌드(node 22 → JDK 21 → JRE 21), 레이어드 jar, 사용자 10001, 포트 8080 |
| 헬스 체크 | liveness `/actuator/health/liveness`, readiness `/actuator/health/readiness` |
| 무중단 | `maxUnavailable: 0`, PodDisruptionBudget |
| 오버레이 | `selfhosted`(운영 기준), `nhn`(학교 공용, 보류), `local`, `staging`(`blog-dev`) |
| 네트워크 정책 | DB·Redis는 앱에서만, MinIO는 앱과 인그레스에서만 |
| 비밀값 | 저장소에 넣지 않음. 쿠버네티스 Secret·GitHub Secret, `check-no-secrets.sh`가 CI에서 검사 |
| DB 변경 | Flyway V1~V35, 앱이 뜰 때 적용 |
| 백업 | `pg-backup` CronJob 매일 03:30 |
| 릴리스 노트 | CHANGELOG를 jar에 넣어 `/releases` 화면과 `GET /api/release-notes`로도 보여 줌 |

## 코드 위치

- `deploy/docker/Dockerfile`, `deploy/k8s/{base, overlays, components/in-cluster-deps, addons}`, `deploy/scripts/{rollout.sh, check-no-secrets.sh}`
- `.github/workflows/{backend-ci, frontend-ci, deploy, release, ci-notify, pr-notify}.yml`
- `CONTRIBUTING.md`, `.github/RELEASE_TEMPLATE.md`, `app/backend/.../release/ReleaseNotes`

## 예상 질문

- **배포가 실패하면 서비스가 멈추나요?** 새 파드가 준비되기 전에는 옛 파드를 내리지 않고, 시간 안에 준비되지 않으면 직전 리비전으로 되돌립니다. `ROLLBACK_TEST=true`로 일부러 실패시켜 롤백을 확인할 수 있습니다.
- **문서만 고쳐도 배포되나요?** 그래서 설계 문서는 이 docs 저장소로 나눴습니다. 코드 저장소에서도 README·CHANGELOG·specs만 바꾸는 PR은 버전을 올리지 않습니다.
