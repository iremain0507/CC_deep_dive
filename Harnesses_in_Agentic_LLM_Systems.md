# Harnesses in Agentic LLM Systems: A Comprehensive Technical Deep Dive

**Research Date: April 2026**
**Audience: AI Engineers building production agent systems**

---

## Table of Contents

1. [Defining the Agent Harness](#1-defining-the-agent-harness)
2. [Harness vs. Framework vs. Orchestrator](#2-harness-vs-framework-vs-orchestrator)
3. [The Anatomy of a Harness: Core Components](#3-the-anatomy-of-a-harness-core-components)
4. [The Agent Loop: Inner and Outer Execution](#4-the-agent-loop-inner-and-outer-execution)
5. [Context Engineering within the Harness](#5-context-engineering-within-the-harness)
6. [Tool Execution and Management](#6-tool-execution-and-management)
7. [State Management, Checkpointing, and Error Recovery](#7-state-management-checkpointing-and-error-recovery)
8. [Permission Models and Human-in-the-Loop](#8-permission-models-and-human-in-the-loop)
9. [Multi-Session and Long-Horizon Patterns](#9-multi-session-and-long-horizon-patterns)
10. [Multi-Agent Harness Architectures](#10-multi-agent-harness-architectures)
11. [Production Harness Designs from Major Providers (2025-2026)](#11-production-harness-designs-from-major-providers-2025-2026)
12. [The IMPACT Framework](#12-the-impact-framework)
13. [Production Best Practices and Anti-Patterns](#13-production-best-practices-and-anti-patterns)
14. [The Future: Harness Engineering as a Discipline](#14-the-future-harness-engineering-as-a-discipline)
15. [Sources](#15-sources)

---

## 1. Defining the Agent Harness

### The Canonical Definition

An agent harness is **the complete software infrastructure surrounding an AI model that manages everything except the model's actual reasoning**. It encompasses all code, configuration, and execution logic that is not the model itself.

The defining equation is:

```
Agent = Model + Harness
```

The model provides intelligence; the harness provides utility. As Anthony Alcaraz defines it: "the complete architectural system surrounding an LLM that manages the lifecycle of context: from intent capture through specification, compilation, execution, verification, and persistence."

### The Operating System Analogy

The most illuminating framing, articulated across multiple 2026 sources, maps agent architecture to operating system concepts:

| OS Concept | Agent Equivalent |
|---|---|
| CPU | The LLM (raw processing power) |
| RAM | The context window |
| Operating System / Kernel | The harness (curating context, handling tool dispatch, providing drivers) |
| System Calls | Tools |
| Persistent Storage | The filesystem (progress files, git history, memory) |
| Applications | The agent itself, running atop the harness |

This analogy makes a critical architectural point: **the harness is not an accessory to the model; it is the enabling infrastructure that makes the model usable for real work**.

### Why Harnesses Emerged

Harnesses developed to address five fundamental limitations of raw LLMs:

1. **Statelessness**: LLMs have no memory between calls. The harness provides persistence through files, databases, and structured state.
2. **Bounded Context**: Fixed context windows mean models cannot hold entire project states. The harness manages what enters the window and when.
3. **No Action Capability**: Models generate text only. The harness bridges this gap by executing tool calls externally and feeding results back.
4. **No Self-Verification**: Models cannot run their own tests or check their work against reality. The harness provides verification loops.
5. **No Cross-Session Continuity**: Each new context window starts fresh. The harness provides handoff artifacts, progress files, and environmental scaffolding.

In Anthropic's experiments, Claude Opus 4.5 without a harness failed to build production web applications across multiple context windows -- it either attempted one-shot implementations that exhausted tokens or declared premature success on partial progress.

---

## 2. Harness vs. Framework vs. Orchestrator

These three concepts are frequently conflated but serve distinct architectural roles:

### Framework

**Definition**: A library or collection of abstractions for building agent components. Examples include LangChain, LlamaIndex, and Microsoft Semantic Kernel.

**Role**: Provides building blocks -- prompt templates, retrieval abstractions, tool interfaces, chain/graph primitives. Frameworks are used *during development* and supply the parts you assemble.

**Key characteristic**: Frameworks supply components but do not dictate runtime behavior.

### Harness

**Definition**: A full runtime system with opinionated defaults and integrations that executes an agent with tools, memory, and state management.

**Role**: Assembles framework components (or raw primitives) into a functioning operational system. A harness is what you *run in production*. It manages context loading, tool availability, permitted actions, failure handling, and session lifecycle.

**Key characteristic**: The harness is the assembled, functioning whole. You can build a harness using a framework, or you can build one from scratch.

**Relationship to framework**: LangChain's DeepAgents exemplifies a harness built atop the LangChain framework, adding default prompts, tool handling, and file system persistence that you would otherwise assemble manually. Claude Code and Codex CLI are harnesses that use no external frameworks at all.

### Orchestrator

**Definition**: The control flow logic that determines when and how to invoke the model, implementing reasoning loops and decision routing.

**Role**: Supplies the operational "brain" -- it decides the sequence of actions, manages branching logic, and handles delegation. The orchestrator lives *inside* the harness.

**Key characteristic**: Orchestration is a *function* of the harness, not a separate entity.

### The Critical Distinction

Anthropic's "Building Effective Agents" post (December 2024) draws a foundational line between **workflows** and **agents**:

- **Workflows**: "Systems where LLMs and tools are orchestrated through predefined code paths." The harness controls the flow.
- **Agents**: "Systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks." The model controls the flow; the harness provides the infrastructure.

Frameworks are ideal for development and early-stage projects. Harnesses are what you run in production. Orchestrators are the control flow component within either. As Anthropic advises: "Start with direct LLM APIs -- many patterns can be implemented in a few lines of code." Frameworks "create extra layers of abstraction that can obscure the underlying prompts and responses."

---

## 3. The Anatomy of a Harness: Core Components

Drawing from Anthropic, OpenAI, LangChain, and the broader 2025-2026 ecosystem, a production harness consists of these essential components:

### 3.1 The Planning Loop (Agent Loop)

The execution core. Every production agent implements a variant of the ReAct (Reason + Act) pattern:

```
while task_not_complete:
    state = read_files() + read_test_output() + read_errors()
    context = harness.select_context(state, task)
    plan = model.reason(context, task)
    result = harness.dispatch_tool(plan.next_action)
    outcome = harness.evaluate(result)
    
    if outcome.needs_retry:
        context = harness.add_error_trace(outcome.error)
        continue
    if outcome.needs_human:
        harness.escalate(outcome)
        break
    
    harness.checkpoint(result)
```

This loop fails without reliable feedback from tests, linters, and type checkers. Agents with test suites can work effectively through projects; agents without hallucinate progress.

### 3.2 Tool Integration Layer

Defines what agents can execute: file operations, code execution in sandboxes, database queries, API calls, web access, and browser automation.

The harness:
- Intercepts tool call requests from the model
- Validates parameters before execution
- Executes operations in the appropriate sandbox
- Returns sanitized results to the model
- Manages tool availability (which tools are exposed at any given state)

**Four production tool categories** (from the Decoding AI analysis of OpenCode, Claude Code, and Codex):

1. **General-Purpose Execution**: Bash/shell for arbitrary commands, enabling agents to design tools dynamically
2. **Specialized Filesystem Operations**: Read/edit with enforced absolute paths and line limits (prevents error-prone bash file operations)
3. **State Management Tools**: Session-scoped task tracking (e.g., `ToDoAdd`, `ToDoRead`)
4. **Orchestration Tools**: Launch isolated subagents with separate context windows

### 3.3 Memory Architecture (Three Layers)

Memory operates across three planes with distinct characteristics:

**Layer 1 -- Filesystem (Long-term, Persistent)**:
- Survives across sessions and context windows
- Holds progress files, git history, session transcripts, feature lists
- Every production harness uses the filesystem as its primary state mechanism
- JSON format preferred over Markdown because models are less likely to inappropriately modify structured data

**Layer 2 -- RAM / Session State (Short-term, Working)**:
- Fast but volatile within active sessions
- Contains conversation history, tool results, in-progress state
- Durable logs tracking tool results, completed subtasks, and progress notes

**Layer 3 -- Context Window (What the Model Sees)**:
- The strictest constraint -- everything the model knows must fit within token limits
- Curated by context engineering from Layers 1 and 2
- Subject to context rot as length increases

The harness orchestrates movement between layers:
- **Read path**: Relevant state loads from disk into RAM, then context engineering assembles the window via compaction, progressive disclosure, and just-in-time retrieval
- **Write path**: Important state persists back to disk before being discarded from context

**Critical invariant**: Memory must flush to disk before being discarded from context. Rehydration occurs as tool-shaped actions where agents search and retrieve specific data rather than dumping everything into windows.

### 3.4 Sandbox / Execution Environment

A security-capability tradeoff with three common approaches:

| Approach | Example | Characteristics |
|---|---|---|
| Hard Sandboxing | OpenAI Codex | Isolated cloud container per task, preloaded with repo. Maximum safety; agents cannot access host. Container destroyed after task. |
| Soft Sandboxing | Claude Code (local mode) | Workspace is the default directory with full filesystem access. More risk, more capability. |
| Container Sandboxing | Cursor, Devin | Docker containers or isolated cloud environments providing middle ground. |

Cloud sandboxes enable horizontal scaling (parallel agent sessions) and access to powerful resources (GPUs for training, build servers).

### 3.5 Verification and Guardrails

Production harnesses verify outputs before treating work as complete. Three mechanisms:

1. **Rules-Based Feedback** (most effective): Code linting, type checking, test execution. Anthropic notes that "it is usually better to generate TypeScript and lint it than generate pure JavaScript because it provides multiple additional layers of feedback."

2. **Visual Feedback**: Screenshot-based verification for UI work via Playwright or Puppeteer MCP. Agents verify layout, color, formatting, and responsiveness.

3. **LLM-as-Judge**: Secondary models evaluate output against criteria. Anthropic notes this "is generally not a very robust method, and can have heavy latency tradeoffs, but for applications where any boost in performance is worth the cost, it can be helpful."

**Key finding**: Providing verification mechanisms improves agent quality by 2-3x (Decoding AI / OpenCode analysis). Feedback loops are the single most important principle around tooling.

### 3.6 Context Engineering Engine

The subsystem determining what tokens enter the context window at each model invocation. This is covered extensively in Section 5 below.

### 3.7 Configuration and System Prompts

Project-level context files (CLAUDE.md, .cursorrules, AGENTS.md) loaded at session start. These encode:
- Project conventions and constraints
- Known failure modes and mitigations
- Tool usage guidance
- Architectural decisions
- Behavioral conditioning rules

---

## 4. The Agent Loop: Inner and Outer Execution

### The Core Loop (Inner Loop)

The inner loop is the immediate execution cycle that runs at machine speed:

1. **Assemble context** from available inputs (system prompt + tools + history + retrieved data)
2. **Invoke the LLM** to reason and select an action
3. **Execute the action** via tool call
4. **Observe the outcome** and feed it back into the next iteration
5. **Repeat** until task complete or stopping condition reached

This is the ReAct pattern formalized by Yao et al. (2022). Each iteration processes one thought-action-observation cycle.

### The Outer Loop (Strategic Loop)

The outer loop operates at a higher level of abstraction:

- **Project lifecycle management**: Initialization, feature planning, sprint contracts
- **Multi-session coordination**: Handoff between context windows, progress tracking
- **Strategic replanning**: When inner loop progress stalls, the outer loop can reset strategy entirely
- **Human oversight**: Stakeholder relationships, approval gates, checkpoint reviews

Microsoft's Magentic-One explicitly implements this dual-loop system: the outer loop handles strategic planning while the inner loop executes step-by-step, with the ability to reset the entire strategy when progress stalls.

### The Relationship Between Loops

The fast inner loops (tool execution, status checking, error recovery, communication) run at machine speed. The slow outer loop (project lifecycle, strategic judgment, stakeholder relationships) remains under human control or high-level agent orchestration, with the inner loops feeding intelligence upward.

In Anthropic's long-running agent pattern:
- **Outer loop**: The harness spawns successive coding agent sessions, each working on one feature
- **Inner loop**: Within each session, the agent runs the ReAct cycle over file reads, edits, test runs, and commits

### Ralph Loop Pattern

For tasks requiring continuation across context windows, the "Ralph Loop" pattern intercepts agent exit attempts and reinjects the original prompt into a clean context window, forcing continuation toward completion goals while preserving filesystem state. The filesystem holds continuity; the context window is ephemeral.

---

## 5. Context Engineering within the Harness

### Foundational Concepts

Anthropic's "Effective Context Engineering for AI Agents" (September 2025) establishes context engineering as the evolution beyond prompt engineering. While prompt engineering focuses on crafting instructions, context engineering manages the **entire context state** across multiple inference turns -- system instructions, tools, MCP definitions, external data, and message history.

**The Attention Budget Problem**: Context is "a finite resource with diminishing marginal returns," analogous to human working memory. As context length increases:
- Every token attends to every other token (n-squared pairwise relationships)
- Models trained on shorter sequences have fewer parameters for long-range dependencies
- Position encoding interpolation enables longer sequences but with degradation
- Critical information buried in the middle of long prompts is less likely to be recalled ("Lost in the Middle" effect, Liu et al., Stanford 2023)

### System Prompt Design

Anthropic identifies a "Goldilocks zone" with two failure modes:

1. **Overly Brittle**: Hardcoding complex if-else logic creates fragility
2. **Overly Vague**: High-level guidance without concrete signals fails

Recommendations:
- Use simple, direct language at the "right altitude"
- Organize into distinct sections using XML tags or Markdown headers
- Start minimal on best-available models, then add instructions based on failure modes
- Strive for "the minimal set of information that fully outlines expected behavior" -- minimal does not mean short

### Five Context Management Strategies

#### Strategy 1: CLAUDE.md / Project Context Files
Project-level files loaded into every session providing the minimum viable context for the entire project. Identified consistently as the single most impactful move for agent quality.

#### Strategy 2: Just-in-Time Retrieval (Progressive Disclosure)
Agents maintain lightweight identifiers (file paths, stored queries, URLs) and dynamically load data at runtime using tools. Claude Code achieves 95% context reduction through MCP tool search combined with lazy loading. Agents incrementally discover relevant context through exploration rather than pre-loading everything.

**Hybrid approach (Claude Code)**: CLAUDE.md files are naively dropped into context up front, while primitives like glob and grep allow navigation and just-in-time file retrieval, "effectively bypassing the issues of stale indexing and complex syntax trees."

#### Strategy 3: Compaction
Taking a conversation nearing the context window limit, summarizing its contents, and reinitiating with the compressed summary.

**Claude Code implementation**:
1. Pass message history to model for compression
2. Preserve: architectural decisions, unresolved bugs, implementation details
3. Discard: redundant tool outputs, completed messages
4. Continue with compressed context plus five most recently accessed files

**Tuning approach**: "Start by maximizing recall to ensure compaction captures every relevant piece of information, then iterate to improve precision by eliminating superfluous content."

**Quick win -- Tool result clearing**: Once a tool is called deep in history, the raw result is rarely needed again. This is "one of the safest, lightest-touch forms of compaction."

#### Strategy 4: Structured Note-Taking (Agentic Memory)
Agents write notes persisted outside the context window, retrieved later. Anthropic demonstrated this with Claude playing Pokemon, where the agent maintained "precise tallies across thousands of game steps -- tracking objectives like 'for the last 1,234 steps I've been training my Pokemon in Route 1, Pikachu has gained 8 levels toward the target of 10.'"

Anthropic released a memory tool in public beta on the Claude Developer Platform enabling file-based information storage outside the context window.

#### Strategy 5: Sub-Agent Context Isolation
Specialized sub-agents handle focused tasks with clean context windows. Each sub-agent may use tens of thousands of tokens but returns "only a condensed, distilled summary of its work (often 1,000-2,000 tokens)."

### Manus Production Context Engineering (Five Dimensions)

Manus (acquired by Meta for ~$2B in December 2025) developed five production-proven context engineering dimensions:

1. **Context Offloading**: Treat the filesystem as unlimited context. Web page content can be dropped from context if the URL persists; document contents can be omitted if paths remain accessible in the sandbox. This enables "recoverable compression" -- shrinking context without permanently losing information.

2. **Context Reduction**: Compaction and summarization. Critical insight: "you can't reliably predict which observation might become critical ten steps later," making irreversible compression inherently risky.

3. **Context Retrieval**: File-based search tools for on-demand data loading.

4. **Context Isolation**: Multi-agent architectures where each agent gets clean context for its specific task.

5. **Context Caching (KV-Cache Optimization)**: The single most important production metric. Manus reports:
   - Average 100:1 token ratio between prefilling and decoding
   - Cached tokens cost $0.30/MTok vs. $3/MTok uncached (10x differential with Claude Sonnet)
   - Three practices: stable prompt prefixes (even a single-token difference invalidates cache), append-only context (never modify previous observations), and explicit cache breakpoints

### Manus Production Insights

**Attention Manipulation via Recitation**: When handling complex tasks (average ~50 tool calls), Manus agents create a `todo.md` file and update it step-by-step, checking off completed items. This "pushes the global plan into the model's recent attention span, avoiding 'lost-in-the-middle' issues."

**Preserving Failure Context**: Counterintuitively, leave wrong turns in context. "When the model sees a failed action and the resulting stack trace, it implicitly updates its internal beliefs," shifting its prior away from similar decisions.

**Avoiding Few-Shot Brittleness**: When processing many similar items (e.g., 20 resumes), models fall into repetitive patterns. Solution: introduce "small amounts of structured variation in actions and observations -- different serialization templates, alternate phrasing, minor noise."

---

## 6. Tool Execution and Management

### Tool Design Principles (Anthropic)

From "Building Effective Agents" and the Claude Agent SDK documentation:

- Tools should be **self-contained, robust to error, and extremely clear about intended use**
- Input parameters must be **descriptive, unambiguous, and play to model strengths**
- Provide sufficient tokens for model reasoning
- Keep formats close to naturally-occurring internet text
- Eliminate formatting overhead (line counting, string escaping)
- Include example usage, edge cases, input format requirements, and boundaries
- Critical test: **"If a human engineer can't definitively say which tool should be used in a given situation, an AI agent can't be expected to do better"**
- Common failure mode: "Bloated tool sets that cover too much functionality or lead to ambiguous decision points"

### The SWE-bench Lesson

Anthropic's SWE-bench implementation revealed that "the model would make mistakes with tools using relative filepaths after the agent had moved out of the root directory." Solution: **require absolute filepaths only**. This exemplifies how harness engineering converts observed failures into permanent tool-level fixes.

### Logit Masking vs. Dynamic Tool Loading (Manus)

As tool counts grow, a critical architectural decision emerges:

**Why not dynamically load/unload tools**:
- Tool definitions appear near the context front after serialization; any change invalidates the downstream KV-cache
- Orphaned tool references in past observations confuse models, causing "schema violations or hallucinated actions"

**The Manus solution -- Logit Masking**:
Rather than removing tools, mask token logits during decoding to prevent or enforce selection of certain actions based on current state. This preserves KV-cache while constraining behavior.

**Three function-calling modes** (using Hermes format):
1. **Auto**: Model may call a function or not -- prefill only `<|im_start|>assistant`
2. **Required**: Must call something -- prefill to `<|im_start|>assistant<tool_call>`
3. **Specified**: Must call from a subset -- prefill to `<|im_start|>assistant<tool_call>{"name": "browser_`

**Naming convention strategy**: Design consistent prefixes (`browser_*`, `shell_*`) to enforce tool group selection without needing stateful logit processors.

### Tool Execution Patterns Across Providers

| System | Approach | Context Reduction |
|---|---|---|
| Claude Code | MCP + lazy loading | 95% reduction |
| Cursor | Renamed tools per model + explicit dispatch | Model-specific optimization |
| Manus | Logit masking via state machines | Preserves KV-cache |
| OpenCode | 75+ provider-agnostic tool layer | Standardized interface |

### Feedback Loops as the Critical Pattern

The most important principle around tooling: **give the model a way to verify its work**. This improves quality by 2-3x. Production implementations include:
- OpenCode's LSP (Language Server Protocol) integration feeding real-time code diagnostics back to the planning loop
- Anthropic's Puppeteer/Playwright MCP for visual verification
- Test suite execution after each feature implementation
- Linter error messages specifically crafted to teach fixes (OpenAI's approach)

---

## 7. State Management, Checkpointing, and Error Recovery

### State Management

Production harnesses implement state management through a combination of:

1. **Filesystem artifacts**: Progress files (JSON-structured feature lists, `claude-progress.txt`), configuration files, session transcripts
2. **Version control**: Git commits as checkpoints with descriptive messages enabling rollback
3. **Structured task tracking**: Feature lists with pass/fail status for each item
4. **Environment scripts**: `init.sh` files that reproduce the working environment

**JSON over Markdown**: Anthropic specifically chose JSON for feature lists and progress tracking because "models are less likely to inappropriately modify structured data" that multiple sessions depend on.

### Checkpointing

Checkpointing is not optional for production agents -- it is fundamental.

**Git-as-checkpoint pattern** (Anthropic):
- Commit changes with descriptive messages after each completed feature
- Each commit creates a save point for rollback
- Failed attempts can be reverted to last known good state
- Clean commits enable human team handoffs without incomplete state

**LangGraph's approach**: When compiling a StateGraph, LangGraph produces an executable graph that manages state transitions, validates updates, and provides built-in checkpoint-based error recovery. If the harness crashes mid-task, the agent resumes from the last checkpoint rather than restarting.

### Error Recovery Hierarchy

Production harnesses implement a graduated recovery strategy:

**Level 1 -- Retry with Context**:
Agent sees the error, adjusts approach, tries again. Most errors resolve here (wrong file path, syntax error). The error trace remains in context, shifting the model's prior away from the failed approach.

**Level 2 -- Rollback to Checkpoint**:
Agent reverts to the last known good state via `git checkout` or equivalent, then re-attempts with a different strategy. Triggered when Level 1 fails multiple times.

**Level 3 -- Decompose the Task**:
Break the failing task into smaller subtasks, delegate to sub-agents with focused context. Addresses complexity-related failures.

**Level 4 -- Escalate to Human**:
Agent writes a clear summary of attempted approaches, failures, and specific needs. Triggered when the task requires judgment or information beyond the agent's capabilities.

### Verification Loops

Verification after each tool call is the **single highest-impact pattern in agent harness engineering**. A verification step checking response schema, validating outputs, running tests before committing, and catching silent failures prevents the compounding error problem:

- Individual agent steps may have 95% accuracy
- But 20 sequential unverified steps: 0.95^20 = 36% overall accuracy
- Verification loops push the compounding failure rate back toward acceptable levels

---

## 8. Permission Models and Human-in-the-Loop

### The Authority Tradeoff

Permission controls represent a fundamental tension: tight controls reduce risk but kill productivity; loose controls risk unintended modifications. The IMPACT framework identifies Authority as "the overlooked element" -- excessive approval prompts ("stutter-step agents") destroy the user experience.

### Permission Model Spectrum

| System | Model | Characteristics |
|---|---|---|
| Cline | Explicit approval for every file change | Maximum safety, slowest |
| Claude Code | Configurable levels from full approval to skip permissions, with hooks for custom automation | Flexible, user-controlled |
| Cursor | Sandbox-aware with explicit filesystem paths and network controls | Middle ground |
| Codex | Pause-and-approve model; agent waits for "allow"/"deny" | Clear separation of propose/commit |
| Devin | Full sandboxed cloud environment; nothing escapes without explicit PR submission | Maximum isolation |

### The Propose/Commit Pattern

The most reliable human-in-the-loop pattern separates:

1. **Propose**: Store a structured action payload in a durable store (the agent suggests what to do)
2. **Commit**: Execute the tool call with strict checks -- idempotency keys, precondition validation, and post-action verification

This reduces accidental side effects and prevents agents from "doing first and asking later."

### Implementation Approaches

**LangGraph**: When a model proposes an action requiring review, middleware can pause execution by checking each tool call against a configurable policy, issuing an interrupt that halts execution while graph state is saved using the persistence layer.

**Codex**: "The agent's turn pauses until the user responds with 'allow' or 'deny,' allowing the agent to balance autonomy with human oversight without hardcoding that policy into the agent loop itself."

**Cloudflare Agents**: Durable state persistence enables the agent to hibernate while awaiting human input, then resume exactly where it left off without re-running prior steps.

### Risk-Based Tiering

Production systems categorize operations by risk level:

- **Low risk** (read operations, search, analysis): Auto-approve
- **Medium risk** (file writes, code changes in development): Configurable -- approve once per category or auto-approve
- **High risk** (production database writes, external communications, financial transactions, destructive operations): Always require explicit human approval

---

## 9. Multi-Session and Long-Horizon Patterns

### Anthropic's Initializer-Executor Pattern

The foundational pattern for long-running coding tasks, documented in "Effective Harnesses for Long-Running Agents" (November 2025):

**Session 1 -- Initializer Agent** (runs once):
1. Creates `init.sh` script (development server startup, basic end-to-end tests)
2. Creates comprehensive feature list in JSON format (the Claude.ai clone example contained 200+ features)
3. Creates `claude-progress.txt` for cumulative human-readable status
4. Makes initial git commit establishing baseline

**Feature list structure**:
```json
{
    "category": "functional",
    "description": "New chat button creates a fresh conversation",
    "steps": [
      "Navigate to main interface",
      "Click the 'New Chat' button",
      "Verify a new conversation is created",
      "Check that chat area shows welcome state",
      "Verify conversation appears in sidebar"
    ],
    "passes": false
}
```

**Session N -- Coding Agent** (repeats):
1. Run `pwd` to confirm working directory
2. Read git logs and progress files for recent context
3. Read feature list and select highest-priority incomplete feature
4. Start development server via `init.sh`
5. Run basic end-to-end verification test
6. Begin feature work only if baseline functionality confirmed
7. Implement exactly ONE feature per session
8. Test via browser automation (Puppeteer/Playwright MCP)
9. Commit changes with descriptive messages
10. Update progress files

**Critical constraint**: Agents receive strong instructions against editing/removing tests -- "this could lead to missing or buggy functionality." This behavioral conditioning effectively guides model behavior.

**One feature per session**: This constraint ensures production-ready code before context window exhaustion. Clean commits create save points for rollback and enable human team handoffs.

### Anthropic's Three-Agent Architecture

The evolution from the two-agent pattern, documented in "Harness Design for Long-Running Application Development":

1. **Planner Agent**: Transforms simple 1-4 sentence prompts into detailed product specifications, emphasizing scope and high-level direction rather than implementation details

2. **Generator Agent**: Executes work iteratively, implementing features one at a time using the tech stack, with version control

3. **Evaluator Agent**: Conducts quality assurance by testing running applications via Playwright MCP, verifying functionality against negotiated sprint contracts

**Generator-Evaluator Loop (GAN-Inspired)**:
This separates work production from quality assessment, addressing the critical "self-evaluation bias" problem. Anthropic found that "when asked to evaluate work they've produced, agents tend to respond by confidently praising the work -- even when, to a human observer, the quality is obviously mediocre."

**Sprint Contracts**: Before implementation, generator and evaluator negotiate explicit deliverables and success criteria, bridging the gap between high-level specs and testable outcomes.

**Evaluator Calibration**: Evaluators were "calibrated using few-shot examples with detailed score breakdowns" to maintain consistent judgment. Four grading criteria for frontend work: Design Quality, Originality, Craft, and Functionality.

### Performance Metrics

| Configuration | Time | Cost | Result |
|---|---|---|---|
| Solo agent (no harness) | 20 min | $9 | Broken core functionality |
| Full three-agent harness | 6 hours | $200 | Fully operational multi-feature application |
| Simplified harness (DAW) | 3h 50min | $124.70 | Complete application with QA-caught fixes |

### Context Window Continuity

The core problem: "Engineers on shifts with no handoff documentation creates cascading failures and wasted effort."

**Why compaction alone is insufficient**: Compaction loses detail. For multi-window continuity, explicit environmental scaffolding (init scripts, progress files, feature lists, git history) proves necessary. The project environment becomes shared memory across all sessions. Each executor session learns project state from files rather than context.

---

## 10. Multi-Agent Harness Architectures

### Anthropic's Workflow Patterns

From "Building Effective Agents," five distinct patterns at increasing complexity:

**1. Prompt Chaining**: Sequential steps where each LLM call processes prior outputs. Programmatic gates verify intermediate results. Trades latency for accuracy.

**2. Routing**: Classifies inputs and directs to specialized handlers. Example: directing customer queries by type, or routing easy questions to smaller models and complex ones to larger models.

**3. Parallelization**: Multiple LLM calls simultaneously with aggregated outputs.
- *Sectioning*: Independent subtasks run in parallel
- *Voting*: Same task run multiple times for diverse outputs

**4. Orchestrator-Workers**: Central LLM dynamically breaks down tasks and delegates. Unlike parallelization, "subtasks aren't pre-defined, but determined by the orchestrator based on the specific input."

**5. Evaluator-Optimizer**: One LLM generates, another evaluates in iterative loops. "Most effective when we have clear evaluation criteria, and when iterative refinement provides measurable value."

### Context Isolation Benefit

Multi-agent architectures solve a fundamental context problem: a single agent refactoring three modules simultaneously pollutes its context with details from all three. Three parallel agents, each focused on one module, maintain clean context. The orchestrator only needs high-level status.

### Multi-Agent Implementations (2025-2026)

| System | Multi-Agent Approach |
|---|---|
| Claude Code | Agent Teams via MCP; subagent tool for isolated tasks |
| Cursor | Subagent system for discrete parallel subtasks |
| Windsurf | Cascade via git worktrees (5 agents on 5 bugs simultaneously) |
| Grok Build | Direct parallelism (8 simultaneous agents) |
| Devin | Parallel sandboxed cloud sessions |

### DeepMind's Generator-Verifier-Reviser (G-V-R)

Google DeepMind's Aletheia project implemented a three-part agent harness for research agents:

1. **Generator**: Proposes a candidate solution
2. **Verifier**: Checks for flaws or hallucinations (natural language)
3. **Reviser**: Corrects errors until final output approved

Critical finding: "Explicitly separating verification helps the model recognize flaws it initially overlooks during generation." This directly addresses the hallucination avalanche problem in long-horizon contexts.

### When to Use What (Anthropic's Guidance)

- Simple single-LLM calls with retrieval suffice for most applications
- Workflows offer "predictability and consistency for well-defined tasks"
- Agents enable "flexibility and model-driven decision-making" for open-ended problems where step count and path cannot be predetermined
- The recommendation: "Only add complexity when it demonstrably improves outcomes"

---

## 11. Production Harness Designs from Major Providers (2025-2026)

### Anthropic: Claude Code / Claude Agent SDK

**Architecture**: Three-layer design:
1. Orchestration loop managing planning and execution
2. Permission-gated tool system
3. Multi-layer memory architecture

**Key design principle**: "Give your agents a computer, allowing them to work like humans do."

**Agentic loop**: Four-phase feedback cycle: Gather Context, Take Action, Verify Work, Iterate.

**Context management**: Hybrid approach -- CLAUDE.md files loaded upfront, plus glob/grep for just-in-time retrieval. Automatic compaction when context limit approaches.

**Tool philosophy**: Tools are "the primary building blocks of execution," prominent in context window. Bash serves as "a general-purpose tool to allow the agent to do flexible work using a computer."

**MCP integration**: Standardized tool integrations handling authentication automatically through the growing MCP ecosystem.

### OpenAI: Codex / Codex CLI

**Architecture**: The Codex harness provides the core agent loop and execution logic. A single user request might trigger the agent to "read several files, run tests, edit code, run tests again, and fix errors before producing a final commit."

**Harness engineering philosophy**: OpenAI coined the term in the context of their Codex development. Their approach: write linter error messages specifically teaching fixes, so every failure becomes context for the next attempt.

**App Server**: JSON-RPC over stdio, evolving from terminal tool to production server.

**Production metrics**: One team achieved ~1 million lines of code in five months, ~1,500 PRs merged, averaging 3.5 PRs per engineer per day with Codex harness support.

**Sandboxing**: Hard cloud containers per task. Maximum isolation -- agents cannot access host. Results extracted, container destroyed.

### Manus (acquired by Meta, December 2025, ~$2B)

**Architecture**: Five complete harness rebuilds in six months, each improving reliability. Uses foundation models from Anthropic and OpenAI but differentiates entirely on harness quality.

**Key innovations**:
- KV-cache optimization as the primary production metric (100:1 prefill-to-decode ratio)
- Logit masking via state machines (not dynamic tool removal)
- Filesystem as unlimited, recoverable context
- `todo.md` attention manipulation
- Error preservation in context
- Structured variation to prevent few-shot brittleness

**Naming convention pattern**: All browser tools start with `browser_`, shell tools with `shell_`, enabling prefix-based tool group enforcement.

### LangChain: DeepAgents

**Architecture**: A harness built atop the LangChain framework with "default prompts, tool handling, planning utilities, file system access, and more baked in."

**LangGraph**: Stateful graph-based orchestration for multi-step workflows with built-in tool routing, memory persistence, and checkpoint-based error recovery. Fastest execution with most efficient state management among framework-based solutions.

### Inngest: Utah (Reference Implementation)

**Architecture**: Event-driven harness where each LLM or tool call becomes an independently retryable unit. "If a process fails on iteration five, iterations one through four remain persisted."

**Six-function decomposition**:
1. `handleMessage`: Main agent loop
2. `sendReply`: Channel-specific response delivery
3. `acknowledgeMessage`: Typing indicators
4. `failureHandler`: Global error handling
5. `heartbeat`: Scheduled check-ins
6. `subAgent`: Isolated runs via `step.invoke()`

**Concurrency**: Singleton concurrency per conversation -- new messages cancel current runs and restart with fresh context.

**Context management**: Two-tier pruning (keep last 3 assistant turns; soft-trim old results; hard-clear at threshold), plus compaction, budget warnings, and overflow recovery.

---

## 12. The IMPACT Framework

swyx's IMPACT framework (2025-2026) identifies six essential components separating production agents from demos:

### I -- Intent
Goals encoded via multimodal I/O and verified through evals. Agents must understand success criteria before acting. Vague prompts produce vague code; a failing test is an unambiguous specification.

### M -- Memory
Long-running coherence through skill libraries and reusable workflow patterns persisting across sessions. Not just conversation history but structured knowledge about what works.

### P -- Planning
Multi-step editable plans. Evidence shows allowing users to modify agent plans mid-execution significantly improves outcomes compared to static plans.

### A -- Authority
Trust mechanisms: permission models, approval gates, sandbox boundaries. "Stutter-step agents get old fast" -- excessive prompts kill productivity. The tradeoff between safety and velocity is the central design challenge.

### C -- Control Flow
How much the LLM decides execution paths. Real agents have dynamic, non-deterministic paths; preset workflows are deterministic. The spectrum ranges from fully deterministic pipelines to fully autonomous agent control.

### T -- Tools
RAG, sandboxed code execution, browser automation. The key disagreement centers on management approach: static registration vs. dynamic discovery vs. logit masking.

---

## 13. Production Best Practices and Anti-Patterns

### Best Practices

**1. One feature per session**: Prevents context exhaustion and ensures production-ready code before window depletion. Clean commits create rollback points.

**2. Filesystem as the primary state mechanism**: No fancy vector databases in production harness systems -- just files, git, and structured JSON. Every production harness converges on this pattern.

**3. Test-first development**: Simon Willison's red/green workflow -- human writes failing test, tells agent "make this test pass." The tight loop produces better code with minimal human intervention.

**4. Spec-driven development**: One function, one feature at a time. Commit after each chunk. Quality gates at every step. Cross-check with multiple models (Addy Osmani's approach).

**5. Treat every failure as a permanent engineering fix**: Mitchell Hashimoto's principle -- harness engineering is "engineering a solution every time an agent makes a mistake, ensuring it never makes that specific mistake again." Two practices:
- **Instruction updates**: Modify the agent's instruction file with rules preventing known failures. Each line corresponds to a specific observed failure.
- **Tool construction**: Build tools making correct behavior mechanically verifiable.

**6. Optimize for KV-cache hit rate**: Maintain stable prompt prefixes, append-only contexts, deterministic serialization. This is the single most important production metric.

**7. Preserve reasoning traces**: Cursor discovered that dropping reasoning traces causes 30% performance drops. The harness must maintain reasoning continuity across multi-turn interactions.

**8. Strong-worded behavioral constraints work**: Anthropic found that phrases like "unacceptable to remove tests" effectively condition model behavior. For evaluators, specific terminology like "museum quality" directly shapes output convergence.

### Anti-Patterns

**1. One-shot implementations**: Allowing agents to attempt entire projects in a single pass leads to token exhaustion or premature completion claims.

**2. Self-evaluation without separation**: Agents asked to evaluate their own work "tend to respond by confidently praising the work -- even when the quality is obviously mediocre." Always separate generation from evaluation.

**3. Irreversible context compression**: "You can't reliably predict which observation might become critical ten steps later." Prefer recoverable compression (filesystem URLs, paths) over irreversible summarization.

**4. Dynamic tool definition changes**: Modifying tool definitions invalidates KV-cache and confuses models about prior actions. Use logit masking instead.

**5. Uniform few-shot patterns**: Repetitive action-observation pairs cause models to "fall into a rhythm -- repeating similar actions simply because that's what it sees in context." Introduce structured variation.

**6. Relative filepaths**: Models make systematic errors with relative paths after navigating away from root directories. Enforce absolute paths.

**7. Excessive approval prompts**: Tight permission controls destroy productivity. Use risk-based tiering rather than blanket approval requirements.

**8. Cleaning up failure traces**: Failed actions and error messages should remain in context -- they shift the model's prior away from repeating mistakes.

### Model Improvement and Harness Reassessment

A critical insight from Anthropic: as Claude versions improved (4.5 to 4.6), previously necessary harness components became load-bearing only for edge cases. **Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve.**

---

## 14. The Future: Harness Engineering as a Discipline

### The 2025-2026 Paradigm Shift

If 2025 was the year of the agent, 2026 is the year of the harness. The industry discovered that building an agent is the easy part. Making it reliable, cost-predictable, and safe in production is where the real engineering happens.

### Harness Engineering as Software Engineering

This is genuine software engineering with its own lifecycle:
- The harness becomes its own product with bugs, drift, and maintenance burden
- Martin Fowler and Birgitta Boeckeler identify three components: context engineering, architectural constraints, and entropy management (periodic "garbage collection" agents)
- A weaker model with strong orchestration often outperforms a stronger model with poor orchestration

### Open Research Questions

From Anthropic and the broader community:

1. **Multi-agent specialization vs. single general-purpose agents**: One well-harnessed agent with memory and smart context engineering has been reported to outperform five specialized agents in some contexts. The recommendation: always start single-agent.

2. **Parallel agent coordination on shared codebases**: Orchestrating hundreds of parallel agents remains an unsolved research problem.

3. **Trace-based self-analysis**: Agents analyzing their own execution traces to identify failure patterns.

4. **Dynamic just-in-time tool assembly**: Tools generated on-the-fly for specific tasks rather than pre-registered.

5. **Model-harness co-evolution**: Models becoming specialized within their trained harness contexts, creating optimization potential through harness-specific tuning.

6. **Domain generalization**: Moving beyond software engineering to scientific research, financial modeling, and other domains.

### The Acquisition Signal

Meta's ~$2B acquisition of Manus in December 2025 -- explicitly for the harness, not the model (Manus uses foundation models from Anthropic and OpenAI) -- validates that the harness is where production value lives.

---

## 15. Sources

### Primary Anthropic Sources
- [Effective Harnesses for Long-Running Agents](https://anthropic.com/engineering/effective-harnesses-for-long-running-agents) -- Justin Young, Anthropic Engineering, November 2025
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) -- Anthropic Research, December 2024
- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) -- Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield et al., Anthropic Engineering, September 2025
- [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) -- Anthropic Engineering
- [Building Agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk) -- Anthropic

### OpenAI Sources
- [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) -- OpenAI
- [Unrolling the Codex Agent Loop](https://openai.com/index/unrolling-the-codex-agent-loop/) -- OpenAI
- [Unlocking the Codex Harness: How We Built the App Server](https://openai.com/index/unlocking-the-codex-harness/) -- OpenAI

### Industry Analysis and Frameworks
- [Agent Engineering: Harness Patterns, IMPACT Framework & Coding Agent Architecture](https://www.morphllm.com/agent-engineering) -- MorphLLM, 2026
- [The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/) -- LangChain Blog
- [What Is an Agent Harness?](https://parallel.ai/articles/what-is-an-agent-harness) -- Parallel Web Systems
- [What Is an Agent Harness? The Infrastructure That Makes AI Agents Actually Work](https://www.firecrawl.dev/blog/what-is-an-agent-harness) -- Firecrawl
- [Your Agent Needs a Harness, Not a Framework](https://www.inngest.com/blog/your-agent-needs-a-harness-not-a-framework) -- Inngest Blog
- [Agentic Harness Engineering: LLMs as the New OS](https://www.decodingai.com/p/agentic-harness-engineering) -- Decoding AI

### Manus / Context Engineering
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) -- Manus Blog

### Additional References
- [2025 Was Agents. 2026 Is Agent Harnesses.](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e) -- Aakash Gupta, Medium
- [Agent Frameworks vs Runtime vs Harnesses](https://www.analyticsvidhya.com/blog/2025/12/agent-frameworks-vs-runtimes-vs-harnesses/) -- Analytics Vidhya
- [Anthropic's Three-Agent Harness for Full-Stack AI Development](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/) -- InfoQ
- [The GAN-Style Agent Loop: Deconstructing Anthropic's Harness Architecture](https://www.epsilla.com/blogs/anthropic-harness-engineering-multi-agent-gan-architecture) -- Epsilla Blog
- [Production Multi-Agent System with LangGraph](https://markaicode.com/langgraph-production-agent/) -- Markaicode
- [Human-in-the-Loop Patterns](https://developers.cloudflare.com/agents/guides/human-in-the-loop/) -- Cloudflare Agents Docs
- [Human-in-the-Loop Approval Framework](https://agentic-patterns.com/patterns/human-in-loop-approval-framework/) -- Awesome Agentic Patterns
