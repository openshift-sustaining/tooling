# OCP Sustaining Agentic Bot
## Architecture & Design Document

---

## 1. Executive Summary

The **OCP Sustaining Agentic Bot** is an extensible, AI-powered assistant designed for team internal repeatable tasks and informational workflows. Built with a modular architecture, it enables easy onboarding of new tools/capabilities while providing a transparent, consent-driven user experience. Task execution can be accessed via both UI and API.

### Key Differentiators
- **Transparent AI Reasoning**: Collapsible thinking/planning panel (similar to Cursor, Claude)
- **Consent-First Execution**: User approval required before tool execution
- **White-Label Ready**: Configurable branding for multi-team adoption
- **Extensible Tool System**: Add new capabilities via simple tool registration

---

## 2. High-Level Architecture

![OCP Sustaining Bot - Architecture](./architecture-new.png?v=2)

*High-level architecture showing separate Frontend applications (Chat App, Dashboard App), Programmatic API Access (ARC, Scripts, CI/CD, External Systems), shared Backend (Orchestrator, State, Tools), External Services, and Central Database*

---

## 3. Core Components - Design

### 3.1 Agentic Orchestrator

The brain of the system that coordinates planning, execution, and user consent.

#### 3.1.1 Planner Module

**Purpose**: Analyzes user intent and creates an execution plan using available tools.

**Responsibilities**:
- Parse and understand natural language prompts via LLM
- Query Tool Registry for available capabilities
- Generate step-by-step action plan with tool invocations
- Handle "capability not available" scenarios gracefully
- Stream thinking/reasoning process to frontend

**Input/Output Flow**:
```
User Prompt → Intent Analysis → Tool Matching → Plan Generation → Execution Plan
```

**Plan Structure**:
```
Plan {
    plan_id: string
    user_prompt: string
    thinking_process: string[]          // Reasoning steps (displayed in collapsible panel)
    steps: PlanStep[]
    estimated_duration: string
    requires_consent: boolean
    created_at: timestamp
}

PlanStep {
    step_id: string
    order: number
    tool_name: string
    tool_description: string
    parameters: dict
    expected_output: string
    risk_level: "low" | "medium" | "high"
    requires_individual_consent: boolean
    dependencies: string[]              // step_ids this depends on
    status: "pending" | "approved" | "rejected" | "executing" | "completed" | "failed"
}
```

#### 3.1.2 Executor Module

**Purpose**: Executes approved plan steps by invoking tools while maintaining awareness of Central State.

**Responsibilities**:
- Read from Central State to check preconditions
- Invoke tools with state-derived parameters
- Allow tools to write their outputs to state
- **Executor also writes** execution metadata (step status, retry count, errors)
- Handle consent flow when tool requires it
- Make retry/skip/abort decisions based on state

**State Write Permissions**:
| Writer | What It Writes |
|--------|----------------|
| **Tool** | Domain outputs (cve_details, affected_packages, test_results, etc.) |
| **Executor** | Execution metadata (current_step, retry_count, last_error, step_status) |

**Execution Loop**:
```
for each step in plan:
    1. GET tool config from Tool Registry (including retry policy)
    2. CHECK state.data for required inputs → skip if missing
    3. REQUEST consent if tool.requires_consent == true
    4. EXECUTE tool → tool writes results to state.data
    5. EXECUTOR writes step_status, retry_count to state.data
    6. ON FAILURE:
       ├── IF tool.is_retriable AND state.retry_count < tool.max_retries
       │   └── Wait tool.retry_delay_ms, increment retry_count, GOTO step 4
       ├── ELSE ask Planner for alternative or abort
```

#### 3.1.3 Consent Manager

**Purpose**: Controls the consent workflow for tool executions.

**Consent Modes**:
| Mode | Description |
|------|-------------|
| **Step-by-Step** | User approves each tool action individually |
| **Bulk Approve All Risks** | User approves all remaining steps regardless of risk level |
| **Bulk Approve Low Risks** | Auto-approve "low" risk steps, ask for "medium" and "high" |
| **Bulk Approve Low & Medium Risks** | Auto-approve "low" and "medium" risk steps, ask only for "high" |
| **Skip Confirmation** | For trusted/safe operations (user-configurable) |

**Consent Request Structure**:
```
ConsentRequest {
    request_id: string
    plan_id: string
    step_id: string
    tool_name: string
    tool_description: string
    parameters_summary: string          // Human-readable parameter description
    risk_level: "low" | "medium" | "high"
    reversible: boolean
    timeout_seconds: number             // Auto-reject if no response
    bulk_options: {
        approve_all_risks: boolean      // Allow user to approve all remaining steps
        approve_low_risks: boolean      // Allow user to auto-approve low risk steps
        approve_low_medium_risks: boolean  // Allow user to auto-approve low & medium risk steps
    }
}

ConsentResponse {
    request_id: string
    decision: "approved" | "rejected" | "modified" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium"
    modified_parameters: dict | null    // If user wants to modify params
    user_feedback: string | null        // Optional feedback for improvement
}
```

---

### 3.2 Tool Registry System

#### 3.2.1 Tool Interface

