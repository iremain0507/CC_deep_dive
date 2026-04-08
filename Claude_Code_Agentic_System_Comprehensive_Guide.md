# 하네스 엔지니어링: Claude Code로 배우는 Agentic LLM System 설계

> **이 문서의 목적**: Claude Code(~513K줄 TypeScript, 43개 도구)의 14개월 진화를 따라가며, Agentic LLM System의 하네스를 어떻게 설계·구축·발전시키는지를 AI 엔지니어가 단계적으로 습득할 수 있도록 구성한 기술 가이드.

---

## 목차

- **Part 1**: 이론적 기반 — 구현 이전에 확립된 원칙
- **Part 2**: 14개월의 진화 — 타임라인과 각 시점의 핵심 변화
- **Part 3**: 하네스의 단계적 구축 — Day 1부터 성숙까지
- **Part 4**: 하네스의 7개 서브시스템 상세
- **Part 5**: 설계 원칙과 구축 로드맵

---

# Part 1: 이론적 기반

## Agent = Model + Harness

에이전트 하네스는 **AI 모델의 추론을 제외한, 에이전트를 운영하기 위한 모든 소프트웨어 인프라**다. LLM이 CPU라면, 하네스는 운영체제다.

| OS 개념 | 에이전트 대응물 | Claude Code 구현 |
|---------|--------------|----------------|
| CPU | LLM | Anthropic API (Opus/Sonnet/Haiku) |
| RAM | 컨텍스트 윈도우 | 200K ~ 1M 토큰 |
| OS/Kernel | **하네스** | query.ts (1,729줄 Agentic Loop 상태 머신) |
| System Calls | 도구 호출 | 43 Tool Directories (isConcurrencySafe 플래그 기반 스케줄링) |
| File System | 영구 저장소 | memdir/ + history.ts + .claude/ |
| Scheduler | 도구 스케줄링 | StreamingToolExecutor (스트리밍 중 병렬 실행) |
| Permissions | 권한 | Sandbox(OS 격리) + 5-Layer Permission Pipeline |
| Virtual Memory | 컨텍스트 압축 | 5단계 파이프라인 + Reactive Recovery |

## "Building Effective Agents" (2024.12) — 설계 철학의 원점

Claude Code 탄생 2개월 전에 Anthropic이 발표한 에이전트 설계 원칙.

**핵심 주장**:
1. **"단순한 것부터 시작하라"** — 대부분의 문제는 에이전트가 필요 없다
2. **Workflow(사전 정의)와 Agent(자율적)를 구분하라**
3. **도구 설계에 프롬프트만큼 공을 들여라**

```
복잡도 순서:
  1. Prompt Chaining — 가장 단순
  2. Routing — 분류 후 위임
  3. Parallelization — 동시 실행
  4. Orchestrator-Workers — 동적 분해
  5. Evaluator-Optimizer — 반복 개선

→ 새 기능을 설계할 때마다 "이것은 몇 번 패턴인가?"를 먼저 물어라
```

## MCP와 Agent Skills: 두 개의 개방형 표준

| 표준 | 역할 | 채택 현황 |
|------|------|----------|
| **MCP** (Model Context Protocol) | 에이전트-도구 연결 | 월 9,700만 SDK 다운로드, Linux Foundation AAIF 거버넌스 |
| **Agent Skills** (agentskills.io) | 에이전트 절차적 지식 | 33+ 플랫폼 (GitHub Copilot, Codex, Gemini CLI, Cursor 등) |

```
MCP: "이 에이전트가 어떤 도구를 호출할 수 있는가"
Skills: "그 도구들로 이 작업을 어떻게 수행하는가"
→ 자체 프로토콜을 만들면 33+ 플랫폼 생태계에서 고립된다
```

---

# Part 2: 14개월의 진화 — 각 시점에서 무엇이 왜 추가되었는가

