# Claude Code로 배우는 Enterprise Agentic LLM System 구축의 기술

## 14개월간의 진화가 증명한 Agentic System 설계의 원칙과 실전

> **이 문서의 목적**: AI 엔지니어가 Claude Code라는 하나의 Production-Grade 구현체를 통해, Agentic LLM System을 설계·구축하는 데 필요한 모든 기술적 통찰을 시간순으로 자연스럽게 습득할 수 있도록 구성한 종합 가이드.
>
> **읽는 방법**: 이 문서는 Claude Code의 진화 순서를 따라갑니다. 각 시기마다 "왜 이런 변화가 필요했는가 → 어떻게 해결했는가 → 당신의 시스템에 무엇을 적용할 것인가"의 3단 구조로 설명합니다.

---

## 서문: 왜 Claude Code인가

2025년 2월 연구 프리뷰로 시작하여 14개월 만에 연간 $1B+ 매출을 달성한 Claude Code는, 현존하는 가장 성숙한 Production-Grade Agentic Coding System이다.

이 시스템은 단순한 "LLM + 도구 호출"이 아니다. **~800K줄의 TypeScript, 40+ 도구, 5계층 권한 시스템, 4단계 컨텍스트 압축, 7가지 오류 복구 경로**를 갖춘 완전한 에이전트 하네스(Harness)다.

그러나 이 모든 것이 처음부터 존재한 것은 아니다. Claude Code는 Anthropic이 자체 블로그에서 공유한 원칙들을 직접 코드로 실현하면서 진화해 왔고, 그 과정에서 발견한 교훈들이 다시 블로그로 돌아가 공유되었다.

```
이론 → 구현 → 프로덕션 교훈 → 이론 보완 → 구현 개선

  "Building Effective Agents" (2024.12)
        ↓
  Claude Code Research Preview (2025.02)
        ↓
  프로덕션 피드백 수집
        ↓
  "Context Engineering" + "Agent SDK" (2025.09)
        ↓
  Checkpoints, Hooks, Subagents 추가
        ↓
  "Effective Harnesses" (2025.11)
        ↓
  Auto Mode, 1M Context, Plugins
        ↓
  "Measuring Agent Autonomy" (2026.02)
        ↓
  현재: 하루 1개 릴리스, 500K+ 세션 데이터
```

이 문서는 이 순환을 따라가며, 각 단계에서 **어떤 문제가 발견되었고, 어떤 해법이 만들어졌으며, 그것이 당신의 시스템에 어떤 의미가 있는지**를 설명한다.

---

## 목차

- **Part 1: 이론적 기반** — 구현 이전에 확립된 원칙들
- **Part 2: 탄생** — 최소 하네스로 시작하기
- **Part 3: 프로덕션의 현실** — 사용자가 부딪힌 문제들
- **Part 4: 하네스의 성숙** — 시스템이 복잡성을 흡수하는 방법
- **Part 5: 생태계 확장** — 확장성과 표준화
- **Part 6: 자율성의 시대** — 신뢰를 코드로 설계하기
- **Part 7: 종합** — 당신의 시스템에 적용하기

---

# Part 1: 이론적 기반 (2024.03 — 2024.12)

## 모델이 에이전트가 되기까지

### Claude 3 → 3.5 Sonnet: 코딩 능력의 임계점

| 시점 | 모델 | SWE-bench | 의미 |
|------|------|-----------|------|
| 2024.03 | Claude 3 Opus | — | 200K 컨텍스트, 하지만 코딩에 불충분 |
| 2024.06 | Claude 3.5 Sonnet | 33.4% | **임계점**: 코딩 에이전트가 실용적으로 가능해진 순간 |
| 2024.10 | Claude 3.5 Sonnet (업그레이드) | 49.0% | 컴퓨터 사용(Computer Use) 베타와 함께 출시 |

**2024년 6월이 전환점이었다.** Claude 3.5 Sonnet이 코딩 벤치마크에서 이전 최고 모델(3 Opus)을 넘어서면서, "LLM이 코드를 쓰는 것"이 장난감에서 도구로 전환되었다.

### MCP: 도구 연결의 표준화 (2024.11)

2024년 11월, Anthropic은 **Model Context Protocol (MCP)**을 오픈소스로 공개했다.

```
MCP 이전:                          MCP 이후:
  에이전트 ──→ GitHub API (직접 구현)    에이전트 ──→ MCP Client
  에이전트 ──→ Slack API (직접 구현)              ──→ MCP Server (GitHub)
  에이전트 ──→ DB API (직접 구현)                 ──→ MCP Server (Slack)
  (N개 통합 = N개 커스텀 코드)                    ──→ MCP Server (DB)
                                               (1개 프로토콜 = N개 서버)
```

**당신의 시스템에 주는 교훈**: 도구 통합을 직접 구현하지 말라. MCP를 채택하면 커뮤니티가 만든 수천 개의 서버를 즉시 사용할 수 있다. 2025년 12월 Linux Foundation AAIF에 기부된 이후, MCP는 OpenAI, Google, Microsoft까지 채택한 업계 표준이 되었다 (월 9,700만 SDK 다운로드).

### "Building Effective Agents": 설계 철학의 원점 (2024.12)

Claude Code가 태어나기 2개월 전, Anthropic은 에이전트 설계의 바이블이 된 글을 발표했다.

**핵심 주장 3가지:**

**1. "단순한 것부터 시작하라"**
> "Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short."

대부분의 문제는 에이전트가 필요 없다. Prompt Chaining이나 Routing으로 충분하다. 에이전트는 **마지막 수단**이다.

**2. Workflow와 Agent를 구분하라**
- **Workflow**: LLM이 사전 정의된 코드 경로를 따름 (결정론적)
- **Agent**: LLM이 스스로 프로세스와 도구 사용을 결정 (자율적)

**3. 도구 설계에 프롬프트만큼 공을 들여라**
> "Tool design deserves the same rigor as prompt engineering."

5가지 조합 가능한 패턴이 제시되었다:

```
복잡도 순서:
  1. Prompt Chaining (순차 체인) — 가장 단순
  2. Routing (라우팅) — 분류 후 위임
  3. Parallelization (병렬화) — 동시 실행
  4. Orchestrator-Workers (오케스트레이터) — 동적 분해
  5. Evaluator-Optimizer (평가-최적화) — 반복 개선
```