```
Tool {
    // === Identity ===
    name: string                        // Unique identifier
    display_name: string                // Human-friendly name
    description: string                 // What this tool does
    category: "internal" | "external"   // See 3.2.2
    capabilities: string[]              // Keywords for intent matching
    
    // === State Dependencies ===
    state_inputs: string[]              // Required fields from state.data (e.g., ["cve_id", "repo_url"])
    state_outputs: string[]             // Fields this tool writes to state.data (e.g., ["affected_packages"])
    
    // === Behavior ===
    requires_consent: boolean           // Pause for user approval before execution
    risk_level: "low" | "medium" | "high"
    estimated_duration: string
    
    // === Retry Policy (defined per tool) ===
    is_retriable: boolean               // Can this tool be retried on failure?
    max_retries: number                 // Max retry attempts (e.g., 3)
    retry_delay_ms: number              // Delay between retries (e.g., 1000)
    
    // === Tasks (what the tool actually does) ===
    tasks: Task[]                       // One or more low-level operations
    
    // === Execution ===
    execute(state) → ToolResult         // Reads from state, writes back to state
}

Task {
    type: "api_call" | "mcp_call" | "llm_call" | "evaluate" | "store"
    description: string
    target: string                      // API endpoint, MCP server, LLM model, etc.
}

ToolResult {
    success: boolean
    error: string | null
    execution_time_ms: number
    tasks_completed: string[]           // Which tasks ran successfully
}
```

**Example Tool Definition:**
```
Tool: assess_cve
  category: external
  capabilities: ["analyze CVE", "check vulnerability", "assess impact"]
  
  state_inputs: ["cve_id", "cloned_repo_path", "cve_details"]
  state_outputs: ["is_vulnerable", "affected_packages", "is_false_alarm"]
  
  requires_consent: false
  risk_level: low
  
  is_retriable: true
  max_retries: 2
  retry_delay_ms: 1000
  
  tasks:
    - {type: "llm_call", description: "Analyze CVE impact on repo", target: "llm"}
    - {type: "evaluate", description: "Match affected packages with go.mod"}
    - {type: "store", description: "Write results to state"}
```

#### 3.2.2 Tool Categories

| Category | Type | Description | Examples |
|----------|------|-------------|----------|
| **Internal** | Query | Read/process data within the system | search_code, analyze_gomod |
| **Internal** | Process | Transform or evaluate data | assess_cve, check_false_alarm |
| **Internal** | System | Bot utilities | fetch_feedback_insights, generate_summary |
| **External** | API | Call external REST/GraphQL APIs | fetch_cve_data (OSV/NVD), create_pr (GitHub) |
| **External** | MCP | Call MCP servers | Any registered MCP server tool |
| **External** | LLM | Invoke LLM for reasoning | Used within tools as task type |

#### 3.2.3 Tool Registry Structure

```
ToolRegistry {
    tools: Map<tool_name, Tool>
    capability_index: Map<keyword, tool_name[]>   // For fast intent matching
    
    // Methods
    register(tool: Tool)
    get_by_name(name: string) → Tool
    match_capabilities(keywords: string[]) → Tool[]
    get_by_category(category: string) → Tool[]
}
```

---

### 3.3 Frontend Components

#### 3.3.1 Layout Structure

![OCP Sustaining Bot - Chat Panel](./chatpanel.png)

*Chat panel with real-time thinking, execution plan, and consent bar*

![OCP Sustaining Bot - Dashboard](./dashboard.png)

*Dashboard showing session history, analytics, and feedback metrics (Admin only)*

#### 3.3.2 Component Breakdown

**1. Chat Panel**
- Displays conversation history
- Supports Markdown rendering
- Shows tool outputs inline
- Real-time message streaming

**2. Thinking/Planning Panel (Collapsible)**
- Default: Collapsed with summary ("🧠 Planning...")
- Expanded: Shows full reasoning chain
- Live updates during planning
- Reasoning steps with timestamps

**3. Execution Plan Panel**
- Visual step-by-step progress
- Status indicators per step (pending/running/done/failed)
- Click step to see details
- Estimated time remaining

**4. Consent Control Bar**
- Appears when consent needed
- Shows tool name, description, risk level
- Buttons: Approve, Reject, Modify, Approve All Remaining
- Optional: View full parameters

**5. Branding Configuration Layer**
- Logo URL
- Primary/Secondary colors
- Product name
- Custom CSS injection point

**6. Dashboard App (Separate Application - Admin Only)**
- Session History Table: Time, request, status, rating, duration, tools used
- Summary Stats Cards: Total sessions, success rate, avg rating, avg duration
- Tool Performance Chart: Success rate per tool (horizontal bar)
- Feedback Distribution: Star rating breakdown (5★ to 1★)
- Recent Feedback List: 👍/👎 with comments and tool context
- Date Filters: Last 7 days, Last 30 days, All time
- User Authentication: Admin role required for access

#### 3.3.3 Branding Configuration Schema

