# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A RAG (Retrieval-Augmented Generation) chatbot system for querying course materials. The system combines ChromaDB vector search with Claude AI's tool-use capabilities to answer questions about educational content.

## Running the Application

**Setup:**
```bash
# Install dependencies
uv sync

# Create .env file with API key
echo "ANTHROPIC_API_KEY=your-key" > .env
```

**Start server:**
```bash
# Quick start
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

**Access:**
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

## Architecture

### Request Flow
```
User Query → FastAPI (app.py) → RAGSystem → AIGenerator → Claude API
                                                              ↓
                                                      [Tool Use Decision]
                                                              ↓
                                              search_course_content tool
                                                              ↓
                                                   VectorStore (ChromaDB)
                                                              ↓
                                              Tool Results → 2nd Claude Call
                                                              ↓
                                                      Final Response → User
```

### Core Components

**rag_system.py** - Central orchestrator
- Coordinates all components (vector store, AI generator, session manager, tools)
- Entry point for query processing: `query(query, session_id)` → (response, sources)
- Manages document ingestion: `add_course_folder()` and `add_course_document()`

**ai_generator.py** - Claude API integration
- Handles tool-use workflow with Claude Sonnet
- **Critical limitation**: Single-turn tool execution only
  - 1st API call: Claude can request tools
  - 2nd API call: Final response WITHOUT tool access
  - Tools parameter is intentionally removed after first execution (line 127-131)
- System prompt enforces "One search per query maximum" (line 12)

**vector_store.py** - ChromaDB interface
- Two collections:
  - `course_catalog`: Course metadata (titles, instructors, lessons) for semantic course name matching
  - `course_content`: Chunked course material for retrieval
- `search()` method (line 61): Unified search interface that resolves course names semantically and filters by course/lesson
- Course name resolution uses vector similarity, not exact matching (line 102)

**search_tools.py** - Tool definitions
- `CourseSearchTool`: Implements Anthropic tool calling interface
- Tracks sources from searches for frontend display via `last_sources`
- `ToolManager`: Registers tools and executes them during Claude's tool-use phase

**session_manager.py** - Conversation persistence
- Maintains conversation history per session_id
- Limited to MAX_HISTORY exchanges (default: 2) to manage context size

**document_processor.py** - Document ingestion
- Parses course documents (TXT, PDF, DOCX)
- Chunks text with overlap (default: 800 chars, 100 overlap)
- Extracts course metadata (title, instructor, lessons)

**app.py** - FastAPI application
- `/api/query`: Main chat endpoint
- `/api/courses`: Course statistics
- Loads documents from `../docs` folder on startup
- Serves frontend from `../frontend`

### Configuration (config.py)

Key settings:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (Sentence Transformers)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db"

## Key Design Patterns

**Tool Execution Pattern:**
The system uses a two-phase tool execution model:
1. Claude receives tools and can request to use them
2. Tools are executed and results added to conversation
3. Second API call is made WITHOUT tools parameter to get final answer

This is NOT a multi-turn agentic loop. If Claude needs more information after the first search, it cannot search again automatically.

**Semantic Course Name Matching:**
Course filtering uses vector similarity, not exact string matching. Users can say "MCP course" or "Introduction" and the system will find the closest matching course title via `_resolve_course_name()` in vector_store.py:102.

**Dual Collection Strategy:**
- Course metadata collection enables semantic course discovery
- Content collection stores actual searchable material
- This separation allows efficient course filtering before content search

## Important Constraints

1. **Single tool execution per query** - After Claude uses the search tool once, it cannot search again in the same query
2. **No direct database access** - All queries go through vector similarity search; no SQL-style exact lookups
3. **Conversation history is limited** - Only last 2 exchanges kept to control context size
4. **Windows compatibility** - Requires Git Bash for running shell scripts

## Data Flow for Adding Courses

```
Document File → DocumentProcessor.process_course_document()
                        ↓
                 (Course, CourseChunks)
                        ↓
    ┌──────────────────┴──────────────────┐
    ↓                                      ↓
VectorStore.add_course_metadata()    VectorStore.add_course_content()
    ↓                                      ↓
course_catalog collection            course_content collection
(for name matching)                  (for semantic search)
```

## Frontend

Simple vanilla JavaScript SPA located in `../frontend`:
- Real-time chat interface with markdown support
- Displays sources from search results
- Maintains session_id for conversation continuity
- No build step required
- Always use UV instead of pip