**당신의 시스템에 주는 교훈**: 이 5가지 패턴을 외워라. 새로운 기능을 설계할 때마다 "이것은 몇 번 패턴인가?"를 먼저 물어라. 5번(평가-최적화)이 필요한 것을 확인하기 전에 4번(오케스트레이터)을 만들지 말라.

---

# Part 2: 탄생 — 최소 하네스로 시작하기 (2025.02)

## Claude Code Research Preview (2025.02.24)

Claude 3.7 Sonnet(최초의 확장된 사고 모델)과 함께, **Claude Code가 "제한된 연구 프리뷰"로 탄생**했다.

### 최초의 하네스: 무엇이 있었고, 무엇이 없었는가

```
있었던 것 (최소 하네스):              없었던 것 (이후 추가됨):
  ✓ 터미널 기반 CLI                   ✗ IDE 통합
  ✓ 코드 검색/읽기                    ✗ 서브에이전트
  ✓ 파일 편집/쓰기                    ✗ 체크포인트/롤백
  ✓ 테스트 실행                       ✗ Hooks
  ✓ Git 커밋                          ✗ Plugin 시스템
  ✓ 기본 권한 확인                    ✗ Auto Mode
  ✓ while(true) Agentic Loop         ✗ 4단계 컨텍스트 압축
                                      ✗ Error Withholding
                                      ✗ Streaming Tool Executor
                                      ✗ CacheSafeParams
                                      ✗ Team Memory
```

**이것이 핵심 교훈이다**: Claude Code는 처음부터 완벽하지 않았다. `while(true)` 루프 + 기본 도구 + 기본 권한으로 시작했다. 그리고 **프로덕션 피드백이 모든 후속 개선을 이끌었다.**

### 최초의 Agentic Loop

가장 단순한 형태의 하네스:

```
while (true) {
  1. 사용자 메시지 + 시스템 프롬프트 → API 호출
  2. 응답에 tool_use 블록이 있으면:
     a. 각 도구에 대해 권한 확인
     b. 도구 실행
     c. 결과를 메시지에 추가
     d. continue (루프 반복)
  3. tool_use가 없으면:
     → break (완료)
}
```

이것이 모든 Agentic 시스템의 시작점이다. 당신의 시스템도 여기서 출발해야 한다.

**당신의 시스템에 주는 교훈**: Day 1에는 이것만 만들어라:
1. `while(true)` 루프
2. LLM API 호출 (스트리밍)
3. tool_use 감지 → 도구 실행 → 결과 주입
4. 기본 권한 확인 (모든 도구 호출에 사용자 확인)
5. 종료 조건 (tool_use 없음 = 완료)

나머지는 전부 **프로덕션 피드백이 요구할 때** 추가한다.

---

# Part 3: 프로덕션의 현실 — 사용자가 부딪힌 문제들 (2025.05 — 2025.09)

## GA와 동시에 드러난 Pain Points

2025년 5월 22일, Claude Code가 GA(일반 출시)되면서 사용자 수가 폭발적으로 증가했다. Claude 4 모델(SWE-bench 72.5%)과 함께 출시되어 품질은 극적으로 향상되었지만, **규모에서 오는 새로운 문제들**이 드러났다.

### Pain Point #1: "매번 허용 버튼을 눌러야 해요"

**문제**: 10회 도구 호출 = 10번의 "허용하시겠습니까?" 중단. 사용자가 "승인 버튼 누르는 기계"로 전락.

**해법의 진화**:
```
Phase 1 (초기): 모든 도구 호출에 사용자 확인
  → 안전하지만, 사용성 최악

Phase 2 (규칙 학습): "Yes, always allow npm run:*"
  → 사용자의 과거 결정에서 규칙 자동 추출
  → bashPermissions.ts: 접두사 추출, 환경변수 필터링, Heredoc 처리
  → 한 번 허용하면 유사 명령어 자동 승인

Phase 3 (ML 분류기): 200ms Grace Period + 비동기 분류기
  → 사용자가 키보드를 누르기 전에 분류기가 먼저 판단
  → interactiveHandler.ts: 대화상자 + 분류기 레이싱
  → 대부분의 안전한 명령어가 대화상자 없이 실행

Phase 4 (Auto Mode, 2026.03):
  → 서버측 프롬프트 인젝션 탐지 + Sonnet 4.6 기반 트랜스크립트 분류기
  → 0.4% 오탐률로 안전한 도구 자동 승인
  → 경험 많은 사용자의 40%+ 세션이 자동 승인 모드
```

**이 진화가 보여주는 것**: 보안과 사용성은 트레이드오프가 아니다. **시스템이 사용자로부터 학습**하면 둘 다 향상된다. 핵심은 "사용자의 과거 결정을 미래에 재활용"하는 것.

### Pain Point #2: "갑자기 에러가 나요"

**문제**: API 429(Rate Limit), 529(과부하), 413(컨텍스트 초과) 등이 빈번. 사용자는 기술적 세부사항을 알 필요가 없는데, 에러 메시지가 그대로 노출.

**해법의 진화**:
```
Phase 1 (초기): 모든 에러를 사용자에게 표시
  → "API Error 429: Rate limit exceeded" 같은 메시지가 그대로 노출
  → 사용자: "뭘 해야 하죠?" (답: 아무것도. 기다리면 됨)

Phase 2 (재시도 은닉): 처음 3번의 재시도를 숨김
  → withRetry.ts: retryAttempt < 4이면 UI에 전달하지 않음
  → 99%의 일시적 오류가 사용자 모르게 해결

Phase 3 (Error Withholding): 복구 가능한 에러를 보류
  → query.ts: 413이 오면 UI에 전달하지 않고, 컨텍스트 압축 시도
  → 압축 성공 시 에러 폐기 — 사용자는 에러가 있었는지조차 모름
  → "복구에 성공한 에러는 존재하지 않았던 것이다"

Phase 4 (모델 Fallback): Opus 과부하 시 자동 Sonnet 전환
  → 사용자: "좀 느리네" (품질은 유사)
  → 해결 방법이 없는 정보는 불안만 유발하므로, 알리지 않음
```

**이 진화가 보여주는 것**: 에러 처리의 최고 수준은 **에러를 사용자에게 보여주지 않는 것**이다. "이 에러를 사용자에게 보여줬을 때 사용자가 할 수 있는 조치가 있는가?" — 없다면 숨기고 자동 복구하라.