```
BrandingConfig {
    product_name: string                // "OCP Sustaining Bot"
    product_tagline: string             // "AI-Powered CVE Remediation"
    logo_url: string                    // Path to logo image
    favicon_url: string
    
    theme: {
        primary_color: string           // "#E00" (Red Hat red)
        secondary_color: string
        background_color: string
        text_color: string
        accent_color: string
        font_family: string
    }
    
    footer: {
        company_name: string
        links: [{label, url}]
    }
    
    features: {
        show_thinking_panel: boolean
        default_consent_mode: "step_by_step" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium" | "skip_confirmation"
        enable_feedback: boolean
    }
}
```

---

### 3.4 Central State Management

The **Central Session State** is the single source of truth shared across all tools and the executor. Every tool reads inputs from state and writes outputs back to state, enabling downstream tools to use upstream results.

#### 3.4.1 Central State Design Principles

| Principle | Description |
|-----------|-------------|
| **Single Source of Truth** | One state object per session; all tools reference the same instance |
| **Read-Before-Execute** | Each tool reads required inputs from state before execution |
| **Write-After-Execute** | Each tool writes results/updates to state after completion |
| **Executor State-Awareness** | Executor checks state before making retry/skip/fail decisions |
| **Immutable History** | Previous tool results preserved; new results append/update |

#### 3.4.2 Tool Registry ↔ State Integration

The **Tool Registry configuration** is essential for safe and dynamic state management. Each tool declares its state dependencies (`state_inputs`, `state_outputs`) which the Executor uses to validate and orchestrate workflow execution.

**Tool Registry Configuration for State Management:**

