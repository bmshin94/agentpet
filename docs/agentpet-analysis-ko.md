# AgentPet 전수조사 분석 정리 (한국어)

> 작성일: 2026-10-01
> 작성: Claude Code 세션 분석 결과 정리
> 분석 대상 커밋: `7bf1485` (main)

---

## 🔗 GitHub 및 관련 주소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/agentpet |
| **원본 저장소 (업스트림)** | https://github.com/ntd4996/agentpet |
| 공식 웹사이트 | https://agentpet.thenightwatcher.online |
| 설치 페이지 | https://agentpet.thenightwatcher.online/install |
| 펫 갤러리 | https://agentpet.thenightwatcher.online/gallery |
| 리더보드 | https://agentpet.thenightwatcher.online/leaderboard |
| 컨트리뷰터 명예의 전당 | https://agentpet.thenightwatcher.online/contributors |
| Discord | https://discord.gg/kzFJKsZav |
| 릴리즈 | https://github.com/ntd4996/agentpet/releases |
| 펫 포맷 원조 (Petdex) | https://github.com/crafter-station/petdex |
| 커뮤니티 Linux 포팅 | https://github.com/tranhuuhuy297/agentpet-linux |
| 원작자 | https://github.com/ntd4996 (Nguyễn Thành Đạt) |
| 후원 | https://ko-fi.com/ntd4996 · https://buymeacoffee.com/ntd4996 |

---

## 1. 이게 뭐하는 프로젝트야?

### 한 줄 요약

**AgentPet** = 내 컴퓨터에서 돌아가는 **AI 코딩 에이전트들(Claude Code, Codex, Cursor 등)의 상태를
실시간으로 보여주는 데스크톱 모니터링 앱** + 그 작업량을 먹고 자라는 **다마고치 펫**.

- 라이선스: **MIT** (애플리케이션 코드만. 펫 아트는 제외)
- 플랫폼: macOS 13+ (네이티브 Swift) / Windows 10·11 / Linux (Tauri)
- 커밋 57개, 파일 426개, 약 66MB
- 주 기여자: ntd4996(44), Nguyễn Thành Đạt(5), Chun(4), bmshin94(2), 외

### 두 가지 정체성

1. **실용 도구**: 에이전트 여러 개를 동시에 돌릴 때 "누가 일하는 중 / 누가 끝났 / 누가 내 입력을
   기다리는지"를 메뉴바 하나로 파악
2. **게임**: 에이전트가 실제로 태운 토큰과 완료한 세션이 펫의 먹이가 되어 XP·레벨·진화·업적으로 이어짐

---

## 2. 폴더 전수조사 결과

| 폴더 | 정체 | 내용 |
|---|---|---|
| `Sources/AgentPetCore/` | **두뇌** (Swift 34개, 3,782줄) | 이벤트 모델, 훅 설치기, 토큰 집계, 펫 성장 로직. 순수 로직으로 분리되어 테스트 가능 |
| `Sources/App/` | **macOS 앱 UI** (Swift 약 60개) | SwiftUI 메뉴바 앱, 떠다니는 펫 창(`NSPanel`), 설정창, 말풍선, 네이티브 알림 |
| `Tests/` | **테스트 45개 파일** | 에이전트별 훅 페이로드 파싱, 펫 레벨 계산, 토큰 비용, 로컬라이즈 패리티 테스트 |
| `windows/src-tauri/` | **Windows·Linux 앱** (Rust 8개) | Tauri 2 기반. `hooks.rs`, `server.rs`, `statemap.rs`, `transcript.rs`로 macOS 로직 재구현 |
| `web/` | **커뮤니티 웹사이트** (Astro 6 + TS 69개) | 펫 갤러리, 리더보드, 프로필, 사용량 대시보드, 관리자. Cloudflare Workers + D1 |
| `landing/` | **구버전 랜딩** (Pages Functions) | 초기 펫 갤러리. `web/`로 대체된 레거시 |
| `cdn-proxy/worker.js` | **펫 미러링 CDN** | Petdex·OpenPets의 펫을 R2로 미러링해 자체 도메인에서 서빙 |
| `data/*.json` | **펫 카탈로그** | `merged-manifest.json` = 펫 **4,044마리** (Petdex 2,926 + OpenPets 1,118) |
| `Localizations/` | **다국어** | en / vi / zh-Hans / zh-Hant — **한국어 없음 (기여 기회)** |
| `docs/specs/` | **설계 문서** | 초기 설계서(베트남어) + **Arena 오토배틀러 설계서(미구현)** |
| `docs/superpowers/` | 계획·스펙 | 상태 감지 수정, 모델 말풍선 토큰 관련 |
| `.github/workflows/` | **CI/CD 5개** | ci(macOS 빌드·테스트), windows-build, linux-build, release, pages |
| `scripts/` | 빌드·릴리즈 도구 | `build-app.sh`, `ci-dmg.sh`, `release.sh`, 아이콘·배너 생성 Swift 스크립트 |