Claude Code는 처음부터 완벽하지 않았다. 각 시점의 변화는 **이전 단계에서 발견된 구체적 문제**에 대한 해법이다. 아래 타임라인은 "무엇이 바뀌었는가"뿐 아니라 **"왜 바뀌어야 했는가"와 "사용자 경험이 어떻게 바뀌었는가"**를 함께 설명한다.

---

### 2024.12 — "Building Effective Agents" 발표

5가지 워크플로우 패턴과 "단순한 것부터 시작하라" 원칙. Think Tool 패턴 공개 — 부작용 없는 "생각" 도구로 순차 도구 호출 품질 **54% 향상** (JSON 5줄 구현).

---

### 2025.02 — Claude Code 연구 프리뷰 탄생 (Day 1 하네스)

**있었던 것**: `while(true)` Agentic Loop, 기본 도구(검색/읽기/쓰기/실행), 기본 권한(모든 호출에 사용자 확인), 스트리밍 출력, Git 커밋

**없었던 것**: 컨텍스트 압축, 에러 복구, 서브에이전트, Hooks, Plugin, Auto Mode — 전부 이후 단계에서 추가됨

```
이 시점의 Agentic Loop (가장 단순한 형태):
  while (true) {
    messages → API 호출 (스트리밍)
    tool_use 블록 감지 → 권한 확인 → 도구 실행 → 결과를 messages에 추가
    tool_use 없으면 → 종료
  }
```

**발견된 문제**: 내부 테스트에서 "45분 걸리던 작업을 한 번에 완료"하지만, 대화가 길어지면 컨텍스트가 가득 차 성능이 급격히 저하. 모든 도구 호출마다 권한 확인이 필요해 사용자가 "승인 버튼 누르는 기계"로 전락.

---

### 2025.05 — GA 출시 + Claude 4 (SWE-bench 72.5%)

**추가된 것**: VS Code/JetBrains 통합, Claude Code SDK, Background Tasks

**사용자 규모 폭증으로 발견된 3대 Pain Point**:

**Pain Point 1: "매번 허용 눌러야 해"**
→ 이 시점에서 **규칙 학습 시스템** 도입. 사용자가 `npm run build`를 한 번 허용하면, 시스템이 접두사 `npm run:*`을 추출하여 유사 명령어를 자동 승인. (bashPermissions.ts: 접두사 추출 + 환경변수 필터링 + Heredoc 처리)

**Pain Point 2: "갑자기 에러 나요"**
→ **withRetry.ts** 도입: DEFAULT_MAX_RETRIES=10, 지수 백오프(500ms × 2^attempt). 529(과부하) MAX_529_RETRIES=3 연속 시 자동 모델 Fallback.

**Pain Point 3: "대화가 길어지면 멍해져"**
→ 초기에는 수동 `/compact` 명령어만 존재. 사용자가 직접 "언제 압축해야 하는지" 판단해야 하는 설계 — 시스템의 한계를 사용자에게 떠넘기는 상태.

---

### 2025.09 — 하네스 성숙의 전환점

3개의 핵심 블로그 포스트와 함께 Claude Code가 대규모 업데이트:

**"Context Engineering for AI Agents"**: "주의력 예산(Attention Budget)" 개념 — 컨텍스트에 정보를 추가하면 비용만 증가하는 게 아니라 기존 정보의 효과도 감소한다(Context Rot).

**"Building Agents with the Claude Agent SDK"**: 4단계 Agentic Loop 공식화 — Gather Context → Take Action → **Verify Work** → Loop. 검증(Verify) 단계의 추가가 Stop Hook의 이론적 근거.

**이 시점에 추가된 기능과 해결한 문제**:

