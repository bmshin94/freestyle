# Freestyle 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-29
> 분석 대상: **Freestyle** — 오픈소스 음성 받아쓰기 + AI 에이전트 데스크톱 앱
> 분석 도구: Claude Code (Opus 5)

## 📌 저장소 주소

| 구분 | URL |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/freestyle |
| **원본 저장소 (업스트림)** | https://github.com/freestyle-voice/freestyle |
| 최신 릴리스 다운로드 | https://github.com/freestyle-voice/freestyle/releases/latest |
| 공식 문서 | https://docs.freestylevoice.com/user-guide |
| 다운로드 페이지 | https://freestylevoice.com/downloads |
| Discord 커뮤니티 | https://discord.gg/Fmgt5yZCDu |
| Docker 이미지 | `ghcr.io/freestyle-voice/freestyle-server` |
| 플러그인 SDK (npm) | `freestyle-voice` |
| 플러그인 생성 CLI (npm) | `create-freestyle-plugin` |
| 라이선스 | MIT |

> 참고: 현재 브랜치는 `claude/great-heisenberg-g9laxe`, 업스트림 커밋 위에
> `CLAUDE.md` 페르소나 가이드가 PR #1로 머지되어 있는 상태입니다.

---

## 1. 이게 뭐하는 프로젝트인가

### 한 줄 요약
단축키를 누르고 말하면, 어떤 앱이든 커서 위치에 **AI가 다듬은 텍스트**가 자동으로
붙여넣기 되는 데스크톱 앱. 여기에 **예약 작업 · 앱 연동 · MCP · 에이전트
워크스페이스(Remix)** 가 얹혀 있다.

### 동작 예시
입력(음성):
> "어... 그 저기, 오늘 배포가 좀 밀릴 것 같은데요, 음... API 게이트웨이 쪽에서
> 타임아웃이 나서 그거 좀 보고 있습니다"

출력(붙여넣기된 텍스트):
> "오늘 배포가 다소 지연될 것 같습니다. API Gateway에서 타임아웃이 발생하여
> 원인을 확인 중입니다."

필러워드 제거 + 문장 정리 + 사전 치환(`에이피아이` → `API`)이 자동으로 일어난다.

### 핵심 기능
- **음성 받아쓰기** — 푸시투토크 / 토글, 붙여넣기 or 복사 선택
- **AI 클린업** — 문법·구두점 정리, 필러워드 제거
- **사전(Dictionary)** — 전사 후 문구 치환 (`type script` → `TypeScript`)
- **어휘(Vocabulary)** — 고유명사/전문용어 인식률 향상
- **포맷(Formats)** — 앱별 톤 자동 전환 (Gmail=격식, Slack=캐주얼)
- **Remix** — 질문/리서치/초안 작성용 에이전트 워크스페이스
- **예약 작업 & 알림** — 주기적으로 확인하고 필요할 때만 사용자 호출
- **연동 앱 & MCP** — Gmail, Google Calendar, Slack, GitHub, Notion, Drive, Sheets
- **다국어 & 번역** — 한국어로 말하고 영어로 출력 가능
- **Notes / Tasks / Brain** — 평문 메모와 편집 가능한 장기 기억
- **승인 게이트** — 이메일 발송·파일 변경·명령 실행은 항상 명시적 승인 요구
- **플러그인 & SDK** — 전사 파이프라인 확장

---

## 2. 저장소 구조 전수조사

규모: 파일 1,281개 / TypeScript·TSX 약 **130,858줄**

```
freestyle/
├── apps/
│   ├── electron/   💻 데스크톱 앱 (Win/Mac/Linux) — 메인 제품
│   ├── server/     ⚙️ 내장 Hono API 서버 (핵심 두뇌)
│   ├── mobile/     📱 iOS/Android (Expo 57 + RN 0.86)
│   └── docs/       📖 문서 사이트 (Mintlify)
├── packages/
│   ├── sdk/                     🔌 플러그인 SDK (npm: freestyle-voice)
│   ├── stt/                     🗣️ STT 후처리 / 바이어스 / 토큰
│   ├── create-freestyle-plugin/ 🛠️ 플러그인 스캐폴딩 CLI
│   ├── utils/  validations/     공용 유틸 · Zod 스키마
├── plugins/
│   ├── emoji/              말투 분석 후 이모지 삽입
│   ├── profanity-filter/   비속어 필터
│   └── audio-transcription/오디오 파일 전사
├── templates/      basic / with-ui (React+Vite) 플러그인 템플릿
├── specs/          📝 내부 설계문서·감사보고서 19개
├── .agents/skills/ 🤖 AI 에이전트 스킬 8종
├── docs/superpowers/ 에이전트 플랜/스펙
├── .lore.md        😲 566KB AI 장기기억 파일
├── DESIGN.md       🎨 디자인 시스템 (색·폰트·여백 전부 규정)
└── AGENTS.md / CLAUDE.md  AI 에이전트 지침서
```

