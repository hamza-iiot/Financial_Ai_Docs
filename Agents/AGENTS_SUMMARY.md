# Yomnai AI Agents - Quick Reference

**Last Updated:** October 5, 2025

## 🚀 Dual-Mode Execution (NEW)

**Status:** ✅ Infrastructure Complete | ⏳ 2 of 12 agents updated

All agents now support:
- **INSIGHTS MODE**: Deep analysis with `think=true` (~50s per agent) - Cached for 24h
- **CHAT MODE**: Fast responses with `think=false` (~5s) - Uses cached context

**Performance:** 10x faster chat responses when insights are cached!

---

## Transaction Agents (6)

Analyze bank transaction data from CSV/Excel files.

### 1. ExpenseAgent 💰
**Description:** Analyzes business expenses, categorizes spending, and identifies cost-saving opportunities for Saudi corporate banking.

**Tools:** ChromaDB (vector search), Pandas (data analysis), Qwen3-14B (LLM reasoning)

**Dual-Mode:** ✅ Fully implemented

---

### 2. IncomeAgent 💵
**Description:** Tracks revenue streams, identifies income patterns, and analyzes business income stability and growth.

**Tools:** ChromaDB (vector search), Pandas (data analysis), Qwen3-14B (LLM reasoning)

**Dual-Mode:** ✅ Fully implemented

---

### 3. FeeHunterAgent 🔍
**Description:** Detects banking fees, recurring charges, and calculates potential savings from fee optimization.

**Tools:** ChromaDB (vector search), Pandas (fee calculations), Qwen3-14B (LLM reasoning)

---

### 4. BudgetAdvisorAgent 📊
**Description:** Evaluates cash flow health, calculates financial health scores (0-100), and provides budget recommendations.

**Tools:** ChromaDB (vector search), Pandas (cash flow analysis), Qwen3-14B (LLM reasoning)

---

### 5. TrendAnalystAgent 📉
**Description:** Identifies spending trends, revenue patterns, and seasonal business cycles using time-series analysis.

**Tools:** ChromaDB (vector search), Pandas (data aggregation), NumPy (trend calculations), Qwen3-14B (LLM reasoning)

---

### 6. TransactionInvestigatorAgent 🕵️
**Description:** Searches for specific transactions by amount, merchant, or keywords with relevance-based ranking.

**Tools:** ChromaDB (semantic search), Regex (parameter extraction), Pandas (ranking), Qwen3-14B (LLM reasoning)

---

## Financial Statement Agents (6)

Analyze financial statements (Balance Sheet, P&L, Cash Flow).

### 7. RatioAnalystAgent 📈
**Description:** Calculates 13+ financial ratios (liquidity, profitability, leverage) and provides health scoring (0-100).

**Tools:** Python calculations (ratio formulas), Qwen3-14B (LLM analysis)

---

### 8. ProfitabilityAgent 💹
**Description:** Analyzes profit margins, cost structure, and revenue optimization strategies with YoY comparisons.

**Tools:** Python calculations (margin analysis), Qwen3-14B (LLM analysis)

---

### 9. LiquidityAgent 💧
**Description:** Evaluates working capital, cash position, and short-term financial stability for payment capacity.

**Tools:** Python calculations (liquidity metrics), Qwen3-14B (LLM analysis)

---

### 10. FinancialTrendAgent 📊
**Description:** Tracks financial statement trends over time, growth rates, and forecasts future performance.

**Tools:** Python calculations (growth analysis), Qwen3-14B (LLM analysis)

---

### 11. RiskAssessmentAgent ⚠️
**Description:** Identifies financial risks, compliance issues, and provides risk scoring for Saudi business regulations.

**Tools:** Python calculations (risk metrics), Qwen3-14B (LLM analysis)

---

### 12. EfficiencyAgent ⚙️
**Description:** Measures operational efficiency, asset utilization, and productivity metrics for business optimization.

**Tools:** Python calculations (efficiency ratios), Qwen3-14B (LLM analysis)

---

## Models Used

- **Gemma3:4b** - Fast intent classification (router only)
- **Qwen3-14B-32k** - Deep analysis and response generation (all agents)
- **ChromaDB** - Vector database for transaction storage (transaction agents only)
- **SentenceTransformers** - Embeddings for semantic search (all-MiniLM-L6-v2)

---

**Total:** 12 specialized agents working together for comprehensive financial analysis.
