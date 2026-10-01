# Superlog 전수조사 분석 리포트 (한국어)

> 작성: Claude Code 세션 (페르소나: 카리나 / CLAUDE.md 기준)
> 작성일: 2026-10-01
> 분석 대상 커밋: `5db1fed` (Merge pull request #1 from bmshin94/feat/claude-guide)

## 📎 관련 GitHub / 링크 모음

| 구분 | 주소 |
|---|---|
| **이 레포 (오빠 것)** | https://github.com/bmshin94/superlog |
| 원본(업스트림) 레포 | https://github.com/superloglabs/superlog |
| 설치용 Agent Skills 레포 | https://github.com/superloglabs/skills |
| OTel 헬퍼 레포 | https://github.com/superloglabs/otel-helpers |
| 공식 웹사이트 | https://superlog.sh |
| MCP 엔드포인트 | https://api.superlog.sh/mcp |
| Discord 커뮤니티 | https://discord.gg/wJ56aRh8hx |
| MCP Toplist 등재 | https://mcptoplist.com/server/sh.superlog%2Fsuperlog |
| 라이선스 | Apache License 2.0 (`LICENSE.md`) |

---

## 1. 한 줄 정의

> **Superlog = 내 서버 장애를 AI 에이전트가 스스로 조사하고, 원인을 찾아 GitHub PR까지 올려주는 오픈소스 관측(Observability) 플랫폼**

Datadog / Sentry 같은 모니터링 도구 + Claude 기반 조사 에이전트의 결합.
만든 곳은 Superlog Labs (Y Combinator P26 배치).

## 2. 레포 기본 정보

| 항목 | 값 |
|---|---|
| 라이선스 | Apache 2.0 (상업적 이용·수정·재배포 가능) |
| 규모 | TS/TSX 1,059 파일 / 약 231,000 줄 |
| 테스트 | 테스트 파일 390개 |
| DB | Postgres 테이블 73개, 마이그레이션 127개 |
| 분석 DB | ClickHouse 26.1 (스키마 + 마이그레이션 10개) |
| 모노레포 도구 | pnpm workspace + Turborepo + Biome + Drizzle ORM |
| 런타임 | Node.js 20+ / pnpm 9+ / Docker |

## 3. 폴더 구조

```
superlog/
├── apps/
│   ├── web/      315 files  Vite + React 19 + TS 프론트엔드
│   ├── api/      265 files  Hono HTTP API + MCP 서버 구현
│   ├── worker/   284 files  백그라운드 워커 + 에이전트 오케스트레이션
│   ├── proxy/     47 files  OTLP 텔레메트리 수집 프록시
│   └── sample/     8 files  Next.js 테스트 앱 (/api/broken 로 의도적 에러)
├── packages/
│   ├── db/              Drizzle 스키마 (73 테이블)
│   ├── telemetry-query/ ClickHouse 쿼리 빌더
│   ├── fingerprint/     에러 지문 생성 (같은 에러 묶기)
│   ├── otel/            OpenTelemetry 헬퍼
│   ├── topology/        서비스 맵 자동 생성
│   ├── billing/         가격/과금 로직 (가격의 원천)
│   ├── net-guard/       SSRF 방어
│   ├── gcp-auth/        GCP 워크로드 아이덴티티
│   └── cloudflare, railway, render/   플랫폼 커넥터
├── infra/
│   ├── clickhouse/  스키마 + 마이그레이션 (롤업 테이블, TTL)
│   ├── collector/   OTel Collector 설정
│   └── aws-connect/ AWS IAM 연동
├── specs/SPEC.md     ★ 23KB 에이전트 설계 명세서 (최고 가치)
├── docs/             GitHub App / Sentry / Webhooks 가이드
├── autumn.config.ts  가격 정책 as code (Autumn + Stripe)
├── server.json       MCP 서버 매니페스트
├── CLAUDE.md         카리나 페르소나 (사용자 직접 커밋)
└── docker-compose.yml  Postgres 16 + ClickHouse 26.1 + OTel Collector
```

## 4. 데이터 흐름

```
[내 앱] --OTel SDK--> [proxy :4101] --> [ClickHouse]  (로그/트레이스/메트릭 원본)
                                              |
                          [worker] 가 주기적으로 폴링
                                              |
   (1) 에러 지문화(fingerprint) -> "Issue" 생성
   (2) 그룹핑 에이전트(Claude)가 기존 Incident 와 같은 근본원인인지 판단
   (3) 새 원인이면 "Incident" 개설 -> "Agent Run" 시작
   (4) 에이전트가 텔레메트리(MCP) + GitHub 소스 + AWS(읽기전용) + 과거 메모리 조사
   (5) 결론 분기:
         - 진짜 버그 -> 패치 생성 -> 플랫폼이 GitHub PR 생성
         - 노이즈    -> Issue silenced
         - 일회성    -> Issue under observation + 에스컬레이션 임계치
         - 외부원인  -> report_external_cause
         - 불명확    -> ask_human (Slack 질문)
   (6) Slack / Linear / Notion 알림 + 주 1회 요약 리포트
```

## 5. 핵심 도메인 개념 (specs/SPEC.md)

```
Issue (증상)
  = 동일 에러를 묶은 단위. 종류: 에러 로그 / 에러 스팬 / 알럿 에피소드
  = 상태: open -> silenced | under observation | resolved
  = under observation 은 escalation trigger 필수 (분당 에러율 또는 누적 건수)
  = 알럿 에피소드 Issue 는 open/resolved 만 사용 (silence 불가)

Incident (문제 1건)
  = Issue 1개 이상의 묶음. 상태: open / resolved

Agent Run (조사 세션)
  = Incident 1개당 영속 세션 1개
  = PR 코멘트/머지/클로즈 같은 외부 이벤트가 오면 같은 세션이 컨텍스트 유지하며 재개

Turn (턴)
  = 종료형 툴을 성공 호출하면 턴 종료. 거부된 툴 호출은 턴을 끝내지 않음
```

### 에이전트 툴 계약 (6개)

| 툴 | 종류 | 역할 |
|---|---|---|
| `report_findings` | 비종료 | 요약 / 근본원인+신뢰도 / 영향도 / 심각도(SEV) / 핸드오프 노트. 반복 호출 가능(last-write-wins) |
| `propose_pr` | 종료 | 여러 레포에 PR 생성(레포당 1개). 패치는 `/mnt/session/outputs/` 의 unified diff 파일 |
| `complete_investigation` | 종료 | PR 생성 불가 환경에서 조사 결과만 제출. Linear 연결 시 티켓 자동 생성 |
| `ask_human` | 종료 | 사람에게 질문하고 세션 유지 상태로 대기 |
| `report_external_cause` | 종료 | 외부 장애임을 근거와 함께 보고, Incident 는 open 유지 |
| `resolve_incident` | 종료 | Incident 종결 + 연결된 모든 Issue 상태를 원자적으로 확정 |

**선행 조건**: `propose_pr`, `complete_investigation`, `report_external_cause`,
`resolve_incident` 는 `report_findings` 가 먼저 호출되지 않으면 거부됨.

## 6. 가장 중요한 발견: 오픈코어 경계선

`apps/worker/src/infra/agent-runner/backend.ts`:

```ts
if (runtime === "community") return communityRunnerBackend;  // 레포에 포함
if (runtime === "disabled")  return disabledRunnerBackend;   // 레포에 포함
if (runtime === "anthropic")                                  // 모듈이 레포에 없음!
  return loadConfiguredRunner("anthropic", "AGENT_RUNNER_ANTHROPIC_MODULE");
```

- `community` 러너는 정적 요약 한 줄만 생성 (모델명 `community/static`, 토큰 0)
- 실제 조사 + PR 생성을 하는 `anthropic` 러너는 **환경변수로 외부 모듈을 동적 import**
  하는 구조이며 그 모듈은 이 레포에 없음 → 클라우드(유료) 전용
- `apps/api/src/system-capabilities.ts` 의 `edition: "community" | "cloud" | "private"`

**단, 보조 AI 기능은 레포에 전부 구현되어 있음** (`@anthropic-ai/sdk` 직접 사용):

| 기능 | 파일 |
|---|---|
| 인시던트 그룹핑 에이전트 | `apps/worker/src/grouping/agent.ts` |
| 자동복구 판단 에이전트 | `apps/worker/src/autorecovery/agent.ts` |
| 주간 다이제스트 랭킹 | `apps/worker/src/digest/ranker.ts` |
| 토폴로지 보강 | `apps/worker/src/topology/enrich.ts` |

→ `ANTHROPIC_API_KEY` 만 넣으면 이 기능들은 셀프호스팅에서도 동작.

### 3층 케이크 비유

```
3층: AI 조사 에이전트 (유료 핵심)        <- anthropic 러너, 레포에 없음
2층: 인시던트 관리 + 보조 AI             <- 레포에 있음
1층: 관측 플랫폼 (수집/저장/조회/알럿)    <- 레포에 있음
```

## 7. MCP 서버 (약 45개 툴)

`server.json` 으로 등록, streamable-http 전송, `https://api.superlog.sh/mcp`.
구현: `apps/api/src/mcp/` (server / alerts / dashboards / incidents / agent-config / projects / oauth / scope-authorization)

| 그룹 | 툴 |
|---|---|
| 텔레메트리 | `query_logs`, `query_traces`, `query_metrics`, `list_services` |
| 프로젝트 | `list_projects`, `get_active_project`, `set_active_project` |
| 인시던트 | `get_incident`, `search_incidents` |
| 알럿 | `list/get/create/update/delete/preview/test_alert` |
| 대시보드 | `get_home`, `set_home_builtin`, `add_home_widget`, `add_home_link`, `update_home_layout`, `remove_home_item`, `list/get/create/update/delete_dashboard`, `set_dashboard_variables`, `add/update/delete_dashboard_widget` |
| 에이전트 설정 | `get/set_project_context`, `list/create/update/delete_agent_memory`, `get/update/preview_issue_filter`, `list/add/update/remove_agent_mcp_server`, `start_agent_mcp_oauth`, `connect_agent_mcp_client_credentials`, `disconnect_agent_mcp_oauth`, `test_agent_mcp_server` |

인증: OAuth 2.1(PKCE) 또는 Personal Access Token 베어러.
스코프 기반 **읽기전용 / 텔레메트리전용** 세션 분리 — 읽기전용이면 쓰기 툴을 애초에 등록하지 않음.

### 클라이언트 연결 명령 (실제 코드에서 발췌)

```bash
# Claude Code
claude mcp add --transport http superlog https://api.superlog.sh/mcp

# Codex
codex mcp add superlog --url https://api.superlog.sh/mcp
codex mcp login superlog
```

```json
// Cursor
{ "mcpServers": { "superlog": { "url": "https://api.superlog.sh/mcp" } } }
```

## 8. 설치 및 사용법

### 방법 A — 클라우드 (5분)

1. https://superlog.sh 가입 (무료 티어: 조사 50회/월)
2. 프로젝트 생성 → Ingest Key 발급
3. OTel SDK 연결, 또는 Vercel/Railway/Render/Cloudflare 는 원클릭 OAuth 연동
4. 코딩 에이전트로 자동 설치: `npx skills add superloglabs/skills --all`

### 방법 B — 셀프호스팅

```bash
git clone https://github.com/bmshin94/superlog.git
cd superlog
pnpm install
docker compose up -d
pnpm --filter @superlog/db db:migrate
pnpm dev
```

| 서비스 | URL |
|---|---|
| 웹 대시보드 | http://localhost:5173 |
| API | http://localhost:4100 |
| OTLP 인테이크 | http://localhost:4101 |
| 샘플 앱 | http://localhost:3005 |

```bash
# 환경변수
cp apps/api/.env.example    apps/api/.env
cp apps/worker/.env.example apps/worker/.env
cp apps/web/.env.example    apps/web/.env
cp apps/proxy/.env.example  apps/proxy/.env
cp packages/db/.env.example packages/db/.env

openssl rand -base64 32   # BETTER_AUTH_SECRET / STATE_SIGNING_SECRET / AGENT_SECRETS_KEY

# 유용한 명령
pnpm typecheck / pnpm lint / pnpm format
pnpm dev:portless / :status / :stop      # 워크트리별 포트·DB 분리
pnpm demo:seed:everything                 # 가짜 텔레메트리 심기 (첫 체험 필수)
```

## 9. 필요한 토큰 / 시크릿

### 클라우드 사용 시
| 토큰 | 필수 | 비고 |
|---|---|---|
| Ingest Key | 필수 | 대시보드 발급 |
| MCP PAT / OAuth | MCP 사용 시 | `POST /api/me/mcp-tokens`, 평문은 발급 시 1회만 노출 |
| Anthropic API Key | 불필요 | 클라우드가 자체 키 사용 (크레딧 과금) |

### 셀프호스팅 시
```
# 필수
DATABASE_URL, CLICKHOUSE_URL
BETTER_AUTH_SECRET      (32B)
STATE_SIGNING_SECRET    (32B)
AGENT_SECRETS_KEY       (base64 32B, 연동 시크릿 at-rest 암호화 — 없으면 연동 라우트 500)

# AI 기능
ANTHROPIC_API_KEY             (그룹핑/다이제스트/자동복구에 사용)
AGENT_RUNNER_ANTHROPIC_MODULE (모듈 자체는 레포에 없음)

# GitHub PR
GITHUB_APP_ID / GITHUB_APP_SLUG / GITHUB_APP_PRIVATE_KEY_BASE64 / GITHUB_APP_WEBHOOK_SECRET
  -> docs/github-app-setup.md 참고

# 선택 연동
SLACK_* / LINEAR_* / SENTRY_* / GOOGLE_* / CLOUDFLARE_* / VERCEL_* / RAILWAY_* / RENDER_*
RESEND_API_KEY (이메일), AUTUMN_SECRET_KEY (과금)
```

### 토큰 보안 설계 (배울 점)
- PAT 는 해시로만 저장(`hashToken`), 평문은 발급 1회 노출
- MCP OAuth: 코드 재사용 방어 + 리프레시 토큰 로테이션
- 스코프 기반 읽기전용/텔레메트리전용 분리
- 연동 시크릿 전부 `AGENT_SECRETS_KEY` 로 암호화 (커밋 #515)

## 10. AI 에이전트 구축에 참고할 패턴 10가지

1. **툴을 종료형/비종료형으로 분리** — 에이전트가 "언제 멈출지"를 툴 타입으로 해결
2. **선행 조건 강제** — `report_findings` 없이는 종료 툴 거부 (프롬프트 부탁이 아니라 코드 강제)
3. **서버 측 툴 인자 재검증** — 잘못된 호출에 모델이 읽을 수 있는 에러를 반환해 자가 교정, 동일 툴 에러는 턴당 N회 제한으로 무한루프 방어
4. **레거시 툴 이름 호환** — 은퇴한 툴(`report_failure`, `silence_as_noise` 등)도 인식해서 현재 툴로 안내
5. **프롬프트 인젝션 신뢰 경계** — `agent-content-boundary.ts` 의 `wrapUntrustedContent()` / `wrapUntrustedJsonValue()` + 버전 관리 (커밋 #516)
6. **권한 최소화** — 에이전트는 push 권한 없음. 패치만 생성하고 플랫폼이 PR 생성. 인프라 변경은 승인 프롬프트 후 플랫폼이 "승인된 명령을 그대로" 실행
7. **영속 세션 + 외부 이벤트 재개** — `agent-runs/resume.ts`, `session-termination.ts`, `recovery.ts`. 세션 만료 시 머지로 자동 해결하는 폴백
8. **러너 추상화** — community / disabled / anthropic 교체 가능 → 테스트 용이
9. **토큰 → 크레딧 → 청구** — `ai-usage.ts` → `billing/usage-notifier.ts` → Autumn/Stripe
10. **에이전트 메모리** — Comments(Issue별) + Project Memory(전역, 날짜 있는 자유 텍스트) + Project Context(아키텍처 등 영속 사실). MCP instructions 가 메모리 기록을 직접 유도

### 꼭 읽어볼 파일 Top 7
```
1. specs/SPEC.md                                  (최우선)
2. apps/worker/src/agent-outcome-tools.ts
3. apps/worker/src/agent-content-boundary.ts
4. apps/worker/src/infra/agent-runner/backend.ts
5. apps/api/src/mcp/server.ts
6. apps/worker/src/grouping/agent.ts
7. apps/worker/src/ai-usage.ts
```

### 실전 교훈: "트리거 해피" 문제
초기 피드백은 "Superlog 가 PR을 너무 남발한다" 였고, 404 에러 로그를 WARN 으로 바꾸는 PR을
계속 올려서 팀이 PR 지옥에 빠졌다. SPEC.md 는 이를 반례로 명시하고 **"코드를 고치지 말고
Issue 를 침묵시켜라"** 는 원칙을 넣었다.
→ **에이전트에게 "아무것도 하지 않기" 선택지를 반드시 줘야 한다.**

## 11. 분류: 플러그인 / 스킬 / MCP?

| 분류 | 여부 | 설명 |
|---|---|---|
| 클로드 코드 플러그인 | 아님 | plugin 매니페스트 없음 |
| Agent Skill | 부분 | 설치용 스킬은 별도 레포(`superloglabs/skills`). 이 레포의 `skills-lock.json` 은 외부 스킬(mattpocock) 1개 사용 중 |
| MCP 서버 | 맞음(일부) | `server.json` + `apps/api/src/mcp/` 전체 구현 |
| 독립 SaaS 플랫폼 | **본질** | 자체 DB/인증/과금/프론트엔드를 갖춘 완전한 제품 |

## 12. 비즈니스 모델 (`autumn.config.ts`)

| 플랜 | 가격 | 조사 크레딧 | 초과 단가 |
|---|---|---|---|
| Free | $0 | 50/월 (하드캡, 초과 시 차단) | — |
| Pay-as-you-go | 사용량 | 50 포함 | $1.50/회 |
| Pro (`pack_150`) | $150/월 | 120 | $1.25/회 |
| Max (`pack_300`) | $300/월 | 300 | $1.00/회 |

텔레메트리 종량제: 스팬 1M당 $0.50 / 로그 1M당 $0.50 / 메트릭 포인트 1M당 $0.15
(유료 플랜은 무료 티어와 동일한 포함량을 free units 로 제공 — 플랜 변경 시 사용량 캐리오버)

## 13. 연동 가능한 서비스

GitHub App · Slack · Linear · Notion · Sentry · AWS · GCP · Cloudflare · Vercel · Railway · Render · Revyl(모바일 테스트)

## 14. React / PHP 로 만들 수 있나?

**React**: `apps/web` 이 이미 Vite + React 19 + TS + Tailwind. 그대로 사용 가능.
(디자인 규칙: `design.md` — monospace + 대문자 조합 금지)

**PHP**: 영역별로 난이도가 다름.

| 영역 | PHP 가능성 | 비고 |
|---|---|---|
| 웹 UI (Blade/Inertia) | 쉬움 | 단순 CRUD |
| REST API | 쉬움 | Laravel 이 더 빠를 수도 |
| Postgres 관리 | 쉬움 | Eloquent 마이그레이션 |
| ClickHouse 쿼리 | 가능 | `smi2/phpclickhouse` |
| 알럿 평가 / 크론 | 가능 | Scheduler + Queue |
| OTLP 수집 (protobuf/gRPC) | 어려움 | PHP gRPC 서버 약함, 고스루풋 부담 |
| MCP 서버 (SSE 스트리밍) | 어려움 | PHP-FPM 부적합 (Swoole/RoadRunner 필요) |
| 에이전트 워커 (장시간 세션) | 비추천 | 세션이 수시간~수일 유지되어야 함 |

### 추천 아키텍처
```
React (Vite)              <- 그대로 재사용
Laravel / PHP             <- 제품 API, 인증, 결제, 설정, 관리자
Node (또는 Go)             <- OTLP 수집 프록시 + 에이전트 워커 (그대로 재사용)
Postgres + ClickHouse
```
결론: **새로 만들지 말고 가져다 쓰고, PHP 레이어만 붙이는 것이 합리적.**

### 직접 만들 경우 MVP 범위 (2~3주)
```
포함: JSON 수집 엔드포인트 / ClickHouse 저장+조회 UI / 에러 지문화 / Issue 상태머신 /
      Anthropic API 1회 호출로 원인 요약 / Slack 웹훅
제외: OTLP protobuf·gRPC / 영속 세션+PR 자동생성 / MCP 서버 / 멀티테넌시+과금
```

## 15. 유튜브 강의 제작 가능성

가능. Apache 2.0 이므로 화면 설명·수정·광고 수익·유료 강의 판매 모두 허용.
단, **"Superlog" 상표는 사용 금지** — 서비스화할 때는 새 브랜드 필요.
영상 설명란에 원본 레포 링크와 Apache 2.0 표기 권장.

### 10부작 기획안
| 회차 | 제목 | 길이 |
|---|---|---|
| EP1 | 새벽 3시 장애를 AI가 대신 고쳐준다면? (데모 먼저) | 8분 |
| EP2 | 관측 5분 정리 — 로그/트레이스/메트릭 | 10분 |
| EP3 | 23만줄 오픈소스 해부 — 모노레포 구조 투어 | 15분 |
| EP4 | 왜 Postgres 와 ClickHouse 를 같이 쓰나 | 12분 |
| EP5 | 에러 수십만 건을 1개로 묶는 지문 기술 | 12분 |
| EP6 | AI 에이전트 툴 설계의 정석 — 종료형 vs 비종료형 | 18분 |
| EP7 | 에이전트가 PR을 남발하는 문제를 어떻게 고쳤나 | 15분 |
| EP8 | 프롬프트 인젝션 실전 방어 — 신뢰 경계 패턴 | 15분 |
| EP9 | MCP 서버 직접 만들기 (OAuth 포함) | 20분 |
| EP10 | AI 서비스로 돈 받기 — 토큰→크레딧→Stripe | 15분 |

### 제작 팁
1. EP1 은 결과부터 보여주기 (에러 → 슬랙 → PR 타임랩스)
2. `pnpm demo:seed:everything` 먼저 실행 (빈 화면 방지)
3. API 키 / GitHub App 개인키 / `.env` 전체 블러 필수
4. 에디터 폰트 16pt 이상 (모바일 시청자 다수)
5. 클라우드와 셀프호스팅 차이를 솔직히 안내 (조사 에이전트는 클라우드 전용)

---

## 16. 수익화 아이디어 (상세)

### 시장 배경
- 글로벌 Observability 시장 $50B+, 연 15% 성장
- Datadog/New Relic 비용 부담이 공통 고통점
- 한국: 영어 SaaS 진입장벽 + 데이터 국외이전 규제 + 온프레미스 선호
- "AI 에이전트 운영 자동화"는 초기 시장 → 선점 여지
- 무기: Apache 2.0 로 23만줄 프로덕션 코드 확보

### 아이디어 1 — 한국형 매니지드 호스팅
포크 + 한국어 UI + 국내 리전 + 국내 결제로 SaaS 판매.
타겟: 10~100인 국내 스타트업, Datadog 비용 부담 팀, 해외 SaaS 불가 조직.

| 플랜 | 월 가격 | 포함 |
|---|---|---|
| 스타터 | ₩49,000 | 로그 500만/월, 조사 30회 |
| 그로스 | ₩199,000 | 로그 5천만/월, 조사 150회 |
| 엔터프라이즈 | ₩990,000~ | 무제한 + 전용 리전 + SLA |
| 온프레미스 | ₩5,000,000/년 | 고객 서버 설치 + 지원 (마진 최고) |

할 일: i18n, 국내 클라우드 배포, 토스페이먼츠/이니시스 연동, 세금계산서, 브랜드 교체.
난이도 높음 / 초기투자 중 / 수익 잠재력 매우 높음 / 3~6개월.

### 아이디어 2 — "AI 온콜 대행" 서비스 (최우선 추천)
SaaS 가 아니라 서비스를 판매. "우리가 당신 서버를 감시하고 새벽 장애도 대응합니다."
타겟: 개발자 1~3명 소규모 팀, 1인 SaaS 운영자, 외주사 유지보수 계약 프로젝트.

```
베이직    월 ₩300,000   셋업 + 모니터링 + 주간 리포트 + 평일 대응
스탠다드  월 ₩800,000   + AI 조사 리포트 + 24/7 알림 대응 + PR 리뷰
프리미엄  월 ₩2,000,000 + 실제 버그 수정까지 (AI PR -> 사람 검증 -> 머지)
초기 구축비 1회 ₩1,500,000 ~ ₩3,000,000
```

장점: 추가 개발 거의 불필요 / 고객 5곳이면 월 400만 / AI 레버리지 / 리테이너 현금흐름 /
수요 확인 후 아이디어 1로 전환 가능.

| 리스크 | 대응 |
|---|---|
| 고객 코드 접근 | GitHub App 읽기+PR 생성만, push 권한 미수령 (플랫폼 설계와 일치) |
| 장애 책임 | 계약서에 "모니터링·권고 서비스" 명시 |
| 새벽 대응 부담 | AI 1차 분류 후 SEV-1 만 알림 |
| LLM 비용 | 월 조사 횟수 상한 + 초과 과금 |

난이도 낮음 / 초기투자 매우 낮음 / 2~4주 내 시작 가능.

### 아이디어 3 — 교육 상품
| 상품 | 가격 |
|---|---|
| 무료 유튜브 10부작 (리드 수집 깔때기) | ₩0 |
| 인플런 강의 | ₩99,000~₩249,000 |
| Udemy 영어 버전 | $89 |
| 전자책 "에이전트 툴 설계 패턴" | ₩29,000 |
| 1:1 멘토링 / 코드리뷰 | ₩150,000/시간 |
| 기업 출강 | ₩1,500,000/일 |

차별화: 시중 강의는 "LangChain 챗봇" 수준 → 이건 실제 YC 스타트업 프로덕션 코드 해부.

커리큘럼: (1) 에이전트 설계 원칙 (2) 안전한 에이전트 (3) 영속 세션 (4) 수익화 (5) 미니 Superlog 실습.

### 아이디어 4 — 버티컬 특화
| 버티컬 | 근거 | 추가 개발 |
|---|---|---|
| PHP/Laravel 생태계 | OTel 지원 약함, 경쟁 없음, 국내 레거시 다수 | Laravel 패키지 + PHP 스택트레이스 심볼화 |
| WordPress / 커머스 | 장애 = 즉시 매출 손실 | WP 플러그인 + 장바구니 에러 룰 |
| 공공/금융 온프레미스 | 해외 SaaS 금지 → 셀프호스팅이 유일 선택 | 폐쇄망 설치 + 보안 인증 문서 |
| 게임 서버 | 장애 민감도 최고 | 세션/매치메이킹 메트릭 |
| **AI 에이전트 운영 모니터링** | 신규 시장, 경쟁 거의 없음 | LLM 토큰/레이턴시/환각률 대시보드 (`ai-usage.ts` 재활용) |

### 아이디어 5 — 부가 상품 (저노력 고마진)
| 상품 | 가격 |
|---|---|
| 셀프호스팅 설치 스크립트 (Terraform/Helm) | ₩99,000 |
| Docker 원클릭 템플릿 (Railway/Render 배포 버튼) | ₩49,000 |
| 한국어 패치 팩 (i18n) | ₩199,000 |
| Slack 봇 커스터마이징 | ₩500,000/건 |
| 클라우드 비용 최적화 리포트 | ₩800,000/건 |
| Agent Skill 패키지 | ₩39,000 |

### 종합 비교
| 아이디어 | 난이도 | 초기비용 | 월 예상수익 | 시작까지 | 추천도 |
|---|---|---|---|---|---|
| 1. 매니지드 SaaS | 높음 | 높음 | 500만~5000만 | 3~6개월 | ★★★ |
| 2. AI 온콜 대행 | 낮음 | 매우 낮음 | 300만~2000만 | 2~4주 | ★★★★★ |
| 3. 교육 상품 | 낮음 | 매우 낮음 | 100만~1000만 | 1~2주 | ★★★★ |
| 4. 버티컬 특화 | 중간 | 중간 | 300만~3000만 | 2~3개월 | ★★★★ |
| 5. 부가 상품 | 매우 낮음 | 거의 0 | 50만~300만 | 1주 | ★★★ |

### 실행 로드맵
```
1~2주차 : 셀프호스팅 완전 정복 / 내 프로젝트 1개 실제 연결 / 유튜브 EP1 업로드
3~6주차 : "AI 온콜 대행" 랜딩페이지 + 가격표 / 지인 회사 2곳 무료 파일럿 / EP2~5
2~3개월 : 유료 고객 3~5곳 (월 300~800만) / 유료 강의 오픈 / 버티컬 반응 데이터 수집
4~6개월 : 검증된 버티컬로 SaaS 전환 (투자 유치 가능 구간)
```

### 법적 체크리스트
```
[o] Apache 2.0 전문 + NOTICE 유지
[o] 저작권 고지 유지 (삭제 시 라이선스 위반)
[o] 수정 사항 명시 (Apache 2.0 Section 4b)
[x] "Superlog" 상표 사용 금지 -> 새 브랜드 필수
[x] 원본 로고/디자인 자산 그대로 사용 금지
[o] 고객 데이터: 개인정보처리방침 + 위탁 계약
[o] "AI가 코드를 수정함" 고객 동의서
```

### 핵심 조언
Superlog 는 이미 23만줄 완성품이다. 부족한 것은 코드가 아니라 **돈 내고 쓸 고객**이다.
따라서 **아이디어 2(AI 온콜 대행)부터** 시작하는 것이 가장 현실적이다.
2주면 시작할 수 있고, 고객이 생기면 무엇을 만들어야 하는지 알게 된다.

---

## 17. 부록 — 최근 주요 커밋 (보안 강화 흐름)

```
b6f3223  Add an external-content prompt boundary (#516)
306d0c8  Encrypt stored integration credentials (#515)
80da90a  Fix concurrent OAuth code redemption (#514)
9a077eb  Secure integration install authorization (#513)
b1f1384  Enable shared auth rate limiting (#512)
52328ec  Remove legacy LLM proxy routes (#511)
cc1761a  Bound OTLP payload decompression (#510)
```

→ 최근 개발 방향이 **에이전트/연동 보안 강화**에 집중되어 있음. 셀프호스팅 시
`AGENT_SECRETS_KEY`, `STATE_SIGNING_SECRET`, 인증 레이트리밋 설정을 반드시 챙길 것.

## 18. 로드맵 (ROADMAP.md 요약)

| 단계 | 항목 |
|---|---|
| Now | AWS 리소스 대시보드 / 서비스 맵 / 이상 탐지(선제적 인시던트 개설) |
| Next | 에이전트 제안 인프라 수정(승인형) / 속성별 ingest 규칙 / SDK 확대 |
| Later | 프로젝트별 비용 통제 / 이슈트래커·인시던트·채팅 연동 확대 |