### Pain Point #3: "대화가 길어지면 멍해져요"

**문제**: 20회 도구 호출 후 컨텍스트 윈도우의 75% 소비. 모델이 이전 대화를 "잊어버리는" 것처럼 행동.

**해법의 진화**:
```
Phase 1 (초기): 컨텍스트 초과 시 에러
  → 413 에러 → 사용자에게 "대화를 짧게 유지하세요" 안내
  → 시스템의 한계를 사용자에게 떠넘기는 최악의 설계

Phase 2 (수동 압축): /compact 명령어 추가
  → 사용자가 직접 대화 요약을 트리거
  → 여전히 사용자가 "언제 압축해야 하는지" 판단해야 함

Phase 3 (자동 압축): 87% 임계값에서 자동 트리거
  → autoCompact.ts: effectiveWindow의 87% 도달 시 Forked Agent가 요약
  → 사용자는 아무것도 모름 — "이 시스템은 절대 잊어버리지 않네"

Phase 4 (4단계 계층적 압축):
  → Snip(0ms) → Microcompact(<1ms) → Auto-compact → Reactive compact
  → 저비용부터 시도, 고비용은 최후 수단
  → Circuit Breaker: 연속 3회 실패 시 재시도 중단
  → Post-compact 복원: 스킬 5개 + 파일 5개 자동 재주입
```

**이 진화가 보여주는 것**: "사용자에게 시스템의 한계를 관리하라고 요청하는 것"은 설계 실패다. 컨텍스트 윈도우 관리는 100% 하네스의 책임이다.

## "Context Engineering" 이론의 탄생 (2025.09)

이 시기에 Anthropic은 프로덕션 경험을 바탕으로 **3개의 핵심 블로그 포스트**를 동시에 발표했다:

### 1. "Effective Context Engineering for AI Agents"

**핵심 통찰: "주의력 예산(Attention Budget)"**

> "컨텍스트 윈도우의 모든 토큰은 모델의 '주의력 예산'을 소비한다. 최소한의 고신호 토큰으로 원하는 결과의 가능성을 최대화하는 것이 핵심이다."

이것은 단순한 "토큰 절약" 이상의 통찰이다. **정보를 추가하면 비용만 증가하는 게 아니라, 기존 정보의 효과도 감소**한다 (Context Rot). 따라서 컨텍스트 엔지니어링은 "무엇을 넣을 것인가"만큼 "무엇을 빼낼 것인가"가 중요하다.

5가지 전략이 제시되었다:

| 전략 | 목적 | Claude Code 구현 |
|------|------|----------------|
| 프로젝트 파일 | 세션 시작 시 핵심 맥락 | CLAUDE.md 계층, Git 상태 |
| Just-In-Time 검색 | 필요할 때만 로드 | Grep/Glob/Read 도구 |
| Compaction | 히스토리 압축 | 4단계 자동 압축 |
| 구조화된 메모 | 외부 메모리 | memdir/ (MEMORY.md) |
| Sub-Agent 격리 | 깨끗한 컨텍스트 | Forked Agent (CacheSafeParams) |

### 2. "Building Agents with the Claude Agent SDK"

**핵심 통찰: 4단계 Agentic Loop**

```
Gather Context → Take Action → Verify Work → Loop
(맥락 수집)      (행동)         (검증)         (반복)
```

이 4단계 모델은 이전의 단순한 "API 호출 → 도구 실행 → 반복"보다 진화한 것이다. **검증(Verify)** 단계가 추가되었다. 이것이 Stop Hook과 Evaluator 패턴의 이론적 근거가 된다.

### 2b. "The Think Tool" — 54% 추론 품질 향상 (2025.03)

이 시기에 발표된 가장 실용적인 패턴 중 하나. **부작용이 없는 "생각" 도구**를 추가하는 것만으로 복잡한 순차 도구 호출의 품질이 극적으로 향상된다.

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": { "type": "string" }
    },
    "required": ["thought"]
  }
}
```

**벤치마크**: Think + 최적화 프롬프트 = **0.584** vs 기본 0.332 (**54% 상대적 향상**). SWE-bench에서도 1.6% 향상 (p < .001, d = 1.47).

**핵심**: Extended Thinking은 첫 응답 전에만 작동하지만, Think Tool은 **도구 호출 사이사이에** 추론을 가능하게 한다. 정책 준수가 중요한 Enterprise 에이전트에 필수적이다.

**당신의 시스템에 적용**: 구현은 JSON 5줄이지만 효과는 54%. 모든 Agentic 시스템에 Think Tool을 추가하라. 도메인 복잡도에 비례하여 프롬프트에 "모든 행동 전에 think를 사용하여 적용 규칙을 나열하라"를 추가하라.

### 2c. "Writing Effective Tools for Agents" — 도구 설계의 7가지 원칙

Anthropic이 강조한 "도구 설계는 프롬프트 엔지니어링만큼 중요하다"의 실체:

| 원칙 | 구체적 지침 |
|------|-----------|
| **3-5개 고영향 도구에 집중** | 모든 API 엔드포인트를 래핑하지 말라. 다중 단계를 통합한 도구 |
| **시맨틱 응답** | UUID → 의미 있는 이름으로. "검색 정밀도가 크게 향상" |
| **response_format 파라미터** | `"detailed"` (206토큰) vs `"concise"` (72토큰) 에이전트 제어 |
| **토큰 효율** | 기본 25,000 토큰 응답 제한. 잘린 결과는 "왜 + 더 나은 검색 전략 제안" |
| **행동 가능한 에러** | "Error 403" → "admin 역할 필요. get_user_roles로 확인하세요" |
| **네임스페이싱** | 접두사 vs 접미사 선택이 성능을 측정 가능하게 변화시킨다 |
| **평가 루프** | 4가지 메트릭: 정확도, 실행시간, 토큰 소비, 에러율 |

### 3. Claude Code의 실제 변경사항 (2025.09.29)

같은 날 Claude Code에 추가된 기능들:

| 기능 | 해결한 Pain Point | 구현 패턴 |
|------|-----------------|----------|
| **Checkpoint** | "Claude가 코드를 망쳤어" | 매 어시스턴트 메시지마다 자동 스냅샷 + Esc-Esc 롤백 |
| **Subagents** | "한 번에 한 가지만 해" | 병렬 개발을 위한 독립 컨텍스트 에이전트 |
| **Hooks** | "매번 린터를 수동으로 돌려야 해" | PreToolUse/PostToolUse/Stop 자동화 |
| **Background Tasks** | "장시간 작업을 기다려야 해" | GitHub Actions 기반 비동기 실행 |

**당신의 시스템에 주는 교훈**: 이 시점이 "최소 하네스"에서 "성숙한 하네스"로의 전환점이다. 프로덕션에서 사용자의 Pain Point를 수집하고, 그 Pain Point를 시스템으로 흡수하는 기능을 추가하라. 핵심 순서:

```
1. 체크포인트 (되돌리기 가능성)     ← 안전망
2. 자동 컨텍스트 압축              ← 성능 유지
3. 에러 자동 복구                 ← 안정성
4. 규칙 학습 (권한)               ← 마찰 감소
5. 서브에이전트                   ← 병렬성
6. Hooks                        ← 확장성
```

---

# Part 4: 하네스의 성숙 — 시스템이 복잡성을 흡수하는 방법 (2025.09 — 2025.12)

## 하네스의 7개 서브시스템

프로덕션 피드백에 의해 진화한 Claude Code의 하네스는, 최종적으로 7개의 서브시스템으로 구성된다:

```
┌──────────────────────────────────────────────────────┐
│                    HARNESS                             │
│                                                      │
│  ① 입력 처리    사용자 텍스트 → API 호출 형식 변환       │
│  ② Agentic Loop while(true) 상태 머신                  │
│  ③ 도구 실행    배치, 동시성, 스트리밍 오버랩             │
│  ④ 컨텍스트     4단계 압축, KV-Cache 보존               │
│  ⑤ 오류 복구    Error Withholding, Cascade Recovery    │
│  ⑥ 체크포인트   세션/파일/비용 영속화                    │
│  ⑦ 종료 조건    Stop Hook, Token Budget, Graceful Exit │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 각 서브시스템의 핵심 숫자들