| 기능 | 해결한 문제 | 기술적 구현 |
|------|-----------|-----------|
| **Auto-compact** | "언제 압축해야 하지?" → 시스템이 판단 | effectiveWindow - 13K 토큰 버퍼 초과 시 Forked Agent가 자동 요약 |
| **Error Withholding** | "에러 메시지가 불안해요" → 복구 가능한 에러를 숨김 | 413/max_tokens 에러를 withheld 변수로 보류, 복구 성공 시 폐기 |
| **Checkpoint** | "Claude가 코드를 망쳤어" → 매 어시스턴트 메시지마다 자동 스냅샷 | .claude/.file-history/에 버전별 백업, Esc-Esc 롤백 |
| **Subagents** | "한 번에 한 가지만 해" → 독립 컨텍스트 에이전트 | CacheSafeParams로 부모 캐시 공유하며 분기 |
| **Hooks** | "매번 린터를 수동으로" → 이벤트 기반 자동화 | PreToolUse/PostToolUse/Stop 20+ 이벤트, query.ts의 도구 실행 파이프라인에 직접 통합 |
| **StreamingToolExecutor** | "도구 실행 기다리는 중" → 스트리밍 중 병렬 도구 실행 | isConcurrencySafe 플래그 기반 동시성 제어 |

**Hook이 예외 없이 실행되는 이유**:
```
Hook은 query.ts의 Agentic Loop에 직접 통합되어 있다.
도구 호출의 흐름:

  1. 모델이 tool_use 블록 생성
  2. canUseTool() 호출 (hooks/useCanUseTool.tsx)
     → 이 함수 내에서 executePermissionRequestHooks() 실행
     → PreToolUse Hook이 여기서 실행됨
     → Hook 결과에 따라 allow/deny/ask/defer 결정
  3. 도구 실행
  4. PostToolUse Hook 실행
  5. 모든 도구 완료 + 모델이 도구 호출을 멈추면
     → handleStopHooks() 실행 (query/stopHooks.ts)
     → Stop Hook이 blocking error를 반환하면
       → 에러를 사용자 메시지로 주입 → 루프 계속
       → 모델이 에러를 보고 수정 시도

핵심: Hook은 "선택적 플러그인"이 아니라 "도구 실행 파이프라인의 필수 단계"다.
canUseTool()이 Hook을 건너뛰는 경로는 존재하지 않는다.
설정에서 Hook을 등록하면, 그 시점부터 모든 도구 호출이 Hook을 통과한다.

세션 시작 시 captureHooksConfigSnapshot()으로 Hook 설정을 스냅샷 —
세션 중 설정 파일이 변경되어도 숨겨진 Hook 수정이 불가능 (보안).
```

---

### 2025.10 — 플랫폼 전환

| 기능 | 해결한 문제 | 기술적 구현 |
|------|-----------|-----------|
| **Agent Skills** | "에이전트가 전문 지식이 없어" | SKILL.md 3단계 Progressive Disclosure: 메타데이터(~100 토큰) → 본문(<5K) → 상세(무제한) |
| **Plugins** | "도구/훅/스킬을 배포하기 어려워" | Skills + Hooks + MCP + Agents를 하나의 Git 레포로 패키징 |
| **Web App** | "로컬 설치가 필요해" | claude.ai/code — 격리된 클라우드 VM + 네트워크 화이트리스트 |
| **OS 수준 Sandboxing** | "보안이 걱정돼" | bubblewrap(Linux)/seatbelt(macOS) 이중 경계 — Permission 프롬프트 **84% 감소** |
| **Tool Search** | "도구가 너무 많아" | 핵심 도구만 상시 로드 + 나머지 on-demand → 토큰 **85% 절감**, 정확도 49%→74% |

**Sandboxing이 보안과 사용성을 동시에 개선한 방법**:
```
이전: 모든 파일 쓰기마다 "허용하시겠습니까?" (사용자가 매번 판단)
이후: 프로젝트 디렉토리 내 쓰기 → 샌드박스가 보장 → 자동 허용
      프로젝트 외부 쓰기 → 샌드박스가 물리적으로 차단 → 권한 자체가 불필요

→ "개별 행동의 안전성을 판단하는 대신, 안전하지 않은 것이 물리적으로 불가능한 환경"
→ 네트워크도 동일: Unix 도메인 소켓 → 프록시 경유, Git 자격증명은 샌드박스 외부에만 존재
```

