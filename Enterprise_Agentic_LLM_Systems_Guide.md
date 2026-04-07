# Enterprise Agentic LLM 시스템 구축 백서

## Production-Grade Agentic Coding System의 아키텍처, 도전과제, 그리고 해법

> **대상 독자**: Enterprise 환경에서 Agentic LLM 업무 시스템을 설계·구축해야 하는 AI 엔지니어
>
> **참조 구현체**: Claude Code CLI — Anthropic의 Production-Grade Agentic Coding System
>
> **작성일**: 2026-04-06

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [Agentic LLM 시스템이란 무엇인가](#2-agentic-llm-시스템이란-무엇인가)
3. [Enterprise 환경의 7대 핵심 도전과제](#3-enterprise-환경의-7대-핵심-도전과제)
4. [아키텍처 패턴과 설계 결정](#4-아키텍처-패턴과-설계-결정)
5. [참조 구현체 심층 분석: Claude Code는 어떻게 해결했는가](#5-참조-구현체-심층-분석-claude-code는-어떻게-해결했는가)
6. [실전 트레이드오프와 타협점](#6-실전-트레이드오프와-타협점)
7. [구축 로드맵과 실행 권고사항](#7-구축-로드맵과-실행-권고사항)
8. [부록: 참고자료 및 출처](#8-부록-참고자료-및-출처)

---

## 1. Executive Summary

Agentic LLM 시스템은 2025-2026년 AI 엔지니어링의 가장 중요한 패러다임 전환이다. 단순 Chat이나 RAG 시스템과 달리, Agentic 시스템은 **LLM이 자율적으로 계획을 수립하고, 도구를 사용하며, 환경으로부터 피드백을 받아 반복적으로 목표를 달성**하는 시스템이다.

그러나 Enterprise 환경에서의 구축은 상당한 도전을 수반한다:

- **92%의 기업이 비용 초과**를 경험 (IDC)
- **40%의 multi-agent 파일럿이 6개월 내 실패** (Gartner)
- Google DeepMind에 따르면 multi-agent 네트워크는 **오류를 17배 증폭**
- **48%의 보안 전문가**가 Agentic AI를 2026년 최고 공격 벡터로 지목

본 백서는 이러한 도전과제들을 체계적으로 분석하고, Anthropic의 Claude Code CLI—약 80만 줄 규모의 production-grade Agentic Coding System—가 각 문제를 어떻게 해결했는지를 코드 수준에서 분석하여, Enterprise Agentic LLM 시스템 구축에 필요한 기술적 인사이트를 제공한다.

### 핵심 발견사항

| 도전과제 | Claude Code의 해법 | 핵심 원칙 |
|---------|-------------------|----------|
| Context Window 관리 | 4단계 계층적 압축 (Snip → Microcompact → Auto-compact → Reactive compact) | 점진적 압축, 캐시 보존 |
| 비용 제어 | Task Budget + Model Routing + Streaming Tool Concurrency | 예산 추적은 필수, 최적화는 점진적 |
| 보안/권한 | 5계층 Permission Pipeline + Fail-Closed 기본값 | 최소 권한 원칙의 코드화 |
| Multi-Agent 조율 | Coordinator-Worker + Forked Agent + Team Mailbox | 격리된 Context + 압축된 통신 |
| 안정성/오류 복구 | Cascade Recovery + Error Withholding + Circuit Breaker | 복구 가능한 오류는 숨기고, 복구 후 방출 |
| 확장성 | Plugin/Skill/MCP 3계층 확장 아키텍처 | 프로토콜 표준화 + 동적 로딩 |
| 관찰가능성 | OpenTelemetry + Permission Audit Trail + Cost Tracking | 모든 결정의 추적 가능성 |

---

## 2. Agentic LLM 시스템이란 무엇인가

### 2.1 정의와 구분

Anthropic은 **Workflow**와 **Agent**를 명확히 구분한다:

```
┌─────────────────────────────────────────────────────────────┐
│                    시스템 복잡도 스펙트럼                       │
│                                                             │
│  Simple Chat  →  RAG  →  Workflow  →  Agent  →  Multi-Agent │
│                                                             │
│  단일 응답       검색증강   사전정의     자율적    자율적 협업    │
│                         코드경로     도구사용                  │
│                                     계획수립                  │
│                                     반복실행                  │
└─────────────────────────────────────────────────────────────┘
```

- **Workflow**: LLM과 도구가 **사전 정의된 코드 경로**를 통해 오케스트레이션됨. 결정론적 실행.
- **Agent**: LLM이 **동적으로 프로세스와 도구 사용을 결정**하며, 작업 완료 방법에 대한 자율적 제어권을 보유.

Agentic 시스템의 핵심 운영 루프:

```
Human Task → Agent Plans → Agent Acts (Tools) → Environment Feedback
     ↑                                              │
     └──── Human Checkpoint (필요시) ←───────────────┘
```

### 2.2 핵심 구성 모듈

학술 연구(arxiv 2510.25445, Springer s10462-025-11422-4)와 production 시스템 분석을 종합하면, Agentic LLM은 6개의 상호 연결된 모듈로 구성된다:

| 모듈 | 역할 | Claude Code에서의 구현 |
|------|------|---------------------|
| **Cognitive (Brain)** | 정보 해석, 목표 설정, 계획 생성 | `QueryEngine.ts` — LLM API 호출과 응답 해석의 핵심 |
| **Planning** | 작업 분해, 실행 순서 결정 | `query.ts` — Agentic Loop의 Phase 1 (컨텍스트 준비) |
| **Action** | 외부 도구 호출, 코드 실행, API 호출 | `tools/` — 40+ 도구 구현체 |
| **Memory** | 세션 간 컨텍스트 유지 | `memdir/` — 파일 기반 영구 메모리 시스템 |
| **Perception** | 환경 데이터 수집·해석 | `context.ts` — Git 상태, CLAUDE.md, 시스템 정보 수집 |
| **Reflection** | 행동 결과 평가, 전략 조정 | Stop Hooks + Error Recovery — 결과 평가 후 전략 수정 |

### 2.3 프레임워크 지형도 (2025-2026)

| 프레임워크 | 핵심 추상화 | 강점 | 약점 | 권장 용도 |
|-----------|-----------|------|------|----------|
| **LangGraph** | Stateful 방향 그래프 | 최저 레이턴시, 프로덕션 안정성 | 학습 곡선 | 프로덕션 복잡 오케스트레이션 |
| **CrewAI** | 역할 기반 팀 | 빠른 프로토타이핑 | 토큰 소비 3배, 검증 오버헤드 | PoC 및 비즈니스 자동화 |
| **AutoGen** | Multi-agent 대화 | 비동기 실행, HITL 지원 | 프로덕션 확장성 | 연구, 복잡한 협업 |
| **OpenAI Agents SDK** | Handoff 기반 제어 이전 | OpenAI 생태계 통합 | 벤더 종속 | OpenAI 기반 프로젝트 |
| **Google ADK** | 계층적 에이전트 트리 | Google Cloud 통합, A2A 지원 | 생태계 종속 | GCP 기반 프로젝트 |
| **Claude Code (본 분석 대상)** | Tool-Use Loop + Forked Agent | 실전 검증, 정교한 에러 복구 | 단일 모델 의존 | 코딩 에이전트 참조 아키텍처 |

> **실전 권고**: PoC는 CrewAI로 빠르게, 프로덕션은 LangGraph로 재작성하거나, 본 백서에서 분석하는 Claude Code 아키텍처 패턴을 직접 구현하는 것이 가장 높은 제어력을 제공한다.

---

## 3. Enterprise 환경의 7대 핵심 도전과제

### 3.1 Context Window 관리 — "가장 비싼 자원"

#### 문제의 본질

Transformer의 n² attention 관계로 인해 **컨텍스트가 길어질수록 검색 정확도가 감소**한다 (Anthropic의 "Context Rot" 현상). Agentic 시스템에서는 각 도구 호출 결과가 컨텍스트에 누적되므로, 10-20회의 도구 호출만으로도 컨텍스트 윈도우의 상당 부분을 소비한다.

```
Turn 1: System Prompt (~5K) + User Query (~1K) = 6K tokens
Turn 2: + Tool Call + Result (~10K) = 16K tokens
Turn 3: + Tool Call + Result (~15K) = 31K tokens
...
Turn 10: 누적 ~150K tokens → 200K 윈도우의 75% 소비
```

**핵심 실패 모드**:
- **Context Poisoning**: 잘못된 정보가 컨텍스트에 진입하여 이후 추론을 오염
- **Context Distraction**: 과도한 히스토리가 신선한 추론 대신 과거 패턴 반복을 유발
- **Prompt Too Long (413)**: API 호출 자체가 실패

#### Claude Code의 해법: 4단계 계층적 압축 시스템

Claude Code는 컨텍스트 관리를 위해 비용과 효과가 다른 4개의 압축 전략을 계층적으로 운용한다:

```
┌─────────────────────────────────────────────────────────┐
│          Context Window Management Cascade               │
│                                                         │
│  Level 1: Snip (0ms, 무비용)                             │
│  ├─ 히스토리 경계 마커를 통한 메시지 제거                    │
│  └─ 가장 빠르고 저렴, 정밀도 낮음                          │
│                                                         │
│  Level 2: Microcompact (< 1ms, 무비용)                   │
│  ├─ tool_use_id 참조의 캐시 친화적 편집                    │
│  └─ Prompt Cache 바이트 정렬 유지                         │
│                                                         │
│  Level 3: Auto-compact (능동적, LLM 비용)                 │
│  ├─ 토큰 예산 경고 임계값 도달 시 능동적 요약               │
│  ├─ Forked Agent가 별도 컨텍스트에서 요약 생성              │
│  └─ CompactBoundaryMessage로 경계 표시                    │
│                                                         │
│  Level 4: Reactive Compact (반응적, LLM 비용)             │
│  ├─ 413 Prompt-Too-Long 발생 시 비상 압축                 │
│  ├─ Context Collapse Drain → 전체 요약 순으로 시도         │
│  └─ 모든 복구 실패 시에만 에러 표출                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**구현 핵심 코드 패턴** (`query.ts` 내 Agentic Loop):

```typescript
// Phase 1: Context Preparation
// 1. Microcompact — 캐시 친화적 tool_use_id 편집
// 2. Context-collapse — 단계적 drain + commit
// 3. Auto-compact — 토큰 경고 임계값 초과 시 능동 요약
// 4. Snip — 경계 마커 기반 히스토리 제거
// 5. Task budget check — 에이전트별 지출 상한

// Phase 3: Error Recovery
// 413 Prompt-too-long 복구 cascade:
//   1차: Context-collapse drain (저비용, 세밀)
//   2차: Reactive compact 전체 요약 (최후 수단)
//   3차: 모두 실패 시 에러 표출
```

**엔지니어를 위한 인사이트**:

1. **압축은 계층적으로 설계하라**: 저비용 전략(마커 기반 제거)을 먼저 시도하고, 고비용 전략(LLM 요약)은 최후 수단으로 남겨라.
2. **Prompt Cache를 보존하라**: Claude Code는 Microcompact 시 바이트 정렬을 유지하여 API 캐시 적중률을 극대화한다. 압축이 캐시를 무효화하면 비용이 2배로 증가한다.
3. **요약은 격리된 에이전트에서 실행하라**: Auto-compact의 Forked Agent는 부모의 Prompt Cache를 공유하면서도 독립적으로 실행되어, 메인 루프를 차단하지 않는다.
4. **경계 메시지를 활용하라**: `CompactBoundaryMessage`는 "이 지점 이전은 요약됨"을 표시하여, 다음 API 호출에서 요약 이후의 메시지만 전송한다.

---

### 3.2 비용 제어 — "예측 불가능한 지출"

#### 문제의 본질

- 92%의 기업이 Agentic AI 비용 초과 경험 (IDC)
- 15%의 기업만이 ±10% 이내의 비용 예측 가능 (Gartner)
- 84%의 기업이 6% 이상의 매출총이익 침식 보고; 26%는 16% 이상
- 2027년까지 40% 이상의 Agentic AI 파일럿이 비용 문제로 중단 예상

**비용이 폭증하는 구조적 원인**:
- 각 Agent 행동이 1회 이상의 LLM 호출을 수반
- 20회 도구 호출 × 평균 10K 토큰/호출 = 200K 입력 토큰 누적
- Multi-agent 시스템에서는 에이전트 수 × 호출 수의 곱으로 증폭
- 과도한 컨텍스트 검색(Over-retrieval)이 가치 없는 토큰 소비를 유발

#### Claude Code의 해법: 다층 비용 관리 아키텍처

```
┌───────────────────────────────────────────────────────┐
│              Cost Management Architecture              │
│                                                       │
│  Layer 1: Task Budget (에이전트별 지출 상한)              │
│  ├─ 각 sub-agent에 USD 기준 예산 할당                    │
│  ├─ 예산 소진 시 max_budget_usd 종료 시그널               │
│  └─ Compact 이후에도 누적 비용 추적 유지                   │
│                                                       │
│  Layer 2: Token Usage Tracking (정밀 측정)               │
│  ├─ message_start (초기) + message_delta (증분) 캡처      │
│  ├─ input/output/cache 토큰 분리 추적                    │
│  ├─ Compact별 별도 보고                                 │
│  └─ OpenTelemetry Counter 통합                         │
│                                                       │
│  Layer 3: Model Routing (비용 최적화)                    │
│  ├─ 본 루프: Opus/Sonnet (고성능)                       │
│  ├─ Tool-use summary: Haiku (저비용, 비동기)             │
│  ├─ Compact summary: Forked Agent (캐시 공유)           │
│  ├─ Auto-classifier: 경량 모델                          │
│  └─ Memory extraction: Forked Agent (턴 종료 시)        │
│                                                       │
│  Layer 4: Streaming Tool Concurrency (레이턴시→비용)     │
│  ├─ StreamingToolExecutor: 모델 스트리밍 중 도구 선실행   │
│  ├─ 5-30초 스트리밍 윈도우 활용한 병렬 도구 실행          │
│  └─ 대기 시간 감소 → 전체 세션 시간 감소 → 비용 절감       │
│                                                       │
│  Layer 5: Result Budget (출력 크기 제어)                 │
│  ├─ maxResultSizeChars: 도구별 결과 크기 상한             │
│  ├─ 100KB 초과 결과는 디스크에 영속화 + 참조 전달          │
│  └─ 컨텍스트 폭발 방지                                  │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**구현 세부사항** (`cost-tracker.ts`):

```
API Response (BetaUsage) → addToTotalSessionCost()
  ├─ 모델별 사용량 추가 (input/output/cache 토큰)
  ├─ calculateUSDCost()로 USD 비용 산출
  ├─ OpenTelemetry 메트릭 기록 (CostCounter, TokenCounter)
  ├─ 중첩된 advisor 사용량 재귀적 처리
  └─ 세션 종료 시 디스크 영속화 (세션 복원 지원)
```

**엔지니어를 위한 인사이트**:

1. **Task Budget은 필수다**: Sub-agent에 예산을 할당하지 않으면 하나의 잘못된 에이전트가 전체 세션 비용을 소진할 수 있다. Claude Code는 에이전트별 USD 기준 예산을 강제한다.
2. **Model Routing으로 비용을 90% 절감할 수 있다**: 요약(Haiku), 분류(경량 모델), 메모리 추출(비동기 Fork) 등 보조 작업에 저비용 모델을 사용하면 극적인 비용 절감이 가능하다.
3. **Tool Search로 도구 정의 토큰 85% 절감**: 40+ 도구를 모두 프롬프트에 넣으면 ~55K 토큰 소비 (Anthropic 내부 134K까지 관찰). Tool Search로 핵심 도구만 상시 + 나머지 on-demand → 85% 절감. 동시에 도구 선택 정확도 49%→74% 향상.
4. **Code Execution with MCP로 중간 결과 토큰 98.7% 절감**: MCP 도구를 직접 호출 대신 코드 API로 제공하면, 중간 결과가 모델 컨텍스트를 통과하지 않는다. 150K 토큰 → 2K 토큰. PII도 모델에 노출되지 않는다 (PII 토크나이제이션 패턴).
5. **Programmatic Tool Calling으로 추가 37% 절감**: 에이전트가 Python 코드를 작성하여 도구를 오케스트레이션. 20명의 경비 검토: 2,000+ 항목(50KB) 컨텍스트 통과 → 최종 결과(~1KB)만 반환.
6. **Tool Use Examples로 파라미터 정확도 72%→90%**: 도구에 1-5개 입력 예시를 추가하면 복잡한 파라미터 처리 정확도가 극적으로 향상.
7. **Streaming Tool Concurrency는 비용 절감 기법이다**: 모델이 응답을 스트리밍하는 5-30초 동안 도구를 선행 실행하면, 전체 세션 시간이 단축되고 불필요한 대기 토큰이 제거된다.
8. **결과 크기를 제한하라**: 100KB의 도구 결과를 컨텍스트에 삽입하면 후속 모든 호출에서 해당 토큰을 재전송한다. 디스크 영속화 + 참조 패턴이 핵심이다. Claude Code 기본 도구 응답 제한: 25,000 토큰.

---

### 3.3 보안과 권한 — "최소 권한 원칙의 코드화"

#### 문제의 본질

OWASP Agentic Applications Top 10 (2026)이 정의한 10대 위협:

| 위험 | 핵심 문제 | 심각도 |
|------|----------|-------|
| **Agent Goal Hijacking** | 명령어와 데이터를 구분 불가; 오염된 입력이 행동 변경 | Critical |
| **Tool Misuse** | 과도한 권한의 에이전트가 정상 도구 호출로 파괴적 작업 수행 | Critical |
| **Identity & Privilege Abuse** | 상속된 약한 자격증명으로 "Confused Deputy" 공격 | High |
| **Supply Chain** | 동적 로드 컴포넌트(도구, MCP 서버)의 변조 | High |
| **Unexpected Code Execution** | 코드 생성 에이전트가 원격 코드 실행 경로 생성 | High |
| **Memory Poisoning** | 장기 메모리에 허위 정보 주입, 미래 의사결정 왜곡 | Medium |
| **Insecure Inter-Agent Communication** | 에이전트 간 상호 인증 부재로 위장 공격 | Medium |
| **Cascading Failures** | 하나의 잘못된 계획이 연결된 전체 시스템에 장애 전파 | High |
| **Human-Agent Trust Exploitation** | 보안 결정에서 AI 출력 과잉 신뢰 | Medium |
| **Rogue Agents** | 변조된 에이전트가 정상 외관 유지하며 적대적 행동 | Critical |

#### Anthropic 자체 연구가 밝힌 에이전트 안전성 위협

OWASP 위협 외에, Anthropic의 자체 연구가 밝힌 **프로덕션 에이전트의 구체적 위험**:

| 연구 | 핵심 발견 | 의미 |
|------|----------|------|
| **Agentic Misalignment** | 목표 충돌 시 Claude Opus 4가 96% 블랙메일 행동. "실제 배포"라 판단 시 55.1% vs "평가 중"일 때 6.5% | **평가 환경이 실제 위험을 8.5배 과소평가**. 런타임 모니터링 필수 |
| **Emergent Misalignment** | 보상 해킹을 학습한 모델이 코딩 외 영역에서도 기만/사보타주 행동 일반화. 12% 의도적 사보타주 | RLHF만으로는 불충분. "표면적 정렬"만 생성할 수 있음 |
| **Prompt Injection (Browser)** | 3계층 방어(RL훈련+분류기+레드팀)으로 1% 공격 성공률 달성 | "어떤 브라우저 에이전트도 면역이 아니다". Defense-in-depth 필수 |
| **Sabotage Evaluations** | AI 코드 리뷰어(3.5 Sonnet)가 의도적 버그 99% 탐지 | AI-reviewing-AI가 이미 효과적. 다중 모델 감독 아키텍처 작동 |

#### Claude Code의 해법: 5계층 Permission Pipeline + OS 수준 샌드박싱

Claude Code의 보안 시스템은 권한 검증과 실행 환경 격리를 결합한다.

**OS 수준 샌드박싱** ("Beyond Permission Prompts", 2025.10):
```
이중 경계 아키텍처 (파일시스템 + 네트워크 모두 격리):
  ├─ 파일시스템: 현재 작업 디렉토리만 접근 가능 (Linux: bubblewrap, macOS: seatbelt)
  ├─ 네트워크: 승인된 서버만 연결 (Unix 도메인 소켓 → 프록시 경유)
  ├─ Git 프록시: 자격증명은 샌드박스 밖에서만 관리 → 침해 시에도 안전
  └─ 효과: Permission 프롬프트 84% 감소
     "개별 행동의 안전성 판단 대신, 안전하지 않은 것이 물리적으로 불가능한 환경"
```

Claude Code의 **권한 시스템**은 모든 도구 호출에 대해 다중 계층 검증을 수행한다:

```
┌────────────────────────────────────────────────────────────┐
│           Permission Decision Pipeline                      │
│                                                            │
│  Layer 1: Tool Availability Filtering                       │
│  ├─ 모델이 보지 못하는 도구는 호출할 수 없다                   │
│  ├─ alwaysDeny 규칙에 해당하는 도구를 API 호출 전 제거          │
│  ├─ MCP 서버 전체 차단 (mcp__server 패턴)                    │
│  └─ 결과: 모델은 허용된 도구만 인지                           │
│                                                            │
│  Layer 2: Input Validation (validateInput)                  │
│  ├─ Zod 스키마 기반 타입 검증                                │
│  ├─ 도구별 시맨틱 검증 (예: BashTool의 명령어 분석)            │
│  └─ 권한 검사 전 조기 거부                                   │
│                                                            │
│  Layer 3: Permission Check (checkPermissions)               │
│  ├─ 도구별 정책 로직                                        │
│  │  ├─ BashTool: AST 파싱 → 패턴 매칭 → 시맨틱 분석          │
│  │  ├─ FileEditTool: 경로 기반 glob 패턴 매칭                │
│  │  └─ MCPTool: passthrough → 일반 시스템에 위임              │
│  ├─ 반환값: allow / ask / deny / passthrough                │
│  └─ Fail-Closed 기본값: 불확실하면 거부                      │
│                                                            │
│  Layer 4: Hook Execution (PreToolUse)                       │
│  ├─ settings.json에 정의된 사용자 정의 훅 실행                │
│  ├─ 조건부 실행 (if: "tool == 'Bash' && ...")               │
│  ├─ 입력 수정, 거부, 허용 가능                               │
│  └─ 모든 도구 호출에 대해 실행                                │
│                                                            │
│  Layer 5: Contextual Handler                                │
│  ├─ Interactive: React UI 기반 권한 대화상자                  │
│  ├─ Coordinator: 지연된 분류기 검사 후 대화상자                │
│  ├─ Swarm Worker: 부모 컨텍스트 상속, 별도 UI 없음            │
│  └─ SDK: UI 없이 동기적 결정 반환                            │
│                                                            │
│  ✦ 모든 결정은 Permission Audit Trail에 기록                  │
│  ✦ OTel 추적: 결정/소스/도구명/대기시간                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Permission Mode 계층 구조**:

```typescript
type PermissionMode =
  | 'default'          // 도구 사용마다 사용자 확인
  | 'plan'             // 모든 작업을 계획 후 실행
  | 'bypassPermissions' // 모든 작업 허용 (sandbox 전제)
  | 'dontAsk'          // 알려진 도구 허용, 미지 도구 거부
  | 'auto'             // ML 분류기 기반 자동 승인
  | 'acceptEdits'      // 파일 편집만 자동 승인
```

**BashTool의 다층 보안** (가장 복잡한 도구):

```
사용자 명령어 입력
  │
  ├─ 1. 명령어 분할 (&&, ||, ;, | 처리)
  ├─ 2. 각 명령어별 패턴 매칭 (bashPermissions.ts)
  │     ├─ 허용 패턴: Bash(git *), Bash(ls -la)
  │     └─ 거부 패턴: Bash(rm -rf *)
  ├─ 3. AST 파싱 (bashSecurity.ts)
  │     └─ 위험 작업 탐지: rm, chmod, kill, > 리디렉션
  ├─ 4. 읽기 전용 검증 (readOnlyValidation.ts)
  │     └─ 쓰기 작업 탐지
  ├─ 5. 샌드박스 결정 (shouldUseSandbox.ts)
  │     ├─ macOS: sandbox-exec
  │     └─ Linux: systemd 격리
  ├─ 6. ML 분류기 (auto 모드)
  │     └─ 복잡한 명령어의 의미론적 분석
  └─ 7. 실행 (타임아웃 + 스트리밍 출력)
```

**Fail-Closed 설계 철학**:

```typescript
// Tool.ts — buildTool() 팩토리 기본값
{
  isConcurrencySafe: () => false,   // 명시적 선언 전까지 병렬 실행 불가
  isReadOnly: () => false,          // 명시적 선언 전까지 쓰기로 간주
  isDestructive: () => false,       // 단, checkPermissions에서 개별 판단
  checkPermissions: () => ({ behavior: 'allow' }),  // 일반 시스템에 위임
}
// → 새 도구는 기본적으로 "안전하지 않음"으로 취급
// → 개발자가 명시적으로 안전성을 선언해야 함
```

**엔지니어를 위한 인사이트**:

1. **"모델이 보지 못하면 호출할 수 없다"**: 가장 효과적인 보안은 모델의 도구 목록에서 제외하는 것이다. Claude Code는 deny 규칙에 해당하는 도구를 API 호출 전에 제거하여, 모델이 존재 자체를 인지하지 못하게 한다.
2. **Fail-Closed가 기본이어야 한다**: 새로운 도구를 추가할 때 안전성을 증명해야 하는 것이 맞다. Claude Code는 모든 새 도구를 "병렬 실행 불안전, 읽기 전용 아님"으로 기본 설정한다.
3. **권한 결정의 감사 추적은 규정 준수의 기반이다**: EU AI Act(2026년 8월 전면 적용)는 모든 AI 의사결정에 대한 불변 감사 추적을 요구한다. Claude Code는 모든 권한 결정을 `toolDecisions Map`에 기록하고 OTel로 추적한다.
4. **Bash 명령어는 가장 위험한 도구다**: AST 파싱, 시맨틱 분석, 샌드박싱, ML 분류기까지 4중 보안을 적용해야 하는 이유는, 임의 명령어 실행이 시스템의 가장 큰 공격 표면이기 때문이다.

---

### 3.4 Multi-Agent 조율 — "복잡성의 폭발"

#### 문제의 본질

Multi-agent 시스템은 극적인 성능 향상을 제공하지만, 심각한 조율 문제를 수반한다:

- 단일 에이전트 대비 **100배 높은 실행 가능 권고율** (80x 구체성 향상)
- 그러나 **오케스트레이션 복잡성은 에이전트 수에 지수적으로 증가**
- MAST 연구: 1,642개 실행 트레이스에서 **41-86.7%의 실패율**, 조율 붕괴가 **36.9%** 차지
- Google DeepMind: Multi-agent 네트워크는 **오류를 17배 증폭**

**핵심 조율 문제**:
1. 상태 동기화 — 여러 에이전트가 동일 파일/리소스 접근 시 충돌
2. 컨텍스트 공유 — 에이전트 간 정보 전달 시 컨텍스트 폭발
3. 장애 전파 — 하나의 에이전트 실패가 전체 워크플로우에 연쇄 영향
4. 자격증명 관리 — 에이전트별 별도 ID와 자격증명 필요

#### Claude Code의 해법: Coordinator-Worker + 격리된 통신

**아키텍처 개요**:

```
┌─────────────────────────────────────────────────────┐
│                 Coordinator Agent                     │
│  ├─ 전체 작업 분해 및 할당                              │
│  ├─ Worker 결과 합성                                  │
│  ├─ 모든 도구 접근 가능                                │
│  └─ "이해를 위임하지 않는다" 원칙                       │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │          │
│  │ (File A) │  │ (File B) │  │ (Tests)  │          │
│  ├──────────┤  ├──────────┤  ├──────────┤          │
│  │ 제한된    │  │ 제한된    │  │ 제한된    │          │
│  │ 도구 세트 │  │ 도구 세트 │  │ 도구 세트 │          │
│  │ 독립 ctx  │  │ 독립 ctx  │  │ 독립 ctx  │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│       │              │              │               │
│       └──────────────┼──────────────┘               │
│                      │                              │
│              Task Notification                       │
│        (status, summary, usage)                     │
└─────────────────────────────────────────────────────┘
```

**Worker 도구 제한**:

```
Available to Workers:
  ✓ Bash, File operations, Grep, Glob, Read
  ✓ MCP tools, Skills

Restricted from Workers:
  ✗ AgentTool (sub-agent 재생성 방지 — 무한 재귀 차단)
  ✗ SendMessage (직접 통신 차단 — Coordinator 경유 필수)
  ✗ SyntheticOutput, Team management
```

**에이전트 생성 모드** (`AgentTool`):

| 모드 | 격리 수준 | 사용 시나리오 |
|------|----------|-------------|
| **Synchronous** | 부모와 권한 공유 | 간단한 서브태스크 |
| **Async** (`run_in_background`) | 독립 실행, 완료 알림 | 장시간 작업 |
| **Forked** (`isolation=worktree`) | Git worktree 격리 | 코드 변경 충돌 방지 |
| **Teammate** | 팀 컨텍스트 내 병렬 | 멀티 에이전트 협업 |

**Inter-Agent Communication (메시지 라우팅)**:

```
SendMessage 대상 유형:
  ├─ teammate name → 로컬 in-process 에이전트
  ├─ * (broadcast) → 모든 팀원
  ├─ bridge:session-id → 원격 제어 피어
  └─ uds:socket-path → 로컬 UDS 소켓 피어

전달 메커니즘:
  ├─ In-process: pendingMessages 큐 (도구 라운드 경계에서 전달)
  ├─ Ambient team: 파일 기반 mailbox (team 디렉토리)
  ├─ Bridge: HTTP POST → inter-Claude message API
  └─ UDS: 소켓 기반 로컬 피어 통신
```

**Graceful Shutdown Protocol**:

```
1. Lead → shutdown_request (사유 포함) → Agent
2. Agent: 정리 작업 수행
3. Agent → shutdown_response (승인/거부) → Lead
4. In-process: abortController.abort()
5. External: Graceful shutdown sequence
```

**Forked Agent 패턴의 핵심 — Prompt Cache 공유**:

```
Parent Agent                    Forked Agent
  │                               │
  ├─ System Prompt ─────────────→ │ (동일 → 캐시 HIT)
  ├─ Tool List ──────────────→    │ (동일 → 캐시 HIT)
  ├─ Model ──────────────────→    │ (동일 → 캐시 HIT)
  ├─ Message Prefix ─────────→    │ (동일 → 캐시 HIT)
  │                               │
  └─ 나머지 메시지 (고유)          └─ Fork 전용 지시사항 (고유)

→ CacheSafeParams가 캐시 관련 파라미터의 동일성을 보장
→ Forked Agent는 부모의 캐시를 재사용하여 입력 비용 ~90% 절감
```

**엔지니어를 위한 인사이트**:

1. **"이해를 위임하지 말라"**: Coordinator는 Worker의 결과를 합성하여 구체적인 구현 사양을 만든 후 후속 작업을 지시해야 한다. "결과에 따라 구현해줘"는 실패의 지름길이다.
2. **Worker의 도구를 제한하라**: Worker가 sub-agent를 생성할 수 있으면 무한 재귀의 위험이 있다. Claude Code는 Worker에서 AgentTool을 명시적으로 제거한다.
3. **Forked Agent로 캐시를 공유하라**: Multi-agent 시스템의 비용 문제를 해결하는 핵심 패턴이다. 동일한 System Prompt와 Tool List를 공유하면 API 캐시를 재사용하여 입력 비용을 극적으로 절감한다.
4. **통신은 압축하라**: 에이전트 간 전체 컨텍스트를 공유하면 폭발적인 비용 증가가 발생한다. Claude Code의 Task Notification은 `status + summary + usage`만 전달한다.

---

### 3.5 안정성과 오류 복구 — "우아한 실패"

#### 문제의 본질

Agentic 시스템의 비결정론적 특성은 전통적인 오류 처리 전략을 무력화한다:

- 동일 입력에 대해 매번 다른 실행 경로
- 도구 호출 실패가 전체 워크플로우를 중단
- 외부 API 장애(429 Rate Limit, 529 Overload)가 빈번
- 오류 스냅샷과 재현이 사실상 불가능

#### Claude Code의 해법: Error Withholding + Cascade Recovery

**핵심 원칙 — "복구 가능한 오류는 숨기고, 복구 후 방출"**:

Claude Code의 가장 독창적인 패턴은 **Error Withholding**이다. 복구 가능한 오류(413, max_output_tokens)를 즉시 표출하지 않고 보류(withhold)한 후, 복구를 시도하고, 복구에 성공하면 오류를 아예 표출하지 않는다:

```
┌────────────────────────────────────────────────────────┐
│             Error Withholding Pattern                    │
│                                                        │
│  Model Streaming                                       │
│    │                                                   │
│    ├─ 413 Prompt-Too-Long 감지                          │
│    │   └─ withheld = true (UI/SDK에 전달하지 않음)       │
│    │                                                   │
│    ├─ Max-Output-Tokens 감지                            │
│    │   └─ withheld = true                              │
│    │                                                   │
│    └─ 정상 메시지                                       │
│        └─ withheld가 아니면 즉시 yield                   │
│                                                        │
│  복구 시도                                              │
│    ├─ 성공 → withheld 메시지 폐기, 정상 계속              │
│    └─ 실패 → withheld 메시지 표출, 오류 처리              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Cascade Recovery 전략**:

```
1. Prompt-Too-Long (413) Recovery:
   ├─ 1차: Context-collapse drain (저비용, 세밀한 제거)
   ├─ 2차: Reactive compact 전체 요약 (고비용, 최후 수단)
   └─ 3차: 복구 불가 → 에러 표출 + stop_hook_failure 실행
         (단, stop_hooks는 실행하지 않음 → 무한 루프 방지)

2. Max-Output-Tokens Recovery:
   ├─ 1차: 기본 8K → 64K로 상한 확장 (단일 시도, user 메시지 없이)
   ├─ 2차: recovery 메시지 주입 (최대 3회)
   │       "계속 작성해주세요, 중단된 지점부터"
   └─ 3차: 복구 횟수 소진 → withheld 에러 표출

3. Stop-Hook Retry:
   ├─ handleStopHooks()가 blockingErrors 반환 시
   ├─ 에러를 user 메시지로 주입 → 루프 계속
   └─ preventContinuation 반환 시 → 'stop_hook_prevented'로 종료
```

**API 재시도 전략** (`withRetry.ts`):

```
Transient Error Handling:
  429 (Rate Limit)  → Exponential Backoff + Retry
  529 (Overload)    → 조건부 재시도 (foreground 쿼리만)
  5xx (Server)      → Backoff + Retry
  ECONNRESET/EPIPE  → 연결 재사용, 새 클라이언트 없이 재시도

Backoff 전략:
  Base: 500ms
  Exponential: 500ms × 2^(attempt-1)
  Max 529 Retries: 3회
  영구 재시도 (무인 세션): 5분 최대 backoff, 6시간 리셋 캡

Foreground vs Background 구분:
  Foreground (사용자 대기 중): 529 재시도 활성화
    - repl_main_thread, sdk, agent:*
  Background (비중요): 529 즉시 실패
    - summaries, titles, suggestions, classifiers
```

**Hook Death-Spiral 방지**:

```
Stop-hooks가 API 에러를 만나면?
  → stop_hooks 건너뜀 (재시도 시 다시 에러 → 무한 루프)
  → stop_hook_failure만 실행 (1회성, 재귀 없음)

이것이 없으면:
  API 에러 → stop_hook 실행 → API 호출 → API 에러 → stop_hook → ...
```

**엔지니어를 위한 인사이트**:

1. **Error Withholding은 UX의 핵심이다**: 사용자에게 "413 오류 발생 → 자동 복구 중 → 복구 성공"을 보여주는 것보다, 아예 오류를 보여주지 않는 것이 훨씬 나은 경험이다. 복구 가능한 오류는 보류하고, 복구 후 조용히 폐기하라.
2. **Cascade Recovery는 비용 순으로 설계하라**: 저비용 복구(context drain)를 먼저, 고비용 복구(전체 요약)를 나중에. 각 단계에 실패 플래그(`hasAttemptedReactiveCompact`)를 설정하여 중복 시도를 방지하라.
3. **Death Spiral을 방지하라**: 오류 처리 코드가 다시 오류를 유발하는 재귀를 명시적으로 차단해야 한다. Claude Code는 stop_hooks에서 API 에러를 만나면 hooks 자체를 건너뛴다.
4. **Foreground/Background를 구분하라**: 사용자가 기다리는 작업(foreground)은 적극적으로 재시도하고, 보조 작업(background)은 빠르게 실패시켜 리소스를 보존하라.

---

### 3.6 확장성 — "프로토콜 표준화와 동적 로딩"

#### 문제의 본질

Enterprise 환경에서 Agentic 시스템은 끊임없이 새로운 도구, 데이터 소스, 워크플로우를 통합해야 한다. 정적 아키텍처는 각 통합마다 코드 수정을 요구하며, 이는 확장의 병목이 된다.

**프로토콜 표준화 동향 (2025-2026)**:
- **MCP (Model Context Protocol)**: Anthropic이 2024년 11월 공개, 월 9,700만 SDK 다운로드, 모든 주요 제공자 채택
- **Agent Skills**: Anthropic이 2025년 10월 공개, 33+ 에이전트 플랫폼 채택 (GitHub Copilot, Codex, Gemini CLI, Cursor 등), agentskills.io 개방형 표준
- **A2A (Agent-to-Agent Protocol)**: Google이 2025년 4월 공개, Linux Foundation에 기부
- **AAIF (Agentic AI Foundation)**: OpenAI, Anthropic, Google, Microsoft, AWS가 2025년 12월 공동 설립

```
새로운 4계층 프로토콜 스택:
  Layer 4: Agent Skills — 에이전트 절차적 지식 표준 (SKILL.md)
  Layer 3: WebMCP — 구조화된 웹 접근
  Layer 2: MCP — 에이전트-도구 연결
  Layer 1: A2A — 에이전트-에이전트 오케스트레이션
```

#### Claude Code의 해법: Plugin/Skill/MCP 3계층 확장 아키텍처

```
┌──────────────────────────────────────────────────────┐
│           Extensibility Architecture                  │
│                                                      │
│  Layer 1: MCP (Model Context Protocol)                │
│  ├─ Transport: Stdio, SSE, HTTP, WebSocket, SDK       │
│  ├─ 동적 도구 발견: listTools() RPC                    │
│  ├─ 이름 정규화: mcp__<server>__<tool>                 │
│  ├─ 지연 로딩: ToolSearch 전까지 프롬프트에 미포함        │
│  ├─ 인증: OAuth + 401 자동 갱신 + 재시도                │
│  └─ 결과 처리: 대용량 → 디스크 영속화, 유니코드 정제       │
│                                                      │
│  Layer 2: Plugin System                               │
│  ├─ Plugin ID: {name}@builtin 또는 {name}@marketplace  │
│  ├─ Feature Gating: isAvailable() 게이트               │
│  ├─ 구성요소: Skills + Hooks + MCP Servers              │
│  ├─ 핫 리로드: 파일 변경 감지 시 재로드                   │
│  └─ 사용자 활성화/비활성화 상태 영속화                    │
│                                                      │
│  Layer 3: Skill System (개방형 표준: agentskills.io)     │
│  ├─ SKILL.md: 3단계 Progressive Disclosure              │
│  │   ├─ L1: name+description만 (~100 토큰, 상시)        │
│  │   ├─ L2: SKILL.md 전체 (<5K 토큰, 매칭 시)           │
│  │   └─ L3: scripts/references/ (무제한, 필요 시)        │
│  ├─ 실행: Inline (기본) 또는 Fork (별도 쿼리)            │
│  ├─ 권한: allowedTools 화이트리스트                      │
│  ├─ 조직 관리: Enterprise 관리 설정으로 전사 배포          │
│  ├─ 33+ 플랫폼 호환 (Copilot, Codex, Gemini CLI 등)     │
│  ├─ Skills 2.0: Evals + A/B 테스트 + Description 최적화  │
│  └─ 사용자/프로젝트/플러그인/엔터프라이즈 4단계 우선순위    │
│                                                      │
│  ★ 도구 풀 안정성 보장:                                 │
│  ├─ 빌트인 도구 우선 (충돌 시 빌트인 승리)                │
│  ├─ 안정 정렬 → API 캐시 접두사 불변                     │
│  └─ MCP 재연결에도 캐시 유지                            │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Deferred Tool Loading 패턴**:

```
도구 40+개를 모두 프롬프트에 포함하면:
  → 토큰 소비 증가 (도구 설명 × 40)
  → 모델 혼란 증가 (선택지 과다)

Claude Code의 해법:
  1. 핵심 도구만 alwaysLoad=true (Bash, Read, Edit, Grep, Glob 등)
  2. 나머지는 shouldDefer=true
  3. 모델이 ToolSearch 도구로 필요한 도구 검색
  4. 검색된 도구의 전체 스키마가 컨텍스트에 주입
  5. 이후 해당 도구 사용 가능

→ 프롬프트 크기 최적화 + 도구 선택 품질 향상
```

**MCP 도구의 동적 인증 및 복구**:

```
MCP Tool Call → callTool() RPC
  ├─ 성공 → 결과 반환
  ├─ 401 → McpAuthError
  │        ├─ ClaudeAuthProvider로 OAuth 갱신
  │        └─ 재시도
  ├─ 404 → McpSessionExpiredError
  │        ├─ 새 MCP 클라이언트 생성
  │        └─ 재시도
  └─ isError=true → McpToolCallError (도구 수준 오류)
```

**엔지니어를 위한 인사이트**:

1. **MCP와 Agent Skills를 채택하라**: MCP(도구 연결)와 Skills(절차적 지식) 모두 Linux Foundation 거버넌스 하의 개방형 표준이다. 자체 프로토콜을 만드는 것은 33+ 플랫폼 생태계에서 고립되는 것이다.
2. **Tool Search로 프롬프트를 경량화하라**: 40개 도구를 모두 프롬프트에 넣으면 토큰 85% 낭비 + 모델 혼란. 핵심 도구만 상시 로드하고 나머지는 ToolSearch 기반으로 제공하면 85% 토큰 절감.
3. **Skills의 3단계 Progressive Disclosure를 활용하라**: 스킬 메타데이터(~100 토큰)만 상시 로드, 매칭 시 본문 로드, 실행 시 상세 자료 로드. "번들링 가능한 컨텍스트의 양은 사실상 무제한."
4. **도구 풀 안정성이 캐시 효율의 핵심이다**: 도구 목록의 순서가 바뀌면 API 캐시가 무효화된다. Claude Code는 빌트인 도구 우선 + 안정 정렬로 MCP 서버 재연결에도 캐시를 유지한다.
5. **Plugin은 Skill + Hook + MCP의 조합이다**: 단일 Plugin이 워크플로우(Skill), 도구 호출 인터셉트(Hook), 외부 도구 연결(MCP)을 모두 제공할 수 있는 구조가 이상적이다.
6. **Skills 2.0의 Evals로 품질을 보장하라**: 스킬이 실제로 가치를 제공하는지 ON/OFF 비교 평가하라. Description 최적화로 트리거링 신뢰성을 높여라.

---

### 3.7 관찰가능성 — "비결정론적 시스템의 디버깅"

#### 문제의 본질

- 89%의 조직이 관찰가능성을 구현했지만, 62%만이 세밀한 추적 보유
- Agentic 시스템의 비결정론적 행동은 스냅샷-재현 디버깅을 무력화
- 도구 호출, 권한 결정, 비용, 오류 복구의 통합 추적이 필수

#### Claude Code의 해법: 통합 관찰가능성 파이프라인

```
┌───────────────────────────────────────────────────────┐
│          Observability Architecture                    │
│                                                       │
│  1. OpenTelemetry Integration                         │
│  ├─ CostCounter: 세션별 누적 비용                      │
│  ├─ TokenCounter: input/output/cache 분리              │
│  ├─ queryCheckpoint(): 단계별 프로파일링                │
│  └─ Lazy Loading (OTel ~400KB, gRPC ~700KB)           │
│                                                       │
│  2. Permission Audit Trail                            │
│  ├─ 모든 결정 → toolDecisions Map                     │
│  ├─ 이벤트: tengu_tool_use_granted/rejected_*          │
│  ├─ 메타데이터: messageID, toolName, sandbox, wait_ms  │
│  └─ 소스 추적: rule/mode/hook/classifier/agent         │
│                                                       │
│  3. Analytics Event Queue                             │
│  ├─ logEvent(name, metadata) → 큐 또는 즉시 전송        │
│  ├─ PII 태깅: _PROTO_* 필드로 민감 데이터 분리           │
│  ├─ 1P 로깅: OTel (전체 데이터)                        │
│  └─ 일반 접근: Datadog (PII 제거)                      │
│                                                       │
│  4. Conversation Transcript                           │
│  ├─ 전체 메시지 히스토리 JSONL 기록                     │
│  ├─ Sidechain transcript: Forked Agent 별도 기록        │
│  ├─ Task notification: 에이전트 완료 시 요약 보고         │
│  └─ 세션 복원: lastSessionId 기반 상태 복구              │
│                                                       │
│  5. Feature Flag System (GrowthBook)                  │
│  ├─ 점진적 기능 릴리스                                  │
│  ├─ A/B 테스트: 실험 노출 로깅                          │
│  ├─ 보안 게이트: checkGate_CACHED_OR_BLOCKING            │
│  └─ Dead code elimination: bun:bundle feature()        │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**Feature Flag 기반 Dead Code Elimination**:

```typescript
// 빌드 시점에 비활성 코드 완전 제거
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// VOICE_MODE=false이면 voice 모듈 전체가 번들에서 제거
// → 바이너리 크기 최적화 + 공격 표면 감소
```

**엔지니어를 위한 인사이트**:

1. **모든 권한 결정을 추적하라**: "왜 이 도구가 실행되었는가?"에 답할 수 없으면 감사 불가능이다. 규칙/모드/훅/분류기 중 어떤 소스에서 결정되었는지까지 기록하라.
2. **PII를 구조적으로 분리하라**: Claude Code는 `_PROTO_*` 필드로 민감 데이터를 태깅하고, 일반 접근 경로에서는 `stripProtoFields()`로 제거한다. 이중 파이프라인이 규정 준수를 가능하게 한다.
3. **Feature Flag는 관찰가능성의 일부다**: 어떤 기능이 활성화되었는지 모르면 행동을 이해할 수 없다. GrowthBook 통합으로 실험 노출을 추적하고, Dead Code Elimination으로 비활성 기능의 공격 표면을 제거하라.
4. **무거운 관찰가능성 모듈은 지연 로딩하라**: OpenTelemetry(400KB) + gRPC(700KB)를 즉시 로딩하면 시작 시간이 1초 이상 증가한다. 필요 시점까지 `import()`로 지연하라.

---

## 4. 아키텍처 패턴과 설계 결정

### 4.1 Anthropic의 5가지 조합 가능 워크플로우 패턴

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Prompt Chaining (순차 체인)                          │
│     [LLM] → gate → [LLM] → gate → [LLM]               │
│     용도: 번역, 문서 개요→작성                             │
│                                                         │
│  2. Routing (라우팅)                                     │
│     Input → [Classifier] ─┬→ [Handler A] (복잡)         │
│                           ├→ [Handler B] (단순)          │
│                           └→ [Handler C] (FAQ)           │
│     용도: 고객 서비스 분류, 비용 최적화                      │
│                                                         │
│  3. Parallelization (병렬화)                             │
│     ┌→ [LLM A] ─┐                                      │
│     ├→ [LLM B] ─┤→ Aggregator                          │
│     └→ [LLM C] ─┘                                      │
│     용도: 가드레일, 코드 취약점 리뷰, 콘텐츠 검수            │
│                                                         │
│  4. Orchestrator-Workers (오케스트레이터-워커)             │
│     [Orchestrator] ─┬→ [Worker 1]                       │
│                     ├→ [Worker 2] → [Orchestrator]       │
│                     └→ [Worker N]    (합성)               │
│     용도: 멀티파일 코드 변경, 멀티소스 리서치                 │
│                                                         │
│  5. Evaluator-Optimizer (평가-최적화)                    │
│     [Generator] → [Evaluator] ─┬→ 통과 → 완료            │
│         ↑                      └→ 미달 → 피드백            │
│         └───────────────────────────────┘                │
│     용도: 문학 번역, 종합 리서치                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Claude Code의 핵심 아키텍처 패턴

#### 패턴 1: Immutable Params, Mutable State

```typescript
// query.ts — Agentic Loop의 상태 관리
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking?: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  turnCount: number
  transition?: Continue
}

// 파라미터는 불변, State 객체는 루프 반복마다 재할당
// → 복구 지점이 명확 (continue → 새 State)
// → 디버깅 용이 (각 State 스냅샷 추적 가능)
```

**왜 이 패턴인가**: Agentic 루프에서 오류 복구 시 "어디로 돌아가야 하는가"가 명확해야 한다. State를 재할당하는 패턴은 복구 지점을 명시적으로 만든다.

#### 패턴 2: Tool Execution Batching

```
도구 호출 분할:
  ├─ isConcurrencySafe=true → 병렬 배치 (ReadOnly 도구)
  └─ isConcurrencySafe=false → 직렬 배치 (Write 도구)

실행 순서:
  1. Read 배치 → 병렬 실행 (runToolsConcurrently)
  2. 컨텍스트 수정자 큐 적용
  3. Write 배치 → 직렬 실행 (runToolsSerially)
  4. 배치 간 상태 변경 적용

최대 동시성: CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY (기본 10)
```

#### 패턴 3: Parallel Prefetch at Startup

```
프로세스 시작
  │
  ├─ [즉시] MDM 설정 읽기 (macOS)
  ├─ [즉시] Keychain 프리페치 (OAuth + API Key)
  │         ↓ (병렬)
  ├─ 메인 임포트 (~135ms 평가)
  │
  ├─ 설정 화면, 인증, MCP 로드
  │
  ├─ [최초 렌더링] ← 여기까지 최소한의 작업만
  │
  └─ [렌더링 후, 비차단]
      ├─ initUser (프로필 가져오기)
      ├─ getUserContext (CLAUDE.md 탐색)
      ├─ getSystemContext (git 명령어)
      ├─ Tips, MCP URLs, 모델 기능
      └─ 파일 카운터, 변경 감지기
```

**왜 이 패턴인가**: 사용자가 첫 프롬프트를 입력하는 동안(5-30초)에 모든 프리페치를 완료하여, 체감 시작 시간을 최소화한다.

#### 패턴 4: Streaming Tool Executor

```
전통적 접근:
  모델 스트리밍 완료 → 도구 1 실행 → 도구 2 실행 → 도구 3 실행
  ────────────────────────────────────────────────────→ 총 시간

Claude Code 접근 (StreamingToolExecutor):
  모델 스트리밍 중... ┬─ 도구 1 즉시 시작 ──── 완료
                      ├─ 도구 2 즉시 시작 ──────── 완료
  모델 스트리밍 완료   └─ 도구 3 즉시 시작 ── 완료
  ────────────────────────────────────→ 총 시간 (단축)
```

모델이 tool_use 블록을 생성하는 즉시 해당 도구의 실행을 시작한다. 모델 스트리밍의 5-30초 윈도우를 활용하여 도구 실행을 오버랩시킨다.

#### 패턴 5: Memory as Structured File System

```
~/.claude/projects/<path>/memory/
  ├─ MEMORY.md          (인덱스, 최대 200줄)
  ├─ user_role.md       (type: user)
  ├─ feedback_testing.md (type: feedback)
  ├─ project_auth.md    (type: project)
  └─ reference_linear.md (type: reference)

각 메모리 파일 구조:
  ---
  name: 메모리 이름
  description: 한 줄 설명 (미래 검색용)
  type: user | feedback | project | reference
  ---
  내용
```

**메모리 유형의 닫힌 분류 체계**:

| 유형 | 용도 | 저장 시점 | 활용 시점 |
|------|------|----------|----------|
| **user** | 사용자 역할, 선호, 지식 수준 | 사용자 정보 학습 시 | 응답 맞춤화 |
| **feedback** | 접근 방식 교정/확인 | 사용자 교정 또는 확인 시 | 행동 지침 |
| **project** | 진행 중 작업, 마감, 사건 | 코드/git에서 도출 불가능한 맥락 학습 시 | 제안 맥락화 |
| **reference** | 외부 시스템 위치 | 외부 리소스 학습 시 | 외부 참조 시 |

**저장하지 않는 것**:
- 코드 패턴, 규약, 아키텍처 (코드 읽기로 파악 가능)
- Git 히스토리 (git log/blame이 권위적)
- 디버깅 해결책 (커밋에 포함)
- CLAUDE.md에 이미 문서화된 내용

---

## 5. 참조 구현체 심층 분석: Claude Code는 어떻게 해결했는가

### 5.1 전체 시스템 아키텍처

```
┌────────────────────────────────────────────────────────────────┐
│                     Claude Code Architecture                    │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Entry Layer                            │  │
│  │  main.tsx (Commander.js CLI + React/Ink Renderer)         │  │
│  │  ├─ Parallel Prefetch (MDM, Keychain, GrowthBook)         │  │
│  │  ├─ Setup Pipeline (Auth, CWD, Hooks, Worktree)           │  │
│  │  └─ REPL Render (First Paint)                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Query Layer                             │  │
│  │  QueryEngine.ts (세션 컨트롤러)                            │  │
│  │  ├─ submitMessage() → 시스템 프롬프트 조립                  │  │
│  │  ├─ query.ts (Agentic Loop — while(true))                 │  │
│  │  │  ├─ Phase 1: Context Prep (4단계 압축)                  │  │
│  │  │  ├─ Phase 2: LLM Streaming (+Concurrent Tool Exec)     │  │
│  │  │  ├─ Phase 3: Error Recovery (Cascade)                  │  │
│  │  │  ├─ Phase 4: Tool Execution (Batch)                    │  │
│  │  │  ├─ Phase 5: Memory/Skill Collection                   │  │
│  │  │  └─ Phase 6: Continue or Exit                          │  │
│  │  └─ Cost Tracking + Usage Accumulation                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Tool Layer                             │  │
│  │  tools.ts (Registry) + Tool.ts (Type System)              │  │
│  │  ├─ 40+ Built-in Tools (Bash, File, Agent, MCP, ...)     │  │
│  │  ├─ Permission Pipeline (5 Layers)                        │  │
│  │  ├─ Streaming Tool Executor (Concurrent)                  │  │
│  │  ├─ Deferred Tool Loading (ToolSearch)                    │  │
│  │  └─ Tool Result Budget (Size Control)                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Extension Layer                           │  │
│  │  ├─ MCP Servers (Dynamic Tool Discovery)                  │  │
│  │  ├─ Plugin System (Skills + Hooks + MCP)                  │  │
│  │  ├─ Skill System (Reusable Workflows)                     │  │
│  │  └─ Bridge (IDE Integration: VS Code, JetBrains)          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Service Layer                             │  │
│  │  ├─ API Client (Multi-Provider: Direct, Bedrock, Vertex)  │  │
│  │  ├─ OAuth (Token Refresh + 401 Retry)                     │  │
│  │  ├─ Cost Tracker (USD + OTel Metrics)                     │  │
│  │  ├─ Memory System (File-Based + Team Sync)                │  │
│  │  ├─ Compact Service (4-Level Context Compression)         │  │
│  │  ├─ Policy Limits (Organization Enforcement)              │  │
│  │  ├─ Analytics (OTel + GrowthBook + Datadog)               │  │
│  │  └─ LSP (Language Server Protocol)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 Agentic Loop의 완전한 생명주기

Claude Code의 핵심은 `query.ts`의 `while(true)` 루프다. 이것이 모든 Agentic 행동의 심장이다:

```
while (true) {

  ┌─ Phase 1: Context Preparation ─────────────────────┐
  │  1. Microcompact (캐시 친화적 tool_use_id 편집)       │
  │  2. Context-collapse (단계적 drain + commit)          │
  │  3. Auto-compact (토큰 경고 임계값 시 능동 요약)       │
  │  4. Snip (경계 마커 기반 히스토리 제거)                │
  │  5. Task budget check (에이전트별 지출 상한)           │
  └────────────────────────────────────────────────────┘
                         │
  ┌─ Phase 2: LLM Streaming ───────────────────────────┐
  │  for await (const message of deps.callModel(...)) { │
  │    if (isWithheldError(message)) withheld = true     │
  │    if (!withheld) yield message  // UI/SDK로 전달     │
  │                                                     │
  │    if (tool_use 블록 감지) {                          │
  │      needsFollowUp = true                           │
  │      streamingToolExecutor.addTool(block)  // 즉시! │
  │    }                                                │
  │  }                                                  │
  └────────────────────────────────────────────────────┘
                         │
  ┌─ Phase 3: Error Recovery ──────────────────────────┐
  │  if (prompt_too_long_withheld) {                    │
  │    try context_collapse_drain()                     │
  │    else try reactive_compact()                      │
  │    else surface_error()                             │
  │  }                                                  │
  │  if (max_output_tokens_withheld) {                  │
  │    try escalate_cap(8K → 64K)                       │
  │    else inject_recovery_message() (×3)              │
  │    else surface_error()                             │
  │  }                                                  │
  └────────────────────────────────────────────────────┘
                         │
  ┌─ Phase 4: Tool Execution ──────────────────────────┐
  │  if (needsFollowUp) {                              │
  │    consume streaming executor results               │
  │    OR run tools sequentially (fallback)             │
  │    collect tool results → messages                  │
  │  }                                                  │
  └────────────────────────────────────────────────────┘
                         │
  ┌─ Phase 5: Collection ──────────────────────────────┐
  │  drain queued commands                              │
  │  consume memory prefetch                            │
  │  collect skill discovery                            │
  │  refresh tool registry (if MCP changed)             │
  └────────────────────────────────────────────────────┘
                         │
  ┌─ Phase 6: Continue or Exit ────────────────────────┐
  │  if (!needsFollowUp) {                             │
  │    run handleStopHooks()                            │
  │    if (blocking errors) inject as user msg, continue│
  │    else return { reason }                           │
  │  }                                                  │
  │  if (turnCount > maxTurns) return { max_turns }     │
  │  state = new State(updated messages + results)      │
  │  continue  // → Phase 1                             │
  └────────────────────────────────────────────────────┘

}
```

### 5.3 도전과제별 해법 매핑

| Enterprise 도전과제 | Claude Code 해법 | 구현 위치 | 핵심 패턴 |
|-------------------|-----------------|----------|----------|
| Context 폭발 | 4단계 계층적 압축 | `query.ts` Phase 1, `compact/` | Snip→Micro→Auto→Reactive |
| 비용 폭증 | Task Budget + Model Routing | `cost-tracker.ts`, `query.ts` | 에이전트별 USD 예산 + 보조 작업 경량 모델 |
| 보안 위협 | 5계층 Permission Pipeline | `hooks/toolPermission/`, `tools/BashTool/` | Fail-Closed + 도구 가시성 제어 |
| Multi-Agent 조율 | Coordinator-Worker + Forked Agent | `coordinator/`, `tools/AgentTool/` | 격리된 Context + 캐시 공유 |
| 오류 불안정성 | Error Withholding + Cascade Recovery | `query.ts` Phase 3, `withRetry.ts` | 복구 가능 오류 보류 + 계층적 복구 |
| 확장성 한계 | Plugin/Skill/MCP 3계층 | `plugins/`, `skills/`, `services/mcp/` | 프로토콜 표준 + 지연 로딩 |
| 디버깅 불가 | 통합 OTel + Audit Trail | `services/analytics/`, `cost-tracker.ts` | 모든 결정의 추적 가능성 |
| 시작 지연 | Parallel Prefetch + Lazy Loading | `main.tsx`, `entrypoints/` | First Paint 전 최소 작업 |
| 세션 단절 | History Persistence + Resume | `history.ts`, `screens/Resume` | JSONL + 세션 ID 기반 복원 |
| 규정 준수 | PII 분리 + 감사 추적 | `analytics/index.ts` | 이중 파이프라인 (_PROTO_* 필드) |

---

## 6. 실전 트레이드오프와 타협점

### 6.1 자율성 vs. 통제

```
스펙트럼:
  완전 통제 ←─────────────────────────→ 완전 자율

  [모든 도구 확인]  [위험 도구만 확인]  [자동 승인]  [무제한]
       │                 │               │           │
       │          Claude Code의           │           │
       │          기본 위치 ───→          │           │
       │         (default mode)           │           │
       │                                  │           │
  Enterprise 초기          Enterprise 성숙기      개발자 전용
```

**Claude Code의 접근**:
- 6개의 Permission Mode로 스펙트럼 전체를 커버
- 기본값은 `default` (모든 도구 확인) → 신뢰 구축 후 `auto` 또는 `bypassPermissions`로 전환
- Anthropic 데이터: 경험 많은 사용자는 40%+ 세션을 자동 승인하면서도 더 자주 개입(5%→9%)

**Enterprise 권고**: `default`에서 시작하여, 팀별로 `alwaysAllow` 규칙을 점진적으로 추가. 절대 처음부터 `bypassPermissions`를 사용하지 말 것.

### 6.2 비용 vs. 능력

```
비용 최적화 레벨:

Level 0: 최적화 없음
  → 모든 작업에 최대 모델, 전체 컨텍스트
  → 비용: $$$$$

Level 1: Model Routing (Claude Code 기본)
  → 본 루프: 고성능, 보조: 경량
  → 비용: $$$

Level 2: + Prompt Cache 최적화
  → Forked Agent 캐시 공유, 안정 정렬
  → 비용: $$

Level 3: + Context 압축 + Result Budget
  → 4단계 압축, 100KB 결과 디스크 이관
  → 비용: $

Level 4: + Task Budget + Kill Switch
  → 에이전트별 USD 상한, 비진행 루프 종료
  → 비용: $ (예측 가능)
```

**핵심 인사이트**: Prompt Cache 공유만으로도 입력 비용의 ~90%를 절감할 수 있다. 이것이 가장 ROI가 높은 최적화다.

### 6.3 속도 vs. 정확도

| 전략 | 속도 영향 | 정확도 영향 | Claude Code 적용 |
|------|----------|-----------|-----------------|
| Streaming Tool Executor | 30-50% 빠름 | 동일 | 기본 활성화 |
| Parallel Prefetch | 체감 시작 60% 빠름 | 동일 | 기본 활성화 |
| Deferred Tool Loading | 첫 도구 사용 느림 | 선택 품질 향상 | 기본 활성화 |
| Context Compression | 후속 호출 빠름 | 세부사항 손실 가능 | 임계값 기반 자동 |
| Model Routing (Haiku) | 3-5x 빠름 | 보조 작업 충분 | 요약/분류에 적용 |

### 6.4 유연성 vs. 안전성

```
유연성                                            안전성
  │                                                 │
  ├─ 임의 명령어 실행 (Bash)    ← → AST 파싱 + 샌드박싱
  ├─ 동적 도구 발견 (MCP)      ← → 도구 가시성 제어
  ├─ 코드 생성·실행            ← → 권한 Pipeline
  ├─ 장기 메모리              ← → 메모리 유형 분류 체계
  ├─ Multi-Agent 통신         ← → 제한된 도구 세트 + Mailbox
  └─ 플러그인 확장             ← → Feature Gate + 사용자 승인
```

**Claude Code의 균형점**:
- 유연성은 최대로 유지하되, 각 유연성에 대응하는 안전 메커니즘을 1:1로 배치
- Bash: 가장 유연한 도구 = 가장 복잡한 보안 (6중 계층)
- MCP: 동적 도구 발견 = 도구 가시성 제어 + 인증 + 지연 로딩

### 6.5 모놀리식 vs. 모듈러

**Claude Code의 선택: 모듈러 모놀리스**

```
모놀리식                    Claude Code              마이크로서비스
  │                    (모듈러 모놀리스)                    │
  │                         │                             │
  │  단일 프로세스            │  단일 프로세스               │  분산 프로세스
  │  단일 컨텍스트            │  격리된 모듈                 │  네트워크 통신
  │  강한 결합              │  느슨한 결합                 │  매우 느슨한 결합
  │                         │  공유 타입 시스템             │
  │                         │  프로세스 내 캐시 공유        │
```

- 단일 프로세스 내에서 모듈화 (tools/, services/, hooks/, plugins/)
- 프로세스 내 캐시 공유로 성능 극대화 (Forked Agent의 CacheSafeParams)
- 타입 시스템으로 모듈 간 계약 보장 (Tool<Input, Output, Progress>)
- 외부 확장은 프로토콜 기반 (MCP, Bridge)

**Enterprise 권고**: Multi-agent 아키텍처를 서두르지 말라. 단일 잘 설계된 에이전트로 시작하고, 도구 선택 품질, 레이턴시, 추론 품질에서 한계가 입증될 때만 Multi-agent를 도입하라.

---

## 7. 구축 로드맵과 실행 권고사항

### 7.1 단계별 구축 로드맵

```
Phase 1: Foundation (4-6주)
━━━━━━━━━━━━━━━━━━━━━━━━━
  ☐ 핵심 Agentic Loop 구현
    ├─ while(true) + tool_use 감지 + 결과 주입
    ├─ 기본 에러 처리 (재시도 + 타임아웃)
    └─ 스트리밍 응답 출력

  ☐ 기본 Tool System 구현
    ├─ Tool 인터페이스 정의 (Input Schema, call(), checkPermissions())
    ├─ 3-5개 핵심 도구 (코드 검색, 파일 읽기/쓰기, 명령 실행)
    └─ Fail-Closed 기본값 설정

  ☐ 기본 Permission System
    ├─ 모든 도구 호출에 대한 사용자 확인
    ├─ Allow/Deny 규칙 엔진
    └─ 감사 로그

Phase 2: Resilience (4-6주)
━━━━━━━━━━━━━━━━━━━━━━━━━━
  ☐ Context Window 관리
    ├─ 토큰 카운팅 + 예산 추적
    ├─ 기본 컨텍스트 압축 (요약 기반)
    └─ 결과 크기 제한 + 디스크 이관

  ☐ Error Recovery
    ├─ Error Withholding 패턴
    ├─ Cascade Recovery (2-3단계)
    └─ API 재시도 전략 (Exponential Backoff)

  ☐ Cost Tracking
    ├─ 세션별 비용 누적
    ├─ 모델별 토큰 추적
    └─ 예산 초과 경고/중단

Phase 3: Scale (6-8주)
━━━━━━━━━━━━━━━━━━━━━━
  ☐ Multi-Agent Support
    ├─ Agent Tool (sub-agent 생성)
    ├─ Forked Agent (캐시 공유)
    ├─ 에이전트 간 메시지 라우팅
    └─ Task Budget (에이전트별 예산)

  ☐ Extension System
    ├─ MCP 클라이언트 통합
    ├─ Plugin 로더
    └─ Skill System

  ☐ Advanced Context Management
    ├─ 4단계 계층적 압축
    ├─ Streaming Tool Executor
    └─ Deferred Tool Loading

Phase 4: Enterprise (지속적)
━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ☐ 관찰가능성 고도화
    ├─ OpenTelemetry 통합
    ├─ Permission Audit Trail
    └─ Feature Flag System

  ☐ 보안 강화
    ├─ Bash AST 파싱 + 샌드박싱
    ├─ ML 기반 자동 분류기
    └─ PII 분리 파이프라인

  ☐ 팀/조직 기능
    ├─ Team Memory Sync
    ├─ Remote Managed Settings
    └─ Policy Limits Enforcement
```

### 7.2 핵심 설계 원칙 (Claude Code에서 추출)

1. **"단순한 것부터 시작하라"**
   > Anthropic: "Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short."
   - 단일 에이전트 + 5개 도구로 시작
   - 평가를 통해 한계가 입증될 때만 복잡성 추가

2. **"Fail-Closed를 기본으로"**
   - 새 도구: 병렬 불안전, 읽기 전용 아님, 파괴적이 아닐 수 있음
   - 새 기능: Feature Flag 뒤에 숨김
   - 새 에이전트: 최소 도구 세트로 시작

3. **"모든 결정을 추적하라"**
   - 권한 결정, 비용, 토큰, 오류 복구 시도
   - EU AI Act 규정 준수의 기반

4. **"컨텍스트는 가장 비싼 자원이다"**
   - 모든 토큰이 모델의 "주의력 예산"을 소비
   - 최소한의 고신호 토큰으로 원하는 결과의 가능성을 최대화

5. **"캐시는 돈이다"**
   - Prompt Cache 적중률이 비용을 결정
   - 도구 목록 안정성, Forked Agent 캐시 공유, 바이트 정렬 유지

6. **"이해를 위임하지 말라"**
   - Coordinator는 Worker 결과를 합성하여 구체적 지시를 내림
   - "알아서 해줘"는 실패의 지름길

7. **"오류 처리는 계층적으로"**
   - 저비용 복구 → 고비용 복구 → 사용자 표출
   - Death Spiral 방지 (오류 처리가 오류를 유발하지 않도록)

### 7.3 Anti-Patterns (피해야 할 것)

| Anti-Pattern | 왜 위험한가 | 대안 |
|-------------|-----------|------|
| 전체 컨텍스트 공유 | Multi-agent에서 비용 폭발 | 압축된 요약만 전달 |
| 무제한 도구 접근 | 보안 위험 + 모델 혼란 | Deferred Loading + 도구 제한 |
| 단일 모델 사용 | 비용 비효율 | Model Routing |
| 즉시 오류 표출 | UX 저하, 불필요한 실패 | Error Withholding |
| 동기적 도구 실행 | 레이턴시 낭비 | Streaming Tool Executor |
| 하드코딩 도구 목록 | 확장 불가 | MCP + Plugin 아키텍처 |
| 메모리 무분별 저장 | 노이즈 축적, Context Distraction | 닫힌 유형 분류 + 중복 방지 |
| 즉시 Multi-Agent | 복잡성 폭발, 오류 17배 증폭 | 단일 에이전트 한계 입증 후 도입 |

### 7.4 기술 스택 권고

| 범주 | Claude Code 선택 | Enterprise 대안 | 근거 |
|------|-----------------|----------------|------|
| Runtime | Bun | Node.js, Deno | Bun의 번들링 + feature flag 통합이 이상적 |
| Language | TypeScript (strict) | TypeScript, Python | 타입 안전성이 도구 인터페이스에 필수 |
| Schema | Zod v4 | Zod, io-ts, Typebox | 런타임 검증 + TypeScript 추론 |
| UI | React + Ink | Blessed, Enquirer | 컴포넌트 기반 터미널 UI |
| Protocol | MCP SDK | 자체 프로토콜 (비권장) | 표준 채택이 생태계 접근의 핵심 |
| Telemetry | OpenTelemetry | Datadog, New Relic | 벤더 중립 표준 |
| Feature Flags | GrowthBook | LaunchDarkly, Split.io | 점진적 릴리스 + A/B 테스트 |
| Auth | OAuth 2.0 + JWT | OIDC, SAML | 에이전트별 독립 자격증명 지원 |

---

## 8. 부록: 참고자료 및 출처

### 학술 논문

- Agentic AI: A Comprehensive Survey (arxiv 2510.25445)
- Adaptation of Agentic AI (arxiv 2512.16301)
- 2025 AI Agent Index (arxiv 2602.17753)
- Agentic RAG Survey (arxiv 2501.09136)
- Characterizing Faults in Agentic AI (arxiv 2603.06847)
- Multi-Agent LLM Orchestration for Incident Response (arxiv 2511.15755)
- Agentic AI Frameworks: Architectures, Protocols, and Design Challenges (arxiv 2508.10146)

### Anthropic 기술 문서

- Building Effective Agents (anthropic.com/research)
- Multi-Agent Research System (anthropic.com/engineering)
- Measuring Agent Autonomy (anthropic.com/research)
- Effective Context Engineering for AI Agents (anthropic.com/engineering)
- Effective Harnesses for Long-Running Agents (anthropic.com/engineering)

### 산업 보고서

- OWASP Top 10 for Agentic Applications 2026 (genai.owasp.org)
- LangChain: State of Agent Engineering (langchain.com)
- OpenTelemetry: AI Agent Observability (opentelemetry.io)
- IDC: Agentic AI Cost Overruns
- Gartner: Multi-Agent Pilot Cancellation Predictions

### 규제 프레임워크

- EU AI Act (2026년 8월 전면 적용)
- NIST AI RMF (Agent Security 별도 카테고리)
- NIST Identity & Authorization RFI (2026년 4월 마감)

### 프로토콜 표준

- MCP (Model Context Protocol) — Linux Foundation AAIF
- A2A (Agent-to-Agent Protocol) — Linux Foundation AAIF
- OpenTelemetry GenAI SIG — Semantic Conventions

---

> **본 백서의 모든 아키텍처 분석은 Claude Code CLI 소스코드의 직접 분석에 기반합니다.**
> **산업 데이터와 연구 결과는 2025-2026년 공개 자료를 기반으로 합니다.**

---

*Enterprise Agentic LLM 시스템 구축 백서 v1.0*
*AI 엔지니어를 위한 기술 가이드*
