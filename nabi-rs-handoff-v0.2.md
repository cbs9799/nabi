# 나비-rs 핸드오프 문서

**버전**: 0.3.2 (invocation_id 전파 메커니즘 명시 — advisor 3차 검토)
**이전 버전**: 0.3.1 → 0.3 (2026-05-16) → 0.2 (2026-05-15)
**작성일**: 2026-05-16
**대상**: 미래의 밤식 + Claude Code CLI (구현 위탁용)
**상태**: 설계 확정, Phase -1 진입 대기

---

## 0. 이 문서 사용법

- Phase 단위 작업 위임용. Claude Code CLI에 phase 단위로 prompt 후 commit
- 결정사항 변경 시 § 표시한 섹션 업데이트, version bump
- Open Decisions(§ 10) 해소 시 결정 사유 inline 기록
- 모든 phase는 자체 검증 기준 보유. PR 머지 전 통과 필수
- **0.1 → 0.2 → 0.3 변경사항은 § 14 변경 이력 참조**

### 0.1 목차 (v0.3.1 — 3700+ 라인 운용성)

- § 1 프로젝트 개요 / § 2 배경 & 제약 / § 3 핵심 결정사항
- § 4 아키텍처 (4.1 다이어그램 / 4.2 워크스페이스 / 4.3 데이터 흐름 / 4.4 권한 / 4.5 인증 / 4.6 caveman)
- § 5 기술 스택 / § 6 구현 로드맵 (Phase -1 ~ Phase 12)
- § 7 코드 스켈레톤 (7.1 Provider / 7.2 Types / 7.3 ClaudeCliProvider / 7.4 Permission MCP / 7.5 claude-settings / 7.6 nabi.yaml / 7.7 Routing 문법)
- § 8 운영 / § 9 보안 / § 10 Open Decisions
- § 11 리스크 / § 12 참고자료 / § 13 초기 셋업 / § 14 변경 이력
- **§ 15 nabi.db 전체 스키마** ← Phase 3 시작 시
- **§ 16 NabiError 타입** ← Phase 1 시작 시
- **§ 17 WebSocket 프로토콜** ← Phase 7 시작 시
- **§ 18 Skill / Wiki / Daily Log 포맷** ← Phase 3 시작 시
- **§ 19 Migration 시스템** ← Phase 3 시작 시
- **§ 20 CI Workflow** ← Phase 0 시작 시
- **§ 21 Phase Acceptance Matrix** ← 매 Phase 머지 직전
- **§ 22 Per-Crate Cargo.toml** ← Phase 0 시작 시
- § 23 결정 트리 / § 24 코딩 규칙

#### 위탁 시 cross-ref 가이드

- Phase 0 위탁: § 6 Phase 0 + § 13.2 + § 22 + § 20 + § 21 Phase 0
- Phase 1 위탁: § 7.1 + § 7.2 + § 16 + § 22.1 + § 21 Phase 1
- Phase 2a 위탁: § 4.3 + § 7.3 + § 22.2 + § 21 Phase 2a
- Phase 2c 위탁: § 4.4 + § 7.4 + § 17 + § 21 Phase 2c
- Phase 3 위탁: § 15 + § 18 + § 19 + § 22.3 + § 21 Phase 3
- Phase 4 위탁: § 4.6 + § 7.6 prompting + § 22.5 + § 21 Phase 4
- Phase 7 위탁: § 17 + § 22.6 + § 8.1 + § 21 Phase 7

---

## 1. 프로젝트 개요

### 1.1 비전

OpenClaw 기반 나비(현재 OpenClaw) 운영 한계 극복하는 개인용 AI agent harness. Rust 단일 바이너리 + Claude 구독 최대 활용 + 멀티 디바이스 접근. 6개월 내 일상 driver 전환.

### 1.2 목표 (in scope)

- Claude Max 구독을 정식 경로(Agent SDK 크레딧, 6/15+)로 활용
- 멀티 프로바이더(Claude / OpenRouter / 로컬 Ollama) 추상화
- 나비 기존 자산(memory.db, wiki) 100% 호환
- TUI(로컬+원격) + PWA(모바일/회사) + Telegram bot 멀티 인터페이스
- 어떤 디바이스에서도 동일 세션 이어서 사용
- Cloudflare Zero Trust 기반 인증, 비밀번호 관리 0
- B2G/텐드릴/Tistory 등 도메인 specialist 패턴 지원
- **nabi-rs는 Claude CLI runtime의 supervisor — 제어자 아니라 안전한 설정으로 감싸고 stream/permission을 중계**

### 1.3 비목표 (out of scope)

- 일반 사용자 배포 (밤식 1인 사용 전제)
- Windows 네이티브 GUI
- 학습/파인튜닝 기능
- 마켓플레이스 / 플러그인 생태계
- Anthropic 정식 협업 / Claude Code 대체

### 1.4 성공 지표 (재설정 — 0.2)

Phase 12 완료 시점:

- 집 / 회사 / 폰 / 텔레그램 4개 채널 모두 동일 세션 접근 가능
- PM2 24/7 가동, uptime 99% 이상 (MacBook sleep disable 전제)
- 월 비용 Max 20x 구독 $200 한도 내 (extra usage cap 적용)
- **nabi-server RSS < 100MB** (Claude subprocess 제외)
- **end-to-first-stream-event P50 < 2s, P95 < 5s** (Claude CLI subscription mode)
- **mock / OpenRouter / Ollama: P50 < 1s**
- 메모리 사용량은 Phase 2a 실측 후 재조정

---

## 2. 배경 & 제약

### 2.1 외부 환경 타임라인

- **2026-01-09**: Anthropic이 3rd-party harness에 대한 OAuth 토큰 차단 시작
- **2026-02-20**: Anthropic ToS 갱신, OAuth는 Claude Code / claude.ai 전용 명시
- **2026-04-04 12:00 PT**: Pro/Max 구독의 3rd-party 접근 완전 차단 시행. OpenClaw 등 우회 사망
- **2026-05-13**: Anthropic이 Agent SDK 별도 크레딧 발표
- **2026-06-15**: Agent SDK 크레딧 시행 — Pro $20 / Max 5x $100 / Max 20x $200 월간, API list rate 차감
- **현재 (2026-05-15)**: 6/15까지 한 달. 그 사이 직접 API key로 MVP 개발 → 6/15 이후 subprocess 래핑으로 구독 활용

### 2.2 합법 경로 = subprocess 래핑 only

- 직접 Messages API 호출 → API key 필수, 구독 활용 불가
- `claude -p --output-format stream-json` subprocess 호출 → Agent SDK 크레딧 경로
- 구독 활용 시 Claude CLI 바이너리가 인증 / prompt cache / MCP / tool / session 전부 관리
- nabi-rs는 그 위에 layer를 얹는 supervisor 구조

### 2.3 ★ `--bare` 정책 (v0.2 신규)

`--bare`는 OAuth와 keychain reads를 건너뜀. 공식 headless 문서 명시. 구독 OAuth와 충돌.

- **subscription mode**: `--bare` 절대 금지. OAuth 사용
- **api_key mode**: `--bare` 허용. `ANTHROPIC_API_KEY` 환경변수 또는 `--settings` JSON의 apiKeyHelper 사용

Provider 설정에서 명시적으로 모드 분리. § 3.2 참조.

### 2.4 기존 자산 인벤토리 (v0.3 — 실측 정정)

- **memory.db** (v0.3 경로 정정): `~/clawd/memory-db/memory-sync.db` (~700KB, 813 엔트리, sqlite-vec + FTS5). v0.2의 `~/nabi/memory.db` 표기는 오류 — 해당 디렉토리 부재. nabi-rs는 Phase 3에서 **read-only로만 접근**. write는 Phase 4 이후 shadow table
- **wiki**: `~/clawd/` 하위 (MEMORY.md / SOUL.md / PLAYBOOK.md / USER.md / IDENTITY.md / TOOLS.md). Karpathy 패턴
- **기존 서비스 = claude-telegram-bridge** (v0.3 신규 인벤토리):
  - 위치: `~/Projects/claude-telegram-bridge`
  - 스택: Python 3 + FastAPI + uvicorn + SSE
  - 도메인: `nabi.ix-works.xyz`
  - bind: `127.0.0.1:8787` → **nabi-rs 9912와 충돌 없음 확인**
  - DB: `app.db` (FTS5, 메타데이터만) + `~/clawd/memory-db/memory-sync.db` (재사용)
  - PM2 4개: `claude-telegram` (레거시 봇), `cloudflared-tunnel`, `nabi-web` (8787), `telegram-poller`
  - 인증: Cloudflare Access (Google)
  - Cloudflare Tunnel: `bamsigi_tunnel` (uuid `cd1c1086-652b-4aa1-a369-06fbf7da1242`)
  - 상태: **아카이빙 예정** (§ 10.8 결정 완료)
  - Phase 8c cutover 후 → `archived/claude-telegram-bridge` 로 이전, PM2 stop
- **OpenClaw 나비**: 별도, 현재 운영 중. nabi-rs와 병행 가동 후 점진 마이그레이션
- **ix-works.xyz**: Cloudflare Tunnel, PM2 14 서비스 (claude-telegram-bridge 4개 + 기타)
- **Confluence HI space**: cloudId `2d9b452c-4288-482b-8d70-bf379962129f`, spaceId `847937539`, 173 페이지 인덱스
- **Windows server**: 192.168.0.148 / win.ix-works.xyz, PM2 native (WSL 금지) — **backup으로 격하 (Phase 12)**
- **하드웨어**: home MacBook M4 (★ primary 확정), Mac Studio M4 Max 64GB (LLM 추론용), Windows server (backup)

### 2.5 운영 규칙 (절대 위반 금지)

- PM2 kill 절대 금지 (cloudflare-tunnel 함께 죽음, SSH 끊김)
- ix-data.db: 단일 위치 (ix-works dir), copy/junction 금지
- Windows server: PM2는 PowerShell native, WSL 내부 절대 X
- OpenClaw.app LaunchAgent plist 비활성 유지 (secrets REDACTED 덮어쓰기)
- SSH 복잡 명령: 스크립트 파일로, inline 금지
- 마크다운 테이블 금지, bullet list만
- Wiki 파일: append only, overwrite 금지
- 모든 작업: in-session 완료, 지연 금지
- **memory.db: Phase 3까지 read-only. Phase 4부터 shadow table만 write** (신규)
- **`--bare` 사용은 api_key mode에서만** (신규)

---

## 3. 핵심 결정사항

각 결정은 trade-off 명시 + 채택 사유 기록.

### 3.1 언어: Rust

- **대안**: TypeScript (Claude Code 본가), Python (Agent SDK), Go (단순)
- **채택 사유**:
  - 단일 바이너리 배포, 다중 플랫폼 (Mac/Linux/Windows)
  - subprocess + async I/O 강점 (tokio)
  - 장기 daemon 안정성 (메모리 안전, GC pause 없음)
  - 100MB 미만 메모리 풋프린트 (서버 프로세스 한정)
  - 학습 가치 (밤식 향후 독립 사업 차원)
- **비용**: 개발 속도는 TS/Python 대비 느림. crate 생태계 일부 미성숙

### 3.2 LLM 인증 경로 — Provider 모드 분리 (v0.2 개정)

Claude CLI provider를 2개 모드로 분리. 같은 `ClaudeCliProvider` 타입에 mode enum.

#### claude_cli_subscription

- **6/15 이후 기본 경로**
- `--bare` 금지
- Anthropic 인증: `~/.claude/config.json` OAuth 토큰
- prompt cache / MCP auto-discovery / CLAUDE.md / hooks / skills 모두 활용
- 크레딧 차감: Agent SDK 월 크레딧 풀

#### claude_cli_api_key

- **5/15~6/14 개발 / CI / 격리 테스트 용도**
- `--bare` 허용
- 인증: `ANTHROPIC_API_KEY` 또는 `--settings` apiKeyHelper
- auto-discovery 차단 → 재현성
- 과금: 직접 API key 청구

**왜 분리해야 하나**:
- 공식 headless 문서: bare mode는 OAuth/keychain 건너뜀
- subscription에서 `--bare` 붙이면 "Not logged in" 에러
- 한 provider에 두 모드 합치면 Phase 2 첫날 깨짐

### 3.3 멀티모델 전략: 1+1

- **Claude**: CLI subprocess (구독 활용)
- **그 외**: 단일 OpenAI-compat HTTP provider (OpenRouter, Ollama, GLM, Qwen 등 전부 커버)
- **사유**: Smelt처럼 5종 type 직접 구현 = 오버엔지니어링. base_url만 바꿔도 95% 호환
- **라우팅**: 작업 복잡도 기반. 메인 = Claude, 잡일 = OpenRouter GLM, 프라이빗 = Ollama
- **유의 (v0.2)**: provider별 streaming/tool/cost dialect 차이 존재 → Phase 6에서 실측 후 trait 보강 검토

### 3.4 토폴로지: strict server-client 분리

- **결정**: server daemon + thin clients (단일 서버 = 집 MacBook M4)
- **사유**:
  - Claude CLI는 한 머신에 묶임 (computer use 권한)
  - 세션 conflict 방지 = 단일 소스
  - 모바일은 thin client만 가능
  - 디바이스 끊겨도 agent 진행 가능
- **Server primary 확정 (v0.2)**: home MacBook M4
  - 단 sleep disable / 자동 재시작 / 자동 업데이트 차단 필수 (§ 8.5)
  - Windows = backup, Phase 12에서만 검토

### 3.5 인증: Cloudflare Zero Trust Access

- **대안**: 자체 JWT 발급, mTLS, basic auth, OAuth 직접 구현
- **채택**: CF Access (Google IdP)
- **사유**:
  - 기존 Cloudflare Tunnel 활용
  - 비밀번호 관리 0
  - 폰/노트북/회사컴 SSO 한 번
  - 의심 패턴 자동 차단
- **비용**: Cloudflare 의존, 무료 플랜 한도 (50명 미만 OK)
- **TUI는 별도 (v0.2)**: 브라우저 쿠키 없으므로 `cloudflared access login` + token을 OS Keychain에 저장 + WS upgrade 시 `cf-access-token` 헤더

### 3.6 라이선스: Private/Proprietary (v0.2 확정)

- 구독 인증 / 개인 운영 / 시크릿 / 도메인 지식 혼재
- 공개 배포 시 Anthropic ToS 추가 검토 필요
- Provider trait 등 일반화 가능한 부분은 추후 분리 가능

### 3.7 데이터베이스: SQLite + single-writer

- **대안**: PostgreSQL (멀티 클라이언트 강함), DuckDB (분석)
- **채택**: SQLite (`rusqlite` bundled)
- **사유**:
  - 기존 memory.db 호환
  - 단일 파일, 백업 단순 (Litestream)
  - 1인 사용 동시성 부담 없음
  - FTS5 내장
- **Write 전략 (v0.2 보강)**:
  - WAL mode + `busy_timeout=5000`
  - 모든 write는 mpsc queue → single writer actor
  - r2d2 pool은 read 전용
  - memory.db = read-only (OpenClaw 병행 운영 중 충돌 방지)
  - 짧은 transaction만, long-running 금지

### 3.8 nabi-rs 역할 재정의 (v0.2 신규)

Claude CLI를 subprocess로 쓰면 다음은 **Claude CLI 내부**가 처리:

- built-in tool execution (Read / Edit / Bash / Grep / Glob / WebSearch / WebFetch)
- prompt cache
- MCP server lifecycle
- session persistence (`~/.claude/projects/`)
- permission mode evaluation
- local settings discovery (subscription mode)
- CLAUDE.md / skills / hooks discovery (non-bare)

nabi-rs는 다음만 담당:

- **설정 생성자**: `--settings`, `--mcp-config`, `--append-system-prompt-file`, allow/deny 규칙
- **stream 파서**: stream-json NDJSON → 통일 이벤트 (raw audit 포함)
- **process supervisor**: spawn / kill / health / restart / SIGTERM
- **context injector**: AGENTS.md + wiki + memory.db FTS + SKILL.md 동적 주입
- **permission bridge**: `--permission-prompt-tool` MCP 서버로 원격 디바이스 approval 중계
- **fan-out**: WebSocket으로 N개 디바이스에 스트림 분배
- **외부 auth**: CF Access JWT 검증
- **세션 메타데이터**: nabi.db에 디바이스/비용/audit log

이 framing이 § 4, § 7, § 9 전체에 일관 적용.

---

## 4. 아키텍처

### 4.1 시스템 다이어그램

```
                       ┌────────────────────────────────┐
                       │  nabi-server (daemon)          │
                       │  primary: home MacBook M4      │
                       │  bind: 127.0.0.1:9912          │
                       │                                │
                       │  ┌──────────────────────────┐  │
                       │  │ axum HTTP + WebSocket    │  │
                       │  └────────────┬─────────────┘  │
                       │               │                │
                       │  ┌────────────┴─────────────┐  │
                       │  │ Supervisor               │  │
                       │  │  ├ ContextBuilder        │  │
                       │  │  ├ Provider Manager      │  │
                       │  │  ├ Permission MCP Server │  │
                       │  │  └ Broadcast (fan-out)   │  │
                       │  └────────────┬─────────────┘  │
                       │               │                │
                       │  ┌────────────┴─────────────┐  │
                       │  │ Persistence              │  │
                       │  │  ├ nabi.db (write)       │  │
                       │  │  ├ memory.db (read-only) │  │
                       │  │  └ wiki/ (read-only md)  │  │
                       │  └────────────┬─────────────┘  │
                       │               │                │
                       │  ┌────────────┴─────────────┐  │
                       │  │ External                 │  │
                       │  │  ├ claude CLI subprocess │  │
                       │  │  ├ OpenRouter HTTP       │  │
                       │  │  └ Ollama HTTP (Studio)  │  │
                       │  └──────────────────────────┘  │
                       └──────────────┬─────────────────┘
                                      │
                              Cloudflare Tunnel
                                      │
                              nabi.ix-works.xyz
                                      │
                          Cloudflare Zero Trust Access
                                      │
              ┌─────────┬─────────────┼───────────────┬──────────┐
              ▼         ▼             ▼               ▼          ▼
         [집 TUI] [회사 TUI]   [PWA 브라우저]   [iOS 홈화면] [Telegram]
                                                              (overflow)
```

### 4.2 워크스페이스 레이아웃

