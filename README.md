# Deep Research Backend System

A Django-based backend system that wraps the `open_deep_research` agent to provide persistent research history, continuation capabilities, file-based context injection, reasoning visibility, LangSmith tracing, and token/cost tracking.

## Architecture

### Overview

The system is built with a clean separation of concerns:

- **Django + DRF**: RESTful API layer
- **PostgreSQL**: Persistent data storage
- **Celery**: Asynchronous task execution
- **LangChain + LangGraph**: Integration with the deep research agent
- **LangSmith**: Tracing and monitoring
- **Adapter Pattern**: Wraps `open_deep_research` without modifying core logic

### Project Structure

```
creston/
├── creston/           # Django project settings
│   ├── settings.py    # Configuration
│   ├── urls.py        # URL routing
│   └── celery.py      # Celery configuration
├── research/          # Research app
│   ├── models.py      # Data models
│   ├── views.py       # API endpoints
│   ├── serializers.py # DRF serializers
│   ├── tasks.py       # Celery tasks
│   └── urls.py        # URL routing
├── core/              # Core utilities
│   ├── research_adapter.py  # Adapter for open_deep_research
│   └── document_processor.py # Document processing
└── manage.py          # Django management script
```

## Data Models

### ResearchSession
- Stores research queries, status, and results
- Links to parent sessions for continuation
- Tracks trace IDs for LangSmith

### ResearchSummary
- Stores high-level summaries of research findings
- Used for continuation context injection

### ResearchReasoning
- Stores high-level reasoning steps (query planning, source selection)
- **Does NOT store chain-of-thought** - only summarized reasoning

### UploadedDocument
- Manages PDF and TXT file uploads
- Stores extracted text and summaries for context injection

### ResearchCost
- Tracks token usage (input/output/total)
- Calculates estimated costs based on model pricing

## API Endpoints

### POST /api/research/start
Start a new research session.

**Request:**
```json
{
  "query": "What are the latest developments in quantum computing?",
  "user_id": 1  // Optional, defaults to authenticated user
}
```

**Response:**
```json
{
  "session_id": 123,
  "status": "pending",
  "message": "Research session started"
}
```

### POST /api/research/{id}/continue
Continue a research session with a new query, injecting previous research context.

**Request:**
```json
{
  "query": "What are the practical applications of these developments?",
  "user_id": 1  // Optional
}
```

**Response:**
```json
{
  "session_id": 124,
  "parent_id": 123,
  "status": "pending",
  "message": "Research continuation started"
}
```

### POST /api/research/{id}/upload
Upload a document (PDF or TXT) for context injection.

**Request:**
- Multipart form data with `file` field

**Response:**
```json
{
  "document_id": 45,
  "file_name": "research_paper.pdf",
  "file_type": "pdf",
  "message": "Document uploaded and processing started"
}
```

### GET /api/research/history
Get research history for a user.

**Query Parameters:**
- `user_id`: User ID (required if not authenticated)

