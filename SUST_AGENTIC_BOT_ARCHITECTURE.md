# OCP Sustaining Agentic Bot
## Architecture & Design Document

---

## 1. Executive Summary

The **OCP Sustaining Agentic Bot** is an extensible, AI-powered assistant designed for team internal repeatable tasks and informational workflows. Built with a modular architecture using ephemeral containers (1:1 session-to-container mapping), it enables easy onboarding of new tools/capabilities while providing a transparent, consent-driven user experience. Task execution can be accessed via both UI (single React app with chat and dashboard) and programmatic API.

### Key Differentiators
- **Transparent AI Reasoning**: Collapsible thinking/planning panel (similar to Cursor, Claude)
- **Consent-First Execution**: User approval required before tool execution
- **White-Label Ready**: Configurable branding for multi-team adoption
- **Extensible Tool System**: Add new capabilities via simple tool registration

---

## 2. High-Level Architecture

![OCP Sustaining Bot - Architecture](./architecture-new.png?v=2)

*High-level architecture showing Single React App (with Chat and Dashboard), Programmatic API Access (ARC, Scripts, CI/CD, External Systems), ephemeral Backend containers (1:1 per session), External Services, and Central PostgreSQL Database*

**Architecture Model:**
- **1:1 Container-Per-Session:** Each session (UI or API) gets a dedicated ephemeral backend container
- **Single React App:** Unified UI with Chat and Dashboard routes, both connecting to the same backend container
- **Unified Backend:** Single FastAPI application serving WebSocket (chat) + REST API (dashboard, programmatic access)
- **Container Orchestration:** Platform-agnostic design; supports Kubernetes, Serverless, or other orchestrators (see Section 8.4)

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
# Execute steps respecting dependencies (can run parallel steps if no dependencies)
for each step in plan:
    1. GET tool config from Tool Registry (including retry policy)
    2. CHECK step dependencies are satisfied
    3. CHECK state.data for required inputs → skip if missing
    4. REQUEST consent if tool.requires_consent == true
       ├── IF consent timeout → STOP execution, notify user "Consent timeout"
    5. EXECUTE tool → tool writes results to state.data
    6. EXECUTOR writes step_status to state.data
    7. ON FAILURE:
       ├── IF tool.is_retriable AND step_retry_count < tool.max_retries
       │   └── Wait tool.retry_delay_ms, increment step_retry_count, GOTO step 5
       ├── ELSE ask Planner for recovery plan (automatic LLM-driven recovery)
    8. Send step_progress updates to frontend after each step
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
    timeout_seconds: number             // Default: 300 (5 mins); if exceeded, execution STOPS
    bulk_options: {
        approve_all_risks: boolean      // Allow user to approve all remaining steps
        approve_low_risks: boolean      // Allow user to auto-approve low risk steps
        approve_low_medium_risks: boolean  // Allow user to auto-approve low & medium risk steps
    }
}

ConsentResponse {
    request_id: string
    decision: "approved" | "rejected" | "modified" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium"
    modified_parameters: dict | null    // If user wants to modify params (validated against tool schema)
    user_feedback: string | null        // Optional feedback for improvement
}
```

**Consent Timeout Behavior:**
- If user doesn't respond within `timeout_seconds`, execution stops
- User receives notification: "Consent request timed out. Execution stopped."
- Session remains active; user can start a new request

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
    task_id: string                     // Unique identifier within tool
    type: "api_call" | "mcp_call" | "llm_call" | "evaluate" | "store"
    description: string
    target: string                      // API endpoint, MCP server, LLM model, etc.
    dependencies: string[]              // task_ids this task depends on (empty = can run immediately)
    parameters: dict                    // Task-specific parameters
}

ToolResult {
    success: boolean
    error: string | null
    execution_time_ms: number
    tasks_completed: string[]           // Which tasks ran successfully
}

**Task Execution Logic:**
- Tasks with no dependencies run immediately (can be parallelized)
- Tasks with dependencies wait for those tasks to complete
- State is used to pass data between tasks
- If any task fails, tool execution stops unless task is marked as optional
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
    - {task_id: "t1", type: "llm_call", description: "Analyze CVE impact",
       target: "llm", dependencies: [], parameters: {...}}
    - {task_id: "t2", type: "evaluate", description: "Match packages with go.mod",
       dependencies: ["t1"], parameters: {...}}
    - {task_id: "t3", type: "store", description: "Write results to state",
       dependencies: ["t2"], parameters: {...}}
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

    // Methods
    register(tool: Tool)
    get_by_name(name: string) → Tool
    get_by_category(category: string) → Tool[]
    get_all_tools() → Tool[]            // For LLM prompt context
}
```

