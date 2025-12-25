# Deep Research Backend System

A Django-based backend system that wraps the `open_deep_research` agent to provide persistent research history, continuation capabilities, file-based context injection, reasoning visibility, LangSmith tracing, and token/cost tracking.

## Architecture & Design Decisions

### Overview

The system is built with a clean separation of concerns following Django best practices:

- **Django + DRF**: RESTful API layer providing standardized endpoints
- **PostgreSQL**: Persistent data storage for research sessions, costs, and documents
- **Celery**: Asynchronous task execution for non-blocking research operations
- **LangChain + LangGraph**: Integration with the deep research agent
- **LangSmith**: Tracing and monitoring for debugging and observability
- **Adapter Pattern**: Wraps `open_deep_research` without modifying core logic

### Design Principles

1. **Non-Invasive Integration**: The adapter pattern ensures we never modify the core `open_deep_research` agent, maintaining compatibility with upstream updates.

2. **Separation of Concerns**: 
   - Models handle data persistence
   - Views handle HTTP requests/responses
   - Tasks handle async execution
   - Adapter handles external integration
   - Processors handle document operations

3. **Async-First**: All long-running operations (research execution, document processing) are handled asynchronously via Celery to keep the API responsive.

4. **Observability**: Built-in tracing (LangSmith) and cost tracking for monitoring and debugging.

5. **Extensibility**: Clean interfaces allow easy addition of new features (file types, models, etc.) without major refactoring.

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

## Research Continuation Implementation

### How It Works

Research continuation allows building upon previous research sessions without repetition. Here's how it's implemented:

#### 1. **Parent-Child Relationship**
```python
# In ResearchSession model
parent = models.ForeignKey('self', on_delete=models.SET_NULL, 
                          null=True, blank=True, related_name='continuations')
```
- Each continuation creates a new `ResearchSession` with a foreign key to the parent
- This maintains a clear lineage: `parent → child → grandchild`
- The relationship is preserved even if parent is deleted (`SET_NULL`)

#### 2. **Context Injection Process**

When `POST /api/research/{id}/continue` is called:

1. **Extract Parent Summary**: 
   ```python
   parent_summary = parent_session.summary
   if not parent_summary and parent_session.research_summary:
       parent_summary = parent_session.research_summary.content
   ```

2. **Build Enhanced Query** (in `DeepResearchAdapter._build_context()`):
   ```python
   context_parts = [query]  # New query
   
   if parent_summary:
       context_parts.append(
           f"\n\nPrevious Research Summary:\n{parent_summary}\n\n"
           "IMPORTANT: Do not repeat information already covered in the previous research. "
           "Focus on new aspects, deeper analysis, or different angles of the topic."
       )
   ```

3. **Inject Document Summaries**: If documents were uploaded to the parent session, their summaries are also included.

#### 3. **Explicit Avoidance Instructions**

The adapter explicitly instructs the agent to:
- Not repeat already-covered topics
- Focus on new aspects
- Provide deeper analysis
- Explore different angles

This is done through prompt engineering in the enhanced query, ensuring the agent understands the continuation context.

#### 4. **Implementation Location**

- **API Endpoint**: `research/views.py` → `continue_research()` method
- **Context Building**: `core/research_adapter.py` → `_build_context()` method
- **Task Execution**: `research/tasks.py` → `execute_research()` task

#### 5. **Benefits**

- **No Redundancy**: Agent avoids repeating previous findings
- **Progressive Depth**: Each continuation builds on previous knowledge
- **Traceable Lineage**: Full history of research evolution
- **Context Preservation**: All relevant context automatically included

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

## LangSmith Tracing Implementation

### How Tracing Works

#### 1. **Decorator-Based Tracing**

Every research execution is automatically traced using LangSmith's `@traceable` decorator:

```python
# In core/research_adapter.py
@traceable(name="deep_research")
def run_research(self, query: str, parent_summary: str = None, ...):
    # Research execution code
```

**What this does:**
- Automatically creates a trace in LangSmith for each research run
- Captures all LLM calls, tool invocations, and intermediate steps
- Provides full visibility into the agent's decision-making process
- Enables debugging and performance analysis

