# OCP Sustaining Agentic Bot
## Low-Level Design (LLD) Document

---

## 1. Document Overview

This Low-Level Design document provides implementation-ready specifications for the OCP Sustaining Agentic Bot. It covers:
- API endpoint specifications
- Data models (backend Pydantic, frontend TypeScript)
- Component interfaces and classes
- Database schema
- WebSocket protocol
- Configuration schemas
- Error handling patterns

**Architecture Reference:** See `OCP_SUSTAINING_AGENTIC_BOT_ARCHITECTURE.md` for high-level design.

**Assumptions:**
- User authentication handled externally; `user_id` provided in all requests
- Deployment details (K8s, Serverless) deferred; architecture is platform-agnostic
- Focus on core system implementation

---

## 2. Backend Implementation

### 2.1 Technology Stack

```
Backend:
├── Framework: FastAPI 0.109+
├── Language: Python 3.11+
├── Async Runtime: asyncio
├── WebSocket: FastAPI native WebSocket
├── Validation: Pydantic v2
├── Database ORM: SQLAlchemy 2.0+
├── Database Driver: asyncpg (PostgreSQL), aiosqlite (SQLite)
├── LLM SDK: Google GenAI SDK (Gemini), Ollama API (local models)
└── MCP SDK: Model Context Protocol Python SDK
```

---

### 2.2 Project Structure

```
backend/
├── main.py                      # FastAPI app entry point
├── config/
│   ├── settings.py              # Pydantic settings (env vars)
│   ├── tool_registry.yaml       # Tool definitions
│   ├── llm_config.yaml          # LLM provider config
│   └── mcp_servers.yaml         # MCP server config
├── models/
│   ├── session.py               # SessionState, Plan, PlanStep
│   ├── consent.py               # ConsentRequest, ConsentResponse
│   ├── tool.py                  # Tool, Task, ToolResult
│   ├── feedback.py              # Feedback models
│   └── db/                      # SQLAlchemy models
│       ├── session.py
│       ├── feedback.py
│       └── user.py
├── core/
│   ├── orchestrator/
│   │   ├── planner.py           # Planner module
│   │   ├── executor.py          # Executor module
│   │   ├── consent_manager.py  # Consent Manager
│   │   └── response_moderator.py # Response Moderator (LLM-based)
│   ├── tool_registry.py         # Tool Registry
│   ├── state_manager.py         # Central State Manager
│   └── mcp_client.py            # MCP server integration
├── api/
│   ├── websocket.py             # WebSocket endpoints
│   ├── sessions.py              # Session management endpoints
│   ├── queries.py               # Programmatic API endpoints
│   ├── dashboard.py             # Dashboard REST endpoints
│   └── tools.py                 # Tool listing endpoints
├── tools/
│   ├── base.py                  # Base Tool class
│   ├── internal/                # Internal tools
│   │   ├── analyze_gomod.py
│   │   └── ...
│   └── external/                # External tools
│       ├── fetch_cve_data.py
│       ├── github_pr.py
│       └── ...
├── llm/
│   ├── provider.py              # LLM provider abstraction
│   ├── gemini_provider.py
│   └── ollama_provider.py
├── db/
│   ├── database.py              # Database connection
│   ├── repository.py            # Database operations
│   └── migrations/              # Alembic migrations
└── utils/
    ├── logging.py               # Logging configuration
    ├── auth.py                  # JWT validation
    └── errors.py                # Custom exceptions
```

---

## 3. Data Models

### 3.1 Backend Models (Pydantic)

#### 3.1.1 Session Models

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

class ExecutionStatus(str, Enum):
    IDLE = "idle"
    PLANNING = "planning"
    EXECUTING = "executing"
    WAITING_CONSENT = "waiting_consent"
    COMPLETED = "completed"
    FAILED = "failed"
    TIMEOUT = "timeout"

class Message(BaseModel):
    message_id: str
    role: str  # "user" | "assistant" | "system"
    content: str
    timestamp: datetime
    metadata: Optional[Dict[str, Any]] = None

class PlanStep(BaseModel):
    step_id: str
    order: int
    tool_name: str
    tool_description: str
    parameters: Dict[str, Any]
    expected_output: str
    risk_level: str  # "low" | "medium" | "high"
    requires_individual_consent: bool
    dependencies: List[str] = Field(default_factory=list)
    status: str = "pending"  # "pending" | "approved" | "rejected" | "executing" | "completed" | "failed" | "skipped"

class Plan(BaseModel):
    plan_id: str
    user_prompt: str
    thinking_process: List[str] = Field(default_factory=list)
    steps: List[PlanStep]
    estimated_duration: str
    requires_consent: bool
    created_at: datetime

class ConsentRequest(BaseModel):
    request_id: str
    plan_id: str
    step_id: str
    tool_name: str
    tool_description: str
    parameters_summary: str
    risk_level: str
    reversible: bool
    timeout_seconds: int = 300
    bulk_options: Dict[str, bool] = Field(default_factory=lambda: {
        "approve_all_risks": True,
        "approve_low_risks": True,
        "approve_low_medium_risks": True
    })

class ConsentResponse(BaseModel):
    request_id: str
    decision: str  # "approved" | "rejected" | "modified" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium"
    modified_parameters: Optional[Dict[str, Any]] = None
    user_feedback: Optional[str] = None

class SessionData(BaseModel):
    """Central data store - tool I/O"""

    # ========================================
    # CVE Remediation - Go
    # ========================================
    # Inputs
    cve_id: Optional[str] = None
    repo_url: Optional[str] = None
    repo_version: Optional[str] = None
    cloned_repo_path: Optional[str] = None
    are_inputs_valid: Optional[bool] = None

    # CVE Analysis
    cve_details: Optional[Dict[str, Any]] = None
    affected_packages: Optional[List[Dict[str, Any]]] = None
    is_valid_go_cve: Optional[bool] = None
    go_cve_id: Optional[str] = None
    cvss_score: Optional[float] = None
    cvss_severity: Optional[str] = None
    is_vulnerable: Optional[bool] = None
    is_false_alarm: Optional[bool] = None
    analysis_incomplete: Optional[bool] = None
    gomod_analysis: Optional[Dict[str, Any]] = None
    code_chunks: Optional[List[Dict[str, Any]]] = None

    # Remediation
    is_fix_available: Optional[bool] = None
    fix_version: Optional[str] = None
    go_mod_updated: Optional[bool] = None
    go_runtime_version: Optional[str] = None
    version_changes: Optional[List[Dict[str, Any]]] = None

    # Test Results
    build_success: Optional[bool] = None
    test_success: Optional[bool] = None
    is_test_successful: Optional[bool] = None
    is_test_skipped: Optional[bool] = None
    test_skip_reason: Optional[str] = None
    makefile_exists: Optional[bool] = None
    last_failure_step: Optional[str] = None
    last_failure_logs: Optional[str] = None
    test_logs: Optional[str] = None

    # Failure Analysis
    failure_analysis_response: Optional[str] = None

    # Output
    pr_url: Optional[str] = None
    summary: Optional[str] = None
    final_output: Optional[str] = None
    generated_summary: Optional[str] = None

    # ========================================
    # Documentation - Confluence
    # ========================================
    confluence_page_url: Optional[str] = None
    confluence_page_title: Optional[str] = None
    confluence_page_content: Optional[str] = None
    confluence_page_metadata: Optional[Dict[str, Any]] = None
    explanation_summary: Optional[str] = None
    key_concepts: Optional[List[str]] = None
    action_items: Optional[List[str]] = None
    qa_response: Optional[str] = None
    user_question: Optional[str] = None

    # ========================================
    # Project Management - Jira
    # ========================================
    jira_issue_key: Optional[str] = None
    jira_issue_summary: Optional[str] = None
    jira_issue_description: Optional[str] = None
    jira_issue_status: Optional[str] = None
    jira_issue_assignee: Optional[str] = None
    jira_issue_comments: Optional[List[Dict[str, Any]]] = None
    jira_issue_metadata: Optional[Dict[str, Any]] = None
    jql_query: Optional[str] = None
    jira_query_results: Optional[List[Dict[str, Any]]] = None
    matching_issue_keys: Optional[List[str]] = None
    issue_count: Optional[int] = None

    # ========================================
    # Quality Engineering
    # ========================================
    test_failure_logs: Optional[str] = None
    failure_root_cause: Optional[str] = None
    suggested_fixes: Optional[List[str]] = None
    related_code_snippets: Optional[List[Dict[str, Any]]] = None
    error_message: Optional[str] = None
    search_query: Optional[str] = None
    similar_issues: Optional[List[Dict[str, Any]]] = None
    related_prs: Optional[List[Dict[str, Any]]] = None
    known_workarounds: Optional[List[str]] = None

    # ========================================
    # Common/System
    # ========================================
    # Error Tracking
    last_error: Optional[str] = None
    last_error_step: Optional[str] = None
    llm_invoke_error: Optional[str] = None
    step_retry_counts: Dict[str, int] = Field(default_factory=dict)

class FeedbackData(BaseModel):
    step_ratings: Dict[str, str] = Field(default_factory=dict)  # step_id -> "positive"|"negative"
    session_rating: Optional[int] = None  # 1-5 stars
    comments: List[str] = Field(default_factory=list)

class SessionState(BaseModel):
    # Session Identity
    session_id: str
    user_id: Optional[str] = None
    created_at: datetime
    last_activity: datetime
    idle_timeout_ms: int = 300000

    # Conversation
    messages: List[Message] = Field(default_factory=list)

    # Execution Context
    current_plan: Optional[Plan] = None
    execution_status: ExecutionStatus = ExecutionStatus.IDLE
    pending_consent: Optional[ConsentRequest] = None
    current_step_index: int = 0

    # Central Data Store
    data: SessionData = Field(default_factory=SessionData)

    # Tool Execution History
    tool_results: Dict[str, Dict[str, Any]] = Field(default_factory=dict)

    # User Preferences
    consent_mode: str = "step_by_step"

    # Session Metrics
    total_tools_executed: int = 0
    total_approvals: int = 0
    total_rejections: int = 0

    # Feedback
    feedback: FeedbackData = Field(default_factory=FeedbackData)

    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }
```

#### 3.1.2 Tool Models

```python
from typing import List, Dict, Any, Optional, Callable
from pydantic import BaseModel, Field
from enum import Enum

class TaskType(str, Enum):
    API_CALL = "api_call"
    MCP_CALL = "mcp_call"
    LLM_CALL = "llm_call"
    EVALUATE = "evaluate"
    STORE = "store"

class Task(BaseModel):
    task_id: str
    type: TaskType
    description: str
    target: str  # API endpoint, MCP server, LLM model, etc.
    dependencies: List[str] = Field(default_factory=list)
    parameters: Dict[str, Any] = Field(default_factory=dict)

class ToolResult(BaseModel):
    success: bool
    error: Optional[str] = None
    execution_time_ms: int
    tasks_completed: List[str] = Field(default_factory=list)
    output_data: Optional[Dict[str, Any]] = None

class ToolDefinition(BaseModel):
    # Identity
    name: str
    display_name: str
    description: str
    category: str  # "internal" | "external"
    capabilities: List[str] = Field(default_factory=list)

    # State Dependencies
    state_inputs: List[str] = Field(default_factory=list)
    state_outputs: List[str] = Field(default_factory=list)

    # Behavior
    requires_consent: bool = True
    risk_level: str = "medium"  # "low" | "medium" | "high"
    estimated_duration: str = "unknown"

    # Retry Policy
    is_retriable: bool = False
    max_retries: int = 0
    retry_delay_ms: int = 1000

    # Tasks
    tasks: List[Task] = Field(default_factory=list)

class MCPServerConfig(BaseModel):
    transport: str  # "stdio" | "http"
    stdio: Optional[Dict[str, Any]] = None
    http: Optional[Dict[str, Any]] = None
    available_tools: List[Dict[str, Any]] = Field(default_factory=list)

class MCPToolDefinition(ToolDefinition):
    mcp_config: MCPServerConfig
```

#### 3.1.3 API Request/Response Models

```python
from pydantic import BaseModel
from typing import Optional, Dict, Any, List
from datetime import datetime

# Session Creation
class CreateSessionRequest(BaseModel):
    user_id: str  # Provided by auth layer
    consent_mode: str = "step_by_step"

class CreateSessionResponse(BaseModel):
    session_id: str
    container_id: str
    websocket_url: str
    api_base_url: str
    session_token: str
    expires_at: datetime
    idle_timeout_ms: int

# Programmatic Query API
class CreateQueryRequest(BaseModel):
    user_id: str
    prompt: str
    consent_mode: str = "bulk_approve_all"
    callback_url: Optional[str] = None
    timeout_seconds: int = 600

class CreateQueryResponse(BaseModel):
    query_id: str
    status: str
    poll_url: str

class QueryStatusResponse(BaseModel):
    query_id: str
    status: str  # "queued" | "running" | "completed" | "failed"
    progress: Optional[Dict[str, Any]] = None

class QueryResultResponse(BaseModel):
    query_id: str
    status: str
    result: Optional[Dict[str, Any]] = None
    execution_log: List[Dict[str, Any]] = Field(default_factory=list)

# Dashboard API
class SessionListRequest(BaseModel):
    user_id: str  # For role validation
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None
    status_filter: Optional[str] = None
    limit: int = 50
    offset: int = 0

class SessionListResponse(BaseModel):
    sessions: List[Dict[str, Any]]
    total_count: int
    has_more: bool

class AnalyticsSummaryResponse(BaseModel):
    total_sessions: int
    success_rate: float
    avg_rating: float
    avg_duration_seconds: float
    tool_performance: Dict[str, Dict[str, Any]]
```

---

### 3.2 WebSocket Message Protocol

#### 3.2.1 Client → Server Messages

```python
from pydantic import BaseModel
from typing import Optional, Dict, Any

class WSMessage(BaseModel):
    type: str
    payload: Dict[str, Any]

# Message Types:
# 1. chat
class ChatMessagePayload(BaseModel):
    content: str

# 2. consent_response
# Uses ConsentResponse model

# 3. feedback_submit
class FeedbackSubmitPayload(BaseModel):
    step_id: Optional[str] = None
    rating: Optional[int] = None  # 1-5
    sentiment: Optional[str] = None  # "positive" | "negative"
    comment: Optional[str] = None

# 4. cancel_execution
class CancelExecutionPayload(BaseModel):
    plan_id: str

# 5. pong
# Empty payload
```

#### 3.2.2 Server → Client Messages

```python
# Message Types:
# 1. thinking_update
class ThinkingUpdatePayload(BaseModel):
    step: str

# 2. plan_created
class PlanCreatedPayload(BaseModel):
    plan: Plan

# 3. consent_request
# Uses ConsentRequest model

# 4. consent_timeout
class ConsentTimeoutPayload(BaseModel):
    request_id: str
    message: str

# 5. step_started
# Triggers spinner display in chat with step description
class StepStartedPayload(BaseModel):
    step_id: str
    tool_name: str
    description: str  # Human-readable step description

# 6. step_progress
class StepProgressPayload(BaseModel):
    step_id: str
    progress: str

# 7. assistant_message
# LLM-moderated response (replaces spinner when step completes)
class AssistantMessagePayload(BaseModel):
    content: str  # Markdown-formatted message
    metadata: Optional[Dict[str, Any]] = None  # step_id, tool_name, final_response flag

# 8. step_completed
# Internal event - not directly shown to user (assistant_message is sent instead)
class StepCompletedPayload(BaseModel):
    step_id: str
    success: bool
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None

# 9. plan_completed
class PlanCompletedPayload(BaseModel):
    plan_id: str
    summary: str

# 10. session_timeout
class SessionTimeoutPayload(BaseModel):
    message: str

# 11. error
class ErrorPayload(BaseModel):
    message: str
    recoverable: bool

