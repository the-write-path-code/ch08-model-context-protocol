# Chapter 8 — Workflow Diagrams

This file contains Mermaid diagrams for each section of **Chapter 8: The Model Context Protocol (MCP)**.
Each diagram is self-contained and can be dropped directly into the corresponding section of the manuscript.


## Section 8.1 — Why Tool Use Needs a Standard Boundary

> **Diagram 8.1a — The Problem: No Standard Boundary**
>
> The "before" picture. An LLM that is stateless and has no undo button
> calls tools directly with bespoke code and no schema validation.
> Every call to a write-capable tool is a potential irreversible action.
> Use this to motivate why a standard protocol boundary is non-negotiable
> in production — connecting directly to §1.1 ("no undo button").

```mermaid
%%{init: {"themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    U(["User"])
    LLM["LLM\nStateless · No undo\nNo validation"]
    T1["Tool A\nRead-only API\n(low risk)"]
    T2["Tool B\nDatabase\ndirect write"]
    T3["Tool C\nSmart Plug\nraw TCP"]
    T4["Tool D\nFile System\nunrestricted"]
    WARN["⚠️ Every tool call is\npotentially irreversible\nNo schema · No boundary\nNo rollback"]

    U -->|natural language| LLM
    LLM -->|"bespoke code\nno schema"| T1
    LLM -->|"bespoke code\nno schema"| T2
    LLM -->|"bespoke code\nno schema"| T3
    LLM -->|"bespoke code\nno schema"| T4
    T2 & T3 & T4 --> WARN

    style LLM fill:#f87171,color:#fff
    style T2 fill:#fbbf24,stroke:#ef4444,stroke-width:2px,stroke-dasharray:5 5
    style T3 fill:#fbbf24,stroke:#ef4444,stroke-width:2px,stroke-dasharray:5 5
    style T4 fill:#fbbf24,stroke:#ef4444,stroke-width:2px,stroke-dasharray:5 5
    style WARN fill:#fef2f2,stroke:#ef4444,stroke-width:2px,color:#b91c1c
```

> **Diagram 8.1b — The Solution: MCP as a Standard Boundary**
>
> The "after" picture. Every tool is registered on an MCP server.
> The model only sees named tools with documented schemas;
> it cannot bypass the boundary or reach external systems directly.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    U(["User"])
    LLM["LLM / Agent"]

    subgraph MCP_BOUNDARY ["MCP Boundary"]
        direction TB
        MCPClient["MCP Client\n(MultiServerMCPClient)"]
        MCPServer["MCP Server\n(FastMCP)"]
        Tools["Registered Tools\n• fetch_weather\n• turn_device_on\n• turn_device_off\n• get_device_status"]
        MCPClient -->|"Tool call + validated args"| MCPServer
        MCPServer --> Tools
    end

    ExtSys["External Systems\n(APIs / Devices / DBs)"]

    U -->|natural language| LLM
    LLM -->|"tool name + typed args"| MCPClient
    Tools -->|"structured result"| MCPClient
    MCPClient -->|"result"| LLM
    LLM -->|"natural language response"| U
    Tools -->|"safe, bounded calls"| ExtSys
```

---

## Section 8.2 — Schema Validation and Bounded Execution Contexts

> **Diagram 8.2a — How FastMCP Generates a JSON Schema from Python Type Hints**
>
> The path from a Python function signature to the JSON Schema the model
> receives at startup. FastMCP introspects type hints and docstrings automatically —
> the model can never pass a wrong type because the schema is enforced
> before execution, not after.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart LR
    subgraph SERVER ["MCP Server (primitives_server.py)"]
        direction TB
        PY["Python function\n\n@mcp.tool()\ndef add_note(message: str) -> str"]
        SCHEMA["Auto-generated JSON Schema\n\n{\n  name: 'add_note',\n  inputSchema: {\n    type: 'object',\n    properties: {\n      message: { type: 'string' }\n    },\n    required: ['message']\n  }\n}"]
        PY -->|"FastMCP introspects\ntype hints + docstring"| SCHEMA
    end

    MODEL["LLM"]
    SCHEMA -->|"Schema advertised\nto model at startup"| MODEL
    MODEL -->|"{ message: 'Buy milk' }\n(validated before execution)"| PY
```

> **Diagram 8.2b — Bounded Execution: What the Model Can and Cannot Do**
>
> The execution boundary enforced by the smart home server.
> The model can call exactly the four exposed tools; it cannot discover
> other devices, change the target IP, or reach the network directly.
> The device IP is server-controlled, loaded from the environment — the
> model never sees it.