```
nabi-rs/
├── Cargo.toml                    # workspace
├── Cargo.lock
├── README.md
├── CLAUDE.md                     # Claude Code용 instructions
├── AGENTS.md                     # nabi-rs 자체 agent instructions
├── docs/
│   └── handoff.md                # 이 문서
├── crates/
│   ├── nabi-core/                # 공유 도메인
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── types.rs          # Message, Session, StreamEvent, RawEvent
│   │   │   ├── provider.rs       # Provider trait
│   │   │   ├── agent.rs          # 메인 agent loop
│   │   │   └── error.rs
│   │   └── Cargo.toml
│   ├── nabi-providers/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── claude_cli.rs     # ★ 핵심 — subprocess 래퍼 (sub/api 모드 분리)
│   │   │   ├── openai_compat.rs  # OpenRouter, Ollama 등
│   │   │   └── routing.rs        # 작업별 provider 선택
│   │   └── Cargo.toml
│   ├── nabi-memory/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── store.rs          # memory.db (FTS5, read-only)
│   │   │   ├── shadow.rs         # nabi.db memory_events (write-back)
│   │   │   ├── wiki.rs           # 마크다운 wiki loader
│   │   │   ├── sessions.rs       # nabi.db (세션 DB)
│   │   │   └── writer.rs         # single-writer actor
│   │   └── Cargo.toml
│   ├── nabi-skills/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   └── loader.rs         # SKILL.md 동적 로드
│   │   └── Cargo.toml
│   ├── nabi-context/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── builder.rs        # 시스템 프롬프트 동적 구성
│   │   │   ├── budget.rs         # 토큰 예산 관리
│   │   │   └── manifest.rs       # 매 요청 context manifest DB 저장
│   │   └── Cargo.toml
│   ├── nabi-server/              # ★ daemon
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── http.rs           # axum REST
│   │   │   ├── ws.rs             # WebSocket
│   │   │   ├── auth.rs           # CF JWT 검증, JWKS rotation
│   │   │   ├── session.rs        # 세션 lifecycle
│   │   │   ├── broadcast.rs      # multi-device fan-out
│   │   │   ├── permission_mcp.rs # ★ --permission-prompt-tool MCP 서버
│   │   │   ├── telegram.rs       # Telegram ingress
│   │   │   ├── shutdown.rs       # graceful shutdown (SIGTERM)
│   │   │   └── secrets.rs        # macOS Keychain 통합
│   │   └── Cargo.toml
│   ├── nabi-tui/
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── app.rs            # ratatui App
│   │   │   ├── widgets/
│   │   │   ├── transport.rs      # local vs remote 모드
│   │   │   └── cf_auth.rs        # cloudflared access login 연동
│   │   └── Cargo.toml
│   └── nabi-web/                 # PWA 정적 + WS 클라이언트
│       ├── src/
│       │   └── main.rs           # axum static server
│       ├── static/
│       │   ├── index.html
│       │   ├── app.js
│       │   ├── style.css
│       │   ├── manifest.json     # PWA 매니페스트
│       │   ├── service-worker.js # cache versioning + skipWaiting
│       │   └── icons/
│       └── Cargo.toml
├── config/
│   ├── nabi.yaml                 # 설정 (provider, paths)
│   ├── AGENTS.md                 # 글로벌 instructions
│   ├── claude-settings.json      # ★ permissions.deny + sandbox
│   ├── prompting/
│   │   └── caveman.md            # ★ 기본 출력 스타일 (§ 4.6, v0.3)
│   └── skills/
│       ├── tendril.md
│       ├── tistory.md
│       ├── pm2.md
│       └── b2g.md
├── secrets/                      # gitignore, chmod 700
│   └── (Keychain 우선, 파일은 fallback)
├── scripts/
│   ├── install.sh
│   ├── migrate-existing.sh       # ★ Phase -1 자동화
│   ├── deploy-mac.sh
│   ├── deploy-windows.ps1
│   ├── pm2.config.cjs
│   └── pmset-policy.sh           # ★ MacBook sleep 정책
└── tests/
    ├── integration/
    └── fixtures/
        └── stream-json/          # ★ Phase 2a 수집 fixture
            ├── plain.ndjson
            ├── tool-use.ndjson
            ├── permission-abort.ndjson
            ├── api-retry.ndjson
            ├── mcp-failure.ndjson
            └── session-resume.ndjson
```

### 4.3 데이터 흐름 — Claude CLI invocation (v0.2 개정)

실제 실행되는 CLI 명령:

```bash
claude -p "$PROMPT" \
    --output-format stream-json \
    --verbose \
    --include-partial-messages \
    --resume "$SESSION_ID" \
    --append-system-prompt-file "$SYSTEM_PROMPT_PATH" \
    --tools "Read,Grep,Glob,Edit,Bash,WebSearch" \
    --disallowedTools "Bash(rm:*),Bash(sudo:*),Bash(curl:*),WebFetch" \
    --permission-prompt-tool "mcp__nabi_auth__approve" \
    --permission-mode default \
    --strict-mcp-config \
    --mcp-config "$MCP_CONFIG_PATH" \
    --max-budget-usd 5.00 \
    --max-turns 30 \
    --settings "$CLAUDE_SETTINGS_PATH" \
    --model sonnet
```

**플래그 분리 의미 (v0.2)**:

- `--tools`: 모델 컨텍스트에 노출할 도구 화이트리스트
- `--disallowedTools`: 컨텍스트에서 제거 + bash 패턴 차단 (e.g. `Bash(rm:*)`)
- `--allowedTools`: **자동 승인** 리스트 (prompt 없이 통과). 위 명령에 없음 — 모든 도구는 permission-prompt-tool 거침
- `--permission-prompt-tool`: nabi-server가 띄운 MCP 서버 `mcp__nabi_auth__approve` 호출. 응답 포맷 `{"behavior":"allow","updatedInput":{...}}` 또는 `{"behavior":"deny","message":"..."}`
- `--settings`: `claude-settings.json` 경로. `permissions.deny`로 시크릿 경로 차단, `sandbox`로 fs/net 제한
- `--max-budget-usd 5.00`: 매 invocation 강제 cap. 최소 ~$0.05 (시스템 캐시 비용)
- `--max-turns 30`: agent loop 무한 방지
- `--strict-mcp-config`: project/global MCP 안 섞이게
- `--verbose`: partial token stream 필수

**메시지 처리 흐름**:

```
[클라이언트] WS send: {"type":"chat","sid":"x","content":"..."}
        │
        ▼
[nabi-server::ws] 메시지 수신
        │
        ▼
[nabi-server::session] 세션 찾기 / 생성, claude_session_id 매핑
        │
        ▼
[nabi-context::builder] 시스템 프롬프트 동적 구성 (토큰 예산 적용)
  ├─ AGENTS.md (글로벌 + 워크스페이스)
  ├─ MEMORY.md hub 요약
  ├─ 관련 SKILL.md top-2
  ├─ memory.db FTS top-k (토큰 한도 내)
  └─ 오늘/어제 daily log
        │
        ▼ 파일로 저장 후 --append-system-prompt-file
[nabi-context::manifest] context_manifest DB 저장 (어떤 자료 들어갔는지)
        │
        ▼
[nabi-providers::claude_cli] subprocess spawn (모드 분기)
  - subscription: --bare 없이, OAuth 사용
  - api_key: --bare 가능, ANTHROPIC_API_KEY 사용
        │
        ▼
[Claude CLI] OAuth (subscription) 또는 API key (api_key)
        │
        ▼
[Anthropic API] 응답 시작
        │
        ▼
[stdout NDJSON 라인별]
  {"type":"system","subtype":"init","session_id":"abc",...}
  {"type":"stream_event","event":{"type":"content_block_delta","delta":{"type":"text_delta","text":"..."}}}
  {"type":"system","subtype":"api_retry",...}
  {"type":"stream_event","event":{"type":"content_block_start","content_block":{"type":"tool_use",...}}}
  ... (tool execution by Claude CLI 자동, permission-prompt-tool 거침)
  {"type":"result","total_cost_usd":0.043,...}
        │
        ▼
[nabi-providers::claude_cli] NDJSON 파싱 → RawEvent + StreamEvent 변환
  - RawEvent: audit log에 무삭제 저장
  - StreamEvent: nabi 통일 이벤트 (TextDelta, ToolUseStart, etc)
  - 알 수 없는 타입은 RawEvent만 저장
        │
        ▼
[nabi-server::broadcast] 모든 구독 디바이스에 fan-out
        │
        ▼
[클라이언트 N개] WS 수신 → 화면 렌더
        │
        ▼
[nabi-memory::writer] single-writer queue로 DB append
```

**stderr 처리**: `Stdio::piped()`로 열되 별도 task에서 drain하여 trace::warn에 기록. 안 하면 child block.

### 4.4 권한 요청 흐름 (v0.2 재설계)

핵심: Claude CLI는 `--permission-prompt-tool`로 지정된 MCP tool을 호출하여 권한 결정을 받음. nabi-server가 그 MCP tool을 자체 호스팅.

```
[Claude CLI] 위험 도구 사용 시 mcp__nabi_auth__approve 호출
        │ MCP JSON-RPC over stdio
        ▼
[nabi-server::permission_mcp] approval_prompt tool 수신
  - tool_name: "Bash" / input: {"command":"rm -rf ..."}
  - tool_use_id: 추적용
        │
        ▼
[nabi-server] permission_request 생성, DB pending 저장
        │
        ├─ broadcast → 모든 디바이스
        ├─ push notification (PWA Web Push, Phase 10)
        └─ Telegram inline keyboard (Phase 11)
        │
        ▼ (사용자가 어느 디바이스에서든 결정)
[클라이언트] WS send: {"type":"permission_response","request_id":"r1","allow":true}
        │
        ▼
[nabi-server] 첫 응답 wins
  - DB 업데이트 (decided_by_device, decided_at)
  - broadcast → 나머지 디바이스 "이미 결정됨"
        │
        ▼
[nabi-server::permission_mcp] MCP 응답 반환
  allow: {"behavior":"allow","updatedInput":<원본 또는 수정>}
  deny:  {"behavior":"deny","message":"...사유..."}
        │
        ▼
[Claude CLI] 도구 실행 또는 abort
```

**MCP 서버 구현**: `nabi-server::permission_mcp`가 stdio MCP 서버 노출. `claude` 실행 시 `--mcp-config`의 `mcpServers.nabi_auth.command`에 nabi-server 자체 바이너리를 `--mcp-permission-server` 모드로 호출.

### 4.5 인증 흐름 — 두 갈래 (v0.2)

#### 브라우저 / PWA

```
[브라우저/PWA] → nabi.ix-works.xyz/
        │
        ▼
[Cloudflare Edge] CF Access Policy 평가
        │
        ▼ (인증 안 됨)
[Google OAuth] 리디렉션 → 로그인
        │
        ▼ (성공)
[Cloudflare] CF_Authorization 쿠키 set + JWT 발급
        │
        ▼ (이후 요청)
[Cloudflare] 헤더 Cf-Access-Jwt-Assertion 자동 주입
        │
        ▼
[nabi-server::auth] JWT 검증
  ├─ CF JWKS 공개키로 서명 확인 (kid 기준, 6주 + 7일 rotation 캐시)
  ├─ aud claim 검증 (앱별 고유)
  ├─ email claim 추출
  └─ Allow list 체크
```

#### TUI (Terminal)

```
[TUI 첫 실행] nabi-tui auth login https://nabi.ix-works.xyz
        │
        ▼
[내부 호출] cloudflared access login <app>
        │
        ▼
[브라우저 자동 오픈] Google OAuth
        │
        ▼
[cloudflared] 토큰 수신
        │
        ▼
[nabi-tui] cloudflared access token --app <app> 으로 토큰 추출
        │
        ▼
[macOS Keychain] nabi-tui-cf-token 으로 저장
        │
        ▼ (이후 WS 연결)
[nabi-tui] WS upgrade 헤더에 cf-access-token: <token>
        │
        ▼
[Cloudflare Edge] 검증 후 origin 전달
        │
        ▼
[nabi-server] JWT 검증 (브라우저와 동일 흐름)
```

토큰 만료 시 `nabi-tui auth login` 재실행. service token은 healthcheck/CI 전용으로만 사용.

### 4.6 기본 출력 스타일 — caveman mode (v0.3 신규)

**원리**: 출력 토큰만 줄이고 thinking/reasoning은 보존. caveman GitHub 플러그인 방식.

- 관사·필러·헤지·인사·질문 되풀이 제거
- 짧은 동사·명사 단편 우선
- 코드 / 경로 / 명령 / 식별자는 verbatim 보존
- 참고: "Brevity Constraints Reverse Performance Hierarchies" (2026-03) — 간결성 제약이 특정 벤치마크에서 정확도 +26pp

#### 적용 구조

- canonical snippet: `prompting/caveman.md` (Phase 0 후 `config/prompting/caveman.md`)
- context builder의 **layer 0** (AGENTS.md보다 먼저, 토큰 예산 ~250)
- 매 invocation `--append-system-prompt-file` 또는 system prompt 첫 블록으로 주입

#### 인터페이스별 정책 (nabi.yaml `prompting:` 블록에서 제어)

- **TUI**: caveman strict (기본)
- **PWA**: caveman default, UI 토글 제공
- **Telegram**: relaxed (async + 화면 context 부재로 단편 위험)
- **permission-prompt-tool 응답 / 에러 / 온보딩**: 항상 relaxed — 명확성 우선

#### 턴 단위 escape hatch

context builder가 사용자 입력에서 다음 trigger 감지 시 그 턴만 caveman 제외:

- `자세히`, `상세히`, `설명해`, `풀어서`
- `verbose`, `explain`, `walk me through`, `in detail`
- 명시적 README / post-mortem / design doc 요청

#### 트레이드오프 (v0.3 인지)

- 한국어는 영어보다 토큰 효율 낮음. 단편화가 무례하게 느껴질 수 있음 — 1인 사용 전제로 수용
- 권한 요청·에러는 절대 caveman 적용 X (사용자가 결정 못 할 수 있음)
- 첫 1주 운영 후 Telegram 정책 (relaxed → caveman) 재검토

#### Phase 매핑

- Phase 4 Context Builder가 layer 0으로 caveman 주입 구현
- Phase 5 TUI / Phase 9 PWA / Phase 11 Telegram에서 각 인터페이스 정책 적용
- Phase 2c permission MCP 응답은 caveman 우회

---

## 5. 기술 스택

### 5.1 Core 의존성

- **tokio** 1.x — async runtime. features: full
- **serde** + **serde_json** — 직렬화
- **async-trait** — trait async
- **anyhow** + **thiserror** — 에러 처리
- **tracing** + **tracing-subscriber** + **tracing-appender** — 구조화 로깅 + 파일 로테이션
- **clap** 4 — CLI args (derive macro)
- **toml** + **serde_yaml** — 설정 파싱
- **futures** — Stream 유틸
- **async-stream** — Stream macro

### 5.2 Subprocess

- **tokio::process::Command** — 비동기 subprocess
- **tokio::io::BufReader** + **AsyncBufReadExt** — stdout/stderr 라인 파싱

### 5.3 HTTP

- **reqwest** 0.12 — outbound HTTP. features: json, stream, rustls-tls
- **eventsource-stream** — SSE 파싱 (OpenAI compat)
- **axum** 0.7 — 서버. features: ws, macros
- **tower** + **tower-http** — 미들웨어, CORS, trace, body limit
- **tokio-tungstenite** — WebSocket (axum 내장 활용)

### 5.4 인증 & 시크릿

- **jsonwebtoken** — CF JWT 검증
- **reqwest** — JWKS fetch + 캐시 (TTL + kid 기반 invalidation)
- **keyring** 3 — macOS Keychain (features: apple-native)

### 5.5 데이터

- **rusqlite** 0.32 — SQLite, features: bundled, json, blob
- **r2d2** + **r2d2_sqlite** — read 전용 커넥션 풀
- **chrono** — 시간 처리

### 5.6 TUI

- **ratatui** 0.29 — 위젯
- **crossterm** 0.28 — 터미널 백엔드

### 5.7 Telegram

- **teloxide** — Telegram bot 프레임워크

### 5.8 MCP (Permission Server)

- **rmcp** 또는 **mcp-sdk** (Rust 공식 SDK)
- 또는 자체 stdio JSON-RPC 2.0 핸들러 (단순함)

### 5.9 운영

- **PM2** — 프로세스 매니저 (기존 인프라 호환)
- **Cloudflare Tunnel (cloudflared)** — ingress (이미 운영 중, system daemon)
- **Litestream** — SQLite S3/R2 백업

---

## 6. 구현 로드맵

각 phase = 독립 commit-able 단위. 검증 기준 통과 후 다음 phase 진행.

### Phase -1 — 사전 작업 및 기존 서비스 dump (v0.2 신설)

- **목적**: 기존 nabi.ix-works.xyz 서비스 안전하게 인벤토리 + 백업 + MacBook 운영 정책 적용
- **소요**: 1일
- **결과물**:
  - 기존 서비스 인벤토리: PM2 id / 프로세스명 / 사용 포트 / 사용 DB 경로 / 코드 위치
  - 백업 tarball: `~/backups/nabi-migration-20260515/`
    - `~/.claude/` 전체
    - `~/nabi/` 전체
    - `memory.db .backup` + `ix-data.db .backup`
    - PM2 `ecosystem.config.cjs`
    - `~/.cloudflared/config.yml`
  - 시스템 정책 적용:
    - `sudo pmset -a sleep 0 disksleep 0 powernap 0 autorestart 1`
    - `sudo pmset -c womp 1`
    - `sudo softwareupdate --schedule off`
    - Time Machine 제외: `tmutil addexclusion` for `memory.db*`, `nabi.db*`, `~/nabi-rs/target/`
    - Spotlight 제외: `touch ~/nabi/.metadata_never_index ~/nabi-rs/.metadata_never_index`
    - macOS 자동 로그인 활성 결정 (Open Decision § 10.10)
  - Claude CLI 인증 확인: `claude /status` → 어느 계정 / 모델 사용 가능 / 토큰 만료
  - 신규 포트 9912 충돌 확인: `lsof -i :9912`
  - 디스크 사용량 baseline 기록
- **검증**:
  - 백업 tarball을 다른 머신 또는 별도 폴더에서 풀어보고 무결성 확인
  - `pmset -g` 출력에 sleep 0 확인
  - 기존 서비스 5분 stop → 다시 start → 정상 동작 (복구 가능성 검증)
  - `claude -p "ping"` 동작 확인 (subscription mode)
- **롤백 계획**: PM2 ecosystem 복원 + cloudflared config 복원 → 5분 내 원복

### Phase 0 — Workspace 셋업

- **목적**: 빈 골격 + CI 동작
- **결과물**:
  - `cargo new --bin nabi-rs` 후 workspace 변환
  - 9 crates 생성 (§ 4.2 참조)
  - GitHub Actions: clippy + test + fmt 통과
  - README, CLAUDE.md, AGENTS.md 기본 작성
  - `docs/handoff.md`에 이 문서 복사
- **소요**: 0.5일
- **검증**: `cargo run -p nabi-cli` → "nabi alive" 출력. CI 그린

### Phase 1 — Core 타입 + Mock Provider

- **목적**: Provider trait 정의, mock으로 agent loop 검증
- **결과물**:
  - `nabi-core::types` (Message, Session, RawEvent, StreamEvent, ChatRequest)
  - `nabi-core::provider` (Provider trait + Capabilities)
  - `nabi-core::agent` (간단한 chat loop)
  - `nabi-providers::mock` (고정 응답 echo + tool_use 시뮬레이션)
  - `nabi-cli`에서 `nabi chat "hello"` 명령 동작
- **소요**: 2-3일
- **검증**: mock provider로 echo 동작, unit test 통과

### Phase 2a — Claude CLI Protocol Spike (v0.2 분할)

- **목적**: 실제 CLI 동작 정밀 파악
- **결과물**:
  - 인증 모드 확인 매트릭스:
    - `ANTHROPIC_API_KEY` set + `--bare` → 성공
    - `ANTHROPIC_API_KEY` unset + `--bare` → 실패 ("Not logged in")
    - `ANTHROPIC_API_KEY` unset + non-bare → 성공 (subscription OAuth)
    - `ANTHROPIC_API_KEY` set + non-bare → 동작하되 API key 사용 (구독 X)
  - stream-json fixture 수집 (`tests/fixtures/stream-json/`)
    - plain text
    - tool use (Read, Bash)
    - permission abort (denied tool)
    - api retry
    - MCP load failure
    - session resume
  - parser golden test 작성 (fixture → 기대 StreamEvent)
  - first-token latency 실측 (subscription / api_key 양쪽 / bare / non-bare)