#### 2. **Trace ID Capture**

After research completes, the trace ID is captured and stored:

```python
# In core/research_adapter.py
trace_id = os.getenv('LANGCHAIN_TRACE_ID', '')
if not trace_id:
    trace_id = str(uuid.uuid4())  # Fallback if not available

return {
    'trace_id': trace_id,
    # ... other results
}
```

**How it works:**
- LangSmith sets `LANGCHAIN_TRACE_ID` environment variable during tracing
- We capture this value and store it in the database
- If not available, we generate a UUID as fallback

#### 3. **Persistence**

Trace IDs are stored in the `ResearchSession` model:

```python
# In research/tasks.py
session.trace_id = result.get('trace_id', '')
session.save()
```

This allows:
- Linking database records to LangSmith traces
- Quick access to traces from API responses
- Historical trace lookup

#### 4. **Configuration**

Set these environment variables in your `.env` file:

```bash
LANGCHAIN_TRACING_V2=true                    # Enable tracing
LANGCHAIN_API_KEY=your_langsmith_api_key     # Your LangSmith API key
LANGCHAIN_PROJECT=deep-research              # Project name in LangSmith
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com  # LangSmith API endpoint
```

**Getting your API key:**
1. Go to https://smith.langchain.com/
2. Sign up or log in
3. Navigate to Settings → API Keys
4. Create a new API key
5. Copy and add to `.env`

#### 5. **Viewing Traces**

**Method 1: Via LangSmith Dashboard**
1. Go to https://smith.langchain.com/
2. Select project: `deep-research`
3. Browse traces or search by trace_id

**Method 2: Via API Response**
1. Get research details: `GET /api/research/{id}`
2. Extract `trace_id` from response
3. Search for it in LangSmith dashboard

**Method 3: Direct Link**
```
https://smith.langchain.com/o/{org_id}/projects/{project_id}/traces/{trace_id}
```

#### 6. **What's Traced**

The `@traceable` decorator automatically captures:
- **LLM Calls**: All prompts and responses
- **Tool Invocations**: Function calls made by the agent
- **Intermediate Steps**: Decision points in the agent's workflow
- **Timing**: Duration of each operation
- **Errors**: Any exceptions or failures
- **Metadata**: Custom tags and metadata

#### 7. **Benefits**

- **Debugging**: See exactly what the agent did and why
- **Performance**: Identify bottlenecks and slow operations
- **Cost Analysis**: Understand which operations consume most tokens
- **Quality Assurance**: Review agent behavior for improvements
- **Compliance**: Full audit trail of research operations

#### 8. **Design Decisions**

- **Automatic Tracing**: No manual instrumentation needed - decorator handles everything
- **Non-Intrusive**: Tracing doesn't affect research execution performance
- **Trace ID Storage**: Stored in database for easy correlation
- **Optional**: System works without LangSmith (trace_id will be empty)

## Cost & Token Tracking Implementation

### How Token Tracking Works

#### 1. **Callback Handler**

Token tracking is implemented via a custom LangChain callback handler:

```python
# In core/research_adapter.py
class TokenTrackingCallback(BaseCallbackHandler):
    def __init__(self):
        self.input_tokens = 0
        self.output_tokens = 0
        self.model_name = None
    
    def on_llm_end(self, response: LLMResult, **kwargs):
        if response.llm_output:
            token_usage = response.llm_output.get('token_usage', {})
            self.input_tokens += token_usage.get('prompt_tokens', 0)
            self.output_tokens += token_usage.get('completion_tokens', 0)
```

**How it works:**
- LangChain automatically calls `on_llm_end()` after each LLM invocation
- Token usage is extracted from the response metadata
- Tokens are accumulated across all LLM calls in a research session
- Model name is captured from the response

#### 2. **Integration with Research Adapter**

```python
# In DeepResearchAdapter.run_research()
self.token_callback = TokenTrackingCallback()
result = run_deep_research(
    query=enhanced_query,
    llm=self.llm,
    callbacks=[self.token_callback],  # Pass callback here
    **kwargs
)
```