```mermaid
flowchart TD
%%{init: {"themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
    MODEL["LLM / Agent"]

    subgraph ALLOWED ["✅ Within the Execution Boundary"]
        T1["list_smart_devices"]
        T2["turn_device_on"]
        T3["turn_device_off"]
        T4["get_device_status"]
    end

    subgraph BLOCKED ["🚫 Outside the Execution Boundary"]
        B1["Choose a different\ndevice IP"]
        B2["Scan the local\nnetwork"]
        B3["Access the file\nsystem"]
        B4["Make raw TCP\ncalls"]
    end

    ENV[".env file\nKASA_DEVICE_IP\n(server-controlled)"]

    MODEL --> ALLOWED
    MODEL -. "blocked by MCP boundary" .-> BLOCKED
    ENV -->|"read once at startup"| T1
    ENV -->|"read once at startup"| T2
    ENV -->|"read once at startup"| T3
    ENV -->|"read once at startup"| T4

    style ALLOWED fill:#f0fdf4,stroke:#22c55e,stroke-width:2px
    style BLOCKED fill:#fef2f2,stroke:#ef4444,stroke-width:2px,stroke-dasharray:6 4
    style ENV fill:#fef9c3,stroke:#ca8a04
    style MODEL fill:#3b82f6,color:#fff
```

---

## Section 8.3 — Connecting Agents to External Systems Safely

> **Diagram 8.3 — Safe External API Gateway Pattern**
>
> The MCP server acts as an API gateway. The model supplies only
> user-facing arguments; the server owns auth, caps, parsing, and error
> handling before returning a clean structured response.
> Three safety annotations are shown inline: env-only keys, trimmed
> payloads, and server-side result caps.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
sequenceDiagram
    actor User
    participant Agent as LLM Agent
    participant Client as MCP Client
    participant Server as MCP Server<br/>(external_tools_server.py)
    participant WeatherAPI as WeatherAPI.com
    participant NewsAPI as NewsAPI.org

    User->>Agent: "What's the weather in Austin?<br/>And get me 3 AI news headlines."

    Note over Agent: Reasons about intent,<br/>selects tools

    Agent->>Client: fetch_weather(city="Austin")
    Client->>Server: Tool call: fetch_weather
    Note over Server: Loads WEATHER_API_KEY<br/>from env (never from model)
    Server->>WeatherAPI: GET /current.json?q=Austin
    WeatherAPI-->>Server: Raw JSON (full payload)
    Note over Server: Parses + trims:<br/>returns 7 fields only
    Server-->>Client: {city, temp_c, condition, ...}
    Client-->>Agent: Structured weather dict

    Agent->>Client: fetch_top_headlines(topic="AI agents", max_results=3)
    Client->>Server: Tool call: fetch_top_headlines
    Note over Server: Caps max_results ≤ 10<br/>server-side (model cannot override)
    Server->>NewsAPI: GET /v2/everything?q=AI+agents&pageSize=3
    NewsAPI-->>Server: Raw JSON (full payload)
    Note over Server: Extracts title, source,<br/>published_at, url only
    Server-->>Client: {topic, total_found, articles[3]}
    Client-->>Agent: Structured headlines dict

    Agent->>User: "In Austin it's 91°F and sunny.<br/>Here are 3 recent AI headlines..."
```

---

## Section 8.4 — Building an MCP Server with a Smart Plug Example

> **Diagram 8.4a — End-to-End Architecture**
>
> The full system: two separate processes connected over HTTP via the MCP
> protocol. The agent process never imports python-kasa; the server process
> never imports LangGraph. The `.env` file is the only shared secret surface.
> The client uses ChatOpenAI when `OPENAI_API_KEY` is set.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    U(["User"])

    subgraph AGENT_SIDE ["Agent Process (client_kasa_workflow.py)"]
        direction TB
        LLM["LLM — OpenAI at startup\ngpt-5.4-nano"]
        REACT["LangGraph\nReAct Agent"]
        MCPC["MultiServerMCPClient\nhttp://localhost:8000/mcp"]
        LLM <--> REACT
        REACT <--> MCPC
    end

    subgraph SERVER_SIDE ["Server Process"]
        direction TB
        SRVLABEL["kasa_smart_home_server.py (real)\nOR mock_kasa_server.py (no hardware)"]
        FASTMCP["FastMCP Server\n(streamable-http, port 8000)"]
        TOOLS["Registered Tools\n• list_smart_devices\n• turn_device_on\n• turn_device_off\n• get_device_status"]
        KASA_SDK["python-kasa SDK\n(real server only)"]
        SRVLABEL --> FASTMCP
        FASTMCP --> TOOLS
        TOOLS --> KASA_SDK
    end

    PLUG["TP-Link Kasa\nSmart Plug\n(local network)"]
    ENV[".env\nKASA_DEVICE_IP\nOPENAI_API_KEY"]

    U -->|"Natural language command"| REACT
    MCPC <-->|"HTTP — MCP protocol"| FASTMCP
    KASA_SDK <-->|"Wi-Fi (python-kasa)"| PLUG
    PLUG -->|"Power state feedback"| KASA_SDK
    ENV -->|"Loaded at startup"| FASTMCP
    ENV -->|"Loaded at startup"| MCPC
    REACT -->|"Natural language response"| U

```

