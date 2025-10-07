# Filtered Transaction Priority Fix

## Status: ✅ COMPLETED (January 2025)

Applied to all 6 transaction agents to ensure filtered transactions take priority over cached full-dataset analysis.

---

## Problem Statement

When users asked date-filtered questions (e.g., "show me June expenses"), the system would:
1. Query understander correctly extracted filters (`upload_id`, `start_date`, `end_date`)
2. Base agent retrieved filtered transactions and stored in `self._retrieved_transactions`
3. **BUT** agents in chat mode ignored filtered data and used cached full-dataset analysis instead
4. Result: Wrong answers based on full dataset, not the filtered subset

### Example Issue:
- User query: "show me all the expenses in June"
- System retrieved: 11 June transactions correctly
- Agent response: Used cached analysis of 74 total transactions
- Health score wrong: 85/100 (full dataset) instead of correct June-specific score

---

## Root Cause

All 6 transaction agents had this flawed chat mode logic:

```python
# CHAT MODE: Fast response with cached context
elif mode == "chat":
    # Step 2: Optionally search for additional data
    additional_data = None
    if self._needs_fresh_search(query):  # Only triggered by specific keywords
        logger.info(f"Performing fresh search for: {query}")
        all_data = self._gather_data()
        additional_data = self._analyze_xxx(all_data)

    # Step 3: Generate response using cached analysis
    response = self._generate_chat_response(
        query=query,
        cached_analysis=cached_analysis,  # ← Always prioritized
        additional_data=additional_data   # ← Often None
    )
```

**Issues:**
1. Did NOT check for `self._retrieved_transactions` set by base agent
2. Only retrieved fresh data if `_needs_fresh_search()` returned True
3. Prompt always prioritized cached analysis over additional data

---

## Solution Applied

### Fix 1: Check for Pre-Retrieved Filtered Transactions

Added check for `self._retrieved_transactions` BEFORE `_needs_fresh_search()`:

```python
# CHAT MODE: Fast response with cached context
elif mode == "chat":
    logger.info(f"[AGENT] CHAT MODE: Using cached analysis")

    # Step 2: Check if we have pre-retrieved filtered transactions (NEW FIX)
    additional_data = None
    if hasattr(self, '_retrieved_transactions') and self._retrieved_transactions:
        # We have filtered transactions from base agent - analyze them directly
        logger.info(f"[AGENT] Analyzing {len(self._retrieved_transactions)} filtered transactions")
        additional_data = self._analyze_xxx(self._retrieved_transactions)
    elif self._needs_fresh_search(query):
        # No pre-retrieved data, but query needs specific search
        logger.info(f"[AGENT] Performing fresh search for: {query}")
        all_data = self._gather_data()
        additional_data = self._analyze_xxx(all_data)
```

**Key Change:** Check `self._retrieved_transactions` FIRST, then fallback to `_needs_fresh_search()`

---

### Fix 2: Prioritize Filtered Data in Prompt

Updated `_generate_chat_response()` to explicitly prioritize filtered data:

```python
def _generate_chat_response(self, query: str, cached_analysis: Dict,
                            additional_data: Dict = None) -> str:
    """Generate fast chat response using cached context"""

    if additional_data:
        # FILTERED DATA PRIORITY (NEW FIX)
        prompt = f"""Analyze these FILTERED transactions for the user's specific query:

FILTERED DATA (for this specific query):
{self._format_filtered_data(additional_data)}

FILTERED STATISTICS:
- Total: {additional_data.get('total', 0)} SAR
- Transaction Count: {additional_data.get('count', 0)}

BACKGROUND CONTEXT (full dataset analysis):
The complete dataset has {cached_analysis.get('statistics', {}).get('count', 0)} transactions.

USER QUESTION: {query}

Answer using the FILTERED DATA above (not the background context).
Be concise and conversational. Use SAR for all amounts."""
    else:
        # Use cached analysis
        prompt = f"""You previously analyzed this data comprehensively:

CACHED ANALYSIS:
{cached_analysis.get('final_answer', 'No cached analysis available')}

USER QUESTION: {query}

Answer the user's question using the cached analysis as context.
Be concise and conversational. Use SAR for all amounts."""

    response = self.ollama.generate(
        prompt=prompt,
        system_prompt=self._get_system_prompt(),
        think=False,  # Fast generation without deep reasoning
        temperature=0.2,
        max_tokens=32000
    )

    return response
```

**Key Changes:**
1. Conditional prompt: If filtered data exists, use filtered data section first
2. Explicit instruction: "Answer using the FILTERED DATA above (not the background context)"
3. Background context provided but de-emphasized

---

## Files Modified

