# Chapter 8: The Model Context Protocol

Companion code for *Building Safe Agentic AI for Enterprise Systems* by Mohit Aggarwal.

This repository demonstrates how the Model Context Protocol (MCP) places a typed boundary between a language model and external systems. The model discovers named tools through an MCP server, supplies arguments that must match published JSON Schemas, and receives structured results. It does not call external APIs, devices, raw sockets, or databases directly.

MCP provides a boundary. It does not make a registered tool safe by itself. Tool business logic, credentials, retry behavior, idempotency, authorization, and transaction safety remain the responsibility of the system behind the tool. Chapters 11 through 14 address those controls.

## What You Will Run

| Chapter sections | Demonstration | What it shows |
| --- | --- | --- |
| 8.1 and 8.2 | MCP primitives | A small FastMCP server exposes tools, resources, and prompt templates, and derives JSON Schema from Python type annotations and docstrings. |
| 8.3 | External API gateway | MCP tools call weather and news services while keeping API keys, URL construction, result limits, parsing, and error handling on the server side. |
| 8.4 | Smart-home actuation | A client agent discovers a fixed tool set from an MCP server and controls either a mock or physical TP-Link Kasa smart plug. |
| 8.2.1 | Tool-schema evolution | Contract changes are treated as compatibility events and should be detected in CI before a client and server drift apart. |
| 8.2.2 | Enterprise gateway design | The same client-server boundary is extended to a controlled gateway pattern. |

## Production Warning

MCP validates the shape of a tool call. It does not prove that the action is permitted, safe, idempotent, or reversible. A valid request to an unsafe tool is still unsafe.

The physical smart-plug example changes a device state. Use the mock server first. When you use a real plug, make sure that it is a nonessential device on a network you control, and never expose its IP address as a tool argument or place it in a model prompt.

## Prerequisites