**Capability Matching:**
- **LLM-Driven**: No complex keyword indexing needed
- Planner receives full tool list with descriptions and capabilities
- LLM selects relevant tools based on user intent
- Simple and flexible; adapts to novel requests

---

#### 3.2.4 MCP Server Integration

**Model Context Protocol (MCP)** servers are integrated as standard tools in the Tool Registry, supporting both stdio and HTTP/SSE transport modes.

**MCP Tool Configuration:**

```
MCPTool extends Tool {
    // Standard Tool fields
    name: string
    display_name: string
    description: string
    category: "external"

    // MCP-specific fields
    mcp_config: {
        transport: "stdio" | "http"

        // stdio configuration
        stdio?: {
            command: string             // e.g., "npx"
            args: string[]              // e.g., ["-y", "@modelcontextprotocol/server-filesystem"]
            env: dict                   // Environment variables
            per_session: boolean        // true = new process per session, false = shared (NOT SUPPORTED)
        }

        // HTTP configuration
        http?: {
            url: string                 // e.g., "http://localhost:3000/mcp"
            health_check_url: string    // e.g., "http://localhost:3000/health"
            health_check_interval_ms: number  // Default: 30000 (30s)
            headers: dict               // Optional auth headers
        }

        // MCP server capabilities (discovered via tools/list)
        available_tools: MCPServerTool[]
    }
}

MCPServerTool {
    name: string                        // Tool name from MCP server
    description: string
    input_schema: dict                  // JSON schema for parameters
}
```

**Example MCP Tool Definitions:**

```yaml
# stdio MCP Server (Browser)
- name: mcp_browser
  display_name: "Browser Automation"
  description: "Control browser for web interactions"
  category: external
  requires_consent: true
  risk_level: medium
  is_retriable: true
  max_retries: 1
  retry_delay_ms: 2000

  mcp_config:
    transport: stdio
    stdio:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-puppeteer"]
      env: {}
      per_session: true               # New browser instance per session
    available_tools:
      - {name: "puppeteer_navigate", description: "Navigate to URL", input_schema: {...}}
      - {name: "puppeteer_screenshot", description: "Take screenshot", input_schema: {...}}

# HTTP MCP Server (JIRA)
- name: mcp_jira
  display_name: "JIRA Integration"
  description: "Fetch and update JIRA issues"
  category: external
  requires_consent: true
  risk_level: high
  is_retriable: true
  max_retries: 2
  retry_delay_ms: 1000

  mcp_config:
    transport: http
    http:
      url: "http://jira-mcp-server:3000/mcp"
      health_check_url: "http://jira-mcp-server:3000/health"
      health_check_interval_ms: 30000
      headers:
        Authorization: "Bearer ${JIRA_MCP_TOKEN}"
    available_tools:
      - {name: "jira_get_issue", description: "Fetch JIRA issue", input_schema: {...}}
      - {name: "jira_create_issue", description: "Create JIRA issue", input_schema: {...}}
```

**MCP Server Lifecycle Management:**

| Transport | Lifecycle | Health Check | Error Handling |
|-----------|-----------|--------------|----------------|
| **stdio** | Spawned per session (configurable), terminated on session end | Process exit monitoring | Auto-restart on crash (max 3 attempts) |
| **HTTP** | Pre-started external service | HTTP health endpoint polled every 30s | Skip tool execution if unhealthy, notify user |

**MCP Tool Execution Flow:**

```
1. Planner selects MCP tool (e.g., "mcp_browser.puppeteer_navigate")
2. Executor checks:
   ├── stdio: Is process running? If not, spawn and wait for ready
   ├── http: Is server healthy? If not, skip and notify user
3. Executor sends JSON-RPC 2.0 request to MCP server
4. MCP server executes and returns result
5. Tool writes result to state.data
6. Executor continues to next step
```

**MCP Tool Discovery (Optional):**
- MCP servers can be auto-discovered via `tools/list` JSON-RPC call
- Discovered tools added to Tool Registry at startup
- Enables dynamic tool availability based on MCP server state

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
- Expanded: Shows high-level reasoning steps (not full LLM stream)
- Live updates during planning
- Example steps: "Analyzing user intent...", "Identified tools: fetch_cve, assess_cve", "Creating execution plan..."
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