### apps/electron
- `src/main/` — OS 레벨: 전역 단축키(`key-listener.ts`), 붙여넣기(`paste.ts`),
  떠다니는 음성 필 UI(`pill-position.ts`, `sprites/`), 트레이, 권한 체크
- `src/renderer/` — React 19 + Tailwind + Radix UI + Motion 화면
- `native/` — C / Swift 네이티브 바이너리 (접근성 API)
- 테스트 — Vitest(단위) + Playwright(E2E·비주얼)

### apps/server — 라우트 34개
| 라우트 | 역할 |
|---|---|
| `transcribe.ts` | 음성 → 텍스트 |
| `post-process-route.ts` | LLM 문장 다듬기 |
| `remix.ts` / `remix-sessions.ts` | 에이전트 워크스페이스 |
| `agent.ts` / `agent-os.ts` | 에이전트의 로컬 파일/셸 실행 |
| `mcp.ts` | MCP 서버 연결 (stdio / HTTP / OAuth) |
| `scheduled.ts` | 예약 작업 프록시 |
| `connectors.ts` | Gmail/Slack/Notion 등 연동 프록시 |
| `brain.ts` | 장기 기억 |
| `plugins.ts` | 플러그인 설치·관리 |
| `api-keys.ts` | BYOK 키 저장 (last-4만 노출) |
| `whisper.ts` / `mlx-asr.ts` | 로컬 음성모델 실행 |

데이터는 전부 **로컬 SQLite**(`lib/db.ts`)에 저장 — 프라이버시 우선 설계.

---

## 3. 플러그인 파이프라인 (SDK 훅)

```
단축키 누름
   │
[앱]    오디오 녹음 ................ event: recordingStarted
   │
[서버]  음성인식(STT)
   ├─ beforeTranscribe   오디오 전처리 / 모델·언어 교체 / consume
   ↓ 원본 전사
   ├─ afterTranscribe    원문 수정 / "음성 명령"으로 가로채기
   │                     event: transcribed
[서버]  AI 클린업 (선택)
   ├─ beforeCleanup      프롬프트·톤(destination) 조작 / skip
   ├─ 사전 치환 + LLM 리라이트
   ├─ afterCleanup       ⭐ 최종 텍스트 변환 (핵심 훅)
   │                     event: cleaned
[앱]    출력
   ├─ beforeOutput       텍스트 수정 / paste·copy·none 결정
   │                     event: outputDelivered
   ↓
내가 쓰던 앱에 텍스트 등장
```

- 모든 훅은 세 번째 인자 `api`를 받음
  - `api.control.stopPropagation()` — 이 훅에 한해 뒤 플러그인 중단
  - `api.control.consume()` / `abort()` — 파이프라인 전체 중단
  - `api.llm` — 호스트가 설정한 LLM 사용 (서버 훅 한정)
- `enforce: "pre" | "post"` 로 Vite처럼 실행 순서 제어
- `middleware` 로 Hono 미들웨어 기여 가능
- 플러그인은 서버/앱 양쪽 프로세스에 로드되고, 훅은 자기 자리에서만 실행됨

---

## 4. 기술 스택

| 영역 | 기술 |
|---|---|
| 데스크톱 | Electron + electron-vite + electron-builder |
| 프론트 | React 19, Tailwind, Radix UI, Motion, react-router 7, i18next |
| 백엔드 | Hono 4, Zod 4, SQLite, better-auth |
| AI | Vercel AI SDK 6 (OpenAI / Anthropic / Google / Groq / Mistral / ElevenLabs) |
| 로컬 STT | whisper.cpp, MLX(Apple Silicon), HuggingFace Hub |
| 모바일 | Expo 57 + React Native 0.86 |
| 도구 | pnpm 10, Turborepo, Biome, Vitest, Playwright, Knip, Husky |
| 모니터링 | Sentry (electron / node / react-native) |
| 릴리스 | Sentry Craft + GitHub Actions 8개 워크플로 |
| 라이선스 | **MIT** |

---

## 5. 설치 및 사용법