```
Tool Registry Entry:
┌─────────────────────────────────────────────────────────────────────────────┐
│  Tool: assess_cve                                                           │
│  ├── state_inputs: ["cve_id", "cloned_repo_path", "cve_details"]           │
│  ├── state_outputs: ["is_vulnerable", "affected_packages", "is_false_alarm"]│
│  ├── requires_consent: false                                                │
│  ├── risk_level: "low"                                                      │
│  ├── is_retriable: true                                                     │
│  ├── max_retries: 2                                                         │
│  └── retry_delay_ms: 1000                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**How Tool Registry Drives State Management:**

| Tool Config Field | State Management Role |
|-------------------|----------------------|
| `state_inputs` | Executor validates these fields exist in `state.data` before execution |
| `state_outputs` | Executor verifies these fields are updated in `state.data` after execution |
| `requires_consent` | Determines if `pending_consent` is populated before execution |
| `risk_level` | Used with `consent_mode` to determine if consent is auto-approved |
| `is_retriable` | Executor checks this before attempting retry on failure |
| `max_retries` | Compared against `state.data.retry_count` to decide retry vs. abort |
| `retry_delay_ms` | Executor waits this duration before retry attempt |

**Pre-Execution Validation Flow:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRE-EXECUTION VALIDATION                                  │
│                                                                              │
│  1. Executor reads tool.state_inputs from Tool Registry                     │
│     └── ["cve_id", "cloned_repo_path", "cve_details"]                       │
│                                                                              │
│  2. Executor checks each input exists in state.data                         │
│     ├── state.data.cve_id         → ✓ present                              │
│     ├── state.data.cloned_repo_path → ✓ present                            │
│     └── state.data.cve_details    → ✓ present                              │
│                                                                              │
│  3. If any input missing:                                                   │
│     ├── Log warning with missing fields                                     │
│     ├── Check if upstream tool can provide it                               │
│     └── Skip step OR request Planner to re-sequence                        │
│                                                                              │
│  4. If all inputs present:                                                  │
│     └── Proceed to consent check (if tool.requires_consent = true)         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Consent Decision Based on Tool Registry + State:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONSENT DECISION MATRIX                                   │
│                                                                              │
│  Tool Registry:  tool.requires_consent = true                               │
│                  tool.risk_level = "medium"                                 │
│                                                                              │
│  Session State:  state.consent_mode = "bulk_approve_low"                    │
│                                                                              │
│  Decision Logic:                                                            │
│  ├── If consent_mode == "skip_confirmation" → auto-approve                 │
│  ├── If consent_mode == "bulk_approve_all" → auto-approve                  │
│  ├── If consent_mode == "bulk_approve_low" AND risk == "low" → auto-approve│
│  ├── If consent_mode == "bulk_approve_low_medium" AND risk ∈ [low,medium]  │
│  │   → auto-approve                                                         │
│  └── Otherwise → populate state.pending_consent, wait for user response    │
│                                                                              │
│  Result: risk_level="medium" + consent_mode="bulk_approve_low"              │
│          → Requires explicit user consent                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Post-Execution State Update Verification:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POST-EXECUTION VERIFICATION                               │
│                                                                              │
│  1. Tool completes execution                                                │
│                                                                              │
│  2. Executor reads tool.state_outputs from Tool Registry                    │
│     └── ["is_vulnerable", "affected_packages", "is_false_alarm"]            │
│                                                                              │
│  3. Executor verifies state.data was updated:                               │
│     ├── state.data.is_vulnerable    → ✓ updated (true)                     │
│     ├── state.data.affected_packages → ✓ updated ([...])                   │
│     └── state.data.is_false_alarm   → ✓ updated (false)                    │
│                                                                              │
│  4. Record in tool_results:                                                 │
│     └── state.tool_results[step_id] = { success: true, ... }               │
│                                                                              │
│  5. Check for state-triggered workflow changes:                             │
│     └── If state.data.is_false_alarm == true → skip remediation steps      │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.4.3 Session State Structure

```
SessionState {
    // === Session Identity ===
    session_id: string
    user_id: string | null
    created_at: timestamp
    last_activity: timestamp
    
    // === Conversation ===
    messages: Message[]
    
    // === Execution Context ===
    current_plan: Plan | null
    execution_status: ExecutionStatus
    pending_consent: ConsentRequest | null
    current_step_index: number
    
    // === Central Data Store (Tool I/O) ===
    // Each tool reads from and writes to this shared data
    data: {
        // Inputs (populated by early tools, used by later tools)
        cve_id: string | null
        repo_url: string | null
        repo_version: string | null
        cloned_repo_path: string | null
        
        // CVE Analysis Results
        cve_details: CVEDetails | null
        affected_packages: AffectedPackage[] | null
        is_vulnerable: boolean | null
        is_false_alarm: boolean | null
        
        // Remediation State
        fix_available: boolean | null
        fix_version: string | null
        go_mod_updated: boolean | null
        version_changes: VersionChange[] | null
        
        // Test Results
        build_success: boolean | null
        test_success: boolean | null
        test_logs: string | null
        
        // Output
        pr_url: string | null
        summary: string | null
        
        // Error Tracking
        last_error: string | null
        last_error_step: string | null
        retry_count: number
    }
    
    // === Tool Execution History ===
    tool_results: Map<step_id, ToolResult>
    
    // === User Preferences ===
    consent_mode: string
    
    // === Session Metrics ===
    total_tools_executed: number
    total_approvals: number
    total_rejections: number
}
```

#### 3.4.4 Tool ↔ State Interaction Pattern

Every tool follows this pattern:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TOOL EXECUTION PATTERN                               │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  1. READ FROM STATE                                                  │    │
│  │     tool_input = {                                                   │    │
│  │       cve_id: state.data.cve_id,                                    │    │
│  │       repo_path: state.data.cloned_repo_path,                       │    │
│  │       affected_packages: state.data.affected_packages               │    │
│  │     }                                                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  2. EXECUTE TOOL LOGIC                                               │    │
│  │     result = await tool.execute(tool_input)                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  3. WRITE TO STATE                                                   │    │
│  │     state.data.is_vulnerable = result.is_vulnerable                 │    │
│  │     state.data.affected_packages = result.packages                  │    │
│  │     state.tool_results[step_id] = result                            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.4.5 Executor State-Aware Decision Making

The Executor uses current state to make intelligent decisions:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXECUTOR DECISION LOGIC                                   │
│                                                                              │
│  BEFORE each step:                                                           │
│  ├── Check state.data for required inputs                                   │
│  ├── If inputs missing → skip step or request re-plan                      │
│  └── If precondition not met → evaluate skip vs. fail                       │
│                                                                              │
│  AFTER step success:                                                         │
│  ├── Verify state was updated correctly                                     │
│  ├── Check if downstream steps are now viable                               │
│  └── Proceed to next step                                                    │
│                                                                              │
│  AFTER step failure:                                                         │
│  ├── Read state.data.last_error for context                                │
│  ├── Get tool.is_retriable and tool.max_retries from Tool Registry        │
│  ├── Check state.data.retry_count against tool.max_retries                 │
│  ├── Evaluate state to determine:                                           │
│  │   ├── Can we retry? (tool.is_retriable AND retry_count < max_retries)  │
│  │   ├── Can we skip and continue? (check if downstream has alternatives)  │
│  │   └── Must we abort? (critical state missing)                           │
│  └── Update state.data.last_error_step and state.data.retry_count          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Decision Matrix (Sample - Not Exhaustive):**

| Condition | Source | Executor Decision |
|-----------|--------|-------------------|
| `tool.is_retriable == true` AND `state.retry_count < tool.max_retries` | Tool Registry + State | Retry with delay (`tool.retry_delay_ms`) |
| `tool.is_retriable == false` OR `state.retry_count >= tool.max_retries` | Tool Registry + State | Ask Planner for alternative or abort |
| Required input `null` in `state.data` | State | Skip step, log warning |
| `state.is_false_alarm == true` | State | Skip remediation steps, go to summary |
| `state.fix_available == false` | State | Skip apply_fix, generate "no fix" report |
| `state.test_success == false` AND retriable | Tool Registry + State | Retry with Planner guidance |

*Note: This matrix illustrates common decision patterns. Actual decision logic will vary based on the specific tools and workflows configured.*

**Data Source Clarification:**

| Data | Source | Description |
|------|--------|-------------|
| `max_retries` | Tool Registry | Defined per tool (e.g., API tools: 3, LLM tools: 2) |
| `is_retriable` | Tool Registry | Whether tool can be retried on failure |
| `retry_delay_ms` | Tool Registry | Wait time between retries |
| `retry_count` | State | Current attempt count (starts at 0, incremented by Executor) |
| `last_error` | State | Error message from last failure |
| `last_error_step` | State | Step ID that last failed |

#### 3.4.6 State Change Notifications

When state changes, the Executor can notify the Planner for adaptive re-planning:

```
State Change Event:
  state.data.is_false_alarm = true  (set by false_alarm_check tool)
  