| 서브시스템 | 핵심 임계값/상수 | 의미 |
|-----------|----------------|------|
| 입력 처리 | CLAUDE.md 40KB 상한 | 프로젝트 규약의 적정 크기 |
| Agentic Loop | 7 Continue + 11 Exit | 상태 전이의 완전한 열거 |
| 도구 실행 | CONCURRENCY_MAX = 10 | 동시 도구 실행 상한 |
| 컨텍스트 | 87% Auto-compact 임계값 | 능동적 압축 트리거 |
| 오류 복구 | 3회 재시도 은닉 | 사용자가 모르는 복구 횟수 |
| 체크포인트 | 100 스냅샷 상한 | 파일 히스토리 보관 수 |
| 종료 조건 | Recovery 최대 3회 | 출력 토큰 복구 시도 제한 |

### "Effective Harnesses for Long-Running Agents" (2025.11.26)

Anthropic이 프로덕션 경험을 정리한 하네스 설계 가이드:

**핵심 패턴: Initializer-Executor**

```
세션 시작:
  Initializer Agent (1회 실행)
  ├─ 레포지토리 구조 분석
  ├─ 200+ 속성의 JSON 명세 생성
  │   (실패 모드, 관련 파일, 테스트 전략, 엣지 케이스...)
  └─ Executor Agent에 전달

이후:
  Executor Agent (N회 반복)
  ├─ 하나의 기능만 구현 (한 세션에 한 기능)
  ├─ 설명적 Git 커밋
  └─ 브라우저 자동화 테스트로 검증
```

**"한 세션에 한 기능" 원칙의 이유**:
- 여러 기능을 한 세션에 구현하면 컨텍스트가 소진됨
- 하나의 기능이 실패하면 다른 기능까지 영향
- Git 커밋 단위가 명확해야 롤백이 가능

**당신의 시스템에 주는 교훈**: 장시간 실행 에이전트에는 다음이 필수:
1. **진행 파일** — 에이전트가 매 작업 후 진행 상황을 파일에 기록
2. **Git-as-Checkpoint** — 의미 있는 작업 단위마다 설명적 커밋
3. **세션 시작 루틴** — 진행 파일 읽기 → Git 로그 확인 → 테스트 실행 → 기존 이슈 수정
4. **한 세션에 한 기능** — 컨텍스트 소진과 장애 전파 방지

## 다중 에이전트 시스템의 교훈 (2025.06)

"How We Built Our Multi-Agent Research System"에서 Anthropic이 공유한 데이터:

```
단일 에이전트 (Opus 4) vs 멀티 에이전트 (Opus 4 Lead + Sonnet 4 Workers)

  성능: 멀티 에이전트가 90.2% 우수
  토큰: 멀티 에이전트가 ~15x 더 사용
  비용: 멀티 에이전트가 훨씬 비쌈
  
  핵심 발견:
  "토큰 사용량이 성능 분산의 80%를 설명한다"
  → 더 많이 읽고, 더 많이 생각할수록 더 나은 결과
  → 비용이 허용하는 범위 내에서 토큰을 아끼지 말라
```

이 발견이 Claude Code의 Forked Agent 아키텍처에 반영되었다:

```
CacheSafeParams = { 시스템 프롬프트, 도구, 모델, thinking, 메시지 접두사 }

부모 에이전트                    자식 에이전트 (Forked)
  ├─ 시스템 프롬프트 ──────→     동일 (캐시 HIT)
  ├─ 도구 목록 ──────────→      동일 (캐시 HIT)  
  ├─ 모델 ──────────────→      동일 (캐시 HIT)
  └─ 메시지 접두사 ─────→       동일 (캐시 HIT)
  
  → 자식의 입력 비용 ~90% 절감
  → "토큰을 아끼지 말라"와 "비용을 통제하라"의 양립
```

---

# Part 5: 생태계 확장 — 확장성과 표준화 (2025.10 — 2025.12)

## 플러그인 시스템 (2025.10.09)

Claude Code가 "도구"에서 "플랫폼"으로 전환한 순간:

```
Plugin = Skills + Hooks + MCP Servers + Agents

예시: "React 최적화 플러그인"
  ├─ Skill: /optimize-bundle (번들 크기 분석 워크플로우)
  ├─ Hook: PostToolUse에서 React lint 자동 실행
  ├─ MCP Server: 성능 메트릭 대시보드 연결
  └─ Agent: 성능 분석 전문 서브에이전트

설치: 한 명령어
공유: Git 레포지토리
```

