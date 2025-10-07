# Yomnai - Private Financial Analyst

## 🎯 Project Overview

**Yomnai** is a sophisticated multi-agent financial analysis system that transforms complex financial data into actionable insights through natural conversation. All processing happens 100% locally on your machine, ensuring complete privacy while delivering professional-grade financial analysis.

## 🚀 Current Status: **Production Ready**

The system is fully implemented with:
- ✅ 6 Specialized AI Agents
- ✅ Multi-format support (CSV, Excel, PDF)
- ✅ Financial statement analysis
- ✅ Saudi Arabian business context
- ✅ Natural language interface
- ✅ Comprehensive dashboards

---

## 💡 The Vision

Transform financial analysis from a tedious spreadsheet task into a simple conversation:

**Instead of:** Wrestling with Excel formulas and pivot tables
**You can ask:**
- *"What were my top 5 expenses in June?"*
- *"How much did I pay in fees last quarter?"*
- *"Show me all SWIFT transfers above SAR 10,000"*
- *"What's my financial health score?"*

---

## 🎯 Problems Solved

### 1. **Data Overload**
- Bank statements are data-rich but insight-poor
- Hundreds of transactions with cryptic descriptions
- Multiple formats and inconsistent structures

### 2. **Privacy Concerns**
- No need to upload sensitive data to cloud services
- All processing happens locally
- Your financial data never leaves your computer

### 3. **Complexity**
- No financial expertise required
- Natural language questions instead of complex queries
- AI explains findings in simple terms

### 4. **Business Context**
- Understands Saudi Arabian banking (SADAD, GOSI, QIWA)
- Recognizes business transactions vs personal
- Handles Arabic transaction descriptions

---

## 🏗️ System Architecture

### Core Components

```
User Interface (Streamlit)
           ↓
    Document Detection
    ├── Transactions → CSV/Excel Parser
    └── Statements → Financial Parser / PDF Vision AI
           ↓
    Data Validation & Storage
    ├── ChromaDB (Vector Database)
    └── Session Management
           ↓
    Multi-Agent AI System
    ├── Query Understanding (Gemma 3:4b)
    ├── Agent Router
    └── 6 Specialized Agents
           ↓
    Response Generation (Qwen3-14b)
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Frontend** | Streamlit | Web interface & dashboards |
| **Parsing** | Pandas, pdf2image | Data extraction |
| **Vector DB** | ChromaDB | Semantic search & storage |
| **AI/LLM** | Ollama | Local language models |
| **Models** | Qwen3-14b, Gemma3:4b, Qwen2.5-VL | Analysis, routing, vision |
| **RAG** | LangChain | Query orchestration |
| **Language** | Python 3.10+ | Core implementation |

---

## 🤖 Multi-Agent System

### 6 Specialized Financial Agents

1. **ExpenseAgent** - Business expense analysis & categorization
2. **IncomeAgent** - Revenue stream tracking & stability analysis
3. **FeeHunterAgent** - Fee detection & savings opportunities
4. **BudgetAdvisorAgent** - Cash flow & financial health scoring
5. **TrendAnalystAgent** - Pattern recognition & predictions
6. **TransactionInvestigatorAgent** - Specific transaction search

### Intelligent Orchestration

- **QueryUnderstander** - Extracts intent and filters using Gemma3:4b
- **AgentRouter** - Routes queries to appropriate agents
- **Hidden Thinking** - Multi-step reasoning behind the scenes
- **Response Synthesis** - Combines insights from multiple agents

---

## 📊 Supported Document Types

### Transaction Files
- **CSV Bank Statements** - Intelligent header/footer detection
- **Excel Exports** - Multi-sheet support
- **Formats**: Various bank formats with auto-detection

### Financial Statements
- **Balance Sheets** - Assets, liabilities, equity
- **Income Statements** - Revenue, expenses, profit metrics
- **Cash Flow Statements** - Operating, investing, financing
- **PDF Documents** - Via Vision AI (Qwen2.5-VL)

---

## 🎨 User Experience

### 1. **Smart Upload**
- Drag & drop any financial document
- Automatic type detection
- Intelligent parsing based on document structure

### 2. **Interactive Dashboards**
- **Overview** - Key metrics and health scores
- **Chat** - Natural language queries
- **Analysis** - Multi-agent insights
- **Trends** - Visual charts and patterns
- **Transactions** - Detailed data view
- **Statistics** - Comprehensive metrics

### 3. **Natural Conversation**
- Ask questions in plain English
- Get instant, accurate answers
- See which transactions were used
- Understand the AI's reasoning

---

## 🔒 Privacy & Security

### 100% Local Processing
- **No Cloud APIs** - Everything runs on your machine
- **No Data Upload** - Files never leave your computer
- **Session Isolation** - Each session is separate
- **Temporary Storage** - Data cleared on demand
- **Local LLMs** - AI models run via Ollama

---

## 📈 Key Features

### Intelligent Analysis
- ✅ Automatic transaction categorization
- ✅ Spending pattern detection
- ✅ Income stability analysis
- ✅ Fee hunting and optimization
- ✅ Financial health scoring
- ✅ Trend predictions

### Business Features
- ✅ Payroll tracking
- ✅ Vendor payment analysis
- ✅ SWIFT transfer detection
- ✅ Government compliance (GOSI, QIWA, SADAD)
- ✅ VAT calculations
- ✅ Multi-currency support

### Technical Capabilities
- ✅ Handles messy CSV formats
- ✅ PDF processing with Vision AI
- ✅ Semantic search across transactions
- ✅ Context-aware responses
- ✅ Conversation memory
- ✅ Multi-agent coordination

---

## 🚀 Getting Started

### Prerequisites
1. Python 3.10+
2. Ollama installed locally
3. ~2GB disk space for models

### Quick Setup
```bash
# Install dependencies
pip install -r requirements.txt

# Pull AI models
ollama pull qwen3-14b-32k:latest
ollama pull gemma3:4b
ollama pull qwen2.5:7b-vl

# Run application
streamlit run app.py
```

---

## 📊 Performance Metrics

- **Query Response**: ~2 seconds average
- **Transaction Capacity**: 10,000+ transactions
- **Embedding Dimensions**: 384
- **Batch Size**: 100 transactions
- **Models**: 3 specialized LLMs
- **Agents**: 6 domain experts
- **Tools**: 23 AI functions

---

## 🔮 Future Enhancements

### Planned Features
- Multi-file analysis across time periods
- Predictive analytics and forecasting
- Automated budget recommendations
- Custom alert systems
- Export to professional reports

### Technical Roadmap
- Additional language support
- More bank format parsers
- Enhanced visualization options
- API for external integrations
- Desktop application packaging

---

## 📚 Documentation

- **[Technical Documentation](./TECHNICAL_DOCUMENTATION.md)** - Deep technical details
- **[Agent System](./Agents/)** - Multi-agent architecture
- **[PDF Processing](./PDF_Processing/)** - Vision AI implementation
- **[Implementation History](./Implementation_Phases/)** - Development phases

---

## 🌟 Why Yomnai?

1. **Complete Privacy** - Your data never leaves your computer
2. **Professional Analysis** - Enterprise-grade insights
3. **Easy to Use** - Natural language, no expertise needed
4. **Saudi Context** - Understands local banking
5. **Comprehensive** - Handles all financial documents
6. **Transparent** - See exactly how AI reaches conclusions

---

## 📝 License & Support

Yomnai is a private financial analyst that prioritizes your privacy while delivering professional-grade analysis. Built with modern AI technology, it represents the future of personal and business financial management.

**Version**: 2.0
**Status**: Production Ready
**Last Updated**: September 2025