- **소요**: 3일
- **검증**: 모든 매트릭스 셀 결과 문서화. 성능 목표 § 1.4 실측 기준 재설정

### Phase 2b — Provider 본체 + Session

- **목적**: Provider 구현 안정화
- **결과물**:
  - `nabi-providers::claude_cli` 구현 (subscription / api_key 모드 분리)
  - NDJSON parser (stream_event 기반, unknown은 RawEvent로 audit)
  - stderr drain task
  - `--resume`, `--max-budget-usd`, `--max-turns` 통합
  - graceful subprocess shutdown (SIGTERM → wait → SIGKILL fallback)
  - `nabi-cli`에서 `nabi chat --provider claude --mode subscription` 동작
- **소요**: 3-4일
- **검증**: fixture-based test + 실제 invocation. 비용 cap 동작. 세션 재개 OK

### Phase 2c — Permission MCP Bridge

- **목적**: 원격 권한 승인 메커니즘
- **결과물**:
  - `nabi-server::permission_mcp` 모듈 (stdio MCP 서버)
  - `approval_prompt` tool 노출: `mcp__nabi_auth__approve`
  - 응답 포맷: `{"behavior":"allow","updatedInput":...}` / `{"behavior":"deny","message":...}`
  - DB pending → broadcast → 응답 수집 → MCP response 흐름
  - 일단 CLI prompt(stdin)로 응답 받는 토이 클라이언트로 검증 (PWA 전에)
- **소요**: 3-4일
- **검증**: 위험 명령 (rm -rf 등) 시도 시 prompt → allow/deny → 실제 실행/abort. tool_use_id 매핑 정확

### Phase 2d — Provider Trait 안정화

- **목적**: stream parser + cancellation + interrupt
- **결과물**:
  - StreamEvent enum 안정화 (이후 phase의 client 의존성)
  - 사용자 interrupt → child kill → graceful stream close
  - unknown raw event audit log
  - api_retry / mcp 실패 → 사용자 알림 이벤트
- **소요**: 2일
- **검증**: 임의 interrupt 시 stuck 없음. 모든 fixture parse 가능

### Phase 3 — Memory + Wiki + Skills (read-only first)

- **목적**: 나비 기존 자산 통합 (write 안 함)
- **결과물**:
  - `nabi-memory::store` (memory.db read-only, FTS5 검색, schema introspection)
  - `nabi-memory::wiki` (마크다운 파싱, frontmatter, MEMORY.md 허브)
  - `nabi-memory::sessions` (nabi.db 신규 — 세션/메시지/권한/audit/cost)
  - `nabi-memory::writer` (single-writer actor — nabi.db 전용)
  - `nabi-skills::loader` (SKILL.md 동적 로드, 관련성 검색)
  - 기존 memory.db 호환성 검증 (565+ 엔트리 그대로, write 시도 없음)
- **소요**: 4-5일
- **검증**: FTS 검색 결과 OpenClaw와 일치. wiki 모든 페이지 파싱 OK. memory.db 파일 mtime 변동 없음

### Phase 4 — Context Builder

- **목적**: 시스템 프롬프트 동적 구성 (토큰 예산 적용)
- **결과물**:
  - `nabi-context::builder` 완성
  - 토큰 예산 분배 (v0.3 — caveman 추가):
    - **caveman.md (layer 0, max 250)** ← § 4.6, 인터페이스 정책 + escape trigger 적용
    - AGENTS.md 글로벌 + 워크스페이스 (max 4K)
    - MEMORY.md hub 요약 (max 2K)
    - 관련 SKILL.md top-2 (max 4K)
    - memory.db FTS top-k (남은 예산)
    - 오늘/어제 daily log (max 2K)
  - 인터페이스 / escape trigger 평가 로직 (`nabi.yaml prompting:` 블록 기반)
  - 임시 파일 생성 후 `--append-system-prompt-file` 전달
  - `nabi-context::manifest` (어떤 자료/토큰량 들어갔는지 + caveman on/off 기록)
- **소요**: 3-4일
- **검증**:
  - 텐드릴 쿼리 → tendril.md 자동 로드. 토큰 한도 준수. manifest DB에 기록
  - 평범한 쿼리 → caveman 활성, 응답 단편화 확인
  - "자세히 설명해" trigger → caveman OFF, 평소 prose 응답
  - Telegram 경로 → caveman OFF 유지
  - 권한 요청 / 에러 메시지 → caveman OFF 유지

### Phase 5 — TUI 로컬 모드

- **목적**: 집에서 쓸 수 있는 인터페이스
- **결과물**:
  - `nabi-tui` ratatui 기반
  - 레이아웃: history + input + status bar
  - 키바인딩: ctrl+c (quit), ctrl+n (new), ctrl+r (resume), F2 (model)
  - partial token 실시간 표시
  - 도구 실행 상태 표시
  - 세션 picker (`~/.claude/projects/` + `nabi.db`)
  - 비용 누계 표시
  - 위험 명령 권한 요청 → 인라인 다이얼로그 (allow/deny)
- **소요**: 1주
- **검증**: 일상 작업 가능 수준. 응답 P50/P95 § 1.4 기준 도달

### Phase 6 — Multi-provider + Routing

- **목적**: Claude 외 모델 활용
- **결과물**:
  - `nabi-providers::openai_compat` (OpenRouter, Ollama)
  - `nabi-providers::routing` (작업 유형 → provider 매핑)
  - 설정: nabi.yaml에 provider list + routing rules
  - TUI에서 F2로 provider/model 전환
  - Capabilities 기반 분기 (tool_use, streaming, native_caching, session_resume)
- **소요**: 5-7일
- **검증**: 모델 전환 작동. OpenRouter 호출 정상. Ollama 로컬 호출 정상

### Phase 7 — Server-Client 분리 + macOS 운영 통합

- **목적**: daemon 모드 가동, MacBook primary 운영 정착
- **결과물**:
  - `nabi-server` 바이너리 생성
  - bind: `127.0.0.1:9912` 강제 (외부 인바운드 차단)
  - axum HTTP REST 엔드포인트 (§ 4.3 참조)
  - axum WebSocket 핸들러
  - 세션 broker (broadcast::Sender 기반)
  - TUI에 `--remote ws://...` 옵션 추가
  - 로컬 모드 / 원격 모드 동일 UX
  - graceful shutdown (SIGTERM handler, WAL checkpoint, WS notify)
  - tracing-appender 일일 로테이션 (`~/nabi/logs/`)
  - macOS Keychain 통합 (`keyring` 크레이트)
  - 작업 디렉토리 `~/nabi/workspace/` 강제 (TCC 회피)
  - PM2 ecosystem 추가, `max_restarts: 5`, `min_uptime: 10s`
  - launchd / PM2 startup 자동 시작 검증 (재부팅 → 자동 가동)
- **소요**: 1주
- **검증**:
  - 집 Mac에서 server 가동, 같은 머신 TUI가 WS로 접속 → 정상 동작
  - `kill -TERM <pid>` → 5초 내 깔끔 종료, DB 손상 없음
  - 재부팅 후 자동 시작 OK
  - 외부 머신에서 `:9912` 직접 접근 거부

### Phase 8 — Cloudflare Access + 신구 endpoint 병행

- **목적**: 안전한 외부 접근 + 기존 서비스 무중단 교체
- **결과물**:
  - 신규 hostname `nabi-new.ix-works.xyz` 추가 (cloudflared config)
  - CF Access Application 설정 (Google IdP, email allow list)
  - `nabi-server::auth` JWT 검증 + JWKS 캐싱 (TTL 24h, kid 미스 시 강제 refetch)
  - JWKS rotation 대응 (CF 6주 + 7일 유예 — 캐시 invalidation 로직)
  - 모든 엔드포인트 + WS upgrade에 auth middleware
  - TUI `nabi-tui auth login` 명령 (cloudflared access login 래핑)
  - 회사 노트북 + 폰 브라우저 접속 테스트
  - 1-2주 nabi-new로 검증 후 cutover
- **소요**: 5-7일 (검증 기간 별도)
- **검증**:
  - 회사 노트북 / 폰 브라우저에서 Google 로그인 → 접속 OK
  - 인증 없는 요청 401
  - TUI auth login → Keychain 저장 → WS 연결 OK
  - JWKS kid 변경 시뮬레이션 → 자동 refetch OK

### Phase 8c — 기존 서비스 cutover

- **목적**: nabi.ix-works.xyz 정식 전환
- **결과물**:
  - cloudflared config 변경: `nabi.ix-works.xyz` → `localhost:9912`
  - `cloudflared tunnel ingress validate` 통과
  - `pm2 reload` (다운타임 ~5초)
  - 기존 서비스 PM2 stop (삭제는 1-2주 후)
- **소요**: 0.5일
- **검증**: nabi.ix-works.xyz 접속 시 nabi-server 응답

### Phase 9 — PWA

- **목적**: 모바일 + 회사 컴퓨터 인터페이스
- **결과물**:
  - `nabi-web` 크레이트 (axum static + WS proxy)
  - `manifest.json` (홈화면 설치)
  - `service-worker.js` (cache versioning + skipWaiting + 오프라인 히스토리)
  - 모바일-first 레이아웃 (vanilla JS + minimal CSS 또는 Tailwind)
  - WebSocket 클라이언트
  - 세션 목록 / 새 세션 / 메시지 입력 / 토큰 스트림 표시
  - 권한 요청 모달 (in-app, WebSocket 기반)
  - PWA 업데이트 알림 (새 service worker 감지 시 토스트)
  - iOS Safari + Android Chrome 홈화면 추가 동작 확인
- **소요**: 1-2주
- **검증**: 폰에서 PWA 설치, 새 세션 시작, 진행 중 세션 이어 받기, 권한 모달 응답

### Phase 10 — Push Notification

- **목적**: 무인 자동화 + 권한 요청 알림 (백그라운드)
- **결과물**:
  - Web Push API + VAPID 키 발급
  - `nabi-server::push` 모듈
  - permission_request 발생 → 모든 구독 디바이스 푸시 (단 in-app 모달 우선)
  - 비용 임계치 초과 알림 (90% 도달)
  - 작업 완료 알림 (long-running 작업용)
  - iOS PWA 설치 + subscribe onboarding UI
- **소요**: 4-5일
- **검증**: 외출 중 폰으로 push 수신 → 허락 → 집 Mac에서 작업 진행

### Phase 11 — Telegram Bot (overflow)

- **목적**: 가장 가볍게 모바일 접근
- **결과물**:
  - `nabi-server::telegram` (teloxide 기반)
  - 봇 토큰 / chat_id allow list
  - text 메시지 → 새 세션 또는 활성 세션
  - permission_request → inline keyboard (Allow / Deny)
  - 정책: destructive permission은 default deny on Telegram (PWA에서만 allow)
  - 비용 알림, 작업 완료 알림
  - long polling (webhook은 추후)
- **소요**: 1주
- **검증**: 텔레그램에서 메시지 → 응답 받기. 위험 명령은 거부됨

### Phase 12 — Failover + 운영 강화

- **목적**: 24/7 안정 가동
- **결과물**:
  - Litestream으로 memory.db / nabi.db R2 백업 + 복구 절차 문서화
  - Windows server에 nabi-server warm standby (Phase 12까지 미뤘던 부분)
  - 헬스체크 엔드포인트 (`/healthz` 내부용, `/_health` CF용 분리)
  - Prometheus 메트릭 (`/metrics`)
  - 일일 비용 / 토큰 요약 텔레그램 자동 전송
  - 디스크 사용량 알림 (`~/.claude/projects/` 등 누적 모니터링)
  - 백업 복구 테스트 (실제로 다른 머신에서 복원해보기)
- **소요**: 1주
- **검증**: 집 Mac 강제 종료 시 Windows standby로 수동 전환 OK. R2에서 DB 복구 OK

**전체 일정**: Phase -1 ~ Phase 12, 약 9-11주. 5/15 시작 시 8월 초 완료 목표.

---

## 7. 코드 스켈레톤

### 7.1 Provider trait (`nabi-core/src/provider.rs`)

```rust
use async_trait::async_trait;
use futures::Stream;
use serde::{Deserialize, Serialize};
use std::pin::Pin;
use crate::error::NabiError;
use crate::types::{ChatRequest, StreamEvent};

pub type EventStream = Pin<Box<dyn Stream<Item = Result<StreamEvent, NabiError>> + Send>>;

#[async_trait]
pub trait Provider: Send + Sync {
    fn name(&self) -> &str;
    fn capabilities(&self) -> Capabilities;
    async fn chat(&self, req: ChatRequest) -> Result<EventStream, NabiError>;
    async fn list_models(&self) -> Result<Vec<ModelInfo>, NabiError>;
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Capabilities {
    pub tool_use: bool,
    pub streaming: bool,
    pub session_resume: bool,
    pub native_caching: bool,
    pub supports_mcp: bool,
    pub supports_computer_use: bool,
    pub supports_permission_prompt_tool: bool,  // v0.2
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ModelInfo {
    pub id: String,
    pub provider: String,
    pub max_tokens: Option<u32>,
    pub input_cost_per_mtok: Option<f64>,
    pub output_cost_per_mtok: Option<f64>,
}
```

### 7.2 Types (`nabi-core/src/types.rs`) — v0.2 개정

```rust
use serde::{Deserialize, Serialize};
use std::collections::BTreeMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum StreamEvent {
    // v0.3.1 — invocation_id는 Provider::chat() 진입 시 uuid v4로 생성, SessionStarted에 실어 전파.
    // 이후 같은 invocation의 모든 후속 이벤트는 downstream에서 invocation_id를 stash하여 태깅.
    SessionStarted { id: String, invocation_id: String, model: String, tools: Vec<String> },
    TextDelta { content: String },
    ToolUseStart { name: String, id: String },
    ToolUseInput { id: String, partial_json: String },
    ToolResult { id: String, output: String, is_error: bool },
    PermissionRequest { request_id: String, tool: String, input: serde_json::Value },
    ApiRetry { attempt: u32, reason: String },
    McpServerFailure { server: String, error: String },
    Done { cost_usd: Option<f64>, tokens: TokenUsage },
    Error { message: String },
    Unknown { raw: serde_json::Value },  // v0.2 — audit
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct TokenUsage {
    pub input: u64,
    pub output: u64,
    pub cache_read: u64,
    pub cache_write: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RawEvent {
    pub provider: String,
    pub session_id: Option<String>,
    pub raw: serde_json::Value,
    pub received_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatRequest {
    pub session_id: Option<String>,
    pub system_prompt_file: Option<std::path::PathBuf>,  // v0.2 — 파일 전달
    pub messages: Vec<Message>,
    pub tools: Vec<String>,              // v0.2 — 노출 도구 화이트리스트
    pub allowed_tools: Vec<String>,      // v0.2 — 자동 승인
    pub disallowed_tools: Vec<String>,   // v0.2 — 컨텍스트 제거 + bash 차단
    pub mcp_config_path: Option<std::path::PathBuf>,
    pub settings_path: Option<std::path::PathBuf>,
    pub model_hint: Option<String>,
    pub permission_mode: PermissionMode,
    pub max_budget_usd: Option<f64>,     // v0.2 — invocation 강제 cap
    pub max_turns: Option<u32>,          // v0.2
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum PermissionMode {
    Default,
    AcceptEdits,
    Plan,
    BypassPermissions,
    Auto,        // v0.3 — Claude CLI 2.1+ 지원
    DontAsk,     // v0.3 — Claude CLI 2.1+ 지원
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "role", rename_all = "lowercase")]
pub enum Message {
    User { content: String },
    Assistant { content: String },
    Tool { tool_use_id: String, content: String, is_error: bool },
}

// MCP 설정 — Claude Code 표준 (v0.2 수정)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct McpConfigFile {
    #[serde(rename = "mcpServers")]
    pub mcp_servers: BTreeMap<String, McpServer>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct McpServer {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub r#type: Option<String>,  // "stdio" / "http" / "streamable-http"
    #[serde(skip_serializing_if = "Option::is_none")]
    pub command: Option<String>,
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub args: Vec<String>,
    #[serde(default, skip_serializing_if = "BTreeMap::is_empty")]
    pub env: BTreeMap<String, String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub url: Option<String>,
}
```

### 7.3 Claude CLI Provider 핵심 (`nabi-providers/src/claude_cli.rs`) — v0.2 개정

