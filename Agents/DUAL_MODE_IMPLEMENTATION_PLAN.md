# Dual-Mode Agent System - Implementation Plan

## ✅ STATUS: FULLY IMPLEMENTED (January 2025)

All 12 agents (6 transaction + 6 financial) now support dual-mode execution with think parameter control.

## Overview

Implement a two-mode agent execution system to optimize for different use cases:
- **Insights Mode** (Deep Analysis): Full thinking with `think=true`, takes ~50s, generates comprehensive cached analysis
- **Chat Mode** (Fast Responses): Uses cached analysis as context with `think=false`, takes ~5-10s

## Core Concept

### Insights Mode (Generate Financial Insights Button):
```
User clicks "Generate Insights"
    ↓
Run all 6 agents with FULL pipeline (think=true)
    ↓
Cache all 6 agent analysis results in session
    ↓
Display comprehensive insights to user
```

### Chat Mode (Conversational Questions):
```
User asks: "What were my top expenses?"
    ↓
Retrieve cached expense analysis from session
    ↓
Use cached analysis as CONTEXT
    ↓
Optionally search for additional data if needed
    ↓
Generate response with think=false (fast)
    ↓
Return answer in ~5-10s
```

---

## Key Principle

**Every chat question uses one of the 6 transaction agents, and each agent ALWAYS has a cached analysis response available** because insights were generated first.

---

## Implementation Steps

### 1. Add `think` Parameter Support to Ollama Client

**File:** `/yomnai_backend/agents/ollama_client.py`

**Change:**
```python
def generate(self, prompt: str, system_prompt: str = None,
             temperature: float = 0.7, max_tokens: int = 1000,
             think: bool = True):  # ← Add think parameter
    """Generate response from Ollama"""

    # Build request with think flag
    request = {
        "model": self.model,
        "prompt": prompt,
        "system": system_prompt,
        "temperature": temperature,
        "options": {
            "num_predict": max_tokens,
            "think": think  # ← Pass to Ollama
        }
    }

    response = requests.post(f"{self.base_url}/api/generate", json=request)
    return response.json()['response']
```

---

### 2. Modify All 6 Transaction Agents to Support Modes

**Files:**
- `/yomnai_backend/agents/expense_agent.py`
- `/yomnai_backend/agents/income_agent.py`
- `/yomnai_backend/agents/fee_agent.py`
- `/yomnai_backend/agents/budget_agent.py`
- `/yomnai_backend/agents/trend_agent.py`
- `/yomnai_backend/agents/transaction_agent.py`

**Changes for Each Agent:**

```python
def execute(self, query: str, thinking_context: Dict = None,
            mode: str = "insights", cached_analysis: Dict = None) -> Dict[str, Any]:
    """
    Execute agent analysis with mode support

    Args:
        query: User query
        thinking_context: Additional context
        mode: "insights" or "chat"
        cached_analysis: Previous analysis results (for chat mode)
    """

    # INSIGHTS MODE: Full pipeline with thinking
    if mode == "insights":
        # Step 1: Deep thinking (think=true)
        thinking = self._think_hidden(
            thinking_prompt,
            system_prompt,
            think=True  # ← Deep reasoning
        )

        # Step 2: Data retrieval and analysis
        data = self._gather_data()
        analysis = self._analyze_data(data, thinking)

        # Step 3: Final response (think=true) WITH THINKING CONTEXT
        response = self.ollama.generate(
            prompt=f"""User Query: {query}

            Your Internal Thinking:
            {thinking['reasoning']}

            Analysis Results:
            {analysis}

            Provide comprehensive answer based on your thinking and analysis.""",
            system_prompt=self._get_system_prompt(),
            think=True  # ← Deep generation
        )

        return {
            'final_answer': response,
            'thinking': thinking,
            'analysis': analysis,
            'mode': 'insights'
        }

    # CHAT MODE: Fast response with cached context
    elif mode == "chat":
        # Step 1: Skip thinking entirely (saves 20s)

        # Step 2: Check if we have pre-retrieved filtered transactions (NEW FIX)
        additional_data = None
        if hasattr(self, '_retrieved_transactions') and self._retrieved_transactions:
            # We have filtered transactions from base agent - analyze them directly
            logger.info(f"Analyzing {len(self._retrieved_transactions)} filtered transactions")
            additional_data = self._analyze_data(self._retrieved_transactions)
        elif self._needs_fresh_search(query):
            # No pre-retrieved data, but query needs specific search
            additional_data = self._gather_data(query)  # Quick targeted search

        # Step 3: Generate response prioritizing filtered data over cached analysis
        if additional_data:
            # FILTERED DATA PRIORITY (NEW FIX)
            response = self.ollama.generate(
                prompt=f"""Analyze these FILTERED transactions for the user's specific query:

                FILTERED DATA (for this specific query):
                {additional_data}

                BACKGROUND CONTEXT (full dataset analysis):
                The complete dataset analysis is: {cached_analysis['final_answer']}

                USER QUESTION: {query}

                Answer using the FILTERED DATA above (not the background context).
                Be concise and conversational.""",
                system_prompt=self._get_system_prompt(),
                think=False  # ← Fast generation
            )
        else:
            # Use cached analysis
            response = self.ollama.generate(
                prompt=f"""You previously analyzed this data comprehensively:

                CACHED ANALYSIS:
                {cached_analysis['final_answer']}

                CACHED STATISTICS:
                {cached_analysis.get('analysis', {})}

                USER QUESTION: {query}

                Answer the user's question using the cached analysis as context.
                Be concise and conversational.""",
                system_prompt=self._get_system_prompt(),
                think=False  # ← Fast generation (no internal reasoning)
            )

        return {
            'final_answer': response,
            'mode': 'chat',
            'used_cache': True,
            'fresh_search': additional_data is not None
        }

def _needs_fresh_search(self, query: str) -> bool:
    """Determine if query needs additional data beyond cached analysis"""
    # Check if asking for very specific transaction details
    specific_keywords = ['show me', 'list', 'find transaction', 'specific', 'exact']
    return any(keyword in query.lower() for keyword in specific_keywords)
```