### 1. ExpenseAgent
**File:** `/yomnai_backend/agents/expense_agent.py`
- Lines 111-128: Added `self._retrieved_transactions` check
- Lines 162-197: Updated prompt to prioritize filtered data

### 2. IncomeAgent
**File:** `/yomnai_backend/agents/income_agent.py`
- Lines 104-113: Added `self._retrieved_transactions` check
- Lines 147-172: Updated prompt to prioritize filtered data

### 3. FeeHunterAgent
**File:** `/yomnai_backend/agents/fee_agent.py`
- Lines 119-128: Added `self._retrieved_transactions` check
- Lines 162-187: Updated prompt to prioritize filtered data

### 4. BudgetAdvisorAgent
**File:** `/yomnai_backend/agents/budget_agent.py`
- Lines 107-120: Added `self._retrieved_transactions` check
- Lines 136-157: Updated prompt to prioritize filtered data

### 5. TrendAnalystAgent
**File:** `/yomnai_backend/agents/trend_agent.py`
- Lines 92-100: Added `self._retrieved_transactions` check
- Lines 123-152: Updated prompt to prioritize filtered data

### 6. TransactionInvestigatorAgent
**File:** `/yomnai_backend/agents/transaction_agent.py`
- Lines 113-124: Added `self._retrieved_transactions` check
- Lines 149-175: Updated prompt to prioritize filtered data

---

## Related Fixes in Same Session

### Fix A: Analysis Results Transformation
**File:** `/yomnai_backend/app/api/endpoints/analysis.py` (lines 214-236)
- Problem: Frontend expected `{status, summary, findings}` but got `{final_answer, sources, statistics}`
- Solution: Added transformation logic to strip `<think>` tags and convert format

### Fix B: Upload ID Filtering in Vector Store
**File:** `/yomnai_backend/agents/vector_store.py` (lines 462-465)
- Problem: `_build_where_clause()` didn't handle `upload_id` filter for workspace isolation
- Solution: Added upload_id condition to ChromaDB where clause

### Fix C: Upload ID Prefix Preservation
**File:** `/yomnai_backend/agents/orchestrator.py` (line 384)
- Problem: Insights mode stripped `upload_` prefix, breaking filters
- Solution: Changed to `f"upload_{session_id.split('_upload_')[1]}"`

### Fix D: NoneType Error in Query Understander
**File:** `/yomnai_backend/agents/query_understander.py` (line 305)
- Problem: `filters.get('amounts', {})` returned None when amounts existed with None value
- Solution: Changed to `filters.get('amounts') or {}`

### Fix E: Router Model Configuration
**File:** `/yomnai_backend/agents/router.py` (line 20)
- Problem: Router was using gemma3 instead of phi3
- Solution: Changed router model to `phi3:latest`

---

## Testing Verification

### Before Fix:
```
User: "show me all the expenses in June"
[Query Understander] Extracted filters: {upload_id: 'upload_1a4a4ac0869d', start_date: '2024-06-01', end_date: '2024-06-30'}
[VectorStore] Retrieved 11 transactions
[ExpenseAgent] CHAT MODE: Using cached analysis
[ExpenseAgent] Performing fresh search for: show me all the expenses in June
[ExpenseAgent] Retrieved 100 transactions (WRONG - ignored pre-filtered 11)
Response: Based on full dataset with health score 85/100
```

### After Fix:
```
User: "show me all the expenses in June"
[Query Understander] Extracted filters: {upload_id: 'upload_1a4a4ac0869d', start_date: '2024-06-01', end_date: '2024-06-30'}
[VectorStore] Retrieved 11 transactions
[ExpenseAgent] CHAT MODE: Using cached analysis
[ExpenseAgent] Analyzing 11 filtered transactions (CORRECT)
Response: Based on 11 June transactions with correct June-specific metrics
```

---

## Performance Impact

- **No performance degradation**: Still uses cached analysis as fallback
- **Same response time**: ~5-10s in chat mode (filtered analysis is fast)
- **Better accuracy**: Answers now reflect the actual filtered subset

---

## Key Learnings

1. **Base agent responsibility**: Base agent retrieves and stores filtered transactions in `self._retrieved_transactions`
2. **Agent responsibility**: Each agent must check for pre-retrieved data before gathering fresh data
3. **Prompt clarity**: Explicitly instruct LLM which data to prioritize (filtered vs cached)
4. **Conditional prompts**: Use different prompt structures based on data availability

---

## Future Improvements

1. **Standardize pattern**: Consider moving filtered transaction check to base agent's `execute()` method
2. **Validation**: Add assertion that filtered transaction count matches expected filter criteria
3. **Logging**: Add more detailed logging of which data source was used (filtered vs cached vs fresh search)

---

**✅ All 6 transaction agents now correctly prioritize filtered transactions over cached analysis**