The callback is passed to the research agent, which propagates it to all LLM calls.

#### 3. **Cost Calculation**

Costs are calculated in the Celery task after research completes:

```python
# In research/tasks.py
token_usage = result.get('token_usage', {})
model_name = token_usage.get('model_name', settings.DEFAULT_MODEL)
input_tokens = token_usage.get('input_tokens', 0)
output_tokens = token_usage.get('output_tokens', 0)

# Get pricing from settings
pricing = settings.MODEL_PRICING.get(model_name, {'input': 0, 'output': 0})

# Calculate cost (per 1M tokens)
cost = (
    (input_tokens / 1_000_000) * pricing['input'] +
    (output_tokens / 1_000_000) * pricing['output']
)
```

#### 4. **Persistence**

Costs are stored in the `ResearchCost` model:

```python
ResearchCost.objects.update_or_create(
    session=session,
    defaults={
        'model_name': model_name,
        'input_tokens': input_tokens,
        'output_tokens': output_tokens,
        'total_tokens': total_tokens,
        'estimated_cost_usd': cost,
    }
)
```

#### 5. **Configuration**

Model pricing is configurable in `creston/settings.py`:

```python
MODEL_PRICING = {
    'gpt-4-turbo-preview': {'input': 10.0, 'output': 30.0},
    'gpt-4': {'input': 30.0, 'output': 60.0},
    'gpt-3.5-turbo': {'input': 0.5, 'output': 1.5},
}
```

Prices are per 1 million tokens. Update these values to match current provider pricing.

#### 6. **Accessing Cost Data**

Costs are automatically included in research detail responses:

```json
{
  "cost": {
    "model_name": "gpt-4-turbo-preview",
    "input_tokens": 5000,
    "output_tokens": 2000,
    "total_tokens": 7000,
    "estimated_cost_usd": 0.11
  }
}
```

#### 7. **Design Decisions**

- **Estimated Costs**: We calculate estimated costs based on published pricing. Actual costs may vary slightly.
- **Per-Session Tracking**: Each research session has its own cost record for granular tracking.
- **Configurable Pricing**: Easy to update as provider pricing changes.
- **Automatic Calculation**: No manual intervention needed - costs are calculated and stored automatically.

## Setup & Run Instructions

### Prerequisites

Before starting, ensure you have:

- **Python 3.11+**: Check with `python --version`
- **PostgreSQL 12+**: Database server installed and running
- **Redis** (optional): For Celery broker. Can use PostgreSQL instead (see below)
- **OpenAI API Key**: Get from https://platform.openai.com/
- **LangSmith API Key** (optional): Get from https://smith.langchain.com/

### Step-by-Step Setup

#### 1. Install Python Dependencies

```bash
# Install all required packages
pip install -r requirements.txt

# Install open_deep_research agent
pip install git+https://github.com/langchain-ai/open_deep_research.git
```

**Note**: If `open_deep_research` installation fails, you may need to clone the repository manually and adjust import paths in `core/research_adapter.py`.

#### 2. Set Up PostgreSQL Database

**Option A: Using pgAdmin (GUI)**
1. Open pgAdmin
2. Connect to PostgreSQL server
3. Right-click "Databases" → "Create" → "Database"
4. Name: `creston`
5. Click "Save"

**Option B: Using Command Line**
```bash
# Connect to PostgreSQL
psql -U postgres

# Create database
CREATE DATABASE creston;

# Exit
\q
```

**Note**: If `psql` command not found, add PostgreSQL bin directory to PATH or use pgAdmin.

#### 3. Configure Environment Variables

Create a `.env` file in the project root:

