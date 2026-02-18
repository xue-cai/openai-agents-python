# Tools Deep Dive: Architecture & Technical Reference

This document provides a comprehensive technical analysis of the tools system in the OpenAI Agents Python SDK. It covers tool types, MCP server integration, authentication, the execution pipeline, guardrails, and approval workflows — with code snippets from the actual implementation.

---

## Table of Contents

1. [Tool Types Overview](#tool-types-overview)
2. [FunctionTool & the `@function_tool` Decorator](#functiontool--the-function_tool-decorator)
3. [Hosted OpenAI Tools](#hosted-openai-tools)
4. [MCP Server Integration](#mcp-server-integration)
5. [Shell & ApplyPatch Tools](#shell--applypatch-tools)
6. [Agents as Tools](#agents-as-tools)
7. [Authentication & Security](#authentication--security)
8. [Tool Execution Pipeline](#tool-execution-pipeline)
9. [Tool Guardrails](#tool-guardrails)
10. [Approval & Human-in-the-Loop](#approval--human-in-the-loop)
11. [Tool Filtering](#tool-filtering)
12. [Timeouts & Error Handling](#timeouts--error-handling)

---

## Tool Types Overview

All tool types are defined in `src/agents/tool.py` and collected into a single union type:

```python
# src/agents/tool.py
Tool = Union[
    FunctionTool,
    FileSearchTool,
    WebSearchTool,
    ComputerTool[Any],
    HostedMCPTool,
    ShellTool,
    ApplyPatchTool,
    LocalShellTool,
    ImageGenerationTool,
    CodeInterpreterTool,
]
```

These fall into five categories:

| Category | Tools | Runs On |
|----------|-------|---------|
| **Function calling** | `FunctionTool` | Your Python environment |
| **Hosted OpenAI** | `WebSearchTool`, `FileSearchTool`, `CodeInterpreterTool`, `ImageGenerationTool`, `HostedMCPTool` | OpenAI servers |
| **Local runtime** | `ComputerTool`, `ShellTool`, `LocalShellTool`, `ApplyPatchTool` | Your environment |
| **MCP (via `mcp_servers`)** | Tools from `MCPServerStdio`, `MCPServerSse`, `MCPServerStreamableHttp` | Local or remote MCP servers |
| **Agents as tools** | Any `Agent` via `agent.as_tool(...)` | Your environment (nested `Runner.run`) |

---

## FunctionTool & the `@function_tool` Decorator

### How It Works

The `@function_tool` decorator converts any Python function into a `FunctionTool` dataclass. The SDK:

1. **Inspects the function signature** (via `inspect` module) to build a Pydantic model.
2. **Parses the docstring** (via `griffe`) to extract the tool description and per-argument descriptions.
3. **Generates a JSON Schema** from the Pydantic model, ensuring strict-mode compatibility with OpenAI's API.
4. **Wraps the function** in an async `on_invoke_tool` callback that handles JSON parsing, validation, and error recovery.

### Key Dataclass Fields

```python
# src/agents/tool.py
@dataclass
class FunctionTool:
    name: str
    description: str
    params_json_schema: dict[str, Any]
    on_invoke_tool: Callable[[ToolContext[Any], str], Awaitable[Any]]
    strict_json_schema: bool = True
    is_enabled: bool | Callable[..., MaybeAwaitable[bool]] = True
    tool_input_guardrails: list[ToolInputGuardrail[Any]] | None = None
    tool_output_guardrails: list[ToolOutputGuardrail[Any]] | None = None
    needs_approval: bool | Callable[..., Awaitable[bool]] = False
    timeout_seconds: float | None = None
    timeout_behavior: ToolTimeoutBehavior = "error_as_result"
```

### Invocation Flow Inside `_on_invoke_tool_impl`

```python
# src/agents/tool.py – inside function_tool()
async def _on_invoke_tool_impl(ctx: ToolContext[Any], input: str) -> Any:
    # 1. Parse raw JSON string from the LLM
    json_data = json.loads(input) if input else {}

    # 2. Validate against the auto-generated Pydantic model
    parsed = schema.params_pydantic_model(**json_data)

    # 3. Convert to positional + keyword args
    args, kwargs_dict = schema.to_call_args(parsed)

    # 4. Call the original function (async or sync via asyncio.to_thread)
    if not is_sync_function_tool:
        if schema.takes_context:
            result = await the_func(ctx, *args, **kwargs_dict)
        else:
            result = await the_func(*args, **kwargs_dict)
    else:
        if schema.takes_context:
            result = await asyncio.to_thread(the_func, ctx, *args, **kwargs_dict)
        else:
            result = await asyncio.to_thread(the_func, *args, **kwargs_dict)

    return result
```

**Key points:**

- Sync functions are automatically wrapped with `asyncio.to_thread` so they never block the event loop.
- The first parameter is checked for `RunContextWrapper` or `ToolContext` type; if present, the SDK injects the context automatically.
- Error recovery uses `failure_error_function` to convert exceptions into model-visible messages instead of crashing the run.

### Tool Output Types

Function tools can return text, images, or files:

```python
# src/agents/tool.py
class ToolOutputText(BaseModel):
    type: Literal["text"] = "text"
    text: str

class ToolOutputImage(BaseModel):
    type: Literal["image"] = "image"
    image_url: str | None = None
    file_id: str | None = None

class ToolOutputFileContent(BaseModel):
    type: Literal["file"] = "file"
    file_data: str | None = None
    file_url: str | None = None
    file_id: str | None = None
```

---

## Hosted OpenAI Tools

Hosted tools run on OpenAI's servers alongside the model. They are configured as dataclasses and sent as part of the API request — no local execution needed.

### WebSearchTool

```python
# src/agents/tool.py
@dataclass
class WebSearchTool:
    user_location: UserLocation | None = None
    filters: WebSearchToolFilters | None = None
    search_context_size: Literal["low", "medium", "high"] = "medium"

    @property
    def name(self):
        return "web_search"
```

### FileSearchTool

```python
@dataclass
class FileSearchTool:
    vector_store_ids: list[str]
    max_num_results: int | None = None
    include_search_results: bool = False
    ranking_options: RankingOptions | None = None
    filters: Filters | None = None
```

### HostedMCPTool

This is distinct from local MCP servers — it tells the OpenAI Responses API to call a remote MCP server directly:

```python
@dataclass
class HostedMCPTool:
    tool_config: Mcp  # includes server URL, headers, allowed_tools, etc.
    on_approval_request: MCPToolApprovalFunction | None = None

    @property
    def name(self):
        return "hosted_mcp"
```

### CodeInterpreterTool & ImageGenerationTool

```python
@dataclass
class CodeInterpreterTool:
    tool_config: CodeInterpreter

@dataclass
class ImageGenerationTool:
    tool_config: ImageGeneration
```

These tools are **not** calling MCP servers and are **not** using skills in the SDK sense — they are OpenAI Responses API built-in tools.

---

## MCP Server Integration

MCP (Model Context Protocol) servers are the mechanism for connecting to external tool providers. The SDK provides three transport implementations.

### Architecture

```
Agent(mcp_servers=[server1, server2])
       │
       ▼
MCPUtil.get_all_function_tools()    ← converts MCP tools → FunctionTool wrappers
       │
       ├── server.list_tools()      ← fetches tool catalog
       │
       └── server.call_tool()       ← invokes a specific tool
```

### Base Class (`MCPServer`)

```python
# src/agents/mcp/server.py
class MCPServer(abc.ABC):
    @abc.abstractmethod
    async def connect(self): ...

    @abc.abstractmethod
    async def list_tools(self, run_context, agent) -> list[MCPTool]: ...

    @abc.abstractmethod
    async def call_tool(self, tool_name, arguments, meta=None) -> CallToolResult: ...

    @abc.abstractmethod
    async def cleanup(self): ...
```

### Transport Implementations

**1. `MCPServerStdio`** — Spawns a subprocess communicating via stdin/stdout:

```python
# src/agents/mcp/server.py
class MCPServerStdio(_MCPServerWithClientSession):
    def __init__(self, params: MCPServerStdioParams, ...):
        # params includes: command, args, env, cwd, encoding
        ...

    def create_streams(self):
        return stdio_client(
            StdioServerParameters(
                command=self._params["command"],
                args=self._params.get("args", []),
                env=self._params.get("env"),
                cwd=self._params.get("cwd"),
            )
        )
```

**2. `MCPServerSse`** — Connects to a remote server via HTTP Server-Sent Events:

```python
class MCPServerSse(_MCPServerWithClientSession):
    def __init__(self, params: MCPServerSseParams, ...):
        # params includes: url, headers, timeout, sse_read_timeout
        ...

    def create_streams(self):
        return sse_client(
            url=self._params["url"],
            headers=self._params.get("headers", {}),
            timeout=self._params.get("timeout", 5),
        )
```

**3. `MCPServerStreamableHttp`** — Streamable HTTP transport for large payloads:

```python
class MCPServerStreamableHttp(_MCPServerWithClientSession):
    def create_streams(self):
        return streamablehttp_client(
            url=self._params["url"],
            headers=self._params.get("headers", {}),
            timeout=...,
        )
```

### MCP Tool → FunctionTool Conversion

When an agent runs, the SDK fetches tools from each MCP server and wraps them as `FunctionTool` instances:

```python
# src/agents/mcp/util.py – MCPUtil.to_function_tool()
@classmethod
def to_function_tool(cls, tool: MCPTool, server: MCPServer, ...) -> FunctionTool:
    invoke_func_impl = functools.partial(cls.invoke_mcp_tool, server, tool)

    return FunctionTool(
        name=tool.name,
        description=tool.description or "",
        params_json_schema=schema,
        on_invoke_tool=invoke_func,
        strict_json_schema=is_strict,
        needs_approval=needs_approval,
    )
```

### MCP Tool Invocation

```python
# src/agents/mcp/util.py – MCPUtil.invoke_mcp_tool()
@classmethod
async def invoke_mcp_tool(cls, server, tool, context, input_json, *, meta=None):
    json_data = json.loads(input_json) if input_json else {}

    # Resolve optional metadata (e.g., auth tokens)
    resolved_meta = await cls._resolve_meta(server, context, tool.name, json_data)
    merged_meta = cls._merge_mcp_meta(resolved_meta, meta)

    # Call the MCP server
    result = await server.call_tool(tool.name, json_data, meta=merged_meta)

    # Convert MCP content items → ToolOutput (text, images, etc.)
    ...
    return tool_output
```

### Connection Lifecycle

MCP servers support async context manager usage:

```python
async with MCPServerStdio(params={...}) as server:
    agent = Agent(name="Assistant", mcp_servers=[server])
    result = await Runner.run(agent, "Do something")
```

The `_MCPServerWithClientSession` base handles:

- Creating transport streams (`create_streams()`)
- Establishing a `ClientSession` and calling `session.initialize()`
- Tool caching (`cache_tools_list` option)
- Retry logic with exponential backoff (`max_retry_attempts`, `retry_backoff_seconds_base`)
- Graceful cleanup with HTTP error extraction

---

## Shell & ApplyPatch Tools

### ShellTool (Next-Gen)

`ShellTool` supports both **local execution** and **hosted container execution**:

```python
# src/agents/tool.py
@dataclass
class ShellTool:
    executor: ShellExecutor | None = None      # Required for local
    environment: ShellToolEnvironment | None = None  # local, container_auto, container_reference
    needs_approval: bool | ShellApprovalFunction = False
    on_approval: ShellOnApprovalFunction | None = None
```

**Local mode** requires an `executor` function:

```python
ShellTool(executor=my_shell_executor)  # environment defaults to {"type": "local"}
```

**Hosted mode** uses OpenAI's container infrastructure:

```python
ShellTool(environment={
    "type": "container_auto",
    "network_policy": {"type": "disabled"},
    "skills": [skill_reference],
})
```

### Skills

`ShellTool` supports **skills** — reusable tool bundles that can be referenced or inlined:

```python
# Skill reference (hosted)
class ShellToolSkillReference(TypedDict):
    type: Literal["skill_reference"]
    skill_id: str
    version: NotRequired[str]

# Inline skill (hosted)
class ShellToolInlineSkill(TypedDict):
    description: str
    name: str
    source: ShellToolInlineSkillSource  # base64-encoded zip
    type: Literal["inline"]

# Local skill
class ShellToolLocalSkill(TypedDict):
    description: str
    name: str
    path: str
```

Skills are **not** MCP servers — they are OpenAI platform features that provide pre-packaged tool capabilities mounted into shell environments.

### ApplyPatchTool

Lets the model apply file diffs via an `ApplyPatchEditor` interface:

```python
@dataclass
class ApplyPatchTool:
    editor: ApplyPatchEditor
    needs_approval: bool | ApplyPatchApprovalFunction = False
    on_approval: ApplyPatchOnApprovalFunction | None = None
```

---

## Agents as Tools

Any `Agent` can be used as a tool via `agent.as_tool(...)`, which creates a `FunctionTool` that runs a nested `Runner.run()`:

```python
translator = Agent(name="Translator", instructions="Translate to Spanish")

orchestrator = Agent(
    name="Orchestrator",
    tools=[
        translator.as_tool(
            tool_name="translate_to_spanish",
            tool_description="Translate text to Spanish",
        ),
    ],
)
```

This is implemented by setting `_is_agent_tool = True` on the resulting `FunctionTool` and storing a reference in `_agent_instance`.

---

## Authentication & Security

Authentication operates at three distinct layers:

### 1. OpenAI API Authentication

The SDK authenticates with OpenAI's API to make model calls and send tracing data:

```python
# src/agents/_config.py
def set_default_openai_key(key: str, use_for_tracing: bool) -> None:
    _openai_shared.set_default_openai_key(key)
    if use_for_tracing:
        set_tracing_export_api_key(key)

def set_default_openai_client(client: AsyncOpenAI, use_for_tracing: bool) -> None:
    _openai_shared.set_default_openai_client(client)
    if use_for_tracing:
        set_tracing_export_api_key(client.api_key)
```

Typically set via the `OPENAI_API_KEY` environment variable.

### 2. MCP Server Authentication

Each MCP transport handles auth differently:

- **Stdio**: Environment variables passed to the subprocess via `params["env"]`.
- **SSE/StreamableHttp**: HTTP headers and auth via the `HttpClientFactory` protocol:

```python
# src/agents/mcp/util.py
class HttpClientFactory(Protocol):
    def __call__(
        self,
        headers: dict[str, str] | None = None,
        timeout: httpx.Timeout | None = None,
        auth: httpx.Auth | None = None,
    ) -> httpx.AsyncClient: ...
```

- **Tool-level metadata**: The `MCPToolMetaResolver` callback injects per-call metadata (e.g., auth tokens) into the MCP `_meta` field:

```python
# src/agents/mcp/util.py
@dataclass
class MCPToolMetaContext:
    run_context: RunContextWrapper[Any]
    server_name: str
    tool_name: str
    arguments: dict[str, Any] | None

MCPToolMetaResolver = Callable[[MCPToolMetaContext], MaybeAwaitable[dict[str, Any] | None]]
```

### 3. Hosted MCP Auth

For `HostedMCPTool`, authentication headers are included in the `Mcp` config object sent directly to the OpenAI API:

```python
HostedMCPTool(tool_config={
    "type": "mcp",
    "server_label": "my_server",
    "server_url": "https://mcp.example.com",
    "headers": {"Authorization": "Bearer <token>"},
})
```

---

## Tool Execution Pipeline

The tool execution pipeline spans several modules:

### 1. Tool Planning (`src/agents/run_internal/tool_planning.py`)

After the LLM returns tool calls, the SDK builds a `ToolExecutionPlan` mapping each tool call to the correct tool instance.

### 2. Tool Execution (`src/agents/run_internal/tool_execution.py`)

For each tool call:

1. **JSON argument coercion** — parse and validate the LLM's arguments.
2. **Input guardrails** — run `ToolInputGuardrail` checks (may reject or halt).
3. **Approval check** — if `needs_approval` is true/returns true, emit a `ToolApprovalItem` and pause.
4. **Invocation** — call `invoke_function_tool()` with timeout enforcement.
5. **Output guardrails** — run `ToolOutputGuardrail` checks on the result.
6. **Result capture** — produce a `FunctionToolResult` with the output and any run items.

### 3. Function Tool Invocation with Timeouts

```python
# src/agents/tool.py
async def invoke_function_tool(*, function_tool, context, arguments):
    timeout_seconds = function_tool.timeout_seconds
    if timeout_seconds is None:
        return await function_tool.on_invoke_tool(context, arguments)

    tool_task = asyncio.ensure_future(function_tool.on_invoke_tool(context, arguments))
    try:
        return await asyncio.wait_for(tool_task, timeout=timeout_seconds)
    except asyncio.TimeoutError as exc:
        timeout_error = ToolTimeoutError(
            tool_name=function_tool.name,
            timeout_seconds=timeout_seconds,
        )
        if function_tool.timeout_behavior == "raise_exception":
            raise timeout_error from exc
        # "error_as_result" → return error message to model
        return default_tool_timeout_error_message(...)
```

### 4. ToolContext

Every tool invocation receives a `ToolContext` that extends `RunContextWrapper`:

```python
# src/agents/tool_context.py
@dataclass
class ToolContext(RunContextWrapper[TContext]):
    tool_name: str
    tool_call_id: str
    tool_arguments: str
    tool_call: ResponseFunctionToolCall | None = None
    agent: AgentBase[Any] | None = None
    run_config: RunConfig | None = None
```

This gives tools access to:

- The user-defined `context` object
- Accumulated `usage` stats
- The current agent and run configuration
- The raw tool call metadata

---

## Tool Guardrails

Guardrails run before (input) and after (output) tool invocations to enforce safety policies:

```python
# src/agents/tool_guardrails.py
@dataclass
class ToolInputGuardrail(Generic[TContext_co]):
    guardrail_function: Callable[[ToolInputGuardrailData], MaybeAwaitable[ToolGuardrailFunctionOutput]]
    name: str | None = None

@dataclass
class ToolOutputGuardrail(Generic[TContext_co]):
    guardrail_function: Callable[[ToolOutputGuardrailData], MaybeAwaitable[ToolGuardrailFunctionOutput]]
    name: str | None = None
```

### Guardrail Behaviors

```python
class ToolGuardrailFunctionOutput:
    output_info: Any
    behavior: RejectContentBehavior | RaiseExceptionBehavior | AllowBehavior

    @classmethod
    def allow(cls, ...):       # Continue normally
    @classmethod
    def reject_content(cls, message, ...):  # Replace output with message
    @classmethod
    def raise_exception(cls, ...):          # Halt the entire run
```

Guardrails are attached per-tool:

```python
@function_tool(
    tool_input_guardrails=[my_input_check],
    tool_output_guardrails=[my_output_check],
)
async def sensitive_tool(...): ...
```

---

## Approval & Human-in-the-Loop

### FunctionTool Approval

Any tool can require approval before execution:

```python
@function_tool(needs_approval=True)
async def deploy_to_production(...): ...
```

When approval is required:

1. The SDK emits a `ToolApprovalItem` instead of executing.
2. The run pauses (interrupts).
3. External code calls `state.approve(tool_call_id)` or `state.reject(tool_call_id)`.
4. The run resumes from `RunState`.

### MCP Server Approval

MCP servers support per-tool or global approval policies:

```python
# src/agents/mcp/server.py
class MCPServer:
    def __init__(self, require_approval: RequireApprovalSetting = None, ...):
        ...

# RequireApprovalSetting can be:
# - "always" / "never"
# - True / False
# - {"tool_name": "always", "other_tool": "never"}
# - {"always": {"tool_names": [...]}, "never": {"tool_names": [...]}}
```

The policy is normalized and applied per-tool:

```python
def _get_needs_approval_for_tool(self, tool, agent):
    policy = self._needs_approval_policy
    if callable(policy):
        # Dynamic: calls policy(run_context, agent, tool)
        ...
    if isinstance(policy, dict):
        return bool(policy.get(tool.name, False))
    return bool(policy)
```

### Shell & ApplyPatch Approval

Both `ShellTool` and `ApplyPatchTool` support:

- `needs_approval`: bool or callable that checks per-action.
- `on_approval`: auto-approval callback for programmatic review.

---

## Tool Filtering

MCP servers support tool filtering to control which tools are exposed to the agent:

### Static Filtering

```python
server = MCPServerStdio(
    params={...},
    tool_filter={
        "allowed_tool_names": ["read_file", "write_file"],  # whitelist
        "blocked_tool_names": ["delete_file"],               # blacklist
    },
)
```

### Dynamic Filtering

```python
async def my_filter(ctx: ToolFilterContext, tool: MCPTool) -> bool:
    if ctx.agent.name == "ReadOnlyAgent":
        return tool.name.startswith("read_")
    return True

server = MCPServerStdio(params={...}, tool_filter=my_filter)
```

The filter receives a `ToolFilterContext` with the run context, agent, and server name.

---

## Timeouts & Error Handling

### Timeouts

```python
@function_tool(timeout=5.0, timeout_behavior="error_as_result")
async def slow_api_call(...): ...
```

- `"error_as_result"` (default): returns `"Tool 'name' timed out after N seconds."` to the model.
- `"raise_exception"`: raises `ToolTimeoutError`, failing the run.
- `timeout_error_function`: custom formatter for timeout messages.

**Note:** Timeouts are only supported for async `@function_tool` handlers.

### Error Handling

```python
# Default error handler (src/agents/tool.py)
def default_tool_error_function(ctx, error):
    json_decode_error = _extract_tool_argument_json_error(error)
    if json_decode_error is not None:
        return f"An error occurred while parsing tool arguments. Error: {json_decode_error}"
    return f"An error occurred while running the tool. Error: {str(error)}"
```

For MCP tools, the `failure_error_function` on the server overrides the agent-level default:

```python
server = MCPServerStdio(
    params={...},
    failure_error_function=my_custom_handler,  # or None to raise
)
```

---

## Summary

| Feature | Implementation | Key File(s) |
|---------|---------------|-------------|
| Function tools | `@function_tool` decorator → `FunctionTool` dataclass | `src/agents/tool.py` |
| Hosted tools | Dataclass configs sent to OpenAI API | `src/agents/tool.py` |
| MCP servers | `MCPServer` base + Stdio/SSE/StreamableHttp | `src/agents/mcp/server.py` |
| MCP→FunctionTool | `MCPUtil.to_function_tool()` | `src/agents/mcp/util.py` |
| Tool context | `ToolContext` extends `RunContextWrapper` | `src/agents/tool_context.py` |
| Guardrails | `ToolInputGuardrail` / `ToolOutputGuardrail` | `src/agents/tool_guardrails.py` |
| Approval | `needs_approval` + `RunState.approve()`/`reject()` | `src/agents/tool.py`, `src/agents/mcp/server.py` |
| Filtering | Static (allowlist/blocklist) or dynamic callable | `src/agents/mcp/util.py` |
| Timeouts | `asyncio.wait_for` with configurable behavior | `src/agents/tool.py` |
| Auth (OpenAI) | `OPENAI_API_KEY` / `set_default_openai_key()` | `src/agents/_config.py` |
| Auth (MCP) | Headers, env vars, `HttpClientFactory`, `MCPToolMetaResolver` | `src/agents/mcp/server.py`, `src/agents/mcp/util.py` |
| Skills | `ShellToolSkillReference`, `ShellToolInlineSkill`, `ShellToolLocalSkill` | `src/agents/tool.py` |