---

### 2025.11 — 장시간 에이전트 패턴 확립

**"Effective Harnesses for Long-Running Agents"** 발표. Initializer-Executor 패턴, "한 세션에 한 기능" 원칙.

**Advanced Tool Use** 출시:
- Tool Search: 토큰 85% 절감
- Programmatic Tool Calling: 토큰 37% 절감 (에이전트가 Python 코드로 도구 오케스트레이션)
- Tool Use Examples: 파라미터 정확도 72%→90% (1-5개 예시 추가)
- Code Execution with MCP: 98.7% 절감 (150K→2K 토큰, 중간 결과가 모델 컨텍스트를 통과하지 않음)

---

### 2025.12 — 생태계 확립

- MCP를 Linux Foundation AAIF에 기부 (Anthropic + OpenAI + Google + Microsoft + AWS)
- Agent Skills를 agentskills.io 개방형 표준으로 발표 (33+ 플랫폼)
- Bun 런타임 인수 → `feature()` API로 Dead Code Elimination
- Desktop App, Scheduled Tasks, Channels (Telegram/Discord/Webhook)

---

### 2026.02 — 자율성의 시대

**Opus/Sonnet 4.6 (1M 컨텍스트 윈도우)**: 컨텍스트 압축이 "생존의 문제"에서 "최적화의 문제"로 전환. 하지만 KV-Cache 최적화와 주의력 예산 관리는 여전히 필수.

**자율성 측정** (500K+ 세션 데이터):
```
경험 사용자(750+ 세션): 40%+ 자동승인, 동시에 전략적 개입 빈도 5%→9%
→ "더 많이 맡기면서, 더 전략적으로 개입" = 점진적 신뢰 구축의 데이터적 증거
자율 작업 시간 99.9th percentile: 25분→45분 (4개월 만에 2배)
완전 위임 가능 비율: 0-20% (나머지는 인간 검증 필수)
```

**C 컴파일러 사례 (16 에이전트 병렬)**: 2,000 세션, 100K줄 Rust, GCC Torture 99% 통과. 핵심: 파일시스템 기반 태스크 잠금 + Git worktree 격리 + Worker별 예산 통제.

---

### 2026.03 — 신뢰의 코드화

**Auto Mode**: 2계층 안전 시스템 (서버측 프롬프트 인젝션 탐지 + Sonnet 4.6 트랜스크립트 분류기). 오탐률 0.4%.

**Skills 2.0**: Evals(ON/OFF 비교) + Blind A/B 테스트 + Description 최적화(~20회 반복). "추측에서 측정 기반으로."

**Planner-Generator-Evaluator 아키텍처**: 자기 평가 편향(Self-Evaluation Bias) 발견 — "에이전트는 자신의 평범한 결과를 자신있게 칭찬." 생성과 평가를 분리하면 품질 **2-3배 향상**.

**안전성 연구 경고**:
```
Agentic Misalignment: 평가 환경이 실제 위험을 8.5배 과소평가 (55.1% vs 6.5%)
Emergent Misalignment: 보상 해킹이 코딩 외 영역으로 일반화 (12% 사보타주)
Sabotage Evaluations: AI 코드 리뷰어(Sonnet)가 의도적 버그 99% 탐지
→ 평가만 믿지 말라. 런타임 모니터링 + AI-reviewing-AI 필수.
```

---

# Part 3: 하네스의 단계적 구축

## Phase 1: Foundation — Day 1에 만들 것

