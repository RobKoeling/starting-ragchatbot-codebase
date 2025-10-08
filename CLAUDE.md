# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for course materials using ChromaDB vector storage, Anthropic's Claude API, and semantic search. The system allows users to query course content through a web interface and receive AI-powered responses based on retrieved context.

## Development Commands

### Running the Application

```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

Application runs at `http://localhost:8000` with API docs at `http://localhost:8000/docs`

### Setup

```bash
# Install Python dependencies
uv sync

# Set up environment variables
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

## Architecture

### Core RAG Flow

1. **Document Processing** → 2. **Vector Storage** → 3. **Query Processing** → 4. **AI Generation**

The system uses a tool-based approach where Claude can call search tools to retrieve relevant course content before generating responses.

### Key Components

**RAGSystem** (`backend/rag_system.py`): Main orchestrator that coordinates:
- `DocumentProcessor`: Chunks course documents with overlap
- `VectorStore`: Manages ChromaDB collections (course_catalog + course_content)
- `AIGenerator`: Handles Claude API calls with tool support
- `ToolManager`: Manages search tools available to Claude
- `SessionManager`: Tracks conversation history

**Vector Store Architecture** (`backend/vector_store.py`):
- Two ChromaDB collections:
  - `course_catalog`: Course metadata (title, instructor, lessons) for course name resolution
  - `course_content`: Text chunks with course/lesson metadata for semantic search
- Smart course name matching: Uses semantic search to resolve partial course names
- Hierarchical filtering: Can filter by course title and/or lesson number

**Tool-Based Search** (`backend/search_tools.py`):
- Claude uses `search_course_content` tool to query the vector store
- Tool supports optional filtering by course_name and lesson_number
- Tool execution happens within Claude's agentic loop
- Sources are tracked and returned to the UI separately

**AI Generator** (`backend/ai_generator.py`):
- System prompt instructs Claude on when to use search (course-specific) vs general knowledge
- Enforces "one search per query maximum" to control costs
- Handles tool execution loop: initial request → tool calls → results → final response
- Temperature=0 for consistent responses, max_tokens=800

### Document Format

Course documents in `docs/` follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: [title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [title]
...
```

Documents are chunked with sentence-based splitting (800 chars, 100 char overlap). First chunk of each lesson gets context prefix.

### Configuration

Settings in `backend/config.py`:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (sentence-transformers)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation turns

### API Endpoints

- `POST /api/query`: Process user query, returns {answer, sources, session_id}
- `GET /api/courses`: Get course statistics
- Frontend served from `/` via static files

### Session Management

Each user gets a session_id to track conversation history. History is limited to MAX_HISTORY exchanges and used to provide context to Claude.

## Important Notes

- Course documents are loaded on startup from `../docs` relative to backend directory
- ChromaDB data persists in `./chroma_db` directory
- Duplicate courses are prevented by checking existing course titles before adding
- The system uses Anthropic's tool calling API, not traditional prompt engineering with context injection
- Windows users should use Git Bash to run commands