**CRITICAL ADDITION:** Pass internal thinking to final response prompt in insights mode so the final answer generation can reference the structured reasoning.

---

### 3. Update Agent Orchestrator

**File:** `/yomnai_backend/agents/orchestrator.py`

**Changes:**

```python
class AgentOrchestrator:
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.agents = {
            'expense': ExpenseAgent(vector_store, ollama_client),
            'income': IncomeAgent(vector_store, ollama_client),
            'fee': FeeHunterAgent(vector_store, ollama_client),
            'budget': BudgetAdvisorAgent(vector_store, ollama_client),
            'trend': TrendAnalystAgent(vector_store, ollama_client),
            'transaction': TransactionInvestigatorAgent(vector_store, ollama_client)
        }

    def generate_insights(self, session_id: str) -> Dict[str, Any]:
        """
        Generate comprehensive insights using all agents (INSIGHTS MODE)
        Runs with think=true, takes ~50s per agent
        """
        insights = {}

        for agent_name, agent in self.agents.items():
            result = agent.execute(
                query=f"Provide comprehensive {agent_name} analysis",
                mode="insights"  # ← Full pipeline
            )
            insights[agent_name] = result

        # Cache results in session (implementation depends on session storage)
        self._cache_insights(session_id, insights)

        return insights

    def process_chat_query(self, query: str, session_id: str) -> Dict[str, Any]:
        """
        Process chat query using cached insights (CHAT MODE)
        Runs with think=false, takes ~5-10s
        """
        # Route to appropriate agent
        agent_type = self._route_query(query)  # Uses Gemma router
        agent = self.agents[agent_type]

        # Retrieve cached analysis for this agent
        cached_insights = self._get_cached_insights(session_id)
        cached_analysis = cached_insights.get(agent_type, {})

        if not cached_analysis:
            # Fallback: run quick analysis without thinking
            return agent.execute(query, mode="insights")  # Still need full analysis

        # Use cached analysis as context
        result = agent.execute(
            query=query,
            mode="chat",  # ← Fast mode
            cached_analysis=cached_analysis
        )

        return result

    def _cache_insights(self, session_id: str, insights: Dict):
        """Store insights in session (24-hour expiry)"""
        # Implementation depends on backend session storage
        # Could use Redis, file-based, or in-memory
        pass

    def _get_cached_insights(self, session_id: str) -> Dict:
        """Retrieve cached insights from session"""
        # Implementation depends on backend session storage
        pass
```

---

### 4. Update Backend API Endpoints

**File:** `/yomnai_backend/app/api/endpoints/analysis.py` (or similar)

**Add Insights Generation Endpoint:**
```python
@router.post("/generate-insights")
async def generate_insights(session_id: str):
    """
    Generate comprehensive financial insights (runs all 6 agents)
    Takes ~5-6 minutes total (50s × 6 agents)
    """
    orchestrator = AgentOrchestrator(vector_store, ollama_client)
    insights = orchestrator.generate_insights(session_id)

    return {
        "status": "success",
        "insights": insights,
        "cached": True,
        "timestamp": datetime.now().isoformat()
    }
```

**Update Chat Endpoint:**
```python
@router.post("/chat")
async def chat_query(query: str, session_id: str):
    """
    Process chat query using cached insights
    Takes ~5-10s
    """
    orchestrator = AgentOrchestrator(vector_store, ollama_client)
    result = orchestrator.process_chat_query(query, session_id)

    return {
        "status": "success",
        "answer": result['final_answer'],
        "mode": result['mode'],
        "used_cache": result.get('used_cache', False),
        "agent_type": result.get('agent_type')
    }
```