```
┌─────────────────────────────────────────────────────────┐
│  while (true) {                                         │
│    messages + systemPrompt → API 호출 (스트리밍)          │
│    tool_use 블록 감지 → 권한 확인 → 도구 실행 → 결과 추가 │
│    tool_use 없으면 → 종료                                │
│  }                                                      │
│                                                         │
│  + 3-5개 핵심 도구 (검색, 읽기, 쓰기, 명령 실행)          │
│  + Think Tool (JSON 5줄, 추론 품질 54% 향상)              │
│  + 기본 권한 (모든 도구 호출에 사용자 확인)                │
│  + 스트리밍 응답 출력                                    │
│  + 도구 설계 원칙: 시맨틱 응답, 행동 가능한 에러,         │
│    response_format(detailed/concise), 25K 토큰 응답 제한  │
└─────────────────────────────────────────────────────────┘
```

## Phase 2: Resilience — 프로덕션 피드백에 의해 필요해지는 것

```
┌─────────────────────────────────────────────────────────┐
│  컨텍스트 압축 (Auto-compact):                           │
│    effectiveWindow - 13K 버퍼 초과 시 Forked Agent 요약  │
│                                                         │
│  Error Withholding + Cascade Recovery:                   │
│    413/max_tokens 에러를 보류 → 복구 시도 → 성공 시 폐기  │
│    Guard 플래그로 무한 루프 방지                          │
│                                                         │
│  API 재시도 (DEFAULT_MAX_RETRIES=10):                    │
│    지수 백오프 + 529 3회 시 모델 Fallback                 │
│                                                         │
│  비용 추적 (모델별, 에이전트별)                           │
│  체크포인트 (세션 히스토리 JSONL + 파일 스냅샷)            │
│  CLAUDE.md 계층 (관리 → 글로벌 → 프로젝트 → 로컬)        │
└─────────────────────────────────────────────────────────┘
```

## Phase 3: Scale — 복잡한 태스크를 처리하기 위해

```
┌─────────────────────────────────────────────────────────┐
│  Sub-Agent (CacheSafeParams 캐시 공유):                  │
│    부모와 동일한 시스템 프롬프트/도구/모델 → 캐시 HIT     │
│                                                         │
│  MCP 클라이언트 통합 + Tool Search (토큰 85% 절감)       │
│  StreamingToolExecutor (스트리밍 중 병렬 도구 실행)       │
│  권한 학습 (규칙 추출 + 200ms Grace Period + ML 분류기)  │
│  Agent Skills (SKILL.md 3단계 Progressive Disclosure)    │
│  OS 수준 Sandboxing (Permission 프롬프트 84% 감소)       │
│  Hooks (PreToolUse/PostToolUse/Stop, 20+ 이벤트)         │
└─────────────────────────────────────────────────────────┘
```

## Phase 4: Autonomy — 신뢰를 코드로 설계

```
┌─────────────────────────────────────────────────────────┐
│  Auto Mode (ML 분류기 + 서버측 탐지, 오탐률 0.4%)        │
│  Code Execution with MCP (98.7% 토큰 절감)              │
│  검증 분리 (Stop Hook / Evaluator / Writer-Reviewer)     │
│  Skills 2.0 (Evals + Blind A/B + Description 최적화)    │
│  장시간 에이전트 (CLAUDE.md + CHANGELOG.md + Ralph Loop)  │
│  안전성 모니터링 (AI-reviewing-AI, 런타임 이상 탐지)      │
│  관찰가능성 (OpenTelemetry + Permission Audit Trail)      │
└─────────────────────────────────────────────────────────┘
```

---

# Part 4: 하네스의 7개 서브시스템 상세

## 서브시스템 1: 입력 처리

사용자가 텍스트를 입력하면 API 호출이 되기까지 7단계:

```
1. 입력 정규화 — string → ContentBlockParam[]
2. 첨부 수집 — 파일 변경 감지, 메모리, 태스크 알림
3. 시스템 프롬프트 조립 — 기본 + CLAUDE.md 계층 + Git + 메모리 지시
4. 도구 풀 구성 — Built-in + MCP - Deny 규칙, 안정 정렬 (캐시 보존)
5. Agent Skills 로드 — 메타데이터만 (~100 토큰/스킬, 본문은 매칭 시)
6. Tool Search — 핵심 도구만 상시, 나머지 on-demand (85% 절감)
7. CacheSafeParams 스냅샷 — Sub-Agent 캐시 공유용
```