```bash
# Django Settings
SECRET_KEY=your-secret-key-here  # Generate with: python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1

# Database Configuration
DB_NAME=creston
DB_USER=postgres
DB_PASSWORD=your_postgres_password
DB_HOST=localhost
DB_PORT=5432

# Celery Configuration
# Option 1: Using Redis (recommended for production)
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# Option 2: Using PostgreSQL (easier setup, no Redis needed)
CELERY_BROKER_URL=db+postgresql://postgres:your_password@localhost:5432/creston
CELERY_RESULT_BACKEND=db+postgresql://postgres:your_password@localhost:5432/creston
# Note: If password contains @, use %40 instead (e.g., Alpha22@ becomes Alpha22%40)

# OpenAI Configuration
OPENAI_API_KEY=sk-your-openai-api-key-here
DEFAULT_MODEL=gpt-4-turbo-preview

# LangSmith Configuration (optional but recommended)
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your-langsmith-api-key-here
LANGCHAIN_PROJECT=deep-research
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com

# CORS (if needed for frontend)
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

**Important**: Replace all placeholder values with your actual credentials!

#### 4. Run Database Migrations

```bash
# Create migration files (if needed)
python manage.py makemigrations

# Apply migrations to create tables
python manage.py migrate

# If using PostgreSQL for Celery, also run:
python manage.py migrate django_celery_results
```

#### 5. Create Superuser (Optional)

For Django admin access:

```bash
python manage.py createsuperuser
```

Follow prompts to create admin user.

#### 6. Verify Setup

Test that everything is configured correctly:

```bash
# Check Django configuration
python manage.py check

# Test database connection
python manage.py dbshell
# Type \q to exit
```

### Running the Application

You need **2 terminal windows** running simultaneously:

#### Terminal 1: Django Development Server

```bash
python manage.py runserver
```

You should see:
```
Starting development server at http://127.0.0.1:8000/
```

**Verify**: Open http://localhost:8000/api/research/history/ in browser (should return `[]`)

#### Terminal 2: Celery Worker

**If using Redis:**
```bash
celery -A creston worker --loglevel=info
```

**If using PostgreSQL (Windows):**
```bash
celery -A creston worker --loglevel=info --pool=solo
```

**If using PostgreSQL (Linux/Mac):**
```bash
celery -A creston worker --loglevel=info
```

You should see:
```
celery@hostname v5.3.4 (singularity)
...
[tasks]
  . research.tasks.execute_research
  . research.tasks.process_document
```

### Testing the API

#### Start a Research Session

```bash
curl -X POST http://localhost:8000/api/research/start/ \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"What are the latest developments in AI?\", \"user_id\": 1}"
```

**Response:**
```json
{
  "session_id": 1,
  "status": "pending",
  "message": "Research session started"
}
```

#### Check Research Status

```bash
curl http://localhost:8000/api/research/1/
```

#### Check Research History

```bash
curl http://localhost:8000/api/research/history/?user_id=1
```

### Troubleshooting

**Database Connection Errors:**
- Verify PostgreSQL is running: Check Services (Windows) or `systemctl status postgresql` (Linux)
- Check password in `.env` matches PostgreSQL password
- Ensure database `creston` exists

**Celery Not Processing Tasks:**
- Verify Celery worker is running (Terminal 2)
- Check broker URL in `.env` is correct
- If using Redis, ensure Redis is running: `redis-cli ping` (should return `PONG`)
- Check Celery logs for errors

**Import Errors:**
- Ensure `open_deep_research` is installed: `pip list | grep open-deep-research`
- May need to adjust import paths in `core/research_adapter.py` based on actual package structure

**Research Stays in "pending" Status:**
- Check Celery worker is running
- Check Celery logs for errors
- Verify `open_deep_research` is properly installed and importable

### Production Deployment

For production:
1. Set `DEBUG=False` in `.env`
2. Use a strong `SECRET_KEY`
3. Configure proper `ALLOWED_HOSTS`
4. Use a production WSGI server (Gunicorn/uWSGI)
5. Set up proper Celery workers (systemd/supervisor)
6. Use environment variables instead of `.env` file
7. Set up reverse proxy (Nginx)
8. Configure proper logging

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

<<<<<<< HEAD
<<<<<<< HEAD
=======
>>>>>>> 15191a1 (Update README.md)
#
#   D e e p - L e a r n i n g - w i t h - h i s t o r y - c o n t i n o u s - a n d - c o s t - t r a c k  
 