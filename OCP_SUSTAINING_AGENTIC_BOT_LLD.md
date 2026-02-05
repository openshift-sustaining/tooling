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
├── LLM SDK: Configurable (OpenAI SDK, Google GenAI SDK, Anthropic SDK)
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
│   ├── mcp_servers.yaml         # MCP server config
│   └── branding_config.yaml     # White-label config
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
│   │   └── consent_manager.py  # Consent Manager
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
│   ├── openai_provider.py
│   ├── gemini_provider.py
│   └── anthropic_provider.py
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
    # Inputs
    cve_id: Optional[str] = None
    repo_url: Optional[str] = None
    repo_version: Optional[str] = None
    cloned_repo_path: Optional[str] = None

    # CVE Analysis
    cve_details: Optional[Dict[str, Any]] = None
    affected_packages: Optional[List[Dict[str, Any]]] = None
    is_vulnerable: Optional[bool] = None
    is_false_alarm: Optional[bool] = None

    # Remediation
    fix_available: Optional[bool] = None
    fix_version: Optional[str] = None
    go_mod_updated: Optional[bool] = None
    version_changes: Optional[List[Dict[str, Any]]] = None

    # Test Results
    build_success: Optional[bool] = None
    test_success: Optional[bool] = None
    test_logs: Optional[str] = None

    # Output
    pr_url: Optional[str] = None
    summary: Optional[str] = None

    # Error Tracking
    last_error: Optional[str] = None
    last_error_step: Optional[str] = None
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
class StepStartedPayload(BaseModel):
    step_id: str
    tool_name: str

# 6. step_progress
class StepProgressPayload(BaseModel):
    step_id: str
    progress: str

# 7. step_completed
class StepCompletedPayload(BaseModel):
    step_id: str
    success: bool
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None

# 8. plan_completed
class PlanCompletedPayload(BaseModel):
    plan_id: str
    summary: str

# 9. session_timeout
class SessionTimeoutPayload(BaseModel):
    message: str

# 10. error
class ErrorPayload(BaseModel):
    message: str
    recoverable: bool

# 11. ping
# Empty payload
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
        # Returns OpenAI-compatible function schema
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
            await progress_callback("step_started", {"step_id": step.step_id, "tool_name": step.tool_name})

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

```
tools/
├── base.py                    # Base Tool class
├── input/
│   └── fetch_inputs.py        # Extract CVE ID, repo URL, version
├── analysis/
│   ├── assess_cve.py          # Fetch CVE data from multiple APIs
│   └── false_alarm_check.py   # Analyze go.mod + code for vulnerability
├── remediation/
│   ├── apply_fix.py           # Update go.mod versions
│   ├── test_fix.py            # Run tests/builds
│   └── analyse_failure.py     # Analyze test failures
└── output/
    └── generate_output.py     # Generate final summary
```

### 6.2 Base Tool Class

```python
# tools/base.py

from abc import ABC, abstractmethod
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
from pydantic import BaseModel

class Tool(ABC):
    """Base class for all tools."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config

    @property
    @abstractmethod
    def name(self) -> str:
        """Tool identifier."""
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

### 6.3 CVE Tools Implementation

#### 6.3.1 FetchInputs Tool

```python
# tools/input/fetch_inputs.py

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

class FetchInputsTool(Tool):
    def __init__(self, llm_provider: LLMProvider, config: Dict[str, Any]):
        super().__init__(config)
        self.llm = llm_provider

    @property
    def name(self) -> str:
        return "fetch_inputs"

    @property
    def display_name(self) -> str:
        return "Input Extractor"

    @property
    def description(self) -> str:
        return "Extracts CVE ID, repository URL, and version from user's natural language prompt"

    @property
    def category(self) -> str:
        return "internal"

    @property
    def capabilities(self) -> List[str]:
        return ["extract CVE", "parse repository", "understand user intent", "validate inputs"]

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

#### 6.3.2 AssessCVE Tool

```python
# tools/analysis/assess_cve.py

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

class AssessCVETool(Tool):
    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.http_client = httpx.AsyncClient(timeout=30.0)

    @property
    def name(self) -> str:
        return "assess_cve"

    @property
    def display_name(self) -> str:
        return "CVE Assessor"

    @property
    def description(self) -> str:
        return "Fetches CVE data from multiple APIs (OSV, NVD, CVEORG, Bugzilla) and identifies affected Go packages with fix versions"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "fetch CVE data",
            "validate Go vulnerability",
            "find affected packages",
            "identify fix versions",
            "CVSS scoring"
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

#### 6.3.3 FalseAlarmCheck Tool (Abbreviated)

```python
# tools/analysis/false_alarm_check.py

