# Query Flow Analysis: Frontend to Backend

This document traces the complete process of handling a user's query from frontend to backend in the Course Materials RAG System.

## Complete Query Flow: Frontend → Backend

### **1. Frontend Query Initiation** (`script.js:45-96`)

**User Action:**
- User types query and hits Enter or clicks send button
- `sendMessage()` function triggered

**Frontend Processing:**
```javascript
// Disable UI and show loading
chatInput.disabled = true;
addMessage(query, 'user');
const loadingMessage = createLoadingMessage();

// POST to /api/query
fetch('/api/query', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        query: query,
        session_id: currentSessionId
    })
})
```

### **2. API Request Handling** (`app.py:56-74`)

**FastAPI Endpoint:**
```python
@app.post("/api/query", response_model=QueryResponse)
async def query_documents(request: QueryRequest):
    # Create session if not provided
    session_id = request.session_id or rag_system.session_manager.create_session()

    # Process query using RAG system
    answer, sources = rag_system.query(request.query, session_id)

    return QueryResponse(answer=answer, sources=sources, session_id=session_id)
```

### **3. RAG System Processing** (`rag_system.py:102-140`)

**Main Orchestration:**
```python
def query(self, query: str, session_id: Optional[str] = None):
    # Prepare prompt for AI
    prompt = f"Answer this question about course materials: {query}"

    # Get conversation history
    history = self.session_manager.get_conversation_history(session_id)

    # Generate response using AI with tools
    response = self.ai_generator.generate_response(
        query=prompt,
        conversation_history=history,
        tools=self.tool_manager.get_tool_definitions(),
        tool_manager=self.tool_manager
    )

    # Get sources and update history
    sources = self.tool_manager.get_last_sources()
    self.session_manager.add_exchange(session_id, query, response)

    return response, sources
```

### **4. AI Generation & Tool Usage** (`ai_generator.py:43-135`)

**Claude API Call:**
```python
def generate_response(self, query, conversation_history, tools, tool_manager):
    # Build system prompt with history
    system_content = f"{SYSTEM_PROMPT}\n\nPrevious conversation:\n{conversation_history}"

    # API call with tools
    response = self.client.messages.create(
        model=self.model,
        messages=[{"role": "user", "content": query}],
        system=system_content,
        tools=tools,
        tool_choice={"type": "auto"}
    )

    # Handle tool execution if Claude wants to search
    if response.stop_reason == "tool_use":
        return self._handle_tool_execution(response, api_params, tool_manager)

    return response.content[0].text
```

**Tool Execution** (if Claude decides to search):
```python
def _handle_tool_execution(self, initial_response, base_params, tool_manager):
    # Execute search tool
    tool_result = tool_manager.execute_tool(tool_name, **tool_input)

    # Send results back to Claude for final response
    final_response = self.client.messages.create(
        messages=[...previous_messages, tool_results],
        system=base_params["system"]
    )
    return final_response.content[0].text
```

### **5. Vector Search (if triggered)** (`search_tools.py:52-86`)

**Course Search Tool:**
```python
def execute(self, query, course_name=None, lesson_number=None):
    # Use vector store for semantic search
    results = self.store.search(
        query=query,
        course_name=course_name,
        lesson_number=lesson_number
    )

    # Format results with course/lesson context
    return self._format_results(results)
```

**ChromaDB Query** (`vector_store.py`):
- Embeds query using sentence-transformers
- Searches course content collection with optional filters
- Returns semantically similar chunks with metadata

### **6. Response Path Back to Frontend**

**Backend Response:**
```python
return QueryResponse(
    answer=final_ai_response,
    sources=["Course 1 - Lesson 2", "Course 3 - Lesson 1"],
    session_id=session_id
)
```

**Frontend Handling:**
```javascript
const data = await response.json();

// Update session ID if new
currentSessionId = data.session_id;

// Replace loading with response
loadingMessage.remove();
addMessage(data.answer, 'assistant', data.sources);
```

**UI Updates:**
- Loading animation replaced with AI response
- Markdown rendering for formatted text
- Collapsible sources section
- Chat history maintained
- Input re-enabled for next query

### **Key Flow Characteristics:**

- **Tool-driven**: Claude decides whether to search based on query content
- **Contextual**: Conversation history maintained per session
- **Source attribution**: UI shows which courses/lessons informed the response
- **Async/streaming**: Frontend shows loading states during processing
- **Error handling**: Graceful degradation at each layer

The entire flow typically takes 2-3 seconds depending on whether Claude needs to search the vector store.

## Architecture Summary

This RAG system uses a sophisticated tool-based approach where:

1. **Frontend** handles user interaction and API communication
2. **FastAPI** provides REST endpoints and request validation
3. **RAG System** orchestrates all components
4. **AI Generator** interfaces with Claude API and manages tool execution
5. **Search Tools** provide semantic search capabilities
6. **Vector Store** handles ChromaDB operations and embeddings
7. **Session Manager** maintains conversation context

The design allows Claude to intelligently decide when to search course materials versus answering from general knowledge, creating a more natural and efficient user experience.