# 12. ping
# Empty payload - heartbeat to keep connection alive
```

---

### 3.3 Database Schema (SQLAlchemy)

```python
from sqlalchemy import Column, String, Integer, Float, DateTime, Text, Boolean, JSON, ForeignKey, Index
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime
import uuid

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    user_id = Column(String(255), primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    role = Column(String(50), nullable=False, default="user")  # "user" | "admin"
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    sessions = relationship("Session", back_populates="user")

    __table_args__ = (
        Index("idx_users_email", "email"),
        Index("idx_users_role", "role"),
    )

class Session(Base):
    __tablename__ = "sessions"

    session_id = Column(String(255), primary_key=True)
    user_id = Column(String(255), ForeignKey("users.user_id"), nullable=True)

    # Metadata
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    completed_at = Column(DateTime, nullable=True)
    duration_seconds = Column(Integer, nullable=True)

    # Status
    status = Column(String(50), nullable=False)  # "completed" | "failed" | "timeout"

    # Prompt & Summary
    user_prompt = Column(Text, nullable=True)
    summary = Column(Text, nullable=True)

    # Metrics
    total_tools_executed = Column(Integer, default=0)
    total_approvals = Column(Integer, default=0)
    total_rejections = Column(Integer, default=0)

    # Final State (JSON)
    final_data = Column(JSON, nullable=True)

    # Relationships
    user = relationship("User", back_populates="sessions")
    messages = relationship("SessionMessage", back_populates="session", cascade="all, delete-orphan")
    steps = relationship("SessionStep", back_populates="session", cascade="all, delete-orphan")
    feedbacks = relationship("Feedback", back_populates="session", cascade="all, delete-orphan")

    __table_args__ = (
        Index("idx_sessions_user_id", "user_id"),
        Index("idx_sessions_created_at", "created_at"),
        Index("idx_sessions_status", "status"),
    )

class SessionMessage(Base):
    __tablename__ = "session_messages"

    message_id = Column(String(255), primary_key=True, default=lambda: str(uuid.uuid4()))
    session_id = Column(String(255), ForeignKey("sessions.session_id"), nullable=False)

    role = Column(String(50), nullable=False)  # "user" | "assistant" | "system"
    content = Column(Text, nullable=False)
    timestamp = Column(DateTime, default=datetime.utcnow, nullable=False)
    metadata = Column(JSON, nullable=True)

    # Relationships
    session = relationship("Session", back_populates="messages")

    __table_args__ = (
        Index("idx_messages_session_id", "session_id"),
        Index("idx_messages_timestamp", "timestamp"),
    )

class SessionStep(Base):
    __tablename__ = "session_steps"

    step_id = Column(String(255), primary_key=True)
    session_id = Column(String(255), ForeignKey("sessions.session_id"), nullable=False)

    order = Column(Integer, nullable=False)
    tool_name = Column(String(255), nullable=False)
    tool_description = Column(Text, nullable=True)

    # Execution
    status = Column(String(50), nullable=False)  # "pending" | "executing" | "completed" | "failed" | "skipped"
    started_at = Column(DateTime, nullable=True)
    completed_at = Column(DateTime, nullable=True)
    duration_ms = Column(Integer, nullable=True)

    # Results
    success = Column(Boolean, nullable=True)
    error_message = Column(Text, nullable=True)
    retry_count = Column(Integer, default=0)

    # Data
    parameters = Column(JSON, nullable=True)
    result_data = Column(JSON, nullable=True)

    # Relationships
    session = relationship("Session", back_populates="steps")

    __table_args__ = (
        Index("idx_steps_session_id", "session_id"),
        Index("idx_steps_tool_name", "tool_name"),
        Index("idx_steps_status", "status"),
    )

class Feedback(Base):
    __tablename__ = "feedback"

    feedback_id = Column(String(255), primary_key=True, default=lambda: str(uuid.uuid4()))
    session_id = Column(String(255), ForeignKey("sessions.session_id"), nullable=False)

    context = Column(String(50), nullable=False)  # "tool_result" | "plan_complete" | "error"
    related_id = Column(String(255), nullable=True)  # step_id, plan_id, etc.

    # Feedback Content
    rating = Column(Integer, nullable=True)  # 1-5 for plan completion
    sentiment = Column(String(50), nullable=True)  # "positive" | "negative" for tool results
    free_text = Column(Text, nullable=True)

    # Auto-captured Context
    tool_name = Column(String(255), nullable=True)
    error_message = Column(Text, nullable=True)

    timestamp = Column(DateTime, default=datetime.utcnow, nullable=False)

    # Relationships
    session = relationship("Session", back_populates="feedbacks")

    __table_args__ = (
        Index("idx_feedback_session_id", "session_id"),
        Index("idx_feedback_context", "context"),
        Index("idx_feedback_rating", "rating"),
        Index("idx_feedback_sentiment", "sentiment"),
    )

class APIKey(Base):
    __tablename__ = "api_keys"

    key_id = Column(String(255), primary_key=True, default=lambda: str(uuid.uuid4()))
    key_hash = Column(String(255), nullable=False, unique=True)  # bcrypt hash
    key_prefix = Column(String(50), nullable=False)  # "ocp_bot_xxxx" for identification

    user_id = Column(String(255), ForeignKey("users.user_id"), nullable=False)
    scopes = Column(JSON, nullable=False)  # ["read", "execute", "execute:auto_approve"]

    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    last_used_at = Column(DateTime, nullable=True)
    expires_at = Column(DateTime, nullable=True)

    is_active = Column(Boolean, default=True, nullable=False)

    __table_args__ = (
        Index("idx_apikeys_key_hash", "key_hash"),
        Index("idx_apikeys_user_id", "user_id"),
        Index("idx_apikeys_is_active", "is_active"),
    )
```

---

## 4. Core Components Implementation

### 4.1 Planner Module

```python
# core/orchestrator/planner.py

from typing import List, Dict, Any, Optional
from models.session import Plan, PlanStep, SessionState
from llm.provider import LLMProvider
from core.tool_registry import ToolRegistry
import uuid
from datetime import datetime

class Planner:
    def __init__(self, llm_provider: LLMProvider, tool_registry: ToolRegistry):
        self.llm = llm_provider
        self.tool_registry = tool_registry

    async def create_plan(
        self,
        user_prompt: str,
        state: SessionState,
        thinking_callback: Optional[callable] = None
    ) -> Plan:
        """
        Generate execution plan using LLM.

        Args:
            user_prompt: User's natural language request
            state: Current session state
            thinking_callback: Optional callback for streaming thinking updates

        Returns:
            Plan with steps and dependencies
        """
        # Get all available tools
        all_tools = self.tool_registry.get_all_tools()

        # Build LLM prompt with tool descriptions
        system_prompt = self._build_system_prompt(all_tools)
        user_message = self._build_user_message(user_prompt, state)

        # Stream thinking process
        thinking_steps = []

        async def thinking_handler(step: str):
            thinking_steps.append(step)
            if thinking_callback:
                await thinking_callback(step)

        # Call LLM with function calling for structured output
        llm_response = await self.llm.generate_plan(
            system_prompt=system_prompt,
            user_message=user_message,
            tools_schema=self._get_tools_schema(all_tools),
            thinking_callback=thinking_handler
        )

        # Parse LLM response into Plan
        plan = self._parse_llm_response(
            llm_response=llm_response,
            user_prompt=user_prompt,
            thinking_steps=thinking_steps
        )

        return plan

    async def generate_recovery_plan(
        self,
        error: str,
        failed_step: PlanStep,
        completed_steps: List[PlanStep],
        current_state: Dict[str, Any],
        remaining_steps: List[PlanStep]
    ) -> Dict[str, Any]:
        """
        Generate recovery plan after step failure.

        Returns:
            RecoveryPlan dict with strategy and actions
        """
        recovery_prompt = self._build_recovery_prompt(
            error=error,
            failed_step=failed_step,
            completed_steps=completed_steps,
            current_state=current_state,
            remaining_steps=remaining_steps
        )

        recovery_plan = await self.llm.generate_recovery_plan(recovery_prompt)

        return recovery_plan

    def _build_system_prompt(self, tools: List[Dict[str, Any]]) -> str:
        """Build system prompt with tool descriptions."""
        tool_descriptions = "\n".join([
            f"- {tool['name']}: {tool['description']} (Capabilities: {', '.join(tool['capabilities'])})"
            for tool in tools
        ])

        return f"""You are an AI assistant that creates execution plans for software engineering tasks.

Available Tools:
{tool_descriptions}

Your task:
1. Analyze the user's request
2. Select relevant tools from the available list
3. Create a step-by-step execution plan with clear dependencies
4. Ensure each step has the required inputs from previous steps or initial state

Output format: Use the create_plan function with structured steps.
"""

    def _build_user_message(self, user_prompt: str, state: SessionState) -> str:
        """Build user message with context."""
        state_context = f"Current state: {state.data.dict()}" if state.data else "No current state"
        return f"""User Request: {user_prompt}

{state_context}

Please create an execution plan."""

    def _get_tools_schema(self, tools: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Convert tools to JSON schema for LLM function calling."""
        # Implementation depends on LLM provider
        # Returns standard function calling schema (compatible with Gemini function calling)
        pass

    def _parse_llm_response(
        self,
        llm_response: Dict[str, Any],
        user_prompt: str,
        thinking_steps: List[str]
    ) -> Plan:
        """Parse LLM response into Plan model."""
        steps = []
        for idx, step_data in enumerate(llm_response.get("steps", [])):
            step = PlanStep(
                step_id=str(uuid.uuid4()),
                order=idx,
                tool_name=step_data["tool_name"],
                tool_description=step_data.get("description", ""),
                parameters=step_data.get("parameters", {}),
                expected_output=step_data.get("expected_output", ""),
                risk_level=step_data.get("risk_level", "medium"),
                requires_individual_consent=step_data.get("requires_consent", True),
                dependencies=step_data.get("dependencies", []),
                status="pending"
            )
            steps.append(step)

        return Plan(
            plan_id=str(uuid.uuid4()),
            user_prompt=user_prompt,
            thinking_process=thinking_steps,
            steps=steps,
            estimated_duration=llm_response.get("estimated_duration", "unknown"),
            requires_consent=any(s.requires_individual_consent for s in steps),
            created_at=datetime.utcnow()
        )

    def _build_recovery_prompt(
        self,
        error: str,
        failed_step: PlanStep,
        completed_steps: List[PlanStep],
        current_state: Dict[str, Any],
        remaining_steps: List[PlanStep]
    ) -> str:
        """Build prompt for recovery plan generation."""
        return f"""A step has failed during execution. Please suggest a recovery strategy.

Failed Step: {failed_step.tool_name}
Error: {error}

Completed Steps: {[s.tool_name for s in completed_steps]}
Remaining Steps: {[s.tool_name for s in remaining_steps]}
Current State: {current_state}

Recovery Options:
1. retry - Retry the same step with modified parameters
2. skip - Skip this step and continue with remaining steps
3. abort - Stop execution and report failure
4. replan - Generate new steps to replace remaining plan

Choose the best recovery strategy and provide details."""
```

### 4.2 Executor Module

```python
# core/orchestrator/executor.py

from typing import Optional, Callable
from models.session import Plan, PlanStep, SessionState, ToolResult
from core.tool_registry import ToolRegistry
from core.orchestrator.consent_manager import ConsentManager
from core.orchestrator.planner import Planner
import asyncio
from datetime import datetime

class Executor:
    def __init__(
        self,
        tool_registry: ToolRegistry,
        consent_manager: ConsentManager,
        planner: Planner
    ):
        self.tool_registry = tool_registry
        self.consent_manager = consent_manager
        self.planner = planner

    async def execute_plan(
        self,
        plan: Plan,
        state: SessionState,
        progress_callback: Optional[Callable] = None
    ):
        """
        Execute plan steps with consent, retry, and recovery logic.

        Args:
            plan: Execution plan to run
            state: Session state (modified in-place)
            progress_callback: Callback for progress updates
        """
        state.current_plan = plan
        state.execution_status = "executing"

        # Group steps by dependencies for parallel execution
        step_groups = self._group_steps_by_dependencies(plan.steps)

        for group in step_groups:
            # Execute steps in this group in parallel
            tasks = [
                self._execute_step(step, state, progress_callback)
                for step in group
            ]

            results = await asyncio.gather(*tasks, return_exceptions=True)

            # Check for failures
            for step, result in zip(group, results):
                if isinstance(result, Exception) or (isinstance(result, dict) and not result.get("success")):
                    # Handle failure
                    await self._handle_step_failure(step, state, result, progress_callback)

        state.execution_status = "completed"
        state.current_plan = None

    async def _execute_step(
        self,
        step: PlanStep,
        state: SessionState,
        progress_callback: Optional[Callable] = None
    ) -> Dict[str, Any]:
        """Execute a single plan step."""
        step.status = "executing"

        if progress_callback:
            await progress_callback("step_started", {
                "step_id": step.step_id,
                "tool_name": step.tool_name,
                "description": step.description
            })

        # Get tool from registry
        tool = self.tool_registry.get_by_name(step.tool_name)
        if not tool:
            return {"success": False, "error": f"Tool {step.tool_name} not found"}

        # Validate state inputs
        missing_inputs = self._validate_state_inputs(tool, state)
        if missing_inputs:
            step.status = "skipped"
            return {"success": False, "error": f"Missing required inputs: {missing_inputs}", "skipped": True}

        # Check consent
        consent_granted = await self._check_consent(step, tool, state)
        if not consent_granted:
            step.status = "rejected"
            state.total_rejections += 1
            return {"success": False, "error": "Consent denied"}

        state.total_approvals += 1

        # Execute tool with retry logic
        retry_count = state.data.step_retry_counts.get(step.step_id, 0)

        while True:
            try:
                result = await self._run_tool(tool, step, state, progress_callback)

                if result.success:
                    step.status = "completed"
                    state.total_tools_executed += 1
                    state.tool_results[step.step_id] = result.dict()

                    if progress_callback:
                        await progress_callback("step_completed", {
                            "step_id": step.step_id,
                            "success": True,
                            "result": result.output_data
                        })

                    return result.dict()
                else:
                    # Check if retriable
                    if tool.is_retriable and retry_count < tool.max_retries:
                        retry_count += 1
                        state.data.step_retry_counts[step.step_id] = retry_count
                        state.data.last_error = result.error
                        state.data.last_error_step = step.step_id

                        if progress_callback:
                            await progress_callback("step_progress", {
                                "step_id": step.step_id,
                                "progress": f"Retry {retry_count}/{tool.max_retries} after {tool.retry_delay_ms}ms"
                            })

                        await asyncio.sleep(tool.retry_delay_ms / 1000)
                        continue
                    else:
                        # No more retries
                        step.status = "failed"
                        state.data.last_error = result.error
                        state.data.last_error_step = step.step_id

                        if progress_callback:
                            await progress_callback("step_completed", {
                                "step_id": step.step_id,
                                "success": False,
                                "error": result.error
                            })

                        return result.dict()

            except Exception as e:
                step.status = "failed"
                return {"success": False, "error": str(e)}

    async def _run_tool(
        self,
        tool: "Tool",
        step: PlanStep,
        state: SessionState,
        progress_callback: Optional[Callable] = None
    ) -> ToolResult:
        """Run tool execution."""
        start_time = datetime.utcnow()

        # Execute tool (tool reads from and writes to state)
        result = await tool.execute(state, step.parameters)

        execution_time_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)
        result.execution_time_ms = execution_time_ms

        return result

    def _validate_state_inputs(self, tool: "Tool", state: SessionState) -> List[str]:
        """Check if required state inputs are available."""
        missing = []
        for input_field in tool.state_inputs:
            if not getattr(state.data, input_field, None):
                missing.append(input_field)
        return missing

    async def _check_consent(
        self,
        step: PlanStep,
        tool: "Tool",
        state: SessionState
    ) -> bool:
        """Check if consent is granted for this step."""
        if not tool.requires_consent:
            return True

        return await self.consent_manager.request_consent(
            step=step,
            tool=tool,
            state=state
        )

    async def _handle_step_failure(
        self,
        step: PlanStep,
        state: SessionState,
        result: Any,
        progress_callback: Optional[Callable] = None
    ):
        """Handle step failure with LLM-driven recovery."""
        error = result.get("error") if isinstance(result, dict) else str(result)

        # If step was skipped due to missing inputs, don't trigger recovery
        if isinstance(result, dict) and result.get("skipped"):
            return

        # Ask Planner for recovery plan
        recovery_plan = await self.planner.generate_recovery_plan(
            error=error,
            failed_step=step,
            completed_steps=[s for s in state.current_plan.steps if s.status == "completed"],
            current_state=state.data.dict(),
            remaining_steps=[s for s in state.current_plan.steps if s.status == "pending"]
        )

        # Execute recovery strategy
        strategy = recovery_plan.get("strategy")

        if strategy == "retry":
            # Modify step parameters and retry
            step.parameters = recovery_plan.get("modified_parameters", step.parameters)
            await self._execute_step(step, state, progress_callback)

        elif strategy == "skip":
            step.status = "skipped"
            if progress_callback:
                await progress_callback("step_completed", {
                    "step_id": step.step_id,
                    "success": False,
                    "error": f"Skipped: {recovery_plan.get('explanation')}"
                })

        elif strategy == "replan":
            # Insert new steps (simplified - actual implementation more complex)
            new_steps = recovery_plan.get("new_steps", [])
            # Add new steps to plan...

        elif strategy == "abort":
            state.execution_status = "failed"
            if progress_callback:
                await progress_callback("error", {
                    "message": recovery_plan.get("user_notification"),
                    "recoverable": False
                })

    def _group_steps_by_dependencies(self, steps: List[PlanStep]) -> List[List[PlanStep]]:
        """Group steps into parallel execution groups based on dependencies."""
        groups = []
        completed = set()
        remaining = steps.copy()

        while remaining:
            # Find steps with no pending dependencies
            ready = [
                step for step in remaining
                if all(dep in completed for dep in step.dependencies)
            ]

            if not ready:
                # Circular dependency or error
                break

            groups.append(ready)
            for step in ready:
                completed.add(step.step_id)
                remaining.remove(step)

        return groups
```

### 4.3 Consent Manager

```python
# core/orchestrator/consent_manager.py

from models.session import PlanStep, ConsentRequest, ConsentResponse, SessionState
from core.tool_registry import ToolRegistry
import asyncio
import uuid
from datetime import datetime, timedelta

class ConsentManager:
    def __init__(self):
        self.pending_consents: dict = {}  # request_id -> asyncio.Future

    async def request_consent(
        self,
        step: PlanStep,
        tool: "Tool",
        state: SessionState
    ) -> bool:
        """
        Request user consent for tool execution.

        Returns:
            bool: True if consent granted, False if denied
        """
        # Check consent mode
        consent_mode = state.consent_mode
        risk_level = tool.risk_level

        # Auto-approve based on consent mode
        if consent_mode == "skip_confirmation":
            return True

        if consent_mode == "bulk_approve_all":
            return True

        if consent_mode == "bulk_approve_low" and risk_level == "low":
            return True

        if consent_mode == "bulk_approve_low_medium" and risk_level in ["low", "medium"]:
            return True

        # Need explicit consent
        request = ConsentRequest(
            request_id=str(uuid.uuid4()),
            plan_id=state.current_plan.plan_id,
            step_id=step.step_id,
            tool_name=tool.name,
            tool_description=tool.description,
            parameters_summary=self._summarize_parameters(step.parameters),
            risk_level=risk_level,
            reversible=tool.category == "internal",  # Heuristic
            timeout_seconds=300
        )

        # Store request in state
        state.pending_consent = request

        # Create future for response
        future = asyncio.Future()
        self.pending_consents[request.request_id] = future

        # Wait for response with timeout
        try:
            response = await asyncio.wait_for(
                future,
                timeout=request.timeout_seconds
            )

            # Clear pending consent
            state.pending_consent = None

            if response.decision == "approved":
                return True
            elif response.decision == "modified":
                # Update step parameters
                step.parameters.update(response.modified_parameters or {})
                return True
            elif response.decision.startswith("bulk_approve"):
                # Update consent mode
                if response.decision == "bulk_approve_all":
                    state.consent_mode = "bulk_approve_all"
                elif response.decision == "bulk_approve_low":
                    state.consent_mode = "bulk_approve_low"
                elif response.decision == "bulk_approve_low_medium":
                    state.consent_mode = "bulk_approve_low_medium"
                return True
            else:
                return False

        except asyncio.TimeoutError:
            # Consent timeout
            state.pending_consent = None
            state.execution_status = "timeout"
            return False

    def submit_consent_response(self, response: ConsentResponse):
        """Submit consent response from user."""
        future = self.pending_consents.get(response.request_id)
        if future and not future.done():
            future.set_result(response)

    def _summarize_parameters(self, parameters: dict) -> str:
        """Create human-readable parameter summary."""
        if not parameters:
            return "No parameters"

        summary_parts = []
        for key, value in parameters.items():
            if isinstance(value, (str, int, float, bool)):
                summary_parts.append(f"{key}: {value}")
            else:
                summary_parts.append(f"{key}: <complex>")

        return ", ".join(summary_parts)
```

---

### 4.4 Response Moderator

**Purpose:** Converts raw tool execution results into human-readable chat messages using LLM. Operates at two levels:
1. **Step-level**: After each tool execution, generates contextual progress updates
2. **Plan-level**: After plan completion, generates comprehensive final answer to user's question

```python
# core/orchestrator/response_moderator.py

from typing import Dict, Any, List, Optional
from models.session import PlanStep, Plan, SessionState, ToolResult
from llm.manager import LLMProviderManager
import logging

logger = logging.getLogger(__name__)

class ResponseModerator:
    """
    LLM-based response moderator that generates human-readable messages
    from tool execution results.
    """

    def __init__(self, llm_manager: LLMProviderManager):
        self.llm_manager = llm_manager

    async def moderate_step_response(
        self,
        step: PlanStep,
        tool_result: ToolResult,
        user_question: str,
        state: SessionState
    ) -> str:
        """
        Generate human-readable message for a step execution result.

        Args:
            step: The executed plan step
            tool_result: Result from tool execution
            user_question: Original user question
            state: Current session state

        Returns:
            Human-readable message to send to chat
        """
        system_prompt = """You are a helpful assistant responding to intermediate progress updates.
Given the user's question and the result of a tool execution, generate a brief,
conversational message explaining what was accomplished in this step.

Guidelines:
- Be concise (1-3 sentences)
- Focus on what was discovered or accomplished
- Use natural language, not technical jargon
- If the step failed, explain what went wrong simply
- Don't repeat information from previous steps
- Don't mention internal tool names or technical details
"""

        user_message = f"""User's Question: {user_question}

Tool Executed: {step.tool_name}
Step Description: {step.description}

Result:
{self._format_tool_result(tool_result)}

Generate a brief conversational message explaining this step's result."""

        try:
            llm_provider = self.llm_manager.get_provider(use_case="tool_default")
            response = await llm_provider.generate_text(
                system_prompt=system_prompt,
                user_message=user_message,
                temperature=0.4
            )
            return response.strip()

        except Exception as e:
            logger.error(f"Response moderation failed: {e}")
            # Fallback to basic message
            if tool_result.success:
                return f"✓ {step.description} completed."
            else:
                return f"✗ {step.description} failed: {tool_result.error}"

    async def moderate_plan_response(
        self,
        plan: Plan,
        user_question: str,
        state: SessionState,
        all_step_results: List[ToolResult]
    ) -> str:
        """
        Generate comprehensive final response after plan completion.

        This is the main answer to the user's original question, synthesizing
        all step results into a coherent response.

        Args:
            plan: The completed execution plan
            user_question: Original user question
            state: Current session state with accumulated data
            all_step_results: Results from all executed steps

        Returns:
            Comprehensive answer to user's question
        """
        system_prompt = """You are a helpful assistant providing final answers to user questions.
The user asked a question, and a series of tools were executed to gather information.

Your task is to:
1. Directly answer the user's original question
2. Synthesize information from all tool executions
3. Provide clear, actionable insights
4. Structure the response with proper formatting (markdown supported)
5. If any steps failed, acknowledge limitations

Guidelines:
- Start with a direct answer to the user's question
- Be comprehensive but concise
- Use markdown formatting for clarity (headers, lists, code blocks)
- Highlight important findings (CVE severity, vulnerabilities, etc.)
- End with clear next steps or recommendations if applicable
- Don't mention internal tool names or execution details
"""

        # Build context from all steps
        steps_context = self._build_steps_context(plan.steps, all_step_results)
        state_data = self._extract_relevant_state_data(state)

        user_message = f"""User's Original Question: {user_question}

Execution Summary:
- Total Steps: {len(plan.steps)}
- Successful: {sum(1 for r in all_step_results if r.success)}
- Failed: {sum(1 for r in all_step_results if not r.success)}

Step-by-Step Results:
{steps_context}

Accumulated Data from Session:
{state_data}

Generate a comprehensive final answer to the user's question."""

        try:
            llm_provider = self.llm_manager.get_provider(use_case="planner")
            response = await llm_provider.generate_text(
                system_prompt=system_prompt,
                user_message=user_message,
                temperature=0.3,
                max_tokens=2048
            )
            return response.strip()

        except Exception as e:
            logger.error(f"Plan response moderation failed: {e}")
            # Fallback to basic summary
            return self._generate_fallback_summary(plan, all_step_results, state)

    def _format_tool_result(self, tool_result: ToolResult) -> str:
        """Format tool result for LLM consumption."""
        if not tool_result.success:
            return f"Error: {tool_result.error}"

        output_str = f"Success: {tool_result.summary}\n"

        if tool_result.data:
            # Format relevant data fields
            for key, value in tool_result.data.items():
                if isinstance(value, (str, int, float, bool)):
                    output_str += f"- {key}: {value}\n"
                elif isinstance(value, list) and len(value) < 10:
                    output_str += f"- {key}: {', '.join(str(v) for v in value)}\n"

        return output_str

    def _build_steps_context(
        self,
        steps: List[PlanStep],
        results: List[ToolResult]
    ) -> str:
        """Build context string from all step executions."""
        context_parts = []

        for i, (step, result) in enumerate(zip(steps, results), 1):
            status = "✓" if result.success else "✗"
            context_parts.append(f"{i}. {status} {step.description}")

            if result.success and result.summary:
                context_parts.append(f"   Result: {result.summary}")
            elif not result.success:
                context_parts.append(f"   Error: {result.error}")

            # Include key data points
            if result.data:
                for key, value in list(result.data.items())[:3]:  # Limit to 3 items
                    if isinstance(value, (str, int, float, bool)):
                        context_parts.append(f"   - {key}: {value}")

        return "\n".join(context_parts)

    def _extract_relevant_state_data(self, state: SessionState) -> str:
        """Extract relevant data from session state."""
        data_parts = []

        # Extract commonly used fields
        important_fields = [
            'cve_id', 'severity', 'affected_packages', 'repo_url',
            'is_vulnerable', 'fix_applied', 'test_results',
            'jira_issue_key', 'confluence_page_url'
        ]

        for field in important_fields:
            value = getattr(state.data, field, None)
            if value is not None:
                data_parts.append(f"- {field}: {value}")

        return "\n".join(data_parts) if data_parts else "No additional data"

    def _generate_fallback_summary(
        self,
        plan: Plan,
        results: List[ToolResult],
        state: SessionState
    ) -> str:
        """Generate basic summary when LLM moderation fails."""
        success_count = sum(1 for r in results if r.success)
        fail_count = len(results) - success_count

        summary = f"Execution completed: {success_count}/{len(results)} steps successful"

        if fail_count > 0:
            summary += f", {fail_count} failed"

        summary += ".\n\n"

        # Add key findings if available
        if state.data.cve_id:
            summary += f"**CVE ID:** {state.data.cve_id}\n"
        if state.data.severity:
            summary += f"**Severity:** {state.data.severity}\n"
        if state.data.is_vulnerable is not None:
            summary += f"**Repository Vulnerable:** {'Yes' if state.data.is_vulnerable else 'No'}\n"

        return summary
```

**Integration with Executor:**

The Executor should be modified to call ResponseModerator after each step and after plan completion:

```python
# core/orchestrator/executor.py (UPDATED)

from core.orchestrator.response_moderator import ResponseModerator

class Executor:
    def __init__(
        self,
        tool_registry: ToolRegistry,
        consent_manager: ConsentManager,
        planner: Planner,
        response_moderator: ResponseModerator  # NEW
    ):
        self.tool_registry = tool_registry
        self.consent_manager = consent_manager
        self.planner = planner
        self.response_moderator = response_moderator  # NEW

    async def _execute_step(
        self,
        step: PlanStep,
        state: SessionState,
        progress_callback: Optional[Callable] = None
    ) -> Dict[str, Any]:
        """Execute a single plan step."""
        # ... existing execution logic ...

        result = await self._run_tool(tool, step, state, progress_callback)

        # NEW: Generate moderated response for this step
        if result.get("success") and progress_callback:
            moderated_message = await self.response_moderator.moderate_step_response(
                step=step,
                tool_result=result["tool_result"],
                user_question=state.messages[0].content,  # Original question
                state=state
            )

            # Send moderated message to chat
            await progress_callback("assistant_message", {
                "content": moderated_message,
                "metadata": {
                    "step_id": step.step_id,
                    "tool_name": step.tool_name
                }
            })

        return result

    async def execute_plan(
        self,
        plan: Plan,
        state: SessionState,
        progress_callback: Optional[Callable] = None
    ):
        """Execute plan steps with consent, retry, and recovery logic."""
        # ... existing execution logic ...

        all_results = []
        for group in step_groups:
            results = await asyncio.gather(*tasks, return_exceptions=True)
            all_results.extend(results)

        state.execution_status = "completed"

        # NEW: Generate final moderated response after plan completion
        if progress_callback:
            final_response = await self.response_moderator.moderate_plan_response(
                plan=plan,
                user_question=state.messages[0].content,
                state=state,
                all_step_results=[r.get("tool_result") for r in all_results if r.get("tool_result")]
            )

            # Send final comprehensive answer to chat
            await progress_callback("assistant_message", {
                "content": final_response,
                "metadata": {
                    "final_response": True,
                    "plan_id": plan.plan_id
                }
            })
```

---

## 5. API Endpoints Implementation

### 5.1 WebSocket Endpoint

```python
# api/websocket.py

from fastapi import WebSocket, WebSocketDisconnect, Depends
from typing import Dict
from core.orchestrator.planner import Planner
from core.orchestrator.executor import Executor
from core.orchestrator.consent_manager import ConsentManager
from models.session import SessionState, ConsentResponse, Message, ExecutionStatus
from datetime import datetime
import json
import asyncio

class WebSocketManager:
    def __init__(
        self,
        planner: Planner,
        executor: Executor,
        consent_manager: ConsentManager
    ):
        self.planner = planner
        self.executor = executor
        self.consent_manager = consent_manager

    async def handle_connection(
        self,
        websocket: WebSocket,
        session_state: SessionState
    ):
        """Handle WebSocket connection for a session."""
        await websocket.accept()

        # Start ping task
        ping_task = asyncio.create_task(self._ping_loop(websocket))

        try:
            while True:
                # Receive message
                data = await websocket.receive_text()
                message = json.loads(data)

                # Update last activity
                session_state.last_activity = datetime.utcnow()

                # Route message
                await self._handle_message(
                    message_type=message.get("type"),
                    payload=message.get("payload"),
                    websocket=websocket,
                    session_state=session_state
                )

        except WebSocketDisconnect:
            ping_task.cancel()
            # Session disconnected
            pass

    async def _handle_message(
        self,
        message_type: str,
        payload: dict,
        websocket: WebSocket,
        session_state: SessionState
    ):
        """Route incoming WebSocket message."""

        if message_type == "chat":
            await self._handle_chat(payload, websocket, session_state)

        elif message_type == "consent_response":
            await self._handle_consent_response(payload, session_state)

        elif message_type == "feedback_submit":
            await self._handle_feedback_submit(payload, session_state)

        elif message_type == "cancel_execution":
            await self._handle_cancel_execution(payload, session_state)

        elif message_type == "pong":
            pass  # Heartbeat response

    async def _handle_chat(
        self,
        payload: dict,
        websocket: WebSocket,
        session_state: SessionState
    ):
        """Handle chat message from user."""
        content = payload.get("content", "")

        # Store user message
        user_message = Message(
            message_id=str(uuid.uuid4()),
            role="user",
            content=content,
            timestamp=datetime.utcnow()
        )
        session_state.messages.append(user_message)

        # Set status to planning
        session_state.execution_status = ExecutionStatus.PLANNING

        # Stream thinking updates
        async def thinking_callback(step: str):
            await self._send_message(websocket, "thinking_update", {"step": step})

        # Create plan
        plan = await self.planner.create_plan(
            user_prompt=content,
            state=session_state,
            thinking_callback=thinking_callback
        )

        # Send plan to client
        await self._send_message(websocket, "plan_created", {"plan": plan.dict()})

        # Execute plan
        async def progress_callback(event_type: str, payload: dict):
            await self._send_message(websocket, event_type, payload)

        try:
            await self.executor.execute_plan(
                plan=plan,
                state=session_state,
                progress_callback=progress_callback
            )

            # Plan completed
            await self._send_message(websocket, "plan_completed", {
                "plan_id": plan.plan_id,
                "summary": session_state.data.summary or "Execution completed"
            })

        except Exception as e:
            await self._send_message(websocket, "error", {
                "message": str(e),
                "recoverable": False
            })

    async def _handle_consent_response(
        self,
        payload: dict,
        session_state: SessionState
    ):
        """Handle consent response from user."""
        response = ConsentResponse(**payload)
        self.consent_manager.submit_consent_response(response)

    async def _handle_feedback_submit(
        self,
        payload: dict,
        session_state: SessionState
    ):
        """Handle feedback submission."""
        step_id = payload.get("step_id")
        rating = payload.get("rating")
        sentiment = payload.get("sentiment")
        comment = payload.get("comment")

        if step_id and sentiment:
            session_state.feedback.step_ratings[step_id] = sentiment

        if rating:
            session_state.feedback.session_rating = rating

        if comment:
            session_state.feedback.comments.append(comment)

    async def _handle_cancel_execution(
        self,
        payload: dict,
        session_state: SessionState
    ):
        """Handle execution cancellation."""
        plan_id = payload.get("plan_id")
        if session_state.current_plan and session_state.current_plan.plan_id == plan_id:
            session_state.execution_status = ExecutionStatus.FAILED
            # Cancel execution (implementation depends on executor design)

    async def _send_message(
        self,
        websocket: WebSocket,
        message_type: str,
        payload: dict
    ):
        """Send message to client."""
        message = {
            "type": message_type,
            "payload": payload
        }
        await websocket.send_text(json.dumps(message))

    async def _ping_loop(self, websocket: WebSocket):
        """Send periodic ping messages."""
        while True:
            await asyncio.sleep(30)
            try:
                await self._send_message(websocket, "ping", {})
            except:
                break
```

### 5.2 REST API Endpoints

```python
# api/sessions.py

from fastapi import APIRouter, HTTPException, Depends
from models.session import CreateSessionRequest, CreateSessionResponse
from datetime import datetime, timedelta
import uuid

router = APIRouter(prefix="/api/v1/sessions", tags=["sessions"])

@router.post("/", response_model=CreateSessionResponse)
async def create_session(request: CreateSessionRequest):
    """
    Create new session (handled by Session Gateway).

    This endpoint would typically:
    1. Validate user_id
    2. Spawn ephemeral backend container
    3. Return container URLs + session token
    """
    session_id = f"sess-{uuid.uuid4()}"
    container_id = f"container-{uuid.uuid4().hex[:10]}"

    # Generate JWT token
    session_token = generate_jwt_token(
        session_id=session_id,
        user_id=request.user_id,
        role="user",  # From user lookup
        expiration=datetime.utcnow() + timedelta(hours=1)
    )

    # In actual implementation:
    # - Session Gateway spawns backend container (K8s Pod/Job or Serverless)
    # - Waits for container health check
    # - Returns container URLs

    return CreateSessionResponse(
        session_id=session_id,
        container_id=container_id,
        websocket_url=f"wss://{container_id}.cluster/ws/{session_id}",
        api_base_url=f"https://{container_id}.cluster/api/v1",
        session_token=session_token,
        expires_at=datetime.utcnow() + timedelta(hours=1),
        idle_timeout_ms=300000
    )


# api/queries.py

from fastapi import APIRouter, HTTPException, BackgroundTasks
from models.session import CreateQueryRequest, CreateQueryResponse, QueryStatusResponse, QueryResultResponse

router = APIRouter(prefix="/api/v1/queries", tags=["queries"])

@router.post("/", response_model=CreateQueryResponse)
async def create_query(request: CreateQueryRequest, background_tasks: BackgroundTasks):
    """Submit programmatic query (spawns dedicated backend container)."""
    query_id = f"q-{uuid.uuid4()}"

    # Spawn backend container for this query
    # Similar to session creation but for API execution

    # Start async execution in background
    background_tasks.add_task(execute_query_async, query_id, request)

    return CreateQueryResponse(
        query_id=query_id,
        status="queued",
        poll_url=f"/api/v1/queries/{query_id}"
    )

@router.get("/{query_id}", response_model=QueryStatusResponse)
async def get_query_status(query_id: str):
    """Poll query status."""
    # Fetch from database or in-memory store
    query_data = await fetch_query_status(query_id)

    return QueryStatusResponse(
        query_id=query_id,
        status=query_data["status"],
        progress=query_data.get("progress")
    )

@router.get("/{query_id}/result", response_model=QueryResultResponse)
async def get_query_result(query_id: str):
    """Get query result (only if completed)."""
    query_data = await fetch_query_data(query_id)

    if query_data["status"] not in ["completed", "failed"]:
        raise HTTPException(status_code=400, detail="Query not yet completed")

    return QueryResultResponse(
        query_id=query_id,
        status=query_data["status"],
        result=query_data.get("result"),
        execution_log=query_data.get("execution_log", [])
    )


# api/dashboard.py

from fastapi import APIRouter, HTTPException, Depends
from models.session import SessionListRequest, SessionListResponse, AnalyticsSummaryResponse
from db.repository import SessionRepository
from utils.auth import require_admin_role

router = APIRouter(prefix="/api/v1", tags=["dashboard"])

@router.get("/sessions", response_model=SessionListResponse, dependencies=[Depends(require_admin_role)])
async def list_sessions(
    user_id: str,  # From JWT
    date_from: Optional[datetime] = None,
    date_to: Optional[datetime] = None,
    status_filter: Optional[str] = None,
    limit: int = 50,
    offset: int = 0
):
    """List completed sessions (admin only)."""
    repo = SessionRepository()

    sessions, total_count = await repo.list_sessions(
        date_from=date_from,
        date_to=date_to,
        status_filter=status_filter,
        limit=limit,
        offset=offset
    )

    return SessionListResponse(
        sessions=sessions,
        total_count=total_count,
        has_more=(offset + len(sessions)) < total_count
    )

@router.get("/analytics/summary", response_model=AnalyticsSummaryResponse, dependencies=[Depends(require_admin_role)])
async def get_analytics_summary(
    user_id: str,  # From JWT
    date_from: Optional[datetime] = None,
    date_to: Optional[datetime] = None
):
    """Get aggregated analytics (admin only)."""
    repo = SessionRepository()

    analytics = await repo.get_analytics_summary(
        date_from=date_from,
        date_to=date_to
    )

    return AnalyticsSummaryResponse(**analytics)

@router.get("/feedback")
async def list_feedback(
    user_id: str,  # From JWT (admin only)
    session_id: Optional[str] = None,
    limit: int = 50,
    offset: int = 0,
    dependencies=[Depends(require_admin_role)]
):
    """List feedback records (admin only)."""
    repo = SessionRepository()

    feedback_list = await repo.list_feedback(
        session_id=session_id,
        limit=limit,
        offset=offset
    )

    return {"feedback": feedback_list}


# api/tools.py

from fastapi import APIRouter
from core.tool_registry import ToolRegistry

router = APIRouter(prefix="/api/v1/tools", tags=["tools"])

@router.get("/")
async def list_tools(tool_registry: ToolRegistry = Depends()):
    """List all available tools."""
    tools = tool_registry.get_all_tools()

    return {
        "tools": [
            {
                "name": tool.name,
                "display_name": tool.display_name,
                "description": tool.description,
                "category": tool.category,
                "capabilities": tool.capabilities,
                "risk_level": tool.risk_level
            }
            for tool in tools
        ]
    }

@router.get("/health")
async def health_check():
    """Health check endpoint."""
    return {"status": "ok", "timestamp": datetime.utcnow().isoformat()}
```

---

## 6. CVE Remediation Tools Implementation

### 6.1 Tool Organization

**Naming Convention:** `{purpose}_{technology}_{action}_{target}`

```
tools/
├── base.py                              # Base Tool class
│
├── cve_remediation/                     # Purpose: CVE Remediation
│   ├── go/                              # Technology: Go
│   │   ├── cve_go_extract_request_details.py
│   │   ├── cve_go_fetch_vulnerability_data.py
│   │   ├── cve_go_analyze_repository_impact.py
│   │   ├── cve_go_update_dependencies.py
│   │   ├── cve_go_validate_fixes.py
│   │   ├── cve_go_diagnose_upgrade_failure.py
│   │   └── cve_go_generate_report.py
│   │
│   └── python/                          # Future: Python CVE tools
│       └── cve_python_fetch_vulnerability_data.py
│
├── documentation/                       # Purpose: Documentation
│   └── confluence/                      # Technology: Confluence
│       ├── docs_confluence_read_page.py
│       ├── docs_confluence_explain_content.py
│       └── docs_confluence_search_pages.py
│
├── project_management/                  # Purpose: Project Management
│   └── jira/                           # Technology: Jira
│       ├── pm_jira_fetch_issue.py
│       ├── pm_jira_query_issues.py
│       ├── pm_jira_get_sprint_status.py
│       └── pm_jira_create_issue.py
│
└── quality_engineering/                 # Purpose: QE/Testing
    ├── project/                        # Technology: Project-specific
    │   ├── qe_project_analyze_test_failure.py
    │   └── qe_project_read_documentation.py
    └── github/                         # Technology: GitHub
        └── qe_github_search_similar_issues.py
```

### 6.2 Tool Registry with Purpose/Technology Indexing

```python
# core/tool_registry.py

from typing import Dict, List, Optional
from tools.base import Tool
import logging

logger = logging.getLogger(__name__)

class ToolRegistry:
    """
    Central registry for all tools with multi-dimensional indexing.

    Supports:
    - By name (unique identifier)
    - By purpose (cve_remediation, documentation, etc.)
    - By technology (go, python, confluence, jira, etc.)
    - By category (internal, external)
    """

    def __init__(self):
        self.tools: Dict[str, Tool] = {}

        # Multi-dimensional indexes
        self.purpose_index: Dict[str, List[str]] = {}      # purpose -> tool_names
        self.technology_index: Dict[str, List[str]] = {}   # technology -> tool_names
        self.category_index: Dict[str, List[str]] = {}     # category -> tool_names

    def register(self, tool: Tool):
        """Register tool with automatic indexing."""
        if tool.name in self.tools:
            logger.warning(f"Tool {tool.name} already registered, overwriting")

        self.tools[tool.name] = tool

        # Index by purpose
        if tool.purpose not in self.purpose_index:
            self.purpose_index[tool.purpose] = []
        if tool.name not in self.purpose_index[tool.purpose]:
            self.purpose_index[tool.purpose].append(tool.name)

        # Index by technology
        if tool.technology not in self.technology_index:
            self.technology_index[tool.technology] = []
        if tool.name not in self.technology_index[tool.technology]:
            self.technology_index[tool.technology].append(tool.name)

        # Index by category
        if tool.category not in self.category_index:
            self.category_index[tool.category] = []
        if tool.name not in self.category_index[tool.category]:
            self.category_index[tool.category].append(tool.name)

        logger.info(f"Registered tool: {tool.name} (purpose={tool.purpose}, tech={tool.technology})")

    def get_by_name(self, name: str) -> Optional[Tool]:
        """Get tool by exact name."""
        return self.tools.get(name)

    def get_by_purpose(self, purpose: str) -> List[Tool]:
        """Get all tools for a specific purpose."""
        tool_names = self.purpose_index.get(purpose, [])
        return [self.tools[name] for name in tool_names if name in self.tools]

    def get_by_technology(self, technology: str) -> List[Tool]:
        """Get all tools for a specific technology."""
        tool_names = self.technology_index.get(technology, [])
        return [self.tools[name] for name in tool_names if name in self.tools]

    def get_by_category(self, category: str) -> List[Tool]:
        """Get all tools in a category."""
        tool_names = self.category_index.get(category, [])
        return [self.tools[name] for name in tool_names if name in self.tools]

    def get_all_tools(self) -> List[Tool]:
        """Get all registered tools."""
        return list(self.tools.values())

    def get_tool_summary(self) -> Dict[str, any]:
        """Get registry statistics."""
        return {
            "total_tools": len(self.tools),
            "by_purpose": {p: len(names) for p, names in self.purpose_index.items()},
            "by_technology": {t: len(names) for t, names in self.technology_index.items()},
            "by_category": {c: len(names) for c, names in self.category_index.items()}
        }

    def search_tools(
        self,
        purpose: Optional[str] = None,
        technology: Optional[str] = None,
        category: Optional[str] = None
    ) -> List[Tool]:
        """
        Search tools with multiple filters (AND logic).

        Example:
            search_tools(purpose="cve_remediation", technology="go")
            Returns only Go CVE remediation tools
        """
        # Start with all tools
        candidates = set(self.tools.keys())

        # Apply filters
        if purpose:
            candidates &= set(self.purpose_index.get(purpose, []))

        if technology:
            candidates &= set(self.technology_index.get(technology, []))

        if category:
            candidates &= set(self.category_index.get(category, []))

        return [self.tools[name] for name in candidates if name in self.tools]
```

---

### 6.3 Base Tool Class

```python
# tools/base.py

from abc import ABC, abstractmethod
from models.session import SessionState, ToolResult
from typing import Dict, Any, List, Optional
from pydantic import BaseModel

class Tool(ABC):
    """Base class for all tools."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config

    @property
    @abstractmethod
    def name(self) -> str:
        """
        Tool identifier following pattern: {purpose}_{technology}_{action}_{target}
        Example: cve_go_fetch_vulnerability_data
        """
        pass

    @property
    @abstractmethod
    def display_name(self) -> str:
        """Human-friendly name."""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """What this tool does."""
        pass

    @property
    @abstractmethod
    def purpose(self) -> str:
        """
        Primary purpose category of this tool.
        Examples: cve_remediation, documentation, project_management, quality_engineering
        """
        pass

    @property
    @abstractmethod
    def technology(self) -> str:
        """
        Technology or system this tool works with.
        Examples: go, python, confluence, jira, github
        """
        pass

    @property
    @abstractmethod
    def category(self) -> str:
        """Tool category: internal | external."""
        pass

    @property
    @abstractmethod
    def capabilities(self) -> List[str]:
        """Keywords for LLM tool matching."""
        pass

    @property
    @abstractmethod
    def state_inputs(self) -> List[str]:
        """Required state fields."""
        pass

    @property
    @abstractmethod
    def state_outputs(self) -> List[str]:
        """State fields this tool writes."""
        pass

    @property
    def requires_consent(self) -> bool:
        """Whether to ask user consent."""
        return True

    @property
    def risk_level(self) -> str:
        """Risk level: low | medium | high."""
        return "medium"

    @property
    def is_retriable(self) -> bool:
        """Can retry on failure."""
        return False

    @property
    def max_retries(self) -> int:
        """Max retry attempts."""
        return 0

    @property
    def retry_delay_ms(self) -> int:
        """Delay between retries (ms)."""
        return 1000

    @property
    def preferred_llm(self) -> Optional[str]:
        """
        Preferred LLM provider for this tool.

        Returns:
            LLM provider name from llm_config.yaml (e.g., "gemini_flash", "gemini_pro")
            None means use default tool LLM

        Examples:
            - Simple extraction tasks: "gemini_flash" or "ollama_llama3"
            - Complex analysis: "gemini_pro"
            - Code-focused tasks: "ollama_codellama"
        """
        return None  # Default: use system default

    @abstractmethod
    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """
        Execute tool logic.

        Args:
            state: Session state (read inputs, write outputs)
            parameters: Tool-specific parameters from plan

        Returns:
            ToolResult with success/error
        """
        pass
```

### 6.4 Tool Implementation Examples

#### 6.4.1 CVE Remediation: Go Tools

##### cve_go_extract_request_details

```python
# tools/cve_remediation/go/cve_go_extract_request_details.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
from llm.provider import LLMProvider
from pydantic import BaseModel
import logging

logger = logging.getLogger(__name__)

class CVEInput(BaseModel):
    """Structured output for LLM extraction."""
    cve_id: str
    repo_url: str
    repo_version: str = "main"

class CVEGoExtractRequestDetailsTool(Tool):
    def __init__(self, llm_provider: LLMProvider, config: Dict[str, Any]):
        super().__init__(config)
        self.llm = llm_provider

    @property
    def name(self) -> str:
        return "cve_go_extract_request_details"

    @property
    def display_name(self) -> str:
        return "Go CVE Request Parser"

    @property
    def description(self) -> str:
        return """Extracts CVE identifier, Go repository URL, and version/branch from user's
natural language request about Go vulnerability analysis. Validates that the request is
about analyzing a CVE for a Go repository."""

    @property
    def purpose(self) -> str:
        return "cve_remediation"

    @property
    def technology(self) -> str:
        return "go"

    @property
    def category(self) -> str:
        return "internal"

    @property
    def capabilities(self) -> List[str]:
        return [
            "parse Go CVE analysis request",
            "extract CVE identifier",
            "extract Go repository information",
            "validate Go vulnerability request",
            "understand CVE remediation intent"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return []  # Reads from messages

    @property
    def state_outputs(self) -> List[str]:
        return ["cve_id", "repo_url", "repo_version", "are_inputs_valid"]

    @property
    def requires_consent(self) -> bool:
        return False

    @property
    def risk_level(self) -> str:
        return "low"

    @property
    def is_retriable(self) -> bool:
        return True

    @property
    def max_retries(self) -> int:
        return 2

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Extract CVE ID, repo URL, version from last user message."""
        start_time = datetime.utcnow()

        try:
            # Get last user message
            user_messages = [msg for msg in state.messages if msg.role == "user"]
            if not user_messages:
                return ToolResult(
                    success=False,
                    error="No user message found",
                    execution_time_ms=0
                )

            user_prompt = user_messages[-1].content

            # Use LLM with structured output
            system_prompt = """You are an expert at extracting CVE and repository information from natural language.

Extract:
1. CVE ID (format: CVE-YYYY-NNNN)
2. Repository URL (GitHub URL)
3. Repository version (branch/tag, defaults to "main")

If information is missing or ambiguous, set fields to empty string."""

            extraction_result = await self.llm.generate_structured_output(
                system_prompt=system_prompt,
                user_message=f"Extract CVE and repository information:\n\n{user_prompt}",
                output_schema=CVEInput
            )

            cve_input = CVEInput(**extraction_result)

            # Validate
            if not cve_input.cve_id or not cve_input.repo_url:
                state.data.are_inputs_valid = False
                state.data.llm_invoke_error = "Could not extract CVE ID and/or repository URL"

                return ToolResult(
                    success=False,
                    error="Missing required inputs: Please provide CVE ID and repository URL",
                    execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
                )

            # Write to state
            state.data.cve_id = cve_input.cve_id
            state.data.repo_url = cve_input.repo_url
            state.data.repo_version = cve_input.repo_version or "main"
            state.data.are_inputs_valid = True

            logger.info(f"Extracted inputs: CVE={cve_input.cve_id}, Repo={cve_input.repo_url}, Version={cve_input.repo_version}")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={
                    "cve_id": cve_input.cve_id,
                    "repo_url": cve_input.repo_url,
                    "repo_version": cve_input.repo_version
                }
            )

        except Exception as e:
            logger.error(f"Input extraction failed: {e}")
            state.data.are_inputs_valid = False
            state.data.llm_invoke_error = str(e)

            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )
