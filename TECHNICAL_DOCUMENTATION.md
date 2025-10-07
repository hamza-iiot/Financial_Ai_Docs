# Technical Documentation - Yomnai Financial Analyst

**Last Updated:** October 2, 2025

## System Architecture

### Overview
Yomnai is a sophisticated multi-agent financial analysis system built on a modern AI stack with **FastAPI backend** and **React 19 frontend**. All processing occurs locally on the user's machine, ensuring complete data privacy while delivering professional-grade financial insights.

### Architecture Overview (October 2025)
- **Frontend**: React 19 + TypeScript (Port 5173)
- **Backend**: FastAPI + Python 3.10 (Port 8000)
- **Database**: SQLite (sessions/uploads) + ChromaDB (vector storage)
- **AI Runtime**: Ollama (local LLM execution)
- **Integration**: REST API + WebSocket for real-time chat

### High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│              React 19 Frontend (Port 5173)           │
│  • Transaction Dashboard (6 tabs + sidebar nav)      │
│  • Financial Dashboard (5 tabs)                      │
│  • Real-time chat interface (WebSocket)              │
│  • File upload with progress tracking                │
└──────────┬───────────────────────────────────────────┘
           │ REST API + WebSocket
           ▼
┌─────────────────────────────────────────────────────┐
│           FastAPI Backend (Port 8000)                │
│  • Authentication (email whitelist, 24h sessions)    │
│  • File upload endpoints                             │
│  • Transaction retrieval (upload_id filtering)       │
│  • AI agent orchestration                            │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│         Database Layer (SQLite + ChromaDB)           │
│  • SQLite: Sessions, uploads, users, access_logs     │
│  • ChromaDB: Vector storage with upload_id metadata  │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│              Document Processing Layer               │
├─────────────────────────────────────────────────────┤
│ • DocumentTypeDetector - Auto-detects file type     │
│ • CSV/Excel Parser - Transaction extraction         │
│ • Financial Statement Parser - Balance sheet/P&L    │
│ • PDF Vision AI - Qwen2.5-VL for PDF processing     │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│           Multi-Agent AI System (12 Agents)          │
├─────────────────────────────────────────────────────┤
│ • QueryUnderstander (Gemma3:4b) - Intent extraction │
│ • AgentRouter - Query classification                │
│ • 6 Transaction Agents (Qwen3-14b) - Bank data      │
│ • 6 Financial Agents (Qwen3-14b) - Statements       │
│ • Dual Orchestrator - Coordination by data type     │
└─────────────────────────────────────────────────────┘
```

## Technology Stack

### Core Technologies

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Frontend** | React 19 + TypeScript | 19.1.1 / 5.8.3 | Web interface & dashboards |
| **Backend** | FastAPI | 0.104+ | REST API & WebSocket server |
| **Build Tool** | Vite | 7.1.7 | Frontend build & dev server |
| **State Management** | Zustand | 5.0.8 | Frontend state (sessions, uploads, transactions) |
| **Charts** | Recharts | 3.2.1 | Data visualizations |
| **Database** | SQLite + SQLAlchemy | 3.x / 2.0+ | Sessions, users, uploads |
| **Data Processing** | Pandas | 2.2.0 | CSV/Excel parsing |
| **Vector Database** | ChromaDB | 0.4.22 | Transaction storage with upload_id |
| **Embeddings** | SentenceTransformers | all-MiniLM-L6-v2 | Text vectorization |
| **LLM Runtime** | Ollama | Latest | Local model execution |
| **RAG Framework** | LangChain | 0.1.9 | Query orchestration |
| **PDF Processing** | pdf2image, Pillow | 1.17.0, 10.2.0 | PDF to image conversion |
| **Language** | Python 3.10+ / TypeScript | 3.10+ / 5.8.3 | Backend / Frontend |

### AI Models

| Model | Purpose | Size | Temperature | Context |
|-------|---------|------|-------------|---------|
| **qwen3-14b-32k:latest** | Primary analysis | 14B | 0.1-0.3 | 32k tokens |
| **gemma3:4b** | Query understanding | 4B | 0.1 | Fast intent extraction |
| **phi3:latest** | Alternative router | 3.8B | 0.1 | Query classification |
| **qwen2.5:7b-vl** | Vision AI for PDFs | 7B | 0.2 | Image processing |

## Data Flow Architecture

### 1. File Upload & Processing

```python
# Step 1: Document Type Detection
detector = DocumentTypeDetector()
doc_type, reason = detector.detect_document_type(file_path)
# Returns: 'transactions' or 'financial_statement'

# Step 2: Intelligent Parsing
if doc_type == 'transactions':
    parser = BankStatementParser()  # or ExcelStatementParser()
    transactions = parser.parse_file(file_path)