## $1B 매출과 Bun 인수 (2025.12.03)

GA 후 6개월 만에 연간 $1B 매출 달성. 동시에 JavaScript 런타임 **Bun**을 인수.

**왜 Bun인가**: Claude Code는 TypeScript로 구축되어 있으며, Bun의 `feature()` API가 Dead Code Elimination의 핵심이다:

```typescript
// 빌드 시점에 비활성 코드 완전 제거
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
// VOICE_MODE=false이면 voice 모듈 전체가 바이너리에서 제거
// → 바이너리 크기 최적화 + 공격 표면 감소
```

## Agent Skills: 에이전트에게 전문성을 부여하는 개방형 표준 (2025.10.16)

Skills는 Claude Code 확장 아키텍처에서 **가장 과소평가된 혁신**이다. Plugin이 "배포 패키지"라면, Skills는 "절차적 지식"이다.

### Skills란 무엇인가

> Agent Skills는 에이전트가 동적으로 발견하고 로드할 수 있는 **지시사항, 스크립트, 리소스의 조직화된 폴더**이다. 일반 목적 에이전트를 각 사용 사례마다 별도의 커스텀 에이전트를 만들지 않고도 **전문화된 시스템으로 변환**한다.

```
skill-name/
  SKILL.md           # 필수: 메타데이터 + 지시사항
  scripts/           # 선택: 실행 가능한 코드
  references/        # 선택: 참조 문서
  assets/            # 선택: 템플릿, 리소스
```

### 핵심 아키텍처: 3단계 Progressive Disclosure

Skills가 컨텍스트 효율적인 이유:

```
Level 1 — 메타데이터 (~100 토큰)
  세션 시작 시 모든 스킬의 name + description만 시스템 프롬프트에 로드
  에이전트는 어떤 스킬이 있는지 "알지만" 내용은 모름

Level 2 — 핵심 지시사항 (<5,000 토큰 권장)
  작업이 스킬의 description과 매칭되면 SKILL.md 전체를 컨텍스트에 로드
  이 시점에서야 에이전트가 구체적인 절차를 얻음

Level 3 — 상세 자료 (무제한)
  실행 중 필요할 때만 references/, scripts/ 로드
  예: PDF 스킬의 forms.md는 양식 작성할 때만 로드

→ "스킬에 번들링할 수 있는 컨텍스트의 양은 사실상 무제한"
→ 주의력 예산을 소비하지 않으면서 전문 지식 제공
```

### Skills vs MCP vs Hooks vs Plugins

| 구성요소 | 역할 | 비유 |
|---------|------|------|
| **MCP** | "어떤 도구를 호출할 수 있는가" | 시스템 콜 인터페이스 |
| **Skills** | "그 도구들로 이 작업을 어떻게 수행하는가" | 절차 매뉴얼 |
| **Hooks** | "언제 자동으로 특정 로직을 실행하는가" | 이벤트 트리거 |
| **Plugins** | "위 모든 것을 하나로 패키징" | 설치 패키지 |

### 개방형 표준: agentskills.io (2025.12.18)

Anthropic은 Skills 형식을 **개방형 표준**으로 발표했다. MCP와 동일한 전략: 독점이 아닌 업계 표준화.

**채택 현황 (33+ 클라이언트)**:
- Anthropic (Claude Code, Claude.ai)
- Microsoft (GitHub Copilot, VS Code)
- OpenAI (Codex)
- Google (Gemini CLI)
- Cursor, JetBrains Junie, AWS Kiro
- ByteDance TRAE, Mistral AI Vibe, 등

**당신의 시스템에 주는 교훈**: MCP가 "도구 연결"을 표준화했듯, Skills는 "절차적 지식"을 표준화했다. 자체 프롬프트 템플릿 시스템을 만들지 말고, SKILL.md 표준을 채택하면 33+ 에이전트 플랫폼과 호환되는 지식 베이스를 구축할 수 있다.

### Skills 2.0 / Skill-Creator 2.0 (2026.03.03)

Skills 1.0의 핵심 문제: **"스킬이 제대로 작동하는지 시행착오 외에는 알 방법이 없었다."**

Skills 2.0이 추가한 3가지:

**1. Evals (자동 평가)**
```
동일 프롬프트를 "스킬 ON" vs "스킬 OFF"로 실행
  → 결과를 나란히 비교
  → 스킬 없이도 동일 품질이면 → 그 스킬은 불필요
```

**2. Blind A/B 테스트**
```
독립적인 Claude 인스턴스가 두 결과를 비교
  → 어느 버전인지 모른 채 승자 선택 + 이유 설명
  → 스킬 버전 비교, 모델 비교 가능
```

**3. Description 최적화**
```
~20회 반복 × 최대 5라운드:
  1. 합성 프롬프트 20개 생성 (절반은 트리거, 절반은 비트리거)
  2. 60/40 train/test 분할
  3. 현재 description을 프롬프트당 3회 테스트 (2/3 다수결)
  4. 놓친/잘못된 트리거 분석
  5. description 재작성 (이전 시도 참고)
  6. 테스트 점수로 승자 선택 (과적합 방지)

Anthropic 내부 6개 빌트인 스킬 중 5개에서 트리거링 개선
```

**Skills 2.0의 핵심 통찰**:
> "트리거링 신뢰성은 지시사항 품질만큼 중요하다. 훌륭한 지시사항이 있어도 나쁜 description을 가진 스킬은 필요할 때 로드되지 않는 스킬이다."

**스킬 관리 권장사항**:
- SKILL.md는 500줄 / ~5,000단어 이하 유지
- 결정론적 검증은 scripts/에 코드로 (산문 해석 대신 pass/fail)
- 상세 문서는 references/에 (on-demand 로드)
- 활성 스킬 20-50개 초과 시 성능 저하 (모든 description이 매 메시지에 로드)
- 부정 트리거 추가: "단순 데이터 조회에는 사용하지 말 것"

---

## Advanced Tool Use: 도구 발견의 혁신 (2025.11.24)

Skills가 "절차적 지식"의 표준이라면, **Tool Search**와 **Programmatic Tool Calling**은 "도구 발견과 사용"의 혁신이다.

### Tool Search (토큰 85% 절감)