```rust
use async_trait::async_trait;
use tokio::process::{Command, ChildStdout, ChildStderr};
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio_stream::wrappers::LinesStream;
use futures::StreamExt;
use std::process::Stdio;
use std::path::PathBuf;
use nabi_core::provider::{Provider, Capabilities, EventStream, ModelInfo};
use nabi_core::types::{ChatRequest, StreamEvent, TokenUsage, PermissionMode};
use nabi_core::error::NabiError;

#[derive(Debug, Clone, Copy)]
pub enum AuthMode {
    Subscription,  // ~/.claude OAuth, --bare 금지
    ApiKey,        // ANTHROPIC_API_KEY, --bare 허용
}

pub struct ClaudeCliProvider {
    binary_path: PathBuf,
    working_dir: PathBuf,
    default_model: String,
    auth_mode: AuthMode,
    use_bare: bool,  // ApiKey 모드에서만 true 가능
}

impl ClaudeCliProvider {
    pub fn new_subscription(binary: PathBuf, workdir: PathBuf, model: String) -> Self {
        Self {
            binary_path: binary,
            working_dir: workdir,
            default_model: model,
            auth_mode: AuthMode::Subscription,
            use_bare: false,  // 강제
        }
    }

    pub fn new_api_key(binary: PathBuf, workdir: PathBuf, model: String, bare: bool) -> Self {
        Self {
            binary_path: binary,
            working_dir: workdir,
            default_model: model,
            auth_mode: AuthMode::ApiKey,
            use_bare: bare,
        }
    }
}

#[async_trait]
impl Provider for ClaudeCliProvider {
    fn name(&self) -> &str {
        match self.auth_mode {
            AuthMode::Subscription => "claude_cli_subscription",
            AuthMode::ApiKey => "claude_cli_api_key",
        }
    }

    fn capabilities(&self) -> Capabilities {
        Capabilities {
            tool_use: true,
            streaming: true,
            session_resume: true,
            native_caching: true,
            supports_mcp: true,
            supports_computer_use: true,
            supports_permission_prompt_tool: true,
        }
    }

    async fn chat(&self, req: ChatRequest) -> Result<EventStream, NabiError> {
        let mut cmd = Command::new(&self.binary_path);

        // 핵심 print mode 플래그
        cmd.arg("-p").arg(serialize_messages_to_prompt(&req.messages));
        cmd.arg("--output-format").arg("stream-json");
        cmd.arg("--verbose");                    // partial token에 필수
        cmd.arg("--include-partial-messages");

        // bare 처리 — auth_mode에 따라
        if matches!(self.auth_mode, AuthMode::ApiKey) && self.use_bare {
            cmd.arg("--bare");
        }
        // Subscription 모드는 절대 --bare 안 붙임

        // 세션 재개
        if let Some(sid) = &req.session_id {
            cmd.arg("--resume").arg(sid);
        }

        // 시스템 프롬프트 — 파일 경로로 전달 (긴 컨텐츠 대응)
        if let Some(path) = &req.system_prompt_file {
            cmd.arg("--append-system-prompt-file").arg(path);
        }

        // 도구 제한 (노출)
        if !req.tools.is_empty() {
            cmd.arg("--tools").arg(req.tools.join(","));
        }
        // 자동 승인 (의도적, 명시적 필요)
        if !req.allowed_tools.is_empty() {
            cmd.arg("--allowedTools").arg(req.allowed_tools.join(","));
        }
        // 차단
        if !req.disallowed_tools.is_empty() {
            cmd.arg("--disallowedTools").arg(req.disallowed_tools.join(","));
        }

        // 권한 처리 — MCP tool 통해 원격 승인
        cmd.arg("--permission-prompt-tool").arg("mcp__nabi_auth__approve");
        cmd.arg("--permission-mode").arg(permission_mode_str(&req.permission_mode));

        // MCP
        if let Some(path) = &req.mcp_config_path {
            cmd.arg("--mcp-config").arg(path);
            cmd.arg("--strict-mcp-config");  // user/project/global MCP 격리
        }

        // 설정 (deny rules + sandbox)
        if let Some(path) = &req.settings_path {
            cmd.arg("--settings").arg(path);
        }

        // 비용 / 턴 강제 cap
        if let Some(budget) = req.max_budget_usd {
            cmd.arg("--max-budget-usd").arg(budget.to_string());
        }
        if let Some(turns) = req.max_turns {
            cmd.arg("--max-turns").arg(turns.to_string());
        }

        // 모델
        let model = req.model_hint.unwrap_or_else(|| self.default_model.clone());
        cmd.arg("--model").arg(&model);

        // 환경 설정
        cmd.current_dir(&self.working_dir)
           .stdout(Stdio::piped())
           .stderr(Stdio::piped())
           .kill_on_drop(true);

        // Subscription 모드면 ANTHROPIC_API_KEY 차단 (혹시 환경변수에 남아있어도)
        if matches!(self.auth_mode, AuthMode::Subscription) {
            cmd.env_remove("ANTHROPIC_API_KEY");
        }

        let mut child = cmd.spawn()
            .map_err(|e| NabiError::Spawn(e.to_string()))?;

        let stdout = child.stdout.take()
            .ok_or_else(|| NabiError::Internal("no stdout".into()))?;
        let stderr = child.stderr.take()
            .ok_or_else(|| NabiError::Internal("no stderr".into()))?;

        // stderr drain task (안 하면 block 가능)
        tokio::spawn(drain_stderr(stderr));

        // v0.3.1 — invocation_id 생성, parser에 전달
        let invocation_id = uuid::Uuid::new_v4().to_string();
        let stream = parse_ndjson_stream(stdout, child, invocation_id);
        Ok(Box::pin(stream))
    }

    async fn list_models(&self) -> Result<Vec<ModelInfo>, NabiError> {
        // v0.3 — 가격은 nabi.yaml `pricing:` 블록의 runtime overlay로 주입.
        // 하드코딩은 stream `total_cost_usd`가 권위 있는 값이므로 None으로 둠.
        // Anthropic 가격표 변동 시 nabi.yaml만 갱신.
        Ok(vec![
            ModelInfo {
                id: "claude-opus-4-7".into(),
                provider: self.name().into(),
                max_tokens: Some(200_000),
                input_cost_per_mtok: None,
                output_cost_per_mtok: None,
            },
            ModelInfo {
                id: "claude-sonnet-4-6".into(),
                provider: self.name().into(),
                max_tokens: Some(200_000),
                input_cost_per_mtok: None,
                output_cost_per_mtok: None,
            },
            ModelInfo {
                id: "claude-haiku-4-5".into(),
                provider: self.name().into(),
                max_tokens: Some(200_000),
                input_cost_per_mtok: None,
                output_cost_per_mtok: None,
            },
        ])
    }
}

fn permission_mode_str(m: &PermissionMode) -> &'static str {
    match m {
        PermissionMode::Default => "default",
        PermissionMode::AcceptEdits => "acceptEdits",
        PermissionMode::Plan => "plan",
        PermissionMode::BypassPermissions => "bypassPermissions",
        PermissionMode::Auto => "auto",
        PermissionMode::DontAsk => "dontAsk",
    }
}

fn serialize_messages_to_prompt(messages: &[nabi_core::types::Message]) -> String {
    messages.iter().rev()
        .find_map(|m| match m {
            nabi_core::types::Message::User { content } => Some(content.clone()),
            _ => None,
        })
        .unwrap_or_default()
}

async fn drain_stderr(stderr: ChildStderr) {
    let reader = BufReader::new(stderr);
    let mut lines = reader.lines();
    while let Ok(Some(line)) = lines.next_line().await {
        tracing::warn!(target: "claude_cli.stderr", "{}", line);
    }
}

// v0.3 — parser stateful 재구성
// Anthropic stream-json에서 `input_json_delta`는 자체에 tool_use_id를 안 담음.
// `content_block_start` (type=tool_use) 시점에 `index → id` 매핑을 기록하고
// `content_block_delta` 시 `event["index"]`로 조회해야 함.
// content_block_stop에서 매핑 정리.
//
// v0.3.1 — invocation_id 전파: Provider::chat() 진입 시 생성된 uuid를
// state에 담아 SessionStarted 이벤트 yield 시 주입.
struct StreamParserState {
    tool_use_ids: std::collections::HashMap<u64, String>,
    invocation_id: String,
}

fn parse_ndjson_stream(
    stdout: ChildStdout,
    mut child: tokio::process::Child,
    invocation_id: String,
) -> impl futures::Stream<Item = Result<StreamEvent, NabiError>> {
    let reader = BufReader::new(stdout);
    let lines = LinesStream::new(reader.lines());

    async_stream::stream! {
        let mut state = StreamParserState {
            tool_use_ids: std::collections::HashMap::new(),
            invocation_id,
        };
        let mut lines = Box::pin(lines);
        while let Some(line_result) = lines.next().await {
            match line_result {
                Ok(line) => {
                    if line.trim().is_empty() { continue; }
                    match parse_envelope(&line, &mut state) {
                        Ok(events) => {
                            for ev in events {
                                yield Ok(ev);
                            }
                        }
                        Err(e) => {
                            tracing::warn!("parse error: {} — line: {}", e, line);
                            if let Ok(raw) = serde_json::from_str::<serde_json::Value>(&line) {
                                yield Ok(StreamEvent::Unknown { raw });
                            }
                        }
                    }
                }
                Err(e) => {
                    yield Err(NabiError::Io(e.to_string()));
                    break;
                }
            }
        }
        let _ = child.wait().await;
    }
}

// stream-json 실제 구조 기반 (v0.3 — stateful)
fn parse_envelope(
    line: &str,
    state: &mut StreamParserState,
) -> Result<Vec<StreamEvent>, NabiError> {
    let raw: serde_json::Value = serde_json::from_str(line)
        .map_err(|e| NabiError::Parse(format!("ndjson: {}", e)))?;

    let ty = raw["type"].as_str().unwrap_or("");
    match ty {
        "system" => {
            let subtype = raw["subtype"].as_str().unwrap_or("");
            match subtype {
                "init" => {
                    // 새 invocation 시작 시 이전 tool_use 매핑 클리어
                    state.tool_use_ids.clear();
                    Ok(vec![StreamEvent::SessionStarted {
                        id: raw["session_id"].as_str().unwrap_or("").to_string(),
                        invocation_id: state.invocation_id.clone(),
                        model: raw["model"].as_str().unwrap_or("").to_string(),
                        tools: raw["tools"].as_array()
                            .map(|a| a.iter().filter_map(|v| v.as_str().map(String::from)).collect())
                            .unwrap_or_default(),
                    }])
                }
                "api_retry" => Ok(vec![StreamEvent::ApiRetry {
                    attempt: raw["attempt"].as_u64().unwrap_or(0) as u32,
                    reason: raw["reason"].as_str().unwrap_or("").to_string(),
                }]),
                _ => Ok(vec![StreamEvent::Unknown { raw }]),
            }
        }
        "stream_event" => parse_stream_event(&raw["event"], state),
        "result" => {
            let cost = raw["total_cost_usd"].as_f64();
            let tokens = TokenUsage {
                input: raw["usage"]["input_tokens"].as_u64().unwrap_or(0),
                output: raw["usage"]["output_tokens"].as_u64().unwrap_or(0),
                cache_read: raw["usage"]["cache_read_input_tokens"].as_u64().unwrap_or(0),
                cache_write: raw["usage"]["cache_creation_input_tokens"].as_u64().unwrap_or(0),
            };
            Ok(vec![StreamEvent::Done { cost_usd: cost, tokens }])
        }
        "error" => Ok(vec![StreamEvent::Error {
            message: raw["message"].as_str().unwrap_or("unknown").to_string(),
        }]),
        _ => Ok(vec![StreamEvent::Unknown { raw }]),
    }
}

fn parse_stream_event(
    event: &serde_json::Value,
    state: &mut StreamParserState,
) -> Result<Vec<StreamEvent>, NabiError> {
    let event_type = event["type"].as_str().unwrap_or("");
    match event_type {
        "content_block_start" => {
            let index = event["index"].as_u64().unwrap_or(0);
            let block = &event["content_block"];
            if block["type"] == "tool_use" {
                let id = block["id"].as_str().unwrap_or("").to_string();
                state.tool_use_ids.insert(index, id.clone());
                return Ok(vec![StreamEvent::ToolUseStart {
                    name: block["name"].as_str().unwrap_or("").to_string(),
                    id,
                }]);
            }
            Ok(vec![])
        }
        "content_block_delta" => {
            let index = event["index"].as_u64().unwrap_or(0);
            let delta = &event["delta"];
            match delta["type"].as_str() {
                Some("text_delta") => {
                    if let Some(text) = delta["text"].as_str() {
                        return Ok(vec![StreamEvent::TextDelta {
                            content: text.to_string(),
                        }]);
                    }
                }
                Some("input_json_delta") => {
                    // v0.3 — index로 content_block_start에서 기록한 tool_use_id 조회
                    if let (Some(id), Some(partial)) = (
                        state.tool_use_ids.get(&index),
                        delta["partial_json"].as_str(),
                    ) {
                        return Ok(vec![StreamEvent::ToolUseInput {
                            id: id.clone(),
                            partial_json: partial.to_string(),
                        }]);
                    } else {
                        tracing::warn!(
                            target: "claude_cli.parser",
                            "input_json_delta with no matching tool_use_id at index {}", index
                        );
                    }
                }
                _ => {}
            }
            Ok(vec![])
        }
        "content_block_stop" => {
            let index = event["index"].as_u64().unwrap_or(0);
            state.tool_use_ids.remove(&index);
            Ok(vec![])
        }
        _ => Ok(vec![]),  // message_start, message_stop, message_delta 등은 무시
    }
}
```

### 7.4 Permission MCP Server (`nabi-server/src/permission_mcp.rs`)

```rust
use serde::{Deserialize, Serialize};
use tokio::io::{stdin, stdout, AsyncBufReadExt, AsyncWriteExt, BufReader};
use std::sync::Arc;
use crate::AppState;

#[derive(Debug, Deserialize)]
struct ApprovalRequest {
    tool_name: String,
    input: serde_json::Value,
    tool_use_id: Option<String>,
}

#[derive(Debug, Serialize)]
#[serde(untagged)]
enum ApprovalResponse {
    Allow {
        behavior: String,  // "allow"
        #[serde(rename = "updatedInput")]
        updated_input: serde_json::Value,
    },
    Deny {
        behavior: String,  // "deny"
        message: String,
    },
}

pub async fn run_permission_server(state: Arc<AppState>) -> Result<(), anyhow::Error> {
    let stdin = stdin();
    let mut reader = BufReader::new(stdin).lines();
    let mut stdout = stdout();

    while let Some(line) = reader.next_line().await? {
        let rpc: serde_json::Value = serde_json::from_str(&line)?;
        // JSON-RPC 2.0 핸들링
        let method = rpc["method"].as_str().unwrap_or("");
        let id = rpc["id"].clone();

        let response = match method {
            "initialize" => handle_initialize(),
            "tools/list" => handle_tools_list(),
            "tools/call" => handle_tools_call(&rpc["params"], &state).await,
            _ => json_error(-32601, "Method not found"),
        };

        let envelope = serde_json::json!({
            "jsonrpc": "2.0",
            "id": id,
            "result": response,
        });
        let out = serde_json::to_string(&envelope)?;
        stdout.write_all(out.as_bytes()).await?;
        stdout.write_all(b"\n").await?;
        stdout.flush().await?;
    }
    Ok(())
}

fn handle_initialize() -> serde_json::Value {
    serde_json::json!({
        "protocolVersion": "2024-11-05",
        "capabilities": { "tools": {} },
        "serverInfo": { "name": "nabi-auth", "version": "0.1.0" }
    })
}

fn handle_tools_list() -> serde_json::Value {
    serde_json::json!({
        "tools": [{
            "name": "approve",
            "description": "Request user approval for tool use",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "tool_name": { "type": "string" },
                    "input": { "type": "object" },
                    "tool_use_id": { "type": "string" }
                },
                "required": ["tool_name", "input"]
            }
        }]
    })
}

async fn handle_tools_call(
    params: &serde_json::Value,
    state: &Arc<AppState>,
) -> serde_json::Value {
    let args: ApprovalRequest = match serde_json::from_value(params["arguments"].clone()) {
        Ok(a) => a,
        Err(e) => return json_error(-32602, &format!("Invalid params: {}", e)),
    };

    // 1. DB에 pending permission_request 저장
    let request_id = state.permissions.create_pending(&args).await.unwrap();

    // 2. broadcast → 모든 구독 디바이스
    state.broker.emit_permission_request(&request_id, &args).await;

    // 3. 응답 대기 (timeout 5분)
    let decision = state.permissions.await_decision(&request_id, std::time::Duration::from_secs(300)).await;

    // 4. MCP 표준 응답 포맷
    let result = match decision {
        Ok(d) if d.allowed => ApprovalResponse::Allow {
            behavior: "allow".into(),
            updated_input: d.updated_input.unwrap_or(args.input.clone()),
        },
        Ok(_) => ApprovalResponse::Deny {
            behavior: "deny".into(),
            message: "Denied by user".into(),
        },
        Err(_) => ApprovalResponse::Deny {
            behavior: "deny".into(),
            message: "Timeout — no user response within 5 minutes".into(),
        },
    };

    serde_json::json!({
        "content": [{
            "type": "text",
            "text": serde_json::to_string(&result).unwrap()
        }]
    })
}

fn json_error(code: i32, msg: &str) -> serde_json::Value {
    serde_json::json!({ "error": { "code": code, "message": msg } })
}
```

**v0.3 응답 wrap canary (Phase 2c 진입 직전 필수)**:

위 `handle_tools_call`는 응답을 `{content: [{type:"text", text: <stringified JSON>}]}` 로 감싸 돌려줌. 그러나 Claude Code 2.x가 이 wrap을 풀어 `behavior` 필드를 인식하는지는 공식 spec에 명시되지 않음. § 4.3 line 420은 평탄한 `{"behavior":"allow","updatedInput":{...}}`를 기대한다고 적혀 있어 두 형태 중 어느 쪽이 실제 동작하는지 불확실.

Phase 2c 시작 전 canary 절차:

1. 토이 MCP 서버 띄움 (위 코드 그대로)
2. `claude -p "rm -rf /tmp/canary" --permission-prompt-tool mcp__nabi_auth__approve --mcp-config ...` 실행
3. **응답 A (현재 코드)**: `{content:[{type:"text", text:'{"behavior":"allow","updatedInput":{...}}'}]}`
4. 도구 실행 여부 + `--debug api,mcp` 로그 확인
5. 실패 시 **응답 B (평탄)**: `{"behavior":"allow", "updatedInput":{...}}` 으로 변경하여 재시도
6. 결과를 § 7.4에 inline 기록 후 둘 중 하나로 코드 단일화

canary 통과 전까지 Phase 2c 머지 금지.

### 7.5 Claude Settings JSON (`config/claude-settings.json`) — v0.2 신규