## 서브시스템 2: Agentic Loop (State Machine)

query.ts의 1,729줄 AsyncGenerator. 모든 반복의 "왜 계속되었는가"를 `transition` 필드에 기록.

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number      // 0-3, 턴마다 리셋
  hasAttemptedReactiveCompact: boolean      // 턴마다 리셋
  turnCount: number
  transition: Continue | undefined          // 이전 반복의 계속 사유
}

7 Continue 사유: collapse_drain_retry, reactive_compact_retry,
  max_output_tokens_escalate, max_output_tokens_recovery,
  stop_hook_blocking, token_budget_continuation, next_turn

10 Exit 사유: completed, max_turns, stop_hook_prevented (정상),
  model_error, prompt_too_long, image_error, blocking_limit (오류),
  aborted_streaming, aborted_tools, hook_stopped (중단)

Guard 플래그: hasAttemptedReactiveCompact=true이면 재시도 안 함
             maxOutputTokensRecoveryCount≥3이면 복구 포기
             → 이 플래그들이 없으면 무한 루프 발생
```

## 서브시스템 3: 도구 실행

**StreamingToolExecutor** — 모델 스트리밍 중 도구 선행 실행:

```
query.ts:838 — tool_use 블록 감지 즉시 streamingToolExecutor.addTool()
→ 모델 응답이 아직 스트리밍 중인데 도구가 이미 실행 시작

isConcurrencySafe = true (서로 병렬 실행 가능):
  Read, Grep, Glob, WebFetch, WebSearch, LSP

isConcurrencySafe = false (단독 실행, 기본값):
  Bash, Edit, Write, Agent, 기타 쓰기 도구
  → 새 도구는 기본 false (Fail-Closed)

Bash 에러 → 형제 도구에 캐스케이드 (siblingAbortController.abort)
Read/WebFetch 에러 → 독립 (다른 도구 계속 실행)
결과 순서: 병렬이더라도 tool_use 순서대로 yield
```

## 서브시스템 4: 5단계 컨텍스트 관리 파이프라인

매 Agentic Loop 반복마다 API 호출 전에 5단계가 순차 실행. 순서에 의미가 있다.

**Stage 1: Tool Result Budget** (query.ts:379)
```
무엇: 대형 도구 결과를 디스크 파일 참조로 교체
트리거: 매 반복 자동. DEFAULT_MAX_RESULT_SIZE_CHARS=50,000자 초과 시
이유: 100KB 결과가 이후 모든 API 호출에서 재전송 → 디스크 이관으로 비용 제거
```

**Stage 2: Snip** (query.ts:401, feature: HISTORY_SNIP)
```
무엇: 오래된 메시지 그룹을 경계 마커 기반으로 통째로 제거
비용: 0ms (메시지 배열 조작만)
이유: 가장 저비용 방법으로 컨텍스트 확보
순서: Microcompact 전에 실행 — snip된 메시지의 tool_use_id가 MC에 전달되지 않도록
```

**Stage 3: Microcompact** (query.ts:412)
```
무엇: 오래된 Read/Bash/Grep 결과 내용을 "[Old tool result cleared]" 마커로 교체
비용: <1ms (문자열 교체)
두 가지 모드: 일반 MC (클라이언트 측 교체) + Cached MC (API cache_edits 서버측 삭제)
이유: 이미 처리된 도구 결과의 내용은 재전송 불필요. tool_use_id 구조는 유지 → 캐시 정렬 보존
```

**Stage 4: Context Collapse** (query.ts:440, feature: CONTEXT_COLLAPSE)
```
무엇: 인접한 메시지 그룹을 축약된 요약으로 교체
이유: Auto-compact(전체 요약) 전에 실행 — collapse가 충분하면 autocompact 불필요
      → 세분화된 컨텍스트를 보존 (전체 요약보다 정보 손실 적음)
