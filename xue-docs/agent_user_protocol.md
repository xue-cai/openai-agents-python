# Agent-User Protocol: How to Expose Agents to End Users

This document answers a common question: **Does the OpenAI Agents Python SDK use a standard protocol between agents and the users/clients interacting with them?**

For example, if you build a travel agent (research plans, book flights/hotels, create Google Calendar events), host it in the cloud, and expose it as a web app — what protocol do users interact with?

---

## Table of Contents

1. [Short Answer](#short-answer)
2. [What the SDK Provides](#what-the-sdk-provides)
3. [What You Need to Add for a Web App](#what-you-need-to-add-for-a-web-app)
4. [Standards the SDK Does Support](#standards-the-sdk-does-support)
5. [Agent-to-Agent (A2A) Protocol](#agent-to-agent-a2a-protocol)
6. [Practical Recommendation for a Travel Agent](#practical-recommendation-for-a-travel-agent)

---

## Short Answer

The OpenAI Agents Python SDK **does not prescribe or enforce a standard protocol** between your hosted agent and end users. It is a **Python library**, not a web framework or protocol specification. The protocol between your agent and users is entirely up to you — REST with JSON, WebSocket for streaming, Server-Sent Events, etc.

---

## What the SDK Provides

The SDK gives you a Python-level API for orchestrating agents:

```python
from agents import Agent, Runner

travel_agent = Agent(name="TravelAgent", instructions="...", tools=[...])
result = await Runner.run(travel_agent, "Plan a trip to Tokyo")
```

There are three invocation styles:

| Method | Style | Returns |
|---|---|---|
| `Runner.run()` | Async | `RunResult` |
| `Runner.run_sync()` | Blocking/synchronous | `RunResult` |
| `Runner.run_streamed()` | Async streaming | Event stream |

These are all **in-process Python calls**. There is no built-in HTTP server, REST API, or standard wire protocol for exposing agents to remote users.

---

## What You Need to Add for a Web App

To host an agent as a web service, **you choose the protocol and web framework yourself**. The most common pattern in the SDK's examples is **FastAPI**:

| Layer | Your Responsibility | SDK's Role |
|---|---|---|
| **HTTP/WebSocket server** | You build with FastAPI, Flask, etc. | Not provided |
| **Wire protocol** (REST, GraphQL, WebSocket, etc.) | You define the API contract | Not provided |
| **Authentication & authorization** | You implement | Not provided |
| **Agent orchestration** | You call `Runner.run()` in your endpoint | ✅ Provided |
| **Session/conversation memory** | You pick a session backend | ✅ Provided (`SQLiteSession`, `RedisSession`, etc.) |
| **Tool execution** (flights, hotels, calendar) | You define tools | ✅ Framework provided |

---

## Standards the SDK Does Support

While there's no user-facing protocol, the SDK integrates with these standards:

### 1. Model Context Protocol (MCP)

A standard for exposing **tools and context to LLMs** (not to end users). You could expose your travel tools as an MCP server so other agents/LLMs can call them.

The SDK supports 4 MCP transports:

| Transport | Class | Best For |
|---|---|---|
| **stdio** | `MCPServerStdio` | Local processes, stdin/stdout |
| **SSE** | `MCPServerSse` | HTTP with streaming events |
| **Streamable HTTP** | `MCPServerStreamableHttp` | Local/remote servers you control |
| **Hosted** | `HostedMCPTool` | OpenAI-managed infrastructure |

Reference: https://modelcontextprotocol.io/

### 2. OpenAI Realtime API (WebSocket)

For **voice-based** agent interactions. If you want users to talk to your travel agent by voice, use `RealtimeRunner` over WebSocket.

The repo includes:
- A FastAPI + WebSocket example (`examples/realtime/app/server.py`)
- A Twilio phone integration (`examples/realtime/twilio/`)

### 3. OpenAI Responses API / Chat Completions API

The protocol between the SDK and the LLM (not between your app and users). This is the model invocation layer.

---

## Agent-to-Agent (A2A) Protocol

There is **no built-in A2A protocol**. Multi-agent coordination (e.g., a "flight agent" handing off to a "hotel agent") happens via **Handoffs** within the same Python process:

```python
flight_agent = Agent(name="FlightAgent", ...)
hotel_agent = Agent(name="HotelAgent", ...)
travel_agent = Agent(name="TravelAgent", handoffs=[flight_agent, hotel_agent])
```

For distributed multi-agent systems across services, you'd need to build your own orchestration.

---

## Practical Recommendation for a Travel Agent

For a typical web app scenario, you'd likely:

1. **Build a FastAPI (or similar) server** that wraps `Runner.run()` calls
2. **Define a REST or WebSocket API** — e.g., `POST /chat` accepts user messages, returns agent responses
3. **Use session persistence** (`RedisSession` for multi-instance deployments) to maintain conversation state across requests
4. **Define your tools** (flight search, hotel booking, Google Calendar) as `FunctionTool`s or MCP servers
5. **Optionally adopt MCP** if you want other agents or LLM systems to reuse your travel tools

### Example Architecture

```
┌─────────────┐     HTTP/WS      ┌──────────────────┐     OpenAI API    ┌─────────────┐
│  Web Client  │ ──────────────▶ │  FastAPI Server   │ ────────────────▶ │  OpenAI LLM  │
│  (Browser)   │ ◀────────────── │  + Runner.run()   │ ◀──────────────── │              │
└─────────────┘   JSON / Events  │  + Session Mgmt   │                   └─────────────┘
                                 │  + Tool Handlers   │
                                 └──────────────────┘
                                        │
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                    Flight API    Hotel API    Google Calendar
```

### Summary Table: Protocol Choices

| Use Case | Protocol | Transport | Framework |
|---|---|---|---|
| Embedded Python app | Direct `Runner` call | In-process | None required |
| REST API with JSON | Custom wrapper | HTTP/REST | FastAPI/Flask (your choice) |
| Real-time voice | WebSocket + Realtime API | WebSocket | FastAPI (recommended) |
| Phone calls | Twilio + Realtime | WebSocket | FastAPI + Twilio SDK |
| Tool exposure to LLMs | MCP | stdio/HTTP/SSE/Streamable HTTP | FastAPI (for HTTP transports) |
| Hosted (OpenAI manages) | Hosted MCP Tool | OpenAI REST API | None (OpenAI infrastructure) |

---

## Conclusion

The OpenAI Agents Python SDK is **not a web framework** — it's a **composable agent orchestration library**. To expose agents to end users, you:

1. **Choose a communication protocol** (REST API, WebSocket, MCP, etc.)
2. **Choose a web framework** (FastAPI recommended; Flask, aiohttp, etc. also work)
3. **Integrate the SDK** by calling `Runner.run()` or `RealtimeRunner.run()` within your endpoints
4. **Optionally add MCP servers** if you need standardized tool exposure

The SDK is deliberately unopinionated about the user-facing protocol to give you maximum flexibility.
