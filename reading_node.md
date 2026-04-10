# Course Materials RAG Chatbot - Project Overview

This is a **Retrieval-Augmented Generation (RAG) chatbot system** for querying course materials. It's a full‑stack web application that lets users ask questions about courses and get AI‑powered, context‑aware answers.

## 🏗️ **Architecture Overview**

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
│   Frontend  │    │   FastAPI    │    │   RAG        │    │   ChromaDB  │
│  (HTML/JS)  │◄──►│   Backend    │◄──►│   System     │◄──►│   Vector    │
│             │    │   (Python)   │    │   (Orchestrator) │   Store      │
└─────────────┘    └──────────────┘    └──────────────┘    └─────────────┘
                          │                     │
                          │                     │
                    ┌─────▼─────┐         ┌─────▼─────┐
                    │ Anthropic │         │ Document  │
                    │  Claude   │         │ Processor │
                    │   API     │         └───────────┘
                    └───────────┘
```

## 📁 **Project Structure**

```
starting‑ragchatbot‑codebase/
├── backend/                    # Python FastAPI backend
│   ├── app.py                 # Main FastAPI application
│   ├── rag_system.py          # RAG orchestrator
│   ├── vector_store.py        # ChromaDB vector store operations
│   ├── document_processor.py  # Processes course documents into chunks
│   ├── ai_generator.py        # Anthropic Claude API integration
│   ├── search_tools.py        # Tool‑based semantic search
│   ├── session_manager.py     # Conversation session management
│   ├── models.py              # Pydantic models (Course, Lesson, Chunk)
│   └── config.py              # Configuration settings
├── frontend/                  # Static frontend
│   ├── index.html            # Chat interface
│   ├── style.css             # Styling
│   └── script.js             # Client‑side logic
├── docs/                      # Sample course documents
│   ├── course1_script.txt
│   ├── course2_script.txt
│   └── ...
├── pyproject.toml            # Python dependencies (uv)
├── run.sh                    # Startup script
├── README.md                 # Setup instructions
└── .env.example              → Copy to .env with ANTHROPIC_API_KEY
```

## 🔧 **Core Components**

1. **Document Processor** (`document_processor.py`)
   - Parses course documents with format:
     ```
     Course Title: [title]
     Course Link: [url]
     Course Instructor: [instructor]
     Lesson 0: Introduction
     Lesson Link: [url]
     [content...]
     ```
   - Splits content into sentence‑based chunks (configurable size/overlap)

2. **Vector Store** (`vector_store.py`)
   - Uses ChromaDB with two collections:
     - `course_catalog`: Course metadata (title, instructor, lessons)
     - `course_content`: Chunked lesson content with embeddings
   - Embeddings via `sentence‑transformers/all‑MiniLM‑L6‑v2`

3. **AI Generator** (`ai_generator.py`)
   - Integrates Anthropic Claude (default: `claude‑sonnet‑4‑20250514`)
   - Uses tool‑calling to invoke the search tool
   - System prompt enforces concise, educational responses

4. **Search Tool** (`search_tools.py`)
   - Single tool `search_course_content` with parameters:
     - `query`: What to search for
     - `course_name`: Optional course filter (supports partial matches)
     - `lesson_number`: Optional lesson filter
   - Returns formatted results with course/lesson context

5. **RAG System** (`rag_system.py`)
   - Orchestrates the flow: query → search → AI generation → response
   - Manages conversation history (configurable max messages)

6. **Web Interface** (`frontend/`)
   - Clean chat UI with suggested questions
   - Displays course statistics and sources
   - Real‑time markdown rendering

## 🚀 **How It Works**

1. **Ingestion** (on startup or via API):
   - Documents in `docs/` are parsed into `Course`/`Lesson`/`CourseChunk` objects
   - Metadata goes to `course_catalog` collection
   - Chunked content goes to `course_content` collection

2. **Query Flow**:
   - User asks a question → frontend POSTs to `/api/query`
   - RAG system creates prompt with conversation history (if session exists)
   - AI decides whether to use search tool (one search max per query)
   - Tool searches vector store with semantic matching
   - AI synthesizes results into answer
   - Response + sources returned to frontend

3. **Course Resolution**:
   - If a course name is provided (e.g., "MCP"), vector search finds the closest matching course title
   - Results can be filtered by course and/or lesson number

## ⚙️ **Configuration** (`config.py`)

```python
CHUNK_SIZE = 800          # Characters per chunk
CHUNK_OVERLAP = 100       # Overlap between chunks
MAX_RESULTS = 5           # Max search results
MAX_HISTORY = 2           # Conversation turns to remember
EMBEDDING_MODEL = "all‑MiniLM‑L6‑v2"
ANTHROPIC_MODEL = "claude‑sonnet‑4‑20250514"
CHROMA_PATH = "./chroma_db"
```

## 🛠️ **Setup & Running**

1. **Prerequisites**:
   - Python 3.13+, `uv` package manager, Anthropic API key

2. **Installation**:
   ```bash
   uv sync                    # Install dependencies
   cp .env.example .env       # Add your ANTHROPIC_API_KEY
   ```

3. **Run**:
   ```bash
   ./run.sh                   # Or: cd backend && uv run uvicorn app:app --reload --port 8000
   ```
   - Web UI: `http://localhost:8000`
   - API docs: `http://localhost:8000/docs`

4. **Add Course Documents**:
   - Place `.txt` files in `docs/` folder with the expected format
   - On startup, the system automatically ingests them

## 🎯 **Use Cases**

- Ask about course outlines, instructors, lesson content
- Search for specific topics across multiple courses
- Get explanations of technical concepts (RAG, MCP, etc.)
- Find courses that cover certain topics

## 🔍 **Key Design Decisions**

- **Single search tool**: Limits API calls and encourages synthesis
- **Two‑collection vector store**: Separates metadata from content for better filtering
- **Sentence‑based chunking**: Preserves semantic boundaries better than fixed‑size splitting
- **Tool‑based RAG**: Lets the AI decide when to search vs. use general knowledge
- **Static frontend**: Simple, deployable anywhere the backend runs

This is a well‑structured starting point for a course‑focused RAG system that balances simplicity with extensibility. The modular design makes it easy to swap components (e.g., different vector stores, LLM providers) or add new tools.