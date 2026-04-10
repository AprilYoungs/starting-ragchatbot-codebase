# Query Flow Diagram

This document traces the complete flow of a user query from the frontend through the backend RAG system and back to the frontend response.

## Overview

The system uses a tool-based RAG approach where the LLM decides whether to search course content versus using general knowledge. The flow involves:

1. Frontend user interface
2. FastAPI backend endpoints
3. RAG system orchestration
4. Tool-based AI generation with Claude
5. Vector store search with ChromaDB
6. Response synthesis and return

## Detailed Flow

```mermaid
flowchart TD
    User[User Input] --> Frontend
    
    subgraph Frontend [Frontend - Static HTML/JS]
        UI[Chat Interface] -->|User types query| JS[script.js]
        JS -->|sendMessage()| API1[POST /api/query]
        API1 -->|JSON request| Backend
    end
    
    subgraph Backend [Backend - FastAPI]
        API[app.py] -->|query_documents()| RAG[rag_system.py]
        RAG -->|query()| AI[ai_generator.py]
        AI -->|generate_response()| Claude[Claude API]
        Claude -->|Tool use decision| ToolDecision{Use search tool?}
        
        ToolDecision -->|Yes| ToolCall[search_course_content]
        ToolDecision -->|No| DirectResponse[Direct answer]
        
        ToolCall -->|execute()| SearchTool[search_tools.py]
        SearchTool -->|search()| VectorStore[vector_store.py]
        VectorStore -->|search()| ChromaDB[ChromaDB Collections]
        ChromaDB -->|Results| VectorStore
        VectorStore -->|Formatted results| SearchTool
        SearchTool -->|Tool results| AI
        AI -->|Synthesize response| Claude
        Claude -->|Final answer| AI
        AI -->|Return response| RAG
        DirectResponse -->|Return answer| RAG
        
        RAG -->|Update session| Session[session_manager.py]
        RAG -->|Return answer, sources| API
        API -->|JSON response| Frontend
    end
    
    Backend -->|JSON response| JS
    JS -->|Display response| UI
    UI -->|Show answer & sources| User

    subgraph DataStore [Vector Database]
        ChromaDB -->|course_catalog| Metadata[Course Metadata]
        ChromaDB -->|course_content| Content[Chunked Content]
    end
    
    subgraph Tools [Tool Framework]
        ToolCall -->|tool_manager.execute_tool()| SearchTool
        SearchTool -->|last_sources| ToolManager[tool_manager.py]
    end

```

## Step-by-Step Sequence

### 1. Frontend Trigger
- User types query in chat input and presses Enter or clicks Send
- `sendMessage()` function in `frontend/script.js` is called
- Query and optional session ID are sent via POST to `/api/query`

### 2. FastAPI Endpoint (`app.py`)
- `query_documents()` endpoint receives JSON request with `query` and `session_id`
- Creates new session if `session_id` is not provided
- Calls `rag_system.query(query, session_id)`

### 3. RAG System Orchestration (`rag_system.py`)
- `query()` method builds prompt with conversation history from session manager
- Calls `ai_generator.generate_response()` with:
  - Query prompt
  - Conversation history (if any)
  - Tool definitions from `tool_manager`
  - Tool manager instance for execution

### 4. AI Generation with Tool Calling (`ai_generator.py`)
- System prompt instructs Claude to use search tool only for course-specific questions
- Claude decides whether to use `search_course_content` tool or answer directly
- **If tool is used**:
  - Claude returns tool use request with parameters (`query`, optional `course_name`, `lesson_number`)
  - `_handle_tool_execution()` calls `tool_manager.execute_tool()`
  - Tool results are sent back to Claude for synthesis
- **If no tool is used**:
  - Claude returns direct answer

### 5. Tool Execution (`search_tools.py`)
- `CourseSearchTool.execute()` calls `vector_store.search()`
- Tracks sources in `last_sources` for UI attribution

### 6. Vector Store Search (`vector_store.py`)
- **Course name resolution**: If `course_name` provided, searches `course_catalog` collection for best match
- **Filter building**: Creates ChromaDB filter based on course title and lesson number
- **Content search**: Queries `course_content` collection with embeddings
- Returns `SearchResults` object with documents, metadata, and distances

### 7. Response Synthesis
- Formatted search results are returned to Claude as tool results
- Claude synthesizes results into concise, educational answer
- Final response returned to `ai_generator`

### 8. Session Management (`session_manager.py`)
- Conversation history updated with new exchange (query + response)
- Limited to `MAX_HISTORY` turns for context window management

### 9. Return to Frontend
- RAG system returns `(answer, sources)` tuple
- FastAPI endpoint wraps in `QueryResponse` JSON
- Frontend receives response, removes loading indicator, displays:
  - Answer rendered as markdown
  - Collapsible sources section (if any)
- Session ID preserved for continued conversation

## Key Design Decisions

1. **Tool-based RAG**: LLM decides when to search vs. use general knowledge
2. **One search maximum**: Controlled by system prompt to limit API calls
3. **Dual collection strategy**: Separate metadata and content for efficient filtering
4. **Semantic course matching**: Fuzzy course name resolution using vector similarity
5. **Session-based conversations**: Limited history to maintain context window
6. **Source attribution**: Search tool tracks sources for UI display

## Data Flow Summary

```
User → Frontend (JS) → FastAPI (Python) → RAG System → AI Generator → Claude
                                                                        ↓
ChromaDB ← Vector Store ← Search Tool ← Tool Manager ← Tool Execution ← Claude
                                                                        ↓
User ← Frontend ← FastAPI ← RAG System ← AI Generator ← Response Synthesis
```

## Files Involved

| Step | Component | Primary Files |
|------|-----------|---------------|
| Frontend | User Interface | `frontend/index.html`, `frontend/script.js`, `frontend/style.css` |
| API Layer | HTTP Endpoints | `backend/app.py` |
| Orchestration | RAG System | `backend/rag_system.py` |
| AI Generation | Claude Integration | `backend/ai_generator.py` |
| Tool Framework | Search Tools | `backend/search_tools.py` |
| Vector Store | ChromaDB Wrapper | `backend/vector_store.py` |
| Data Models | Pydantic Models | `backend/models.py` |
| Configuration | Settings | `backend/config.py` |
| Session Management | Conversation History | `backend/session_manager.py` |
| Document Processing | Text Chunking | `backend/document_processor.py` |

## Error Handling

- Frontend: Loading states, error messages, retry logic
- Backend: HTTP exception handling, graceful degradation
- Vector Store: Empty result handling, course resolution fallbacks
- AI Generation: Tool execution errors, API failure recovery

This architecture enables efficient, context-aware querying of course materials with clear separation of concerns and scalable design.