Executor detects:
  Remaining steps [apply_fix, test_fix, create_pr] no longer needed
  
Executor action:
  1. Mark remaining steps as "skipped"
  2. Jump to generate_summary step
  3. Notify frontend: "CVE already patched, skipping remediation"
```

#### 3.4.7 Configuration vs. Database Storage

The system separates **application configuration** (static, loaded at startup) from **database storage** (dynamic, persisted for analytics/audit).

**Application Configuration (Loaded at Startup):**
```
App Config (config files / environment):
├── tool_registry.yaml       // Tool definitions, state_inputs/outputs, retry policies
├── branding_config.yaml     // White-label theming, logos, colors
├── llm_config.yaml          // LLM provider, model, API keys
└── system_config.yaml       // Feature flags, timeouts, defaults
```

**Database (Lightweight - Historical Data Only):**
```
Database Tables:
├── sessions                 // Completed session metadata and final state
├── session_messages         // Conversation history (for completed sessions)
├── session_steps            // Plan steps and their execution status
├── session_tool_results     // Tool execution history (for completed sessions)
└── feedback                 // User feedback records
```

**Runtime State (In-Memory):**
```
RuntimeState {
    // Loaded from app config at startup
    tool_registry: Map<tool_name, Tool>
    tool_capability_index: Map<keyword, tool_name[]>
    branding_config: BrandingConfig
    llm_config: LLMConfig
    
    // Active sessions (in-memory only until completed)
    active_sessions: Map<session_id, SessionState>
}
```

**Storage Strategy:**

| Data | Source | Stored In | Reason |
|------|--------|-----------|--------|
| Tool definitions | App config | Memory (loaded at startup) | Static config, no need for DB overhead |
| Branding config | App config | Memory (loaded at startup) | Static config, changes require redeploy |
| LLM config | App config | Memory (loaded at startup) | Static config with secrets |
| Active sessions | Runtime | Memory only | Lightweight, lost on crash (acceptable) |
| Completed sessions | Runtime → DB | Database | Historical data for Dashboard/analytics |
| Feedback | User input → DB | Database | Persistent for analysis and improvement |

**Session Persistence Strategy:**
- Active sessions are kept in memory only during execution
- Session state written to DB only when session is **completed** (success or failure)
- In-progress sessions are not persisted (lost on server crash - acceptable trade-off for simplicity)
- Completed sessions in DB available for Dashboard analytics and audit

---

### 3.5 Feedback System

The feedback system is designed to be **non-blocking** and **lightweight** - it should never slow down execution or interrupt user experience.

#### 3.5.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Non-Blocking** | Feedback submission is fire-and-forget; never waits for DB write |
| **Optional & Unobtrusive** | Small UI elements; user can ignore without penalty |
| **No Analytics Dashboard** | No separate admin UI; feedback accessed via bot tool |
| **On-Demand Insights** | Users query feedback stats by asking the bot directly |

#### 3.5.2 Feedback Collection Points

| Point | UI Element | Behavior |
|-------|------------|----------|
| After tool completion | Small 👍/👎 icons (inline) | Click sends async, no wait |
| After plan completion | Optional 1-5 star rating | Appears briefly, auto-dismisses |
| On error | "Report issue" link | Opens minimal form, submits async |

**UI Design - Minimal & Non-Intrusive:**

```
┌────────────────────────────────────────────────────────────────────────┐
│ ✅ "fetch_cve_data" completed                                          │
│                                                                         │
│ CVE-2024-1234 affects golang.org/x/net versions < 0.17.0               │
│ Severity: HIGH (CVSS 7.5)                                              │
│                                                          [👍] [👎]     │
└────────────────────────────────────────────────────────────────────────┘
       ↑ Small, corner-positioned, doesn't block content
```

**End of Plan - Auto-Dismissing:**
```
┌────────────────────────────────────────────┐
│  How was this analysis?  ⭐⭐⭐⭐⭐        │
│                          [Skip]            │
└────────────────────────────────────────────┘
       ↑ Appears for 5 seconds, then auto-hides if ignored
```

#### 3.5.3 Feedback Data Structure

```
Feedback {
    feedback_id: string
    session_id: string
    context: "tool_result" | "plan_complete" | "error"
    related_id: string                  // step_id, plan_id, etc.
    
    rating: 1-5 | null                  // For plan completion
    sentiment: "positive" | "negative" | null  // For tool results
    free_text: string | null            // Optional comment
    
    // Auto-captured context (no user input required)
    tool_name: string | null
    error_message: string | null
    
    timestamp: timestamp
}
```

#### 3.5.4 Non-Blocking Submission Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ASYNC FEEDBACK SUBMISSION                                 │
│                                                                              │
│   User clicks 👎                                                            │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────┐                                                   │
│   │  Frontend:          │                                                   │
│   │  1. Show "Noted ✓"  │ ← Immediate visual confirmation                   │
│   │  2. Fire WebSocket  │                                                   │
│   │  3. Continue flow   │ ← No waiting for response                         │
│   └─────────────────────┘                                                   │
│        │                                                                     │
│        ▼ (async, non-blocking)                                              │
│   ┌─────────────────────┐                                                   │
│   │  Backend:           │                                                   │
│   │  1. Receive event   │                                                   │
│   │  2. Queue for DB    │ ← Background worker writes to DB                  │
│   │  3. No response     │                                                   │
│   └─────────────────────┘                                                   │
│                                                                              │
│   Execution continues uninterrupted                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```