---

### 5. Session Cache Implementation

**File:** `/yomnai_backend/app/services/session_service.py` (new or existing)

```python
from datetime import datetime, timedelta
import json

class SessionCache:
    """Manage cached analysis results per session"""

    def __init__(self):
        self.cache = {}  # In-memory cache (or use Redis)

    def store_insights(self, session_id: str, insights: Dict):
        """Store insights with 24-hour expiry"""
        self.cache[session_id] = {
            'insights': insights,
            'timestamp': datetime.now(),
            'expires': datetime.now() + timedelta(hours=24)
        }

    def get_insights(self, session_id: str) -> Dict:
        """Retrieve cached insights if not expired"""
        if session_id not in self.cache:
            return {}

        cache_entry = self.cache[session_id]

        # Check expiry
        if datetime.now() > cache_entry['expires']:
            del self.cache[session_id]
            return {}

        return cache_entry['insights']

    def clear_session(self, session_id: str):
        """Clear cached insights for session"""
        if session_id in self.cache:
            del self.cache[session_id]
```

---

## Performance Comparison

| Mode | Thinking | LLM Calls | Time | Use Case |
|------|----------|-----------|------|----------|
| **Insights** | ✅ think=true | 2 calls (thinking + response) | ~50s per agent | Generate comprehensive analysis |
| **Chat** | ❌ think=false | 1 call (response only) | ~5-10s | Quick conversational questions |

---

## Data Flow Diagram

### Insights Mode:
```
User clicks "Generate Insights"
    ↓
Orchestrator.generate_insights()
    ↓
For each of 6 agents:
    ├─ Gemma Router (0.5s)
    ├─ Qwen Thinking (20s, think=true) ← Deep reasoning
    ├─ Data Retrieval + Pandas (0.5s)
    └─ Qwen Response (20s, think=true) ← Uses thinking in prompt
    ↓
Cache all 6 results in session
    ↓
Total: ~300s (5 minutes for all 6 agents)
```

### Chat Mode:
```
User asks: "What were my expenses last month?"
    ↓
Gemma Router: "expense" (0.5s)
    ↓
Retrieve cached ExpenseAgent analysis
    ↓
Optional: Search for specific data (1s)
    ↓
Qwen Generate (5s, think=false) ← Uses cached analysis as context
    ↓
Total: ~6-7s
```

---

## Key Improvements

1. **Include Thinking in Final Prompt** (Insights Mode):
   - Pass internal thinking output to final response generation
   - Provides structured reasoning context for better answers
   - Example: "Your thinking said to focus on Q4, so answer accordingly..."

2. **No Chat Without Cache**:
   - Every chat question assumes cached analysis exists
   - User must run "Generate Insights" first
   - Ensures quality responses with context

3. **Smart Fresh Search** (Chat Mode):
   - Optionally retrieve additional data for specific queries
   - Example: "Show me transactions over 10,000 SAR"
   - Keeps responses accurate without full re-analysis

---

## Files to Modify

### Core Agent Files (6):
1. `/yomnai_backend/agents/expense_agent.py`
2. `/yomnai_backend/agents/income_agent.py`
3. `/yomnai_backend/agents/fee_agent.py`
4. `/yomnai_backend/agents/budget_agent.py`
5. `/yomnai_backend/agents/trend_agent.py`
6. `/yomnai_backend/agents/transaction_agent.py`

### Infrastructure Files:
7. `/yomnai_backend/agents/ollama_client.py` - Add `think` parameter
8. `/yomnai_backend/agents/orchestrator.py` - Add mode support
9. `/yomnai_backend/app/services/session_service.py` - Cache management (new file)
10. `/yomnai_backend/app/api/endpoints/analysis.py` - Insights endpoint (new or update)
11. `/yomnai_backend/app/api/endpoints/chat.py` - Update chat endpoint

### Documentation:
12. `/yomnai_backend/docs/AGENT_EXECUTION_FLOW.md` - Update with dual-mode flow
13. `/yomnai_backend/docs/AGENTS_DOCUMENTATION.md` - Add mode parameter docs

---

## ✅ Implementation Status (January 2025)

### Completed Tasks:
1. ✅ Implemented `think` parameter in Ollama client (only passes `think=False` when disabling)
2. ✅ Updated all 6 transaction agents with dual-mode support + 2-call thinking pattern
3. ✅ Updated all 6 financial agents with dual-mode support + 2-call thinking pattern
4. ✅ Implemented Redis-based session cache (24-hour expiry)
5. ✅ Updated orchestrator with dual-mode logic and transaction pre-retrieval
6. ✅ Created/updated API endpoints for insights and chat
7. ✅ Fixed frontend display (2 cards per row, "Start Analysis" button)
8. ✅ Set max_tokens to 32000 across all agents for full context usage
9. ✅ Implemented cache key mapping and error handling
10. ✅ Added comprehensive logging for insights mode (shows both LLM calls)

