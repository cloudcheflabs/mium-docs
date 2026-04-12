# Multi-Agent AI Platform

Mium is an on-prem-first multi-agent AI platform that enables users to chat with LLMs which can call external tools via per-user stored connections. Instead of building custom integrations for each AI-powered workflow, Mium provides a unified platform where any external system becomes a tool the LLM can invoke.

## Why Mium?

Organizations adopting LLMs face a common challenge: connecting AI to internal systems securely without exposing credentials, without cloud dependencies, and without building custom middleware for each integration. Mium solves this by providing:

- A **credential vault** (ConnectionStore) where each user stores their own tool credentials
- A **tool SPI** that exposes any external system to the LLM as a callable tool
- A **multi-agent orchestration** layer that coordinates planner, executor, and specialist agents
- A **sovereign deployment** model — all data stays on your infrastructure

## Core Concepts

### Agents

Mium supports multi-agent patterns where different agents handle different aspects of a conversation:

- **Planner Agent**: Decomposes complex user requests into executable steps
- **Executor Agent**: Invokes tools and processes results
- **Specialist Agent**: Domain-specific agents for particular tool types

The agent loop orchestrates these patterns, deciding when to call tools, when to respond to the user, and when to delegate to specialist agents.

### Tools

Every external system is a Tool — a pluggable component that the LLM can invoke. Each tool provides metadata and an SQL reference that gets injected into the LLM system prompt, enabling the model to understand what the tool can do and how to use it.

Built-in tools:

- **Ontul SQL**: Query the Ontul distributed data engine via SQL

Future tools:

- **Snowflake**: Query Snowflake data warehouse
- **GitHub**: Interact with repositories, issues, and pull requests
- **Slack**: Send messages and interact with channels

### Connections

Each user manages their own connections to external systems. Credentials are stored encrypted in the ConnectionStore and are never shared between users. When the LLM invokes a tool, Mium retrieves the calling user's credentials for that tool — the tool's own access control decides what data the user can access.

## How It Works

1. User sends a message via the Chat UI or API
2. The agent loop passes the message to the LLM along with available tool descriptions
3. The LLM decides whether to respond directly or invoke a tool
4. If a tool is invoked, Mium retrieves the user's credentials and executes the tool call on a Worker node
5. Tool results are fed back to the LLM for interpretation
6. The LLM generates a response to the user
7. The conversation is persisted in the MemoryStore for future context

## Deployment Model

Mium is designed for sovereign AI deployments:

- All state lives in embedded RocksDB — no external database required
- ZooKeeper is the only external dependency for cluster coordination
- No data leaves your network
- Users bring their own LLM API keys via the ConnectionStore