#### 3.5.5 Feedback Storage

Feedback follows the same persistence strategy as session state:

| Phase | Storage | Notes |
|-------|---------|-------|
| During active session | In-memory (part of SessionState) | Feedback collected as user interacts |
| On session completion | Written to DB | Persisted along with session data |

**SessionState.feedback:**
```
SessionState {
    ...
    feedback: {
        step_ratings: Map<step_id, "positive" | "negative">
        session_rating: number | null        // 1-5 stars
        comments: string[]                   // Optional user comments
    }
}
```

On session completion, feedback is persisted to the `feedback` table in DB and available for the `fetch_feedback_insights` tool to query.


## 4. Dynamic Workflow Architecture

This section explains the core architectural decision to use **LLM-driven dynamic workflows** instead of fixed graph-based flows.

### 4.1 Why Dynamic (Not Fixed Graph)

| Fixed Graph (LangGraph) | Dynamic (LLM-Driven) |
|------------------------|----------------------|
| Hardcoded edges in code | LLM generates at runtime |
| Adding tool = modify graph | Adding tool = register only |
| Predefined retry paths | LLM suggests alternatives |
| Limited to predefined flows | Adapts to novel requests |

### 4.2 Workflow Engine

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PLANNER (LLM)                                        │
│  Inputs:                                                                     │
│  ├── User prompt                                                            │
│  ├── Tool Registry (with capabilities)                                      │
│  └── Current state.data                                                     │
│                                                                              │
│  Output: Execution Plan                                                      │
│  { steps: [{tool, params, depends_on}, ...] }                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXECUTOR                                             │
│  For each step:                                                              │
│  ├── Check state.data for required inputs                                   │
│  ├── Request consent if tool.requires_consent                               │
│  ├── Execute tool → tool writes to state.data                               │
│  └── On failure → ask Planner for recovery plan                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Re-Planning on Failure

When a step fails, Executor asks Planner for recovery:

```
Executor → Planner: {error, failed_step, completed_steps, state.data}

Planner responds:
├── retry with different params
├── skip and continue
└── abort with explanation
```

### 4.4 Capability Not Available

When user requests something no tool can handle:

```
Planner: {
  can_fulfill: false,
  explanation: "I don't have deployment capabilities",
  alternatives: ["I can analyze CVEs", "I can create PRs", "I can run tests"]
}
```

---

## 5. Communication & API

This section consolidates all communication protocols and interfaces for the bot.

### 5.1 Protocol Summary

| Client | Protocol | Format | Auth | Use Case |
|--------|----------|--------|------|----------|
| Chat App | WebSocket (WSS) | JSON messages | Session token | Real-time interactive chat |
| Dashboard App | REST/HTTPS | JSON | Session token | View sessions, analytics, feedback |
| API Clients (ARC, Scripts, CI/CD) | REST/HTTPS | JSON | API key (Bearer) | Programmatic automation |
| Backend → LLM Provider | REST/HTTPS | OpenAI-compatible JSON | API key | Planning, tool execution |
| Backend → External APIs | REST/HTTPS | JSON | Per-service auth | GitHub, OSV/NVD, etc. |
| Backend → MCP Servers | MCP Protocol (stdio/HTTP) | JSON-RPC 2.0 | N/A (local) | Browser, Files, JIRA |

---

### 5.2 Chat App (WebSocket)

Real-time bidirectional communication for interactive chat experience.

**Protocol:** `wss://<host>/ws/{session_id}`

**Connection Flow:**
1. `POST /api/v1/sessions` → returns `session_id`
2. WebSocket connect to `/ws/{session_id}`
3. Bidirectional message stream until disconnect

**Chat App → Backend:**

| Type | Purpose | Payload |
|------|---------|---------|
| `chat` | User sends message | `{content: string}` |
| `consent_response` | User approves/rejects | `ConsentResponse` |
| `cancel_execution` | Stop current plan | `{plan_id}` |
| `pong` | Response to ping | `{}` |

**Backend → Chat App:**

| Type | Purpose | Payload |
|------|---------|---------|
| `thinking_update` | Reasoning stream | `{step: string}` |
| `plan_created` | Plan ready | `{plan: Plan}` |
| `consent_request` | Ask for approval | `ConsentRequest` |
| `step_started` | Tool begins | `{step_id, tool_name}` |
| `step_progress` | During execution | `{step_id, progress: string}` |
| `step_completed` | Tool finishes | `{step_id, success, result/error}` |
| `plan_completed` | All steps done | `{plan_id, summary}` |
| `error` | On failure | `{message, recoverable}` |
| `ping` | Keepalive (every 30s) | `{}` |

**Connection Resilience:**