**6. Dashboard View (Admin Only - Same React App, Different Route)**
- Accessible via `/dashboard` route in the same React app
- Session History Table: Time, request, status, rating, duration, tools used
- Summary Stats Cards: Total sessions, success rate, avg rating, avg duration
- Tool Performance Chart: Success rate per tool (horizontal bar)
- Feedback Distribution: Star rating breakdown (5★ to 1★)
- Recent Feedback List: 👍/👎 with comments and tool context
- Date Filters: Last 7 days, Last 30 days, All time
- User Authentication: Admin role required for access
- **Uses same backend container as chat** - queries historical data via REST API

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
| `max_retries` | Compared against `state.data.step_retry_counts[step_id]` to decide retry vs. ask Planner |
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
    idle_timeout_ms: number             // Default: 300000 (5 mins), configurable

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

        // Error Tracking (per step)
        last_error: string | null
        last_error_step: string | null
        step_retry_counts: Map<step_id, number>  // Retry count per step (NOT global)
    }

    // === Tool Execution History ===
    tool_results: Map<step_id, ToolResult>

    // === User Preferences ===
    consent_mode: string

    // === Session Metrics ===
    total_tools_executed: number
    total_approvals: number
    total_rejections: number

    // === Feedback (persisted to DB on session completion) ===
    feedback: {
        step_ratings: Map<step_id, "positive" | "negative">
        session_rating: number | null   // 1-5 stars
        comments: string[]              // Optional user comments
    }
}
```

**Session Timeout Behavior:**
- If `last_activity` exceeds `idle_timeout_ms`, session is marked as timed out
- User receives notification: "Session timed out due to inactivity. Please refresh."
- Session data remains in memory for potential recovery (configurable retention)

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
│  ├── Check step dependencies are satisfied (can run in parallel if none)    │
│  ├── Check state.data for required inputs                                   │
│  ├── If inputs missing → skip step or request re-plan from Planner         │
│  └── If precondition not met → evaluate skip vs. fail                       │
│                                                                              │
│  AFTER step success:                                                         │
│  ├── Verify state was updated correctly (check state_outputs)              │
│  ├── Check if downstream steps are now viable                               │
│  ├── Send step_progress update to frontend                                  │
│  └── Proceed to next step (or parallel steps if dependencies met)           │
│                                                                              │
│  AFTER step failure:                                                         │
│  ├── Read state.data.last_error for context                                │
│  ├── Get tool.is_retriable and tool.max_retries from Tool Registry        │
│  ├── Get step_retry_count from state.data.step_retry_counts[step_id]      │
│  ├── Evaluate state to determine:                                           │
│  │   ├── Can we retry? (tool.is_retriable AND step_retry_count < max_retries)│
│  │   ├── If yes: Increment step_retry_counts[step_id], wait retry_delay_ms│
│  │   └── If no: Ask Planner for recovery plan (automatic LLM decision)     │
│  └── Update state.data.last_error_step and step_retry_counts[step_id]     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Decision Matrix (Sample - Not Exhaustive):**

| Condition | Source | Executor Decision |
|-----------|--------|-------------------|
| `tool.is_retriable == true` AND `step_retry_count < tool.max_retries` | Tool Registry + State | Retry with delay (`tool.retry_delay_ms`) |
| `tool.is_retriable == false` OR `step_retry_count >= tool.max_retries` | Tool Registry + State | Ask Planner for recovery plan (automatic LLM-driven) |
| Required input `null` in `state.data` | State | Skip step, log warning, ask Planner for re-sequence |
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
| `step_retry_count` | State | Current retry count for this specific step (NOT global) |
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
├── mcp_servers.yaml         // MCP server configurations
└── system_config.yaml       // Feature flags, timeouts, defaults, rate limits
```

**LLM Configuration Schema (llm_config.yaml):**

```yaml
llm_config:
  provider: "openai" | "gemini" | "anthropic" | "ollama"
  model: "gpt-4o" | "gemini-2.0-flash-exp" | "claude-sonnet-4" | "llama3:latest"
  api_key: "${LLM_API_KEY}"          # From environment variable
  base_url: "https://api.openai.com/v1"  # Optional: for custom endpoints

  # Planner LLM settings
  planner:
    temperature: 0.1                  # Low temp for consistent planning
    max_tokens: 4096
    timeout_seconds: 30
    system_prompt_template: "planner_system_prompt.txt"

  # Tool LLM calls (used within tools)
  tool_llm_calls:
    temperature: 0.3                  # Slightly higher for creative analysis
    max_tokens: 2048
    timeout_seconds: 20

  # Fallback configuration (optional)
  fallback:
    enabled: true
    provider: "gemini"
    model: "gemini-1.5-flash"
    trigger_on: ["timeout", "rate_limit", "model_unavailable"]
```