### 방법 A — 그냥 쓰기
1. https://github.com/freestyle-voice/freestyle/releases/latest 에서 다운로드
   - macOS(Apple Silicon / Intel) `.dmg` · Windows `.exe` · Linux `.AppImage` / `.deb`
2. 설치 후 실행 → **마이크** + **접근성(Accessibility)** 권한 허용
   - macOS는 접근성 권한이 없으면 다른 앱 붙여넣기가 동작하지 않음
3. 로그인 → 단축키 설정 → 사용 시작

### 방법 B — 소스 빌드 / 개발
```bash
# 사전 준비: Node.js 22+, pnpm 10+
git clone https://github.com/bmshin94/freestyle.git
cd freestyle
pnpm install
pnpm dev          # Electron + 내장 서버 핫리로드
```

기타 명령어
```bash
pnpm dev:docs     # 문서 사이트
pnpm dev:ios      # iOS
pnpm dev:android  # Android
pnpm test         # 전체 테스트
pnpm format       # Biome
pnpm knip         # 미사용 코드 탐지

pnpm --filter @freestyle-voice/electron build:mac   # build:win / build:linux
```

### 방법 C — 서버만 Docker (헤드리스 / 팀 공유 / API 백엔드)
```bash
docker run -d --name freestyle -p 4649:4649 \
  -v freestyle-data:/data \
  ghcr.io/freestyle-voice/freestyle-server

curl http://localhost:4649/api/health
# {"status":"ok","name":"freestyle"}
```

| 환경변수 | 기본값 | 설명 |
|---|---|---|
| `FREESTYLE_DB_PATH` | `/data/freestyle.db` | **필수** — SQLite 경로 |
| `PORT` | `4649` | 포트 |
| `HOST` | `0.0.0.0` | 바인딩 인터페이스 |
| `FREESTYLE_ENV` | `production` | 환경 플래그 |
| `DO_NOT_TRACK` | (없음) | `1` 이면 텔레메트리 전면 차단 |

---

## 6. 플러그인 / 스킬 / MCP — 정체 구분

**결론: 셋 다 아니고 독립 실행 "앱(호스트)"이다. 다만 셋을 모두 품고 있다.**

| 구분 | Freestyle에서는 | 설명 |
|---|---|---|
| 플러그인 | ✅ **자체 규격** (`freestyle-voice` SDK) | Claude 플러그인이 아니라 Freestyle 전용 플러그인 |
| 스킬 | ✅ `.agents/skills/` 8종 | 제품 기능이 아니라 **이 레포를 개발할 때 쓰는 AI 보조도구** |
| MCP | ✅ **클라이언트** 역할 | 외부 MCP 서버에 접속하는 쪽. Freestyle 자체가 MCP 서버는 아님 |

VS Code 비유
```
VS Code (앱)          ←→  Freestyle (앱)
VS Code 확장           ←→  Freestyle 플러그인 (freestyle-voice SDK)
VS Code가 LSP 사용     ←→  Freestyle이 MCP 사용
```

MCP 구현 (`apps/server/src/lib/mcp/`)
- `client.ts` / `oauth.ts` / `store.ts`
- 지원: 로컬 **stdio** 서버, 원격 **Streamable HTTP** 서버
- 인증: 없음 / Bearer / 커스텀 헤더 / **OAuth 인증코드 플로우**
- 프라이버시: 자격증명·엔드포인트·토큰은 로컬 서버에만 보관하고,
  클라우드에는 **도구 이름·설명·입력 스키마만** 전달. 실행은 로컬, 결과만 정제 전송.

---

## 7. API 토큰이 필요한가

| 모드 | API 키 | 비용 | 인터넷 |
|---|---|---|---|
| ① Freestyle Transcribe (기본) | ❌ 불필요 (로그인만) | 무료티어 + 유료 | 필요 |
| ② BYOK (내 키 사용) | ✅ 필요 | 사용량만큼 | 필요 |
| ③ 로컬 모델 | ❌ 불필요 | **0원** | **불필요** |

### ① Freestyle Transcribe
공식 문서: *"Sign in and it works — there is nothing to configure and no API key to paste."*
전사 + 클린업을 한 번에 처리, zero-day retention.
로드맵상 요금제 계획: 무료 주당 1,000단어 / Pro $9월 무제한.

### ② BYOK 지원 프로바이더 (코드 확인)
- **STT**: Groq, Deepgram, ElevenLabs, Mistral
- **LLM**: OpenAI, Anthropic, Google, Groq, Mistral, OpenRouter, Vercel

