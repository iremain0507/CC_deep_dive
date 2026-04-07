# Agentic LLM 하네스 심층 분석

## 사용자 입력부터 에이전트 제어까지: 하네스가 LLM Agent를 운영하는 방법

> **핵심 공식**: `Agent = Model + Harness`
>
> 모델은 지능을 제공하고, **하네스는 유용성을 제공한다.**
> 모델이 CPU라면, 하네스는 운영체제다.
>
> **참조 구현체**: Claude Code CLI (query.ts: 1,729줄의 Agentic Loop 하네스)
>
> **대상 독자**: Enterprise Agentic LLM 시스템을 설계하는 AI 엔지니어

---

## 목차

1. [하네스란 무엇인가](#1-하네스란-무엇인가)
2. [하네스의 해부학: 7개 핵심 서브시스템](#2-하네스의-해부학-7개-핵심-서브시스템)
3. [사용자 입력부터 에이전트 제어까지: 전체 흐름도](#3-사용자-입력부터-에이전트-제어까지-전체-흐름도)
4. [서브시스템 1: 입력 처리 하네스](#4-서브시스템-1-입력-처리-하네스)
5. [서브시스템 2: Agentic Loop 하네스](#5-서브시스템-2-agentic-loop-하네스)
6. [서브시스템 3: 도구 실행 하네스](#6-서브시스템-3-도구-실행-하네스)
7. [서브시스템 4: 컨텍스트 관리 하네스](#7-서브시스템-4-컨텍스트-관리-하네스)
8. [서브시스템 5: 오류 복구 하네스](#8-서브시스템-5-오류-복구-하네스)
9. [서브시스템 6: 체크포인트 하네스](#9-서브시스템-6-체크포인트-하네스)
10. [서브시스템 7: 종료 조건 하네스](#10-서브시스템-7-종료-조건-하네스)
11. [Sub-Agent 하네스: 하네스 안의 하네스](#11-sub-agent-하네스-하네스-안의-하네스)
12. [Production 하네스 설계 패턴](#12-production-하네스-설계-패턴)
13. [하네스 설계 체크리스트](#13-하네스-설계-체크리스트)

---

## 1. 하네스란 무엇인가

### 1.1 정의

Anthropic의 공식 기술 블로그("Effective Harnesses for Long-Running Agents")에 따르면:

> **에이전트 하네스(Agent Harness)**는 AI 모델의 실제 추론을 제외한, 에이전트를 운영하기 위한 **모든 소프트웨어 인프라**다. 컨텍스트의 전체 수명주기를 관리한다: 의도 포착 → 명세화 → 컴파일 → 실행 → 검증 → 영속화.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    Agent = Model + Harness                   │
│                                                             │
│   ┌─────────────┐         ┌────────────────────────────┐    │
│   │             │         │         HARNESS             │    │
│   │    MODEL    │         │                            │    │
│   │             │         │  ┌─ 입력 처리              │    │
│   │  • 추론     │  ←───→  │  ├─ Agentic Loop 제어      │    │
│   │  • 도구 선택 │         │  ├─ 도구 실행 오케스트레이션 │    │
│   │  • 텍스트 생성│         │  ├─ 컨텍스트 관리          │    │
│   │             │         │  ├─ 오류 복구              │    │
│   │             │         │  ├─ 체크포인트             │    │
│   │             │         │  └─ 종료 조건 판단          │    │
│   └─────────────┘         └────────────────────────────┘    │
│                                                             │
│   모델이 "무엇을 할지" 결정     하네스가 "어떻게 할지" 관리     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 운영체제 비유

2025-2026년 업계에서 합의된 가장 정확한 비유:

| OS 개념 | 에이전트 대응물 | Claude Code 구현 |
|---------|--------------|----------------|
| **CPU** | LLM (원시 처리 능력) | Anthropic API (Opus/Sonnet/Haiku) |
| **RAM** | 컨텍스트 윈도우 | 200K 토큰 윈도우 |
| **운영체제/커널** | 하네스 (컨텍스트 큐레이션, 도구 디스패치) | `query.ts` + `QueryEngine.ts` |
| **시스템 콜** | 도구 호출 | `tools/` (40+ 도구) |
| **파일 시스템** | 영구 저장소 | `memdir/`, `history.ts`, `.claude/` |
| **프로세스 관리** | 에이전트 생명주기 | `AgentTool`, `forkedAgent.ts` |
| **가상 메모리** | 컨텍스트 압축 | `compact/`, `autoCompact.ts` |
| **인터럽트 핸들러** | 오류 복구 | Error Withholding, Cascade Recovery |
| **스케줄러** | 도구 실행 스케줄링 | `StreamingToolExecutor` |
| **사용자 권한** | 도구 권한 | `toolPermission/` (5계층 파이프라인) |

### 1.3 하네스 vs. 프레임워크 vs. 오케스트레이터

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Framework (프레임워크)                                      │
│  ├─ 컴포넌트 라이브러리 제공                                  │
│  ├─ 예: LangChain, CrewAI, AutoGen                          │
│  └─ "부품을 제공한다"                                        │
│                                                             │
│  Orchestrator (오케스트레이터)                                │
│  ├─ 에이전트 간 제어 흐름 관리                                │
│  ├─ 예: LangGraph, Coordinator 패턴                         │
│  └─ "누가 언제 무엇을 하는지 결정한다"                        │
│                                                             │
│  Harness (하네스)                                            │
│  ├─ 조립된 런타임 시스템 전체                                 │
│  ├─ 입력 처리 → 루프 제어 → 도구 실행 → 오류 복구 → 종료      │
│  ├─ 프레임워크 위에 구축하거나, 직접 구축                      │
│  └─ "에이전트가 실제로 돌아가게 만드는 모든 것"                │
│                                                             │
│  관계: Harness ⊃ Orchestrator ⊃ Framework Components        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**핵심 통찰**: Meta가 2025년 12월 Manus를 ~$2B에 인수한 것은 모델이 아니라 **하네스 기술** 때문이었다. "2025년이 에이전트의 해였다면, 2026년은 하네스의 해다."

### 1.4 왜 하네스가 중요한가

Anthropic의 연구에 따르면:
- 동일한 모델이라도 **하네스 설계에 따라 성능이 2-3배 차이**
- 검증 루프(Verification Loop)만 추가해도 품질이 **2-3배 향상**
- "기능 하나를 세션 하나에" 제약만으로 프로덕션 품질의 커밋율 극적 향상
- **자기 평가 편향(Self-evaluation bias)**: 에이전트는 자신의 평범한 결과를 자신있게 칭찬 → 생성과 평가를 분리해야 함

---

## 2. 하네스의 해부학: 7개 핵심 서브시스템

```
┌───────────────────────────────────────────────────────────────────┐
│                     HARNESS ANATOMY                                │
│                                                                   │
│  ┌─────────────┐                                                  │
│  │ 1. 입력 처리  │ ← 사용자 요청 수신, 컨텍스트 조립, 시스템 프롬프트 │
│  └──────┬──────┘                                                  │
│         │                                                         │
│  ┌──────▼─────────────────────────────────────────────────────┐   │
│  │ 2. AGENTIC LOOP (while true)                               │   │
│  │  ┌──────────────────┐                                      │   │
│  │  │ 컨텍스트 준비      │ ← 4. 컨텍스트 관리 하네스            │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │  ┌────────▼─────────┐                                      │   │
│  │  │ LLM API 호출      │ → 스트리밍 응답                      │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │  ┌────────▼─────────┐                                      │   │
│  │  │ 5. 오류 복구       │ ← Error Withholding + Cascade       │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │  ┌────────▼─────────┐                                      │   │
│  │  │ 3. 도구 실행       │ ← 배치, 동시성, 권한, 결과 예산      │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │  ┌────────▼─────────┐                                      │   │
│  │  │ 6. 체크포인트      │ ← 메시지, 파일, 비용 영속화          │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │  ┌────────▼─────────┐                                      │   │
│  │  │ 7. 종료 조건 판단  │ → Continue / Exit                   │   │
│  │  └────────┬─────────┘                                      │   │
│  │           │                                                │   │
│  │     ┌─────┴─────┐                                          │   │
│  │     │ Continue?  │                                          │   │
│  │     └─────┬─────┘                                          │   │
│  │     YES ↙     ↘ NO                                        │   │
│  │  (Loop) ←┘      → Exit with CompletionReason               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 3. 사용자 입력부터 에이전트 제어까지: 전체 흐름도

Claude Code의 실제 코드를 기반으로, 사용자가 프롬프트를 입력한 순간부터 결과가 반환되기까지의 전체 하네스 흐름:

```
사용자가 Enter를 누름
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 0: BOOTSTRAP (main.tsx → setup.ts → init.ts)             │
│                                                                 │
│  [프로세스 시작]                                                  │
│   ├─ startMdmRawRead()          ─┐                              │
│   ├─ startKeychainPrefetch()    ─┤ 병렬 (65ms)                   │
│   ├─ 모듈 임포트 (135ms)         ─┘                              │
│   ├─ CLI 파싱 (Commander.js)                                    │
│   ├─ 설정 로드 + Feature Flag 초기화                              │
│   ├─ Trust Dialog (1회성)                                       │
│   ├─ MCP 서버 승인                                              │
│   ├─ [첫 렌더링] ← 여기서 사용자가 타이핑 시작                    │
│   └─ startDeferredPrefetches() ← 백그라운드 초기화                │
│       ├─ getUserContext() (CLAUDE.md)                            │
│       ├─ getSystemContext() (git)                                │
│       ├─ initUser(), countFiles(), AWS/GCP 프리페치              │
│       └─ 분석 게이트, 스킬/설정 변경 감지기                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: INPUT PROCESSING (QueryEngine.submitMessage)           │
│                                                                 │
│  사용자 프롬프트 수신                                             │
│   │                                                             │
│   ├─ 1. processUserInput()                                      │
│   │   ├─ 문자열 → ContentBlockParam[] 정규화                     │
│   │   ├─ 슬래시 명령어 (/commit, /review 등) 감지 및 처리         │
│   │   └─ UserMessage 생성                                       │
│   │                                                             │
│   ├─ 2. getAttachmentMessages()                                 │
│   │   ├─ 파일 변경 감지 (마지막 턴 이후)                          │
│   │   ├─ 태스크 명령어 큐 소비                                   │
│   │   ├─ 태스크 알림 수집                                       │
│   │   └─ 메모리 파일 첨부                                       │
│   │                                                             │
│   ├─ 3. 시스템 프롬프트 조립                                     │
│   │   ├─ 기본 시스템 프롬프트                                    │
│   │   ├─ + 사용자 컨텍스트 (CLAUDE.md, 현재 날짜)                │
│   │   ├─ + 시스템 컨텍스트 (git 상태, 캐시 브레이커)              │
│   │   ├─ + 메모리 지시사항 (유형 분류, 접근 규칙)                 │
│   │   ├─ + 스킬 목록 (동적 /슬래시 명령어)                       │
│   │   └─ + Coordinator 컨텍스트 (멀티에이전트 모드 시)            │
│   │                                                             │
│   ├─ 4. 도구 풀 조립                                            │
│   │   ├─ Built-in 도구 (Feature Flag 기반 필터링)                │
│   │   ├─ + MCP 도구 (동적 발견)                                 │
│   │   ├─ - Deny 규칙 도구 (모델에서 완전히 제거)                  │
│   │   └─ 안정 정렬 (Prompt Cache 보존)                          │
│   │                                                             │
│   ├─ 5. CacheSafeParams 생성                                    │
│   │   └─ {시스템 프롬프트, 도구, 모델, 메시지 접두사, thinking}    │
│   │                                                             │
│   └─ 6. query() 제너레이터 호출 →→→ Phase 2로                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: AGENTIC LOOP (query.ts — while(true))                  │
│                                                                 │
│  State 초기화:                                                   │
│  {                                                              │
│    messages, toolUseContext, autoCompactTracking,                │
│    maxOutputTokensRecoveryCount: 0,                             │
│    hasAttemptedReactiveCompact: false,                           │
│    turnCount: 0, transition: undefined                          │
│  }                                                              │
│                                                                 │
│  ┌─── while (true) ─────────────────────────────────────────┐   │
│  │                                                           │   │
│  │  [2a] 컨텍스트 준비                                       │   │
│  │   ├─ Snip (경계 마커 기반 메시지 제거, 0ms)                │   │
│  │   ├─ Microcompact (tool_use_id 캐시 편집, <1ms)            │   │
│  │   ├─ Auto-compact (87% 임계값 시 Forked Agent 요약)        │   │
│  │   ├─ Context Collapse (단계적 drain)                      │   │
│  │   └─ Task Budget 검사 (에이전트별 USD 상한)                │   │
│  │                                                           │   │
│  │  [2b] LLM API 스트리밍 호출                               │   │
│  │   ├─ deps.callModel() → Streaming 시작                    │   │
│  │   ├─ for await (message of stream) {                      │   │
│  │   │   ├─ Error Withholding 검사 (413, max_tokens)         │   │
│  │   │   ├─ withheld가 아니면 → yield (UI/SDK로 전달)         │   │
│  │   │   ├─ tool_use 블록 감지 시:                           │   │
│  │   │   │   ├─ needsFollowUp = true                        │   │
│  │   │   │   └─ StreamingToolExecutor.addTool() ← 즉시!     │   │
│  │   │   └─ 완료된 도구 결과 즉시 yield                       │   │
│  │   │ }                                                     │   │
│  │   ├─ [동시에] Memory Prefetch (Haiku sidequery)           │   │
│  │   └─ [동시에] Skill Discovery Prefetch                    │   │
│  │                                                           │   │
│  │  [2c] 오류 복구 (withheld 에러가 있으면)                   │   │
│  │   ├─ 413: Collapse Drain → Reactive Compact → Surface     │   │
│  │   ├─ Max Tokens: 8K→64K 확장 → 복구 메시지 ×3 → Surface   │   │
│  │   └─ 복구 성공 시: continue (loop 재시작)                  │   │
│  │                                                           │   │
│  │  [2d] 도구 실행 (needsFollowUp이면)                       │   │
│  │   ├─ StreamingToolExecutor: 남은 결과 수집                 │   │
│  │   │   OR runTools(): 순차 실행 (폴백)                      │   │
│  │   ├─ 각 도구: 권한 검사 → 실행 → 결과                     │   │
│  │   ├─ 결과 크기 검사 (>100KB → 디스크 이관)                 │   │
│  │   └─ Tool-Use Summary 비동기 생성 (Haiku)                 │   │
│  │                                                           │   │
│  │  [2e] 수집                                                │   │
│  │   ├─ 큐된 명령어 소비                                     │   │
│  │   ├─ Memory Prefetch 소비 (settled이면, 0ms)              │   │
│  │   ├─ Skill Discovery 소비                                 │   │
│  │   └─ MCP 도구 갱신 (새 서버 연결 시)                       │   │
│  │                                                           │   │
│  │  [2f] 종료 조건 판단                                      │   │
│  │   ├─ needsFollowUp == false?                              │   │
│  │   │   ├─ handleStopHooks() 실행                           │   │
│  │   │   │   ├─ blockingErrors → 사용자 메시지로 주입, continue│   │
│  │   │   │   ├─ preventContinuation → EXIT                   │   │
│  │   │   │   └─ 통과 → EXIT (completed)                      │   │
│  │   │   └─ Token Budget: 90% 미달 → nudge 주입, continue    │   │
│  │   ├─ turnCount > maxTurns? → EXIT (max_turns)             │   │
│  │   └─ state = 새 State(메시지 누적 + 결과), continue        │   │
│  │                                                           │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│  EXIT → CompletionReason 반환                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: POST-LOOP (QueryEngine)                                │
│                                                                 │
│  ├─ 비용 누적 + OTel 메트릭 기록                                 │
│  ├─ 세션 비용 영속화                                             │
│  ├─ 자동 메모리 추출 (Forked Agent)                              │
│  ├─ 프롬프트 제안 생성 (Haiku, 비동기)                            │
│  └─ 사용자에게 결과 반환                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 서브시스템 1: 입력 처리 하네스

### 사용자 입력이 API 호출이 되기까지

사용자가 "이 버그 고쳐줘"라고 입력했을 때, 하네스가 하는 일:

```
"이 버그 고쳐줘" (사용자 텍스트)
│
├─ 1. 입력 정규화
│   └─ string → ContentBlockParam[] (API 형식)
│
├─ 2. 슬래시 명령어 감지
│   ├─ /commit, /review 등 → 전용 핸들러로 라우팅
│   └─ 일반 텍스트 → Agentic Loop으로 진행
│
├─ 3. 첨부 메시지 수집 (사용자 모르게)
│   ├─ 파일 변경 감지: 마지막 턴 이후 수정된 파일
│   ├─ 메모리 첨부: MEMORY.md + 관련 메모리 파일
│   ├─ 태스크 알림: 백그라운드 에이전트 완료 알림
│   └─ 스킬 목록: 사용 가능한 /슬래시 명령어
│
├─ 4. 시스템 프롬프트 조립 (사용자 모르게)
│   ├─ 기본 페르소나 + 도구 사용 지침
│   ├─ CLAUDE.md 계층 (관리 → 글로벌 → 프로젝트 → 로컬)
│   ├─ Git 상태 스냅샷 (브랜치, 최근 커밋, 변경사항)
│   ├─ 메모리 시스템 지시사항 (유형 분류, 접근 규칙)
│   └─ 현재 날짜, 모델 정보
│
├─ 5. 도구 풀 결정 (사용자 모르게)
│   ├─ 40+ 빌트인 도구
│   ├─ + MCP 서버 도구 (동적)
│   ├─ + ToolSearch 도구 (지연 로딩, 토큰 85% 절감)
│   ├─ - Deny 규칙 매칭 도구 (모델에서 제거)
│   └─ 안정 정렬 (API 캐시 보존)
│
├─ 6. Agent Skills 로드 (사용자 모르게)
│   ├─ 모든 설치된 스킬의 name + description만 로드 (~100토큰/스킬)
│   ├─ 3단계 Progressive Disclosure:
│   │   L1: 메타데이터만 → L2: 매칭 시 전체 → L3: 실행 시 상세
│   ├─ Enterprise/Personal/Project/Plugin 4단계 우선순위
│   └─ 개방형 표준 (agentskills.io, 33+ 플랫폼 호환)
│
└─ 7. CacheSafeParams 스냅샷
    └─ 시스템 프롬프트 + 도구 + 모델 + 메시지 접두사
       → Forked Agent와 공유할 캐시 키
```

### 엔지니어 인사이트: 컨텍스트 조립은 하네스의 가장 중요한 역할

Anthropic의 "Context Engineering" 문서에서:

> "컨텍스트 윈도우의 모든 토큰은 모델의 '주의력 예산'을 소비한다. 최소한의 고신호 토큰으로 원하는 결과의 가능성을 최대화하는 것이 핵심이다."

Claude Code의 입력 처리 하네스는 이를 구현한다:
- **CLAUDE.md**: 프로젝트 규약을 한 번 작성하면 매 세션에 자동 주입
- **Git 상태**: 개발자가 "지금 main 브랜치야, 최근에 auth 모듈을 수정했어" 같은 맥락을 매번 설명할 필요 없음
- **Agent Skills**: 3단계 Progressive Disclosure로 필요한 절차적 지식만 적시에 로드. 메타데이터 ~100 토큰으로 수십 개 스킬을 "인지"하면서도 컨텍스트 소비 최소화
- **Tool Search**: 40+ 도구를 모두 프롬프트에 넣는 대신, 핵심 도구만 상시 + 나머지는 검색 기반 → 토큰 85% 절감
- **도구 가시성 제어**: 불필요한 도구를 제거하여 모델의 도구 선택 품질 향상
- **안정 정렬**: 도구 순서 불변으로 API Prompt Cache 재사용

---

## 5. 서브시스템 2: Agentic Loop 하네스

### 핵심: while(true) 루프의 상태 머신

Claude Code의 Agentic Loop 하네스는 `query.ts`의 1,729줄 AsyncGenerator 함수다. 이것이 하네스의 심장이다.

#### 상태(State) 정의

```typescript
type State = {
  messages: Message[]                    // 누적 대화 히스토리
  toolUseContext: ToolUseContext          // 도구 런타임 상태
  autoCompactTracking: AutoCompactTrackingState | undefined  // 압축 추적
  maxOutputTokensRecoveryCount: number   // 출력 토큰 복구 시도 횟수
  hasAttemptedReactiveCompact: boolean   // 반응적 압축 시도 플래그
  maxOutputTokensOverride: number | undefined  // 출력 토큰 상한 재정의
  pendingToolUseSummary: Promise<...> | undefined  // 비동기 요약
  stopHookActive: boolean | undefined    // Stop Hook 활성 여부
  turnCount: number                      // 현재 턴 번호
  transition: Continue | undefined       // 이전 반복의 계속 사유
}
```

#### 7개의 Continue(계속) 사유

하네스가 루프를 계속하는 7가지 이유 — 각각 다른 상태 변이를 수반:

| Continue 사유 | 트리거 | 상태 변이 |
|-------------|--------|----------|
| `next_turn` | 도구 실행 완료, 후속 호출 필요 | messages += 어시스턴트 + 도구결과, turnCount++ |
| `collapse_drain_retry` | 413 후 Context Collapse 시도 | messages 축소, 복구 플래그 미변경 |
| `reactive_compact_retry` | 413 후 전체 요약 시도 | messages 요약으로 교체, hasAttemptedReactiveCompact=true |
| `max_output_tokens_escalate` | 8K 상한 도달 | maxOutputTokensOverride=64K |
| `max_output_tokens_recovery` | 64K에서도 상한 도달 | 복구 메시지 주입, count++ (최대 3) |
| `stop_hook_blocking` | Stop Hook이 차단 에러 반환 | 에러를 사용자 메시지로 주입 |
| `token_budget_continuation` | 토큰 예산 미달 (90% 이하) | 넛지 메시지 주입 |

#### 11개의 Exit(종료) 사유

```
┌─────────────────────────────────────────────────────────────┐
│                  COMPLETION REASONS                           │
│                                                             │
│  정상 종료:                                                  │
│  ├─ completed          도구 호출 없이 응답 완료               │
│  ├─ max_turns          최대 턴 수 도달                       │
│  └─ stop_hook_prevented Stop Hook이 명시적 종료              │
│                                                             │
│  오류 종료:                                                  │
│  ├─ model_error        API 에러 (복구 소진 후)               │
│  ├─ prompt_too_long    413 (모든 압축 전략 소진 후)           │
│  ├─ image_error        이미지 크기 오류                      │
│  └─ blocking_limit     토큰 상한 사전 차단                   │
│                                                             │
│  중단 종료:                                                  │
│  ├─ aborted_streaming  사용자가 스트리밍 중 취소              │
│  ├─ aborted_tools      사용자가 도구 실행 중 취소             │
│  └─ hook_stopped       도구 Hook이 실행 중단 시그널           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 턴 간 상태 리셋 vs 보존

```
매 턴(next_turn)마다 리셋:              매 턴마다 보존:
  maxOutputTokensRecoveryCount: 0       messages (누적)
  hasAttemptedReactiveCompact: false     toolUseContext (공유)
  maxOutputTokensOverride: undefined     autoCompactTracking (압축 상태)
  pendingToolUseSummary: undefined       turnCount (증가)
```

**왜 이렇게 설계하는가**: 복구 시도 카운터는 턴 단위로 리셋해야 한다. 이전 턴에서 max_tokens를 3번 복구했다고 해서, 다음 턴에서도 복구 불가능한 것은 아니다. 그러나 `hasAttemptedReactiveCompact`는 연속적인 압축 시도가 무한 루프를 만들 수 있으므로 턴 간에도 신중하게 관리한다.

### 엔지니어 인사이트: Transition 추적의 가치

```
transition 필드의 목적:
  ├─ 디버깅: "왜 이 턴이 계속되었는가?"를 즉시 답할 수 있음
  ├─ 테스트: 메시지 내용 검사 없이 복구 경로 실행을 assert
  ├─ 텔레메트리: 복구 경로별 빈도 분석 가능
  └─ 관찰가능성: 매 턴의 "의도"가 기록됨
```

---

## 6. 서브시스템 3: 도구 실행 하네스

### StreamingToolExecutor: 스트리밍 중 도구 선행 실행

도구 실행 하네스의 가장 독창적인 패턴:

```
┌─────────────────────────────────────────────────────────────┐
│         StreamingToolExecutor Architecture                    │
│                                                             │
│  모델 스트리밍 중:                                           │
│  ├─ tool_use 블록 감지 → addTool(block, message)            │
│  ├─ 동시성 검사:                                            │
│  │   canExecuteTool(isConcurrencySafe) {                    │
│  │     executingTools = filter(status === 'executing')       │
│  │     return executingTools.length === 0 ||                │
│  │       (isConcurrencySafe &&                              │
│  │        executingTools.every(t => t.isConcurrencySafe))    │
│  │   }                                                      │
│  │                                                          │
│  │   규칙:                                                  │
│  │   ├─ 비동시성 도구: 혼자만 실행 (배타적)                   │
│  │   ├─ 동시성 도구: 다른 동시성 도구와 병렬 실행             │
│  │   └─ 혼합 불가: 비동시성 도구 실행 중이면 모두 대기        │
│  │                                                          │
│  ├─ 실행 상태 추적:                                         │
│  │   queued → executing → completed → yielded               │
│  │                                                          │
│  └─ 에러 캐스케이드:                                        │
│      Bash 에러 → siblingAbortController.abort()              │
│      → 형제 도구에 "Cancelled: parallel tool call errored"   │
│      (다른 도구 타입의 에러는 캐스케이드하지 않음)             │
│                                                             │
│  스트리밍 완료 후:                                            │
│  ├─ getRemainingResults()로 미완료 도구 대기                 │
│  └─ 결과 순서 보존 (병렬 실행이더라도)                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 도구 결과 예산 관리 (ContentReplacementState)

```
도구 결과가 100KB 초과 시:
  1. 디스크에 영속화 (~/.claude/projects/.../)
  2. 컨텍스트에는 참조(미리보기 + 파일 경로)만 삽입
  3. ContentReplacementState에 교체 기록

이유:
  100KB 결과가 컨텍스트에 남으면 → 이후 모든 API 호출에서 재전송
  20회 호출 × 100KB = 2MB 추가 토큰 비용
  디스크 이관으로 이 비용을 0에 가깝게 만듦
```

### 엔지니어 인사이트: 도구 실행 하네스의 설계 원칙

```
1. 읽기 도구는 병렬, 쓰기 도구는 직렬
   → isConcurrencySafe 플래그 하나로 제어
   → 새 도구는 기본값 false (Fail-Closed)

2. 결과 순서는 보존
   → 병렬 실행이라도 결과는 도구 순서대로 yield
   → 모델이 결과를 예측 가능한 순서로 받음

3. 에러 캐스케이드는 Bash에만 적용
   → Bash 에러는 형제에게 전파 (하나 실패 → 전체 실패)
   → Read/WebFetch 에러는 독립 (하나 실패해도 나머지 계속)
   → 이유: Bash 명령어는 부작용이 크므로 일관성이 중요

4. 결과 크기를 제한하라
   → 하네스의 가장 중요한 비용 절감 메커니즘
   → 100KB 임계값은 "유용한 컨텍스트"와 "토큰 낭비"의 경계
```

---

## 7. 서브시스템 4: 컨텍스트 관리 하네스

### 토큰 예산 모니터링

하네스는 매 턴마다 컨텍스트 윈도우의 사용률을 모니터링한다:

```
┌─────────────────────────────────────────────────────────────┐
│           Context Window Budget Monitor                      │
│                                                             │
│  200K 토큰 윈도우                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│  │         사용 중                        여유             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  임계값:                                                    │
│  ├─ 87% (167K) → Auto-Compact 트리거                       │
│  │   effectiveWindow = 200K - 20K(요약 예약) = 180K         │
│  │   threshold = 180K - 13K(버퍼) = 167K                   │
│  │                                                          │
│  ├─ 90% (180K) → Warning UI 표시                           │
│  │   threshold = effectiveWindow - 20K(경고)               │
│  │                                                          │
│  ├─ 95% (190K) → Error UI 표시                             │
│  │   threshold = contextWindow - 20K(에러)                  │
│  │                                                          │
│  └─ 100% → API 413 에러                                    │
│      → Error Withholding → Cascade Recovery                │
│                                                             │
│  Circuit Breaker:                                           │
│  ├─ 연속 3회 Auto-Compact 실패 → 재시도 중단                │
│  └─ MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3                │
│                                                             │
│  환경 변수 오버라이드:                                       │
│  └─ CLAUDE_AUTOCOMPACT_PCT_OVERRIDE (테스트용)              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Anthropic의 5가지 컨텍스트 전략과 Claude Code의 구현

Anthropic의 "Context Engineering" 문서에서 정의한 5가지 전략:

| 전략 | 설명 | Claude Code 구현 |
|------|------|----------------|
| **프로젝트 파일** | 세션 시작 시 핵심 파일 로드 | CLAUDE.md 계층, Git 상태 |
| **Just-In-Time 검색** | 필요할 때 파일 읽기 | Grep/Glob/Read 도구 |
| **Compaction** | 히스토리 요약 | 4단계 압축 (Snip→Micro→Auto→Reactive) |
| **구조화된 메모** | 외부 노트 유지 | memdir/ (MEMORY.md + 유형별 파일) |
| **Sub-Agent 격리** | 전문 에이전트에 깨끗한 컨텍스트 | Forked Agent (CacheSafeParams) |

### 엔지니어 인사이트: KV-Cache 최적화

Manus의 연구에 따르면 KV-Cache 최적화가 프로덕션 하네스의 가장 중요한 메트릭:

```
KV-Cache 효율:
  Prefill(캐시 미스) : Decode(캐시 히트) 비율 = 100:1 비용 차이
  
  Claude Code의 KV-Cache 보존 전략:
  1. 도구 목록 안정 정렬 → 순서 변경 시 캐시 무효화 방지
  2. CacheSafeParams → Forked Agent가 부모 캐시 재사용
  3. Microcompact → 바이트 정렬 유지하며 편집
  4. Prompt Cache Breakpoints → 시스템 프롬프트 접두사 불변

  결과: 입력 토큰 비용 ~90% 절감
```

---

## 8. 서브시스템 5: 오류 복구 하네스

### Error Withholding: 하네스의 가장 독창적 패턴

```
┌─────────────────────────────────────────────────────────────┐
│           Error Withholding Flow                             │
│                                                             │
│  API 스트리밍 중:                                            │
│                                                             │
│  for await (message of stream) {                            │
│    │                                                        │
│    ├─ 413 Prompt-Too-Long 감지?                             │
│    │   └─ withheld = true (UI에 전달하지 않음)               │
│    │                                                        │
│    ├─ Max-Output-Tokens 감지?                               │
│    │   └─ withheld = true                                   │
│    │                                                        │
│    ├─ 미디어 크기 초과?                                      │
│    │   └─ withheld = true                                   │
│    │                                                        │
│    └─ 정상 메시지?                                          │
│        └─ withheld가 아니면 yield → UI/SDK                  │
│  }                                                          │
│                                                             │
│  스트리밍 완료 후:                                            │
│                                                             │
│  if (413_withheld) {                                        │
│    ├─ try: context_collapse_drain()                         │
│    │   └─ 성공 → continue (withheld 에러 폐기)              │
│    ├─ try: reactive_compact()                               │
│    │   └─ 성공 → continue (withheld 에러 폐기)              │
│    └─ 모두 실패 → surface_error() (이제야 사용자에게 표시)    │
│  }                                                          │
│                                                             │
│  if (max_tokens_withheld) {                                 │
│    ├─ count == 0? → escalate(8K→64K), continue              │
│    ├─ count < 3? → inject_nudge("계속하세요"), continue       │
│    └─ count >= 3 → surface_error()                          │
│  }                                                          │
│                                                             │
│  핵심 원칙:                                                  │
│  "복구에 성공한 오류는 존재하지 않았던 것이다"                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Death Spiral 방지

```
위험한 패턴:
  Stop Hook 실행 → API 호출 → API 에러 → Stop Hook 실행 → API 호출 → ...

Claude Code의 방어:
  if (lastMessage.isApiErrorMessage) {
    // Stop Hooks 건너뜀 — 재귀 방지
    // stop_hook_failure만 실행 (1회성, 재귀 없음)
  }
```

### 가드(Guard) 플래그 시스템

```
hasAttemptedReactiveCompact: boolean
  ├─ false → Reactive Compact 시도 가능
  ├─ true → 이미 시도함, 다시 시도하지 않음
  └─ 리셋: 다음 턴(next_turn)에서 false로

maxOutputTokensRecoveryCount: number
  ├─ 0-2 → 복구 메시지 주입 가능
  ├─ 3 (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) → 소진, 에러 표출
  └─ 리셋: 다음 턴에서 0으로

autoCompactTracking.consecutiveFailures: number
  ├─ 0-2 → Auto-Compact 재시도 가능
  ├─ 3 (MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) → Circuit Breaker
  └─ 리셋되지 않음 (세션 수명 동안 유지)
```

---

## 9. 서브시스템 6: 체크포인트 하네스

### 이중 체크포인트 시스템

```
┌─────────────────────────────────────────────────────────────┐
│           Dual Checkpointing Architecture                    │
│                                                             │
│  시스템 1: 세션 히스토리 (history.ts)                         │
│  ├─ 저장소: ~/.claude/history.jsonl (글로벌)                  │
│  ├─ 단위: 사용자 입력 + 메타데이터                            │
│  ├─ 대형 붙여넣기: SHA256 해시 → 외부 저장소 (>1KB)           │
│  ├─ 비동기 플러시: pendingEntries 큐 → 배치 기록              │
│  ├─ 제거: soft-delete (skippedTimestamps)                   │
│  └─ 복원: /resume → 세션 ID 기반 필터링                      │
│                                                             │
│  시스템 2: 파일 히스토리 (fileHistory.ts)                     │
│  ├─ 저장소: .claude/.file-history/ (프로젝트별)               │
│  ├─ 단위: 매 어시스턴트 메시지마다 스냅샷                     │
│  ├─ 캡처: 편집 전 파일 내용 (trackEdit)                      │
│  ├─ 스냅샷: 버전 넘버링 (v1, v2, ..., 최대 100개)            │
│  ├─ 상속: 변경 안 된 파일은 이전 스냅샷 참조 재사용            │
│  └─ 용도: "아까 수정한 거 되돌려줘" 가능                     │
│                                                             │
│  시스템 3: 비용 상태 (cost-tracker.ts)                       │
│  ├─ 매 API 응답마다 누적                                    │
│  ├─ 세션 종료 시 프로젝트 설정에 영속화                       │
│  └─ 세션 복원 시 세션 ID 매칭으로 복구                       │
│                                                             │
│  체크포인트의 핵심 가치:                                     │
│  ├─ 세션 복원: 터미널 종료 후에도 /resume 가능                │
│  ├─ 시간 여행: 파일 변경 이력 탐색 가능                      │
│  └─ 비용 연속성: 세션 간 비용 추적 유지                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Anthropic의 체크포인트 권고: Git을 활용하라

Anthropic의 하네스 가이드에서:

> "Git은 에이전트의 가장 강력한 체크포인트 메커니즘이다. 각 의미 있는 작업 단위 후 설명적 커밋을 생성하면, 실패 시 이전 상태로 롤백할 수 있다."

Claude Code의 구현:
- `EnterWorktreeTool`: Git worktree로 에이전트별 격리된 브랜치 생성
- `ExitWorktreeTool`: 작업 완료 후 워크트리 정리 또는 변경사항 보존
- 파일 히스토리: Git 외적인 세밀한 스냅샷 (커밋 단위보다 세밀)

---

## 10. 서브시스템 7: 종료 조건 하네스

### Stop Hook: 하네스의 최종 관문

모델이 "완료"라고 판단한 후에도, 하네스는 **Stop Hook**을 통해 최종 검증을 수행한다:

```
┌─────────────────────────────────────────────────────────────┐
│           Stop Hook Lifecycle                                │
│                                                             │
│  모델 응답 완료 (도구 호출 없음)                               │
│   │                                                         │
│   ├─ API 에러였나? → YES → Stop Hook 건너뜀 (Death Spiral 방지)│
│   │                                                         │
│   └─ NO → handleStopHooks() 실행                            │
│       │                                                     │
│       ├─ 1. 사용자 정의 Hook 병렬 실행                       │
│       │   ├─ 각 Hook의 exit code 분석                        │
│       │   ├─ 비차단 에러 vs 차단 에러 분류                    │
│       │   └─ Hook 요약 메시지 생성 (소요 시간 포함)           │
│       │                                                     │
│       ├─ 2. 결과 분석                                       │
│       │   ├─ blockingErrors > 0?                            │
│       │   │   → 에러를 사용자 메시지로 주입                   │
│       │   │   → continue (stop_hook_blocking)               │
│       │   │   → "모델아, 이 문제를 고쳐"                     │
│       │   │                                                 │
│       │   ├─ preventContinuation == true?                   │
│       │   │   → EXIT (stop_hook_prevented)                  │
│       │   │                                                 │
│       │   └─ 통과                                          │
│       │       → EXIT (completed)                            │
│       │                                                     │
│       └─ 3. 백그라운드 작업 (비차단)                         │
│           ├─ 프롬프트 제안 (Haiku, fire-and-forget)          │
│           ├─ 메모리 추출 (Forked Agent)                     │
│           ├─ Auto-Dream (투기적 포크)                       │
│           └─ CacheSafeParams 스냅샷 저장                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 검증 루프(Verification Loop)의 중요성

Anthropic의 연구:

> "검증 루프만 추가해도 품질이 2-3배 향상된다. 자기 평가 편향(Self-evaluation bias)이 있으므로, 생성과 평가를 분리해야 한다."

Claude Code의 Stop Hook은 이 "분리된 검증"을 사용자 정의 가능하게 만든다:
- 린터/포매터 실행 → 실패 시 모델에게 수정 지시
- 테스트 실행 → 실패 시 모델에게 버그 수정 지시
- 커스텀 규칙 검증 → 위반 시 모델에게 규칙 준수 지시

```json
// settings.json Hook 예시
{
  "hooks": [{
    "type": "stop",
    "command": "npm test",
    "blocking": true
  }]
}
// → 모델이 "완료"라고 해도, 테스트가 실패하면
//   에러를 사용자 메시지로 주입하여 모델이 수정하도록 유도
```

### Token Budget Continuation

Token Budget 기능은 하네스가 모델의 "조기 종료"를 방지한다:

```
Token Budget 설정 (예: 500K 토큰)
  │
  ├─ 모델이 도구 호출 없이 응답 완료
  │
  ├─ 하네스: "출력이 예산의 90% 미만인가?"
  │   ├─ YES → "더 작업하세요" 넛지 메시지 주입, continue
  │   │        (token_budget_continuation)
  │   └─ NO → "충분히 작업했다" → 종료
  │
  └─ 수확 체감(Diminishing Returns) 감지
      └─ 연속된 턴에서 진전이 없으면 → 종료
```

---

## 11. Sub-Agent 하네스: 하네스 안의 하네스

### Forked Agent: 부모 하네스의 캐시를 공유하는 자식 하네스

```
┌─────────────────────────────────────────────────────────────┐
│         Sub-Agent Harness Architecture                       │
│                                                             │
│  Parent Harness                                             │
│  ├─ CacheSafeParams 스냅샷                                  │
│  │   = {시스템 프롬프트, 도구, 모델, thinking, 메시지 접두사}  │
│  │                                                          │
│  ├─ AgentTool.call() 또는 runForkedAgent()                   │
│  │   │                                                      │
│  │   ├─ 1. 컨텍스트 복제                                    │
│  │   │   ├─ cloneFileStateCache() — 파일 캐시 복제           │
│  │   │   ├─ createChildAbortController() — 부모 연결         │
│  │   │   ├─ ContentReplacementState 복제                    │
│  │   │   └─ 새 ToolUseContext (부모 상태 미변경)             │
│  │   │                                                      │
│  │   ├─ 2. 하네스 파라미터 구성                              │
│  │   │   ├─ cacheSafeParams: 부모와 동일 → 캐시 HIT         │
│  │   │   ├─ maxTurns: 에이전트별 턴 제한                     │
│  │   │   ├─ taskBudget: 에이전트별 USD 예산                  │
│  │   │   ├─ allowedTools: 화이트리스트                       │
│  │   │   └─ querySource: 'agent:label'                      │
│  │   │                                                      │
│  │   ├─ 3. query() 재귀 호출 (자식 하네스 시작)              │
│  │   │   └─ 동일한 while(true) 루프 실행                    │
│  │   │                                                      │
│  │   ├─ 4. 결과 수집                                       │
│  │   │   ├─ accumulateUsage() — 토큰/비용 누적              │
│  │   │   ├─ Task Notification (status + summary)             │
│  │   │   └─ Sidechain Transcript 기록                       │
│  │   │                                                      │
│  │   └─ 5. 부모 하네스에 요약 반환                           │
│  │       └─ 전체 컨텍스트가 아닌 압축된 결과만                │
│  │                                                          │
│  └─ 부모 하네스 계속                                        │
│                                                             │
│  캐시 공유 효과:                                             │
│  ├─ 부모 시스템 프롬프트 + 도구 = ~5K 토큰 → 캐시 HIT        │
│  ├─ 입력 비용 ~90% 절감                                     │
│  └─ 자식 하네스의 고유 부분만 새로 계산                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Coordinator-Worker: 하네스의 계층 구조

```
Coordinator 하네스
  ├─ 전체 도구 접근
  ├─ 작업 분해 + 할당
  │
  ├─ Worker 하네스 1 (제한된 도구)
  │   ├─ AgentTool 없음 (sub-agent 생성 불가 — 무한 재귀 방지)
  │   ├─ SendMessage 없음 (직접 통신 불가 — Coordinator 경유)
  │   └─ Task Notification으로 결과 보고 (status + summary + usage)
  │
  ├─ Worker 하네스 2 (독립 컨텍스트)
  │   └─ 부모 캐시 공유하지만, 메시지는 독립
  │
  └─ 결과 합성 → "이해를 위임하지 않는다"
```

---

## 12. Production 하네스 설계 패턴

### 패턴 1: Initializer-Executor (Anthropic)

Anthropic의 SWE-bench 최적화에서 발견:

```
세션 시작 시:
  Initializer Agent (1회 실행)
  ├─ 레포지토리 구조 분석
  ├─ 200+ 속성의 JSON 명세 생성
  │   (실패 모드, 관련 파일, 테스트 전략 등)
  └─ Executor에 전달

이후:
  Executor Agent (N회 반복)
  ├─ Initializer의 명세를 기반으로 작업
  ├─ 한 세션에 하나의 기능만 구현
  └─ 검증 루프로 품질 보장

핵심 제약: "한 세션에 하나의 기능"
  → 컨텍스트 소진 방지
  → 프로덕션 품질 커밋 보장
```

### 패턴 2: 파일시스템을 메모리로 (Industry Consensus)

```
2026년 업계 합의: "모든 프로덕션 하네스는 파일시스템과 Git으로 수렴한다"

Claude Code의 구현:
  ├─ MEMORY.md + 메모리 파일 = 영구 메모리
  ├─ .claude/.file-history/ = 시간 여행
  ├─ history.jsonl = 세션 복원
  ├─ Git worktree = 에이전트 격리
  └─ 디스크 이관 = 대형 결과 저장

왜 벡터 DB가 아닌가:
  ├─ 파일: 검사 가능, 편집 가능, 버전 관리 가능
  ├─ Git: 원자적 커밋, 분기, 병합, 롤백
  ├─ 벡터 DB: 불투명, 디버깅 어려움, 운영 복잡
  └─ "에이전트는 검사 가능한 상태를 선호한다"
```

### 패턴 3: 검증 분리 (Planner-Generator-Evaluator)

```
Anthropic의 3-Agent 아키텍처:

  Planner Agent → "Sprint Contract" 생성
       │
       ▼
  Generator Agent → 코드 생성
       │
       ▼
  Evaluator Agent → 코드 평가
       │
  ┌────┴────┐
  │ 통과?    │
  │ YES → 완료 │
  │ NO → 피드백 → Generator로 돌아감 │
  └─────────┘

핵심 통찰: "자기 평가 편향"
  ├─ 에이전트는 자신의 평범한 결과를 자신있게 칭찬
  ├─ 생성과 평가를 같은 컨텍스트에서 하면 편향 증폭
  └─ 해법: 별도 에이전트 또는 별도 프롬프트로 평가
```

### 패턴 4: Attention 조작 (Manus)

```
Manus의 발견:
  todo.md를 컨텍스트 시작에 배치
  → Transformer의 Attention이 최근 토큰에 편향
  → todo.md가 "가장 최근"이 아니므로 무시될 수 있음
  → 해법: 매 턴마다 todo.md를 컨텍스트 끝에 재주입

Claude Code의 구현:
  ├─ 시스템 프롬프트: 컨텍스트 시작 (캐시 안정성)
  ├─ 메모리 첨부: 매 턴마다 재주입 (Attention 유지)
  ├─ 스킬 목록: 매 턴마다 재주입
  └─ Git 상태: 세션 시작 시 1회 (변경 없으므로)
```

### 패턴 5: 에러 보존 (Manus)

```
Manus의 핵심 원칙:
  "에이전트 오류를 절대 제거하지 말라.
   오류 메시지는 현재 접근 방식이 잘못되었다는 가장 강력한 신호다."

Claude Code의 구현:
  ├─ 도구 에러: 결과 메시지에 그대로 포함 (모델이 학습)
  ├─ Stop Hook 에러: 사용자 메시지로 주입 (모델이 수정)
  └─ API 에러: Withholding 후 복구 불가 시에만 표출

  반대로 제거하는 것:
  ├─ 성공한 큰 도구 결과: 디스크 이관 (에러가 아니므로)
  └─ 오래된 히스토리: 압축 (에러가 아니므로)
```

---

## 13. 하네스 설계 체크리스트

### Enterprise Agentic LLM 하네스를 설계할 때의 체크리스트

```
┌─────────────────────────────────────────────────────────────┐
│          HARNESS DESIGN CHECKLIST                            │
│                                                             │
│  입력 처리 하네스                                            │
│  □ 사용자 입력 정규화 (텍스트, 이미지, 파일)                  │
│  □ 시스템 프롬프트 자동 조립 (프로젝트 규약, Git, 메모리)      │
│  □ 도구 풀 동적 구성 (Deny 규칙 적용, 안정 정렬)             │
│  □ CacheSafeParams 스냅샷 (Sub-Agent 캐시 공유용)           │
│  □ Agent Skills 로드 (3단계 Progressive Disclosure)        │
│  □ Tool Search 통합 (지연 로딩으로 토큰 85% 절감)           │
│                                                             │
│  Agentic Loop 하네스                                        │
│  □ State 타입 정의 (턴 간 보존 vs 리셋 구분)                 │
│  □ Continue 사유 추적 (transition 필드)                      │
│  □ Exit 조건 열거 (최소 7가지: 정상, 에러, 중단)              │
│  □ Guard 플래그 (무한 루프 방지: 시도 횟수, Circuit Breaker)  │
│  □ 턴 카운팅 + maxTurns 제한                                │
│                                                             │
│  도구 실행 하네스                                            │
│  □ 동시성 제어 (읽기 병렬, 쓰기 직렬)                        │
│  □ Streaming Tool Execution (모델 스트리밍 중 도구 선행 실행)  │
│  □ 결과 크기 예산 (임계값 초과 → 디스크 이관)                 │
│  □ 에러 캐스케이드 규칙 (Bash는 전파, 읽기는 독립)            │
│  □ 결과 순서 보존                                           │
│                                                             │
│  컨텍스트 관리 하네스                                        │
│  □ 토큰 예산 모니터링 (87% 능동, 100% 반응)                  │
│  □ 계층적 압축 (저비용 먼저, 고비용 최후)                     │
│  □ Prompt Cache 보존 (바이트 정렬, 안정 정렬)                │
│  □ Post-Compact 복원 (스킬, 파일, MCP 재주입)                │
│  □ Circuit Breaker (연속 실패 시 재시도 중단)                 │
│                                                             │
│  오류 복구 하네스                                            │
│  □ Error Withholding (복구 가능한 에러 보류)                  │
│  □ Cascade Recovery (저비용→고비용 순서)                     │
│  □ Guard 플래그 (복구 시도 횟수 제한)                        │
│  □ Death Spiral 방지 (에러 처리가 에러를 유발하지 않도록)      │
│  □ API 재시도 (처음 3회 은닉, Foreground/Background 분리)     │
│  □ 모델 Fallback (Opus → Sonnet 자동 전환)                  │
│                                                             │
│  체크포인트 하네스                                           │
│  □ 세션 히스토리 영속화 (JSONL, 비동기)                      │
│  □ 파일 히스토리 스냅샷 (매 어시스턴트 메시지마다)             │
│  □ 비용 상태 영속화 (세션 간)                                │
│  □ 세션 복원 (/resume)                                      │
│  □ Git 기반 체크포인트 (worktree 격리)                       │
│                                                             │
│  종료 조건 하네스                                            │
│  □ Stop Hook (사용자 정의 검증 루프)                         │
│  □ Blocking Error → 모델에게 수정 지시 (재귀적 개선)          │
│  □ Token Budget Continuation (조기 종료 방지)                │
│  □ Graceful Shutdown (보류 I/O 완료 대기)                    │
│  □ 시그널 처리 (SIGINT, SIGTERM, SIGHUP)                    │
│                                                             │
│  Sub-Agent 하네스                                           │
│  □ CacheSafeParams 공유 (입력 비용 ~90% 절감)               │
│  □ Task Budget 전파 (에이전트별 USD 상한)                    │
│  □ 도구 제한 (AgentTool 제거 → 무한 재귀 방지)              │
│  □ Abort 시그널 체인 (부모→자식 연결)                       │
│  □ 결과는 요약만 반환 (전체 컨텍스트 공유 X)                 │
│                                                             │
│  관찰가능성                                                 │
│  □ 단계별 프로파일링 (queryCheckpoint)                       │
│  □ 이벤트 로깅 (logEvent: 복구, fallback, 에러)              │
│  □ 비용 추적 (턴별, 에이전트별, 모델별)                      │
│  □ 권한 결정 감사 추적                                      │
│  □ Transition 추적 (왜 계속/종료했는가)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 최종 인사이트: 하네스가 가치를 만드는 곳

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  "2025년이 에이전트의 해였다면, 2026년은 하네스의 해다."      │
│                                                             │
│  모델의 지능은 모든 회사가 동일하게 접근할 수 있다.            │
│  (API를 호출하면 된다)                                       │
│                                                             │
│  하네스의 품질이 제품의 차별화를 만든다.                       │
│  (Meta가 Manus를 $2B에 인수한 이유)                          │
│                                                             │
│  하네스 엔지니어링의 핵심 가치:                               │
│                                                             │
│  1. 모델이 잘하는 것(추론)에 집중하게 하고,                   │
│     나머지(컨텍스트, 도구, 에러, 비용)는 하네스가 흡수한다.    │
│                                                             │
│  2. 사용자가 본질(무엇을 할 것인가)에 집중하게 하고,          │
│     나머지(권한, 대기, 복구, 세션)는 하네스가 흡수한다.        │
│                                                             │
│  3. 하네스의 품질 = 보이지 않는 것의 품질                     │
│     최고의 하네스는 사용자가 그 존재를 인지하지 못하는 것이다.  │
│                                                             │
│  Agent = Model + Harness                                    │
│  Model은 API다. Harness가 제품이다.                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 부록: Claude Code 하네스 핵심 파일 참조

| 하네스 서브시스템 | 핵심 파일 | 코드 규모 |
|----------------|---------|----------|
| Agentic Loop | `query.ts` | 1,729줄 |
| 입력 처리 | `QueryEngine.ts` | ~46K |
| 도구 실행 | `StreamingToolExecutor.ts`, `toolOrchestration.ts` | — |
| 컨텍스트 관리 | `autoCompact.ts`, `compact.ts` | — |
| 오류 복구 | `query.ts` Phase 3, `withRetry.ts` | — |
| 체크포인트 | `history.ts`, `fileHistory.ts` | — |
| 종료 조건 | `query/stopHooks.ts`, `query/tokenBudget.ts` | — |
| Sub-Agent | `forkedAgent.ts`, `AgentTool.tsx` | ~233K |
| 상태 관리 | `bootstrap/state.ts`, `AppState.tsx` | — |
| 관찰가능성 | `queryProfiler.ts`, `analytics/index.ts`, `cost-tracker.ts` | — |

## 부록: 참고 자료

- Anthropic, "Effective Harnesses for Long-Running Agents" (2026)
- Anthropic, "Building Effective Agents" (2024)
- Anthropic, "Effective Context Engineering for AI Agents" (2025)
- Anthropic, "Building Agents with the Claude Agent SDK" (2025)
- Manus, "Context Engineering for AI Agents" (2025)
- LangChain, "Anatomy of an Agent Harness" (2026)
- Inngest, "Your Agent Needs a Harness, Not a Framework" (2025)
- MorphLLM, "Agent Engineering: IMPACT Framework" (2025)

---

*Agentic Harness Deep Dive v1.0*
*Enterprise Agentic LLM 하네스 설계 가이드*