```

##### cve_go_fetch_vulnerability_data

```python
# tools/cve_remediation/go/cve_go_fetch_vulnerability_data.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List, Optional
import httpx
import logging
from pydantic import BaseModel

logger = logging.getLogger(__name__)

class AffectedPackage(BaseModel):
    package_name: str
    root_package: str
    fixed_in: Optional[str]
    repo_url: Optional[str]
    affected_symbols: List[str] = []

class CVEGoFetchVulnerabilityDataTool(Tool):
    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.http_client = httpx.AsyncClient(timeout=30.0)

    @property
    def name(self) -> str:
        return "cve_go_fetch_vulnerability_data"

    @property
    def display_name(self) -> str:
        return "Go CVE Data Fetcher"

    @property
    def description(self) -> str:
        return """Fetches comprehensive CVE vulnerability data from multiple security databases
(OSV, NVD, CVEORG, Bugzilla) specifically for Go language vulnerabilities. Identifies
affected Go packages, fix versions, CVSS scores, and vulnerable symbols/functions."""

    @property
    def purpose(self) -> str:
        return "cve_remediation"

    @property
    def technology(self) -> str:
        return "go"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "fetch Go CVE data from OSV",
            "fetch Go CVE data from NVD",
            "query Go vulnerability database",
            "identify affected Go packages",
            "find Go package fix versions",
            "extract CVSS scores for Go CVEs",
            "map vulnerable Go symbols"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return ["cve_id"]

    @property
    def state_outputs(self) -> List[str]:
        return [
            "is_valid_go_cve",
            "is_fix_available",
            "affected_packages",
            "go_cve_id",
            "cvss_score",
            "cvss_severity"
        ]

    @property
    def requires_consent(self) -> bool:
        return False

    @property
    def risk_level(self) -> str:
        return "low"

    @property
    def is_retriable(self) -> bool:
        return True

    @property
    def max_retries(self) -> int:
        return 3

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Fetch and analyze CVE data from multiple sources."""
        start_time = datetime.utcnow()
        cve_id = state.data.cve_id

        try:
            logger.info(f"Assessing CVE: {cve_id}")

            # Fetch from multiple sources in parallel
            osv_data, nvd_data, cveorg_data, bugzilla_data = await asyncio.gather(
                self._fetch_osv_data(cve_id),
                self._fetch_nvd_data(cve_id),
                self._fetch_cveorg_data(cve_id),
                self._fetch_bugzilla_data(cve_id),
                return_exceptions=True
            )

            # Store raw data
            state.data.fetched_osv_data = osv_data if not isinstance(osv_data, Exception) else None
            state.data.fetched_nvd_data = nvd_data if not isinstance(nvd_data, Exception) else None
            state.data.fetched_cveorg_data = cveorg_data if not isinstance(cveorg_data, Exception) else None
            state.data.fetched_bugzilla_data = bugzilla_data if not isinstance(bugzilla_data, Exception) else None

            # Analyze with LLM to extract structured data
            analysis = await self._analyze_cve_data_with_llm(
                cve_id=cve_id,
                osv_data=state.data.fetched_osv_data,
                nvd_data=state.data.fetched_nvd_data,
                cveorg_data=state.data.fetched_cveorg_data
            )

            # Write to state
            state.data.is_valid_go_cve = analysis["is_valid_go_cve"]
            state.data.go_cve_id = analysis.get("go_cve_id")
            state.data.affected_packages = analysis.get("affected_packages", [])
            state.data.is_fix_available = len([p for p in analysis.get("affected_packages", []) if p.get("fixed_in")]) > 0
            state.data.cvss_score = analysis.get("cvss_score")
            state.data.cvss_severity = analysis.get("cvss_severity")

            # Fetch repository URLs for packages
            for package_data in state.data.affected_packages:
                if not package_data.get("repo_url"):
                    repo_url = await self._fetch_repo_url(package_data["package_name"])
                    package_data["repo_url"] = repo_url

            logger.info(f"CVE Assessment complete: is_go_cve={state.data.is_valid_go_cve}, packages={len(state.data.affected_packages)}")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={
                    "is_valid_go_cve": state.data.is_valid_go_cve,
                    "affected_packages_count": len(state.data.affected_packages),
                    "is_fix_available": state.data.is_fix_available
                }
            )

        except Exception as e:
            logger.error(f"CVE assessment failed: {e}")
            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )

    async def _fetch_osv_data(self, cve_id: str) -> Dict[str, Any]:
        """Fetch from OSV.dev API."""
        url = "https://api.osv.dev/v1/query"
        response = await self.http_client.post(url, json={"commit": "", "version": "", "package": {"name": cve_id, "ecosystem": "Go"}})
        return response.json() if response.status_code == 200 else {}

    async def _fetch_nvd_data(self, cve_id: str) -> Dict[str, Any]:
        """Fetch from NVD NIST API."""
        url = f"https://services.nvd.nist.gov/rest/json/cves/2.0?cveId={cve_id}"
        response = await self.http_client.get(url)
        return response.json() if response.status_code == 200 else {}

    async def _fetch_cveorg_data(self, cve_id: str) -> Dict[str, Any]:
        """Fetch from CVEORG/MITRE API."""
        url = f"https://cveawg.mitre.org/api/cve/{cve_id}"
        response = await self.http_client.get(url)
        return response.json() if response.status_code == 200 else {}

    async def _fetch_bugzilla_data(self, cve_id: str) -> Dict[str, Any]:
        """Fetch from Red Hat Bugzilla."""
        url = f"https://bugzilla.redhat.com/rest/bug?alias={cve_id}"
        response = await self.http_client.get(url)
        return response.json() if response.status_code == 200 else {}

    async def _fetch_repo_url(self, package_name: str) -> Optional[str]:
        """Fetch repository URL from deps.dev."""
        url = f"https://api.deps.dev/v3alpha/systems/go/packages/{package_name}"
        response = await self.http_client.get(url)
        if response.status_code == 200:
            data = response.json()
            return data.get("links", {}).get("repo")
        return None

    async def _analyze_cve_data_with_llm(
        self,
        cve_id: str,
        osv_data: Optional[Dict],
        nvd_data: Optional[Dict],
        cveorg_data: Optional[Dict]
    ) -> Dict[str, Any]:
        """Use LLM to extract structured findings from multi-source data."""
        # Implementation: Send combined data to LLM with structured output schema
        # Returns: is_valid_go_cve, go_cve_id, affected_packages, cvss_score, etc.
        pass
```

##### cve_go_analyze_repository_impact (Abbreviated)

```python
# tools/cve_remediation/go/cve_go_analyze_repository_impact.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
import subprocess
import tempfile
import logging

logger = logging.getLogger(__name__)

class CVEGoAnalyzeRepositoryImpactTool(Tool):
    """
    Analyzes go.mod and codebase to determine if CVE actually affects repository.

    Two-phase analysis:
    1. Phase 1: go.mod check (fast) - Check versions
    2. Phase 2: Code analysis (thorough) - Search for vulnerable symbol usage
    """

    @property
    def name(self) -> str:
        return "cve_go_analyze_repository_impact"

    @property
    def display_name(self) -> str:
        return "Go Repository CVE Impact Analyzer"

    @property
    def description(self) -> str:
        return """Analyzes a Go repository to determine if a CVE actually affects it by examining