**Response:**
```json
[
  {
    "id": 123,
    "user": {"id": 1, "username": "user1"},
    "query": "What are the latest developments...",
    "status": "completed",
    "summary": "Summary of findings...",
    "trace_id": "abc123...",
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

### GET /api/research/{id}
Get detailed information about a research session.

**Response:**
```json
{
  "id": 123,
  "user": {"id": 1, "username": "user1"},
  "query": "What are the latest developments...",
  "status": "completed",
  "trace_id": "abc123...",
  "final_report": "Full structured report...",
  "summary": "High-level summary...",
  "sources": ["source1", "source2"],
  "reasoning_steps": [
    {
      "step_type": "query_planning",
      "description": "Planned research approach...",
      "metadata": {}
    }
  ],
  "cost": {
    "model_name": "gpt-4-turbo-preview",
    "input_tokens": 5000,
    "output_tokens": 2000,
    "total_tokens": 7000,
    "estimated_cost_usd": 0.11
  },
  "uploaded_documents": [],
  "created_at": "2024-01-01T00:00:00Z"
}
```

## Async Execution

Research execution is **non-blocking** using Celery:

1. API endpoint creates a `ResearchSession` with status `pending`
2. Celery task `execute_research` is triggered asynchronously
3. Task updates session status to `running`
4. Research is executed using the adapter
5. Results are persisted and status updated to `completed` or `failed`

### Running Celery Worker

```bash
celery -A creston worker --loglevel=info
```

### Running Celery Beat (if needed)

```bash
celery -A creston beat --loglevel=info
```

## Research Continuation Strategy

When continuing a research session:

1. **Parent Linking**: New `ResearchSession` is created with `parent` FK pointing to the original
2. **Context Injection**: Parent research summary is injected into the enhanced query
3. **Explicit Instruction**: Agent is instructed to avoid repeating already-covered topics
4. **Lineage Preservation**: Parent-child relationship is maintained for tracking

The adapter builds an enhanced query that includes:
- Original query
- Previous research summary
- Instruction to avoid repetition
- Document summaries (if any)

## File Upload & Context Injection

### Supported Formats
- **PDF**: Extracted using PyPDF2
- **TXT**: Direct text extraction

### Processing Flow
1. File is uploaded and stored
2. `UploadedDocument` record is created
3. Async Celery task `process_document` extracts text
4. Summary is generated using LLM (concise, ~1000 chars)
5. Summary is stored and available for context injection

### Context Integration
Document summaries are automatically injected into research context when:
- A new research session is started
- A research session is continued
- Documents were uploaded to the session

## LangSmith Integration

### Configuration
Set environment variables:
```bash
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_langsmith_api_key
LANGCHAIN_PROJECT=deep-research
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
```

### Tracing
- Every research run is traced using `@traceable` decorator
- Trace ID is captured and stored in `ResearchSession.trace_id`
- Trace ID is returned in API responses for debugging

### Viewing Traces
1. Go to LangSmith dashboard
2. Filter by project: `deep-research`
3. Use trace_id from API response to find specific runs

## Cost & Token Tracking

### Token Tracking
- Implemented via `TokenTrackingCallback` (LangChain callback)
- Tracks input tokens, output tokens, and total tokens
- Captures model name used

### Cost Calculation
- Model pricing is configurable in `settings.py` (`MODEL_PRICING`)
- Costs calculated as:
  ```
  cost = (input_tokens / 1M) * input_price + (output_tokens / 1M) * output_price
  ```
- Stored in `ResearchCost` model per session

### Default Pricing (per 1M tokens)
- `gpt-4-turbo-preview`: $10 input / $30 output
- `gpt-4`: $30 input / $60 output
- `gpt-3.5-turbo`: $0.50 input / $1.50 output

## Installation & Setup

### Prerequisites
- Python 3.11+
- PostgreSQL
- Redis (for Celery broker)

### Installation

1. **Clone and install dependencies:**
```bash
pip install -r requirements.txt
```

2. **Install open_deep_research:**
```bash
pip install git+https://github.com/langchain-ai/open_deep_research.git
```

3. **Configure environment variables:**
Create a `.env` file:
```bash
SECRET_KEY=your-secret-key
DEBUG=True
DB_NAME=creston
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
OPENAI_API_KEY=your-openai-api-key
LANGCHAIN_API_KEY=your-langsmith-api-key
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=deep-research
DEFAULT_MODEL=gpt-4-turbo-preview
```

4. **Run migrations:**
```bash
python manage.py migrate
```

5. **Create superuser (optional):**
```bash
python manage.py createsuperuser
```

6. **Start development server:**
```bash
python manage.py runserver
```

7. **Start Celery worker (in separate terminal):**
```bash
celery -A creston worker --loglevel=info
```

## Integration with open_deep_research

The system uses an **adapter pattern** to wrap `open_deep_research`:

- **No core logic modification**: The adapter calls the original agent without changes
- **Context injection**: Enhances queries with parent research and document summaries
- **Token tracking**: Wraps calls with tracking callbacks
- **Tracing**: Uses LangSmith's `@traceable` decorator

### Adapter Location
`core/research_adapter.py` - Contains `DeepResearchAdapter` class

### Adjusting Integration
If the `open_deep_research` API differs from expectations:
1. Update import paths in `core/research_adapter.py`
2. Adjust result extraction in `run_research()` method
3. Update reasoning extraction in `_extract_reasoning()` method

## Trade-offs & Design Decisions

### 1. Async Execution
- **Trade-off**: Results not immediately available
- **Benefit**: Non-blocking API, better scalability
- **Mitigation**: Status polling or webhooks (future enhancement)

### 2. High-Level Reasoning Only
- **Trade-off**: Less detailed reasoning visibility
- **Benefit**: Cleaner, more actionable insights
- **Rationale**: Chain-of-thought can be overwhelming; high-level steps are more useful

### 3. Document Summarization
- **Trade-off**: Some information loss in summarization
- **Benefit**: Token efficiency, focused context
- **Mitigation**: Full text stored in `extracted_text` field for reference

### 4. Adapter Pattern
- **Trade-off**: Additional abstraction layer
- **Benefit**: No modification to core agent, easier updates
- **Rationale**: Maintains compatibility with upstream changes

### 5. Cost Estimation
- **Trade-off**: Estimated costs, not exact
- **Benefit**: Useful for budgeting and monitoring
- **Note**: Actual costs may vary slightly based on provider pricing

## Future Enhancements

- Webhook support for research completion notifications
- Real-time status updates via WebSockets
- Support for additional file formats (DOCX, etc.)
- Research session sharing and collaboration
- Advanced filtering and search for research history
- Export research reports in various formats

## License

[Specify your license]

## Contributing

[Contributing guidelines]

#   - D e e p - R e s e a r c h - S y s t e m - w i t h - H i s t o r y - C o n t i n u a t i o n - C o s t - T r a c k i n g  
 