**System Configuration Schema (system_config.yaml):**

```yaml
system_config:
  # Session settings
  session:
    idle_timeout_ms: 300000           # 5 minutes default
    consent_timeout_seconds: 300      # 5 minutes default
    max_active_sessions: 100          # Per backend instance

  # Rate limiting
  rate_limits:
    session_creation:
      limit: 10                       # Requests per window
      window_seconds: 3600            # 1 hour
      scope: "user"                   # "user" or "ip"

    api_endpoint:
      limit: 100                      # Requests per window
      window_seconds: 60              # 1 minute
      scope: "api_key"

    llm_calls:
      limit: 50                       # Calls per session
      scope: "session"

    tool_executions:
      limit: 100                      # Executions per session
      scope: "session"

  # Security
  security:
    session_secret: "${SESSION_SECRET}"  # From environment
    api_key_bcrypt_rounds: 12
    enable_message_signing: false
    sign_message_types: ["consent_request", "plan_created"]

  # Feature flags
  features:
    enable_feedback: true
    enable_mcp_servers: true
    enable_parallel_steps: true
    enable_auto_recovery: true       # LLM-driven error recovery

  # WebSocket settings
  websocket:
    ping_interval_seconds: 30
    max_message_size_bytes: 1048576  # 1 MB

  # Database settings (see DATABASE_CONFIG in Section 8.2)
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
    branding_config: BrandingConfig
    llm_config: LLMConfig

    // Active session (one per backend container)
    session_state: SessionState
}
```

**Note:** Each ephemeral backend container holds state for exactly one session. No shared state between containers.

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
- Active sessions are kept in memory within their dedicated backend container
- Each container manages exactly one session (ephemeral 1:1 mapping)
- Session state written to PostgreSQL only when session **completes** (success or failure)
- In-progress sessions are not persisted (lost on container crash - acceptable for ephemeral architecture)
- Completed sessions in DB available for Dashboard analytics via `/dashboard` route

---

### 3.5 Feedback System

The feedback system is designed to be **non-blocking** and **lightweight** - it should never slow down execution or interrupt user experience.

#### 3.5.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Non-Blocking** | Feedback submission is fire-and-forget; never waits for DB write |
| **Optional & Unobtrusive** | Small UI elements; user can ignore without penalty |
| **Dashboard Integration** | Feedback visible in `/dashboard` route (admin only) via REST API queries |
| **On-Demand Analytics** | Admins query feedback metrics through dashboard interface |

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
| During active session | In-memory (part of SessionState.feedback) | Feedback collected as user interacts |
| On session completion | Written to DB | Persisted along with session data to `feedback` table |

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
│  ├── Full Tool Registry (all tools with descriptions + capabilities)        │
│  │   └── LLM selects relevant tools (no keyword matching needed)           │
│  └── Current state.data                                                     │
│                                                                              │
│  Output: Execution Plan                                                      │
│  { steps: [{tool, params, depends_on}, ...] }                               │
│                                                                              │
│  Thinking Stream (high-level):                                              │
│  - "Analyzing user intent..."                                               │
│  - "Identified tools: fetch_cve_data, assess_cve, create_pr"                │
│  - "Creating execution plan with 6 steps..."                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXECUTOR                                             │
│  For each step (can run parallel if no dependencies):                       │
│  ├── Check step dependencies satisfied                                      │
│  ├── Check state.data for required inputs                                   │
│  ├── Request consent if tool.requires_consent (with timeout)                │
│  ├── Execute tool → tool writes to state.data                               │
│  ├── Send step_progress update to frontend                                  │
│  └── On failure → ask Planner for recovery plan (automatic LLM decision)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Re-Planning on Failure

When a step fails and retries are exhausted, Executor asks Planner for automatic recovery (LLM-driven decision):

**Recovery Request:**
```
Executor → Planner: {
    error: string,
    failed_step: PlanStep,
    completed_steps: PlanStep[],
    current_state: state.data,
    remaining_steps: PlanStep[]
}
```