키는 로컬 SQLite `api_keys` 테이블에 저장되고 UI에는 마지막 4자리만 노출된다.
```ts
hint: key.length > 8 ? `…${key.slice(-4)}` : "…"
```

### ③ 로컬 모델 (완전 무료)
- `lib/whisper/` — whisper.cpp, 전 플랫폼
- `lib/mlx-asr/` — MLX, Apple Silicon 전용 (초고속)
- 모델은 `@huggingface/hub`로 자동 다운로드 (`lib/hf/`), 이후 오프라인 동작
- LLM도 `local-llm` 프로바이더로 Ollama / LM Studio 연결 가능

추천
```
프라이버시 우선  → ③ 로컬 (Whisper + Ollama)   → 0원
속도/가성비      → ② BYOK (Groq)               → 월 수천 원
가장 편하게      → ① Freestyle Transcribe       → 무료티어
```

---

## 8. 왜 GitHub에서 주목받는가

> 본 분석 세션은 `bmshin94/freestyle`만 접근 가능하여 **원본 저장소의 정확한 스타
> 수는 확인하지 못했습니다.** 아래는 코드/문서 근거로 정리한 인기 요인입니다.

1. **유료 SaaS(Wispr Flow, Superwhisper 월 $12~15)의 MIT 오픈소스 대안** 포지션
2. **개발자 정조준** — "4X faster", "코딩 에이전트에 바로 받아쓰기"
3. **로컬 우선 프라이버시** — 로컬 SQLite, 로컬 모델, `DO_NOT_TRACK=1`,
   MCP 자격증명 비전송
4. **플러그인 생태계** — `npx create-freestyle-plugin` 한 줄로 기여 진입장벽 최소화
5. **크로스플랫폼 완비** — Mac(Intel+ARM) / Windows / Linux / iOS / Android
6. **높은 코드 품질** — Vitest + Playwright(비주얼 테스트), Biome, Knip,
   Actions 8종, Craft 릴리스 자동화, Sentry
7. **AI 네이티브 개발 문화** — `.lore.md`(566KB), `AGENTS.md`, `specs/` 19종,
   `skills-lock.json`(AI 스킬 버전 잠금)
8. **디자인 완성도** — `DESIGN.md`에 크림색 `#F4F0E4` 배경, Instrument Serif +
   DM Sans + JetBrains Mono, 올리브 포인트까지 규정

---

## 9. 로컬 에이전트 구축에 도움이 되는가 — 매우 그렇다

### 그대로 재사용 가능한 부품

| 부품 | 경로 | 가치 |
|---|---|---|
| **에이전트 OS 샌드박스** | `lib/agent-os.ts` | ⭐⭐⭐ 안전한 셸/파일 실행 계층 |
| **MCP 클라이언트** | `lib/mcp/` | stdio + HTTP + OAuth 완성품 |
| **내구성 실행 인프라** | `remix-durable-queue.ts`, `remix-execution-reservations.ts`, `remix-cancel-outbox.ts`, `agent-stream-store.ts`, `agent-message-queue.ts` | 끊기지 않는 에이전트 실행 |
| **메모리 시스템** | `routes/brain.ts`, `.lore.md` | 사용자가 읽고 수정·삭제 가능한 기억 |
| **승인 게이트** | 제품 전반 | Human-in-the-loop 안전장치 |
| **능동형 트리거** | `routes/scheduled.ts`, `attention.ts`, `specs/attention-run-center.md` | 알아서 깨어나 필요할 때만 호출 |

`agent-os.ts`의 안전장치 상수 (그대로 참고할 만한 값)
```ts
const FILE_MAX_CHARS   = 60_000;   // 파일 읽기 상한
const BASH_TIMEOUT_MS  = 30_000;   // 셸 타임아웃
const BASH_OUTPUT_CAP  = 8_192;    // 출력 절단
const WALK_MAX_ENTRIES = 1_000;    // 디렉터리 순회 상한
const GREP_MAX_MATCHES = 60;       // grep 매치 상한
const SKIP_DIRS = new Set(["node_modules", ".git", ".Trash", "Library"]);
```
에러 카테고리도 분류되어 있다.
```ts
type AgentCommandCategory =
  | "success" | "shell-unavailable" | "command-failed"
  | "permission-denied" | "not-found" | "timed-out";
```