```
이전: 40+ 도구를 모두 시스템 프롬프트에 포함
  → 토큰 소비 증가 + 모델 혼란 (선택지 과다)

이후: 핵심 도구만 상시 로드 + ToolSearch 도구 제공
  → 모델이 필요한 도구를 ToolSearch로 검색
  → 검색된 도구만 컨텍스트에 주입
  → 토큰 85% 절감, 도구 선택 품질 향상
```

### Programmatic Tool Calling (토큰 37% 절감)

모델이 도구를 "설명 기반"이 아닌 "코드 기반"으로 호출:
```
이전: 도구 정의 전체를 JSON Schema로 프롬프트에 포함
이후: 코드 실행 도구가 MCP를 통해 프로그래밍 방식으로 도구 호출
→ 컨텍스트 소비 37% 절감
```

---

## Claude Code의 Multi-Surface 확장 (2025.10 — 2026.02)

Claude Code는 터미널 CLI에서 시작했지만, 여러 인터페이스로 확장되었다:

```
┌──────────────────────────────────────────────────────────┐
│          Claude Code Multi-Surface Architecture           │
│                                                          │
│  Terminal CLI (원본)                                      │
│  ├─ 가장 완전한 기능 세트                                 │
│  └─ 개발자 워크플로우의 중심                               │
│                                                          │
│  VS Code Extension (2025.05 베타)                        │
│  ├─ 에디터 내 인라인 diff 리뷰                            │
│  └─ 사이드바 통합                                        │
│                                                          │
│  JetBrains Plugin (2025.05 베타)                         │
│  └─ IntelliJ, WebStorm 등 지원                          │
│                                                          │
│  Desktop App (2025.11)                                   │
│  ├─ 독립 실행형 (macOS/Windows)                           │
│  ├─ 시각적 diff 리뷰, 병렬 세션                           │
│  ├─ Dispatch: 폰 → 데스크톱 태스크 라우팅                 │
│  └─ Git 격리 내장                                        │
│                                                          │
│  Web App (2025.10, claude.ai/code)                       │
│  ├─ 브라우저 기반, Anthropic 호스팅                       │
│  ├─ 격리된 클라우드 환경에서 실행                          │
│  ├─ 로컬 설치 불필요                                     │
│  └─ 모바일 접근 가능                                     │
│                                                          │
│  Apple Xcode (2026.02.03)                                │
│  └─ Xcode 26.3 네이티브 통합                             │
│                                                          │
│  세션 이동 기능:                                          │
│  ├─ /teleport: 웹 → 터미널 핸드오프                      │
│  ├─ /desktop: 터미널 → 데스크톱 핸드오프                  │
│  ├─ Remote Control: 폰/브라우저 세션 핸드오프              │
│  └─ Dispatch: 어디서든 데스크톱으로 태스크 전송             │
│                                                          │
│  자동화:                                                 │
│  ├─ Scheduled Tasks (클라우드): Anthropic 인프라에서 cron  │
│  ├─ Scheduled Tasks (데스크톱): 로컬 머신에서 cron         │
│  ├─ /loop: 세션 내 주기적 실행                            │
│  ├─ Channels: Telegram, Discord, Webhook 통합             │
│  └─ GitHub/Slack: PR 자동 리뷰, Slack에서 태스크 수신      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Claude Code Sandboxing: OS 수준 보안 (2025.10.20)

"Beyond Permission Prompts" 엔지니어링 포스트에서 공개한 보안 아키텍처:

```
OS 수준 샌드박싱:
  Linux: bubblewrap (bwrap) — 네임스페이스 격리
  macOS: sandbox-exec (seatbelt) — 프로파일 기반 제한

효과: Permission 프롬프트 84% 감소
  이전: 파일 쓰기마다 "허용하시겠습니까?"
  이후: 프로젝트 디렉토리 내 쓰기는 샌드박스가 보장 → 자동 허용
       프로젝트 외부 쓰기는 샌드박스가 차단 → 권한 불필요 (불가능)

→ 보안을 강화하면서 동시에 마찰을 줄인 사례
→ Pain Point Transfer의 "Invisible Safety" 패턴의 실체
```

---

## Multi-Agent 실전: C 컴파일러 구축 사례 (2026.02.05)

"Building a C Compiler with Parallel Claudes" — 현재까지 공개된 가장 상세한 multi-agent 프로젝트 사례:

```
규모: 16개 병렬 에이전트, 2,000 세션, $20K, 100K줄 Rust 코드
결과: 완전한 C 컴파일러 (x86-64 코드 생성)

하네스 구조:
  ├─ Coordinator: 작업 분해 + 할당
  ├─ 16 Workers: 각각 독립된 Git worktree에서 작업
  ├─ 통합: Worker 결과를 Coordinator가 합성
  └─ 검증: 테스트 스위트 자동 실행

비용 통제:
  ├─ CacheSafeParams로 캐시 공유
  ├─ Worker별 Task Budget ($1.25/worker 평균)
  └─ 총 $20K (16 workers × 2000 sessions)
```

**당신의 시스템에 주는 교훈**: 이 사례는 "Multi-Agent가 프로덕션에서 작동할 수 있다"를 증명한다. 핵심 성공 요인: (1) 명확한 작업 분해, (2) Git worktree 격리, (3) Worker별 예산 통제.

---

## AAIF 설립과 MCP 기부 (2025.12.09)

Anthropic, OpenAI, Google, Microsoft, AWS가 공동으로 **Agentic AI Foundation**을 설립하고, Anthropic이 MCP를 기부했다.

**의미**: MCP가 한 회사의 오픈소스 프로젝트에서 **업계 거버넌스 하의 표준**으로 승격. 이는 Claude Code의 확장성 레이어가 독점적 기술이 아닌 지속 가능한 표준임을 보장한다.

---

# Part 6: 자율성의 시대 — 신뢰를 코드로 설계하기 (2026.01 — 2026.04)

## 1M 컨텍스트의 시대 (2026.02)

Claude Opus 4.6 (2026.02.05)과 Sonnet 4.6 (2026.02.17)이 **1M 토큰 컨텍스트 윈도우**와 함께 출시되었다.

**컨텍스트 윈도우 확장이 하네스 설계에 미치는 영향**:

```
200K 시대:                          1M 시대:
  컨텍스트 압축이 생존의 문제          컨텍스트 압축이 최적화의 문제
  매 20회 도구 호출마다 압축 필요       100회 도구 호출도 압축 없이 가능
  서브에이전트 필수 (컨텍스트 격리)     단일 에이전트로도 복잡한 작업 가능
  
  하지만 여전히 필요한 것:
  ├─ KV-Cache 최적화 (비용의 핵심)
  ├─ 결과 크기 제한 (100KB × 100 = 10MB → 여전히 비쌈)
  └─ 주의력 예산 관리 (1M 토큰이라도 관련 없는 정보는 노이즈)