from tools.base import Tool
from models.session import SessionState, ToolResult
from typing import Dict, Any, List
import subprocess
import tempfile
import logging

logger = logging.getLogger(__name__)

class FalseAlarmCheckTool(Tool):
    """
    Analyzes go.mod and codebase to determine if CVE actually affects repository.

    Two-phase analysis:
    1. Phase 1: go.mod check (fast) - Check versions
    2. Phase 2: Code analysis (thorough) - Search for vulnerable symbol usage
    """

    @property
    def name(self) -> str:
        return "false_alarm_check"

    @property
    def display_name(self) -> str:
        return "False Alarm Checker"

    @property
    def description(self) -> str:
        return "Analyzes go.mod and source code to determine if CVE actually impacts the repository"

    @property
    def category(self) -> str:
        return "external"

    @property
    def capabilities(self) -> List[str]:
        return [
            "analyze go.mod",
            "check package versions",
            "search code for symbols",
            "determine false alarm",
            "static analysis"
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

#### 6.3.4 Other CVE Tools (Summaries)

```python
# tools/remediation/apply_fix.py
class ApplyFixTool(Tool):
    """
    Updates go.mod with fixed package versions.

    Handles:
    - Replace blocks (complex logic for version overrides)
    - Version selection (nearest higher version)
    - HARDREPLACE detection (manual intervention needed)
    """
    state_inputs = ["cloned_repo_path", "affected_packages"]
    state_outputs = ["go_mod_updated", "version_changes", "go_runtime_version"]
    requires_consent = True
    risk_level = "high"  # Modifies code


# tools/remediation/test_fix.py
class TestFixTool(Tool):
    """
    Validates applied fixes by running tests and builds.

    Executes:
    1. go mod tidy
    2. go mod vendor
    3. make test (extracted from Makefile via LLM)
    4. make build (only if tests pass)
    """
    state_inputs = ["cloned_repo_path", "go_runtime_version"]
    state_outputs = ["is_test_successful", "last_failure_step", "last_failure_logs"]
    requires_consent = True
    risk_level = "medium"


# tools/remediation/analyse_failure.py
class AnalyseFailureTool(Tool):
    """
    Analyzes commit diffs to identify root cause of test failures.

    Process:
    - Clones package repositories
    - Extracts commits between old and new versions
    - Sends commit diffs + failure logs + code context to LLM
    - LLM identifies breaking changes and suggests fixes
    """
    state_inputs = ["version_changes", "last_failure_step", "last_failure_logs", "code_chunks"]
    state_outputs = ["failure_analysis_response"]
    requires_consent = False
    risk_level = "low"


# tools/output/generate_output.py
class GenerateOutputTool(Tool):
    """
    Generates final markdown summary of entire workflow.

    Sections:
    1. CVE Overview
    2. Repository
    3. Analysis Result
    4. Actions Taken
    5. Test Results
    6. Outcome
    7. Recommendations
    """
    state_inputs = []  # Reads entire state
    state_outputs = ["final_output", "generated_summary"]
    requires_consent = False
    risk_level = "low"
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

### 7.2 TypeScript Models

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

### 7.3 WebSocket Hook

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

## 8. Configuration Schemas

### 8.1 Tool Registry Configuration

```yaml
# config/tool_registry.yaml

tools:
  - name: fetch_inputs
    display_name: Input Extractor
    description: Extracts CVE ID, repository URL, and version from user's natural language prompt
    category: internal
    capabilities:
      - extract CVE
      - parse repository
      - understand user intent
      - validate inputs

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

  - name: assess_cve
    display_name: CVE Assessor
    description: Fetches CVE data from multiple APIs and identifies affected Go packages
    category: external
    capabilities:
      - fetch CVE data
      - validate Go vulnerability
      - find affected packages
      - identify fix versions
      - CVSS scoring

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
        description: Analyze combined data
        target: llm
        dependencies: [fetch_osv, fetch_nvd, fetch_cveorg]

  # Additional tools...
```

### 8.2 LLM Configuration

```yaml
# config/llm_config.yaml

llm_config:
  provider: gemini  # openai | gemini | anthropic | ollama
  model: gemini-2.0-flash-exp
  api_key: ${LLM_API_KEY}
  base_url: null  # For custom endpoints

  planner:
    temperature: 0.1
    max_tokens: 4096
    timeout_seconds: 30
    system_prompt_template: prompts/planner_system.txt

  tool_llm_calls:
    temperature: 0.3
    max_tokens: 2048
    timeout_seconds: 20

  fallback:
    enabled: true
    provider: openai
    model: gpt-4o-mini
    trigger_on:
      - timeout
      - rate_limit
      - model_unavailable
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

*Document Version: 1.0*
*Last Updated: February 2026*