go.mod dependencies and searching source code for usage of vulnerable package symbols.
Performs deep static analysis to detect false alarms where CVE is listed but not exploitable."""

    @property
    def purpose(self) -> str:
        return "cve_remediation"

    @property
    def technology(self) -> str:
        return "go"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "analyze Go repository go.mod file",
            "check Go package versions",
            "search Go code for vulnerable symbols",
            "detect CVE false alarms in Go projects",
            "perform Go static code analysis",
            "determine real Go CVE impact"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return ["repo_url", "repo_version", "affected_packages", "cve_id"]

    @property
    def state_outputs(self) -> List[str]:
        return [
            "is_false_alarm",
            "analysis_incomplete",
            "cloned_repo_path",
            "gomod_analysis",
            "code_chunks"
        ]

    @property
    def requires_consent(self) -> bool:
        return True  # Clones repository

    @property
    def risk_level(self) -> str:
        return "medium"

    @property
    def is_retriable(self) -> bool:
        return True

    @property
    def max_retries(self) -> int:
        return 2

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Execute false alarm check."""
        start_time = datetime.utcnow()

        try:
            # Phase 1: Fetch and analyze go.mod
            gomod_result = await self._analyze_gomod(
                repo_url=state.data.repo_url,
                repo_version=state.data.repo_version,
                affected_packages=state.data.affected_packages
            )

            state.data.gomod_analysis = gomod_result

            # If go.mod shows packages missing or at safe versions → False alarm
            if gomod_result["is_false_alarm"]:
                state.data.is_false_alarm = True
                state.data.analysis_incomplete = False

                return ToolResult(
                    success=True,
                    execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                    output_data={"is_false_alarm": True, "reason": "Packages not in go.mod or at safe versions"}
                )

            # Phase 2: Clone repository and analyze code
            cloned_path = await self._clone_repository(
                repo_url=state.data.repo_url,
                repo_version=state.data.repo_version,
                session_id=state.session_id
            )

            state.data.cloned_repo_path = cloned_path

            # Run repo_analyzer binary for each affected package
            code_analysis = await self._analyze_code_usage(
                cloned_path=cloned_path,
                affected_packages=state.data.affected_packages
            )

            state.data.code_chunks = code_analysis["code_chunks"]
            state.data.is_false_alarm = not code_analysis["vulnerable_symbols_found"]
            state.data.analysis_incomplete = code_analysis["incomplete"]

            logger.info(f"False alarm check: is_false_alarm={state.data.is_false_alarm}")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={
                    "is_false_alarm": state.data.is_false_alarm,
                    "vulnerable_symbols_found": code_analysis["vulnerable_symbols_found"]
                }
            )

        except Exception as e:
            logger.error(f"False alarm check failed: {e}")
            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )

    async def _analyze_gomod(
        self,
        repo_url: str,
        repo_version: str,
        affected_packages: List[Dict]
    ) -> Dict[str, Any]:
        """Fetch and analyze go.mod file."""
        # Fetch go.mod from GitHub raw URL
        # Parse for package versions
        # Compare with affected packages
        # Return is_false_alarm + details
        pass

    async def _clone_repository(
        self,
        repo_url: str,
        repo_version: str,
        session_id: str
    ) -> str:
        """Clone repository to temp directory."""
        temp_dir = tempfile.mkdtemp(prefix=f"session-{session_id}-")
        subprocess.run(
            ["git", "clone", "--depth=1", "--branch", repo_version, repo_url, temp_dir],
            check=True
        )
        return temp_dir

    async def _analyze_code_usage(
        self,
        cloned_path: str,
        affected_packages: List[Dict]
    ) -> Dict[str, Any]:
        """Run repo_analyzer binary to find vulnerable symbol usage."""
        # For each affected package:
        #   - Run: repo_analyzer -subpackagesearch <package> <repo_path>
        #   - Parse JSON output
        #   - Match affected_symbols against found symbols
        #   - Build code_chunks with affected code
        # Return: vulnerable_symbols_found, code_chunks, incomplete
        pass
```

##### Other Go CVE Tools (Summaries)

```python
# tools/cve_remediation/go/cve_go_update_dependencies.py
class CVEGoUpdateDependenciesTool(Tool):
    """
    Automatically updates go.mod file to fix CVE vulnerabilities.

    Handles:
    - Replace blocks (complex logic for version overrides)
    - Version selection (nearest higher version)
    - HARDREPLACE detection (manual intervention needed)
    """
    name = "cve_go_update_dependencies"
    purpose = "cve_remediation"
    technology = "go"
    state_inputs = ["cloned_repo_path", "affected_packages"]
    state_outputs = ["go_mod_updated", "version_changes", "go_runtime_version"]
    requires_consent = True
    risk_level = "high"  # Modifies code


# tools/cve_remediation/go/cve_go_validate_fixes.py
class CVEGoValidateFixesTool(Tool):
    """
    Validates that CVE fixes haven't broken the Go repository.

    Executes:
    1. go mod tidy
    2. go mod vendor
    3. make test (extracted from Makefile via LLM)
    4. make build (only if tests pass)
    """
    name = "cve_go_validate_fixes"
    purpose = "cve_remediation"
    technology = "go"
    state_inputs = ["cloned_repo_path", "go_runtime_version"]
    state_outputs = ["is_test_successful", "last_failure_step", "last_failure_logs"]
    requires_consent = True
    risk_level = "medium"


# tools/cve_remediation/go/cve_go_diagnose_upgrade_failure.py
class CVEGoDiagnoseUpgradeFailureTool(Tool):
    """
    Analyzes Go dependency upgrade failures by examining git commit differences.

    Process:
    - Clones package repositories
    - Extracts commits between old and new versions
    - Sends commit diffs + failure logs + code context to LLM
    - LLM identifies breaking changes and suggests fixes
    """
    name = "cve_go_diagnose_upgrade_failure"
    purpose = "cve_remediation"
    technology = "go"
    state_inputs = ["version_changes", "last_failure_step", "last_failure_logs", "code_chunks"]
    state_outputs = ["failure_analysis_response"]
    requires_consent = False
    risk_level = "low"


# tools/cve_remediation/go/cve_go_generate_report.py
class CVEGoGenerateReportTool(Tool):
    """
    Generates comprehensive markdown summary of Go CVE remediation workflow.

    Sections:
    1. CVE Overview
    2. Repository
    3. Analysis Result
    4. Actions Taken
    5. Test Results
    6. Outcome
    7. Recommendations
    """
    name = "cve_go_generate_report"
    purpose = "cve_remediation"
    technology = "go"
    state_inputs = []  # Reads entire state
    state_outputs = ["final_output", "generated_summary"]
    requires_consent = False
    risk_level = "low"
```

---

#### 6.4.2 Documentation: Confluence Tools

```python
# tools/documentation/confluence/docs_confluence_read_page.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
import httpx
import logging

logger = logging.getLogger(__name__)

