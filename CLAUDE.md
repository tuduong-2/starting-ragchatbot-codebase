# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

Requires an `ANTHROPIC_API_KEY` in a `.env` file at the project root (copy `.env.example`).

```bash
# Install dependencies
uv sync

# Start the server (from project root)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

App is served at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

On first startup, the server ingests all `.txt` files from `docs/` into ChromaDB automatically. Re-runs skip already-indexed courses by title.

There is no test suite.

## Architecture

**FastAPI** (`backend/app.py`) serves two API endpoints and mounts the `frontend/` directory as static files — the frontend and backend run from the same origin on port 8000.

The core flow for a query is:

1. `RAGSystem` (`rag_system.py`) is the main orchestrator. It wraps the user query and passes it to `AIGenerator` along with conversation history and the registered tool definitions.
2. `AIGenerator` (`ai_generator.py`) makes a **two-call pattern** to Claude: the first call includes the `search_course_content` tool with `tool_choice: auto`. If Claude decides to search, `_handle_tool_execution()` runs the tool and makes a second call with the results appended to the message thread. If Claude answers directly, there is only one call.
3. `CourseSearchTool` (`search_tools.py`) executes the search: it calls `VectorStore.search()`, which optionally resolves a fuzzy course name via vector similarity against the `course_catalog` collection, then queries the `course_content` collection filtered by course/lesson.
4. `VectorStore` (`vector_store.py`) wraps ChromaDB with two persistent collections: `course_catalog` (one doc per course, used for fuzzy name resolution) and `course_content` (chunked lesson text, used for semantic retrieval). Embeddings use `all-MiniLM-L6-v2` via `sentence-transformers`.
5. `SessionManager` (`session_manager.py`) stores conversation history in memory (keyed by session ID). History is injected into the Claude system prompt as plain text, capped at the last 2 exchanges.

**Document format** (`document_processor.py`): `.txt` files must start with `Course Title:`, `Course Link:`, and `Course Instructor:` on the first three lines, followed by `Lesson N: Title` / `Lesson Link:` markers. Content is split into sentence-based chunks (800 chars, 100 char overlap).

**Tool extensibility**: new tools can be added by subclassing `Tool` (abstract base in `search_tools.py`) and registering with `ToolManager.register_tool()`. The tool definition is passed directly to the Claude API.

## Key config (`backend/config.py`)

| Setting | Default | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Generation model |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Local embedding model |
| `CHUNK_SIZE` | 800 | Characters per chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between chunks |
| `MAX_RESULTS` | 5 | ChromaDB results per search |
| `MAX_HISTORY` | 2 | Conversation exchanges to keep |
| `CHROMA_PATH` | `./chroma_db` | Persistent vector DB location (relative to `backend/`) |