**Recovery Plan Response:**
```
RecoveryPlan {
    strategy: "retry" | "skip" | "abort" | "replan"

    // For "retry" strategy
    modified_step: PlanStep | null      // Retry with different parameters

    // For "replan" strategy
    new_steps: PlanStep[] | null        // Replacement steps for remaining plan

    // For all strategies
    explanation: string                 // Why this recovery approach (shown to user)
    user_notification: string           // Message to display in chat
}
```

**Example Recovery Scenarios:**

| Failure | Planner Decision | Explanation |
|---------|------------------|-------------|
| OSV API timeout | Retry with NVD API instead | "OSV API unreachable, trying NVD API" |
| No vulnerable packages found | Skip to summary | "Repository not affected, skipping remediation" |
| GitHub PR creation failed (auth) | Abort | "GitHub authentication failed, cannot create PR" |
| Build tests fail after patch | Replan with rollback steps | "Tests failed, rolling back changes" |

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

| Client / Component | Protocol | Format | Auth | Use Case |
|-------------------|----------|--------|------|----------|
| React App (/chat route) | WebSocket (WSS) | JSON messages | Session token (JWT) | Real-time interactive chat |
| React App (/dashboard route) | REST/HTTPS | JSON | Session token (JWT) | View sessions, analytics, feedback |
| API Clients (ARC, Scripts, CI/CD) | REST/HTTPS | JSON | API key (Bearer) | Programmatic automation |
| Backend → LLM Provider | REST/HTTPS | OpenAI-compatible JSON | API key | Planning, tool execution |
| Backend → External APIs | REST/HTTPS | JSON | Per-service auth | GitHub, OSV/NVD, etc. |
| Backend → MCP Servers | MCP Protocol (stdio/HTTP) | JSON-RPC 2.0 | N/A (local) | Browser, Files, JIRA |

**Note:** Both `/chat` and `/dashboard` routes connect to the **same ephemeral backend container** for a given session.

---

### 5.2 WebSocket Communication (Chat Route)

Real-time bidirectional communication for interactive chat experience via `/chat` route in React app.

**Protocol:** `wss://<container>/ws/{session_id}`

**Connection Flow:**
1. React app loads → `POST /api/v1/sessions` (to Session Gateway)
2. Gateway spawns dedicated backend container → Returns container URLs + session token
3. React app connects:
   - `/chat` route → WebSocket to dedicated container
   - `/dashboard` route → REST API to same dedicated container
4. Bidirectional communication until session ends

**Session Creation Endpoint:**

```
POST /api/v1/sessions (handled by Session Gateway)

Request:
{
    "user_id": "user@example.com",    // Optional (for authenticated users)
    "consent_mode": "step_by_step" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium" | "skip_confirmation"
}

Response:
{
    "session_id": "sess-abc123def456",
    "container_id": "container-a1b2c3",
    "websocket_url": "wss://container-a1b2c3.cluster/ws/sess-abc123def456",  // Chat route
    "api_base_url": "https://container-a1b2c3.cluster/api/v1",              // Dashboard route
    "session_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",              // JWT token
    "expires_at": "2026-02-04T13:00:00Z",
    "idle_timeout_ms": 300000         // 5 minutes
}
```

**Authentication:**
- Include `session_token` as query parameter: `/ws/{session_id}?token={session_token}` (WebSocket)
- OR include in Authorization header: `Authorization: Bearer {session_token}` (REST API)
- Token validated on connection; invalid token returns 403

**React App → Backend (WebSocket):**

| Type | Purpose | Payload |
|------|---------|---------|
| `chat` | User sends message | `{content: string}` |
| `consent_response` | User approves/rejects | `ConsentResponse` |
| `feedback_submit` | User submits feedback (👍/👎/⭐) | `{step_id?: string, rating?: 1-5, sentiment?: "positive"\|"negative", comment?: string}` |
| `cancel_execution` | Stop current plan | `{plan_id}` |
| `pong` | Response to ping | `{}` |

**Backend → React App (WebSocket):**

| Type | Purpose | Payload |
|------|---------|---------|
| `thinking_update` | Reasoning stream (high-level steps) | `{step: string}` |
| `plan_created` | Plan ready | `{plan: Plan}` |
| `consent_request` | Ask for approval | `ConsentRequest` |
| `consent_timeout` | Consent request timed out | `{request_id, message: "Consent timeout. Execution stopped."}` |
| `step_started` | Tool begins | `{step_id, tool_name}` |
| `step_progress` | During execution | `{step_id, progress: string}` |
| `step_completed` | Tool finishes | `{step_id, success, result/error}` |
| `plan_completed` | All steps done | `{plan_id, summary}` |
| `session_timeout` | Session idle timeout | `{message: "Session timed out. Please refresh."}` |
| `error` | On failure | `{message, recoverable}` |
| `ping` | Keepalive (every 30s) | `{}` |

