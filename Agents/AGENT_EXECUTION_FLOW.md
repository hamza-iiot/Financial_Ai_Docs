# Agent Execution Flow - Complete Process

**Last Updated:** October 5, 2025

This document explains the **exact step-by-step execution flow** when a user asks a query, including the actual Python prompt construction, LLM calls, and **dual-mode execution**.

## Overview

When a user asks a question like "What are my top expenses this month?", the flow depends on the **execution mode**:

### INSIGHTS MODE (First Time / Deep Analysis)
```
User Query → Cache Check (Miss) → Gemma Router → Qwen Thinking (think=true, 20s)
→ Data Analysis → Qwen Response (think=true, 20s) → Cache Result → Return
Total: ~50 seconds
```

### CHAT MODE (Subsequent Questions with Cache)
```
User Query → Cache Check (Hit!) → Gemma Router → Retrieve Cached Analysis
→ Qwen Response (think=false, 5s) → Return
Total: ~5-10 seconds (10x faster!)
```

---

## 🚀 Dual-Mode Execution Details (NEW)

### Cache Check Process

Every query starts with a cache check:

```python
# In orchestrator.py - process_chat_query()
cached_insights = session_cache.get_transaction_insights(session_id)

if cached_insights and cached_insights.get(agent_type):
    mode = "chat"  # Use fast mode
    cached_analysis = cached_insights[agent_type]
else:
    mode = "insights"  # Full analysis
    cached_analysis = None
```

### Mode Decision Tree

```
Query Received
    ↓
Cache Check
    ├─ Cache Hit (insights < 24h old)
    │   ↓
    │  CHAT MODE
    │   ├─ Skip thinking (saves 20s)
    │   ├─ Use cached analysis as context
    │   ├─ Optional fresh search if needed
    │   └─ Generate with think=false (5s)
    │
    └─ Cache Miss (no insights or expired)
        ↓
       INSIGHTS MODE
        ├─ Full thinking (20s)
        ├─ Data analysis (5s)
        ├─ Generate with think=true (20s)
        └─ Cache result for 24h
```

---

## The Complete Flow - Using Expense Query Example

### Step 1: Intent Classification (Gemma3:4b - Fast Router)

**Purpose**: Quickly determine which agent should handle this query

**Model**: Gemma3:4b (fast, lightweight)

**Code Location**: `yomnai_backend/agents/router.py`

```python
# The router sends this to Gemma:
router_prompt = f"""You are a query router. Classify this query into ONE category:
- expense
- income
- fee
- budget
- trend
- transaction

Query: "{user_query}"

Return ONLY the category name."""

# Gemma responds with just:
intent = "expense"  # Single word classification
```

**Output**: Just the intent type (`"expense"`)

**Time**: ~0.3 seconds

---

### Step 2: Agent Selection & Initialization

**Code Location**: `yomnai_backend/agents/orchestrator.py`

```python
# Orchestrator picks the agent based on intent:
if intent == "expense":
    agent = ExpenseAgent(
        vector_store=vector_store,
        ollama_client=ollama_client
    )
```

---

### Step 3: Hidden Reasoning - First Qwen Call

**Purpose**: Deep analysis of the query to plan the analysis

**Model**: Qwen3-14B-32k (deep reasoning, 32k context)

**Code Location**: `yomnai_backend/agents/expense_agent.py`

#### The Python Prompt Construction

```python
def execute_with_data(self, query: str, transactions: List[Transaction]):
    # STEP 1: Build the hidden reasoning prompt
    thinking_prompt = f"""You are a business financial analyst specializing in Saudi Arabian corporate banking.

Think deeply about this expense-related query in the context of Saudi Arabian transactions.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):

1. **Time Period**: Is the user asking about a specific period?
   - This month/quarter/year?
   - Specific date range?
   - Comparative period (e.g., "compared to last month")?

2. **Expense Categories**: Which categories are relevant?
   - Payroll & Benefits (رواتب)
   - Rent & Utilities (إيجار ومرافق)
   - Professional Fees (رسوم مهنية)
   - Government Fees (GOSI, QIWA, SADAD)
   - Travel & Entertainment
   - Office & Supplies
   - Insurance & Legal
   - Marketing & Sales
   - IT & Software
   - Miscellaneous

3. **Analysis Type**: What specific analysis is needed?
   - Total spending breakdown?
   - Category comparison?
   - Trend identification?
   - Outlier detection?
   - Savings opportunities?

4. **Business Context**: Saudi-specific considerations?
   - Government compliance payments (GOSI, QIWA)?
   - SADAD recurring charges?
   - Currency (SAR)?
   - Arabic transaction descriptions?

5. **Data Requirements**: What data do I need to retrieve?
   - All expenses or filtered subset?
   - Specific vendors or categories?
   - Date range filters?

Think step-by-step and outline your analysis approach."""

    # STEP 2: Send to Qwen for hidden reasoning
    hidden_thinking = self.ollama_client.generate(
        prompt=thinking_prompt,
        system_prompt="You are an expert financial analyst. Think systematically."
    )

    # hidden_thinking now contains Qwen's analysis plan
```