### All 12 Agents with Dual-Mode Support:

**Transaction Agents (6) - 2-Call Thinking Pattern:**
- ✅ expense_agent.py - **CALL 1:** `_think_about_expenses()` deep thinking → **CALL 2:** Final response using thinking context
- ✅ income_agent.py - **CALL 1:** `_think_about_income()` deep thinking → **CALL 2:** Final response using thinking context
- ✅ fee_agent.py - **CALL 1:** `_think_about_fees()` deep thinking → **CALL 2:** Final response using thinking context
- ✅ budget_agent.py - **CALL 1:** Hidden thinking via `_think_hidden()` → **CALL 2:** Final response with budget analysis
- ✅ trend_agent.py - **CALL 1:** Hidden thinking via `_think_hidden()` → **CALL 2:** Final response with trend analysis
- ✅ transaction_agent.py - **CALL 1:** Hidden thinking via `_think_hidden()` → **CALL 2:** Final response with search results

**Financial Agents (6) - 2-Call Thinking Pattern:**
- ✅ ratio_analyst_agent.py - **CALL 1:** `_think_hidden()` ratio reasoning → **CALL 2:** Final analysis using thinking
- ✅ profitability_agent.py - **CALL 1:** `_think_hidden()` profitability reasoning → **CALL 2:** Final analysis using thinking
- ✅ liquidity_agent.py - **CALL 1:** `_think_hidden()` liquidity reasoning → **CALL 2:** Final analysis using thinking
- ✅ efficiency_agent.py - **CALL 1:** `_think_hidden()` efficiency reasoning → **CALL 2:** Final analysis using thinking
- ✅ financial_trend_agent.py - **CALL 1:** `_think_hidden()` trend reasoning → **CALL 2:** Final analysis using thinking
- ✅ risk_assessment_agent.py - **CALL 1:** `_think_hidden()` risk reasoning → **CALL 2:** Final analysis using thinking

### 2-Call Thinking Pattern (Insights Mode):

**How It Works:**
```python
# STEP 1: Deep Hidden Thinking (think=True)
thinking = self._think_hidden(
    thinking_prompt="Think deeply about this analysis...",
    system_prompt="You are an Expert...",
    think=True
)
# Returns: String with deep reasoning (10,000+ chars)

# STEP 2: Final Response Using Thinking (think=True)
response = self.ollama.generate(
    prompt=f"""YOUR INTERNAL THINKING:
    {thinking}

    FINANCIAL DATA:
    ...

    Using your thinking above, provide comprehensive analysis.""",
    think=True
)
# Returns: Final analysis informed by thinking (5,000+ chars)
```

**Why 2 Calls?**
- First call focuses purely on structured reasoning and strategy
- Second call generates user-facing analysis using that reasoning as context
- Both calls use `think=True` for maximum quality
- Total time: ~100s per agent (50s thinking + 50s response)

**Chat Mode (1 Call):**
```python
# Uses cached analysis, no thinking needed
response = self.ollama.generate(
    prompt=f"""CACHED ANALYSIS: {cached}
    USER QUESTION: {query}
    Answer concisely.""",
    think=False  # Fast generation
)
# Time: 5-10s
```

### Key Technical Decisions:
1. **Think Parameter**: Only pass `think=False` to Ollama when explicitly disabling (Qwen3 model supports it)
2. **Token Limits**: All agents use 32000 max_tokens for comprehensive responses
3. **2-Call Pattern**: All 12 agents use thinking call + final response call in insights mode
4. **Transaction Retrieval**: Orchestrator retrieves all transactions ONCE in insights mode, passes to agents via thinking_context
5. **Cache Keys**: Mapped agent categories to cache keys (expense→expenses, fee→fees, etc.)
6. **Error Handling**: Chat mode returns error if no cache exists, prompts user to run "Start Analysis"
7. **Financial Agents**: Support dual-mode via `analyze_financial_statement()` method with mode parameter
8. **Logging**: Comprehensive logging shows both LLM calls with 🧠 emoji for insights mode, 🚀 for chat mode
9. **Filtered Transaction Priority (January 2025)**: Chat mode checks for `self._retrieved_transactions` (pre-filtered by base agent) and prioritizes analyzing them over cached full-dataset analysis. Prompt explicitly instructs LLM to use filtered data, not cached context.

---

**✅ Implementation Complete**
**Performance Gain Achieved:** 10x faster chat responses (50s → 5-10s)
**Cache Duration:** 24 hours per session
