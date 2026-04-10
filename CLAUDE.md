# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup
```bash
# Install dependencies with uv
uv sync

# Copy environment template and add your Anthropic API key
cp .env.example .env
# Edit .env to set ANTHROPIC_API_KEY=your-key-here
```

The application only requires `ANTHROPIC_API_KEY` in the `.env` file; other configuration is set in `backend/config.py`.

### Running the Application
```bash
# Quick start using the shell script
chmod +x run.sh
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: `http://localhost:8000`
- API Documentation: `http://localhost:8000/docs`

### Adding Course Documents
Place `.txt` files in the `docs/` directory with this format:
```
Course Title: [Full Course Title]
Course Link: [URL to course]
Course Instructor: [Instructor Name]

Lesson 0: Introduction
Lesson Link: [URL to lesson]
[Lesson content text...]

Lesson 1: [Lesson Title]
[Lesson content text...]
```

Documents are automatically ingested on application startup via the `startup_event` in `app.py`. If ingestion fails, the application continues running with existing data.

To rebuild the vector store from scratch, delete the `chroma_db` directory or modify the startup code to use `clear_existing=True` in `add_course_folder()`.

## Architecture Overview

This is a Retrieval-Augmented Generation (RAG) system for querying course materials. The system uses a tool-based approach where the LLM decides when to search course content versus using general knowledge.

### Core Data Flow
1. **Document Ingestion**: Course documents in `docs/` are processed into `Course` objects with `Lesson` metadata and `CourseChunk` content chunks.
2. **Vector Storage**: Two ChromaDB collections store (a) course metadata for semantic course matching and (b) chunked lesson content with embeddings.
3. **Query Processing**: User queries trigger the LLM with access to a single search tool; the LLM decides whether and how to search.
4. **Response Generation**: Search results are synthesized into concise, educational answers with source attribution.

### Key Components

#### Backend (`backend/`)
- **`app.py`**: FastAPI application with CORS, static file serving, and API endpoints (`/api/query`, `/api/courses`).
- **`rag_system.py`**: Main orchestrator that wires together document processing, vector storage, AI generation, and session management.
- **`vector_store.py`**: ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (chunks). Provides semantic course name resolution.
- **`document_processor.py`**: Parses course documents with sentence-based chunking (configurable size/overlap). Expects a specific format with lesson markers.
- **`ai_generator.py`**: Anthropic Claude integration with tool calling. Uses a strict system prompt that enforces concise, educational responses and limits to one search per query.
- **`search_tools.py`**: Tool framework with a single `search_course_content` tool that accepts `query`, optional `course_name`, and optional `lesson_number` parameters.
- **`session_manager.py`**: Manages conversation history with configurable message retention.
- **`models.py`**: Pydantic models: `Course`, `Lesson`, `CourseChunk`.
- **`config.py`**: Centralized configuration (chunk size, embedding model, API settings).

#### Frontend (`frontend/`)
Static HTML/CSS/JS interface with:
- Chat interface with markdown rendering
- Course statistics sidebar
- Suggested questions
- Source attribution display

#### Configuration (`config.py`)
Key settings:
- `CHUNK_SIZE = 800`, `CHUNK_OVERLAP = 100` – text chunking parameters
- `MAX_RESULTS = 5` – maximum search results returned
- `MAX_HISTORY = 2` – conversation turns remembered
- `EMBEDDING_MODEL = "all-MiniLM-L6-v2"` – sentence transformer for embeddings (downloaded on first use)
- `ANTHROPIC_MODEL = "claude-sonnet-4-20250514"` – default Claude model
- `CHROMA_PATH = "./chroma_db"` – vector database location

### Tool-Based RAG Pattern
Unlike traditional RAG that always retrieves, this system uses Anthropic's tool calling:
1. LLM receives query with conversation history
2. LLM decides whether to use `search_course_content` tool (course-specific questions) or answer directly (general knowledge)
3. If tool is used, vector store searches with semantic course name matching
4. LLM synthesizes tool results into final answer

This approach reduces unnecessary searches and API calls while maintaining accuracy.

### Vector Store Design
- **Dual collection strategy**: Separates metadata from content for efficient filtering.
- **Course name resolution**: Fuzzy course matching using vector similarity on course titles.
- **Contextual chunks**: Chunks include "Course [title] Lesson [number] content:" prefix for better retrieval relevance.

### File Processing Pipeline
1. Read file with UTF-8 encoding fallback
2. Extract metadata from first 3-4 lines using regex patterns
3. Detect lesson markers (`Lesson X: [Title]`) and optional lesson links
4. Split lesson content using sentence-aware chunking with overlap
5. Create `CourseChunk` objects with course title, lesson number, and sequential chunk index

### Important Design Constraints
- **One search maximum per query**: Enforced by system prompt to control costs
- **No meta-commentary in responses**: LLM instructed to provide direct answers only
- **Course documents must follow exact format**: Parser relies on specific headers and lesson markers
- **Session-based conversations**: Each chat session maintains limited history for context

When modifying the system, maintain the separation between metadata and content collections, and preserve the tool-based decision making for search vs. general knowledge responses.