```

**Stage 5: Auto-compact** (query.ts:453)
```
무엇: Forked Agent가 전체 대화를 요약
트리거: 토큰 수 > (effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS(13K))
       effectiveWindow = contextWindow - min(maxOutputTokens, 20K)
       → 200K 모델에서 약 167K 토큰 (~83.5%) 초과 시
비용: LLM 호출 (CacheSafeParams로 부모 캐시 공유 → 입력 비용 절감)
후처리: 파일 5개 + 스킬 25K 토큰 + MCP 정보 재주입
안전장치: MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3 연속 실패 시 세션 내 중단
```

**Stage 6 (비상): Reactive Compact** (query.ts:1119, feature: REACTIVE_COMPACT)
```
무엇: API 413(prompt-too-long) 에러 발생 후 비상 압축
트리거: API 응답에서 413 또는 media size 에러 감지 시 (정상 흐름에서는 절대 실행 안 됨)
동작: Error Withholding으로 에러를 보류 → Collapse drain 시도 → 부족하면 전체 요약
      → 성공 시 에러 폐기 (사용자는 모름), 실패 시 에러 표출
안전장치: hasAttemptedReactiveCompact=true → 동일 턴 내 재시도 방지
```

## 서브시스템 5: 오류 복구 (Error Withholding)

```
API 스트리밍 중 (query.ts:800-820):
  413 감지 → withheld = true (UI에 전달 안 함)
  max_tokens 감지 → withheld = true
  media size 감지 → withheld = true
  정상 메시지 → withheld가 아니면 yield

스트리밍 완료 후 복구 Cascade:
  413: Collapse drain → Reactive compact → 모두 실패 시 에러 표출
  max_tokens: 8K→64K 확장 → 복구 메시지 ×3 → 소진 시 표출
  Guard 플래그: 각 복구 경로에 "이미 시도함" 플래그로 무한 루프 방지

Death Spiral 방지 (query.ts):
  API 에러 후 Stop Hook이 실행되면 다시 API 에러 → 무한 루프
  → lastMessage.isApiErrorMessage이면 Stop Hook 건너뜀
```

## 서브시스템 6: 자동 메모리 시스템

**메커니즘 1: 자동 추출** (extractMemories.ts)
```
트리거: 모델이 도구 호출 없이 응답 완료 시 (턴 종료)
조건: (1) autoMemory 활성, (2) 메인 에이전트가 메모리를 직접 기록하지 않았을 것
      (hasMemoryWritesSince → 메인/포크 상호 배제)
동작: Forked Agent가 대화를 분석하여 기억할 내용을 메모리 파일에 저장
권한: Read/Grep/Glob 무제한, Write/Edit은 메모리 디렉토리만 허용
```

**메커니즘 2: 자동 주입** (findRelevantMemories.ts + query.ts:301)
```
트리거: 매 턴의 Agentic Loop 시작
동작: startRelevantMemoryPrefetch() → Haiku sideQuery → 관련 메모리 5개 선별
타이밍: 모델 스트리밍과 동시에 실행 → 97% 스트리밍 중 완료 → 추가 대기 0ms
```

**메커니즘 3: 4가지 유형 분류** (memdir.ts:189)
```
user: "시니어 백엔드 엔지니어, React 초보" → 응답 난이도 조정에 활용
feedback: "mock 테스트 금지 — 작년에 프로덕션 마이그레이션 실패 숨김" → 실수 반복 방지
project: "3/5까지 모바일 릴리스 브랜치 컷 — 비핵심 머지 동결" → 제안 맥락화
reference: "파이프라인 버그는 Linear 'INGEST' 프로젝트에서 추적" → 외부 참조

기억하지 않는 것: 코드 패턴(코드에서 도출), Git 히스토리(git log이 권위적),
                 디버깅 해결책(커밋에 포함), CLAUDE.md에 이미 있는 내용