| Mechanism | Purpose |
|-----------|---------|
| Heartbeat (ping/pong) | Detect dead connections; prevent proxy timeouts |
| Reconnection | Frontend auto-reconnects on disconnect; resumes session |

---

### 5.3 Dashboard App (REST)

Standard REST API for admin dashboard to view historical data.

**Base URL:** `https://<host>/api/v1`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/sessions` | List completed sessions (with filters) |
| `GET` | `/sessions/{id}` | Get session details |
| `GET` | `/feedback` | Get feedback records |
| `GET` | `/analytics/summary` | Aggregated metrics |

---

### 5.4 Programmatic API (REST)

For automation, CI/CD pipelines, ARC, and external system integration.

**Base URL:** `https://<host>/api/v1`

**Authentication:** `Authorization: Bearer <api_key>`

#### Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/queries` | Submit new query |
| `GET` | `/queries/{id}` | Check status (polling) |
| `GET` | `/queries/{id}/result` | Get final result |
| `GET` | `/tools` | List available tools |
| `GET` | `/health` | Health check |

#### Submit Query

```
POST /api/v1/queries

Request:
{
    "prompt": "Analyze CVE-2024-24786 for openshift/cluster-nfd-operator",
    "consent_mode": "bulk_approve_all",    // or bulk_approve_low, bulk_approve_low_medium
    "callback_url": "https://...",         // Optional: webhook on completion
    "timeout_seconds": 600
}

Response:
{
    "query_id": "q-abc123",
    "status": "queued",
    "poll_url": "/api/v1/queries/q-abc123"
}
```

#### Check Status

```
GET /api/v1/queries/{query_id}

Response:
{
    "query_id": "q-abc123",
    "status": "running",                   // queued | running | completed | failed
    "progress": {
        "current_step": 3,
        "total_steps": 6,
        "current_tool": "assess_cve"
    }
}
```

#### Get Result

```
GET /api/v1/queries/{query_id}/result

Response:
{
    "query_id": "q-abc123",
    "status": "completed",
    "result": {
        "summary": "Repository is vulnerable. PR created.",
        "is_vulnerable": true,
        "pr_url": "https://github.com/...",
        "affected_packages": [...]
    },
    "execution_log": [
        {"step": "fetch_cve_data", "status": "completed", "duration_ms": 1200},
        {"step": "clone_repository", "status": "completed", "duration_ms": 3400}
    ]
}
```

#### Webhook (Optional)

If `callback_url` is provided, backend sends POST on completion:

```json
{
    "query_id": "q-abc123",
    "status": "completed",
    "result_url": "/api/v1/queries/q-abc123/result"
}
```

#### API Authentication

| Mechanism | Description |
|-----------|-------------|
| **API Keys** | Long-lived keys for automation |
| **Scopes** | `read`, `execute`, `execute:auto_approve` |
| **Rate Limiting** | Prevent abuse (configurable) |
| **Audit Logging** | All API calls logged with key ID |

#### Example: CI/CD Integration

```yaml
# GitHub Actions Example
- name: Check CVE
  run: |
    QUERY_ID=$(curl -X POST https://bot.example.com/api/v1/queries \
      -H "Authorization: Bearer ${{ secrets.BOT_API_KEY }}" \
      -d '{"prompt": "Check CVE-2024-1234", "consent_mode": "bulk_approve_all"}' \
      | jq -r '.query_id')
    
    while true; do
      STATUS=$(curl -s https://bot.example.com/api/v1/queries/$QUERY_ID \
        -H "Authorization: Bearer ${{ secrets.BOT_API_KEY }}" \
        | jq -r '.status')
      if [ "$STATUS" == "completed" ] || [ "$STATUS" == "failed" ]; then break; fi
      sleep 10
    done
    
    curl https://bot.example.com/api/v1/queries/$QUERY_ID/result \
      -H "Authorization: Bearer ${{ secrets.BOT_API_KEY }}"
```

---

## 6. Security Considerations

### 6.1 Authentication & Authorization

| Layer | Mechanism |
|-------|-----------|
| Frontend → Backend | HMAC-signed requests (server-side) |
| Backend → External APIs | API keys (environment variables) |
| Session Management | UUID-based sessions with timeout |
| Dashboard Access | Role-based (admin only) |

**User Roles:**

| Role | Permissions |
|------|-------------|
| `user` | Chat access, own session history |
| `admin` | Chat access, Dashboard access, all session history, analytics |

### 6.2 Tool Execution Safety

- All tool parameters validated against schema
- Dangerous operations require explicit consent
- Audit log of all tool executions
- Rate limiting on tool invocations
- Sandboxed execution where possible

### 6.3 Data Protection

- No sensitive data in logs (masked)
- Session data expires after inactivity
- API keys never sent to frontend
- WebSocket connections authenticated

---

## 7. Technology Stack

### 7.1 Stack Overview