**Connection Resilience:**

| Mechanism | Purpose |
|-----------|---------|
| Heartbeat (ping/pong) | Detect dead connections; prevent proxy timeouts |
| Reconnection | Frontend auto-reconnects on disconnect; resumes session |

---

### 5.3 Dashboard REST API

Dashboard route (`/dashboard` in React app) queries historical data via REST API from the same backend container.

**Base URL:** `https://<container>/api/v1` (same backend as chat)

| Method | Endpoint | Purpose | Access |
|--------|----------|---------|--------|
| `GET` | `/sessions` | List completed sessions (with filters) | Admin only |
| `GET` | `/sessions/{id}` | Get session details | Admin only |
| `GET` | `/feedback` | Get feedback records | Admin only |
| `GET` | `/analytics/summary` | Aggregated metrics | Admin only |

**Note:** These endpoints query PostgreSQL for historical data, not in-memory session state.

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

#### 6.1.1 Session Tokens (Chat App WebSocket)

**Mechanism:** JWT (JSON Web Tokens) with HS256 (HMAC-SHA256)

```
Token Payload:
{
    "session_id": "sess-abc123",
    "user_id": "user@example.com",
    "role": "user" | "admin",
    "exp": 1738675200            // Expiration timestamp (1 hour)
}

Signing Secret: Loaded from environment variable (SESSION_SECRET)
```

**Validation:**
- On WebSocket connection establishment
- Can be optionally validated on each critical message (configurable)
- Invalid/expired token → close connection with 403

**Token Lifecycle:**
- Issued on session creation (`POST /api/v1/sessions`)
- Short expiry (1 hour default, configurable)
- No refresh mechanism; user creates new session after expiry

---

#### 6.1.2 API Keys (Programmatic API)

**Mechanism:** UUID v4 with bcrypt hashing

```
Key Format: ocp_bot_1a2b3c4d5e6f7g8h9i0j...  (prefix for identification)

Storage:
- Keys stored as bcrypt hashes in database (never plaintext)
- Bcrypt work factor: 12 (configurable)

Scopes:
- read: Query sessions, tools, analytics
- execute: Submit queries, limited to bulk_approve_low consent mode
- execute:auto_approve: Full execution with bulk_approve_all
```

**Validation:**
- On every API request (Authorization: Bearer header)
- Constant-time comparison to prevent timing attacks
- Failed attempts logged for security monitoring

**Key Management:**
- API keys created via admin interface or CLI
- Support for key rotation (mark old key as deprecated, issue new)
- Keys can be revoked immediately (soft delete in DB)

---

#### 6.1.3 Message Integrity (Optional - If Enabled)

**Mechanism:** HMAC-SHA256 payload signing

```
For sensitive WebSocket messages:
1. Backend generates HMAC signature: HMAC-SHA256(message_payload, session_secret)
2. Sends message with signature: {type, payload, signature}
3. Frontend validates signature (if configured)

Use case: Prevent message tampering in untrusted network environments
```

**Configuration:**
```yaml
security:
  enable_message_signing: false       # Default: disabled for performance
  sign_message_types: ["consent_request", "plan_created"]  # Only sign critical messages
```

---

#### 6.1.4 Rate Limiting

| Type | Limit | Scope | Action on Exceed |
|------|-------|-------|------------------|
| **Session Creation** | 10 sessions/hour | Per user (if authenticated) or per IP | HTTP 429, retry-after header |
| **API Endpoint** | 100 requests/minute | Per API key | HTTP 429, exponential backoff recommended |
| **LLM Calls** | 50 calls/session | Per session | Abort execution, notify user "LLM quota exceeded" |

**Implementation:**
- In-memory token bucket algorithm (Redis for distributed deployments)
- Configurable limits per environment (dev, staging, prod)

---

#### 6.1.5 User Roles