---

## 3. 핵심 동작 원리

```
AI 에이전트 (Claude Code 등)
   │  ① 훅(hook) 실행 — 에이전트 설정파일에 등록된 셸 명령어
   ▼
agentpet hook --event Stop --session abc...        (Sources/App/CLI.swift)
   │  ② JSON 이벤트를 Unix 도메인 소켓으로 전송
   │     ~/.agentpet/agentpet.sock
   │     (앱이 꺼져 있으면 ~/.agentpet/queue/ 에 파일로 적재 후 나중에 처리)
   ▼
EventSocketServer.swift  →  StateMapper.swift       (③ 상태 정규화)
   │     registered / working / waiting / done / idle
   ▼
④ UI 반영: 메뉴바 아이콘 + 펫 애니메이션 + 말풍선 + 네이티브 알림
   ▼
⑤ TranscriptReader.swift (JSONL 파싱) → PetCare.swift (XP 계산) → 펫 성장
```

### 지원 에이전트 12종 (`Sources/AgentPetCore/AgentCatalog.swift`)

Claude Code, Codex, Gemini CLI, Cursor, opencode, Windsurf, Antigravity,
GitHub Copilot, Kiro CLI, Factory Droid, Pi, Grok Build
\+ `agentpet run -- <아무 명령>` 범용 래퍼, GLM(Z.AI)은 Claude Code 경유로 자동 지원

### 훅 포맷 7가지 스타일 (`AgentHooks.swift` / `HookStyle`)

에이전트마다 설정 파일 위치와 JSON 구조가 모두 달라서 스타일로 분기한다.

| 스타일 | 대상 | 설정 경로 |
|---|---|---|
| `claudeNested` | Claude Code, Codex, Gemini, Droid, Grok | `~/.claude/settings.json`, `~/.codex/hooks.json`, `~/.gemini/settings.json`, `~/.factory/hooks.json`, `~/.grok/hooks/agentpet.json` |
| `cursorFlat` | Cursor, GitHub Copilot | `~/.cursor/hooks.json`, `~/.copilot/hooks/agentpet.json` |
| `windsurfFlat` | Windsurf | `~/.codeium/windsurf/hooks.json` |
| `antigravityNested` | Antigravity | `~/.gemini/config/hooks.json` |
| `kiroFlat` | Kiro CLI | `~/.kiro/agents/default.json` |
| `opencodePlugin` | opencode | `~/.config/opencode/plugin/agentpet.js` |
| `piExtension` | Pi | `~/.pi/agent/extensions/agentpet.ts` |

### 설계상 중요한 원칙: Fail-Open

```swift
// Sources/App/CLI.swift
// 모니터링 훅은 절대 에이전트를 막아서는 안 된다.
// PreToolUse의 exit 2 = "거부", Stop의 exit 2 = "계속 작업"으로 해석되므로
// 디코딩 실패 시에도 exit 0 (= 아무것도 안 함) 으로 안전하게 빠진다.
exit(0)
```

위험한 `PreToolUse`는 Copilot·Kiro·Grok에서 아예 등록하지 않는다.

---

## 4. 다마고치(Pet Care) 수치 — `PetCare.swift`