```

## 서브시스템 7: 종료 조건

```
Stop Hook 생명주기 (query/stopHooks.ts):
  1. 모델 응답 완료 (도구 호출 없음)
  2. API 에러였나? → YES → Stop Hook 건너뜀 (Death Spiral 방지)
  3. 사용자 정의 Hook 병렬 실행 (예: npm test, 린터)
  4. blockingErrors > 0? → 에러를 사용자 메시지로 주입 → 루프 계속
     (모델이 에러를 보고 수정 시도 — 검증 루프)
  5. preventContinuation? → 즉시 종료
  6. Token Budget: 출력이 예산의 90% 미만이면 nudge 주입 → 계속

백그라운드 작업 (Stop Hook과 동시에, 비차단):
  - 프롬프트 제안 (Haiku, fire-and-forget)
  - 메모리 추출 (Forked Agent)
  - 도구 사용 요약 생성
```

---

# Part 5: 설계 원칙과 구축 로드맵

## 10가지 설계 원칙

1. **최소 하네스로 시작하라** — Day 1은 while(true) + 기본 도구
2. **사용자의 인지 부하를 시스템의 계산 부하로 변환하라** — 5가지 Pain Point Transfer 패턴: Silent Success, Progressive Learning, Time Overlap, Information Layering, Invisible Safety
3. **에러 처리를 계층적으로 설계하라** — 저비용 → 고비용, Guard 플래그로 무한 루프 방지
4. **컨텍스트는 가장 비싼 자원이다** — 5단계 압축, KV-Cache 보존이 비용의 핵심
5. **캐시 공유가 Multi-Agent 비용을 해결한다** — CacheSafeParams
6. **검증을 생성과 분리하라** — Stop Hook, Evaluator, Writer-Reviewer (2-3배 향상)
7. **자율성은 스펙트럼이다** — default → plan → acceptEdits → dontAsk → auto → bypassPermissions
8. **파일시스템이 메모리다** — 벡터 DB가 아닌 파일과 Git (검사 가능, 편집 가능, 버전 관리)
9. **프로토콜 표준을 채택하라** — MCP, Agent Skills, A2A, OpenTelemetry
10. **하네스는 지속적으로 재평가하라** — 1M 시대에 200K 시대의 기능이 불필요할 수 있다

## 핵심 수치 종합 (코드베이스 검증 완료)

| 메트릭 | 값 | 검증 출처 |
|--------|---|----------|
| Think Tool 추론 향상 | **54%** (항공사 도메인) | Anthropic 블로그 |
| Code Execution 토큰 절감 | **98.7%** | Anthropic 엔지니어링 |
| Tool Search 토큰 절감 / 정확도 | **85%** / 49%→74% | Anthropic 엔지니어링 |
| Sandboxing 권한 프롬프트 감소 | **84%** | Beyond Permission Prompts |
| 검증 루프 품질 향상 | **2-3배** | Building Effective Agents |
| Auto Mode 오탐률 | **0.4%** | Claude Code Auto Mode |
| AI 코드 리뷰 사보타주 탐지 | **99%** | Sabotage Evaluations |
| 평가 vs 실제 위험 차이 | **8.5배** 과소평가 | Agentic Misalignment |
| AUTOCOMPACT_BUFFER_TOKENS | 13,000 | autoCompact.ts:62 |
| DEFAULT_MAX_RETRIES | 10 | withRetry.ts:52 |
| DEFAULT_MAX_RESULT_SIZE_CHARS | 50,000자 | toolLimits.ts:13 |
| MAX_MEMORY_CHARACTER_COUNT | 40,000자 | claudemd.ts:92 |
| query.ts Agentic Loop | 1,729줄, 7 Continue + 10 Exit | query.ts |
| 도구 디렉토리 수 | 43개 | src/tools/ |
| 소스 코드 총 줄 수 | 512,685줄 (src/) | find + wc -l |

---

*하네스 엔지니어링: Claude Code로 배우는 Agentic LLM System 설계 v2.0*