| Role | Permissions |
|------|-------------|
| `user` | Chat access, own session history |
| `admin` | Chat access, Dashboard access, all session history, analytics, user management |

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
| **Active Session Store** | In-memory (Pydantic model / Python dict) | Fast during execution; one session per container |
| **Persistent Database** | SQLite (dev) / PostgreSQL (prod) | SQLite for local dev (zero config), PostgreSQL for ephemeral container production |
| **ORM** | SQLAlchemy | Supports both SQLite and PostgreSQL with same codebase |
| **Containerization** | Docker | Ephemeral containers (1:1 session-to-container mapping) |
| **Orchestration** | Platform-agnostic | Supports Kubernetes, Serverless (Cloud Run/Lambda), or other orchestrators |

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
| **Local/Dev** | SQLite | Single developer, quick setup, file-based, simpler for testing |
| **Production** | PostgreSQL | Ephemeral container deployment, shared historical storage, concurrent writes from multiple containers |

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

### 8.4 Ephemeral Container Architecture (1:1 Session-to-Container)

**Production Deployment** uses ephemeral backend containers with **1:1 mapping per session**.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (Single React App)                      │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │   React App (Chat + Dashboard)                                   │    │
│  │   ├── /chat route      → WebSocket to dedicated container        │    │
│  │   └── /dashboard route → REST API to same dedicated container    │    │
│  └────────┬─────────────────────────────────────────────────────────┘    │
│           │                                                               │
└───────────┼───────────────────────────────────────────────────────────────┘
            │
            │ Session Creation / Query Submission
            │
┌───────────▼───────────────────────────────────────────────────────────────┐
│                    SESSION GATEWAY / ORCHESTRATOR                          │
│  - Spawns ephemeral backend containers (K8s Pods/Jobs or Serverless)     │
│  - Routes requests to appropriate container                               │
│  - Manages container lifecycle (creation, termination)                    │
└───────────┬───────────────────────────────────────────────────────────────┘
            │
            │ Spawns containers on-demand
            │
┌───────────▼───────────────────────────────────────────────────────────────┐
│                    EPHEMERAL BACKEND CONTAINERS                            │
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐           │
│  │ Container A     │  │ Container B     │  │ Container C     │           │
│  │ (UI Session 1)  │  │ (UI Session 2)  │  │ (API Query 1)   │           │
│  │                 │  │                 │  │                 │           │
│  │ • WebSocket     │  │ • WebSocket     │  │ • REST only     │           │
│  │ • REST API      │  │ • REST API      │  │ • Executes      │           │
│  │ • Session State │  │ • Session State │  │   query         │           │
│  │ • Chat + Dash   │  │ • Chat + Dash   │  │                 │           │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘           │
│           │                    │                     │                    │
└───────────┼────────────────────┼─────────────────────┼────────────────────┘
            │                    │                     │
            └────────────────────┴─────────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   PostgreSQL Database    │
                    │   (Shared Historical)    │
                    │                          │
                    │   - Completed sessions   │
                    │   - Feedback records     │
                    │   - Analytics data       │
                    └──────────────────────────┘
```

---

#### 8.4.1 Component Responsibilities

| Component | Purpose | Lifespan |
|-----------|---------|----------|
| **React App** | Single UI with Chat and Dashboard routes | Static (served via CDN/nginx) |
| **Session Gateway** | Spawns/routes to backend containers | Always running |
| **Backend Container** | Handles one session (WebSocket + REST API) | Ephemeral (created per session, terminated on completion) |
| **PostgreSQL** | Persistent historical storage | Always running |

---

#### 8.4.2 Session Lifecycle

**UI Session (Chat + Dashboard):**

```
1. User opens app → POST /api/v1/sessions (to Gateway)
   ↓
2. Gateway spawns Container A (K8s Pod or Serverless instance)
   ↓
3. Gateway returns:
   {
     session_id: "sess-abc123",
     websocket_url: "wss://container-a.cluster/ws/sess-abc123",
     api_base_url: "https://container-a.cluster/api/v1",
     session_token: "jwt-token"
   }
   ↓
4. React App connects:
   - /chat route → WebSocket to wss://container-a.cluster/ws/sess-abc123
   - /dashboard route → REST to https://container-a.cluster/api/v1/sessions, /analytics
   ↓
5. User interacts (chat, views dashboard) → Container A handles everything
   ↓
6. Session ends (completion or timeout):
   - Container A writes final state to PostgreSQL
   - Container A terminates (graceful shutdown)
```

**Programmatic API Session (CI/CD, Scripts):**

```
1. API client → POST /api/v1/queries (to Gateway)
   ↓