class DocsConfluenceReadPageTool(Tool):
    """Fetches and reads content from Confluence wiki pages."""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.confluence_url = config.get("confluence_url")
        self.confluence_token = config.get("confluence_token")

    @property
    def name(self) -> str:
        return "docs_confluence_read_page"

    @property
    def display_name(self) -> str:
        return "Confluence Page Reader"

    @property
    def description(self) -> str:
        return """Fetches and reads content from Confluence wiki pages. Retrieves page title,
body content, attachments, and metadata. Supports both Cloud and Server/Data Center instances."""

    @property
    def purpose(self) -> str:
        return "documentation"

    @property
    def technology(self) -> str:
        return "confluence"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "read Confluence pages",
            "fetch Confluence content",
            "access wiki documentation",
            "retrieve Confluence page metadata",
            "download Confluence attachments"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return ["confluence_page_url"]

    @property
    def state_outputs(self) -> List[str]:
        return ["confluence_page_title", "confluence_page_content", "confluence_page_metadata"]

    @property
    def requires_consent(self) -> bool:
        return False

    @property
    def risk_level(self) -> str:
        return "low"

    @property
    def is_retriable(self) -> bool:
        return True

    @property
    def max_retries(self) -> int:
        return 3

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Fetch Confluence page content."""
        start_time = datetime.utcnow()

        try:
            page_url = state.data.confluence_page_url

            # Parse page ID from URL
            page_id = self._extract_page_id(page_url)

            # Fetch page via Confluence REST API
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{self.confluence_url}/rest/api/content/{page_id}",
                    params={"expand": "body.storage,version,metadata"},
                    headers={"Authorization": f"Bearer {self.confluence_token}"}
                )
                response.raise_for_status()
                page_data = response.json()

            # Write to state
            state.data.confluence_page_title = page_data.get("title")
            state.data.confluence_page_content = page_data.get("body", {}).get("storage", {}).get("value")
            state.data.confluence_page_metadata = {
                "version": page_data.get("version", {}).get("number"),
                "last_modified": page_data.get("version", {}).get("when"),
                "space": page_data.get("space", {}).get("key")
            }

            logger.info(f"Fetched Confluence page: {state.data.confluence_page_title}")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={"title": state.data.confluence_page_title}
            )

        except Exception as e:
            logger.error(f"Failed to fetch Confluence page: {e}")
            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )

    def _extract_page_id(self, url: str) -> str:
        """Extract page ID from Confluence URL."""
        # Parse pageId from URL (e.g., /display/SPACE/Page+Title or /pages/viewpage.action?pageId=12345)
        # Implementation details...
        pass


# tools/documentation/confluence/docs_confluence_explain_content.py

class DocsConfluenceExplainContentTool(Tool):
    """Analyzes and explains technical content from Confluence pages using LLM."""

    name = "docs_confluence_explain_content"
    display_name = "Confluence Content Explainer"
    purpose = "documentation"
    technology = "confluence"
    category = "internal"

    description = """Analyzes and explains technical content from Confluence pages. Provides
simplified explanations, identifies key concepts, extracts action items, and answers
specific questions about the documentation."""

    capabilities = [
        "explain Confluence documentation",
        "summarize technical wiki pages",
        "extract key concepts from docs",
        "answer questions about documentation",
        "simplify complex technical content"
    ]

    state_inputs = ["confluence_page_content", "user_question"]
    state_outputs = ["explanation_summary", "key_concepts", "action_items", "qa_response"]
    requires_consent = False
    risk_level = "low"
```

---

#### 6.4.3 Project Management: Jira Tools

```python
# tools/project_management/jira/pm_jira_fetch_issue.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
from jira import JIRA
import logging

logger = logging.getLogger(__name__)

class PMJiraFetchIssueTool(Tool):
    """Fetches detailed information about a specific Jira issue."""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.jira_client = JIRA(
            server=config.get("jira_url"),
            token_auth=config.get("jira_token")
        )

    @property
    def name(self) -> str:
        return "pm_jira_fetch_issue"

    @property
    def display_name(self) -> str:
        return "Jira Issue Fetcher"

    @property
    def description(self) -> str:
        return """Fetches detailed information about a specific Jira issue including summary,
description, status, assignee, comments, attachments, sprint, and custom fields.
Supports both Jira Cloud and Server/Data Center."""

    @property
    def purpose(self) -> str:
        return "project_management"

    @property
    def technology(self) -> str:
        return "jira"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "fetch Jira issue details",
            "retrieve Jira ticket information",
            "get Jira issue status",
            "access Jira comments",
            "read Jira custom fields"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return ["jira_issue_key"]

    @property
    def state_outputs(self) -> List[str]:
        return [
            "jira_issue_summary",
            "jira_issue_description",
            "jira_issue_status",
            "jira_issue_assignee",
            "jira_issue_comments",
            "jira_issue_metadata"
        ]

    @property
    def requires_consent(self) -> bool:
        return False

    @property
    def risk_level(self) -> str:
        return "low"

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Fetch Jira issue details."""
        start_time = datetime.utcnow()

        try:
            issue_key = state.data.jira_issue_key
            issue = self.jira_client.issue(issue_key, expand='changelog,renderedFields')

            # Write to state
            state.data.jira_issue_summary = issue.fields.summary
            state.data.jira_issue_description = issue.fields.description
            state.data.jira_issue_status = issue.fields.status.name
            state.data.jira_issue_assignee = issue.fields.assignee.displayName if issue.fields.assignee else None
            state.data.jira_issue_comments = [
                {"author": c.author.displayName, "body": c.body, "created": str(c.created)}
                for c in issue.fields.comment.comments
            ]
            state.data.jira_issue_metadata = {
                "priority": issue.fields.priority.name if issue.fields.priority else None,
                "labels": issue.fields.labels,
                "created": str(issue.fields.created),
                "updated": str(issue.fields.updated)
            }

            logger.info(f"Fetched Jira issue: {issue_key}")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={"issue_key": issue_key, "summary": state.data.jira_issue_summary}
            )

        except Exception as e:
            logger.error(f"Failed to fetch Jira issue: {e}")
            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )


# Other Jira tools (summaries)
class PMJiraQueryIssuesTool(Tool):
    """Executes JQL queries to find issues matching criteria."""
    name = "pm_jira_query_issues"
    purpose = "project_management"
    technology = "jira"

class PMJiraCreateIssueTool(Tool):
    """Creates new Jira issues (bugs, stories, tasks, epics)."""
    name = "pm_jira_create_issue"
    purpose = "project_management"
    technology = "jira"
    requires_consent = True
    risk_level = "medium"
```

---

#### 6.4.4 Quality Engineering: QE Tools

```python
# tools/quality_engineering/project/qe_project_analyze_test_failure.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
from llm.provider import LLMProvider
import logging

logger = logging.getLogger(__name__)

class QEProjectAnalyzeTestFailureTool(Tool):
    """Analyzes test failure logs to identify root causes."""

    def __init__(self, llm_provider: LLMProvider, config: Dict[str, Any]):
        super().__init__(config)
        self.llm = llm_provider

    @property
    def name(self) -> str:
        return "qe_project_analyze_test_failure"

    @property
    def display_name(self) -> str:
        return "Test Failure Analyzer"

    @property
    def description(self) -> str:
        return """Analyzes test failure logs, stack traces, and error messages to identify root
causes. Searches code for relevant functions, checks recent commits, and suggests
potential fixes based on failure patterns."""

    @property
    def purpose(self) -> str:
        return "quality_engineering"

    @property
    def technology(self) -> str:
        return "project_specific"

    @property
    def category(self) -> str:
        return "internal"

    @property
    def capabilities(self) -> List[str]:
        return [
            "analyze test failure logs",
            "parse stack traces",
            "identify test failure root cause",
            "suggest test fixes",
            "search code for error context"
        ]

    @property
    def state_inputs(self) -> List[str]:
        return ["test_failure_logs", "repository_url", "repository_version"]

    @property
    def state_outputs(self) -> List[str]:
        return ["failure_root_cause", "suggested_fixes", "related_code_snippets"]

    @property
    def requires_consent(self) -> bool:
        return False

    @property
    def risk_level(self) -> str:
        return "low"

    async def execute(
        self,
        state: SessionState,
        parameters: Dict[str, Any]
    ) -> ToolResult:
        """Analyze test failure."""
        start_time = datetime.utcnow()

        try:
            failure_logs = state.data.test_failure_logs

            # Use LLM to analyze failure logs
            analysis_prompt = f"""Analyze this test failure and provide:
1. Root cause
2. Affected component/module
3. Suggested fixes
4. Code areas to investigate

Test Failure Logs:
{failure_logs}
"""

            analysis_result = await self.llm.generate_analysis(analysis_prompt)

            # Write to state
            state.data.failure_root_cause = analysis_result.get("root_cause")
            state.data.suggested_fixes = analysis_result.get("suggested_fixes", [])
            state.data.related_code_snippets = analysis_result.get("code_snippets", [])

            logger.info(f"Test failure analysis complete")

            return ToolResult(
                success=True,
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000),
                output_data={"root_cause": state.data.failure_root_cause}
            )

        except Exception as e:
            logger.error(f"Test failure analysis failed: {e}")
            return ToolResult(
                success=False,
                error=str(e),
                execution_time_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )
```

---

## 6.5 LLM Provider Abstraction

### 6.5.1 Base LLM Provider Interface

```python
# llm/provider.py

from abc import ABC, abstractmethod
from typing import Dict, Any, List, Optional, AsyncIterator
from pydantic import BaseModel

class LLMProvider(ABC):
    """Base interface for all LLM providers."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.model = config.get("model")
        self.temperature = config.get("temperature", 0.3)
        self.max_tokens = config.get("max_tokens", 4096)
        self.timeout_seconds = config.get("timeout_seconds", 30)

    @abstractmethod
    async def generate_text(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None
    ) -> str:
        """
        Generate text completion.

        Args:
            system_prompt: System instructions
            user_message: User input
            temperature: Override default temperature
            max_tokens: Override default max tokens

        Returns:
            Generated text
        """
        pass

    @abstractmethod
    async def generate_structured_output(
        self,
        system_prompt: str,
        user_message: str,
        output_schema: type[BaseModel],
        temperature: Optional[float] = None
    ) -> Dict[str, Any]:
        """
        Generate structured output matching Pydantic schema.

        Args:
            system_prompt: System instructions
            user_message: User input
            output_schema: Pydantic model class for output structure
            temperature: Override default temperature

        Returns:
            Dict matching the output schema
        """
        pass

    @abstractmethod
    async def generate_with_streaming(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None
    ) -> AsyncIterator[str]:
        """
        Generate text with streaming (for thinking updates).

        Args:
            system_prompt: System instructions
            user_message: User input
            temperature: Override default temperature

        Yields:
            Text chunks as they're generated
        """
        pass

    @abstractmethod
    async def generate_plan(
        self,
        system_prompt: str,
        user_message: str,
        tools_schema: Dict[str, Any],
        thinking_callback: Optional[callable] = None
    ) -> Dict[str, Any]:
        """
        Generate execution plan with tool selection.

        Args:
            system_prompt: Planning instructions
            user_message: User request
            tools_schema: Available tools schema
            thinking_callback: Optional callback for thinking updates

        Returns:
            Plan dict with steps
        """
        pass

    async def generate_recovery_plan(
        self,
        recovery_prompt: str
    ) -> Dict[str, Any]:
        """
        Generate recovery plan after failure.

        Args:
            recovery_prompt: Context about failure

        Returns:
            Recovery strategy dict
        """
        # Default implementation using structured output
        return await self.generate_structured_output(
            system_prompt="You are an expert at diagnosing and recovering from failures.",
            user_message=recovery_prompt,
            output_schema=RecoveryPlan
        )

class RecoveryPlan(BaseModel):
    """Recovery plan schema."""
    strategy: str  # "retry" | "skip" | "abort" | "replan"
    modified_parameters: Optional[Dict[str, Any]] = None
    new_steps: Optional[List[Dict[str, Any]]] = None
    explanation: str
    user_notification: str
```

---

### 6.5.2 Gemini Provider Implementation

```python
# llm/gemini_provider.py

from llm.provider import LLMProvider
from typing import Dict, Any, Optional, AsyncIterator
from pydantic import BaseModel
import google.generativeai as genai
import json
import logging

logger = logging.getLogger(__name__)

class GeminiProvider(LLMProvider):
    """Google Gemini LLM provider."""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        api_key = config.get("api_key")
        genai.configure(api_key=api_key)

        self.client = genai.GenerativeModel(
            model_name=self.model,
            generation_config={
                "temperature": self.temperature,
                "max_output_tokens": self.max_tokens,
            }
        )

    async def generate_text(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None
    ) -> str:
        """Generate text using Gemini."""
        try:
            # Combine system and user prompts
            full_prompt = f"{system_prompt}\n\n{user_message}"

            response = await self.client.generate_content_async(
                full_prompt,
                generation_config={
                    "temperature": temperature or self.temperature,
                    "max_output_tokens": max_tokens or self.max_tokens
                }
            )

            return response.text

        except Exception as e:
            logger.error(f"Gemini generation failed: {e}")
            raise

    async def generate_structured_output(
        self,
        system_prompt: str,
        user_message: str,
        output_schema: type[BaseModel],
        temperature: Optional[float] = None
    ) -> Dict[str, Any]:
        """Generate structured JSON output."""
        try:
            # Add JSON schema to prompt
            schema_json = output_schema.model_json_schema()
            enhanced_prompt = f"""{system_prompt}

Output must be valid JSON matching this schema:
{json.dumps(schema_json, indent=2)}

User request:
{user_message}

Respond with ONLY valid JSON, no other text."""

            response_text = await self.generate_text(
                system_prompt="",
                user_message=enhanced_prompt,
                temperature=temperature
            )

            # Parse JSON
            # Handle markdown code blocks if present
            if "```json" in response_text:
                response_text = response_text.split("```json")[1].split("```")[0].strip()
            elif "```" in response_text:
                response_text = response_text.split("```")[1].split("```")[0].strip()

            return json.loads(response_text)

        except Exception as e:
            logger.error(f"Gemini structured output failed: {e}")
            raise

    async def generate_with_streaming(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None
    ) -> AsyncIterator[str]:
        """Stream text generation."""
        try:
            full_prompt = f"{system_prompt}\n\n{user_message}"

            response = await self.client.generate_content_async(
                full_prompt,
                generation_config={
                    "temperature": temperature or self.temperature,
                },
                stream=True
            )

            async for chunk in response:
                if chunk.text:
                    yield chunk.text

        except Exception as e:
            logger.error(f"Gemini streaming failed: {e}")
            raise

    async def generate_plan(
        self,
        system_prompt: str,
        user_message: str,
        tools_schema: Dict[str, Any],
        thinking_callback: Optional[callable] = None
    ) -> Dict[str, Any]:
        """Generate execution plan."""
        # For Gemini, we'll use structured output with planning schema
        plan_schema = {
            "steps": [{
                "tool_name": "string",
                "description": "string",
                "parameters": "object",
                "expected_output": "string",
                "risk_level": "string",
                "requires_consent": "boolean",
                "dependencies": ["string"]
            }],
            "estimated_duration": "string"
        }

        enhanced_prompt = f"""{system_prompt}

Available Tools:
{json.dumps(tools_schema, indent=2)}

User Request:
{user_message}

Create a step-by-step execution plan."""

        # Stream thinking process if callback provided
        if thinking_callback:
            await thinking_callback("Analyzing user request...")
            await thinking_callback("Selecting relevant tools...")
            await thinking_callback("Creating execution plan...")

        return await self.generate_structured_output(
            system_prompt="",
            user_message=enhanced_prompt,
            output_schema=type("PlanSchema", (BaseModel,), {
                "__annotations__": plan_schema
            })
        )
```

---

### 6.5.3 Ollama Provider Implementation

```python
# llm/ollama_provider.py

from llm.provider import LLMProvider
from typing import Dict, Any, Optional, AsyncIterator
from pydantic import BaseModel
import httpx
import json
import logging

logger = logging.getLogger(__name__)

class OllamaProvider(LLMProvider):
    """Ollama local LLM provider."""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        self.base_url = config.get("base_url", "http://localhost:11434")
        self.client = httpx.AsyncClient(timeout=self.timeout_seconds)

    async def generate_text(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None
    ) -> str:
        """Generate text using Ollama."""
        try:
            response = await self.client.post(
                f"{self.base_url}/api/generate",
                json={
                    "model": self.model,
                    "prompt": f"{system_prompt}\n\n{user_message}",
                    "stream": False,
                    "options": {
                        "temperature": temperature or self.temperature,
                        "num_predict": max_tokens or self.max_tokens
                    }
                }
            )
            response.raise_for_status()

            return response.json()["response"]

        except Exception as e:
            logger.error(f"Ollama generation failed: {e}")
            raise

    async def generate_structured_output(
        self,
        system_prompt: str,
        user_message: str,
        output_schema: type[BaseModel],
        temperature: Optional[float] = None
    ) -> Dict[str, Any]:
        """Generate structured JSON output."""
        try:
            schema_json = output_schema.model_json_schema()
            enhanced_prompt = f"""{system_prompt}

Output must be valid JSON matching this schema:
{json.dumps(schema_json, indent=2)}

User request:
{user_message}