```

**Anthropic의 관찰** ("Harness Design for Long-Running Apps", 2026.03):

> "Claude Opus 4.6의 등장으로 하네스 복잡성의 필요성이 줄어들고 있다. 점점 더 많은 경우에 외부 컨텍스트 메커니즘 없이도 200K+ 토큰 윈도우 내에서 작업을 유지할 수 있다. 하네스 컴포넌트는 모델이 개선됨에 따라 지속적으로 재평가해야 한다."

## 자율성 측정 (2026.02.18)

"Measuring AI Agent Autonomy in Practice" — 500,000+ 세션, 998,481 API 도구 호출의 데이터:

```
핵심 발견들:

1. 자율 작업 시간의 증가
   99.9th percentile: 25분 미만 → 45분 이상 (4개월 만에 거의 2배)

2. 경험에 따른 행동 변화
   신규 사용자: ~20% 세션을 자동 승인
   경험 사용자 (750+ 세션): 40%+ 자동 승인
   
   그러나 동시에:
   경험 사용자의 개입 빈도: 5% → 9% (증가!)
   
   → "더 많이 맡기면서, 더 전략적으로 개입한다"
   → 이것이 "점진적 신뢰 구축"의 데이터적 증거

3. 안전성 지표
   ├─ 도구 호출의 80%에 안전장치 포함
   ├─ 73%가 어떤 형태로든 사람 개입 유지
   ├─ 불가역 작업은 전체의 0.8%에 불과
   └─ 에이전트가 사람보다 2배 자주 확인 질문을 함

4. 소프트웨어 엔지니어링이 ~50%의 에이전트 활동 차지
```

**Auto Mode (2026.03.25)의 기술적 구현**:

```
2계층 안전 시스템:

Layer 1 (서버측): 프롬프트 인젝션 탐지
  └─ 입력에 대한 사전 검사

Layer 2 (클라이언트측): 트랜스크립트 분류기
  ├─ Stage 1: 단일 토큰 고속 필터 (대부분의 안전한 호출 즉시 승인)
  └─ Stage 2: Chain-of-Thought 분석 (불확실한 경우 상세 검토)

3계층 권한:
  ├─ 안전 도구 (Read, Grep, Glob) → 자동 허용
  ├─ 프로젝트 내 파일 작업 (Edit, Write) → 자동 허용
  └─ 나머지 (Bash, 외부 통신) → 분류기 판단

차단 대상:
  ├─ 파괴적 작업 (rm -rf, 데이터 삭제)
  ├─ 보안 수준 하락 (권한 변경, 인증 우회)
  ├─ 신뢰 경계 위반 (자격증명 노출)
  └─ 공유 인프라 우회 (CI/CD 변경)

성능: 0.4% 오탐률, 5.7% 미탐률 (합성 데이터 유출 기준)
```

**당신의 시스템에 주는 교훈**: 자율성은 on/off가 아니라 스펙트럼이다.

```
Permission Mode 스펙트럼 (Claude Code):

  default     plan     acceptEdits    auto    bypassPermissions
  ────────────────────────────────────────────────────────────→
  모든 것 확인   계획 후 실행  편집만 자동    ML 판단   모든 것 자동
  
  사용자가 이 스펙트럼 위에서 자신의 위치를 선택하게 하라.
  시스템은 각 위치에서 가능한 최대의 안전성을 제공하라.
```

## 검증 분리의 중요성 (2026.03.24)

"Harness Design for Long-Running Application Development"에서 발견한 핵심:

**자기 평가 편향(Self-Evaluation Bias)**:

> "에이전트는 자신의 평범한 결과를 자신있게 칭찬한다. 생성과 평가를 분리해야 한다."

**해법: Planner-Generator-Evaluator 아키텍처**:

```
Planner Agent
  └─ 사용자 프롬프트 → Sprint Contract (완료 조건 정의)

Generator Agent
  └─ Sprint Contract에 따라 코드 생성
  └─ 완료 선언

Evaluator Agent (별도 컨텍스트)
  └─ Playwright로 실제 사용자처럼 테스트
  └─ 통과? → 완료
  └─ 실패? → 구체적 피드백 → Generator로 돌아감
```

**비용 vs 품질 데이터**:

| 접근법 | 시간 | 비용 | 결과 |
|--------|------|------|------|
| 단독 에이전트 | 20분 | $9 | 기능 불완전 |
| 전체 하네스 (3-Agent) | 6시간 | $200 | 전체 기능 완성 |

---

# Part 7: 종합 — 당신의 시스템에 적용하기

## Claude Code 진화에서 추출한 10가지 설계 원칙

### 원칙 1: 최소 하네스로 시작하라

```
Day 1: while(true) + tool_use + 기본 권한
Day N: 프로덕션 피드백이 요구하는 것만 추가
```

### 원칙 2: Pain Point를 시스템으로 옮겨라

모든 사용자 마찰에 대해: "이것을 시스템이 대신 처리할 수 있는가?"

```
5가지 Pain Point Transfer 패턴:
  1. Silent Success — 성공한 작업은 보이지 않게
  2. Progressive Learning — 과거 결정에서 미래를 예측
  3. Time Overlap — 대기 시간을 작업 시간으로 변환
  4. Information Layering — 정보를 계층적으로 공개
  5. Invisible Safety — 보안의 대부분을 보이지 않게