#### Qwen's Hidden Response (Example)

```
Based on the query "What are my top expenses this month?":

1. TIME PERIOD: Current month (January 2025)
   - Need to filter transactions from 2025-01-01 to 2025-01-31

2. EXPENSE CATEGORIES: ALL categories
   - User wants "top" expenses, so rank by amount across all categories

3. ANALYSIS TYPE: Spending breakdown with ranking
   - Aggregate by category
   - Sort by total amount descending
   - Show top 5-7 categories

4. BUSINESS CONTEXT:
   - Check for GOSI/QIWA payments (mandatory)
   - Look for SADAD recurring charges
   - Present amounts in SAR

5. DATA REQUIREMENTS:
   - Filter: type = 'debit', date >= 2025-01-01, date <= 2025-01-31
   - Group by: category
   - Sort by: total amount DESC
   - Limit: Top 5-7 categories
```

**Output**: Hidden reasoning plan (not shown to user)

**Time**: ~1.5 seconds

---

### Step 4: Data Retrieval & Analysis (Python Tools)

**Purpose**: Execute the analysis plan using Python libraries

**Tools Used**: ChromaDB, Pandas, NumPy

**Code Location**: `yomnai_backend/agents/expense_agent.py`

```python
    # STEP 3: Based on Qwen's reasoning, execute data retrieval

    # Tool 1: ChromaDB - Semantic search for relevant transactions
    relevant_transactions = self.vector_store.search_transactions(
        query=query,
        n_results=100,
        filter_type="debit"  # Expenses only
    )

    # Tool 2: Pandas - Convert to DataFrame for analysis
    import pandas as pd
    df = pd.DataFrame([{
        'date': t.date,
        'description': t.description,
        'amount': t.amount,
        'category': t.category,
        'currency': t.currency
    } for t in relevant_transactions])

    # Tool 3: Pandas - Filter by date (current month)
    from datetime import datetime
    current_month_start = datetime(2025, 1, 1)
    current_month_end = datetime(2025, 1, 31)

    df_filtered = df[
        (df['date'] >= current_month_start) &
        (df['date'] <= current_month_end)
    ]

    # Tool 4: Pandas - Aggregate by category
    category_totals = df_filtered.groupby('category').agg({
        'amount': ['sum', 'count', 'mean']
    }).round(2)

    category_totals.columns = ['Total', 'Count', 'Average']
    category_totals = category_totals.sort_values('Total', ascending=False)

    # Get top 5 categories
    top_expenses = category_totals.head(5)

    # Calculate percentages
    total_spending = df_filtered['amount'].sum()
    top_expenses['Percentage'] = (top_expenses['Total'] / total_spending * 100).round(1)
```

#### Data Results (Example)

```python
# top_expenses DataFrame:
#
# Category                  Total      Count    Average    Percentage
# Payroll & Benefits       245000.00    15      16333.33   52.3%
# Rent & Utilities          85000.00     3      28333.33   18.1%
# Professional Fees         45000.00     8       5625.00    9.6%
# Government Fees (GOSI)    38000.00     2      19000.00    8.1%
# IT & Software             28000.00    12       2333.33    6.0%
#
# Total Monthly Spending: 468,000 SAR
```

**Output**: Structured data ready for response generation

**Time**: ~0.5 seconds

---

### Step 5: Response Generation - Second Qwen Call

**Purpose**: Convert analysis results into natural language response

**Model**: Qwen3-14B-32k

**Code Location**: `yomnai_backend/agents/expense_agent.py`

#### The Python Prompt Construction