### 구축 권장 순서
```
1) agent-os.ts 이식        → 파일/셸 도구 확보
2) lib/mcp/ 이식            → 외부 도구 연결
3) Hono + AI SDK 라우트      → 에이전트 루프
4) 승인 게이트 패턴 적용      → 안전장치
5) durable-queue 참고        → 끊김 없는 실행
6) .lore.md 방식 도입        → 장기 기억
```

### 타 프레임워크 비교
| | LangChain | AutoGPT | **Freestyle** |
|---|---|---|---|
| 언어 | Python 중심 | Python | **TypeScript** |
| 데스크톱 통합 | ✗ | ✗ | **Electron 완비** |
| MCP 지원 | 부분 | ✗ | **완전** |
| 실제 출시 제품 | 라이브러리 | 실험 | **실제 앱** |
| 음성 입출력 | ✗ | ✗ | **핵심 기능** |

참고 문서: `specs/remix-opencode-comparison-2026-09-10.md` (다른 에이전트와의 비교 분석)

---

## 10. React / PHP로 만들 수 있는가

### React — 이미 사용 중 (React 19)
| 위치 | 기술 |
|---|---|
| `apps/electron/src/renderer` | React 19 + Tailwind + Radix UI + Motion |
| `apps/mobile` | React Native 0.86 + Expo 57 |
| `plugins/emoji/ui` | React 19 + Vite (플러그인 설정 UI) |
| `templates/with-ui` | React 스타터 템플릿 |

```bash
npx create-freestyle-plugin my-plugin   # with-ui 선택 시 React + Vite 프로젝트 생성
```

### PHP — 직접 통합은 불가, HTTP 연동은 가능
불가: 플러그인 본체(Node.js ESM 전용 SDK), Electron 메인 프로세스, 실시간 파이프라인 훅

가능 ①: Docker 서버를 음성 API로 사용
```php
<?php
$ch = curl_init('http://localhost:4649/api/transcribe');
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, ['audio' => new CURLFile($audioPath)]);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$result = json_decode(curl_exec($ch), true);
echo $result['text'];
```

가능 ②: 얇은 Node 플러그인이 PHP 백엔드를 호출하는 브리지
```ts
afterCleanup: async (input, output) => {
  const res = await fetch("https://my-php-server.com/api/process", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: output.text }),
  });
  const { text } = await res.json();
  output.text = text;   // PHP가 가공한 결과로 교체
}
```

| 만들 것 | React | PHP |
|---|---|---|
| 플러그인 본체 | ✗ (TS 필요) | ✗ |
| 플러그인 설정 UI | ✅ | ✗ |
| 앱 화면 수정 | ✅ | ✗ |
| 모바일 앱 | ✅ (RN) | ✗ |
| 웹 대시보드 | ✅ | ✅ |
| 음성 API 활용 서비스 | ✅ | ✅ |
| 후처리 로직 서버 | ✅ | ✅ |

---

## 11. 수익화 아이디어

> MIT 라이선스: 상업적 이용 / 수정 / 재배포 / 비공개 소스화 모두 허용.
> 의무는 **저작권 고지 + 라이선스 문구 포함**뿐.

### 아이디어 1. 한국어 특화 포크 (최우선 추천)
경쟁 제품들의 한국어 처리(조사·존댓말·외래어)가 약하고, 국내 경쟁자가 거의 없음.
- 한국어 개발 용어 사전 기본 탑재 (`타입 스크립트` → `TypeScript`)
- 존댓말/반말 톤 자동 전환 (Slack / 이메일 / 보고서)
- 한국 업무 앱 프리셋 (카카오워크, 잔디, 네이버웍스, 노션)
- 회의록 모드: 화자 분리 + 액션아이템 추출

가격: Free 0원(주 1,000단어) / Pro 월 9,900원 / Team 1인 월 7,900원
예상: 유료 1,000명 → 월 약 990만원 · 난이도 중 · 기간 2~3개월

### 아이디어 2. 전문 직군용 유료 플러그인 (진입 가장 쉬움)
앱 본체를 건드리지 않고 플러그인만 제작 → npm 배포 + 라이선스 키
- 의료 받아쓰기 (월 $29) — 의학용어 교정, SOAP 노트, 환자정보 마스킹
- 법률 문서 (월 $39) — 법조문 인용 포맷, 판례 번호, 계약 조항 템플릿
- 개발자 코드 딕테이션 (월 $12) — 언어별 문법, camelCase 변환, 커밋 포맷
- CS/영업 통화 (월 $49) — 통화 요약, CRM 자동 입력, 감정 분석