| 항목 | 값 |
|---|---|
| XP 1 획득 | 토큰 **5,000개** (`tokensPerXP`) |
| 세션 1개 완료 | XP **25** (`mealXP`) — 소비보다 *완료*를 더 높게 평가 |
| 레벨업 비용 | `60 × n × (n-1)` XP → Lv10 = 5,400 / Lv35 = 71,400 |
| 표시 레벨 | 내부 레벨 − 1 (새 펫은 Lv 0으로 보임) |
| 진화 5단계 | Hatchling(<5) → Companion(5~9) → Scout(10~19) → Hero(20~34) → Legend(35+) |
| 배고픔 5단계 | full(<4h) → satisfied(<10h) → peckish(<24h) → hungry(<48h) → starving(48h+) |
| 업적 14개 | 첫 식사, 세션 100/500, 토큰 1M/10M/50M, 레벨 5/10/20/35, 연속 7/14/30일, Night Owl |

---

## 5. 비용 추적 — `ModelPricing.swift`

per-million-token USD 단가가 하드코딩되어 있다.

| 모델 | Input | Output |
|---|---|---|
| Haiku | $1.00 | $5.00 |
| Sonnet (기본값) | $3.00 | $15.00 |
| Opus | $15.00 | $75.00 |
| 캐시 쓰기 | input × 1.25 | — |
| 캐시 읽기 | input × 0.1 | — |

추가로 `NativeUsageProbe.swift`가 **내 구독 사용량 한도를 실시간 조회**한다.

- Claude: Claude Code가 키체인 또는 `~/.claude/.credentials.json`에 저장한 OAuth 토큰을 **읽기만** 해서
  `https://api.anthropic.com/api/oauth/usage` 호출
- Codex: `~/.codex/auth.json` → ChatGPT usage 엔드포인트
- 토큰을 **쓰거나 갱신하지 않음**. 5분 주기 폴링. 잔량 15% 미만이면 경고

---

## 6. 프라이버시 / 보안 설계 (코드 검증 결과)

- 기본 **100% 오프라인**. Unix 도메인 소켓은 로컬 전용
- GitHub OAuth 로그인(선택) → `agentpet://link` 딥링크로 **64자 hex 디바이스 토큰** 발급,
  이후 `Authorization: Bearer <token>` 으로 동기화
- 서버로 가는 프로젝트 정보는 **폴더 경로의 마지막 이름만** (전체 경로 전송 안 함)
- 동기화 머지 규칙: 누적 카운터(xp/tokens/meals)는 `MAX()`로 **전진만** →
  오래된 기기가 다른 기기에서 올린 레벨을 깎을 수 없음.
  streak·lastFedAt은 "가장 최근 급여" 기준으로 따라감
- 훅 설치/제거는 명령어 문자열에 `agentpet`이 포함된 항목만 건드려서
  사용자의 다른 훅은 보존 (멱등성 보장)
- 설정 파일이 유효한 JSON이 아니면 덮어쓰지 않고 에러 (`HookInstallerError`)

---

## 7. 설치 및 사용법

### macOS

```bash
# Homebrew (추천)
brew install --cask ntd4996/tap/agentpet

# 직접 다운로드: Releases에서 AgentPet.dmg → Applications로 드래그
# Developer ID 서명 + Apple 공증 완료 → Gatekeeper 경고 없음
```

### Windows 10/11 (64-bit)

웹사이트 또는 Releases에서 설치파일/포터블 다운로드.
코드서명이 안 되어 있어 첫 실행 시 SmartScreen 경고 → **추가 정보 → 실행**.
관리자 권한 불필요(사용자 단위 설치), 자동 업데이트 지원.

### Linux

```bash
# 원라이너 (root 불필요, 최신 AppImage)
curl -fsSL https://agentpet.thenightwatcher.online/install.sh | sh

chmod +x AgentPet_*_amd64.AppImage && ./AgentPet_*_amd64.AppImage
sudo dpkg -i AgentPet_*_amd64.deb        # Debian / Ubuntu
sudo rpm -i AgentPet-*.x86_64.rpm        # Fedora / openSUSE
```