```python
    # STEP 4: Build the response generation prompt
    response_prompt = f"""You are a helpful financial analyst presenting expense analysis to a business owner.

QUERY: {query}

ANALYSIS RESULTS:
{top_expenses.to_string()}

Total Monthly Spending: {total_spending:,.2f} SAR

KEY INSIGHTS:
- Top expense category: {top_expenses.index[0]} ({top_expenses.iloc[0]['Percentage']}% of total)
- Number of expense categories: {len(category_totals)}
- Average transaction size: {df_filtered['amount'].mean():,.2f} SAR

INSTRUCTIONS:
1. Present the findings in a clear, business-friendly format
2. Use both English and Arabic (عربي) where appropriate for Saudi context
3. Highlight the top 5 expense categories with amounts and percentages
4. Provide actionable insights:
   - Which categories dominate spending?
   - Are there any unusual patterns?
   - Any potential savings opportunities?
5. Keep the tone professional but conversational

Write your response now:"""

    # STEP 5: Send to Qwen for final response generation
    final_response = self.ollama_client.generate(
        prompt=response_prompt,
        system_prompt="You are a professional financial analyst. Be clear, concise, and helpful."
    )

    return final_response
```

#### Qwen's Final Response (Example)

```
**Top Expenses This Month (January 2025)**

Here's your expense breakdown for this month:

**📊 Top 5 Expense Categories:**

1. **Payroll & Benefits (رواتب ومزايا)** - 245,000 SAR (52.3%)
   - 15 transactions, averaging 16,333 SAR each

2. **Rent & Utilities (إيجار ومرافق)** - 85,000 SAR (18.1%)
   - 3 transactions, averaging 28,333 SAR each

3. **Professional Fees (رسوم مهنية)** - 45,000 SAR (9.6%)
   - 8 transactions, averaging 5,625 SAR each

4. **Government Fees - GOSI (التأمينات)** - 38,000 SAR (8.1%)
   - 2 transactions (mandatory compliance)

5. **IT & Software** - 28,000 SAR (6.0%)
   - 12 transactions, averaging 2,333 SAR each

**Total Monthly Spending: 468,000 SAR**

**💡 Key Insights:**

- Your payroll dominates spending at 52%, which is typical for service businesses
- Fixed costs (rent + utilities) represent 18% - relatively stable
- Government compliance (GOSI) is on track at 8%
- IT spending shows 12 small transactions - consider consolidating vendors for better rates

**🎯 Recommendations:**
- Review IT subscriptions - multiple small charges suggest potential duplicates
- Professional fees vary widely - negotiate retainer agreements for consistency
```

**Output**: User-facing natural language response

**Time**: ~2 seconds

---

## Complete Execution Summary

### Total Process Flow

```
1. User Query: "What are my top expenses this month?"
   ↓
2. Gemma Router (0.3s): intent = "expense"
   ↓
3. Qwen Hidden Reasoning (1.5s): Analysis plan
   ↓
4. Python Tools (0.5s): Data retrieval & analysis
   - ChromaDB: Get relevant transactions
   - Pandas: Filter, aggregate, calculate
   ↓
5. Qwen Response Generation (2s): Natural language output
   ↓
6. User sees final response
```

**Total Time**: ~4.3 seconds

### LLM Usage Breakdown

| Step | Model | Purpose | Prompt Type |
|------|-------|---------|-------------|
| 1 | Gemma3:4b | Intent classification | Router prompt |
| 2 | Qwen3-14B | Hidden reasoning | Thinking prompt |
| 3 | None | Data analysis | Python code |
| 4 | Qwen3-14B | Response generation | Response prompt |

### Key Points

1. **Gemma ONLY routes** - No reasoning, just classification
2. **Qwen does reasoning AND response** - Two separate calls
3. **Python tools do data work** - No LLM needed for calculations
4. **Prompts are Python f-strings** - The agent code constructs them programmatically

## All 12 Agents Follow This Pattern

Every agent (ExpenseAgent, IncomeAgent, FeeAgent, etc.) follows this exact flow:

1. Router picks the agent (Gemma)
2. Agent does hidden thinking (Qwen)
3. Agent retrieves/analyzes data (Python)
4. Agent generates response (Qwen)

The only differences are:
- The specific prompt content
- The tools/calculations used
- The response format

---

## Code References

- **Router**: `/yomnai_backend/agents/router.py:35` (Gemma classification)
- **Orchestrator**: `/yomnai_backend/agents/orchestrator.py:78` (Agent selection)
- **ExpenseAgent**: `/yomnai_backend/agents/expense_agent.py:45` (Full execution)
- **Ollama Client**: `/yomnai_backend/agents/ollama_client.py:28` (LLM calls)
- **Vector Store**: `/yomnai_backend/agents/vector_store.py:67` (ChromaDB search)

---

**Last Updated**: January 2025
