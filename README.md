# Yomnai Documentation

## 📚 Documentation Structure

The Yomnai documentation is organized into logical sections for easy navigation:

```
docs/
├── Agents/                      # Multi-Agent AI System
├── PDF_Processing/              # PDF Processing with Vision AI
├── Implementation_Phases/       # Historical Development Phases (Complete)
└── [Main Documentation Files]   # Core Technical Docs
```

---

## 📁 Main Documentation Files

### **[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)**
- Complete project overview and vision
- Current status and capabilities
- System architecture diagram
- Getting started guide
- Why Yomnai?

### **[TECHNICAL_DOCUMENTATION.md](./TECHNICAL_DOCUMENTATION.md)**
- Deep technical architecture
- Complete technology stack
- Data flow and processing pipeline
- Multi-agent system details
- API reference and deployment

---

## 🤖 Agents/ - Multi-Agent AI System

### Core Architecture
- **[Financial_Agents_Architecture.md](./Agents/Financial_Agents_Architecture.md)** - Main hub with agent summaries and system overview
- **[Agent_Router_System.md](./Agents/Agent_Router_System.md)** - Query routing and orchestration

### Specialized Agents (6 Total)
1. **[Expense_Agent.md](./Agents/Expense_Agent.md)** - Business expense analysis
2. **[Income_Agent.md](./Agents/Income_Agent.md)** - Revenue stream tracking
3. **[Fee_Hunter_Agent.md](./Agents/Fee_Hunter_Agent.md)** - Fee detection and optimization
4. **[Budget_Advisor_Agent.md](./Agents/Budget_Advisor_Agent.md)** - Cash flow and budget health
5. **[Trend_Analyst_Agent.md](./Agents/Trend_Analyst_Agent.md)** - Pattern and trend analysis
6. **[Transaction_Investigator_Agent.md](./Agents/Transaction_Investigator_Agent.md)** - Specific transaction search

---

## 📄 PDF_Processing/ - Vision AI Document Processing

- **[PDF_PROCESSING_README.md](./PDF_Processing/PDF_PROCESSING_README.md)** - Main guide for PDF processing features
- **[PDF_PROCESSING_STRATEGY.md](./PDF_Processing/PDF_PROCESSING_STRATEGY.md)** - Vision AI approach and architecture
- **[PDF_IMPLEMENTATION_CHANGELOG.md](./PDF_Processing/PDF_IMPLEMENTATION_CHANGELOG.md)** - Implementation history and updates

### Key Features:
- Automatic document type detection
- Vision AI with Qwen2.5-VL model
- Supports bank statements and financial statements
- 100% local processing

---

## 📋 Implementation_Phases/ - Development Roadmap (Historical)

**Note:** These phases are now complete. Yomnai is fully functional with all core features implemented.

1. **Phase_1_Foundation_Setup.md** - Project structure and dependencies
2. **Phase_2_Data_Processing_Pipeline.md** - CSV parsing and validation
3. **Phase_3_Vector_Database_Integration.md** - ChromaDB setup
4. **Phase_4_AI_LLM_Integration.md** - Ollama and RAG pipeline
5. **Phase_5_Streamlit_Frontend.md** - User interface
6. **Phase_6_Core_Features_Implementation.md** - Integration and security
7. **Phase_7_Testing_Validation.md** - Test suites
8. **Phase_8_Enhancement_Optimization.md** - Advanced features
9. **Phase_9_Future_Features.md** - Future development ideas

---

## 🚀 Quick Navigation

### For Users
- Start with [TECHNICAL_DOCUMENTATION.md](./TECHNICAL_DOCUMENTATION.md) for system overview
- Check [Financial_Agents_Architecture.md](./Agents/Financial_Agents_Architecture.md) for AI capabilities

### For Developers
- [Agent documentation](./Agents/) for understanding the multi-agent system
- [PDF Processing](./PDF_Processing/) for Vision AI implementation
- [Implementation Phases](./Implementation_Phases/) for historical context

### Key Technologies
- **Frontend:** Streamlit
- **AI/LLM:** Ollama (Qwen3-14b, Gemma3:4b, Qwen2.5-VL)
- **Vector DB:** ChromaDB
- **RAG:** LangChain
- **Language:** Python 3.10+

---

## 🔒 Privacy & Security

All Yomnai features maintain 100% local processing:
- No cloud APIs
- No data uploads
- Complete privacy
- Local LLM models via Ollama
- Session-based data isolation

---

## 📈 Current Status

✅ **Fully Implemented:**
- Multi-agent financial analysis system
- Transaction processing (CSV/Excel)
- Financial statement analysis
- PDF processing with Vision AI
- Natural language queries
- Saudi Arabian business context
- Comprehensive UI with dashboards

🚧 **Future Enhancements:**
- Multi-file analysis
- Predictive analytics
- Budget management system
- Automated insights engine

---

## 📞 Support

For questions or issues:
- Review the [Technical Documentation](./TECHNICAL_DOCUMENTATION.md)
- Check the [Agent Architecture](./Agents/Financial_Agents_Architecture.md)
- Consult implementation phases for detailed code examples