GNOME에서 트레이 아이콘이 안 보이면 AppIndicator 확장을 설치해야 한다.

### 소스 빌드 (macOS, Xcode 16 / Swift 6 필요)

```bash
git clone https://github.com/bmshin94/agentpet.git
cd agentpet
./scripts/build-app.sh release
open build/AgentPet.app

swift build && swift test     # 테스트
```

### 사용 순서

1. 앱 실행 → 메뉴바에 아이콘 등장
2. **Settings → General** → Claude Code 옆 **Install** 클릭 → 알림 **Enable**
3. 에이전트 실행 → 펫이 반응 시작
4. (선택) **Settings → Care → Sign in with GitHub** → 리더보드·크로스디바이스 동기화

### 완전 삭제

```bash
# ⚠️ 먼저 Settings → General에서 각 에이전트 [Remove] 를 눌러 훅을 제거할 것
#    (안 하면 앱 삭제 후 에이전트가 "명령어 없음" 에러를 냄)
brew uninstall --cask agentpet
rm -rf ~/.agentpet                                    # 펫 + 로컬 상태
rm -f  ~/Library/Preferences/com.agentpet.app.plist   # 설정
```

---

## 8. 플러그인? 스킬? MCP? → **셋 다 아니다**

> **독립 데스크톱 애플리케이션이고, Claude Code와는 "훅(Hooks)"으로만 연결된다.**

| 구분 | 정의 | AgentPet |
|---|---|---|
| MCP 서버 | Claude에게 새 **도구(tool)** 제공. 양방향, AI가 호출 | ❌ MCP 코드 없음 |
| Skill | Claude가 읽는 마크다운 지침서(`SKILL.md`) | ❌ 아님 |
| Plugin | Claude Code 확장 꾸러미(skill+hook+MCP) | ❌ 아님 (설치 방식이 다름) |
| **Hook** | 특정 이벤트에 **셸 명령어 실행**. 단방향 알림 | ✅ **이것** |
| **네이티브 앱** | 독립 실행 데스크톱 앱 | ✅ **이것** |

방향성의 차이가 핵심이다.

- MCP = Claude가 도구에게 **"이거 해줘"** (AI → 도구)
- Hook = Claude가 도구에게 **"나 이거 했어"** (이벤트 → 도구)

즉 AgentPet은 **AI의 능력을 확장하는 것이 아니라 AI를 관찰하는 쪽**이다.

### 설치 결과물

```
/Applications/AgentPet.app     ← 독립 앱 (SwiftUI)
~/.claude/settings.json        ← 여기에 hook 항목이 추가됨
~/.agentpet/agentpet.sock      ← 로컬 Unix 소켓
~/.agentpet/queue/             ← 앱이 꺼져 있을 때의 이벤트 큐
~/.agentpet/pets/              ← 입양한 펫
~/.agentpet/approval-gate.json ← (옵트인) 승인 게이트 설정
```

### 단 하나의 양방향 예외: 승인 게이트

`ApprovalGateConfig.swift` — `~/.agentpet/approval-gate.json`에 `{"tools":["Bash"]}`를 넣으면
`PreToolUse` 훅이 최대 10초 대기하면서 **말풍선에서 직접 허용/거부**를 누를 수 있다.
훅의 stdout으로 권한 결정을 돌려주는 방식. 기본은 **꺼짐(옵트인)**.
타임아웃은 데몬 10초 / 클라이언트 12초로 **의도적으로 어긋나게** 설정해
응답이 항상 열린 fd에 도착하도록 보장한다.

---

## 9. API 토큰이 필요한가? → **아니요**

| 구분 | API 키 필요 | 설명 |
|---|---|---|
| 기본 기능 전부 | ❌ | 모니터링·펫·알림·히스토리는 전부 오프라인 |
| 내 구독 한도 표시 | ❌ | **기존** OAuth 토큰을 읽기만 함. 새로 입력할 것 없음 |
| 웹 프로필·리더보드 | ❌ | GitHub OAuth → 디바이스 토큰 자동 발급 |
| 펫 갤러리 조회 | ❌ | 공개 CDN |