else:
    parser = FinancialStatementParser()
    statement = parser.parse_file(file_path)

# Step 3: Validation
validator = TransactionValidator()
validation_result = validator.validate_transactions(transactions)
```

### 2. Vector Storage

```python
# Step 4: Vectorization & Storage
vector_store = VectorStoreManager()
vector_store.add_transactions(transactions, session_id)

# Creates rich text representation
document_text = transaction.to_text()
# "On June 15, spent SAR 2,500 on vendor payment..."

# Generates embeddings
embeddings = SentenceTransformers('all-MiniLM-L6-v2')

# Stores with metadata
metadata = {
    'date_timestamp': float,
    'amount': float,
    'type': 'debit/credit',
    'category': string,
    'session_id': string
}
```

### 3. Query Processing Pipeline

```python
# Step 5: Query Understanding
query_understander = QueryUnderstander(vector_store, ollama_client)
understanding = query_understander.analyze_and_retrieve(query)
# Returns: {intent, transactions, metadata}

# Step 6: Agent Routing
orchestrator = AgentOrchestrator(vector_store, ollama_client)
result = orchestrator.process_chat_query(query)

# Internal flow:
# 1. Extract intent using Gemma3:4b
# 2. Retrieve relevant transactions
# 3. Route to specialized agent
# 4. Generate hidden thinking
# 5. Analyze with domain expertise
# 6. Synthesize response
```

## Multi-Agent System

### Agent Architecture

```python
class AgentOrchestrator:
    def __init__(self):
        self.agents = {
            'expense': ExpenseAgent(),
            'income': IncomeAgent(),
            'fee': FeeHunterAgent(),
            'budget': BudgetAdvisorAgent(),
            'trend': TrendAnalystAgent(),
            'transaction': TransactionInvestigatorAgent()
        }
```

### Agent Capabilities

| Agent | Specialization | Key Features | Tools |
|-------|---------------|--------------|-------|
| **ExpenseAgent** | Business expenses | Categorization, patterns, savings | 6 tools |
| **IncomeAgent** | Revenue analysis | Source tracking, stability | 5 tools |
| **FeeHunterAgent** | Fee detection | Hidden costs, savings potential | 5 tools |
| **BudgetAdvisorAgent** | Cash flow | Health scoring, recommendations | 3 tools |
| **TrendAnalystAgent** | Patterns | Time-series, predictions | 3 tools |
| **TransactionInvestigatorAgent** | Search | Complex queries, ranking | 3 tools |

### Hidden Thinking Process

Each agent performs multi-step reasoning that's hidden from the user:

```python
def execute_with_data(self, query, thinking_context, transactions):
    # Step 1: Hidden reasoning
    internal_analysis = self._think_step_by_step(query, thinking_context)

    # Step 2: Analyze pre-retrieved data
    analysis = self._deep_analyze(transactions, internal_analysis)

    # Step 3: Generate user response
    final_answer = self._generate_user_response(analysis, query)

    return {
        'final_answer': final_answer,
        'thinking': internal_analysis  # Hidden from UI
    }
```

## ChromaDB Vector Database

### Storage Schema

```python
# Document creation with upload_id metadata (October 2025 Update)
def _create_document_text(transaction):
    parts = [
        transaction.to_text(),
        f"Transaction details: {transaction.description}",
        f"Amount: SAR {transaction.amount:.2f}",
        f"Type: {transaction.type.value}",
        f"Date: {transaction.date.strftime('%Y-%m-%d')}",
        f"Month: {transaction.date.strftime('%B %Y')}",
        f"Day of week: {transaction.date.strftime('%A')}"
    ]

    # Add semantic tags
    tags = _generate_semantic_tags(transaction)
    # Tags: ['expense', 'large_transaction', 'vendor_payment']

    return " | ".join(parts)

# Metadata includes upload_id for filtering (CRITICAL FIX)
def _create_metadata(transaction, session_id=None, upload_id=None):
    metadata = {
        'date_timestamp': transaction.date.timestamp(),
        'amount': float(transaction.amount),
        'type': transaction.type.value,
        'category': transaction.category or 'uncategorized'
    }
    if upload_id:
        metadata["upload_id"] = upload_id  # Enables per-upload filtering
    if session_id:
        metadata["session_id"] = session_id
    return metadata
```

### Retrieval Methods

1. **Semantic Search**: Query-based similarity search
2. **Structured Filtering**: Date ranges, amounts, types
3. **Hybrid Retrieval**: Combines semantic + structured

```python
# Structured retrieval
where_clause = {
    "$and": [
        {"date_timestamp": {"$gte": start_timestamp}},
        {"date_timestamp": {"$lte": end_timestamp}},
        {"type": {"$eq": "debit"}}
    ]
}