| Layer | Technology | Why This Choice |
|-------|------------|-----------------|
| **Frontend Framework** | React + TypeScript | Component-based architecture, strong typing, large ecosystem, easy to find developers |
| **UI Styling** | Tailwind CSS | Rapid styling, utility-first approach, easy theming for white-label requirements |
| **Frontend State** | React Context + useReducer | Simple, no external dependencies, sufficient for session-scoped state |
| **Backend Framework** | Python + FastAPI | Native async support, automatic Pydantic validation, WebSocket built-in, excellent for AI/ML integrations |
| **Real-time Communication** | WebSocket (native) | Bidirectional streaming required for thinking/progress updates, lower latency than polling |
| **LLM Provider** | Any LLM with function calling (e.g., Gemini, OpenAI, Claude, Ollama) | Configurable; requires structured output and function calling support |
| **Active Session Store** | In-memory (Pydantic model / Python dict) | Fast during execution; no DB latency for active sessions |
| **Persistent Database** | SQLite (dev) / PostgreSQL (prod) | SQLite for local dev (zero config), PostgreSQL for multi-container production |
| **ORM** | SQLAlchemy | Supports both SQLite and PostgreSQL with same codebase |
| **Containerization** | Docker | Portable, consistent environments across dev/staging/prod |
| **Orchestration** | Docker Compose (dev) / Kubernetes (prod) | Simple local dev, scalable production deployment |

---

## 8. Persistence Layer

This section covers the database architecture for storing session history, feedback, and analytics data.

### 8.1 Storage Strategy

| Phase | Storage | Purpose |
|-------|---------|---------|
| **Active Session** | In-memory (Pydantic model / Python dict) | Fast read/write during execution |
| **Session End** | Push to Database | Persist for dashboard & analytics |
| **Dashboard Query** | Read from Database | Historical view & reporting |

### 8.2 Database Support

The system supports both **SQLite** (local development) and **PostgreSQL** (production/containers) via configuration:

```
DATABASE_CONFIG {
    type: "sqlite" | "postgresql"
    
    # SQLite (local/dev)
    sqlite_path: "./data/ocp_bot.db"
    
    # PostgreSQL (production/containers)
    postgres_host: string
    postgres_port: number (default: 5432)
    postgres_database: string
    postgres_user: string
    postgres_password: string (from secret)
    
    # Connection pool
    pool_size: number (default: 5)
    max_overflow: number (default: 10)
}
```

| Mode | Database | Use Case |
|------|----------|----------|
| **Local/Dev** | SQLite | Single developer, quick setup, file-based |
| **Production** | PostgreSQL | Multi-container deployment, concurrent access, scalable |

### 8.3 Data Model

Core entities stored in the database:

| Entity | Purpose |
|--------|---------|
| **sessions** | Session metadata, status, duration, user prompt, outputs |
| **session_messages** | Conversation history (user and assistant messages) |
| **session_steps** | Individual tool executions per session with status and timing |
| **feedback** | User ratings and comments (session-level and step-level) |
| **users** | User accounts with roles (user/admin) |

> **Note:** Detailed schemas, entity relationships, and archival flows are documented separately in the Database Design Document.

### 8.4 Scalable Architecture

For production deployments where each session runs in a separate backend container:

- **Chat App**: Standalone React app for user interactions (WebSocket → Backend)
- **Dashboard App**: Standalone React app for admin analytics (REST API → Backend/DB)
- **Backend Containers**: One per active session, state held in-memory, serves both apps
- **Central Database**: PostgreSQL container for persistent storage
- **Session Archival**: On session end, data pushed to central DB

### 8.5 Dashboard Access Control

| Role | Chat App | Dashboard App |
|------|----------|---------------|
| `user` | ✓ Full access | ✗ No access |
| `admin` | ✓ Full access | ✓ Full access |

- Dashboard App requires authentication with admin role
- Backend validates role on all dashboard API endpoints (403 if unauthorized)
- Separate deployments allow different access policies per app

---

## 9. Success Criteria

| Criteria | Measurement |
|----------|-------------|
| Extensibility | New tool added in < 1 hour |
| Transparency | 100% of planning visible in UI |
| Consent | Zero tool executions without approval |
| Rebranding | White-label config change in < 5 mins |
| User Satisfaction | > 80% positive feedback |

---

## 10. Functional Flow Diagram

![OCP Sustaining Bot - Functional Flow](./representative_flow.png)

*Representative flow diagram illustrating the key execution flows for understanding. Not exhaustive, but captures the important interactions between Clients, Backend components (Planner, Executor, Consent Manager, Tools, Feedback Handler), and External services.*

**Flow Summary:**

| Component | Role |
|-----------|------|
| **Clients** | Chat App (WSS), API Client (REST), Dashboard (REST) |
| **Planner** | Receives intent, generates Plan Steps, interacts with State and LLM |
| **Executor** | Runs steps in loop: Get Step → Consent → Run Tool → Success/Fail |
| **Consent Manager** | Checks consent_mode for each step before execution |
| **Tools** | Execute against External services (GitHub, JIRA, OSV/NVD, Browser, Files) |
| **Feedback Handler** | Collects feedback, flushes State to Database on completion |

**Key Flows:**

1. **Normal Flow**: Client → Planner → Executor Loop → Tools → Success → Feedback → DB
2. **Re-plan Flow**: On failure, loops back to Planner for revised plan
3. **Step Loop**: Executor iterates through plan steps until all complete
4. **State Flush**: On session completion, state is persisted to Database

---

*Document Version: 1.0*  
*Last Updated: January 2026*