`--settings` 플래그로 전달. 1차 방어선.

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.config/**)",
      "Read(~/Library/Keychains/**)",
      "Read(~/.claude/config.json)",
      "Bash(rm -rf /*)",
      "Bash(rm -rf ~*)",
      "Bash(rm -rf /Users/*)",
      "Bash(sudo:*)",
      "Bash(dd:*)",
      "Bash(mkfs:*)",
      "Bash(curl * | sh)",
      "Bash(curl * | bash)",
      "Bash(wget * -O -*|*)",
      "WebFetch(file://*)"
    ],
    "ask": [
      "Bash(git push:*)",
      "Bash(npm publish:*)",
      "Bash(cargo publish:*)",
      "Edit(/etc/**)",
      "Edit(~/Library/**)"
    ],
    "allow": [
      "Read(~/nabi/workspace/**)",
      "Edit(~/nabi/workspace/**)",
      "Bash(git status*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(cargo check*)",
      "Bash(cargo build*)",
      "Bash(cargo test*)",
      "Bash(ls *)",
      "Bash(cat ~/nabi/workspace/*)",
      "Grep(*)",
      "Glob(*)"
    ]
  },
  "sandbox": {
    "filesystem": {
      "allowRead": [
        "~/nabi/workspace",
        "~/nabi/wiki",
        "~/nabi-rs"
      ],
      "denyRead": [
        "~/.ssh",
        "~/.aws",
        "~/.config",
        "~/Library/Keychains",
        "~/.claude/config.json"
      ]
    },
    "network": {
      "allowedDomains": [
        "api.anthropic.com",
        "github.com",
        "raw.githubusercontent.com",
        "docs.anthropic.com",
        "claude.ai"
      ]
    }
  }
}
```

### 7.6 nabi.yaml (v0.2 개정)

```yaml
# 글로벌 설정
server:
  host: "127.0.0.1"                    # 외부 인바운드 차단 강제
  port: 9912                           # nabi.ix-works.xyz용 (기존 서비스와 분리)
  cf_team_domain: "your-team.cloudflareaccess.com"
  cf_aud: "abc123..."
  graceful_shutdown_timeout_sec: 10

# 프로바이더
providers:
  - name: claude_subscription
    type: claude_cli_subscription      # v0.2 — 모드 분리
    binary: "claude"
    working_dir: "~/nabi/workspace"
    default_model: "sonnet"
    settings_path: "config/claude-settings.json"
    # tools 화이트리스트 (노출)
    tools:
      - Read
      - Grep
      - Glob
      - Edit
      - Bash
      - WebSearch
    # 차단 (컨텍스트 제거 + bash 패턴)
    disallowed_tools:
      - "Bash(rm:*)"
      - "Bash(sudo:*)"
      - "Bash(curl:*)"
      - WebFetch
    # 자동 승인 — 비어 있음 (모든 도구는 permission-prompt-tool 거침)
    allowed_tools: []
    permission_mode: "default"
    max_budget_usd_per_invocation: 5.00
    max_turns: 30

  - name: claude_api
    type: claude_cli_api_key           # v0.2 — 개발/CI용
    binary: "claude"
    working_dir: "~/nabi/workspace"
    default_model: "sonnet"
    use_bare: true                     # api_key 모드에서만 허용
    api_key_source: "keychain:nabi-anthropic-api-key"
    max_budget_usd_per_invocation: 2.00
    max_turns: 20

  - name: openrouter
    type: openai_compatible
    base_url: "https://openrouter.ai/api/v1"
    api_key_source: "keychain:nabi-openrouter-api-key"
    default_model: "z-ai/glm-4.7"

  - name: ollama_studio
    type: openai_compatible
    base_url: "http://192.168.0.???:11434/v1"
    api_key_source: "none"
    default_model: "qwen3.5:32b"

# 기본값
defaults:
  provider: "claude_subscription"
  model: "sonnet"

# 보조 모델
auxiliary:
  provider: "openrouter"
  model: "google/gemini-flash-2.0"
  use_for:
    - session_title
    - memory_summary
    - btw

# 라우팅 규칙
routing:
  code_heavy: "claude_subscription/opus"
  default: "claude_subscription/sonnet"
  quick_chat: "openrouter/glm-4.7"
  sensitive: "ollama_studio/qwen3.5:32b"
  bulk: "openrouter/qwen-flash"

# 메모리 / wiki (v0.3 — 실측 경로)
memory:
  db_path: "~/clawd/memory-db/memory-sync.db"   # v0.3 — 실측 경로
  db_mode: "read_only"                          # v0.2 — Phase 4까지 read-only
  wiki_path: "~/clawd"                          # v0.3 — MEMORY.md / SOUL.md / PLAYBOOK.md 등
  sessions_db_path: "~/nabi-rs/data/nabi.db"    # 신규, nabi-rs 워크스페이스 하위
  shadow_table: "memory_events"                 # Phase 4부터 write-back
  legacy_app_db: "~/Projects/archived/claude-telegram-bridge/data/app.db"  # ETL용 (일회성)

# Skills
skills:
  paths:
    - "config/skills/"
    - "~/nabi/wiki/skills/"

# MCP 서버 (Claude CLI에 전달)
mcp_servers:
  nabi_auth:
    # nabi-server 자체를 MCP 모드로 띄움
    command: "nabi-server"
    args: ["--mcp-permission-server"]
    env: {}
  # nabi_memory MCP는 Phase 7+ deferred (v0.3.1).
  # 별도 crate가 아니라 nabi-server subcommand로 통합 예정:
  #   command: "nabi-server"
  #   args: ["--mcp-memory-server"]
  # 활성화 전까지 Claude는 자체 Read/Grep/Glob으로 ~/clawd/ 접근

# 보안
security:
  redact_secrets: true                 # 보조 안전장치
  redact_patterns:
    - "sk-[a-zA-Z0-9]{40,}"
    - "ghp_[a-zA-Z0-9]{36}"
    - "xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+"
  # 1차 방어는 claude-settings.json deny rules

# Context Builder 토큰 예산
context_budget:
  total_max: 30000
  caveman: 250                         # v0.3 — layer 0, 최우선
  agents_md: 4000
  memory_hub: 2000
  skills_top_k: 4000
  memory_fts: 8000
  daily_log: 2000

# 출력 스타일 (v0.3)
prompting:
  default_style: "caveman"             # caveman | relaxed
  caveman_snippet_path: "config/prompting/caveman.md"
  per_interface:
    tui: "caveman"
    pwa: "caveman"                     # UI에서 토글 가능
    telegram: "relaxed"                # async / 단편 위험
    permission_prompt: "relaxed"       # 명확성 우선
    error: "relaxed"
    onboarding: "relaxed"
  escape_triggers:                     # 사용자 입력에 포함되면 그 턴만 caveman 제외
    - "자세히"
    - "상세히"
    - "설명해"
    - "풀어서"
    - "verbose"
    - "explain"
    - "walk me through"
    - "in detail"

# Telegram
telegram:
  enabled: false                       # Phase 11에서 true
  bot_token_source: "keychain:nabi-telegram-bot-token"
  allowed_chat_ids:
    - 123456789
  destructive_permission_default: "deny"

# Push (PWA)
push:
  enabled: false                       # Phase 10에서 true
  vapid_public_source: "keychain:nabi-vapid-public"
  vapid_private_source: "keychain:nabi-vapid-private"
  subject: "mailto:bamsik@example.com"

# 비용
cost:
  daily_cap_usd: 30.0
  alert_thresholds: [0.5, 0.8, 0.95]
  alert_channels:
    - telegram
    - push

# 로깅
logging:
  dir: "~/nabi/logs"
  rotation: "daily"
  retention_days: 30
  level: "info,nabi=debug"

# macOS 운영 정책
macos:
  spotlight_exclusion_marker: true
  time_machine_exclusion:
    - "~/nabi/memory.db"
    - "~/nabi/memory.db-wal"
    - "~/nabi/memory.db-shm"
    - "~/nabi/nabi.db"
    - "~/nabi-rs/target"
  require_sleep_disabled: true         # 시작 시 pmset 검증
```

### 7.7 Routing 규칙 문법 (v0.3 신규)

§ 7.6 nabi.yaml의 `routing:` 블록 값은 `<provider_name>/<model_id>` 형식.

- `provider_name`: § 7.6 `providers:` 리스트의 `name` 필드와 정확히 일치
- `model_id`: 해당 provider가 인식하는 모델 id 또는 alias

파싱 위치: `nabi-providers::routing::RoutingRule::parse(&str) -> Result<(String, String), NabiError>`.

규칙:
- **첫 `/`만 split**, 양쪽 공백 trim
- `provider_name`에 `/` 금지 (예약), `model_id`는 `/` 허용 (OpenRouter `vendor/model` 형식 호환)
- 단일 토큰은 fallback: `defaults.provider` + 토큰을 model로 간주 — 단 명시적 권장 X
- 알 수 없는 provider 참조 시 **nabi-server boot 시점 fail-fast** (lazy 매칭 금지)

예시 (`입력 → provider | model`):

- `claude_subscription/opus` → `claude_subscription` | `opus`
- `claude_subscription/claude-sonnet-4-6` → `claude_subscription` | `claude-sonnet-4-6`
- `openrouter/z-ai/glm-4.7` → `openrouter` | `z-ai/glm-4.7`
- `ollama_studio/qwen3.5:32b` → `ollama_studio` | `qwen3.5:32b`

ChatRequest 흐름:
- 라우터가 `(provider_name, model_id)` 결정
- ProviderManager가 `provider_name`으로 Provider 인스턴스 lookup
- `ChatRequest.model_hint = Some(model_id)` 로 설정
- Provider 내부는 `model_hint`만 보고 `--model` 플래그에 전달

---

## 8. 운영

### 8.1 배포

**Mac (primary)**:

```bash
# 빌드
cd ~/nabi-rs
cargo build --release

# 바이너리 권한
chmod 700 ~/nabi-rs/target/release/nabi-server
chmod 700 ~/nabi-rs/target/release/nabi-tui

# secrets 디렉토리 권한
mkdir -p ~/nabi-rs/secrets
chmod 700 ~/nabi-rs/secrets

# PM2 ecosystem
cat > ecosystem-nabi.config.cjs << 'EOF'
module.exports = {
  apps: [
    {
      name: 'nabi-server',
      script: '/Users/bamsik/nabi-rs/target/release/nabi-server',
      args: '--config /Users/bamsik/nabi-rs/config/nabi.yaml',
      cwd: '/Users/bamsik/nabi-rs',
      autorestart: true,
      max_restarts: 5,
      min_uptime: '10s',
      max_memory_restart: '500M',
      kill_timeout: 10000,
      env: {
        RUST_LOG: 'info,nabi=debug',
        RUST_BACKTRACE: '1',
      }
    }
  ]
};
EOF

pm2 start ecosystem-nabi.config.cjs
pm2 save
pm2 startup  # 재부팅 시 자동 시작 (launchd 등록 명령 출력됨, sudo 실행)
```

### 8.2 Cloudflare Tunnel 설정

```yaml
# ~/.cloudflared/config.yml 에 추가
ingress:
  - hostname: nabi.ix-works.xyz
    service: http://localhost:9912
  - hostname: nabi-new.ix-works.xyz    # Phase 8 검증 기간
    service: http://localhost:9912
  - hostname: win.ix-works.xyz
    service: http://192.168.0.148:???
  - service: http_status:404
```

cloudflared는 **system daemon**으로 등록 (이미 운영 중일 가능성). PM2 사용 안 함.

### 8.3 Cloudflare Access 설정

1. CF Dashboard → Zero Trust → Access → Applications → Add
2. Self-hosted, Application name: nabi
3. Application domain: nabi.ix-works.xyz (그리고 검증용 nabi-new.ix-works.xyz)
4. Identity provider: Google
5. Policy: Allow if email is bamsik@your-email.com
6. Session duration: 30 days
7. App AUD tag 복사 → nabi.yaml `cf_aud`에 입력
8. Team domain → `cf_team_domain`에 입력

### 8.4 모니터링

- 일일 비용 리포트: nabi-server 내부 schedule → Telegram
- Prometheus 메트릭 (`/metrics`):
  - `nabi_session_active_count`
  - `nabi_message_total{role}`
  - `nabi_tool_use_total{tool}`
  - `nabi_permission_request_total{decision}`
  - `nabi_cost_usd_total{provider}`
  - `nabi_request_duration_seconds`
  - `nabi_disk_usage_bytes{path}` — `~/.claude/projects/`, `~/nabi/logs/`

### 8.5 macOS 운영 정책 (v0.2 신규)

#### 8.5.1 Sleep / Wake

```bash
# AC 전원 기준
sudo pmset -c sleep 0 disksleep 0 powernap 0 autorestart 1 womp 1

# 검증
pmset -g | grep -E "sleep|disksleep|powernap|autorestart|womp"

# 기대 출력:
# sleep        0
# disksleep    0
# powernap     0
# autorestart  1
# womp         1
```

배터리 모드는 별도 (`-b`). MacBook 클램셸 모드 운영 시 외부 모니터/키보드/전원 필수.

#### 8.5.2 자동 업데이트 비활성

```bash
sudo softwareupdate --schedule off
```

보안 패치는 수동 적용 (월 1회 점검).

#### 8.5.3 Time Machine 제외

```bash
sudo tmutil addexclusion -p ~/nabi/memory.db
sudo tmutil addexclusion -p ~/nabi/memory.db-wal
sudo tmutil addexclusion -p ~/nabi/memory.db-shm
sudo tmutil addexclusion -p ~/nabi/nabi.db
sudo tmutil addexclusion -p ~/nabi/nabi.db-wal
sudo tmutil addexclusion -p ~/nabi/nabi.db-shm
sudo tmutil addexclusion -p ~/nabi-rs/target
```

#### 8.5.4 Spotlight 제외

```bash
touch ~/nabi/.metadata_never_index
touch ~/nabi-rs/.metadata_never_index
```

#### 8.5.5 자동 시작 (재부팅 후)

선택 A — PM2 (간단, 권장):
- 자동 로그인 활성 필요
- `pm2 startup` 명령으로 LaunchAgent 등록

선택 B — launchd 직접 (보안 우선):
- `/Library/LaunchDaemons/com.nabi.server.plist` 작성
- KeepAlive, RunAtLoad true
- 자동 로그인 불필요
- 권한 관리 필요

Open Decision § 10.10 참조.

#### 8.5.6 macOS Application Firewall

- 외부 인바운드 차단 (System Settings → Network → Firewall)
- nabi-server는 `127.0.0.1` 바인딩으로 어차피 외부 접근 불가
- 추가 보호 차원

### 8.6 백업

**Litestream 설정** (Phase 12):

```yaml
# /etc/litestream.yml
dbs:
  - path: /Users/bamsik/nabi/memory.db
    replicas:
      - type: s3
        bucket: nabi-backups
        endpoint: https://<account>.r2.cloudflarestorage.com
        access-key-id: ...
        secret-access-key: ...

  - path: /Users/bamsik/nabi/nabi.db
    replicas:
      - type: s3
        bucket: nabi-backups
        endpoint: https://<account>.r2.cloudflarestorage.com
```

**Wiki Git 자동 커밋** (기존 운영):
```bash
cd ~/nabi/wiki
git add -A
git diff-index --quiet HEAD || git commit -m "wiki: auto $(date -Iseconds)"
```

### 8.7 비용 관리 (v0.3 — 트랙 분리)

두 트랙은 **독립적**으로 운영. nabi-rs가 합산하지 않음. § 1.4 성공지표의 "월 $200"과 nabi.yaml `daily_cap_usd: 30.0`은 서로 다른 트랙의 다른 단위.

#### 트랙 A — Max 구독 (claude_cli_subscription, 6/15+ 기본)

- 차감: Anthropic Agent SDK 크레딧 풀 (Max 20x = 월 $200 상당, API list rate로 차감)
- nabi-rs는 직접 청구 받지 않음. nabi.db는 stream `total_cost_usd`를 **참고치**로만 기록
- **extra usage OFF로 시작** → 크레딧 소진 시 자동 일시 차단됨 (Anthropic 측)
- extra usage ON 검토는 사용 패턴 1개월 누적 후. ON 시 Anthropic Console에서 monthly cap 별도 설정 필수 (nabi.yaml로는 제어 불가)
- nabi.yaml `daily_cap_usd`는 이 트랙에서는 **soft 알림용** (hard cap 효력 없음)
- § 1.4 "월 비용 Max 20x 구독 $200 한도 내" = 이 트랙

#### 트랙 B — API key 직과금

- 청구: Anthropic (claude_cli_api_key) / OpenRouter / 기타 직접 결제
- nabi.yaml `daily_cap_usd: 30.0` = **이 트랙의 hard cap**. 초과 시 nabi-server가 자동 차단 + 알림
- `alert_thresholds: [0.5, 0.8, 0.95]` (cap 대비 누적 비율)
- Phase -1 ~ 6/14 개발 / CI / 격리 테스트 / 보조 provider 모두 이 트랙

#### 공통 강제 (트랙 무관, 모든 invocation)

- `--max-budget-usd 5.00` per invocation (Claude CLI 자체 cap)
- `--max-turns 30` per invocation
- 라우팅으로 비용 분산: 잡일은 GLM / Gemini Flash로
- context manifest 검토로 토큰 출처 추적 (§ 9.5 `context_manifests`)

#### 운영 점검

- 일일 텔레그램 리포트: 트랙별 누적 비용 / 호출 수 / 평균 토큰
- 크레딧 소진 임박 시 (트랙 A 80%+ 또는 트랙 B 80%+) 알림
- 트랙 B `daily_cap_usd` 도달 시 routing이 자동으로 트랙 A로 fallback (단 트랙 A 크레딧 잔량 확인)

### 8.8 PWA 업데이트

```javascript
// service-worker.js — cache versioning + skipWaiting
const CACHE_VERSION = 'nabi-v0.2.0';
const CACHE_NAME = `${CACHE_VERSION}-static`;

self.addEventListener('install', (event) => {
  self.skipWaiting();  // 즉시 활성
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache =>
      cache.addAll(['/index.html', '/app.js', '/style.css'])
    )
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k))
      )
    ).then(() => self.clients.claim())
  );
});
```

`app.js`는 `navigator.serviceWorker.ready` 후 새 SW 감지 시 사용자에게 새로고침 토스트.

---

## 9. 보안

### 9.1 인증 레이어

- **Layer 1 (외부)**: Cloudflare Zero Trust Access
  - Google OAuth, email allow list
  - 30일 세션, 자동 갱신
  - 의심 패턴 자동 차단
  - JWKS 6주 + 7일 유예 rotation 대응

- **Layer 2 (내부)**: nabi-server 자체 검증
  - CF JWT 서명 검증 (kid + JWKS 캐시)
  - aud claim 검증
  - email claim → 내부 사용자 식별
  - WebSocket upgrade 시 동일 검증

- **Layer 3 (작업별)**: 권한 시스템
  - `claude-settings.json` deny/ask/allow 규칙 (1차 방어)
  - `--permission-prompt-tool` MCP 경유 사용자 승인 (2차)
  - PermissionMode 적용
  - 위험 명령 패턴 (deny rules)
  - 세션별 작업 디렉토리 격리 (`~/nabi/workspace/`)
  - sandbox filesystem.denyRead

### 9.2 시크릿 관리 (v0.2 강화)

- **1순위: macOS Keychain**
  - `keyring` 크레이트 + features `apple-native`
  - 키 이름 규약: `nabi-<purpose>` (e.g. `nabi-anthropic-api-key`)
- **2순위: 환경변수**
- **3순위: 파일 (`secrets/` chmod 700, 파일 0600)**
- **금지**: nabi.yaml에 평문 시크릿
- **redact_secrets**: 보조 안전장치 (1차 방어 아님)
- **OpenClaw.app LaunchAgent**: 절대 활성화 금지

### 9.3 Claude CLI 시크릿 노출 방지 (v0.2 신규)

**1차 방어**: `claude-settings.json` deny rules (§ 7.5)

- `Read(./.env)`, `Read(./secrets/**)`, `Read(~/.ssh/**)` 등 차단
- `sandbox.filesystem.denyRead`로 OS 레벨 차단
- `sandbox.network.allowedDomains`로 외부 통신 제한

**2차 방어**: 작업 디렉토리 격리

- `working_dir: ~/nabi/workspace/`로 강제
- agent는 그 외 디렉토리 접근 차단
- 다른 디렉토리 필요 시 `--add-dir` 명시

**3차 방어 (보조)**: redact_secrets

- 사용자 입력 / tool 결과에 정규식 매치 시 `[REDACTED]`
- 단 Claude CLI 내부 도구가 파일 읽으면 이미 모델 컨텍스트 진입 후

### 9.4 Secret Canary Test (v0.2 신규)

Phase 2c 진입 전 통과 필수:

```bash
# 1. 테스트용 fake secret 생성
echo "FAKE_API_KEY=sk-test-do-not-use-canary-xyz123" > ~/nabi-rs/.env.canary

# 2. Claude에게 읽으라고 지시
claude -p "Read .env.canary and tell me what's in it" \
    --settings config/claude-settings.json \
    --permission-mode default

# 3. 기대 결과: Read 도구가 deny rules에 막혀 거부됨
# 4. 실패 (읽힘) 시: settings deny rules 보강 후 재시도
```

매 빌드마다 CI에서 자동 실행.

### 9.5 감사 로그

```sql
-- nabi.db
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY,
    timestamp INTEGER NOT NULL,
    session_id TEXT,
    event_type TEXT NOT NULL,
    actor TEXT,                         -- device_id 또는 'agent'
    details JSON NOT NULL,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_session ON audit_log(session_id);
CREATE INDEX idx_audit_type ON audit_log(event_type);
CREATE INDEX idx_audit_time ON audit_log(timestamp);

-- raw event audit (Claude CLI 원본 stream)
CREATE TABLE raw_events (
    id INTEGER PRIMARY KEY,
    session_id TEXT,
    provider TEXT,
    raw JSON NOT NULL,
    received_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- context manifest (어떤 자료가 들어갔는지)
CREATE TABLE context_manifests (
    id INTEGER PRIMARY KEY,
    session_id TEXT NOT NULL,
    message_seq INTEGER NOT NULL,
    skills_loaded JSON,                 -- 어떤 SKILL.md 들어갔는지
    memory_entries JSON,                -- 어떤 memory.db entry id 들어갔는지
    wiki_pages JSON,
    token_estimate INTEGER,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### 9.6 위협 모델

- **위협 1**: CF Access 우회 시도
  - 완화: origin 직접 노출 X (Tunnel only), JWT 검증 필수, bind 127.0.0.1

- **위협 2**: 디바이스 도난 (폰 분실)
  - 완화: per-device session revocation, CF Access 세션 즉시 무효화

- **위협 3**: prompt injection으로 위험 명령 유도
  - 완화: deny rules + sandbox + permission-prompt-tool + max-budget-usd

- **위협 4**: LLM에 시크릿 유출
  - 완화: deny rules (1차) + sandbox (2차) + redact (보조) + canary test (검증)

- **위협 5**: nabi-server 자체 취약점
  - 완화: 외부 입력 항상 검증, `cargo audit`, bind 127.0.0.1

- **위협 6**: Telegram 채널 침해
  - 완화: destructive permission default deny, chat_id allow list, nonce/expiry

- **위협 7**: macOS 시스템 침해
  - 완화: 자동 로그인 시 보안 trade-off 인지, Keychain 사용, secrets/ 권한 0700

---

## 10. Open Decisions

### 10.1 Web 프레임워크 — **결정 완료 (v0.2)**

- **A**: axum + vanilla JS / htmx ✅
- 사유: 1인용 PWA에 React 오버헤드 과함. Rust 일관성. Phase 13+에서 필요 시 React 전환

### 10.2 Server primary 머신 — **결정 완료 (v0.2)**

- **MacBook M4** ✅
- 사유: Claude CLI 인증 보유 머신, 운영 중인 자산 위치
- 조건: § 8.5 macOS 정책 적용 필수

### 10.3 OpenClaw 마이그레이션 — **결정 완료**

- **점진 이관**
- memory.db는 Phase 4까지 read-only

### 10.4 라이선스 — **결정 완료**

- **Private/Proprietary**

### 10.5 MCP 도구 구현 위치 — **결정 완료 (v0.2)**

- Claude provider: MCP 위임
- non-Claude provider: 내부 tool runner
- **Permission approval: Phase 2c부터 MCP 구현 필수**

### 10.6 세션 충돌 처리 — **결정 완료**

- **A**: 첫 입력자 lock
- provider-level mutex 추가 (Claude session id 중복 `--resume` 방지)

### 10.7 PWA 알림 — **결정 완료**

- 홈화면 설치 전제
- Telegram fallback 병행
- iOS는 첫 접속 시 onboarding으로 설치 유도

### 10.8 기존 nabi.ix-works.xyz 서비스 처리 — **결정 완료 (v0.3)**

- 서비스 정체: `~/Projects/claude-telegram-bridge` (FastAPI, port 8787, PM2 4개) — § 2.4 참조
- **결정: 아카이빙** (사용자 확인)
- 실행 순서:
  1. Phase 8c cutover (nabi-rs가 `nabi.ix-works.xyz` 응답)
  2. PM2 `nabi-web` / `telegram-poller` / `claude-telegram` stop (cloudflared-tunnel은 계속 가동)
  3. `~/Projects/claude-telegram-bridge` → `~/Projects/archived/claude-telegram-bridge` 이전
  4. `pm2 delete`는 1-2주 안정 가동 확인 후
- 데이터 이관: `app.db`의 conversations / messages_fts는 nabi-rs `nabi.db`로 ETL (Phase 3 또는 별도 일회성 스크립트). `~/clawd/memory-db/memory-sync.db`는 그대로 read-only 재사용

### 10.9 포트 번호 — **결정 완료 (v0.2)**

- **9912** (기존 14 PM2 서비스와 분리)
- Phase -1에서 충돌 검증

### 10.10 macOS 자동 로그인 — **결정 필요**

- 옵션 A: 자동 로그인 활성 + PM2 startup (간단, 보안 trade-off)
- 옵션 B: launchd Daemon 직접 등록 (자동 로그인 불필요)
- **임시 추천**: A. 단일 사용자 머신이고 물리 접근 위험 낮으면 OK
- **답 필요**: 본인 운영 환경 판단

### 10.11 다운타임 윈도우 — **결정 필요**

- Phase 8c (cutover) 실행 시간대
- nabi-new에서 한 달 검증 후 cutover면 다운타임 ~5초
- 가능 시간 명시 (밤 또는 새벽)

---

## 11. 리스크 & 완화

### 11.1 Anthropic 정책 추가 변경

- **위험**: 6/15 이후 추가 제약 가능 (subprocess 래핑 자체 차단 등)
- **완화**:
  - Provider 추상화 견고 → Claude 차단 시 다른 provider로 즉시 전환
  - OpenRouter로 Claude 모델 우회 (가격 차이만)
  - Smelt 식 ChatGPT/Copilot 구독 OAuth 옵션 백로그 (Phase 13+)

### 11.2 비용 폭주

- **위험**: agent loop 무한 / context bloat / 무인 자동화 spike
- **완화 (v0.2 강화)**:
  - `--max-budget-usd 5.00` per invocation 강제
  - `--max-turns 30` 강제
  - daily_cap_usd 강제
  - extra usage 기본 OFF
  - 라우팅 잡일 저비용 모델
  - 일일 텔레그램 비용 리포트
  - context manifest 검토 (어디서 토큰 쓰는지 추적)

### 11.3 단일 머신 의존

- **위험**: home Mac 다운 시 모든 디바이스 동시 불가
- **완화**:
  - macOS sleep disable / 자동 재시작
  - PM2 max_restarts + min_uptime
  - launchd 자동 시작
  - Windows server warm standby (Phase 12)
  - Litestream 실시간 복제
  - PWA 오프라인 히스토리 캐싱

### 11.4 보안 사고

- **위험**: CF Access 우회 / 디바이스 도난 / prompt injection / 시크릿 유출
- **완화**:
  - 다층 인증 (§ 9.1)
  - audit log + 일일 검토
  - per-device revocation
  - deny rules + sandbox + canary test
  - Keychain + 파일 권한

### 11.5 Rust 개발 속도

- **위험**: TS/Python 대비 느린 개발, 일정 지연
- **완화**:
  - Phase 단위 작게 자르기
  - 검증된 crate 사용
  - Claude Code CLI로 코드 생성 가속

### 11.6 OpenClaw 운영 중단

- **위험**: nabi-rs 안정화 전 OpenClaw 사용 불가
- **완화**:
  - 병행 운영
  - memory.db는 read-only (Phase 4까지)

### 11.7 stream-json 포맷 변경

- **위험**: Anthropic이 stream-json 스펙 변경 시 parser 깨짐
- **완화 (v0.2)**:
  - Unknown event raw 저장 → 운영 중 발견
  - fixture-based test로 회귀 감지
  - 신버전 detection 시 알림

### 11.8 macOS 업데이트 강제 재부팅

- **위험**: 보안 업데이트로 강제 재부팅 다운타임
- **완화**:
  - 자동 업데이트 비활성
  - 월 1회 점검 (예정된 다운타임)
  - PM2 startup + autorestart

---

## 12. 참고 자료

### 12.1 Prior Art

- **Smelt** (https://github.com/leonardcser/smelt)
  - Rust TUI 코딩 에이전트, 멀티 프로바이더
  - 구독 인증 (Codex OAuth, Copilot device-code) 참고
  - 설정 구조 (YAML, provider/model 명명) 참고
  - 4-mode 권한 시스템 참고

- **claude-code-rust** (https://github.com/srothgan/claude-code-rust)
  - Rust + TS bridge로 Agent SDK 래핑
  - 3-layer 아키텍처 참고
  - npm 배포 패턴

- **Octomind** (https://github.com/Muvon/octomind)
  - Specialist agent 등록소 (tap 시스템)
  - TOML 1 파일 = 1 agent 패턴
  - Adaptive compaction

- **Claurst** (https://github.com/Kuberwastaken/claurst)
  - Claude Code clean-room Rust 재구현
  - Multi-provider, plugin system

- **Nanobot** (HKU) — 4K 라인 Python, 학습용

- **Hermes Agent** — Skill learning loop

### 12.2 Anthropic 문서

- Run Claude Code programmatically: https://code.claude.com/docs/en/headless
- CLI reference: https://code.claude.com/docs/en/cli-reference
- SDK Permissions: https://docs.claude.com/en/api/agent-sdk/permissions
- Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- Use Claude Agent SDK with Claude plan: https://support.claude.com/en/articles/15036540

### 12.3 핵심 Rust 크레이트 문서

- tokio: https://docs.rs/tokio
- axum: https://docs.rs/axum
- ratatui: https://ratatui.rs
- rusqlite: https://docs.rs/rusqlite
- keyring: https://docs.rs/keyring
- async-trait: https://docs.rs/async-trait
- teloxide: https://docs.rs/teloxide
- jsonwebtoken: https://docs.rs/jsonwebtoken
- tracing: https://docs.rs/tracing
- async-stream: https://docs.rs/async-stream

### 12.4 인프라

- Cloudflare Tunnel: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- Cloudflare Zero Trust Access: https://developers.cloudflare.com/cloudflare-one/identity/
- cloudflared access CLI: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/
- Litestream: https://litestream.io
- PM2: https://pm2.keymetrics.io

### 12.5 macOS 운영

- pmset(1): `man pmset`
- tmutil(8): `man tmutil`
- launchd: `man launchd.plist`

---

## 13. 초기 셋업 명령어

### 13.1 Phase -1 자동화 스크립트 (v0.2 신규)

```bash
#!/usr/bin/env bash
# scripts/phase-minus-1.sh
set -euo pipefail

BACKUP_DIR="$HOME/backups/nabi-migration-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

echo "==> Phase -1: 사전 작업 시작"

# 1. 기존 서비스 인벤토리
echo "==> PM2 list dump"
pm2 list --no-color > "$BACKUP_DIR/pm2-list.txt"
pm2 jlist > "$BACKUP_DIR/pm2-jlist.json"

# 2. cloudflared config 백업
if [ -f ~/.cloudflared/config.yml ]; then
    cp ~/.cloudflared/config.yml "$BACKUP_DIR/cloudflared-config.yml"
fi

# 3. nabi 자산 백업
if [ -d ~/.claude ]; then
    tar czf "$BACKUP_DIR/dot-claude.tar.gz" -C ~ .claude
fi
if [ -d ~/nabi ]; then
    tar czf "$BACKUP_DIR/nabi.tar.gz" -C ~ nabi
fi

# 4. SQLite 백업
for db in ~/nabi/memory.db ~/nabi/ix-data.db; do
    if [ -f "$db" ]; then
        sqlite3 "$db" ".backup '$BACKUP_DIR/$(basename $db).backup'"
    fi
done

# 5. claude CLI 상태 확인
claude /status > "$BACKUP_DIR/claude-status.txt" 2>&1 || true

# 6. 포트 충돌 확인
echo "==> Port 9912 check"
if lsof -i :9912 > "$BACKUP_DIR/port-9912.txt" 2>&1; then
    echo "WARNING: Port 9912 is in use!"
    cat "$BACKUP_DIR/port-9912.txt"
fi

# 7. 디스크 사용량 baseline
df -h ~ > "$BACKUP_DIR/disk-baseline.txt"
du -sh ~/.claude/projects 2>/dev/null > "$BACKUP_DIR/claude-projects-size.txt" || true

# 8. macOS 정책 적용
echo "==> Applying macOS policies"
sudo pmset -c sleep 0 disksleep 0 powernap 0 autorestart 1 womp 1
sudo softwareupdate --schedule off

# Time Machine 제외
for path in ~/nabi/memory.db ~/nabi/memory.db-wal ~/nabi/memory.db-shm \
            ~/nabi/nabi.db ~/nabi/nabi.db-wal ~/nabi/nabi.db-shm; do
    if [ -e "$path" ]; then
        sudo tmutil addexclusion -p "$path" 2>/dev/null || true
    fi
done

# Spotlight 제외
mkdir -p ~/nabi
touch ~/nabi/.metadata_never_index
mkdir -p ~/nabi-rs
touch ~/nabi-rs/.metadata_never_index

echo "==> 완료. 백업: $BACKUP_DIR"
echo "==> 검증 단계로 진행"
```

### 13.2 워크스페이스 생성

```bash
mkdir -p ~/nabi-rs
cd ~/nabi-rs
git init

# 루트 Cargo.toml (workspace)
cat > Cargo.toml << 'EOF'
[workspace]
resolver = "2"
members = [
    "crates/nabi-core",
    "crates/nabi-providers",
    "crates/nabi-memory",
    "crates/nabi-skills",
    "crates/nabi-context",
    "crates/nabi-server",
    "crates/nabi-tui",
    "crates/nabi-web",
    "crates/nabi-cli",
]

[workspace.package]
version = "0.1.0"
edition = "2021"
authors = ["bamsik"]
license = "Proprietary"
repository = "private"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
async-trait = "0.1"
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
tracing-appender = "0.2"
clap = { version = "4", features = ["derive", "env"] }
reqwest = { version = "0.12", default-features = false, features = ["json", "stream", "rustls-tls"] }
axum = { version = "0.7", features = ["ws", "macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "cors"] }
ratatui = "0.29"
crossterm = "0.28"
rusqlite = { version = "0.32", features = ["bundled", "json", "blob"] }
r2d2 = "0.8"
r2d2_sqlite = "0.25"
chrono = { version = "0.4", features = ["serde"] }
futures = "0.3"
tokio-stream = { version = "0.1", features = ["io-util"] }
async-stream = "0.3"
keyring = { version = "3", features = ["apple-native"] }
jsonwebtoken = "9"
teloxide = "0.13"

[profile.release]
lto = "thin"
codegen-units = 1
strip = true
EOF

# 크레이트 초기화
for crate in nabi-core nabi-providers nabi-memory nabi-skills nabi-context; do
    cargo new --lib crates/$crate
done
for crate in nabi-server nabi-tui nabi-web nabi-cli; do
    cargo new --bin crates/$crate
done

# .gitignore
cat > .gitignore << 'EOF'
target/
Cargo.lock.bak
*.swp
.DS_Store
secrets/
config/local.yaml
*.db
*.db-journal
*.db-wal
*.db-shm
.env
.env.*
.metadata_never_index
fixtures/local/
EOF

# 초기 README
cat > README.md << 'EOF'
# nabi-rs

개인용 AI agent harness. 자세한 내용은 [handoff 문서](./docs/handoff.md) 참조.

## Quick start

```bash
# Phase -1 (최초 1회)
./scripts/phase-minus-1.sh

# 빌드
cargo build --release

# 실행
cargo run -p nabi-cli
```
EOF

# 핸드오프 문서 보존
mkdir -p docs
# (이 파일을 docs/handoff.md로 복사)

# CLAUDE.md
cat > CLAUDE.md << 'EOF'
# Project: nabi-rs

개인용 AI agent harness, Rust 워크스페이스.
설계 문서: docs/handoff.md (필수 참조)

## 운영 규칙
- 마크다운 테이블 금지, bullet list만
- 모든 작업 in-session 완료
- 코드 작성 시 안정성 우선 (`anyhow`, 명시적 에러)
- 비동기는 tokio 일관
- memory.db는 Phase 4까지 read-only
- --bare는 api_key 모드에서만

## Phase 진행
현재 Phase: -1 (사전 작업)
다음 Phase: 0 (workspace 셋업)

각 Phase 작업 시:
1. docs/handoff.md § 6에서 해당 Phase 결과물 확인
2. 검증 기준 통과 후에만 commit
3. 결정사항 변경 시 docs/handoff.md 업데이트 + version bump
EOF

# 첫 commit
git add .
git commit -m "Phase 0: workspace skeleton (v0.2 handoff)"
```

### 13.3 첫 빌드 검증

```bash
cargo build
cargo run -p nabi-cli   # "nabi alive" 출력
```

### 13.4 개발 도구

```bash
rustup component add clippy rustfmt
cargo install cargo-audit cargo-watch
cargo watch -x check -x test
```

---

## 14. 변경 이력

### 0.3.2 (2026-05-16) — invocation_id 전파 메커니즘 명시 (advisor 3차)

advisor 3차 검토에서 진짜 결함 1개 발견 — invocation_id가 § 15에 정의되었으나 § 7.2 StreamEvent enum 어디에도 필드 없음 → Phase 2b 구현 시 막힘.

- § 7.2 `SessionStarted` variant에 `invocation_id: String` 필드 추가
- § 7.3 `StreamParserState`에 `invocation_id` 필드, `parse_ndjson_stream(.., invocation_id)` 시그니처, `Provider::chat()` 진입 시 uuid v4 생성 후 전달
- § 7.3 `parse_envelope`의 `init` 분기에서 `state.invocation_id.clone()` 으로 SessionStarted 채움
- § 15.2.1: 전파 메커니즘 = SessionStarted 1회 전송 → downstream stash (envelope 비대화 방지) 명시

**advisor 권고**: 무한 리뷰 모드 진입 신호. 이 패치 후 더 이상 검토 라운드 X, Phase -1 진입.

**Phase 진행 중 자연스럽게 만나면 고칠 항목** (지금 패치 X):
- § 17 `session_archive` server→client ack 응답 없음
- § 15 `cost_ledger.track` enum vs § 8.7 명명 불일치 (`subscription`/`api_key` vs `트랙 A`/`트랙 B`)
- § 22.5 `tiktoken-rs`는 OpenAI BPE — Claude 토큰 카운트는 근사치 한계

### 0.3.1 (2026-05-16) — cross-section 정합성 패치 + 운용성

advisor 세부검토에서 발견한 v0.3 패치들 간 모순 6건 해소:

- § 15.1 `messages.role`: `system` 제거 (§ 7.2 Message enum과 일치). meta prompt는 ChatRequest 별도 필드로
- § 15.1: `messages.invocation_id` 컬럼 + § 15.2.1 신규 (uuid v4 생성·전파·join 규약)
- § 15.1: FTS5 UPDATE 트리거 `messages_au` 추가
- § 7.6: `nabi_memory` MCP 서버 정의 → Phase 7+ deferred (별도 crate 없이 nabi-server subcommand `--mcp-memory-server`로 통합 예정). 활성화 전까지 Claude는 자체 도구로 `~/clawd/` 접근
- § 17: `chat_with_attachments` 타입 제거. § 17.6 신규 — 멀티모달은 `chat` envelope의 `attachments` 필드로 통일 (Phase 13+ deferred)
- § 21 Phase 5 acceptance: 60fps frame budget과 § 1.4 응답 latency 척도 분리
- § 22.10: uuid를 workspace deps에 명시 + 각 crate 참조 규약
- § 21 Phase 0: `prompting/caveman.md` → `config/prompting/caveman.md` 이동 task 추가 (git mv)
- § 0.1 신규: TOC + Phase별 cross-ref 가이드

### 0.3 (2026-05-16) — Claude CLI 2.1.142 실측 반영 + parser 정합성 + 비용 트랙 분리

**검증**: `claude --help` 및 hidden flag probe (`--max-turns`, `--permission-prompt-tool`, `--append-system-prompt-file` 실존 확인). v0.2의 § 4.3 CLI 명령 라인 그대로 유효.

**수정**:

- § 7.2: `PermissionMode` enum에 `Auto`, `DontAsk` 추가 (Claude CLI 2.1+ 지원)
- § 7.3 `permission_mode_str`: 두 variant 매핑 추가
- § 7.3 `list_models`: 하드코딩 가격 제거 → `None` + nabi.yaml runtime overlay 방식. stream `total_cost_usd`가 권위. claude-haiku-4-5 추가
- § 7.3 parser: `StreamParserState` 도입. `input_json_delta`의 tool_use_id는 `content_block_start`의 `index → id` 매핑으로 stateful 조회. `content_block_stop`에서 매핑 정리. v0.2 버그 (delta 자체에서 `id` 찾던 코드) 수정
- § 7.4: 응답 wrap canary 절차 추가. Phase 2c 진입 전 `{content:[{text:...}]}` wrap vs 평탄 응답 중 동작하는 형식 확정 필요
- § 7.7 신규: Routing 규칙 문법 (`<provider>/<model>` 첫 `/` split, OpenRouter 호환)
- § 8.7: 비용 트랙 A (구독 크레딧) / 트랙 B (API key 직과금) 명시 분리. `daily_cap_usd`는 트랙 B의 hard cap, 트랙 A에서는 soft 알림

**v0.3 후속 패치 (같은 세션 내)**:

- § 2.4: 실측 경로로 정정. `~/nabi/`는 부재, memory db는 `~/clawd/memory-db/memory-sync.db`. claude-telegram-bridge 인벤토리 추가 (port 8787, PM2 4개, Cloudflare Tunnel `bamsigi_tunnel`)
- § 7.6 nabi.yaml `memory:` 블록 경로 정정
- § 10.8 결정 완료: 기존 서비스 = `~/Projects/claude-telegram-bridge`, 아카이빙 확정 (사용자 확인)
- § 4.6 신규: 기본 출력 스타일 = caveman mode (출력 토큰만 단축, thinking 보존). canonical snippet `prompting/caveman.md` 추가
- § 4.2: workspace layout에 `config/prompting/caveman.md` 추가
- § 7.6: nabi.yaml `prompting:` 블록 + `context_budget.caveman: 250` 추가
- § 6 Phase 4: Context Builder layer 0 = caveman, escape trigger 평가, 인터페이스별 정책 검증

**v0.3 ultraplan 보강 (one-shot 개발 가능 수준으로 detail 추가)**:

- § 15 신규: nabi.db 전체 스키마 (devices / sessions / messages + FTS / permission_requests / cost_ledger / push_subscriptions / telegram_chats / audit_log / raw_events / context_manifests / memory_events). V0001__initial.sql 한 곳에 모음
- § 16 신규: `NabiError` thiserror 기반 enum 전체 정의 + From 구현
- § 17 신규: WebSocket 프로토콜 (v=1 envelope, client→server / server→client 메시지 전체, 재접속 / 하트비트 / 백필 / 에러 코드 enum)
- § 18 신규: SKILL.md / MEMORY.md / Daily / Topic 마크다운 frontmatter 스펙
- § 19 신규: refinery 기반 migration 시스템 (forward-only, PRAGMA setup)
- § 20 신규: `.github/workflows/ci.yml` 구체 yaml (fmt/clippy/build/test/audit + macOS canary 분리)
- § 21 신규: Phase Acceptance Matrix — Phase -1 ~ Phase 12 모두 구체 명령 / exit 0 / grep 조건
- § 22 신규: 9개 crate 각각의 Cargo.toml [dependencies] (workspace deps 활용), workspace deps 추가 항목 patch
- § 23 신규: 결정 트리 (CLI spec 차이 / provider 추가 / schema 변경 / 비용 폭주)
- § 24 신규: 코딩 규칙 (async / error / channel / DB / 비밀 관리)

**미수정 / 후속**:

- § 3.1 pmset 모드 통일 (`-a` vs `-c`) — Phase -1 스크립트 시점 실제 운영 모드 확인 후 반영
- § 3.2 MCP 서버명 표기 (`nabi-auth` vs `nabi_auth`) — Phase 2c canary에서 어느 게 실 effective인지 확인 후 통일
- § 10.10 / § 10.11 — 사용자 결정 대기

### 0.2 (2026-05-15) — GPT 분석 반영 + MacBook primary 운영 보강

**검증 후 수용한 주요 변경**:

- § 2.3: `--bare` 정책 명시 (subscription 금지, api_key only)
- § 3.2: Claude CLI provider 모드 2개로 분리 (subscription / api_key)
- § 3.4: Server primary = MacBook M4 확정
- § 3.6: 라이선스 Private/Proprietary 확정
- § 3.7: SQLite single-writer 전략 명시
- § 3.8: nabi-rs 역할 재정의 (supervisor / parser / bridge / injector)
- § 4.3: 실제 CLI 명령 라인 재작성 (`--tools`, `--disallowedTools`, `--permission-prompt-tool`, `--max-budget-usd`, `--max-turns`, `--settings`, `--strict-mcp-config` 등 추가, `--bare` 조건부)
- § 4.4: 권한 요청 흐름 MCP 기반으로 재설계
- § 4.5: TUI auth 흐름 추가 (`cloudflared access login` + Keychain)
- § 6: Phase -1 신설 (사전 작업), Phase 2를 2a/2b/2c/2d로 분할, Phase 8c (cutover) 분리
- § 7.2: ChatRequest 필드 분리 (tools / allowed_tools / disallowed_tools 등), McpConfigFile top-level `mcpServers`
- § 7.3: claude_cli.rs 전면 재작성 (AuthMode 분기, stream_event 기반 parser, stderr drain, unknown raw 처리)
- § 7.4: permission_mcp.rs 신규 (MCP stdio 서버)
- § 7.5: `claude-settings.json` 신규 (deny/ask/allow + sandbox)
- § 7.6: nabi.yaml 전면 개정 (모드 분리, Keychain source, context_budget, macos 정책)
- § 8.5: macOS 운영 정책 섹션 신규 (sleep / 자동 업데이트 / Time Machine / Spotlight / 자동 시작 / firewall)
- § 8.8: PWA 업데이트 흐름 (service worker cache versioning)
- § 9.3: Claude CLI 시크릿 노출 방지 다층 방어
- § 9.4: Secret Canary Test 신규
- § 9.5: raw_events / context_manifests 테이블 추가
- § 10: Open Decisions 정리 (10.1~10.7 확정, 10.8~10.11 추가)
- § 11: 리스크 11.7, 11.8 추가
- § 13.1: Phase -1 자동화 스크립트 추가
- § 1.4: 성공 지표 재설정 (RSS / P50 / P95 분리)

**검증 미수용 / 부분 수용**:

- StreamingDialect / ToolDialect / CostModel 추가는 over-engineering 가능 → Phase 6 실측 후 결정
- Windows backup: Phase 12로 미루기 (변경 없음)

### 0.1 (2026-05-15) — 초기 작성

Phase 0-12 정의, 아키텍처 확정, Open Decisions 7개 식별.

---

---

## 15. nabi.db 전체 스키마 (v0.3 신규 / ultraplan 보강)

§ 9.5의 audit_log / raw_events / context_manifests를 포함해 전체 테이블을 한 곳에 모음. `migrations/V0001__initial.sql`이 정확한 source of truth.

### 15.1 V0001__initial.sql

```sql
-- WAL + busy_timeout은 connection open 시 PRAGMA로 (§ 19)

CREATE TABLE devices (
    id TEXT PRIMARY KEY,                    -- uuid v4
    name TEXT NOT NULL,                     -- "home-mac-tui", "iphone-pwa"
    kind TEXT NOT NULL,                     -- tui | pwa | telegram | service
    fingerprint TEXT,
    first_seen_at TEXT NOT NULL,
    last_seen_at TEXT NOT NULL,
    revoked_at TEXT
);

CREATE TABLE sessions (
    id TEXT PRIMARY KEY,                    -- nabi 세션 uuid
    claude_session_id TEXT,                 -- ~/.claude/projects/ id (provider별)
    provider_name TEXT NOT NULL,
    model TEXT NOT NULL,
    title TEXT,                             -- 자동 생성 (보조 모델)
    source TEXT NOT NULL,                   -- 'web' | 'tui' | 'telegram' | 'imported'
    archived INTEGER NOT NULL DEFAULT 0,
    pinned INTEGER NOT NULL DEFAULT 0,
    cost_usd_total REAL NOT NULL DEFAULT 0,
    tokens_in_total INTEGER NOT NULL DEFAULT 0,
    tokens_out_total INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sessions_updated ON sessions(updated_at DESC);
CREATE INDEX idx_sessions_archived ON sessions(archived);
CREATE INDEX idx_sessions_claude_sid ON sessions(claude_session_id);

CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    seq INTEGER NOT NULL,
    role TEXT NOT NULL,                     -- user | assistant | tool  (system은 meta prompt, 저장 안 함 — § 7.2 Message enum과 일치)
    content_preview TEXT,                   -- 첫 200자
    tokens_in INTEGER,
    tokens_out INTEGER,
    cost_usd REAL,
    invocation_id TEXT,                     -- v0.3.1 — Provider::chat() 진입 시 생성한 uuid, cost_ledger / raw_events / context_manifests / audit_log 공통 join key
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(session_id, seq)
);
CREATE INDEX idx_messages_session ON messages(session_id, seq);
CREATE INDEX idx_messages_invocation ON messages(invocation_id);

-- FTS는 content_preview만 인덱싱 (full content는 ~/.claude/projects/ jsonl이 ground truth)
CREATE VIRTUAL TABLE messages_fts USING fts5(
    session_id UNINDEXED,
    role UNINDEXED,
    content,
    seq UNINDEXED,
    content='messages',
    content_rowid='id',
    tokenize='unicode61 remove_diacritics 2'
);

CREATE TRIGGER messages_ai AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, session_id, role, content, seq)
    VALUES (new.id, new.session_id, new.role, new.content_preview, new.seq);
END;

CREATE TRIGGER messages_ad AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, session_id, role, content, seq)
    VALUES('delete', old.id, old.session_id, old.role, old.content_preview, old.seq);
END;

-- v0.3.1 — content_preview update 시 FTS도 갱신
CREATE TRIGGER messages_au AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, session_id, role, content, seq)
    VALUES('delete', old.id, old.session_id, old.role, old.content_preview, old.seq);
    INSERT INTO messages_fts(rowid, session_id, role, content, seq)
    VALUES (new.id, new.session_id, new.role, new.content_preview, new.seq);
END;

CREATE TABLE permission_requests (
    id TEXT PRIMARY KEY,                    -- uuid
    session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    tool_use_id TEXT,
    tool_name TEXT NOT NULL,
    input JSON NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending', -- pending | allowed | denied | timeout | superseded
    decided_by_device TEXT REFERENCES devices(id),
    decided_at TEXT,
    response_input JSON,                    -- updatedInput (allow 시)
    reason TEXT,                            -- deny 사유
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_perm_session ON permission_requests(session_id);
CREATE INDEX idx_perm_status ON permission_requests(status, created_at);

CREATE TABLE cost_ledger (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT REFERENCES sessions(id) ON DELETE SET NULL,
    provider_name TEXT NOT NULL,
    track TEXT NOT NULL,                    -- 'subscription' | 'api_key'
    model TEXT NOT NULL,
    tokens_in INTEGER,
    tokens_out INTEGER,
    tokens_cache_read INTEGER,
    tokens_cache_write INTEGER,
    cost_usd REAL NOT NULL,
    invocation_id TEXT,
    occurred_at TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_cost_occurred ON cost_ledger(occurred_at);
CREATE INDEX idx_cost_track ON cost_ledger(track, occurred_at);
CREATE INDEX idx_cost_session ON cost_ledger(session_id);

CREATE TABLE push_subscriptions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id TEXT NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    endpoint TEXT NOT NULL UNIQUE,
    p256dh TEXT NOT NULL,
    auth TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_used_at TEXT,
    failures INTEGER NOT NULL DEFAULT 0      -- 4xx 5+ → 자동 해제
);

CREATE TABLE telegram_chats (
    chat_id INTEGER PRIMARY KEY,
    title TEXT,
    allowed INTEGER NOT NULL DEFAULT 0,
    active_session_id TEXT REFERENCES sessions(id) ON DELETE SET NULL,
    last_msg_at TEXT
);

CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    session_id TEXT,
    event_type TEXT NOT NULL,
    actor TEXT,                              -- device_id 또는 'agent'
    details JSON NOT NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_audit_session ON audit_log(session_id);
CREATE INDEX idx_audit_type ON audit_log(event_type);
CREATE INDEX idx_audit_time ON audit_log(timestamp);

CREATE TABLE raw_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    provider TEXT,
    raw JSON NOT NULL,
    received_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_raw_session ON raw_events(session_id);

CREATE TABLE context_manifests (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    message_seq INTEGER NOT NULL,
    caveman_active INTEGER NOT NULL DEFAULT 1,
    skills_loaded JSON,
    memory_entries JSON,
    wiki_pages JSON,
    token_estimate INTEGER,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_manifest_session ON context_manifests(session_id, message_seq);

-- memory.db (read-only) shadow events. Phase 4부터 write 시작.
CREATE TABLE memory_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type TEXT NOT NULL,                -- 'add' | 'update' | 'tag'
    memory_db_id INTEGER,                    -- legacy memory.db 참조 (있으면)
    content TEXT NOT NULL,
    category TEXT,
    tags JSON,
    source_session_id TEXT REFERENCES sessions(id) ON DELETE SET NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_memev_session ON memory_events(source_session_id);
```

### 15.2 후속 마이그레이션 예약 슬롯

- V0002: Phase 6 — provider state 캐시 테이블 (모델 가격 / capabilities 동적 overlay)
- V0003: Phase 10 — push notification 로그 / VAPID 키 회전 기록
- V0004: Phase 12 — health metrics snapshot 테이블 (선택)

### 15.2.1 invocation_id 규약 (v0.3.1 / 0.3.2 정합화)

- 생성: `Provider::chat()` 진입 시점에 `uuid::Uuid::new_v4()` 1회 생성
- 형식: 표준 UUID v4 문자열 (hyphenated)
- 전파 메커니즘 (v0.3.2 명시):
  - Provider가 `StreamParserState.invocation_id`에 stash
  - 첫 `StreamEvent::SessionStarted { invocation_id, .. }` 이벤트에 실어 yield
  - downstream consumer (nabi-server)가 SessionStarted에서 invocation_id를 stash → 이후 같은 stream의 모든 이벤트를 그 값으로 태깅
  - StreamEvent variant 다수에 invocation_id 필드를 추가하지 않음 (envelope 비대화 방지)
- join key 역할:
  - `messages.invocation_id` — 어시스턴트 응답 메시지가 어느 invocation 산출인지
  - `cost_ledger.invocation_id` — 비용 ledger 행
  - `raw_events` — `session_id` + 타임스탬프 범위로 묶지만, 필요 시 invocation_id 컬럼 V0002에서 추가
  - `context_manifests` — V0002에서 invocation_id 컬럼 추가, 1:1 매핑
  - `audit_log.details` JSON 내 `invocation_id` 필드
- 사용 시점: 동일 nabi 세션 내 여러 invocation을 분리 분석 / 비용 추적 / 재현용

V0001은 messages.invocation_id만 포함, raw_events / context_manifests에는 V0002에서 컬럼 추가.

### 15.3 데이터 보존 정책

- `raw_events`, `audit_log`: 90일 이후 일별 archive (월별 sqlite 파일 분리)
- `cost_ledger`: 무기한 보존 (월별 요약은 view로)
- `messages`, `permission_requests`: 무기한, 단 사용자 명시적 삭제 시 cascade

---

## 16. NabiError 타입 (v0.3 신규)

`nabi-core/src/error.rs`:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum NabiError {
    #[error("io: {0}")]
    Io(String),

    #[error("parse: {0}")]
    Parse(String),

    #[error("subprocess spawn failed: {0}")]
    Spawn(String),

    #[error("subprocess died unexpectedly: exit={0}")]
    SubprocessExit(i32),

    #[error("auth failed: {0}")]
    Auth(String),

    #[error("permission denied: {tool} — {reason}")]
    PermissionDenied { tool: String, reason: String },

    #[error("permission timeout: {request_id}")]
    PermissionTimeout { request_id: String },

    #[error("provider error: {provider} — {message}")]
    Provider { provider: String, message: String },

    #[error("provider not found: {0}")]
    ProviderNotFound(String),

    #[error("db: {0}")]
    Db(String),

    #[error("mcp: {0}")]
    Mcp(String),

    #[error("budget exceeded: invocation=${cost_usd:.4} > ${limit:.2}")]
    BudgetExceeded { cost_usd: f64, limit: f64 },

    #[error("daily cap exceeded: track={track}, used=${used:.2}, cap=${cap:.2}")]
    DailyCapExceeded { track: String, used: f64, cap: f64 },

    #[error("cancelled by user")]
    Cancelled,

    #[error("config: {0}")]
    Config(String),

    #[error("internal: {0}")]
    Internal(String),
}

impl From<std::io::Error> for NabiError {
    fn from(e: std::io::Error) -> Self { NabiError::Io(e.to_string()) }
}
impl From<rusqlite::Error> for NabiError {
    fn from(e: rusqlite::Error) -> Self { NabiError::Db(e.to_string()) }
}
impl From<serde_json::Error> for NabiError {
    fn from(e: serde_json::Error) -> Self { NabiError::Parse(format!("json: {}", e)) }
}
impl From<reqwest::Error> for NabiError {
    fn from(e: reqwest::Error) -> Self { NabiError::Provider { provider: "http".into(), message: e.to_string() } }
}
```

운영 규칙: anyhow는 main / binary 진입점에서만, 라이브러리는 NabiError 사용.

---

## 17. WebSocket 프로토콜 (v0.3 신규)

### 17.1 Envelope

모든 메시지: `{"v":1,"type":"...", ...}`

- `v`: protocol 버전, 현재 1. breaking change 시 +1
- `type`: 메시지 종류
- 선택 `id`: ack/reply 매칭용
- 선택 `sid`: 세션 uuid

### 17.2 Client → Server

```json
{"v":1,"type":"chat","id":"c-1","sid":"<sid>","content":"안녕"}
{"v":1,"type":"cancel","sid":"<sid>"}
{"v":1,"type":"permission_response","request_id":"<rid>","allow":true,"updated_input":null}
{"v":1,"type":"permission_response","request_id":"<rid>","allow":false,"reason":"too dangerous"}
{"v":1,"type":"session_new","provider":"claude_subscription","model":"sonnet","source":"pwa"}
{"v":1,"type":"session_list","limit":50,"archived":false,"cursor":null}
{"v":1,"type":"session_subscribe","sid":"<sid>"}
{"v":1,"type":"session_unsubscribe","sid":"<sid>"}
{"v":1,"type":"session_archive","sid":"<sid>","archived":true}
{"v":1,"type":"ping","t":1735000000}
```

### 17.3 Server → Client

```json
{"v":1,"type":"session_started","sid":"<sid>","claude_sid":"<csid>","model":"sonnet","tools":["Read","Bash"]}
{"v":1,"type":"text_delta","sid":"<sid>","content":"..."}
{"v":1,"type":"tool_use_start","sid":"<sid>","tool_use_id":"<tid>","name":"Bash"}
{"v":1,"type":"tool_use_input","sid":"<sid>","tool_use_id":"<tid>","partial_json":"..."}
{"v":1,"type":"tool_result","sid":"<sid>","tool_use_id":"<tid>","output":"...","is_error":false}
{"v":1,"type":"permission_request","sid":"<sid>","request_id":"<rid>","tool":"Bash","input":{"command":"rm ..."},"timeout_sec":300}
{"v":1,"type":"permission_already_decided","request_id":"<rid>","decided_by_device":"<did>","allowed":true}
{"v":1,"type":"api_retry","sid":"<sid>","attempt":1,"reason":"overloaded"}
{"v":1,"type":"mcp_failure","sid":"<sid>","server":"nabi_auth","error":"..."}
{"v":1,"type":"done","sid":"<sid>","cost_usd":0.043,"tokens_in":1234,"tokens_out":567,"cache_read":0,"cache_write":2000}
{"v":1,"type":"error","sid":"<sid>","code":"budget_exceeded","message":"..."}
{"v":1,"type":"heartbeat","t":1735000000}
{"v":1,"type":"pong","t":1735000000}
{"v":1,"type":"session_list_reply","sessions":[{"id":"<sid>","title":"...","updated_at":"...","cost_usd":0.5,"archived":false}],"next_cursor":null}
{"v":1,"type":"cost_alert","track":"api_key","used":24.50,"cap":30.00,"threshold":0.80}
```

### 17.4 재접속 / 하트비트 / 백필

- 클라이언트: 25초마다 `ping`. 60초간 server 무응답 → 재접속
- 서버: 30초마다 모든 구독자에 `heartbeat`
- 재접속 후: `session_subscribe`로 진행 중 세션 stream 재구독. 누락 토큰은 별도 REST `GET /sessions/{sid}/messages?since_seq=N`로 backfill
- 진행 중 invocation 도중 클라이언트 끊김 → 서버는 계속 실행 (Claude CLI subprocess 유지). 응답은 nabi.db에 저장. 재접속 시 누락분 backfill
- 권한 요청은 끊긴 디바이스 외 모두에게 전파. 어느 디바이스가 먼저 결정하면 나머지에 `permission_already_decided`

### 17.5 에러 코드 enum

`code` 필드 표준 값:

- `budget_exceeded`, `daily_cap_exceeded`, `subprocess_died`, `provider_unavailable`, `auth_failed`, `permission_timeout`, `session_not_found`, `rate_limited`, `internal`

### 17.6 멀티모달 (v0.3.1 — Phase 13+ deferred)

이미지 / 파일 첨부는 현재 out of scope. Phase 13+에서 도입 시 envelope 후보:

```json
{"v":1,"type":"chat","id":"c-1","sid":"<sid>","content":"...","attachments":[{"kind":"image","data_url":"..."}]}
```

이전 v0.3 draft에 `chat_with_attachments` 타입이 있었으나, type 분리 대신 `chat` envelope에 `attachments` 필드 옵션 추가하는 방식으로 통일. ChatRequest (§ 7.2)에도 동시 추가 필요. 도입 전까지 모든 attachments 필드 거부.

---

## 18. Skill / Wiki / Daily Log 포맷 (v0.3 신규)

### 18.1 SKILL.md (`config/skills/*.md` 및 `~/clawd/skills/*.md`)

```markdown
---
name: tendril
description: 텐드릴 페이스 API 클라이언트 사용법
triggers: [tendril, 텐드릴, tendril-api]
tags: [api, 텐드릴]
priority: 1
max_tokens: 2000
---

# 텐드릴 페이스 API

OAuth: POST /api/auth/token
...
```

frontmatter 스키마:
- `name` (필수, kebab-case, 파일명과 일치)
- `description` (필수, 1줄)
- `triggers` (배열, 부분 매치 — 사용자 메시지에 단어 등장 시 자동 로드)
- `tags` (배열, cross-link용)
- `priority` (정수, 낮을수록 우선. 다중 매치 시 정렬 + 토큰 예산 내 절단)
- `max_tokens` (정수, 본문 토큰 cap)

파서: `nabi-skills::loader`. yaml frontmatter는 `serde_yaml`, 본문은 그대로 문자열.

### 18.2 MEMORY.md hub (`~/clawd/MEMORY.md`)

```markdown
# MEMORY hub

## Active topics
- [tendril](topics/tendril.md) — 텐드릴 통합
- [tistory](topics/tistory.md) — 블로그 자동화

## Daily
- [2026-05-16](daily/2026-05-16.md)
- [2026-05-15](daily/2026-05-15.md)

## Index
[전체 색인](index.md)
```

파서: 마크다운 링크 추출 → topic / daily 분류. Context Builder가 최신 항목 우선 포함, 토큰 예산 내 절단.

### 18.3 Daily Log (`~/clawd/daily/YYYY-MM-DD.md`)

```markdown
---
date: 2026-05-16
tags: [nabi-rs, planning]
---

# 2026-05-16

## 작업
- nabi-rs handoff v0.3 작성

## 결정
- 기존 서비스 아카이빙 (§ 10.8)

## 학습
- caveman plugin = 출력 토큰만 단축
```

frontmatter 없어도 동작. 파서는 첫 `# YYYY-MM-DD` 또는 frontmatter `date` 둘 중 하나로 날짜 결정.

### 18.4 Topic Note (`~/clawd/topics/*.md`)

자유 양식. frontmatter 권장:

```markdown
---
title: 텐드릴 통합
status: active        # active | dormant | archived
last_updated: 2026-05-16
---
```

---

## 19. Migration 시스템 (v0.3 신규)

- crate: `refinery` (Rust 친화, runtime 적용, forward-only)
- 위치: `migrations/V0001__initial.sql`, `V0002__...sql`
- 적용: `nabi-server` boot 시점 (memory writer actor 시작 전)
- 명명: `V<4자리>__<snake_case_description>.sql`
- 적용 이력: `refinery_schema_history` 테이블 자동 생성
- rollback: forward-only. 문제 시 백업 (Litestream R2)에서 복원
- connection 초기화 PRAGMA: `journal_mode=WAL`, `busy_timeout=5000`, `synchronous=NORMAL`, `foreign_keys=ON`

```rust
// nabi-memory/src/migrations.rs
mod embedded {
    use refinery::embed_migrations;
    embed_migrations!("../../migrations");
}

pub fn run(conn: &mut rusqlite::Connection) -> Result<(), NabiError> {
    embedded::migrations::runner()
        .run(conn)
        .map_err(|e| NabiError::Db(format!("migration: {}", e)))?;
    Ok(())
}

pub fn setup_pragmas(conn: &rusqlite::Connection) -> Result<(), NabiError> {
    conn.execute_batch("
        PRAGMA journal_mode=WAL;
        PRAGMA busy_timeout=5000;
        PRAGMA synchronous=NORMAL;
        PRAGMA foreign_keys=ON;
    ")?;
    Ok(())
}
```

memory-sync.db는 read-only이므로 migration 적용 안 함. PRAGMA `query_only=ON`으로 connection 열기.

---

## 20. CI Workflow (v0.3 신규)

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  rust:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
      - name: fmt
        run: cargo fmt --all -- --check
      - name: clippy
        run: cargo clippy --all-targets --all-features -- -D warnings
      - name: build
        run: cargo build --workspace --all-features
      - name: test
        run: cargo test --workspace --all-features
      - name: audit
        run: |
          cargo install cargo-audit --locked
          cargo audit --deny warnings
```

Phase 9.4 secret canary는 macOS runner 필요 (claude CLI 의존). 별도 nightly job으로 분리:

```yaml
  canary-macos:
    runs-on: macos-latest
    if: github.event_name == 'schedule'
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - name: install claude
        run: |
          curl -fsSL https://claude.ai/install.sh | bash
      - name: secret canary
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_CANARY_KEY }}
        run: ./scripts/secret-canary.sh
```

cron schedule (`on: schedule: - cron: "0 18 * * *"` = 한국 시간 03:00) 추가.

---

## 21. Phase Acceptance Matrix (v0.3 신규)

각 Phase 머지 전 통과해야 할 구체 명령. 모두 exit 0 + grep 매치 시 머지 OK.

### Phase -1
- `./scripts/phase-minus-1.sh` exit 0
- `ls -d ~/backups/nabi-migration-*` 결과 1개 이상
- `pmset -g | grep -E '^\s*sleep\s+0\s*$'` 매치
- `lsof -i :9912` empty
- `claude -p "ping"` 응답 받음
- `lsof -i :8787` 매치 (claude-telegram-bridge 가동 중 확인)

### Phase 0
- `cargo build --workspace` exit 0
- `cargo run -p nabi-cli -- --version` 출력에 `nabi`
- `cargo fmt --all -- --check` exit 0
- `cargo clippy --all-targets -- -D warnings` exit 0
- `.github/workflows/ci.yml` last run = success
- **caveman.md 이동**: `prompting/caveman.md` → `config/prompting/caveman.md` (`git mv`로 이동, history 보존). nabi.yaml `caveman_snippet_path` 경로 일치 검증

### Phase 1
- `cargo test -p nabi-core --lib` exit 0
- `cargo run -p nabi-cli -- chat --provider mock "echo: hi"` stdout에 "echo: hi"
- Provider trait + Capabilities + StreamEvent + Message + ChatRequest 컴파일

### Phase 2a
- `tests/fixtures/stream-json/` 파일 6개 모두 존재 (plain, tool-use, permission-abort, api-retry, mcp-failure, session-resume)
- `tests/fixtures/auth-matrix.md` 4셀 결과 기록
- `cargo test -p nabi-providers --test claude_cli_protocol` exit 0
- first-token P50 / P95 측정 결과 `tests/fixtures/latency-baseline.md` 기록

### Phase 2b
- `cargo test -p nabi-providers` exit 0
- 실측: `nabi chat --provider claude_subscription --model sonnet "say 'hi'"` 응답
- 세션 재개: `nabi chat --sid <SID> --provider ... "continue"` 동작
- SIGTERM → 5초 내 graceful exit, child orphan 없음

### Phase 2c
- `nabi-server --mcp-permission-server` stdin/stdout MCP 응답 (initialize / tools/list / tools/call)
- canary 통과: `claude -p "rm -rf /tmp/canary" --permission-prompt-tool mcp__nabi_auth__approve --mcp-config ...` → 권한 prompt → deny → 실행 안 됨
- 응답 wrap 형태 (`text` vs 평탄) 결과 문서화 (§ 7.4 canary)
- timeout 동작 (5분 초과 시 deny 반환)

### Phase 2d
- 모든 fixture parse 성공
- unknown 이벤트 → `Unknown { raw }` + raw_events 테이블 행
- SIGINT → 5초 내 graceful, stream 정리

### Phase 3
- `nabi-memory` integration test exit 0
- FTS 검색 결과: nabi-rs vs OpenClaw 동일 (`node ~/clawd/memory-db/search.js "..."`)
- `~/clawd/memory-db/memory-sync.db` mtime 변동 없음 (read-only 검증)
- migration: `nabi.db` 새로 생성 시 V0001 적용 + 모든 테이블 존재
- skill loader: `triggers` 부분 매치 단위 테스트

### Phase 4
- caveman 활성 시 평균 응답 토큰 / 비활성 시 토큰 비율 < 0.6
- "자세히 설명해" trigger → caveman OFF 검증
- Telegram interface → caveman OFF 유지
- 권한 요청 메시지 → caveman OFF
- `context_manifests` 테이블에 매 invocation 행 추가

### Phase 5
- ratatui 키바인딩 6종 (ctrl+c, ctrl+n, ctrl+r, F2, esc, enter) 동작
- partial token 시각화 (frame budget < 16ms = 60fps render — 별도 척도)
- 세션 picker: `~/.claude/projects/` + `nabi.db` 둘 다 표시, 충돌 시 nabi.db 우선
- 비용 누계 UI 표시
- **§ 1.4 응답 latency: first-stream-event P50 < 2s, P95 < 5s** (Claude CLI subscription mode 실측)
- mock / OpenRouter / Ollama: first-stream-event P50 < 1s

### Phase 6
- OpenRouter / Ollama 각각 응답 받음 (mock 아닌 실 HTTP)
- F2로 provider/model 전환 → 즉시 적용
- Routing rule `<provider>/<model>` 파싱 unit test
- 라우팅 의도: code_heavy → opus, quick_chat → glm, sensitive → ollama

### Phase 7
- `pm2 start ecosystem-nabi.config.cjs` → online
- `kill -TERM` → 5초 내 종료, DB WAL checkpoint 완료
- `lsof -i :9912` → nabi-server만 listen, 127.0.0.1 bind
- 외부 머신에서 `<MAC_IP>:9912` 직접 접근 거부
- 재부팅 후 자동 가동 (수동 검증, `pm2 list` online)

### Phase 8 + 8c
- CF Access Application 생성 + AUD 캡처
- 회사 노트북 / iPhone Safari 둘 다 접속 OK (Google OAuth)
- `cloudflared tunnel ingress validate` exit 0
- TUI: `nabi-tui auth login` → Keychain `nabi-tui-cf-token` 항목 생성 → WS upgrade 정상
- Cutover 다운타임 측정 < 10초
- nabi-new와 nabi 병행 1주 검증 후 cutover (가능 시간 § 10.11 확정 후)

### Phase 9 (PWA)
- iOS Safari 홈화면 추가 → 오프라인 첫 화면 로드
- WS 연결 → 메시지 전송 → 응답 → 세션 재개
- 권한 모달 → allow/deny 동작
- service worker 캐시 버전업 → 새로고침 토스트

### Phase 10 (Push)
- VAPID 키 생성 + Keychain 저장
- push 구독 DB 저장
- 폰 background 상태에서 push 수신 + 탭 시 PWA 열림
- 4xx 5회 누적 시 자동 구독 해제

### Phase 11 (Telegram)
- 봇 토큰 Keychain + chat_id allow list 적용
- 메시지 → 응답 받기
- destructive permission default deny (Telegram 경로)
- inline keyboard allow/deny 동작

### Phase 12 (Failover)
- Litestream R2 백업 가동 (`/etc/litestream.yml`)
- 별도 머신에서 백업 복원 → nabi-server 가동 OK
- `/healthz` 200 + `/metrics` Prometheus format
- 일일 비용 텔레그램 리포트 자동 전송
- 디스크 사용량 알림 (`~/.claude/projects/`)

---

## 22. Per-Crate Cargo.toml (v0.3 신규)

각 crate `Cargo.toml`의 `[dependencies]` 핵심 (workspace deps 활용).

### 22.1 nabi-core
```toml
[dependencies]
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
async-trait = { workspace = true }
thiserror = { workspace = true }
futures = { workspace = true }
chrono = { workspace = true }
async-stream = { workspace = true }
```

### 22.2 nabi-providers
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
tokio = { workspace = true, features = ["process", "io-util", "macros", "sync"] }
tokio-stream = { workspace = true }
async-stream = { workspace = true }
reqwest = { workspace = true }
eventsource-stream = "0.2"
serde = { workspace = true }
serde_json = { workspace = true }
async-trait = { workspace = true }
tracing = { workspace = true }
futures = { workspace = true }
```

### 22.3 nabi-memory
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
rusqlite = { workspace = true }
r2d2 = { workspace = true }
r2d2_sqlite = { workspace = true }
refinery = { version = "0.8", features = ["rusqlite"] }
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }
tokio = { workspace = true, features = ["sync", "rt"] }
tracing = { workspace = true }

[build-dependencies]
refinery = { version = "0.8", features = ["rusqlite"] }
```

### 22.4 nabi-skills
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
serde = { workspace = true }
serde_yaml = { workspace = true }
walkdir = "2"
regex = "1"
tracing = { workspace = true }
```

### 22.5 nabi-context
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
nabi-skills = { path = "../nabi-skills" }
nabi-memory = { path = "../nabi-memory" }
tokio = { workspace = true, features = ["fs"] }
tiktoken-rs = "0.6"
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }
tracing = { workspace = true }
```

### 22.6 nabi-server
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
nabi-providers = { path = "../nabi-providers" }
nabi-memory = { path = "../nabi-memory" }
nabi-skills = { path = "../nabi-skills" }
nabi-context = { path = "../nabi-context" }
axum = { workspace = true }
tower = { workspace = true }
tower-http = { workspace = true }
jsonwebtoken = { workspace = true }
reqwest = { workspace = true }
keyring = { workspace = true }
teloxide = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
tracing-appender = { workspace = true }
anyhow = { workspace = true }
clap = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
serde_yaml = { workspace = true }
chrono = { workspace = true }
uuid = { version = "1", features = ["v4", "serde"] }
async-trait = { workspace = true }
```

### 22.7 nabi-tui
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
ratatui = { workspace = true }
crossterm = { workspace = true }
tokio = { workspace = true }
tokio-tungstenite = "0.21"
reqwest = { workspace = true }
keyring = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
clap = { workspace = true }
tracing = { workspace = true }
anyhow = { workspace = true }
```

### 22.8 nabi-web
```toml
[dependencies]
axum = { workspace = true }
tower-http = { workspace = true, features = ["fs", "trace"] }
tokio = { workspace = true }
tracing = { workspace = true }
clap = { workspace = true }
anyhow = { workspace = true }
```

### 22.9 nabi-cli
```toml
[dependencies]
nabi-core = { path = "../nabi-core" }
nabi-providers = { path = "../nabi-providers" }
nabi-memory = { path = "../nabi-memory" }
clap = { workspace = true }
tokio = { workspace = true }
anyhow = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
serde_json = { workspace = true }
```

### 22.10 workspace deps 추가 필요 항목 (§ 13.2 patch)

```toml
walkdir = "2"
regex = "1"
uuid = { version = "1", features = ["v4", "serde"] }   # nabi-server, nabi-memory 양쪽 사용
tiktoken-rs = "0.6"
tokio-tungstenite = "0.21"
eventsource-stream = "0.2"
refinery = { version = "0.8", features = ["rusqlite"] }
```

각 crate `Cargo.toml`에서는 `uuid = { workspace = true }` 형태로 참조.

---

## 23. 결정 트리 (v0.3 ultraplan)

각 Phase에서 막힐 때 결정 순서:

### 23.1 Claude CLI 동작이 spec과 다를 때
1. § 7.3 fixture 갱신 → golden test 추가
2. § 11.7 unknown event raw 저장으로 살아남기
3. spec과 안 맞으면 § 4.3 CLI 명령 수정 + version pin

### 23.2 provider 추가 시
1. § 22 deps 추가
2. § 7.6 nabi.yaml 신규 entry
3. Capabilities (§ 7.1) 채움
4. Phase 6 routing 규칙에 매핑

### 23.3 schema 변경 시
1. `migrations/V<next>__<desc>.sql` 추가 (forward-only)
2. 영향받는 crate 컴파일 에러 → 새 컬럼 처리
3. Phase 12 백업 복원 시나리오 검증

### 23.4 비용 폭주 감지 시
1. invocation cap (`--max-budget-usd`) 확인
2. 트랙 B daily cap 도달 → routing이 트랙 A로 fallback
3. context manifest로 어디서 토큰 쓰는지 추적
4. 라우팅 규칙으로 잡일 모델 강제

---

## 24. 운영 시 준수할 코딩 규칙 (v0.3)

- async fn은 `&self` 또는 `&mut self`만, `self` consume 지양 (재사용)
- error는 `?` + `From` 변환. `.unwrap()`은 binary main / test에서만
- `tokio::spawn`은 명시 join handle 보관. fire-and-forget 금지 (드롭 시 task 취소됨)
- channel은 `tokio::sync::mpsc` (다대일), `broadcast` (일대다), `oneshot` (응답)
- subprocess는 항상 `kill_on_drop(true)` + stderr drain task
- DB writer는 single actor. read는 r2d2 pool
- JSON 파싱은 `serde_json::Value`로 받은 뒤 도메인 타입으로 변환 (unknown 살아남기)
- 모든 외부 입력은 boundary에서 검증, 내부는 trust
- 비밀 (token, api key)은 Keychain → 환경변수 → 파일 순. 평문 로그 금지

---

**문서 끝.**

다음 액션:
1. Open Decisions § 10.10 (자동 로그인), § 10.11 (다운타임 윈도우) 확정
2. Phase -1 시작 (`scripts/phase-minus-1.sh` 실행)
3. v0.3 후속 점검: pmset 모드 통일 (§ 3.1), MCP 서버명 통일 (§ 3.2), Permission MCP 응답 wrap canary (§ 7.4) — Phase 2a/2c에서 실측 반영
4. CLI agent에게 위탁 가능 단위: Phase 0 (§ 22 deps + § 20 CI), Phase 1 (§ 16 Error + § 17 ws envelope mock 통합), Phase 2a (§ 21 acceptance 매트릭스 기반)
3. Phase 0 워크스페이스 셋업
