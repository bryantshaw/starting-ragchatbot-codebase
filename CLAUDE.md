# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Install dependencies (from project root)
uv sync

# Start the server (from project root)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

Requires a `.env` file in the project root with `ANTHROPIC_API_KEY=...`.

The app serves both the API and frontend at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

## Architecture

This is an **agentic RAG** system — Claude decides when to search rather than retrieval being pre-fetched before generation.

**Query flow:**
1. Frontend (`frontend/`) POSTs `{ query, session_id }` to `POST /api/query`
2. `app.py` creates a session if needed and delegates to `RAGSystem.query()`
3. `rag_system.py` loads conversation history, then calls `AIGenerator.generate_response()` with tool definitions
4. Claude decides to call the `search_course_content` tool → `CourseSearchTool.execute()` → `VectorStore.search()`
5. ChromaDB performs cosine similarity search on embedded chunks and returns top-N results
6. Claude synthesizes retrieved chunks into a response; sources are collected from `ToolManager` and returned alongside the answer

**Key design decisions:**
- `course_title` is used as the ChromaDB document ID — titles must be unique across all loaded documents
- ChromaDB uses two collections: `course_catalog` (course-level metadata for fuzzy name resolution) and `course_content` (chunked lesson text for retrieval)
- Conversation history is stored in-memory in `SessionManager` (not persisted across server restarts); capped at `MAX_HISTORY * 2` messages
- The tool call pattern is single-turn: one tool use max per query (enforced by the system prompt, not code)

**Configuration** (`backend/config.py`):
- `CHUNK_SIZE`: 800 chars, `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 chunks returned per search
- `MAX_HISTORY`: 2 exchanges retained per session
- `CHROMA_PATH`: `./chroma_db` (relative to `backend/`)
- `EMBEDDING_MODEL`: `all-MiniLM-L6-v2` (via Sentence Transformers)

## Document Format

Course documents go in the `/docs` folder and are loaded on server startup. Expected plain text format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <lesson title>
...
```

Supported file types: `.txt`, `.docx`, `.pdf`. To reload documents without duplicates, the system checks existing course titles in ChromaDB before inserting. To force a full rebuild, call `rag_system.add_course_folder(path, clear_existing=True)`.