> **Diagram 8.4b — Step-by-Step Request Lifecycle**
>
> Maps the six numbered steps to every participant in the stack.
> Use this alongside the code walkthrough in §8.4 to show
> exactly where the MCP protocol boundary sits in the call chain.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
sequenceDiagram
    actor User
    participant Agent as ReAct Agent<br/>(client_kasa_workflow.py)
    participant LLM as LLM<br/>(OpenAI)
    participant MCPClient as MultiServerMCPClient
    participant MCPServer as FastMCP Server
    participant Kasa as python-kasa SDK
    participant Plug as Smart Plug

    Note over User,Plug: STEP 1 — User speaks their wish
    User->>Agent: "Turn on the Smart Plug."

    Note over Agent,LLM: STEP 2 — The AI thinks and plans
    Agent->>LLM: Prompt + tool schemas
    LLM-->>Agent: Tool choice: turn_device_on()

    Note over Agent,MCPServer: STEP 3 — The client makes the call
    Agent->>MCPClient: turn_device_on()
    MCPClient->>MCPServer: POST /mcp — tool: turn_device_on

    Note over MCPServer,Plug: STEP 4 — The server gets to work
    MCPServer->>Kasa: plug.turn_on()
    Kasa->>Plug: Wi-Fi command: power ON

    Note over Plug,MCPServer: STEP 5 — The plug responds
    Plug-->>Kasa: ACK (is_on: true)
    Kasa-->>MCPServer: State updated
    MCPServer-->>MCPClient: {alias, is_on: true, status: success}

    Note over MCPClient,User: STEP 6 — The AI reports back
    MCPClient-->>Agent: Structured result
    Agent->>LLM: Format response
    LLM-->>Agent: "The Smart Plug is now on."
    Agent-->>User: "The Smart Plug is now on."
```

> **Diagram 8.4c — Four-Step Workflow: Tool Calls and Return Values**
>
> Reference table for the four `agent.ainvoke()` calls in
> `client_kasa_workflow.py`.

| Step | User instruction | Tool called | Return value |
|------|-----------------|-------------|--------------|
| 1 | "List all smart home devices" | `list_smart_devices()` | `[{alias, is_on: ?, host}]` |
| 2 | "Turn on the Smart Plug" | `turn_device_on()` | `{alias, is_on: true, status: success}` |
| 3 | "What is the status?" | `get_device_status()` | `{alias, is_on: true, status: success}` |
| 4 | "Turn off the Smart Plug" | `turn_device_off()` | `{alias, is_on: false, status: success}` |

> **Diagram 8.4d — MCP Tool Registration and Discovery**
>
> How `@mcp.tool()` decorators become a runtime tool manifest and
> reach the agent. FastMCP's introspection mechanism — not manual wiring —
> generates the JSON Schema for each tool automatically at server startup.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart LR
    subgraph SERVER ["kasa_smart_home_server.py / mock_kasa_server.py"]
        DECS["@mcp.tool() decorators\n(4 tools registered)\nlist_smart_devices\nturn_device_on\nturn_device_off\nget_device_status"]
        INTROSPECT["FastMCP introspection\nat server startup:\nreads type hints + docstrings\ngenerates JSON Schema per tool"]
        DECS -->|"decoration"| INTROSPECT
    end

    subgraph REGISTRY ["FastMCP Tool Registry"]
        R["Tool manifest\n{ name, description,\n  inputSchema } × 4"]
    end

    subgraph CLIENT ["client_kasa_workflow.py"]
        GE["client.get_tools()\n(fetches manifest at runtime)"]
        AGENT["create_react_agent\n(model, tools=tools)"]
        GE -->|"LangChain tool wrappers"| AGENT
    end

    INTROSPECT -->|"stored in registry"| R
    R -->|"served at GET /mcp"| GE

```

---

## Section 8.5 — Summary

> **Diagram 8.5 — What MCP Enforces vs. What It Does Not**
>
> A capstone reference table for the chapter summary.

| Dimension | What MCP enforces | What MCP does NOT enforce |
|-----------|-------------------|---------------------------|
| **Tool discovery** | Model sees only registered, named tools with schemas | Whether those tools are safe to call |
| **Argument types** | FastMCP validates types before execution | Business logic correctness of the arguments |
| **Auth / secrets** | API keys and device IPs live in env, never exposed to model | Rotation, expiry, or revocation of those secrets |
| **Execution scope** | Model cannot call tools outside the registered set | What the registered tools themselves can do |
| **Response shape** | Server controls what fields are returned to the model | Whether the model interprets those fields correctly |
| **Irreversibility** | Bounded — model cannot pick arbitrary targets | MCP does not add undo/rollback to write operations |

> 💡 **The gap in the last row is the bridge to Chapter 14.** MCP enforces *who* can call *what* —
> but it does not make write operations safe to retry or rollback. That requires
> the idempotency and fail-closed patterns covered in Part 4.