2. Gateway spawns Container B
   ↓
3. Gateway returns:
   {
     query_id: "q-xyz789",
     status_url: "https://container-b.cluster/api/v1/queries/q-xyz789"
   }
   ↓
4. API client polls GET https://container-b.cluster/api/v1/queries/q-xyz789
   ↓
5. Container B executes query → Writes results to PostgreSQL → Terminates
```

---

#### 8.4.3 Backend Container Endpoints

**Each ephemeral backend container serves:**

| Endpoint Type | Path | Used By |
|---------------|------|---------|
| **WebSocket** | `/ws/{session_id}` | React App (/chat route) - real-time communication |
| **Dashboard Queries** | `GET /api/v1/sessions`, `/feedback`, `/analytics` | React App (/dashboard route) - historical data |
| **Programmatic API** | `POST /api/v1/queries`, `GET /api/v1/queries/{id}` | CI/CD, scripts - query execution |
| **Utility** | `GET /api/v1/tools`, `/health` | All clients - tool discovery, health checks |

**Note:** `POST /api/v1/sessions` (session creation) is handled by the **Session Gateway**, not the backend containers. The Gateway spawns containers and returns their URLs.

---

#### 8.4.4 Container Orchestration

**The architecture supports multiple orchestration platforms:**

- **Kubernetes / OpenShift**: Each session spawned as Job or Pod
- **Serverless (Cloud Run, Lambda, etc.)**: Each session as serverless instance
- **Decision deferred to deployment time** based on target environment

**Requirements from orchestrator:**
- Ability to spawn ephemeral containers on-demand
- WebSocket support (for chat functionality)
- Container lifecycle management (creation, termination)
- Environment variable injection (session_id, database credentials)

---

#### 8.4.5 Scaling Characteristics

| Metric | Behavior |
|--------|----------|
| **Concurrency** | Linear - 100 sessions = 100 containers |
| **Resource Usage** | Isolated - each container has dedicated CPU/memory |
| **Cost** | Pay-per-session (serverless) or per-pod-time (K8s) |
| **Fault Isolation** | Complete - one container crash doesn't affect others |
| **Startup Time** | Cold start (1-5s K8s, 0.5-2s serverless) |
| **Max Sessions** | Limited by orchestrator capacity (thousands on K8s/serverless) |

---

#### 8.4.6 Session Gateway Responsibilities

The Session Gateway is a lightweight service that:

1. **Creates sessions** - Spawns backend containers
2. **Routes requests** - Directs clients to correct container
3. **Manages lifecycle** - Terminates containers on session end
4. **Health checks** - Monitors container health
5. **Load balancing** - Distributes sessions across cluster nodes (K8s scheduler handles this)

**Implementation:**
- Small always-running service (can be part of API gateway, ingress controller, or custom service)
- Can be implemented as K8s Operator or simple REST service

### 8.5 Dashboard Access Control

| Role | Chat Route (/chat) | Dashboard Route (/dashboard) |
|------|-------------------|------------------------------|
| `user` | ✓ Full access | ✗ No access (redirect to /chat) |
| `admin` | ✓ Full access | ✓ Full access |

**Implementation:**
- Frontend React Router checks user role before rendering `/dashboard` route
- Backend validates role on all dashboard API endpoints (`GET /api/v1/sessions`, `/analytics`, etc.)
- Returns 403 Forbidden if non-admin tries to access dashboard endpoints
- Both chat and dashboard use same backend container and session

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
| **Clients** | React App (/chat route via WSS, /dashboard route via REST), API Clients (REST) |
| **Planner** | Receives intent, generates Plan Steps, interacts with State and LLM |
| **Executor** | Runs steps in loop: Get Step → Consent → Run Tool → Success/Fail |
| **Consent Manager** | Checks consent_mode for each step before execution |
| **Tools** | Execute against External services (GitHub, JIRA, OSV/NVD, Browser, Files) |
| **Feedback Handler** | Collects feedback, flushes State to Database on completion |

**Note:** Single React App with both chat and dashboard functionality; each session uses one dedicated backend container.

**Key Flows:**

1. **Normal Flow**: Client → Planner → Executor Loop → Tools → Success → Feedback → DB
2. **Re-plan Flow**: On failure, loops back to Planner for revised plan
3. **Step Loop**: Executor iterates through plan steps until all complete
4. **State Flush**: On session completion, state is persisted to Database

---

*Document Version: 2.0*  
*Last Updated: Feb 2026*

