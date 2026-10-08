# 집 PC AI(Ollama) 연결하기

작성 2026-10-08 · 근거: 민서님 답 4번 변경("집 PC(12GB GPU, Ollama) 기본 + Gemini 보조"). 코드는 spec 052(v1.27.0)에 들어 있고, 이 문서대로 설정만 하면 붙습니다.

## 어떻게 동작하나요

| 상황 | 쓰는 AI |
|---|---|
| 집 PC가 켜져 있음 | 집 PC Ollama (태그 추천, 텔레그램 메모 다듬기) |
| 집 PC가 꺼져 있거나 응답 없음 | 그 요청은 Gemini가 처리하고, 60초 동안 집 PC를 건너뜀(`OLLAMA_DOWN_FOR`) |
| 집 PC가 바쁨(동시 처리 수 초과) | Gemini |
| Gemini 하루 한도도 다 씀 | "지금은 추천할 수 없어요" |

- MCP `write_devlog`는 글을 쓰는 AI가 **사용자 쪽 AI**(Claude·ChatGPT 등)라 집 PC·Gemini를 쓰지 않습니다. 그래서 회원이 늘어도 집 PC에 줄이 서지 않습니다. 집 PC가 맡는 건 태그 추천·메모 다듬기, 그리고 앞으로 들어올 임베딩(의미 검색)입니다.
- 임베딩·요약 같은 뒤처리는 **의미 검색 기능(다음 spec)** 과 함께 Redis 대기열로 넣습니다. 임베딩 모델은 하나(`bge-m3` 추천)로 고정하고, 집 PC가 꺼져 있으면 새 글은 켜진 뒤에 의미 검색에 반영됩니다. 같은 글을 두 번 처리해도 결과가 같게(덮어쓰기) 만듭니다.
- 예전처럼 Gemini를 먼저 쓰려면 서버에 `AI_PREFER=gemini`를 넣으면 됩니다.

## 1. 집 PC에 Ollama 설치 (민서님)

1. [ollama.com](https://ollama.com)에서 Windows용을 설치합니다. 설치하면 `http://localhost:11434`에서 돌고, 이 주소는 PC 밖에서 보이지 않습니다(그대로 두세요).
2. 모델을 받습니다. 12GB GPU면 14B급 4비트가 맞습니다.
   ```powershell
   ollama pull qwen2.5:14b      # 대화·태그·요약 (약 9GB)
   ollama pull bge-m3           # 임베딩 (의미 검색용, 나중에 씀)
   ollama run qwen2.5:14b "스프링 부트 태그 3개만"
   ```
   답이 몇 초 안에 나오면 됩니다. 느리면 `qwen2.5:7b`로 낮춥니다.
3. 맥북(M1 16GB)은 개발·시험용입니다. 로컬에서 띄울 때는 `OLLAMA_BASE_URL=http://localhost:11434`, `OLLAMA_MODEL=qwen2.5:7b`면 충분합니다.

## 2. Cloudflare Tunnel로 열기 (민서님)

Ollama 주소를 그냥 인터넷에 열면 누구나 GPU를 쓸 수 있으니, **터널 + 접근 토큰** 뒤에 둡니다.

1. 집 PC에 `cloudflared`를 설치하고 로그인합니다.
   ```powershell
   winget install --id Cloudflare.cloudflared
   cloudflared tunnel login
   cloudflared tunnel create devlog-ai
   cloudflared tunnel route dns devlog-ai ai.devlog.life
   ```
2. `%USERPROFILE%\.cloudflared\config.yml`을 만듭니다(터널 번호는 create 결과의 값).
   ```yaml
   tunnel: <터널 번호>
   credentials-file: C:\Users\<사용자>\.cloudflared\<터널 번호>.json
   ingress:
     - hostname: ai.devlog.life
       service: http://localhost:11434
       originRequest:
         httpHostHeader: localhost:11434   # Ollama는 다른 Host 이름을 거절한다
     - service: http_status:404
   ```
3. PC를 켜면 자동으로 돌게 서비스로 등록합니다: `cloudflared service install`.
4. **접근 토큰 걸기**: Cloudflare 대시보드 › Zero Trust › Access › Applications에서 `ai.devlog.life`를 Self-hosted 앱으로 만들고, 정책 동작을 **Service Auth**로, 허용 대상은 새로 만든 **Service Token**으로 합니다. 이때 나오는 Client ID와 Client Secret을 GitHub Secrets 탭에 넣습니다(아래 3단계). 이 토큰이 없는 요청은 Cloudflare가 막습니다.

## 3. 서버 설정 (인프라)

| 이름 | 값 | 넣는 곳 |
|---|---|---|
| `OLLAMA_BASE_URL` | `https://ai.devlog.life` | GitHub › Settings › Secrets and variables › Actions › **Variables** 탭 (비우면 집 PC AI를 끄고 Gemini만 씀) |
| `OLLAMA_ACCESS_CLIENT_ID` | Cloudflare Access 서비스 토큰 Client ID | 같은 화면의 **Secrets** 탭 |
| `OLLAMA_ACCESS_CLIENT_SECRET` | Cloudflare Access 서비스 토큰 Client Secret | **Secrets** 탭 |
| `OLLAMA_AUTH_TOKEN` | Ollama 앞에 토큰 확인 프록시를 따로 둘 때만 (Bearer) | **Secrets** 탭 |
| `GEMINI_API_KEY` | 집 PC가 꺼졌을 때 쓰는 Gemini 키 | **Secrets** 탭 |
| `OLLAMA_MODEL` | `qwen2.5:14b` | 배포 설정(app.env)에 이미 들어 있음 |
| `OLLAMA_CONCURRENCY` | `1` (12GB GPU는 한 번에 하나) | 배포 설정에 이미 들어 있음 |
| `OLLAMA_DOWN_FOR` | `60s` | 배포 설정에 이미 들어 있음 |
| `AI_PREFER` | `local` (기본값, 생략 가능) | 넣지 않아도 됨 |

주소는 비밀이 아니므로 Secrets가 아니라 Variables 탭에 넣습니다. Secrets 값은 배포 때 서버 비밀값(secret.env)으로만 들어가고, git·문서·메모리에는 쓰지 않습니다.

## 4. 확인

- 서버에서 `curl -H "CF-Access-Client-Id: …" -H "CF-Access-Client-Secret: …" https://ai.devlog.life/api/tags`가 모델 목록을 돌려주면 연결된 것입니다. 토큰 없이 부르면 Cloudflare 로그인 화면(403)이 나와야 합니다.
- 글쓰기 화면의 [AI 태그 추천]에서 "조금 걸려요"(자체 AI) 안내가 보이면 집 PC를 쓰고 있는 것입니다. PC를 끄면 다음 요청부터 Gemini로 넘어갑니다.