코드 검증: `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 사용처 **0건**.
`Bearer`가 등장하는 곳은 모두 (a) 기존 OAuth 토큰 읽기, (b) 자체 디바이스 토큰이다.

> 단, 웹 백엔드를 **직접 운영**하려면 Cloudflare 계정 + GitHub OAuth 앱 자격증명이 필요하다.
> 앱을 쓰는 입장에서는 전혀 필요 없다.

---

## 10. AI 에이전트 구축에 도움이 되는가? → 간접적으로 매우 큼

AgentPet 자체는 에이전트 **프레임워크가 아니다**. 하지만 다음은 그대로 배우거나 가져올 수 있다.

1. **멀티 에이전트 관측(Observability) 설계** — 상태를 5개로 정규화해서
   에이전트 12종을 UI 하나로 수용하는 패턴
2. **훅 어댑터 패턴** — 12개 에이전트의 설정 경로·이벤트명·JSON 구조가 한 파일에 문서화되어 있음
   (직접 리버스 엔지니어링하면 수 주 소요)
3. **Fail-Open 원칙** — 훅 exit code를 잘못 쓰면 에이전트가 무한 루프를 돌거나 도구가 막힌다는
   실전 함정과 회피법
4. **토큰/비용 계측** — `TranscriptReader`(JSONL 파싱) + `ModelPricing`(캐시 할인 포함)은
   그대로 차용 가능
5. **Human-in-the-loop 승인** — `PendingApprovalRegistry` + 어긋난 타임아웃 설계는
   동기 승인 처리의 정석
6. **세션 아카이브** — 90일 보관(`SessionArchiveStore`), 질문 감지(`QuestionDetector`),
   활동 요약(`ActivityFormatter`)

반면 **LLM 호출, 프롬프트 체이닝, RAG, 툴 정의, 멀티턴 상태관리는 전혀 없다.**

> 요약: 에이전트를 **만드는** 법은 안 배우지만,
> 에이전트를 **운영·관측·과금추적·승인처리**하는 법은 아주 잘 배울 수 있다.

---

## 11. React / PHP로 만들 수 있는가? → 부분적으로 가능

| 영역 | React | PHP | 비고 |
|---|---|---|---|
| 웹사이트 전체 (갤러리·리더보드·대시보드) | ✅ 완벽 | ✅ 완벽 | 일반 CRUD + 차트 |
| 백엔드 API | ✅ Next.js | ✅ Laravel | D1 → MySQL/PostgreSQL |
| 데스크톱 앱 (창·펫) | ✅ Electron/Tauri | ❌ | |
| 메뉴바/트레이 아이콘 | 🟡 Electron `Tray` | ❌ | 네이티브보다 투박 |
| 투명 클릭스루 펫 창 | 🟡 `setIgnoreMouseEvents` | ❌ | macOS는 까다로움 |
| 훅 수신 CLI 바이너리 | 🟡 Node 필요 | ❌ | PHP는 훅마다 인터프리터 부팅(50~100ms) |
| Unix 소켓 데몬 | ✅ Node `net` | 🟡 비현실적 | |
| 키체인 OAuth 읽기 | 🟡 `keytar` | ❌ | |
| 스프라이트 애니메이션 | ✅ Canvas/CSS | ❌ | React가 더 쉬움 |

### 현실적 추천

**추천 1 — React + Electron 전체 리메이크** (난이도 중, 2~3주)

```
Electron 메인 프로세스 (Node.js)
  ├─ net.createServer()        → Unix 소켓 데몬 (Swift 대체)
  ├─ Tray + BrowserWindow      → 메뉴바 + 투명 펫 창
  └─ 작은 Node/Go CLI 바이너리 → 훅 핸들러
React 렌더러
  ├─ 펫 스프라이트 (Canvas)
  ├─ 에이전트 목록 UI
  └─ 설정 화면
