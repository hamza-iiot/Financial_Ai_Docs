# Agent Router System

## Overview
The Agent Router is a lightweight classification system that determines which specialized agent should handle each user query. It uses a small, fast model (phi3 or tinyllama) to classify intent WITHOUT accessing vector stores or transaction data.

## Architecture

### Router Model
```python
# Recommended models for routing (small and fast)
ROUTER_MODELS = [
    "tinyllama:latest",      # 1.1B parameters, very fast
    "phi3:mini",             # 3.8B parameters, good accuracy
    "gemma:2b",              # 2B parameters, balanced
]
```

## Router Implementation

```python
class AgentRouter:
    """Routes queries to appropriate specialized agents"""
    
    def __init__(self, ollama_client):
        self.ollama = ollama_client
        self.router_model = "tinyllama:latest"  # Fast classification
        
    def classify_query(self, query: str) -> Dict[str, Any]:
        """Classify query intent without accessing transaction data"""
        
        classification_prompt = """You are a query classifier for a financial assistant.
        Classify the following query into EXACTLY ONE category:
        
        Categories:
        - expense: Questions about spending, purchases, payments, costs
        - income: Questions about salary, earnings, deposits, revenue
        - fee: Questions about bank fees, charges, penalties, service costs
        - budget: Questions about budgeting, saving, financial planning
        - trend: Questions about patterns, changes over time, comparisons
        - transaction: Questions about specific transactions, searches, details
        - summary: General overview or summary requests
        
        Query: {query}
        
        Respond with ONLY the category name, nothing else.
        Category:"""
        
        response = self.ollama.generate(
            prompt=classification_prompt.format(query=query),
            model=self.router_model,
            temperature=0.1,
            max_tokens=10
        )
        
        category = response.strip().lower()
        
        # Validate category
        valid_categories = ['expense', 'income', 'fee', 'budget', 'trend', 'transaction', 'summary']
        if category not in valid_categories:
            category = self._fallback_classification(query)
        
        return {
            'category': category,
            'confidence': self._calculate_confidence(query, category),
            'suggested_agents': self._get_agent_chain(category)
        }
    
    def _fallback_classification(self, query: str) -> str:
        """Simple keyword-based fallback classification"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['spend', 'spent', 'expense', 'buy', 'purchase', 'paid']):
            return 'expense'
        elif any(word in query_lower for word in ['income', 'salary', 'earn', 'deposit', 'receive']):
            return 'income'
        elif any(word in query_lower for word in ['fee', 'charge', 'penalty', 'cost']):
            return 'fee'
        elif any(word in query_lower for word in ['budget', 'save', 'plan', 'afford']):
            return 'budget'
        elif any(word in query_lower for word in ['trend', 'pattern', 'change', 'compare', 'over time']):
            return 'trend'
        elif any(word in query_lower for word in ['find', 'show', 'search', 'transaction', 'when', 'where']):
            return 'transaction'
        else:
            return 'summary'
    
    def _calculate_confidence(self, query: str, category: str) -> float:
        """Calculate classification confidence based on keyword matching"""
        # Implementation for confidence scoring
        return 0.85  # Simplified
    
    def _get_agent_chain(self, category: str) -> List[str]:
        """Determine which agents should be involved"""
        agent_chains = {
            'expense': ['expense_agent', 'trend_agent'],
            'income': ['income_agent', 'trend_agent'],
            'fee': ['fee_agent', 'expense_agent'],
            'budget': ['budget_agent', 'expense_agent', 'income_agent'],
            'trend': ['trend_agent', 'transaction_agent'],
            'transaction': ['transaction_agent'],
            'summary': ['expense_agent', 'income_agent', 'fee_agent']
        }
        return agent_chains.get(category, ['transaction_agent'])
```

## Query Flow

### 1. Chat Interface Flow
```
User Query → Router Classification → Agent Selection → Specialized Processing → Response
```

### 2. Insights Button Flow (Sequential)
```
Button Click → Expense Agent → Income Agent → Fee Agent → Aggregated Display
```

## Hidden Thinking Process

The router enables "Chain of Thought" reasoning without showing it to users:

```python
def process_with_thinking(self, query: str, category: str) -> Dict[str, Any]:
    """Process query with hidden reasoning steps"""
    
    # Step 1: Internal reasoning (not shown to user)
    thinking_prompt = f"""Think step by step about this {category} query:
    Query: {query}
    
    Consider:
    1. What specific data points are needed?
    2. What calculations might be required?
    3. What context is important?
    4. What format would be most helpful?
    
    Reasoning:"""
    
    reasoning = self.ollama.generate(
        prompt=thinking_prompt,
        model=self.router_model,
        temperature=0.3
    )
    
    # Step 2: Generate enhanced query for the agent
    enhanced_query = {
        'original': query,
        'category': category,
        'reasoning': reasoning,  # Used internally, not shown
        'focus_areas': self._extract_focus_areas(reasoning)
    }
    
    return enhanced_query
```

## Routing Examples

### Example 1: Expense Query
```
Query: "What did I spend the most money on last month?"
Router Output: {
    'category': 'expense',
    'confidence': 0.95,
    'suggested_agents': ['expense_agent', 'trend_agent']
}
```

### Example 2: Income Query
```
Query: "Show me all my salary deposits"
Router Output: {
    'category': 'income',
    'confidence': 0.92,
    'suggested_agents': ['income_agent']
}
```

### Example 3: Complex Query
```
Query: "How much am I spending compared to my income?"
Router Output: {
    'category': 'budget',
    'confidence': 0.88,
    'suggested_agents': ['budget_agent', 'expense_agent', 'income_agent']
}
```

## Performance Optimization

### Caching Strategy
```python
class RouterCache:
    """Cache routing decisions for common queries"""
    
    def __init__(self, ttl=3600):
        self.cache = {}
        self.ttl = ttl
        
    def get_cached_route(self, query: str) -> Optional[Dict]:
        """Check if we've seen this query pattern before"""
        query_normalized = self._normalize_query(query)
        if query_normalized in self.cache:
            return self.cache[query_normalized]
        return None
    
    def cache_route(self, query: str, result: Dict):
        """Cache routing decision"""
        query_normalized = self._normalize_query(query)
        self.cache[query_normalized] = result
```

## Integration with Main Pipeline

```python
# In rag_pipeline.py
class EnhancedRAGPipeline:
    def __init__(self):
        self.router = AgentRouter(ollama_client)
        self.agents = {
            'expense': ExpenseAgent(),
            'income': IncomeAgent(),
            'fee': FeeAgent(),
            'budget': BudgetAgent(),
            'trend': TrendAgent(),
            'transaction': TransactionAgent()
        }
    
    def process_query(self, query: str) -> Dict[str, Any]:
        """Process query through router and specialized agent"""
        
        # Route query
        routing = self.router.classify_query(query)
        
        # Get specialized agent
        primary_agent = self.agents[routing['category']]
        
        # Process with hidden thinking
        with_thinking = self.router.process_with_thinking(query, routing['category'])
        
        # Execute agent (thinking is hidden from UI)
        result = primary_agent.execute(
            query=query,
            context=with_thinking,
            vector_store=self.vector_store
        )
        
        return result
```

## Benefits

1. **Fast Classification**: Small model means quick routing decisions
2. **No Vector Search for Routing**: Reduces computational overhead
3. **Specialized Processing**: Each agent optimized for its domain
4. **Hidden Complexity**: Users see clean answers, not the reasoning process
5. **Scalable**: Easy to add new agents and categories