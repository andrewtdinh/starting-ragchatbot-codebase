# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Environment Setup
```bash
# Install dependencies (requires uv package manager)
uv sync

# Set up environment variables
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Development Server
- Application runs on `http://localhost:8000`
- API documentation available at `http://localhost:8000/docs`
- Uses FastAPI with auto-reload in development

## Architecture Overview

### Core RAG System Design
This is a tool-based RAG system where Claude AI autonomously decides whether to search course materials or answer from general knowledge. The system uses a sophisticated orchestration pattern:

1. **RAG System** (`rag_system.py`) - Main orchestrator that coordinates all components
2. **AI Generator** (`ai_generator.py`) - Manages Claude API interactions with tool calling
3. **Tool Manager** (`search_tools.py`) - Handles tool registration and execution
4. **Vector Store** (`vector_store.py`) - ChromaDB interface for semantic search
5. **Document Processor** (`document_processor.py`) - Parses and chunks course documents
6. **Session Manager** (`session_manager.py`) - Maintains conversation context

### Key Architectural Patterns

#### Tool-Based AI Decision Making
Claude receives tools but decides autonomously when to use them. The system supports this through:
- `CourseSearchTool` with semantic course name matching
- Tool definitions provided to Claude API with `tool_choice: "auto"`
- Two-phase execution: initial response → tool execution → final synthesis

#### Document Processing Pipeline
Documents follow a structured format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [optional]
[lesson content]
```

The processor:
- Extracts metadata from headers
- Segments by lesson markers using regex
- Creates overlapping chunks (800 chars, 100 overlap)
- Adds contextual prefixes for better search

#### Vector Storage Strategy
Uses ChromaDB with two collections:
- `course_metadata` - Course-level information for broad queries
- `course_content` - Text chunks with lesson context for specific searches

#### Session Management
Maintains conversation context with configurable history length (default: 2 exchanges). Session IDs created on first query, conversation history passed to Claude for context.

### Configuration
All settings centralized in `backend/config.py`:
- Anthropic model: `claude-sonnet-4-20250514`
- Embedding model: `all-MiniLM-L6-v2`
- Chunk settings: 800 chars with 100 overlap
- ChromaDB path: `./chroma_db`

### Frontend Integration
Vanilla HTML/CSS/JavaScript frontend communicates via REST API:
- POST `/api/query` - Main query endpoint
- GET `/api/courses` - Course statistics
- WebSocket-style loading states with markdown rendering

### Development Notes

#### Document Format Requirements
Course documents must follow the structured format above. The system automatically:
- Skips duplicate courses (checks by title)
- Handles documents with or without lesson structure
- Maintains full traceability from chunks back to source course/lesson

#### Tool System Extension
To add new tools:
1. Implement `Tool` abstract base class
2. Register with `ToolManager`
3. Tool definitions automatically provided to Claude

#### ChromaDB Persistence
Database persists in `./chroma_db` directory. On startup, the system loads documents from `../docs` if available, avoiding duplicates.

The system is designed for educational content RAG with intelligent tool usage, context preservation, and source attribution.