- Git
- [uv](https://docs.astral.sh/uv/)
- Python 3.12
- A terminal capable of running two local processes for client-server examples
- Optional: an OpenAI-compatible API key for the client agent
- Optional: WeatherAPI.com and NewsAPI.org keys for the external-tools example
- Optional: a TP-Link Kasa smart plug on your local network for the physical-device example

You can run the MCP primitives and mock smart-plug tool tests without physical hardware. The full conversational smart-home client requires the configured model provider.

## Quick Start

### 1. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Clone and synchronize the repository

```bash
git clone https://github.com/the-write-path-code/ch08-model-context-protocol.git
cd ch08-model-context-protocol
uv sync
```

The repository includes `uv.lock`. Run `uv sync` after pulling changes so the environment matches the committed dependency set.

### 3. Create local configuration

```bash
cp .env.example .env
```

The environment file separates credentials by demonstration. Configure only the values needed for the path you plan to run.

## Configuration

| Variable | Used by | Required? | Purpose |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | Smart-home client | Yes for the conversational client | LLM provider credential |
| `WEATHER_API_KEY` | External-tools server | Yes for weather requests | Server-side WeatherAPI.com credential |
| `NEWS_API_KEY` | External-tools server | Yes for news requests | Server-side NewsAPI.org credential |
| `KASA_DEVICE_IP` | Real smart-home server | Yes for physical-device control | Server-configured target device address |
| `KASA_DEVICE_ALIAS` | Real smart-home server | Yes for physical-device control | Human-readable device label |

Do not place credentials or device addresses in source files, model prompts, or client tool arguments. The MCP server reads them from its own environment at startup.

> **Tip**
>
> Begin with the MCP primitives or mock smart-plug server. Both let you inspect schemas and tool results without API keys, cloud services, or a physical device.

## Run the Chapter Demonstrations

### 1. Inspect MCP Primitives, Sections 8.1 and 8.2

The `mcp_primitives` module shows the three basic MCP primitives:

| Primitive | Decorator | Role |
| --- | --- | --- |
| Tool | `@mcp.tool()` | A model-callable operation with typed inputs |
| Resource | `@mcp.resource()` | Read-only data available at a stable URI |
| Prompt | `@mcp.prompt()` | A reusable prompt template offered by the server |

Open the primitives in the MCP development interface:

```bash
uv run mcp dev mcp_primitives/primitives_server.py
```

FastMCP reads function type hints and docstrings, derives the tool schema, and publishes it to clients as `inputSchema`. A malformed argument should become a protocol error before tool logic runs.

To install the primitives server into Claude Desktop, if you use it:

```bash
uv run mcp install mcp_primitives/primitives_server.py --name "Chapter 8 Primitives"
```

### 2. Run the External API Gateway, Section 8.3

Configure `WEATHER_API_KEY` and `NEWS_API_KEY` in `.env`, then start the server:

```bash
uv run mcp dev external_tools/external_tools_server.py
```

The server exposes a weather tool and a news-headlines tool. The model provides only user-facing arguments such as a city or topic. The server owns the API keys, request construction, maximum result count, response parsing, and structured error payloads.

Read `external_tools/README.md` before using this path. API credentials and provider limits change independently of the book.

### 3. Run the Mock Smart Plug, Section 8.4

Use this path before attempting physical actuation.

In Terminal 1, start the mock MCP server:

```bash
uv run python smart_home/mock_kasa_server.py
```

In Terminal 2, validate its tools without a model call:

```bash
uv run python smart_home/test_mock.py
```

Then run the client workflow when the model-provider configuration is available:

```bash
uv run python smart_home/client_kasa_workflow.py
```

The client calls `get_tools()` at runtime and builds its agent from the schemas the server advertises. It does not hard-code the available device operations.

### 4. Run a Physical Kasa Plug, Section 8.4

Set the device variables in `.env`:

```dotenv
KASA_DEVICE_IP=192.168.1.42
KASA_DEVICE_ALIAS=Smart Plug
```

Start the real server in Terminal 1:

```bash
uv run python smart_home/kasa_smart_home_server.py
```

Start the client in Terminal 2:

```bash
uv run python smart_home/client_kasa_workflow.py
```

The real server registers a fixed tool set: list devices, turn the configured device on, turn it off, and retrieve its status. The server loads the target address when it starts. The model has no argument that can redirect the tool to another device.

If you need to locate a Kasa device on your local network, run:

```bash
uv run python -m kasa discover
```

Assign a DHCP reservation before relying on a device IP in an example or local deployment. A consumer router can otherwise assign a different address after a restart.

### 5. Review Tool-Schema Evolution, Section 8.2.1

Tool signatures are API contracts. A renamed argument, changed type, removed field, or newly required field can break clients whose sessions began before the server deployment.

Use the repository's schema-evolution documentation to review this pattern:

1. Register a breaking tool version under a new name.
2. Keep the older version available while active sessions drain.
3. Diff the server's published schemas against the client catalog in CI.
4. Add a client-side adapter only when an older payload can be translated safely.

Do not patch a live tool signature and assume a model or a cached client tool catalog will discover the change on its own.

## Expected Results

### MCP primitives

The development interface should display tools, resources, and prompts with their generated schemas. A tool argument that does not match the schema should be rejected before the tool body executes.

### External API gateway

A successful tool result should be a structured response with only the fields the tool contract allows. A provider error should return a structured error result. It must not look like an empty successful answer.

### Mock smart plug

The mock server should return deterministic state transitions. The test script verifies tool behavior without requiring physical hardware.

### Physical smart plug

The server should control only the configured target. A missing `KASA_DEVICE_IP` should stop the server at startup rather than create a broad discovery or target-selection path for the model.

## Run the Tests

Run the mock-server tests:

```bash
uv run python smart_home/test_mock.py
```

Run the full repository test suite when available:

```bash
uv run pytest
```

Run tests before changing a tool signature, server configuration, client discovery logic, response schema, or error contract. The main properties to preserve are:

- The client discovers tools from the server.
- The server owns target selection and secrets.
- Tool arguments are schema-validated.
- Tools return structured error results on execution failure.
- A server failure is not converted into an empty successful result.

## Repository Layout

```text
.
├── README.md
├── pyproject.toml
├── uv.lock
├── .env.example
├── mcp_primitives/
│   ├── primitives_server.py           # Tools, resources, and prompts
│   └── README.md                      # Sections 8.1 and 8.2
├── external_tools/
│   ├── external_tools_server.py       # Server-side weather and news tools
│   └── README.md                      # Section 8.3 setup and limits
├── smart_home/
│   ├── kasa_smart_home_server.py      # Physical Kasa device server
│   ├── mock_kasa_server.py            # No-hardware MCP server
│   ├── client_kasa_workflow.py        # Runtime tool discovery and client agent
│   └── test_mock.py                   # Mock tool tests
├── workflow/
│   └── workflow.md                    # Mermaid diagrams for Chapter 8
└── tests/
```

## Architecture Diagrams and Supporting Documents

The `workflow/workflow.md` document contains the Chapter 8 diagrams:

- The failure mode when a model calls unrelated tools through bespoke code.
- The MCP client-server boundary.
- Schema generation from Python type annotations and docstrings.
- Bounded execution, including server-owned device targeting.
- External API gateway behavior.
- The smart-home request lifecycle from user instruction through server response.
- Tool discovery and schema evolution.

The module-level READMEs contain the detailed commands for each demonstration. The root README gives the reading order and the safety boundary shared by all three.

## Safety and Operational Limits

- MCP restricts what a model can discover and call. It does not make a registered tool's business logic correct or authorized.
- MCP schema validation checks argument shape. It does not prevent a correctly shaped request from causing an unsafe action.
- Do not expose raw SQL, raw sockets, network scanners, file-system controls, or arbitrary target addresses as model-callable tools.
- Keep secrets and fixed targets in the server environment. Never return them in tool results.
- Empty lists are not always successful results. A device connection failure should return a typed error, not an empty device list that a model might misread as a successful scan.
- MCP does not provide idempotency, rollback, transaction isolation, approval gates, or retry-safe writes. Apply the persistence and fail-closed controls in Chapters 11 through 14 when a tool can change state.

## Troubleshooting

### `mcp` command is not found

Synchronize the project environment and run the command through uv:

```bash
uv sync
uv run mcp --help
```

### A client cannot discover tools

Confirm that the server is running, that the client uses the configured MCP endpoint, and that the server has successfully registered its tools. Inspect server logs before changing the client prompt or tool list.

### A tool fails schema validation

Compare the client arguments with the server's published `inputSchema`. Do not relax the server schema just to accept a malformed payload. Treat an unexpected schema failure as a compatibility issue between client and server versions.

### The mock smart plug works but the physical plug does not

Confirm that the device is on the same network as the server, that `KASA_DEVICE_IP` is correct, and that the device responds to `uv run python -m kasa discover`. Do not add model-driven network scanning as a workaround.

### An external API tool returns no data

Inspect the server's structured error result and provider logs. A missing key, rate limit, or network failure must remain visible to the caller. Do not convert it into an empty result.

## Related Chapters

- Chapter 2 separates deterministic stages from model-driven stages and defines the Agentic Context Layer.
- Chapter 7 uses deterministic tools and a side channel to protect sensitive staffing and map state.
- Chapter 9 applies specialist tools and MCP boundaries to context-aware agricultural recommendations.
- Chapters 11 through 13 place idempotency and concurrency controls at the write boundary.
- Chapter 14 adds fail-closed policy gates and human approval holds for sensitive actions.
- Chapter 15 tests tool schemas and safety boundaries continuously in CI.

## License and Errata

See `LICENSE` for licensing terms. Report documentation or code issues through this repository's GitHub issue tracker.