```

단점: 메모리 150MB+, 번들 100MB+

**추천 2 — PHP(Laravel) + React로 웹 플랫폼만 (가장 현실적)**

데스크톱 앱은 기존 AgentPet(MIT)을 쓰고, 웹 SaaS만 직접 만든다.
핵심 가치(비용 분석, 팀 리더보드)가 웹 쪽에 있으므로 수익화도 가장 빠르다.

**비추천 — PHP로 데스크톱 앱**: 장기 실행 GUI 데몬에 부적합.

---

## 12. 유튜브 강의 영상 제작 가능? → 가능, 소재가 우수

좋은 소재인 이유: 비주얼이 귀엽고(썸네일 유리), AI 코딩 트렌드 정중앙,
난이도 스펙트럼이 넓고, MIT라서 코드 공개·수정·강의 판매가 합법,
그리고 **한국어 콘텐츠가 거의 없다**.

### 추천 시리즈 10편

| # | 제목 | 길이 | 난이도 |
|---|---|---|---|
| 1 | Claude Code 여러 개 돌릴 때 이거 없으면 손해 | 8분 | 입문 |
| 2 | 내 AI 토큰 어디로 샜나 — 비용 추적 세팅 | 12분 | 입문 |
| 3 | Claude Code 훅(Hook) 완전정복 — settings.json 해부 | 20분 | 중급 |
| 4 | Hook vs MCP vs Skill, 뭐가 다른데? | 15분 | 중급 |
| 5 | 에이전트 12종 통합기 코드 리딩 | 25분 | 중급 |
| 6 | Swift로 투명하게 떠다니는 창 만들기 (NSPanel) | 18분 | 고급 |
| 7 | Unix 소켓으로 앱끼리 통신하기 | 20분 | 고급 |
| 8 | Tauri로 Windows 포팅하기 — Swift → Rust | 30분 | 고급 |
| 9 | Astro + Cloudflare D1로 리더보드 만들기 | 25분 | 중급 |
| 10 | 한국어 번역 PR 보내서 컨트리뷰터 되기 | 15분 | 입문 |

### 주의사항

- 🚨 **펫 캐릭터 저작권**: 갤러리에 포켓몬·아니메 캐릭터 펫이 다수 존재.
  영상에는 **오리지널 펫만** 노출할 것 (수익창출 이슈 가능)
- MIT 라이선스 고지 + 원작자 크레딧(설명란에 원본 저장소 링크)
- Windows SmartScreen 경고를 미리 설명하면 댓글 질문이 줄어듦

---

## 13. 수익화 아이디어

### 전제

| 항목 | 상태 |
|---|---|
| 라이선스 | **MIT** → 상업적 이용·수정·클로즈드소스 재배포 모두 허용 |
| 현재 수익모델 | Ko-fi / Buy me a coffee 기부뿐 (사실상 미수익화) |
| 결제·구독 코드 | **0줄** (백지상태) |
| 펫 아트 | 🚨 **MIT 아님**, 각 제작자 소유 → **상업화 금지** |
| 의무 | LICENSE 파일 + 저작권 고지 유지 |

핵심 통찰: **데스크톱 앱(무료 OSS)은 그대로 두고 "데이터 레이어"를 유료화**하는 것이 정석.
이미 `usage_daily`, `care_pets` 테이블에 돈이 되는 데이터가 쌓이는 구조가 있다.

### Tier 1 — 가장 현실적 (⭐⭐⭐)

**1. 팀/기업용 AI 코딩 비용 관제 SaaS** ← 최우선 추천

문제: 회사에서 개발자 20명이 Claude Code를 쓰는데, 관리자는
"이번 달 AI 비용이 얼마고 누가 얼마 썼는지" 전혀 모른다.
Anthropic 콘솔은 팀·레포별 분석을 제공하지 않는다.

```
Free       : 개인, 7일 보관
Pro        : $9/월    — 무제한 보관, CSV 내보내기, 예산 알림
Team       : $15/시트/월 — 팀 집계, 멤버·레포별 비용, Slack 알림, SSO
Enterprise : 문의     — 온프레미스, 감사 로그, 비용 배분(chargeback)
```

근거: `ModelPricing.swift`에 캐시 할인까지 포함된 단가 계산이 이미 구현됨.
`api/usage/sync.ts`에 프로젝트별·에이전트별 집계 엔드포인트가 이미 존재.
프라이버시 설계(폴더 마지막 이름만 전송)로 기업 보안팀 설득이 쉽다.

예상: 팀 50개 × 평균 8시트 × $15 ≈ **월 $6,000**
난이도: 중 (React + Laravel/Next.js, 2~3개월). **웹만 만들면 되므로 React/PHP로 충분**

**2. 한국 시장 로컬라이즈 + 유료 서비스** (난이도 하)

현재 한국어 지원이 없다.

1. 한국어 번역 PR → 컨트리뷰터 등록 (무료, 신뢰도 확보)
2. 한국 개발자용 세팅 가이드 블로그·뉴스레터 → 유료 멤버십
3. "한국 개발자 AI 토큰 사용량 리포트" 월간 발행 → 유료 구독
4. 기업 도입 컨설팅(설치 + 커스텀 대시보드) → 건당 과금

**3. 유튜브/강의 콘텐츠** (난이도 하, 초기 투자 거의 0)

```
유튜브 광고 + 멤버십       → 월 $300~2,000 (구독자 1만 기준)
인프런/유데미 유료강의     → ₩55,000 × 300명 ≈ ₩1,650만
기술 뉴스레터 (유료)       → 월 $5 × 200명 = $1,000
```

### Tier 2 — 중간 난이도 (⭐⭐)

**4. 오리지널 펫 마켓플레이스 (수수료 모델)**

⚠️ 반드시 **오리지널 펫만**. 기존 4,044마리는 저작권 리스크.

```
제작자가 오리지널 펫 업로드 → 유료 판매 ($2~5), 플랫폼 수수료 20~30%
+ 시즌 한정 펫, 기업 마스코트 콜라보
+ 기업 커스텀 펫 제작 의뢰 → 건당 $200~500
```

`submissions`, `pet_likes`, `pet_downloads` 테이블 + 관리자 승인 플로우가 이미 구현되어 있어
결제(Stripe / 토스페이먼츠)만 붙이면 된다.

**5. Arena 오토배틀러 — 게임 수익화**

`docs/specs/2026-07-15-arena-design.md`에 **미구현 설계서가 그대로 존재**한다.
18유닛 / 6종족 시너지 / 서버 권위 결정론 엔진 / 비동기 고스트 매칭,
봇 대 봇 4,000전 시뮬로 밸런싱(승률 41~58% 밴드)까지 완료.

```
배틀패스(시즌당 $5), 코스메틱 스킨(키운 펫을 아레나 스킨으로), 일일 런 추가 구매
→ 설계 원칙상 Pay-to-Win 금지. 코스메틱만.
```

**6. AI 에이전트 옵저버빌리티 DevTool로 피벗**

펫을 떼고 진지한 개발자 도구로:
세션 리플레이, 실패 패턴 분석, 팀 단위 승인 정책 관리, 에이전트별 성능 벤치마크.
Datadog/Langfuse의 "코딩 에이전트 전용" 포지션.
`SessionArchiveStore`(90일 보관), `ActivityFormatter`, `PendingApprovalRegistry`가 토대.

### Tier 3 — 틈새/보조 (⭐)

7. **기업 브랜딩 펫 (B2B)** — 회사 마스코트를 개발팀 데스크톱 펫으로. 건당 $500~2,000
8. **교육용 커리큘럼 판매** — "한 프로젝트로 배우는 Swift + Rust + TypeScript 풀스택"
9. **호스팅 대행** — Cloudflare Workers + D1 세팅·운영 대행. 월 $50~200
10. **번역/현지화 서비스** — 한국어 외 확장, 기업 과금

### 종합 추천 순위

| 순위 | 아이디어 | 수익성 | 난이도 | 기간 | React/PHP |
|---|---|---|---|---|---|
| 1 | 팀 AI 비용 관제 SaaS | ⭐⭐⭐⭐⭐ | 중 | 2~3개월 | ✅ |
| 2 | 유튜브 + 강의 콘텐츠 | ⭐⭐⭐ | 하 | 즉시 | ✅ |
| 3 | 한국어 현지화 + 컨설팅 | ⭐⭐⭐ | 하 | 1~2주 | ✅ |
| 4 | 오리지널 펫 마켓 | ⭐⭐⭐ | 중 | 1~2개월 | ✅ |
| 5 | Arena 게임 | ⭐⭐⭐ | 상 | 3~4개월 | ✅ |
| 6 | 옵저버빌리티 DevTool | ⭐⭐⭐⭐ | 상 | 6개월+ | ✅ |

### 실행 로드맵

```
Week 1-2   한국어 번역 PR → 컨트리뷰터 등록 + 앱 직접 사용 (신뢰도 + 경험)
Week 3-6   유튜브 1~3편 발행 → 반응 테스트 및 니즈 수집
Month 2-4  React + PHP로 "팀 비용 관제 SaaS" MVP → 무료 베타 10팀 → Pro $9 과금
Month 5+   Arena 또는 펫 마켓으로 확장 (바이럴 엔진)
```

### 법적 체크리스트

- ✅ MIT LICENSE 파일 + 원작자 저작권 고지 유지
- 🚨 기존 펫 아트 상업화 금지 (포켓몬·아니메 캐릭터 다수)
- ✅ 수익화 버전은 다른 이름/브랜드 사용 (상표 혼동 방지)
- ✅ 원작자(@ntd4996)에게 사전 고지 (커뮤니티 매너)
- ✅ 사용자 데이터 수집 시 개인정보처리방침 필수

---

## 14. 이 저장소에서 바로 해볼 수 있는 것

1. **한국어 로컬라이즈 추가** — `Localizations/ko.lproj/Localizable.strings` 신규 작성.
   `Tests/AgentPetCoreTests/LocalizationParityTests.swift`가 키 누락을 잡아준다
2. **Arena 구현 착수** — `docs/specs/2026-07-15-arena-design.md` 기반
3. **웹 대시보드 포크** — `web/src/pages/api/usage/` 를 기반으로 팀 집계 추가
4. **커스텀 에이전트 추가** — `AgentCatalog.swift` + `AgentHooks.swift`에 새 스타일 등록

---

## 15. 핵심 파일 빠른 참조

| 파일 | 역할 |
|---|---|
| `Sources/App/CLI.swift` | 훅 CLI 엔트리. Fail-Open 원칙 |
| `Sources/AgentPetCore/AgentHooks.swift` | 에이전트 12종 훅 스펙(경로·이벤트·스타일) |
| `Sources/AgentPetCore/HookInstaller.swift` | 훅 설치/제거 (멱등, 타인 훅 보존) |
| `Sources/AgentPetCore/EventSocketServer.swift` | Unix 도메인 소켓 수신 데몬 |
| `Sources/AgentPetCore/StateMapper.swift` | 에이전트 이벤트 → 5개 상태 정규화 |
| `Sources/AgentPetCore/PetCare.swift` | XP·레벨·진화·배고픔·업적 규칙 |
| `Sources/AgentPetCore/ModelPricing.swift` | 모델별 토큰 단가 (캐시 할인 포함) |
| `Sources/AgentPetCore/TranscriptReader.swift` | JSONL 트랜스크립트에서 사용량 추출 |
| `Sources/AgentPetCore/ApprovalGateConfig.swift` | 말풍선 승인 게이트 (옵트인) |
| `Sources/App/NativeUsageProbe.swift` | 구독 한도 실시간 조회 (읽기 전용) |
| `Sources/App/CareSyncController.swift` | 디바이스 토큰 기반 클라우드 동기화 |
| `web/src/lib/db.ts` | D1 스키마 18개 테이블 |
| `web/src/pages/api/care/sync.ts` | 전진-only 머지 동기화 |
| `windows/src-tauri/src/` | Windows·Linux Rust 구현 |
| `cdn-proxy/worker.js` | 펫 R2 미러링 CDN |
