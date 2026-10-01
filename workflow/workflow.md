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
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    U(["User"]) -->|"Natural language"| LLM["LLM<br/>Stateless · No undo<br/>No validation"]
    LLM -->|"Bespoke code<br/>(no schema)"| T1["Tool A<br/>Read-only API<br/>(low risk)"]
    LLM -->|"Bespoke code<br/>(no schema)"| T2["Tool B<br/>Database<br/>direct write"]
    LLM -->|"Bespoke code<br/>(no schema)"| T3["Tool C<br/>Smart Plug<br/>raw TCP"]
    LLM -->|"Bespoke code<br/>(no schema)"| T4["Tool D<br/>File System<br/>unrestricted"]
    T2 & T3 & T4 --> WARN["<div style='min-width: 520px;'>⚠️ <b>Unchecked Execution</b><br/>Every tool call is potentially irreversible<br/>No schema · No boundary · No rollback</div>"]
```

> **Diagram 8.1b — The Solution: MCP as a Standard Boundary**
>
> The "after" picture. Every tool is registered on an MCP server.
> The model only sees named tools with documented schemas;
> it cannot bypass the boundary or reach external systems directly.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    U(["User"]) <-->|"Natural language"| LLM["LLM / Agent"]
    LLM <-->|"Tool name + typed args"| MCPClient

    subgraph MCP_BOUNDARY ["MCP Boundary (Mediated Security Layer)"]
        direction TB
        MCPClient["MCP Client (MultiServerMCPClient)"]
        MCPServer["MCP Server (FastMCP)"]
        MCPClient <-->|"Validated JSON-RPC & structured results"| MCPServer

        T1["fetch_weather<br/>(read-only API)"]
        T2["turn_device_on<br/>(bounded action)"]
        T3["turn_device_off<br/>(bounded action)"]
        T4["get_device_status<br/>(read-only status)"]

        MCPServer <--> T1 & T2 & T3 & T4
    end

    ExtSys["External Systems (APIs / Physical Devices / Databases)"]

    T1 & T2 & T3 & T4 <--> ExtSys
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
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
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
flowchart TD
    TOP["<div style='min-width: 650px;'><b>User Request & Agent Tool Selection</b><br/>User: 'Weather in Austin & 3 AI headlines'<br/>➔ LLM Agent dispatches <code>fetch_weather</code> & <code>fetch_top_headlines</code></div>"]

    subgraph GW ["<b>MCP Server: Safe External API Gateway</b> (external_tools_server.py)"]
        direction TB

        subgraph ROW1 ["Tool 1: Weather Flow (Auth Isolation & Response Trimming)"]
            direction LR
            W1["<b>1. Call</b><br/><code>fetch_weather(city='Austin')</code>"]
            W2["<b>2. Auth Isolation</b><br/>Loads <code>WEATHER_API_KEY</code> from <code>.env</code><br/><i>(Key never exposed to model)</i>"]
            W3["<b>3. Response Trimming</b><br/>Trims raw JSON to 7 safe fields<br/><code>{city, temp_c, condition, ...}</code>"]
            W1 --> W2 --> W3
        end

        subgraph ROW2 ["Tool 2: News Flow (Server-Side Rate & Volume Cap)"]
            direction LR
            N1["<b>1. Call</b><br/><code>fetch_top_headlines(topic='AI', max=3)</code>"]
            N2["<b>2. Server Boundary</b><br/>Enforces cap: <code>max_results ≤ 10</code><br/><i>(Server-side hard limit)</i>"]
            N3["<b>3. Schema Minimization</b><br/>Extracts minimal 3 safe fields<br/><code>{topic, total_found, articles[3]}</code>"]
            N1 --> N2 --> N3
        end

        ROW1 ~~~ ROW2
    end

    BOT["<div style='min-width: 650px;'><b>Synthesized Final Response to User</b><br/>'In Austin: 91°F, sunny. Top 3 AI headlines: 1. DeepMind announces...'</div>"]

    TOP --> GW
    GW --> BOT
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
    U(["User"]) -->|"Natural language command"| REACT["LangGraph ReAct Agent"]

    subgraph AGENT_SIDE ["Agent Process (client_kasa_workflow.py)"]
        REACT -->|"Prompt"| LLM["LLM Client (gpt-5.4-nano)"]
        REACT -->|"Tool call"| MCPC["MultiServerMCPClient (port 8000)"]
    end

    subgraph SERVER_SIDE ["Server Process (kasa_smart_home_server.py)"]
        FASTMCP["FastMCP Server (streamable-http:8000)"]
        FASTMCP --> TOOLS["Registered Tools (4 operations)"]
        FASTMCP --> KASA_SDK["python-kasa SDK (hardware driver)"]
    end

    PLUG["TP-Link Kasa Smart Plug (local network)"]
    ENV[".env: KASA_DEVICE_IP · OPENAI_API_KEY"]

    MCPC -->|"HTTP — MCP protocol"| FASTMCP
    KASA_SDK -->|"Local Wi-Fi"| PLUG
    ENV -.->|"Loaded at startup"| AGENT_SIDE
    ENV -.->|"Loaded at startup"| SERVER_SIDE
```

> **Diagram 8.4b — Step-by-Step Request Lifecycle**
>
> Maps the six numbered steps to every participant in the stack.
> Use this alongside the code walkthrough in §8.4 to show
> exactly where the MCP protocol boundary sits in the call chain.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    subgraph STACK ["End-to-End Request Lifecycle (Six Sequential Steps)"]
        direction TB

        S1["<div style='min-width: 750px;'><b>Step 1 — User Instruction:</b> User issues natural language command<br/><code>'Turn on the Smart Plug.'</code> ➔ Received by LangGraph ReAct Agent</div>"]

        S2["<div style='min-width: 750px;'><b>Step 2 — Model Reasoning & Tool Selection:</b> Agent queries LLM with tool schemas<br/>Model reasoning resolves intent ➔ Selected tool: <code>turn_device_on()</code></div>"]

        S3["<div style='min-width: 750px;'><b>Step 3 — Protocol Dispatch:</b> <code>MultiServerMCPClient</code> dispatches validated call<br/>HTTP JSON-RPC request: <code>POST /mcp (tool: turn_device_on)</code> ➔ FastMCP Server</div>"]

        S4["<div style='min-width: 750px;'><b>Step 4 — Hardware Command Execution:</b> FastMCP Server invokes <code>python-kasa</code> SDK<br/>SDK translates tool call to local Wi-Fi command: <code>plug.turn_on()</code> ➔ Smart Plug</div>"]

        S5["<div style='min-width: 750px;'><b>Step 5 — Device ACK & State Update:</b> Smart Plug confirms execution: <code>{is_on: true}</code><br/>Server receives ACK and formats structured JSON: <code>{alias, is_on: true, status: success}</code></div>"]

        S6["<div style='min-width: 750px;'><b>Step 6 — Synthesized Conversational Response:</b> MCP Client returns result to Agent<br/>Agent synthesizes final confirmation ➔ <code>'The Smart Plug is now on.'</code> ➔ User</div>"]

        S1 ==> S2 ==> S3 ==> S4 ==> S5 ==> S6
    end
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