# Semantic search
results = collection.query(
    query_texts=[query],
    n_results=limit,
    where=where_clause
)
```

## PDF Processing with Vision AI

### Vision AI Pipeline

```python
class VisionAIProcessor:
    def __init__(self):
        self.model = "qwen2.5:7b-vl"

    def process_page(self, image_path):
        # 1. Convert PDF page to image
        # 2. Send to Vision AI model
        # 3. Extract structured JSON
        # 4. Validate and clean data

        prompt = """Extract financial data as JSON:
        {
            "transactions": [...],
            "metadata": {...}
        }"""

        return ollama.generate(
            model=self.model,
            images=[image_path],
            prompt=prompt
        )
```

## Session Management

### Data Isolation

```python
# Each user session is isolated
session_id = datetime.now().strftime("%Y%m%d_%H%M%S")

# All data tagged with session
vector_store.add_transactions(transactions, session_id)

# Clear specific session
vector_store.clear_collection(session_id)

# Or clear everything
vector_store.clear_collection()  # No session_id = clear all
```

## Performance Characteristics

### System Metrics

| Metric | Value | Notes |
|--------|-------|-------|
| Query Response Time | ~2 seconds | Average for typical query |
| Transaction Capacity | 10,000+ | Per session |
| Embedding Dimensions | 384 | SentenceTransformers |
| Batch Processing Size | 100 | Transactions per batch |
| Context Window | 32k tokens | Qwen3 model |
| Vector Search Results | 10-50 | Configurable per query |

### Optimization Strategies

1. **Fast Query Understanding**: Gemma3:4b for intent extraction
2. **Pre-retrieved Data**: Agents receive data, don't search
3. **Batch Processing**: 100 transactions at a time
4. **Session Caching**: Reuse vector store across queries
5. **Lazy Loading**: Models loaded only when needed

## Security & Privacy

### Local-First Architecture

- **No Cloud APIs**: All processing on local machine
- **No External Calls**: Ollama runs locally
- **Session Isolation**: Each upload is separate
- **Temporary Storage**: Data in ./data/chroma_db
- **Clear Function**: Complete data removal
- **File Cleanup**: Temp files auto-deleted

### Data Protection

```python
# Automatic cleanup
try:
    # Process file
    process_uploaded_file(tmp_path)
finally:
    # Always cleanup
    if os.path.exists(tmp_path):
        os.unlink(tmp_path)
```

## Error Handling

### Graceful Degradation

```python
try:
    # Try with AI
    result = agent.execute(query)
except AIServiceError:
    # Fallback to simple search
    result = vector_store.search(query)
except Exception as e:
    # User-friendly error
    return "Unable to process. Please try rephrasing."
```

## API Reference

### Core Classes

```python
# Main application orchestrator
class YomnaiCore:
    def process_file(file_path: str) -> ProcessingResult
    def initialize_ai_components() -> bool
    def process_query(query: str) -> Dict[str, Any]

# Vector storage
class VectorStoreManager:
    def add_transactions(transactions: List, session_id: str)
    def search(query: str, n_results: int) -> Dict
    def retrieve_by_filters(filters: Dict) -> Dict
    def clear_collection(session_id: Optional[str])

# Multi-agent orchestrator
class AgentOrchestrator:
    def process_chat_query(query: str) -> Dict
    def generate_insights() -> Dict

# Query understanding
class QueryUnderstander:
    def analyze_and_retrieve(query: str) -> Dict
```

## Deployment

### Requirements

```bash
# System requirements
- Python 3.10+
- 8GB RAM minimum
- 2GB disk space for models
- Ollama installed

# Python dependencies
pip install -r requirements.txt

# AI models
ollama pull qwen3-14b-32k:latest
ollama pull gemma3:4b
ollama pull qwen2.5:7b-vl
```

### Running the Application

```bash
# Development
streamlit run app.py

# Production
streamlit run app.py \
  --server.port=8501 \
  --server.headless=true \
  --server.maxUploadSize=50
```

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Ollama not found | Ensure `ollama serve` is running |
| Model not available | Run `ollama pull [model]` |
| ChromaDB error | Delete ./data/chroma_db folder |
| Memory issues | Reduce batch_size in settings |
| Slow responses | Use smaller model (phi3 instead of qwen3) |

## Future Architecture Considerations

### Scalability Options

1. **Distributed Processing**: Agent parallelization
2. **Model Optimization**: Quantized models for speed
3. **Caching Layer**: Redis for frequent queries
4. **API Gateway**: REST API for integrations
5. **Multi-user Support**: User authentication system

### Extension Points

- Additional file format parsers
- New specialized agents
- Custom financial models
- Export formats (PDF, Excel reports)
- Scheduled analysis jobs

---

*This technical documentation represents the current production implementation of Yomnai, a sophisticated financial analysis system that prioritizes privacy while delivering professional-grade insights.*