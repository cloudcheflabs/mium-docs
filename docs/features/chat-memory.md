# Chat & Memory

Mium provides a persistent chat system where users interact with LLMs through a conversational interface. Chat history is stored in the MemoryStore, enabling context-aware conversations across sessions.

## Chat Interface

The Admin UI provides a full-featured chat interface at the root path (`/` and `/c/:sessionId`):

- Create new chat sessions
- Continue previous conversations with full context
- View and manage chat history
- Export conversations to PDF, PPTX, or XLSX

## Chat Sessions

Each chat session maintains:

- **Message History**: All user and assistant messages in the conversation
- **Tool Invocations**: Records of tool calls and their results
- **Context**: The accumulated context that the LLM uses to generate responses

Sessions are identified by unique IDs and can be resumed at any time.

## MemoryStore

The MemoryStore is a RocksDB-backed per-user persistent store for chat sessions and messages:

- **Leader-owned**: The leader Master owns the MemoryStore and handles all writes
- **Synchronized**: Chat data is replicated to follower Masters via the NIO protocol (MEMORY_SYNC opcode)
- **Per-user isolation**: Each user's chat history is stored separately

## Chat API

The Chat endpoints on the Admin HTTP server provide:

- Create a new session
- Send a message and receive a response (with optional tool invocations)
- List sessions for the current user
- Get session history
- Delete sessions

## Export

Conversations can be exported in multiple formats:

- **PDF**: Full conversation with formatting
- **PPTX**: Presentation format for sharing insights
- **XLSX**: Spreadsheet format for data-heavy conversations
