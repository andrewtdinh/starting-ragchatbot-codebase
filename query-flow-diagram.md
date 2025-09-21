# Query Flow Diagram

## Complete Request-Response Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   FRONTEND                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  User Input                                                                         │
│  ┌─────────────┐    Enter/Click    ┌──────────────────┐                             │
│  │ Chat Input  │ ─────────────────▶│ sendMessage()    │                             │
│  │ Field       │                   │ function         │                             │
│  └─────────────┘                   └──────────────────┘                             │
│                                              │                                      │
│                                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ POST /api/query                                                             │    │
│  │ {                                                                           │    │
│  │   "query": "What is RAG?",                                                  │    │
│  │   "session_id": "abc123"                                                    │    │
│  │ }                                                                           │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                      │
└──────────────────────────────────────────────┼──────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  BACKEND API                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  FastAPI Endpoint (app.py:56)                                                       │
│  ┌─────────────────────────────────┐                                                │
│  │ @app.post("/api/query")         │                                                │
│  │ async def query_documents()     │                                                │
│  └─────────────────────────────────┘                                                │
│                    │                                                                │
│                    ▼                                                                │
│  ┌─────────────────────────────────┐                                                │
│  │ rag_system.query(               │                                                │
│  │   request.query,                │                                                │
│  │   session_id                    │                                                │
│  │ )                               │                                                │
│  └─────────────────────────────────┘                                                │
│                    │                                                                │
└────────────────────┼────────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                RAG SYSTEM                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Main Orchestrator (rag_system.py:102)                                             │
│                                                                                    │
│  ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐            │
│  │ Session Manager │    │ Tool Manager     │    │ AI Generator        │            │
│  │ - Get history   │    │ - Get tool defs  │    │ - Build prompt      │            │
│  │ - Store convo   │    │ - Reset sources  │    │ - Call Claude API   │            │
│  └─────────────────┘    └──────────────────┘    └─────────────────────┘            │
│                                  │                          │                      │
│                                  ▼                          ▼                      │
│                          ┌─────────────────────────────────────────────────┐       │
│                          │ ai_generator.generate_response()                │       │
│                          │ - System prompt + conversation history          │       │
│                          │ - Tools available for Claude                    │       │
│                          └─────────────────────────────────────────────────┘       │
│                                                  │                                  │
└──────────────────────────────────────────────────┼──────────────────────────────────┘
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             CLAUDE AI DECISION                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │ Claude analyzes query and decides:                                          │   │
│   │                                                                             │   │
│   │ ┌─────────────────────┐          ┌─────────────────────────────────────┐    │   │
│   │ │ General Knowledge   │          │ Course-Specific Query               │    │   │
│   │ │ Answer directly     │    OR    │ Use search_course_content tool      │    │   │
│   │ │ without searching   │          │ to find relevant information        │    │   │
│   │ └─────────────────────┘          └─────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                  │                                  │
│                                                  ▼                                  │
│                                        [IF TOOL USE NEEDED]                         │
└──────────────────────────────────────────────────┼──────────────────────────────────┘
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TOOL EXECUTION                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Course Search Tool (search_tools.py:52)                                            │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ CourseSearchTool.execute()                                                  │    │
│  │ - query: "What is RAG?"                                                     │    │
│  │ - course_name: null (optional)                                              │    │
│  │ - lesson_number: null (optional)                                            │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                      │                                              │
│                                      ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ vector_store.search()                                                       │    │
│  │ - Embed query using sentence-transformers                                   │    │
│  │ - Search ChromaDB course content collection                                 │    │
│  │ - Apply optional course/lesson filters                                      │    │
│  │ - Return top K similar chunks with metadata                                 │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                      │                                              │
│                                      ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ Format Results                                                              │    │
│  │ ┌─────────────────────────────────────────────────────────────────────────┐ │    │
│  │ │ [Course: "Building RAG Systems" - Lesson 1]                             │ │    │
│  │ │ RAG stands for Retrieval-Augmented Generation...                        │ │    │
│  │ │                                                                         │ │    │
│  │ │ [Course: "AI Fundamentals" - Lesson 3]                                  │ │    │
│  │ │ RAG combines information retrieval with language generation...          │ │    │
│  │ └─────────────────────────────────────────────────────────────────────────┘ │    │
│  │ - Store sources: ["Building RAG Systems - Lesson 1", ...]                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                      │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            CLAUDE FINAL RESPONSE                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Second API Call to Claude (ai_generator.py:89)                                     │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ Messages:                                                                   │    │
│  │ 1. User: "What is RAG?"                                                     │    │
│  │ 2. Assistant: [tool_use] search_course_content(query="What is RAG?")        │    │
│  │ 3. User: [tool_result] [Formatted search results from vector store]         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                      │                                              │
│                                      ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ Claude synthesizes search results into coherent answer:                     │    │
│  │                                                                             │    │
│  │ "RAG (Retrieval-Augmented Generation) is a technique that combines          │    │
│  │ information retrieval with language generation. It works by first           │    │
│  │ searching relevant documents, then using that information to generate       │    │
│  │ more accurate and contextual responses..."                                  │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                      │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            RESPONSE ASSEMBLY                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  RAG System (rag_system.py:129-140)                                                │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Get sources from tool_manager.get_last_sources()                        │   │
│  │    → ["Building RAG Systems - Lesson 1", "AI Fundamentals - Lesson 3"]     │   │
│  │                                                                             │   │
│  │ 2. Update conversation history with session_manager                        │   │
│  │    → Store user query + AI response for future context                     │   │
│  │                                                                             │   │
│  │ 3. Reset sources with tool_manager.reset_sources()                         │   │
│  │                                                                             │   │
│  │ 4. Return (response, sources) tuple                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              API RESPONSE                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  FastAPI Response (app.py:67-72)                                                   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ QueryResponse(                                                              │   │
│  │   answer="RAG (Retrieval-Augmented Generation) is a technique...",         │   │
│  │   sources=["Building RAG Systems - Lesson 1", "AI Fundamentals - Lesson 3"], │   │
│  │   session_id="abc123"                                                       │   │
│  │ )                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             FRONTEND RENDERING                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Response Handling (script.js:76-85)                                               │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Replace loading animation with response                                  │   │
│  │ 2. Render markdown content using marked.js                                 │   │
│  │ 3. Add collapsible sources section                                         │   │
│  │ 4. Update session ID for future requests                                   │   │
│  │ 5. Re-enable input field                                                   │   │
│  │ 6. Scroll chat to bottom                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ Chat UI                                                                     │   │
│  │ ┌─────────────────────────────────────────────────────────────────────────┐ │   │
│  │ │ User: What is RAG?                                                      │ │   │
│  │ │                                                                         │ │   │
│  │ │ Assistant: RAG (Retrieval-Augmented Generation) is a technique...      │ │   │
│  │ │ ┌─ Sources ──────────────────────────────────────────────────────────┐ │ │   │
│  │ │ │ Building RAG Systems - Lesson 1, AI Fundamentals - Lesson 3       │ │ │   │
│  │ │ └────────────────────────────────────────────────────────────────────┘ │ │   │
│  │ └─────────────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Data Structures

### ChromaDB Collections
```
course_metadata/
├── courses collection
│   ├── course titles
│   ├── instructors
│   └── course links
│
course_content/
├── text chunks with embeddings
├── metadata: course_title, lesson_number, chunk_index
└── semantic search indices
```

### Session Management
```
Session {
  id: "abc123",
  history: [
    {user: "What is RAG?", assistant: "RAG is...", timestamp: "..."},
    {user: "How does it work?", assistant: "It works by...", timestamp: "..."}
  ],
  created_at: "2024-01-01T00:00:00Z"
}
```

## Key Decision Points

1. **Tool Usage**: Claude autonomously decides whether to search based on query analysis
2. **Search Scope**: Tool can filter by course name and/or lesson number
3. **Context Preservation**: Full conversation history maintained per session
4. **Error Handling**: Each layer has fallback mechanisms
5. **Performance**: Async operations with loading states for better UX