```

### 원칙 3: 에러 처리를 계층적으로 설계하라

```
저비용 복구 → 고비용 복구 → 사용자 표출
Guard 플래그로 무한 루프 방지
Death Spiral 방지 (에러 처리가 에러를 유발하지 않도록)
```

### 원칙 4: 컨텍스트는 가장 비싼 자원이다

```
추가하는 모든 토큰이 주의력 예산을 소비한다.
KV-Cache 보존이 비용의 핵심이다.
압축은 4단계로: Snip → Microcompact → Auto-compact → Reactive compact
```

### 원칙 5: 캐시 공유가 Multi-Agent 비용을 해결한다

```
CacheSafeParams: 부모와 자식이 동일한 시스템 프롬프트/도구/모델을 공유
→ API Prompt Cache 재사용
→ 입력 비용 ~90% 절감
```

### 원칙 6: 검증을 생성과 분리하라

```
자기 평가 편향: 에이전트는 자신의 결과를 과대평가
해법: Stop Hook, 별도 Evaluator Agent, 외부 테스트
검증 루프만 추가해도 품질 2-3배 향상 (Anthropic 데이터)
```

### 원칙 7: 자율성은 스펙트럼이다

```
모든 것 확인 → 규칙 학습 → ML 분류기 → 완전 자율
사용자가 스펙트럼 위에서 자신의 위치를 선택하게 하라
경험이 쌓이면 자연스럽게 자율성이 증가한다 (20% → 40%)
```

### 원칙 8: 파일시스템이 메모리다

```
벡터 DB가 아닌 파일과 Git을 사용하라
├─ 검사 가능, 편집 가능, 버전 관리 가능
├─ Git: 원자적 커밋, 분기, 병합, 롤백
└─ 에이전트의 상태는 항상 검사 가능해야 한다
```

### 원칙 9: 프로토콜 표준을 채택하라

```
MCP: 에이전트-도구 연결 (Linux Foundation AAIF 거버넌스)
A2A: 에이전트-에이전트 통신
OpenTelemetry: 관찰가능성
자체 프로토콜은 생태계에서 고립된다
```

### 원칙 10: 하네스는 지속적으로 재평가하라

```
"하네스 컴포넌트는 모델이 개선됨에 따라 지속적으로 재평가해야 한다"
200K 시대의 필수 기능이 1M 시대에는 불필요할 수 있다
복잡성은 비용이다 — 필요 없어진 복잡성은 제거하라
```

## 구축 로드맵

```
Phase 1: Foundation (4-6주)
  □ while(true) Agentic Loop
  □ 3-5개 핵심 도구 (검색, 읽기, 쓰기, 실행)
  □ 기본 권한 (모든 도구에 사용자 확인)
  □ 스트리밍 응답 출력

Phase 2: Resilience (4-6주)
  □ 컨텍스트 압축 (자동, 최소 2단계)
  □ Error Withholding + Cascade Recovery
  □ API 재시도 (은닉 + Foreground/Background 분리)
  □ 비용 추적

Phase 3: Scale (6-8주)
  □ Sub-Agent (CacheSafeParams 캐시 공유)
  □ MCP 클라이언트 통합
  □ Streaming Tool Executor
  □ 권한 학습 (규칙 추출)

Phase 4: Autonomy (지속적)
  □ ML 기반 자동 승인
  □ Plugin/Skill 확장 시스템
  □ 검증 분리 (Stop Hook / Evaluator)
  □ 관찰가능성 (OpenTelemetry)
```

---

## 부록: Claude Code 진화 연대기

| 시점 | 이벤트 | 의미 |
|------|--------|------|
| 2024.03 | Claude 3 출시 | 200K 컨텍스트, 코딩 에이전트의 기반 |
| 2024.06 | Claude 3.5 Sonnet | 코딩 능력의 임계점 돌파 |
| 2024.10 | 3.5 Sonnet 업그레이드 + Computer Use | SWE-bench 49%, 에이전트 야망 선언 |
| 2024.11 | MCP 오픈소스 | 도구 연결 표준화 |
| 2024.12 | "Building Effective Agents" | 설계 철학의 원점 |
| **2025.02** | **Claude Code 연구 프리뷰** | **탄생** |
| **2025.05** | **GA + Claude 4 + SDK** | **프로덕션 전환** |
| 2025.06 | Multi-Agent Research System | 90.2% 성능 향상 데이터 |
| **2025.09** | **Context Engineering + Checkpoints + Hooks** | **하네스 성숙** |
| **2025.10** | **Agent Skills + Plugins + Web App + Sandboxing** | **플랫폼 전환 + 보안 + 전문화** |
| 2025.11 | Opus 4.5 + Effective Harnesses + Tool Search | 장시간 에이전트 + 도구 발견 혁신 |
| 2025.12 | $1B + AAIF + Bun + Skills 오픈 표준 + Desktop App | 생태계 확립 |
| 2026.01 | Evals 가이드 + Context Compaction API + Cowork | 평가 프레임워크 + 클라우드 압축 |
| 2026.02 | Opus/Sonnet 4.6 (1M) + 자율성 측정 + C컴파일러(16 Agent) | 자율성 + Multi-Agent 실증 |
| 2026.03 | Auto Mode + Skills 2.0 + Planner/Generator/Evaluator | 신뢰의 코드화 + 스킬 평가 |

---

## 부록: 참고 자료

**Anthropic 블로그 (시간순)**
1. Building Effective Agents (2024.12.19)
2. Claude 3.7 Sonnet and Claude Code (2025.02.24)
3. Introducing Claude 4 + Claude Code GA (2025.05.22)
4. How We Built Our Multi-Agent Research System (2025.06.13)
5. Effective Context Engineering for AI Agents (2025.09.29)
6. Building Agents with the Claude Agent SDK (2025.09.29)
7. Enabling Claude Code to Work More Autonomously (2025.09.29)
8. Customize Claude Code with Plugins (2025.10.09)
9. Introducing Claude Opus 4.5 (2025.11.24)
10. Effective Harnesses for Long-Running Agents (2025.11.26)
11. Anthropic Acquires Bun / $1B Milestone (2025.12.03)
12. AAIF / MCP Donation (2025.12.09)
13. Introducing Agent Skills (2025.12.18)
14. Demystifying Evals for AI Agents (2026.01.09)
15. Introducing Claude Opus 4.6 (2026.02.05)
16. Introducing Claude Sonnet 4.6 (2026.02.17)
17. Measuring AI Agent Autonomy in Practice (2026.02.18)
18. Harness Design for Long-Running Application Development (2026.03.24)
19. Claude Code Auto Mode (2026.03.25)

**코드베이스 분석 기반 보고서 (본 프로젝트)**
- Enterprise_Agentic_LLM_Systems_Guide.md — 아키텍처 백서
- Pain_Point_Transfer_Report.md — UX 설계 보고서
- Agentic_Harness_Deep_Dive.md — 하네스 설계 가이드

---

*Claude Code로 배우는 Enterprise Agentic LLM System 구축의 기술 v1.0*