Respond with ONLY valid JSON, no other text."""

            response_text = await self.generate_text(
                system_prompt="",
                user_message=enhanced_prompt,
                temperature=temperature
            )

            # Parse JSON (handle markdown if present)
            if "```json" in response_text:
                response_text = response_text.split("```json")[1].split("```")[0].strip()
            elif "```" in response_text:
                response_text = response_text.split("```")[1].split("```")[0].strip()

            return json.loads(response_text)

        except Exception as e:
            logger.error(f"Ollama structured output failed: {e}")
            raise

    async def generate_with_streaming(
        self,
        system_prompt: str,
        user_message: str,
        temperature: Optional[float] = None
    ) -> AsyncIterator[str]:
        """Stream text generation."""
        try:
            async with self.client.stream(
                "POST",
                f"{self.base_url}/api/generate",
                json={
                    "model": self.model,
                    "prompt": f"{system_prompt}\n\n{user_message}",
                    "stream": True,
                    "options": {
                        "temperature": temperature or self.temperature
                    }
                }
            ) as response:
                async for line in response.aiter_lines():
                    if line:
                        chunk = json.loads(line)
                        if "response" in chunk:
                            yield chunk["response"]

        except Exception as e:
            logger.error(f"Ollama streaming failed: {e}")
            raise

    async def generate_plan(
        self,
        system_prompt: str,
        user_message: str,
        tools_schema: Dict[str, Any],
        thinking_callback: Optional[callable] = None
    ) -> Dict[str, Any]:
        """Generate execution plan."""
        # Similar to Gemini implementation
        pass
```

---

### 6.5.4 LLM Provider Manager

```python
# llm/manager.py

from llm.provider import LLMProvider
from llm.gemini_provider import GeminiProvider
from llm.ollama_provider import OllamaProvider
from typing import Dict, Any, Optional
import yaml
import logging

logger = logging.getLogger(__name__)

class LLMProviderManager:
    """
    Manages multiple LLM providers and selects appropriate one per request.

    Handles:
    - Loading provider configurations
    - Instantiating providers
    - Selecting provider based on tool preferences
    - Fallback on errors
    """

    def __init__(self, config_path: str = "config/llm_config.yaml"):
        self.config = self._load_config(config_path)
        self.providers: Dict[str, LLMProvider] = {}
        self._initialize_providers()

    def _load_config(self, config_path: str) -> Dict[str, Any]:
        """Load LLM configuration."""
        with open(config_path, 'r') as f:
            return yaml.safe_load(f).get("llm_config", {})

    def _initialize_providers(self):
        """Initialize all configured LLM providers."""
        provider_configs = self.config.get("providers", {})

        for provider_name, provider_config in provider_configs.items():
            try:
                provider_type = provider_config.get("provider")

                if provider_type == "gemini":
                    self.providers[provider_name] = GeminiProvider(provider_config)
                elif provider_type == "ollama":
                    self.providers[provider_name] = OllamaProvider(provider_config)
                else:
                    logger.warning(f"Unknown provider type: {provider_type}")

                logger.info(f"Initialized LLM provider: {provider_name} ({provider_type}/{provider_config.get('model')})")

            except Exception as e:
                logger.error(f"Failed to initialize provider {provider_name}: {e}")

    def get_provider(
        self,
        provider_name: Optional[str] = None,
        tool_name: Optional[str] = None,
        use_case: str = "tool_default"
    ) -> LLMProvider:
        """
        Get LLM provider based on preferences.

        Priority:
        1. Explicit provider_name parameter
        2. Tool-specific override from config
        3. Use case default (planner, tool_default, recovery_planner)
        4. First available provider

        Args:
            provider_name: Explicit provider name
            tool_name: Tool requesting LLM (for overrides)
            use_case: Use case type (planner, tool_default, recovery_planner)

        Returns:
            LLMProvider instance
        """
        selected_provider_name = None

        # Priority 1: Explicit provider
        if provider_name:
            selected_provider_name = provider_name

        # Priority 2: Tool-specific override
        elif tool_name:
            tool_overrides = self.config.get("tool_overrides", {})
            if tool_name in tool_overrides:
                selected_provider_name = tool_overrides[tool_name]
                logger.debug(f"Using tool override for {tool_name}: {selected_provider_name}")

        # Priority 3: Use case default
        if not selected_provider_name:
            defaults = self.config.get("defaults", {})
            selected_provider_name = defaults.get(use_case, defaults.get("tool_default"))

        # Priority 4: First available
        if not selected_provider_name or selected_provider_name not in self.providers:
            selected_provider_name = list(self.providers.keys())[0] if self.providers else None

        if not selected_provider_name:
            raise ValueError("No LLM providers available")

        logger.debug(f"Selected LLM provider: {selected_provider_name} (use_case={use_case}, tool={tool_name})")
        return self.providers[selected_provider_name]

    def get_provider_with_fallback(
        self,
        provider_name: Optional[str] = None,
        tool_name: Optional[str] = None,
        use_case: str = "tool_default"
    ) -> tuple[LLMProvider, Optional[LLMProvider]]:
        """
        Get primary provider and fallback provider.

        Returns:
            Tuple of (primary_provider, fallback_provider or None)
        """
        primary = self.get_provider(provider_name, tool_name, use_case)

        fallback_config = self.config.get("fallback", {})
        if fallback_config.get("enabled"):
            fallback_name = fallback_config.get("provider")
            fallback = self.providers.get(fallback_name)
            return (primary, fallback)

        return (primary, None)

    def get_planner_provider(self) -> LLMProvider:
        """Get LLM provider for planning."""
        return self.get_provider(use_case="planner")

    def get_recovery_provider(self) -> LLMProvider:
        """Get LLM provider for recovery planning."""
        return self.get_provider(use_case="recovery_planner")
```

---

## 7. Frontend Implementation

### 7.1 Technology Stack

```
Frontend:
├── Framework: React 18+ with TypeScript
├── Routing: React Router v6
├── State Management: React Context + useReducer
├── Styling: Tailwind CSS
├── WebSocket: Native WebSocket API
├── HTTP Client: Axios
├── Markdown: react-markdown
└── Build Tool: Vite
```

---

### 7.2 Project Structure

```
frontend/
├── public/
│   ├── index.html
│   └── favicon.ico
├── src/
│   ├── main.tsx                 # App entry point
│   ├── App.tsx                  # Root component with routing
│   ├── config/
│   │   └── branding.ts          # Branding config loader
│   ├── components/
│   │   ├── chat/
│   │   │   ├── ChatWindow.tsx           # Main chat container (3-panel layout)
│   │   │   ├── ChatHeader.tsx           # Header with logo, title, status indicators
│   │   │   ├── MessageList.tsx          # Scrollable message container
│   │   │   ├── MessageItem.tsx          # Individual message bubble (markdown support)
│   │   │   ├── WelcomeMessage.tsx       # Welcome banner with emoji
│   │   │   ├── InputBar.tsx             # Message input with Send button
│   │   │   ├── ConsentDialog.tsx        # Bottom consent approval dialog
│   │   │   ├── FeedbackButtons.tsx      # Thumbs up/down buttons (per message)
│   │   │   ├── AgentReasoningPanel.tsx  # Right sidebar - reasoning steps
│   │   │   ├── ExecutionPlanPanel.tsx   # Right sidebar - plan visualization
│   │   │   ├── PlanStepCard.tsx         # Individual step in execution plan
│   │   │   ├── StepStatusIndicator.tsx  # Checkmark/loading/error icon
│   │   │   ├── StepErrorActions.tsx     # Report Issue/Retry buttons
│   │   │   └── StatusBadge.tsx          # Connected/Analyzing badge
│   │   ├── dashboard/
│   │   │   ├── DashboardContainer.tsx   # Main dashboard container
│   │   │   ├── DashboardHeader.tsx      # Header with user info
│   │   │   ├── TimeFilterButtons.tsx    # Last 7/30 Days, All Time
│   │   │   ├── AnalyticsCards.tsx       # 4 metric cards container
│   │   │   ├── MetricCard.tsx           # Individual metric card
│   │   │   ├── SessionsTable.tsx        # Recent sessions table
│   │   │   ├── SessionRow.tsx           # Individual session row
│   │   │   ├── ToolPerformanceChart.tsx # Horizontal bar chart (success rate)
│   │   │   ├── FeedbackChart.tsx        # Feedback distribution chart
│   │   │   └── StarRating.tsx           # Star rating display component
│   │   ├── common/
│   │   │   ├── ErrorBoundary.tsx
│   │   │   ├── LoadingSpinner.tsx
│   │   │   ├── UserAvatar.tsx           # User avatar with initials
│   │   │   ├── ProgressIndicator.tsx    # Step progress (4/6 - 1 failed)
│   │   │   └── RiskBadge.tsx            # MEDIUM/HIGH/LOW RISK badge
│   │   └── layout/
│   │       ├── MainLayout.tsx
│   │       └── ThreePanelLayout.tsx     # Chat 3-panel layout wrapper
│   ├── hooks/
│   │   ├── useWebSocket.ts      # WebSocket connection hook
│   │   ├── useSession.ts        # Session state management
│   │   ├── useDashboard.ts      # Dashboard data fetching
│   │   └── useBranding.ts       # Branding config hook
│   ├── types/
│   │   ├── session.ts           # Session-related types
│   │   ├── message.ts           # Message types
│   │   ├── plan.ts              # Plan & PlanStep types
│   │   ├── tool.ts              # Tool types
│   │   └── branding.ts          # Branding config types
│   ├── services/
│   │   ├── websocket.ts         # WebSocket service
│   │   ├── api.ts               # REST API client
│   │   └── storage.ts           # LocalStorage utilities
│   ├── context/
│   │   ├── SessionContext.tsx   # Session state context
│   │   └── BrandingContext.tsx  # Branding context
│   ├── pages/
│   │   ├── ChatPage.tsx         # Chat interface page
│   │   └── DashboardPage.tsx    # Dashboard page
│   ├── utils/
│   │   ├── formatters.ts        # Date/time formatters
│   │   └── validators.ts        # Input validators
│   └── styles/
│       └── globals.css          # Global styles
├── branding_config.yaml         # White-label configuration
├── package.json
├── tsconfig.json
├── vite.config.ts
└── tailwind.config.js
```

---

### 7.3 TypeScript Models

```typescript
// frontend/src/types/session.ts

export interface Message {
  message_id: string;
  role: "user" | "assistant" | "system";
  content: string;
  timestamp: string;
  metadata?: Record<string, any>;
}

export interface PlanStep {
  step_id: string;
  order: number;
  tool_name: string;
  tool_description: string;
  parameters: Record<string, any>;
  expected_output: string;
  risk_level: "low" | "medium" | "high";
  requires_individual_consent: boolean;
  dependencies: string[];
  status: "pending" | "approved" | "rejected" | "executing" | "completed" | "failed" | "skipped";
}

export interface Plan {
  plan_id: string;
  user_prompt: string;
  thinking_process: string[];
  steps: PlanStep[];
  estimated_duration: string;
  requires_consent: boolean;
  created_at: string;
}

export interface ConsentRequest {
  request_id: string;
  plan_id: string;
  step_id: string;
  tool_name: string;
  tool_description: string;
  parameters_summary: string;
  risk_level: "low" | "medium" | "high";
  reversible: boolean;
  timeout_seconds: number;
  bulk_options: {
    approve_all_risks: boolean;
    approve_low_risks: boolean;
    approve_low_medium_risks: boolean;
  };
}

export interface ConsentResponse {
  request_id: string;
  decision: "approved" | "rejected" | "modified" | "bulk_approve_all" | "bulk_approve_low" | "bulk_approve_low_medium";
  modified_parameters?: Record<string, any>;
  user_feedback?: string;
}

export interface SessionState {
  session_id: string;
  user_id?: string;
  messages: Message[];
  current_plan?: Plan;
  execution_status: "idle" | "planning" | "executing" | "waiting_consent" | "completed" | "failed" | "timeout";
  pending_consent?: ConsentRequest;
}

// WebSocket message types
export interface WSMessage {
  type: string;
  payload: any;
}

export interface ThinkingUpdatePayload {
  step: string;
}

export interface StepStartedPayload {
  step_id: string;
  tool_name: string;
}

export interface StepProgressPayload {
  step_id: string;
  progress: string;
}

export interface StepCompletedPayload {
  step_id: string;
  success: boolean;
  result?: any;
  error?: string;
}
```

---

### 7.4 UI Layout & Component Specifications

Based on the design mockups (chatpanel.png and dashboard.png), here are the detailed UI specifications:

#### 7.4.1 Chat Page Layout (Three-Panel Design)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [Red Hat Logo]  OCP Sustaining Bot          🔄 Analyzing... (46s)  🟢 Connected│
│                  Agentic Team Bot                                               │
├────────────────────────────────────┬────────────────────────────────────────────┤
│                                    │  🤖 Agent Reasoning                   ▼    │
│  👋 Welcome! I can help you...     │  ✓ Extracted CVE ID and repository info    │
│                                    │  ✓ Fetched CVE details from OSV database   │
│ ┌────────────────────────────────┐ │  ✓ Cloned repository and analyzed go.mod  │
│ │ [User Prompt Bubble]           │ │  🔴 Checking if affected packages are...  │
│ │ Analyze CVE-2024-24786 for...│ │                                            │
│ └────────────────────────────────┘ │ ─────────────────────────────────────────  │
│                                    │  📋 Execution Plan            4/6 - 1 failed│
│ 👤 10:23:45                        │                                       ▼    │
│ I'll analyze this CVE for you...  │  ┌──────────────────────────────────────┐  │
│                                    │  │ ✓ Fetch CVE Data                    │  │
│ 🤖 10:23:52                        │  │   Retrieved vulnerability details    │  │
│ I've retrieved CVE-2024-24786.    │  └──────────────────────────────────────┘  │
│ It's a HIGH severity issue        │  ┌──────────────────────────────────────┐  │
│ (CVSS 7.5) affecting protobuf...  │  │ ✓ Clone Repository                  │  │
│                       👍  👎      │  │   Cloned openshift/cluster-nfd-op... │  │
│                                    │  └──────────────────────────────────────┘  │
│ 🤖 10:24:01                        │  ┌──────────────────────────────────────┐  │
│ I've cloned your repository and   │  │ ✓ Assess CVE Impact                 │  │
│ analyzed the go.mod file...       │  │   Analyzed if vulnerable packages... │  │
│                       👍  👎      │  └──────────────────────────────────────┘  │
│                                    │  ┌──────────────────────────────────────┐  │
│ 🤖 Applying fix to go.mod...       │  │ ⟳ Apply Fix                        │  │
│    ⟳ [Spinner]                    │  │   Updating dependencies...          │  │
│                                    │  └──────────────────────────────────────┘  │
│                                    │  ┌──────────────────────────────────────┐  │
│                                    │  │ ✗ Test Fix                          │  │
│                                    │  │   Run build and tests to validate   │  │
│ ┌────────────────────────────────┐ │  │   ⚠️ Build failed: undefined ref... │  │
│ │ Type your request here... 📤Send│ │  │        🔴 Report Issue  🔁 Retry   │  │
│ └────────────────────────────────┘ │  └──────────────────────────────────────┘  │
├────────────────────────────────────┤  ┌──────────────────────────────────────┐  │
│ ⚠️ Approve "Apply Fix" to go.mod? │  │ ○ Create PR                         │  │
│ This will update google.golang...  │  │   Generate pull request with fix    │  │
│ [Reject] [Approve All] [Approve]   │  └──────────────────────────────────────┘  │
│ MEDIUM RISK                        │                                            │
└────────────────────────────────────┴────────────────────────────────────────────┘
```

**Layout Proportions:**
- Left panel (main chat): ~65% width
- Right panel (reasoning + plan): ~35% width
- Header: Fixed height ~60px
- Consent dialog: Fixed bottom position when active

**Key Components:**

1. **ChatHeader:**
   - Logo + title (left)
   - Status indicators: "Analyzing... (46s)", "Connected" (right)
   - Background: white with subtle border

2. **MessageList:**
   - Alternating user/bot messages
   - User messages: Pink/red bubbles, right-aligned
   - Bot messages: White/gray bubbles, left-aligned with markdown rendering
   - Timestamps with avatar icons
   - **Spinner/Loading Indicator:** During step execution
     - Shows blue bubble with spinning icon and step description
     - Example: "🤖 Fetching CVE vulnerability data... ⟳"
     - Automatically replaced by moderated message when step completes
   - **Important:** Bot responses are LLM-moderated messages from ResponseModerator
     - After each step: Brief progress update (e.g., "I've fetched the CVE details...")
     - After plan completion: Comprehensive answer to user's question
   - Feedback buttons (thumbs up/down) shown per assistant message

3. **ConsentDialog:**
   - Fixed bottom position, full width
   - Yellow/warning background
   - Risk level badge (MEDIUM/HIGH/LOW RISK)
   - Three action buttons: Reject, Approve All, Approve
   - Shows what will be modified

4. **AgentReasoningPanel:**
   - Collapsible section with robot emoji
   - Live-updating reasoning steps
   - Green checkmarks for completed steps
   - Red dot with loading animation for current step

5. **ExecutionPlanPanel:**
   - Collapsible section with document emoji
   - Progress indicator: "4/6 - 1 failed"
   - Color-coded step cards:
     - Green border + checkmark: completed
     - Red border + X: failed
     - Gray + empty circle: pending
   - Failed steps show error message + action buttons
   - Each step shows description/subtitle

**Response Flow Architecture:**

The chat panel displays **LLM-moderated responses** rather than raw tool outputs. This provides:

1. **Step-Level Updates** (during execution):
   - After each tool executes, ResponseModerator calls LLM to generate a brief, conversational update
   - Example: After `cve_go_fetch_vulnerability_data` executes → "I've retrieved CVE-2024-24786. It's a HIGH severity issue..."
   - These updates keep the user informed of progress in natural language
   - Each update gets its own message bubble with feedback buttons

2. **Plan-Level Final Response** (after all steps complete):
   - ResponseModerator takes all step results + original user question
   - Calls LLM to synthesize a comprehensive, well-formatted answer
   - Response directly addresses the user's original question
   - Uses markdown formatting for clarity (headers, lists, code blocks, etc.)
   - Example structure:
     ```markdown
     ## CVE Analysis Complete

     I've analyzed CVE-2024-24786...

     ### Vulnerability Summary
     - **CVE ID:** CVE-2024-24786
     - **Severity:** HIGH (CVSS 7.5)

     ### Repository Status
     ✅ The repository IS vulnerable

     ### Next Steps
     ...
     ```

**Benefits:**
- Generic frontend components (not task-specific like CVEResultsCard)
- Better UX with context-aware, conversational responses
- LLM handles formatting, emphasis, and tone
- Same architecture works for CVE analysis, Jira queries, documentation, etc.
- Clean separation: Executor handles orchestration, ResponseModerator handles presentation

---

#### 7.4.2 Dashboard Page Layout

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  OCP Sustaining Bot                         [AD] Admin User  🟢 Connected       │
│  Dashboard                                      Admin                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  Session History & Analytics          [Last 7 Days] [Last 30 Days] [All Time] │
│                                                                                 │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐      │
│  │Total Sessions │ │ Success Rate  │ │  Avg Rating   │ │ Avg Duration  │      │
│  │     147       │ │     89%       │ │   ⭐ 4.2      │ │   2m 34s      │      │
│  │ ↑ 12% from... │ │ ↑ 3% from...  │ │ Based on 98.. │ │ ↓ 15s from... │      │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘      │
│                                                                                 │
│  Recent Sessions                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ TIME         REQUEST                           STATUS   RATING  DURATION │   │
│  ├─────────────────────────────────────────────────────────────────────────┤   │
│  │ 2 hours ago  Analyze CVE-2024-24786 for...  ✓ Completed ⭐⭐⭐⭐⭐ 1m 42s  5│   │
│  │ 3 hours ago  Check CVE-2024-1234 for...     ✗ Failed    ⭐⭐⭐⭐⭐ 0m 58s  3│   │
│  │ 5 hours ago  Remediate golang.org/x/net...  ✓ Completed ⭐⭐⭐⭐⭐ 3m 12s  6│   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────┐ ┌─────────────────────────────────┐  │
│  │ Tool Performance (Success Rate)     │ │ Feedback Distribution           │  │
│  │                                     │ │                                 │  │
│  │ fetch_cve_data    ████████████ 95% │ │ ⭐⭐⭐⭐⭐  ██████████████  45%   │  │
│  │ clone_repository  ███████████  92% │ │ ⭐⭐⭐⭐    ████████       28%   │  │
│  │ assess_cve        ██████████   88% │ │ ⭐⭐⭐      ████          18%   │  │
│  │ apply_fix         █████████    82% │ │ ⭐⭐        ██            9%   │  │
│  └─────────────────────────────────────┘ └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **DashboardHeader:**
   - Title + subtitle (left)
   - User avatar with initials, name, role, connection status (right)

2. **TimeFilterButtons:**
   - Three buttons: "Last 7 Days", "Last 30 Days", "All Time"
   - Active button: Red/primary color
   - Inactive: Gray outline

3. **AnalyticsCards (4 cards):**
   - Total Sessions: Large number with trend arrow
   - Success Rate: Percentage with green trend
   - Avg Rating: Star icon with number
   - Avg Duration: Time with trend arrow
   - Each card shows comparison text: "↑ 12% from last week"

4. **SessionsTable:**
   - Headers: TIME | REQUEST | STATUS | RATING | DURATION | TOOLS USED
   - Status column: Green "✓ Completed" or Red "✗ Failed"
   - Rating: 5-star display (filled stars)
   - Truncated request text with ellipsis
   - Alternating row backgrounds for readability

5. **ToolPerformanceChart:**
   - Horizontal bar chart
   - Tool name (left), bar (middle), percentage (right)
   - Green bars with varying lengths
   - Shows success rate per tool

6. **FeedbackChart:**
   - Horizontal bar chart
   - Star rating (left), bar (middle), percentage (right)
   - Orange/yellow bars
   - Distribution of 5-star to 1-star ratings

---

### 7.5 WebSocket Hook

```typescript
// frontend/src/hooks/useWebSocket.ts

import { useEffect, useRef, useState } from 'react';
import { WSMessage } from '../types/session';

export const useWebSocket = (
  sessionId: string,
  sessionToken: string,
  onMessage: (message: WSMessage) => void
) => {
  const wsRef = useRef<WebSocket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const wsUrl = `${process.env.REACT_APP_WS_URL}/ws/${sessionId}?token=${sessionToken}`;
    const ws = new WebSocket(wsUrl);

    ws.onopen = () => {
      console.log('WebSocket connected');
      setIsConnected(true);
    };

    ws.onmessage = (event) => {
      const message: WSMessage = JSON.parse(event.data);
      onMessage(message);

      // Handle ping/pong
      if (message.type === 'ping') {
        sendMessage('pong', {});
      }
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    ws.onclose = () => {
      console.log('WebSocket disconnected');
      setIsConnected(false);
      // Auto-reconnect logic could go here
    };

    wsRef.current = ws;

    return () => {
      ws.close();
    };
  }, [sessionId, sessionToken]);

  const sendMessage = (type: string, payload: any) => {
    if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ type, payload }));
    }
  };

  return { isConnected, sendMessage };
};
```

---

### 7.6 Key Component Implementations

#### 7.6.1 Chat Window with Three-Panel Layout

```typescript
// frontend/src/components/chat/ChatWindow.tsx

import React from 'react';
import ChatHeader from './ChatHeader';
import MessageList from './MessageList';
import InputBar from './InputBar';
import ConsentDialog from './ConsentDialog';
import AgentReasoningPanel from './AgentReasoningPanel';
import ExecutionPlanPanel from './ExecutionPlanPanel';
import { useSession } from '../../hooks/useSession';

export const ChatWindow: React.FC = () => {
  const { session, messages, plan, pendingConsent, executingStep } = useSession();

  return (
    <div className="flex flex-col h-screen">
      <ChatHeader
        status={session.status}
        analyzing={session.current_step !== null}
      />

      <div className="flex flex-1 overflow-hidden">
        {/* Main Chat Area - 65% */}
        <div className="flex-1 flex flex-col w-2/3 border-r">
          <MessageList
            messages={messages}
            executingStep={executingStep}
          />
          <InputBar disabled={!!pendingConsent} />
          {pendingConsent && (
            <ConsentDialog consent={pendingConsent} />
          )}
        </div>

        {/* Right Sidebar - 35% */}
        <div className="w-1/3 flex flex-col overflow-y-auto bg-gray-50 p-4 space-y-4">
          <AgentReasoningPanel
            thinkingSteps={session.thinking_steps}
          />
          <ExecutionPlanPanel
            plan={plan}
            currentStep={session.current_step}
          />
        </div>
      </div>
    </div>
  );
};
```

#### 7.6.2 Message Item with Markdown Support

**Note:** All assistant messages are LLM-moderated responses from the backend's ResponseModerator.
The ResponseModerator converts raw tool results into human-readable messages with proper formatting.

```typescript
// frontend/src/components/chat/MessageItem.tsx

import React from 'react';
import ReactMarkdown from 'react-markdown';
import FeedbackButtons from './FeedbackButtons';
import { Message } from '../../types/message';

interface MessageItemProps {
  message: Message;
}

export const MessageItem: React.FC<MessageItemProps> = ({ message }) => {
  const isUser = message.role === 'user';
  const isAssistant = message.role === 'assistant';

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'} mb-4`}>
      <div className={`max-w-3xl ${isUser ? 'bg-red-500 text-white' : 'bg-white'} rounded-lg shadow-md p-4`}>
        {/* Avatar and timestamp */}
        <div className="flex items-center mb-2">
          <span className="text-sm opacity-75">
            {isUser ? '👤' : '🤖'} {new Date(message.timestamp).toLocaleTimeString()}
          </span>
        </div>

        {/* Message content with markdown support */}
        <div className="prose prose-sm max-w-none">
          {isAssistant ? (
            <ReactMarkdown
              components={{
                // Custom rendering for code blocks
                code: ({ inline, children, ...props }) => (
                  inline
                    ? <code className="bg-gray-100 px-1 rounded text-xs" {...props}>{children}</code>
                    : <pre className="bg-gray-900 text-gray-100 p-3 rounded overflow-x-auto"><code {...props}>{children}</code></pre>
                ),
                // Custom rendering for lists
                ul: ({ children }) => <ul className="list-disc pl-5 space-y-1">{children}</ul>,
                ol: ({ children }) => <ol className="list-decimal pl-5 space-y-1">{children}</ol>,
                // Custom rendering for headings
                h1: ({ children }) => <h1 className="text-xl font-bold mt-3 mb-2">{children}</h1>,
                h2: ({ children }) => <h2 className="text-lg font-semibold mt-2 mb-1">{children}</h2>,
                h3: ({ children }) => <h3 className="text-md font-medium mt-2 mb-1">{children}</h3>,
              }}
            >
              {message.content}
            </ReactMarkdown>
          ) : (
            <p>{message.content}</p>
          )}
        </div>

        {/* Feedback buttons for assistant messages */}
        {isAssistant && (
          <FeedbackButtons messageId={message.message_id} />
        )}
      </div>
    </div>
  );
};
```

**Example moderated responses from ResponseModerator:**

1. **Step-level response** (after CVE fetch step):
```markdown
I've retrieved the vulnerability details for CVE-2024-24786. It's a **HIGH severity** issue
(CVSS 7.5) affecting `google.golang.org/protobuf` before version v1.33.0. The vulnerability
involves infinite recursion in the unmarshal function.
```

2. **Plan-level final response** (after full CVE analysis):
```markdown
## CVE Analysis Complete

I've analyzed CVE-2024-24786 for your repository `openshift/cluster-nfd-operator` on branch
`release-4.15`.

### Vulnerability Summary
- **CVE ID:** CVE-2024-24786
- **Severity:** HIGH (CVSS 7.5)
- **Affected Package:** google.golang.org/protobuf
- **Current Version:** v1.31.0 ⚠️
- **Fixed Version:** v1.33.0

### Repository Status
✅ **The repository IS vulnerable** - the affected package is actively used in your codebase.

### Remediation Applied
I've updated your `go.mod` file to use the patched version `v1.33.0`. However, the build tests
failed due to an undefined reference in `proto.Marshal`.

### Next Steps
1. **Review the build error** - There may be a breaking change in the protobuf upgrade
2. **Choose an action:**
   - Click "Retry" to attempt the build again
   - Click "Report Issue" to create a detailed issue report
   - I can create a pull request with the current fix for manual review

Would you like me to proceed with creating a PR?
```

---

#### 7.6.3 Thinking Indicator (Step Execution Spinner)

Shows a loading spinner when a step is currently executing, before the moderated response appears.

```typescript
// frontend/src/components/chat/ThinkingIndicator.tsx

import React from 'react';

interface ThinkingIndicatorProps {
  stepDescription: string;
  toolName?: string;
}

export const ThinkingIndicator: React.FC<ThinkingIndicatorProps> = ({
  stepDescription,
  toolName
}) => {
  return (
    <div className="flex justify-start mb-4">
      <div className="max-w-3xl bg-blue-50 rounded-lg shadow-md p-4 border border-blue-200">
        <div className="flex items-center space-x-3">
          {/* Spinning loader icon */}
          <div className="flex-shrink-0">
            <svg
              className="animate-spin h-5 w-5 text-blue-600"
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
            >
              <circle
                className="opacity-25"
                cx="12"
                cy="12"
                r="10"
                stroke="currentColor"
                strokeWidth="4"
              />
              <path
                className="opacity-75"
                fill="currentColor"
                d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
              />
            </svg>
          </div>

          {/* Step description */}
          <div className="flex-1">
            <div className="flex items-center">
              <span className="text-sm text-blue-800 font-medium">
                🤖 {stepDescription}
              </span>
            </div>
            {toolName && (
              <div className="text-xs text-blue-600 mt-1">
                Executing: {toolName}
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

#### 7.6.4 Message List with Spinner

Shows messages and displays a thinking indicator when a step is executing.

```typescript
// frontend/src/components/chat/MessageList.tsx

import React, { useEffect, useRef } from 'react';
import MessageItem from './MessageItem';
import ThinkingIndicator from './ThinkingIndicator';
import WelcomeMessage from './WelcomeMessage';
import { Message } from '../../types/message';

interface ExecutingStepInfo {
  step_id: string;
  description: string;
  tool_name: string;
}

interface MessageListProps {
  messages: Message[];
  executingStep?: ExecutingStepInfo | null;
}

export const MessageList: React.FC<MessageListProps> = ({
  messages,
  executingStep
}) => {
  const messagesEndRef = useRef<HTMLDivElement>(null);

  // Auto-scroll to bottom when new messages arrive
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, executingStep]);

  return (
    <div className="flex-1 overflow-y-auto p-4 bg-gray-50">
      {/* Show welcome message if no messages yet */}
      {messages.length === 0 && <WelcomeMessage />}

      {/* Render all messages */}
      {messages.map((message) => (
        <MessageItem key={message.message_id} message={message} />
      ))}

      {/* Show thinking indicator if step is executing */}
      {executingStep && (
        <ThinkingIndicator
          stepDescription={executingStep.description}
          toolName={executingStep.tool_name}
        />
      )}

      {/* Scroll anchor */}
      <div ref={messagesEndRef} />
    </div>
  );
};
```

**How it works:**

1. **Step starts executing:**
   - Backend sends `step_started` WebSocket message
   - Frontend stores executingStep in session state
   - ThinkingIndicator appears with spinner and step description

2. **Step completes:**
   - Backend sends moderated response via `assistant_message` WebSocket message
   - Frontend clears executingStep and adds message to messages array
   - ThinkingIndicator is replaced by the MessageItem with the response

3. **Visual flow:**
   ```
   User: "Analyze CVE-2024-24786..."

   [Spinner] 🤖 Fetching CVE vulnerability data...
            Executing: cve_go_fetch_vulnerability_data

   → (After step completes, spinner is replaced with:)

   🤖 I've retrieved CVE-2024-24786. It's a HIGH severity issue...
      [👍] [👎]
   ```

---

#### 7.6.5 Execution Plan Panel with Step Cards

```typescript
// frontend/src/components/chat/ExecutionPlanPanel.tsx

import React, { useState } from 'react';
import PlanStepCard from './PlanStepCard';
import { Plan } from '../../types/plan';

interface ExecutionPlanPanelProps {
  plan: Plan | null;
  currentStep: number | null;
}

export const ExecutionPlanPanel: React.FC<ExecutionPlanPanelProps> = ({
  plan, currentStep
}) => {
  const [collapsed, setCollapsed] = useState(false);

  if (!plan) return null;

  const completedSteps = plan.steps.filter(s => s.status === 'completed').length;
  const failedSteps = plan.steps.filter(s => s.status === 'failed').length;

  return (
    <div className="bg-white rounded-lg shadow-sm">
      <div
        className="flex items-center justify-between p-3 cursor-pointer border-b"
        onClick={() => setCollapsed(!collapsed)}
      >
        <div className="flex items-center">
          <span className="mr-2">📋</span>
          <h3 className="font-semibold text-gray-800">Execution Plan</h3>
        </div>
        <div className="flex items-center space-x-2">
          <span className="text-sm text-gray-600">
            {completedSteps}/{plan.steps.length}
            {failedSteps > 0 && ` - ${failedSteps} failed`}
          </span>
          <span className="text-gray-400">{collapsed ? '▼' : '▲'}</span>
        </div>
      </div>

      {!collapsed && (
        <div className="p-3 space-y-3">
          {plan.steps.map((step, index) => (
            <PlanStepCard
              key={step.step_id}
              step={step}
              isActive={currentStep === index}
            />
          ))}
        </div>
      )}
    </div>
  );
};
```

#### 7.6.6 Plan Step Card with Status

```typescript
// frontend/src/components/chat/PlanStepCard.tsx

import React from 'react';
import StepStatusIndicator from './StepStatusIndicator';
import StepErrorActions from './StepErrorActions';
import { PlanStep } from '../../types/plan';

interface PlanStepCardProps {
  step: PlanStep;
  isActive: boolean;
}

export const PlanStepCard: React.FC<PlanStepCardProps> = ({ step, isActive }) => {
  const borderColors = {
    completed: 'border-l-4 border-green-500 bg-green-50',
    failed: 'border-l-4 border-red-500 bg-red-50',
    in_progress: 'border-l-4 border-blue-500 bg-blue-50',
    pending: 'border-l-4 border-gray-300 bg-gray-50',
  };

  return (
    <div className={`rounded-r-lg p-3 ${borderColors[step.status]}`}>
      <div className="flex items-start">
        <StepStatusIndicator status={step.status} isActive={isActive} />
        <div className="ml-3 flex-1">
          <h4 className="font-medium text-sm text-gray-800">{step.description}</h4>
          {step.result && (
            <p className="text-xs text-gray-600 mt-1">{step.result.summary}</p>
          )}

          {step.status === 'failed' && step.result?.error && (
            <div className="mt-2">
              <div className="flex items-center text-xs text-red-600">
                <span className="mr-1">⚠️</span>
                <span>{step.result.error}</span>
              </div>
              <StepErrorActions stepId={step.step_id} />
            </div>
          )}
        </div>
      </div>
    </div>
  );
};
```

#### 7.6.7 Consent Dialog

```typescript
// frontend/src/components/chat/ConsentDialog.tsx

import React from 'react';
import RiskBadge from '../common/RiskBadge';
import { ConsentRequest } from '../../types/session';
import { useSession } from '../../hooks/useSession';

interface ConsentDialogProps {
  consent: ConsentRequest;
}

export const ConsentDialog: React.FC<ConsentDialogProps> = ({ consent }) => {
  const { respondToConsent } = useSession();

  return (
    <div className="border-t border-gray-300 bg-yellow-50 p-4">
      <div className="flex items-center justify-between">
        <div className="flex-1">
          <div className="flex items-center mb-1">
            <span className="mr-2">⚠️</span>
            <h3 className="font-semibold text-gray-800">
              Approve "{consent.action}" to {consent.target}?
            </h3>
          </div>
          <p className="text-sm text-gray-600 ml-6">
            {consent.description}
          </p>
        </div>

        <div className="flex items-center space-x-2 ml-4">
          <RiskBadge level={consent.risk_level} />
          <button
            onClick={() => respondToConsent('reject')}
            className="px-4 py-2 border border-gray-300 rounded-md text-sm font-medium text-gray-700 hover:bg-gray-50"
          >
            Reject
          </button>
          <button
            onClick={() => respondToConsent('approve_all')}
            className="px-4 py-2 border border-gray-300 rounded-md text-sm font-medium text-gray-700 hover:bg-gray-50"
          >
            Approve All
          </button>
          <button
            onClick={() => respondToConsent('approve')}
            className="px-4 py-2 bg-green-600 text-white rounded-md text-sm font-medium hover:bg-green-700"
          >
            Approve
          </button>
        </div>
      </div>
    </div>
  );
};
```

#### 7.6.8 Dashboard Analytics Cards

```typescript
// frontend/src/components/dashboard/AnalyticsCards.tsx

import React from 'react';
import MetricCard from './MetricCard';
import { DashboardMetrics } from '../../types/dashboard';

interface AnalyticsCardsProps {
  metrics: DashboardMetrics;
}

export const AnalyticsCards: React.FC<AnalyticsCardsProps> = ({ metrics }) => {
  return (
    <div className="grid grid-cols-4 gap-4 mb-6">
      <MetricCard
        title="Total Sessions"
        value={metrics.total_sessions}
        trend={metrics.session_trend}
        trendLabel={`${Math.abs(metrics.session_trend)}% from last week`}
      />
      <MetricCard
        title="Success Rate"
        value={`${metrics.success_rate}%`}
        trend={metrics.success_rate_trend}
        trendLabel={`${Math.abs(metrics.success_rate_trend)}% from last week`}
        valueColor="text-green-600"
      />
      <MetricCard
        title="Avg Rating"
        value={metrics.avg_rating}
        icon="⭐"
        subtitle={`Based on ${metrics.total_ratings} ratings`}
      />
      <MetricCard
        title="Avg Duration"
        value={metrics.avg_duration}
        trend={-metrics.duration_trend}  // Negative is good for duration
        trendLabel={`${metrics.duration_trend}s from last week`}
      />
    </div>
  );
};
```

#### 7.6.9 Sessions Table

```typescript
// frontend/src/components/dashboard/SessionsTable.tsx

import React from 'react';
import SessionRow from './SessionRow';
import StarRating from './StarRating';
import { Session } from '../../types/session';

interface SessionsTableProps {
  sessions: Session[];
}

export const SessionsTable: React.FC<SessionsTableProps> = ({ sessions }) => {
  return (
    <div className="bg-white rounded-lg shadow-sm overflow-hidden mb-6">
      <div className="px-6 py-4 border-b">
        <h3 className="font-semibold text-gray-800">Recent Sessions</h3>
      </div>

      <table className="w-full">
        <thead className="bg-gray-50 text-xs font-medium text-gray-500 uppercase">
          <tr>
            <th className="px-6 py-3 text-left">Time</th>
            <th className="px-6 py-3 text-left">Request</th>
            <th className="px-6 py-3 text-left">Status</th>
            <th className="px-6 py-3 text-left">Rating</th>
            <th className="px-6 py-3 text-left">Duration</th>
            <th className="px-6 py-3 text-left">Tools Used</th>
          </tr>
        </thead>
        <tbody className="divide-y divide-gray-200">
          {sessions.map(session => (
            <SessionRow key={session.session_id} session={session} />
          ))}
        </tbody>
      </table>
    </div>
  );
};
```

---

## 8. Configuration Schemas

### 8.1 Tool Registry Configuration

```yaml
# config/tool_registry.yaml

# ==============================================================================
# Tool Registry - Purpose-Based Organization
# Naming Pattern: {purpose}_{technology}_{action}_{target}
# ==============================================================================

tools:
  # ============================================================================
  # Purpose: CVE Remediation - Go Language
  # ============================================================================

  - name: cve_go_extract_request_details
    display_name: Go CVE Request Parser
    purpose: cve_remediation
    technology: go
    description: >
      Extracts CVE identifier, Go repository URL, and version/branch from user's
      natural language request about Go vulnerability analysis. Validates that the
      request is about analyzing a CVE for a Go repository.
    category: internal
    capabilities:
      - parse Go CVE analysis request
      - extract CVE identifier
      - extract Go repository information
      - validate Go vulnerability request
      - understand CVE remediation intent

    state_inputs: []
    state_outputs:
      - cve_id
      - repo_url
      - repo_version
      - are_inputs_valid

    requires_consent: false
    risk_level: low
    estimated_duration: "5 seconds"

    is_retriable: true
    max_retries: 2
    retry_delay_ms: 1000

    tasks:
      - task_id: llm_extract
        type: llm_call
        description: Extract structured data from prompt
        target: llm
        dependencies: []
        parameters:
          output_schema: CVEInput

  - name: cve_go_fetch_vulnerability_data
    display_name: Go CVE Data Fetcher
    purpose: cve_remediation
    technology: go
    description: >
      Fetches comprehensive CVE vulnerability data from multiple security databases
      (OSV, NVD, CVEORG, Bugzilla) specifically for Go language vulnerabilities.
      Identifies affected Go packages, fix versions, CVSS scores, and vulnerable symbols.
    category: external
    capabilities:
      - fetch Go CVE data from OSV
      - fetch Go CVE data from NVD
      - query Go vulnerability database
      - identify affected Go packages
      - find Go package fix versions
      - extract CVSS scores for Go CVEs
      - map vulnerable Go symbols

    state_inputs:
      - cve_id
    state_outputs:
      - is_valid_go_cve
      - is_fix_available
      - affected_packages
      - go_cve_id
      - cvss_score
      - cvss_severity

    requires_consent: false
    risk_level: low
    estimated_duration: "15 seconds"

    is_retriable: true
    max_retries: 3
    retry_delay_ms: 2000

    tasks:
      - task_id: fetch_osv
        type: api_call
        description: Fetch from OSV.dev
        target: https://api.osv.dev/v1/query
        dependencies: []

      - task_id: fetch_nvd
        type: api_call
        description: Fetch from NVD NIST
        target: https://services.nvd.nist.gov/rest/json/cves/2.0
        dependencies: []

      - task_id: fetch_cveorg
        type: api_call
        description: Fetch from CVEORG
        target: https://cveawg.mitre.org/api/cve
        dependencies: []

      - task_id: analyze_llm
        type: llm_call
        description: Analyze combined CVE data
        target: llm
        dependencies: [fetch_osv, fetch_nvd, fetch_cveorg]

  - name: cve_go_analyze_repository_impact
    display_name: Go Repository CVE Impact Analyzer
    purpose: cve_remediation
    technology: go
    description: >
      Analyzes a Go repository to determine if a CVE actually affects it by examining
      go.mod dependencies and searching source code for usage of vulnerable package
      symbols. Performs deep static analysis to detect false alarms.
    category: external
    capabilities:
      - analyze Go repository go.mod file
      - check Go package versions
      - search Go code for vulnerable symbols
      - detect CVE false alarms in Go projects
      - perform Go static code analysis
      - determine real Go CVE impact

    state_inputs:
      - repo_url
      - repo_version
      - affected_packages
      - cve_id
    state_outputs:
      - is_false_alarm
      - analysis_incomplete
      - cloned_repo_path
      - gomod_analysis
      - code_chunks

    requires_consent: true
    risk_level: medium
    estimated_duration: "60 seconds"

    is_retriable: true
    max_retries: 2
    retry_delay_ms: 3000

  - name: cve_go_update_dependencies
    display_name: Go Dependency Version Updater
    purpose: cve_remediation
    technology: go
    description: >
      Automatically updates go.mod file to fix CVE vulnerabilities by upgrading
      affected Go packages to safe versions. Handles replace blocks and version
      selection. Requires manual intervention for hard replace blocks.
    category: external
    capabilities:
      - update go.mod dependencies
      - upgrade Go packages to safe versions
      - fix Go CVE vulnerabilities
      - handle Go replace blocks
      - manage Go indirect dependencies

    state_inputs:
      - cloned_repo_path
      - affected_packages
    state_outputs:
      - go_mod_updated
      - version_changes
      - go_runtime_version

    requires_consent: true
    risk_level: high
    estimated_duration: "10 seconds"

    is_retriable: false
    max_retries: 0

  # ============================================================================
  # Purpose: Documentation - Confluence
  # ============================================================================

  - name: docs_confluence_read_page
    display_name: Confluence Page Reader
    purpose: documentation
    technology: confluence
    description: >
      Fetches and reads content from Confluence wiki pages. Retrieves page title,
      body content, attachments, and metadata. Supports both Cloud and Server/Data
      Center instances.
    category: external
    capabilities:
      - read Confluence pages
      - fetch Confluence content
      - access wiki documentation
      - retrieve Confluence page metadata
      - download Confluence attachments

    state_inputs:
      - confluence_page_url
    state_outputs:
      - confluence_page_title
      - confluence_page_content
      - confluence_page_metadata

    requires_consent: false
    risk_level: low
    estimated_duration: "5 seconds"

    is_retriable: true
    max_retries: 3
    retry_delay_ms: 2000

  - name: docs_confluence_explain_content
    display_name: Confluence Content Explainer
    purpose: documentation
    technology: confluence
    description: >
      Analyzes and explains technical content from Confluence pages. Provides
      simplified explanations, identifies key concepts, extracts action items,
      and answers specific questions about the documentation.
    category: internal
    capabilities:
      - explain Confluence documentation
      - summarize technical wiki pages
      - extract key concepts from docs
      - answer questions about documentation
      - simplify complex technical content

    state_inputs:
      - confluence_page_content
      - user_question
    state_outputs:
      - explanation_summary
      - key_concepts
      - action_items
      - qa_response

    requires_consent: false
    risk_level: low
    estimated_duration: "10 seconds"

    is_retriable: true
    max_retries: 2
    retry_delay_ms: 1000

  # ============================================================================
  # Purpose: Project Management - Jira
  # ============================================================================

  - name: pm_jira_fetch_issue
    display_name: Jira Issue Fetcher
    purpose: project_management
    technology: jira
    description: >
      Fetches detailed information about a specific Jira issue including summary,
      description, status, assignee, comments, attachments, and custom fields.
      Supports both Jira Cloud and Server/Data Center.
    category: external
    capabilities:
      - fetch Jira issue details
      - retrieve Jira ticket information
      - get Jira issue status
      - access Jira comments
      - read Jira custom fields

    state_inputs:
      - jira_issue_key
    state_outputs:
      - jira_issue_summary
      - jira_issue_description
      - jira_issue_status
      - jira_issue_assignee
      - jira_issue_comments
      - jira_issue_metadata

    requires_consent: false
    risk_level: low
    estimated_duration: "5 seconds"

    is_retriable: true
    max_retries: 3
    retry_delay_ms: 2000

  - name: pm_jira_query_issues
    display_name: Jira JQL Query Tool
    purpose: project_management
    technology: jira
    description: >
      Executes JQL (Jira Query Language) queries to find issues matching specific
      criteria. Useful for finding all issues in a sprint, filtering by assignee,
      status, labels, or custom queries.
    category: external
    capabilities:
      - query Jira issues with JQL
      - search Jira tickets
      - filter issues by criteria
      - find sprint issues
      - get issues by status

    state_inputs:
      - jql_query
    state_outputs:
      - jira_query_results
      - matching_issue_keys
      - issue_count

    requires_consent: false
    risk_level: low
    estimated_duration: "5 seconds"

    is_retriable: true
    max_retries: 3
    retry_delay_ms: 2000

  # ============================================================================
  # Purpose: Quality Engineering - Testing
  # ============================================================================

  - name: qe_project_analyze_test_failure
    display_name: Test Failure Analyzer
    purpose: quality_engineering
    technology: project_specific
    description: >
      Analyzes test failure logs, stack traces, and error messages to identify
      root causes. Searches code for relevant functions, checks recent commits,
      and suggests potential fixes based on failure patterns.
    category: internal
    capabilities:
      - analyze test failure logs
      - parse stack traces
      - identify test failure root cause
      - suggest test fixes
      - search code for error context

    state_inputs:
      - test_failure_logs
      - repository_url
      - repository_version
    state_outputs:
      - failure_root_cause
      - suggested_fixes
      - related_code_snippets

    requires_consent: false
    risk_level: low
    estimated_duration: "15 seconds"

    is_retriable: true
    max_retries: 2
    retry_delay_ms: 1000

  - name: qe_github_search_similar_issues
    display_name: GitHub Similar Issue Finder
    purpose: quality_engineering
    technology: github
    description: >
      Searches GitHub issues and pull requests for similar problems based on error
      messages, stack traces, or symptom descriptions. Helps identify if issue is
      already known, fixed, or has workarounds.
    category: external
    capabilities:
      - search GitHub issues
      - find similar problems
      - query GitHub PRs
      - identify known issues
      - discover workarounds

    state_inputs:
      - repository_url
      - error_message
      - search_query
    state_outputs:
      - similar_issues
      - related_prs
      - known_workarounds

    requires_consent: false
    risk_level: low
    estimated_duration: "10 seconds"

    is_retriable: true
    max_retries: 3
    retry_delay_ms: 2000
```

### 8.2 LLM Configuration

```yaml
# config/llm_config.yaml

llm_config:
  # ============================================================================
  # LLM Provider Definitions
  # Multiple providers can be configured and selected per-tool
  # ============================================================================

  providers:
    # Gemini (Google)
    gemini_flash:
      provider: gemini
      model: gemini-2.0-flash-exp
      api_key: ${GEMINI_API_KEY}
      base_url: null
      temperature: 0.3
      max_tokens: 4096
      timeout_seconds: 30

    gemini_pro:
      provider: gemini
      model: gemini-2.0-pro-exp
      api_key: ${GEMINI_API_KEY}
      base_url: null
      temperature: 0.2
      max_tokens: 8192
      timeout_seconds: 60

    # Ollama (Local)
    ollama_llama3:
      provider: ollama
      model: llama3:latest
      api_key: null  # Not required for Ollama
      base_url: http://localhost:11434
      temperature: 0.3
      max_tokens: 4096
      timeout_seconds: 60

    ollama_codellama:
      provider: ollama
      model: codellama:latest
      api_key: null
      base_url: http://localhost:11434
      temperature: 0.2
      max_tokens: 4096
      timeout_seconds: 60

    ollama_mistral:
      provider: ollama
      model: mistral:latest
      api_key: null
      base_url: http://localhost:11434
      temperature: 0.3
      max_tokens: 4096
      timeout_seconds: 60

  # ============================================================================
  # Default LLM Selection by Component
  # ============================================================================

  defaults:
    # Planner LLM (for plan generation)
    planner: gemini_pro
    planner_config:
      temperature: 0.1  # Override temperature for planning
      max_tokens: 4096
      system_prompt_template: prompts/planner_system.txt

    # Tool LLM calls (default for tools that use LLM)
    tool_default: gemini_flash

    # Recovery planning (when step fails)
    recovery_planner: gemini_flash

  # ============================================================================
  # Per-Tool LLM Override
  # Tools can specify preferred LLM in their configuration
  # ============================================================================

  tool_overrides:
    # CVE analysis tools - use powerful models for accuracy
    cve_go_fetch_vulnerability_data: gemini_pro
    cve_go_diagnose_upgrade_failure: gemini_pro

    # Simple extraction - use fast/cheap models
    cve_go_extract_request_details: gemini_flash

    # Documentation tools - local models for privacy
    docs_confluence_explain_content: ollama_llama3

    # Code analysis - use code-optimized models
    qe_project_analyze_test_failure: ollama_codellama

  # ============================================================================
  # Fallback Configuration
  # ============================================================================

  fallback:
    enabled: true
    provider: gemini_flash  # Fallback to this provider
    trigger_on:
      - timeout
      - rate_limit
      - model_unavailable
      - api_error

  # ============================================================================
  # Cost Optimization
  # ============================================================================

  cost_optimization:
    enabled: true
    prefer_local_for_simple_tasks: true  # Use Ollama for low-complexity tasks
    complexity_threshold: 5  # 1-10 scale, tasks below this use local models
```

### 8.3 System Configuration

```yaml
# config/system_config.yaml

system_config:
  session:
    idle_timeout_ms: 300000
    consent_timeout_seconds: 300
    max_active_sessions: 100

  rate_limits:
    session_creation:
      limit: 10
      window_seconds: 3600
      scope: user

    api_endpoint:
      limit: 100
      window_seconds: 60
      scope: api_key

    llm_calls:
      limit: 50
      scope: session

    tool_executions:
      limit: 100
      scope: session

  security:
    session_secret: ${SESSION_SECRET}
    api_key_bcrypt_rounds: 12
    enable_message_signing: false

  features:
    enable_feedback: true
    enable_mcp_servers: true
    enable_parallel_steps: true
    enable_auto_recovery: true

  websocket:
    ping_interval_seconds: 30
    max_message_size_bytes: 1048576

  database:
    type: postgresql  # sqlite | postgresql
    postgres_host: ${DB_HOST}
    postgres_port: 5432
    postgres_database: ${DB_NAME}
    postgres_user: ${DB_USER}
    postgres_password: ${DB_PASSWORD}
    pool_size: 5
    max_overflow: 10
```

---

### 8.4 Branding Configuration

```yaml
# frontend/branding_config.yaml

branding:
  # Application Branding
  app_name: "OCP Sustaining Bot"
  app_tagline: "AI-Powered Automation for Sustaining Engineering"
  company_name: "Red Hat"

  # Logo & Favicon
  logo_url: "/assets/logo.png"
  logo_dark_url: "/assets/logo-dark.png"  # For dark mode
  favicon_url: "/assets/favicon.ico"

  # Colors (Tailwind CSS compatible)
  theme:
    primary_color: "#0066CC"      # Primary brand color
    secondary_color: "#EE0000"    # Secondary accent color
    success_color: "#3E8635"
    warning_color: "#F0AB00"
    error_color: "#C9190B"

  # UI Customization
  ui:
    show_company_logo: true
    show_powered_by: false        # Hide "Powered by..." text
    enable_dark_mode: true
    default_theme: "light"        # light | dark | auto

  # Chat Interface
  chat:
    bot_avatar_url: "/assets/bot-avatar.png"
    bot_name: "OCP Assistant"
    welcome_message: |
      Hello! I'm your OCP Sustaining Assistant. I can help you with:
      - CVE analysis and remediation for Go projects
      - Documentation lookup from Confluence
      - Jira issue management
      - Test failure analysis

      How can I assist you today?

    placeholder_text: "Type your message here..."
    enable_markdown: true
    enable_code_highlighting: true

  # Dashboard
  dashboard:
    show_analytics: true
    show_session_history: true
    max_sessions_display: 50

  # Footer
  footer:
    show_footer: true
    copyright_text: "© 2026 Red Hat, Inc."
    links:
      - label: "Documentation"
        url: "https://docs.example.com"
      - label: "Support"
        url: "https://support.example.com"
      - label: "Privacy Policy"
        url: "https://privacy.example.com"

  # Contact & Support
  support:
    show_support_button: true
    support_email: "support@example.com"
    support_url: "https://support.example.com"
```

**TypeScript Interface:**

```typescript
// frontend/src/types/branding.ts

export interface BrandingConfig {
  app_name: string;
  app_tagline: string;
  company_name: string;

  logo_url: string;
  logo_dark_url?: string;
  favicon_url: string;

  theme: {
    primary_color: string;
    secondary_color: string;
    success_color: string;
    warning_color: string;
    error_color: string;
  };

  ui: {
    show_company_logo: boolean;
    show_powered_by: boolean;
    enable_dark_mode: boolean;
    default_theme: "light" | "dark" | "auto";
  };

  chat: {
    bot_avatar_url: string;
    bot_name: string;
    welcome_message: string;
    placeholder_text: string;
    enable_markdown: boolean;
    enable_code_highlighting: boolean;
  };

  dashboard: {
    show_analytics: boolean;
    show_session_history: boolean;
    max_sessions_display: number;
  };

  footer: {
    show_footer: boolean;
    copyright_text: string;
    links: Array<{
      label: string;
      url: string;
    }>;
  };

  support: {
    show_support_button: boolean;
    support_email: string;
    support_url: string;
  };
}
```

**Loading Branding Config:**

```typescript
// frontend/src/config/branding.ts

import brandingConfigYaml from '../../branding_config.yaml';

export const loadBrandingConfig = (): BrandingConfig => {
  // Load from YAML file or fetch from API
  return brandingConfigYaml.branding;
};

// Apply theme colors to CSS variables
export const applyTheme = (config: BrandingConfig) => {
  const root = document.documentElement;
  root.style.setProperty('--color-primary', config.theme.primary_color);
  root.style.setProperty('--color-secondary', config.theme.secondary_color);
  root.style.setProperty('--color-success', config.theme.success_color);
  root.style.setProperty('--color-warning', config.theme.warning_color);
  root.style.setProperty('--color-error', config.theme.error_color);
};
```

---

## 9. Environment Variables

```bash
# .env

# LLM Configuration
LLM_API_KEY=your_gemini_api_key_here
LLM_PROVIDER=gemini

# Database
DB_TYPE=postgresql
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ocp_bot
DB_USER=postgres
DB_PASSWORD=your_db_password

# Security
SESSION_SECRET=your_secret_key_for_jwt_signing
API_KEY_SALT=your_bcrypt_salt

# External APIs (Optional - for tools)
GITHUB_TOKEN=your_github_token
NVD_API_KEY=your_nvd_api_key

# Application
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5173
```

---

## 10. Error Handling Patterns

```python
# utils/errors.py

class BotException(Exception):
    """Base exception for bot errors."""
    pass

class ToolExecutionError(BotException):
    """Tool execution failed."""
    def __init__(self, tool_name: str, error: str):
        self.tool_name = tool_name
        self.error = error
        super().__init__(f"Tool {tool_name} failed: {error}")

class ConsentTimeoutError(BotException):
    """Consent request timed out."""
    pass

class SessionNotFoundError(BotException):
    """Session not found."""
    pass

class ValidationError(BotException):
    """Input validation failed."""
    pass

# Error handler middleware
@app.exception_handler(BotException)
async def bot_exception_handler(request: Request, exc: BotException):
    return JSONResponse(
        status_code=400,
        content={"error": str(exc), "type": exc.__class__.__name__}
    )
```

---

## 11. Testing Strategy

### 11.1 Unit Tests

```python
# tests/unit/test_fetch_inputs_tool.py

import pytest
from tools.input.fetch_inputs import FetchInputsTool
from models.session import SessionState, Message
from llm.provider import MockLLMProvider

@pytest.mark.asyncio
async def test_fetch_inputs_success():
    # Arrange
    llm = MockLLMProvider()
    tool = FetchInputsTool(llm, {})

    state = SessionState(
        session_id="test-123",
        messages=[
            Message(
                message_id="msg-1",
                role="user",
                content="Analyze CVE-2024-1234 for github.com/org/repo at version v1.0.0",
                timestamp=datetime.utcnow()
            )
        ],
        created_at=datetime.utcnow(),
        last_activity=datetime.utcnow()
    )

    # Act
    result = await tool.execute(state, {})

    # Assert
    assert result.success is True
    assert state.data.cve_id == "CVE-2024-1234"
    assert state.data.repo_url == "github.com/org/repo"
    assert state.data.repo_version == "v1.0.0"
```

### 11.2 Integration Tests

```python
# tests/integration/test_cve_workflow.py

@pytest.mark.asyncio
async def test_full_cve_remediation_workflow():
    """Test complete CVE remediation flow."""
    # Create session
    # Execute: fetch_inputs → assess_cve → false_alarm_check
    # Verify state transitions
    pass
```

---

## 12. Tool Naming Convention & Organization Guide

### 12.1 Purpose-Based Naming Pattern

**Pattern:** `{purpose}_{technology}_{action}_{target}`

**Components:**
1. **Purpose**: Primary use case category (e.g., `cve`, `docs`, `pm`, `qe`)
2. **Technology**: Platform/language being worked with (e.g., `go`, `python`, `confluence`, `jira`)
3. **Action**: What the tool does (e.g., `fetch`, `analyze`, `update`, `create`)
4. **Target**: What it acts upon (e.g., `vulnerability_data`, `page`, `issue`)

### 12.2 Purpose Categories

| Purpose | Full Name | Description | Example Technologies |
|---------|-----------|-------------|---------------------|
| `cve` | CVE Remediation | Security vulnerability analysis and fixes | go, python, rust, javascript |
| `docs` | Documentation | Read, explain, search documentation | confluence, sharepoint, notion |
| `pm` | Project Management | Issue tracking, sprint management | jira, github, gitlab |
| `qe` | Quality Engineering | Testing, debugging, failure analysis | project_specific, jenkins, github |
| `deploy` | Deployment | CI/CD, infrastructure, releases | kubernetes, docker, terraform |
| `monitor` | Monitoring | Metrics, alerts, observability | prometheus, grafana, splunk |

### 12.3 Benefits

#### 1. **Clear Intent for LLM**
LLM can instantly understand:
- **What**: Purpose indicates the domain
- **Where**: Technology specifies the platform
- **How**: Action describes the operation

**Example:**
```
User: "Analyze CVE-2024-1234 for my Python app"

LLM sees tools:
- cve_go_fetch_vulnerability_data     ❌ (wrong technology)
- cve_python_fetch_vulnerability_data ✅ (correct match)
- docs_confluence_read_page           ❌ (wrong purpose)
```

#### 2. **Easy Tool Discovery**
```python
# Find all Go CVE tools
registry.search_tools(purpose="cve_remediation", technology="go")

# Find all documentation tools
registry.get_by_purpose("documentation")

# Find all Confluence-related tools
registry.get_by_technology("confluence")
```

#### 3. **Scalable for Future**
Adding new capabilities is straightforward:

```yaml
# Adding Python CVE support
- name: cve_python_fetch_vulnerability_data
- name: cve_python_update_requirements_txt
- name: cve_python_validate_fixes

# Adding Slack integration
- name: pm_slack_send_notification
- name: pm_slack_create_channel

# Adding Kubernetes deployment
- name: deploy_kubernetes_apply_manifest
- name: deploy_kubernetes_scale_deployment
```

#### 4. **Avoid Conflicts**
Different purposes can have similar actions:

```yaml
- cve_go_analyze_repository_impact      # Scans for vulnerabilities
- qe_project_analyze_test_failure       # Scans for test issues
- docs_confluence_analyze_page_structure # Analyzes documentation
```

All have "analyze" but are clearly different contexts.

### 12.4 Capability Keywords Best Practices

**Include:**
1. **Purpose keyword** - e.g., "Go CVE", "Confluence documentation", "Jira issue"
2. **Action verbs** - e.g., "fetch", "analyze", "update", "create", "search"
3. **Target nouns** - e.g., "vulnerability data", "wiki page", "test failure"
4. **Platform specifics** - e.g., "go.mod", "JQL query", "stack trace"

**Example:**
```yaml
capabilities:
  - fetch Go CVE data from OSV          # Purpose + Action + Platform
  - identify affected Go packages       # Action + Domain + Technology
  - extract CVSS scores for Go CVEs     # Action + Target + Purpose
  - map vulnerable Go symbols           # Action + Target + Technology
```

### 12.5 Tool Organization Structure

```
tools/
├── base.py                                    # Base Tool class
│
├── cve_remediation/                           # Purpose
│   ├── go/                                    # Technology
│   │   ├── cve_go_extract_request_details.py
│   │   ├── cve_go_fetch_vulnerability_data.py
│   │   └── ...
│   ├── python/
│   │   └── cve_python_*.py
│   └── rust/
│       └── cve_rust_*.py
│
├── documentation/
│   ├── confluence/
│   │   └── docs_confluence_*.py
│   ├── sharepoint/
│   │   └── docs_sharepoint_*.py
│   └── notion/
│       └── docs_notion_*.py
│
├── project_management/
│   ├── jira/
│   │   └── pm_jira_*.py
│   └── github/
│       └── pm_github_*.py
│
└── quality_engineering/
    ├── project_specific/
    │   └── qe_project_*.py
    └── github/
        └── qe_github_*.py
```

### 12.6 Adding New Tools - Checklist

When adding a new tool:

1. ✅ **Determine purpose** - Which category does it belong to?
2. ✅ **Identify technology** - What platform/language does it work with?
3. ✅ **Name appropriately** - Follow `{purpose}_{technology}_{action}_{target}` pattern
4. ✅ **Write clear description** - Explain what it does in 2-3 sentences
5. ✅ **List capabilities** - Include purpose, technology, action keywords
6. ✅ **Define state I/O** - What fields does it read/write?
7. ✅ **Set risk level** - Based on what it modifies (read-only = low, modify code = high)
8. ✅ **Configure retry** - Is it safe to retry? How many times?
9. ✅ **Register in YAML** - Add to `tool_registry.yaml`
10. ✅ **Update SessionData** - Add required state fields if new

---

*Document Version: 2.0*
*Last Updated: February 2026*

**Changelog:**
- v2.0: Added purpose-based tool naming convention and multi-dimensional indexing
- v2.0: Added Confluence, Jira, and QE tool examples
- v2.0: Updated SessionData model with new state fields
- v2.0: Enhanced ToolRegistry with purpose/technology search
- v1.0: Initial LLD with Go CVE remediation tools