예상: 3종 × 각 100명 → 월 $3,000~5,000

### 아이디어 3. Docker 서버를 음성 API 백엔드로
`docker run -p 4649:4649 ...` → `POST /api/transcribe` 가 그대로 음성 API가 된다.
- 회의록 SaaS (월 29,000원) / 유튜브 자막 생성 (건당 2,000원)
- 팟캐스트 텍스트화 (월 19,000원) / 콜센터 QA (기업 견적)
- 강의 노트 서비스 (월 9,900원)

마진 예시: Groq Whisper 1시간 ≈ $0.04(약 55원), 판매가 3,000원 → 마진율 98%

### 아이디어 4. 팀/기업용 SaaS 래퍼
오픈소스에 없는 것을 얹어 판매: 팀 사전 중앙관리, 관리자 대시보드, SSO/SAML,
감사 로그, 온프레미스, 데이터 레지던시, SLA
가격: Team 1인 월 15,000원 / Business 1인 월 25,000원 / Enterprise 연 3,000만원~
타깃: 병원, 로펌, 콜센터, 보험사, 공공기관

### 아이디어 5. 프리셋/템플릿 마켓 (착수 비용 최저)
공식 로드맵에 "역할별 템플릿" 항목이 있어 수요가 검증된 영역.
개발자 팩 9,900원 / 의료진 팩 29,900원 / 마케터 팩 14,900원 /
학생 팩 4,900원 / 유튜버 팩 19,900원 → 월 200개 판매 시 200~400만원

### 아이디어 6. 교육 콘텐츠 & 기술 브랜딩
강의("Electron + AI 데스크톱 앱") 99,000원, 유료 뉴스레터 월 9,900원,
Notion 템플릿 29,000원, 유튜브 코드리뷰 시리즈
커리큘럼: 모노레포 → Electron 3프로세스 → Hono/Zod/SQLite → AI SDK →
플러그인 훅 설계 → MCP 클라이언트 → `.lore.md` 문서체계 → 릴리스 자동화
예상: 수강생 300명 × 99,000원 → 2,970만원(1회성)

### 아이디어 7. 기업 도입 컨설팅 / SI
온프레미스 구축 건당 500~2,000만원, 커스텀 플러그인 300~1,000만원,
ERP/CRM 연동 500만원~, 연간 유지보수 600~1,200만원

### 종합 비교
| # | 아이디어 | 난이도 | 초기비용 | 수익까지 | 월 예상수익 |
|---|---|---|---|---|---|
| 1 | 한국어 특화 포크 | 중상 | 중 | 3개월 | 500~2,000만원 |
| 2 | 유료 플러그인 | 중 | 낮음 | 1개월 | 300~600만원 |
| 3 | 음성 API 서비스 | 중상 | 중 | 2개월 | 300~1,500만원 |
| 4 | 팀 SaaS | 상 | 높음 | 6개월 | 1,000~3,000만원 |
| 5 | 프리셋 마켓 | 하 | 최저 | 2주 | 100~400만원 |
| 6 | 교육 콘텐츠 | 중 | 낮음 | 2개월 | 300~1,000만원 |
| 7 | 컨설팅/SI | 상 | 낮음 | 즉시~ | 건당 수백~수천만원 |

### 권장 로드맵
```
1개월차    프리셋 팩 1종 출시 → 시장 반응 테스트, 첫 수익
2~3개월차  개발자용 유료 플러그인 출시 + 블로그/유튜브로 인지도 확보
4~6개월차  한국어 포크 또는 음성 API 서비스로 확장
7개월차~   팀 SaaS 또는 기업 컨설팅
```

### 법적 체크포인트
- MIT 고지 필수 — LICENSE 파일 포함 + 저작권 문구 유지
- 상표권 주의 — "Freestyle" 명칭 그대로 사용하지 말고 별도 브랜딩
- 업스트림 기여 — 버그 수정은 PR로 환원하면 커뮤니티 평판에 유리
- 개인정보 — 음성은 민감정보. 국내 개인정보보호법 준수 필요
- 규제 산업 — 의료(HIPAA 등)·법률 분야는 별도 규제 확인 필수

---

## 12. 핵심 결론 3줄

1. **사용자 관점** — 말하면 정리된 문장이 나오는 음성 입력기 + 자동화 비서
2. **학습자 관점** — Electron / React / Hono 실무급 모노레포 교과서
3. **제작자 관점** — 플러그인 SDK와 Docker 서버로 바로 수익화 가능한 발판 (MIT)
