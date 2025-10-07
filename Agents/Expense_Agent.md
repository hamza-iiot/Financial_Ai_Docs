# Expense Agent

## Overview
The Expense Agent is a specialized financial analyst focused on spending patterns, expense categorization, and cost optimization. It uses multiple search queries and hidden reasoning to provide comprehensive expense analysis in Saudi Riyals (SAR).

## Agent Profile

- **Specialization**: Spending analysis, expense patterns, cost breakdown
- **Model**: qwen3-14b-32k:latest
- **Search Depth**: 20-30 transactions per query
- **Query Count**: 5-7 different search angles

## Implementation

```python
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
import pandas as pd

class ExpenseAgent:
    """Specialized agent for expense analysis"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Saudi-specific expense categories
        self.expense_categories = {
            'food': ['restaurant', 'cafe', 'grocery', 'supermarket', 'food', 'lunch', 'dinner'],
            'transport': ['uber', 'careem', 'taxi', 'petrol', 'gas', 'parking', 'stc pay'],
            'utilities': ['electricity', 'water', 'internet', 'phone', 'stc', 'mobily', 'zain'],
            'shopping': ['mall', 'store', 'amazon', 'noon', 'jarir', 'extra', 'ikea'],
            'healthcare': ['pharmacy', 'hospital', 'clinic', 'doctor', 'medicine'],
            'entertainment': ['cinema', 'movie', 'game', 'spotify', 'netflix', 'shahid'],
            'banking': ['fee', 'charge', 'commission', 'atm'],
            'housing': ['rent', 'mortgage', 'maintenance', 'repair'],
            'education': ['school', 'university', 'course', 'training', 'book'],
            'personal': ['salon', 'gym', 'fitness', 'clothing', 'shoes']
        }
    
    def execute_with_data(self, query: str, thinking_context: Dict = None,
                         transactions: List[Dict] = None) -> Dict[str, Any]:
        """Execute expense analysis with pre-retrieved data"""
        
        # Step 1: Hidden thinking process
        internal_analysis = self._think_about_expenses(query, thinking_context)
        
        # Step 2: Work with pre-retrieved transactions
        # No need to search - QueryUnderstander already retrieved relevant data
        
        # Step 3: Deep analysis with categorization (hidden from user)
        expense_analysis = self._analyze_expenses(transactions, internal_analysis)
        
        # Step 4: Generate user-friendly response
        final_response = self._format_expense_response(expense_analysis, query)
        
        return {
            'final_answer': final_response,
            'sources': transactions[:5] if transactions else [],
            'statistics': expense_analysis['stats'],
            'thinking': internal_analysis  # Hidden from UI
        }
    
    def _think_about_expenses(self, query: str, context: Dict = None) -> Dict:
        """Hidden chain-of-thought reasoning about expenses"""
        
        thinking_prompt = f"""Think deeply about this expense-related query in the context of Saudi Arabian transactions.
        
        Query: {query}
        
        Consider these aspects (your thinking will be hidden from the user):
        1. Time period: Is the user asking about a specific period?
        2. Categories: Which expense categories are relevant?
        3. Amount ranges: Are they interested in large or small expenses?
        4. Patterns: Are they looking for recurring vs one-time expenses?
        5. Comparisons: Do they want to compare periods or categories?
        6. Saudi context: Consider local payment methods (SADAD, STC Pay, mada)
        
        Provide detailed reasoning:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are an expert financial analyst specializing in Saudi Arabian banking and expenses. Think step-by-step.",
            temperature=0.3,
            max_tokens=500
        )
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'query_type': self._classify_expense_query(query),
            'context': context
        }
    
    def _analyze_expenses(self, transactions: List[Dict], thinking: Dict) -> Dict:
        """Deep analysis of expense transactions"""
        
        if not transactions:
            return {
                'total': 0,
                'categories': {},
                'top_expenses': [],
                'patterns': {},
                'stats': {}
            }
        
        # Convert to DataFrame for analysis
        df = pd.DataFrame(transactions)
        
        # Categorize expenses
        categorized = self._categorize_expenses(df)
        
        # Calculate statistics
        stats = {
            'total_expenses': float(df['amount'].sum()),
            'average_expense': float(df['amount'].mean()),
            'median_expense': float(df['amount'].median()),
            'largest_expense': float(df['amount'].max()),
            'smallest_expense': float(df['amount'].min()),
            'transaction_count': len(df),
            'daily_average': float(df['amount'].sum() / 30)  # Approximate
        }
        
        # Identify patterns
        patterns = self._identify_patterns(df)
        
        # Get top expenses
        top_expenses = df.nlargest(10, 'amount').to_dict('records')
        
        return {
            'total': stats['total_expenses'],
            'categories': categorized,
            'top_expenses': top_expenses,
            'patterns': patterns,
            'stats': stats
        }
    
    def _categorize_expenses(self, df: pd.DataFrame) -> Dict:
        """Categorize expenses based on descriptions"""
        
        categories = {cat: [] for cat in self.expense_categories.keys()}
        categories['other'] = []
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            categorized = False
            
            for category, keywords in self.expense_categories.items():
                if any(keyword in desc_lower for keyword in keywords):
                    categories[category].append(row.to_dict())
                    categorized = True
                    break
            
            if not categorized:
                categories['other'].append(row.to_dict())
        
        # Calculate totals per category
        category_totals = {}
        for cat, trans_list in categories.items():
            if trans_list:
                total = sum(t.get('amount', 0) for t in trans_list)
                category_totals[cat] = {
                    'total': total,
                    'count': len(trans_list),
                    'average': total / len(trans_list)
                }
        
        return category_totals
    
    def _identify_patterns(self, df: pd.DataFrame) -> Dict:
        """Identify spending patterns"""
        
        patterns = {}
        
        # Day of week analysis
        if 'date' in df.columns:
            df['date'] = pd.to_datetime(df['date'])
            df['day_of_week'] = df['date'].dt.day_name()
            patterns['by_day'] = df.groupby('day_of_week')['amount'].sum().to_dict()
        
        # Recurring transactions (same amount multiple times)
        amount_counts = df['amount'].value_counts()
        patterns['recurring'] = [
            {'amount': amount, 'count': count}
            for amount, count in amount_counts.items()
            if count > 1
        ][:5]
        
        return patterns
    
    def _format_expense_response(self, analysis: Dict) -> str:
        """Format analysis into user-friendly response"""
        
        prompt = f"""Based on this expense analysis, provide a clear and helpful response in the context of Saudi Arabian banking.

        Analysis Data:
        - Total Expenses: SAR {analysis['total']:,.2f}
        - Number of Transactions: {analysis['stats'].get('transaction_count', 0)}
        - Average Expense: SAR {analysis['stats'].get('average_expense', 0):,.2f}
        - Largest Expense: SAR {analysis['stats'].get('largest_expense', 0):,.2f}
        
        Category Breakdown:
        {self._format_categories(analysis['categories'])}
        
        Top 5 Expenses:
        {self._format_top_expenses(analysis['top_expenses'][:5])}
        
        Provide insights in Saudi Riyals (SAR) with practical recommendations.
        Be specific and actionable. Format amounts as "SAR X,XXX.XX".
        """
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are Yomnai, a financial analyst specializing in Saudi Arabian personal finance.
            Always use SAR for currency. Be concise but thorough. Provide actionable insights.""",
            temperature=0.2,
            max_tokens=500
        )
        
        return response
    
    def _format_categories(self, categories: Dict) -> str:
        """Format category data for prompt"""
        if not categories:
            return "No categories identified"
        
        lines = []
        for cat, data in sorted(categories.items(), key=lambda x: x[1].get('total', 0), reverse=True)[:5]:
            if data:
                lines.append(f"- {cat.title()}: SAR {data['total']:,.2f} ({data['count']} transactions)")
        
        return '\n'.join(lines)
    
    def _format_top_expenses(self, expenses: List[Dict]) -> str:
        """Format top expenses for prompt"""
        if not expenses:
            return "No expenses found"
        
        lines = []
        for i, exp in enumerate(expenses, 1):
            lines.append(f"{i}. SAR {exp.get('amount', 0):,.2f} - {exp.get('description', 'N/A')} ({exp.get('date', 'N/A')})")
        
        return '\n'.join(lines)
    
    def analyze_for_insights(self) -> Dict[str, Any]:
        """Special method for insights dashboard - comprehensive analysis"""
        
        # Hidden thinking for comprehensive analysis
        thinking = self._think_about_expenses(
            "Provide comprehensive expense analysis",
            context={'mode': 'insights', 'comprehensive': True}
        )
        
        # Generate comprehensive queries
        queries = [
            {'query': 'expense payment purchase debit', 'n_results': 50, 'filters': {'type': 'debit'}},
            {'query': 'large significant big expensive', 'n_results': 30, 'filters': {'type': 'debit', 'amount': {'$gte': 1000}}},
            {'query': 'recurring monthly subscription regular', 'n_results': 30, 'filters': {'type': 'debit'}},
            {'query': 'food restaurant grocery supermarket', 'n_results': 30, 'filters': {'type': 'debit'}},
            {'query': 'transport uber careem taxi petrol', 'n_results': 30, 'filters': {'type': 'debit'}},
            {'query': 'shopping mall store online', 'n_results': 30, 'filters': {'type': 'debit'}},
            {'query': 'utilities electricity water internet phone', 'n_results': 20, 'filters': {'type': 'debit'}}
        ]
        
        # Gather all data
        all_data = self._gather_expense_data(queries)
        
        # Comprehensive analysis
        analysis = self._analyze_expenses(all_data, thinking)
        
        # Generate detailed summary
        summary_prompt = f"""Generate a comprehensive expense analysis summary for Saudi Arabian transactions:

        Total Expenses: SAR {analysis['total']:,.2f}
        Transaction Count: {analysis['stats'].get('transaction_count', 0)}
        Daily Average: SAR {analysis['stats'].get('daily_average', 0):,.2f}
        
        Categories: {self._format_categories(analysis['categories'])}
        
        Provide:
        1. Key findings (2-3 points)
        2. Largest expense category
        3. Potential savings opportunities
        4. Unusual patterns or concerns
        
        Use SAR for all amounts. Be specific and actionable."""
        
        summary = self.ollama.generate(
            prompt=summary_prompt,
            system_prompt="You are a Saudi financial advisor. Provide clear, actionable insights in SAR.",
            temperature=0.2,
            max_tokens=400
        )
        
        return {
            'summary': summary,
            'data': analysis,
            'query_count': len(queries),
            'transactions_analyzed': len(all_data)
        }
```

## Prompts

### System Prompt
```python
EXPENSE_SYSTEM_PROMPT = """You are an expert expense analyst for Saudi Arabian personal finance.
You specialize in:
- Identifying spending patterns and trends
- Categorizing expenses accurately
- Finding cost-saving opportunities
- Understanding Saudi payment methods (mada, SADAD, STC Pay)
- Recognizing local merchant names and services

Always:
- Use Saudi Riyals (SAR) for all amounts
- Format currency as "SAR X,XXX.XX"
- Consider local context (Friday-Saturday weekend, Ramadan patterns)
- Provide actionable recommendations
- Be specific with amounts and dates
"""
```

### Analysis Prompts
```python
EXPENSE_ANALYSIS_PROMPTS = {
    'general': """Analyze these expense transactions and provide insights about spending patterns.
    Focus on: total spending, major categories, unusual expenses, and savings opportunities.""",
    
    'category': """Categorize these expenses into standard spending categories.
    Include: Food, Transport, Utilities, Shopping, Entertainment, Healthcare, Banking, Housing, Personal.""",
    
    'trends': """Identify spending trends and patterns in these transactions.
    Look for: recurring expenses, spending spikes, day-of-week patterns, increasing/decreasing trends.""",
    
    'optimization': """Suggest ways to optimize spending based on these transactions.
    Consider: unnecessary fees, duplicate services, cheaper alternatives, bulk savings opportunities.""",
    
    'comparison': """Compare spending across different time periods or categories.
    Highlight: changes in spending, new expense types, eliminated expenses, seasonal variations."""
}
```

## Query Examples

### User Query → Agent Response

**Query**: "What am I spending the most money on?"
```
Hidden Thinking:
- User wants category breakdown
- Need to aggregate by type
- Should highlight top 3-5 categories
- Include specific examples

Response:
Your top spending categories are:

1. **Shopping** - SAR 4,350.00 (35% of total)
   - Noon.com: SAR 1,200.00
   - Extra Stores: SAR 890.00
   
2. **Food & Dining** - SAR 2,840.00 (23% of total)
   - Grocery stores: SAR 1,560.00
   - Restaurants: SAR 1,280.00

3. **Transport** - SAR 1,950.00 (16% of total)
   - Petrol: SAR 1,200.00
   - Uber/Careem: SAR 750.00

Consider setting monthly budgets for shopping and dining to reduce expenses.
```

**Query**: "Show me all my subscriptions"
```
Hidden Thinking:
- Look for recurring same amounts
- Search for subscription keywords
- Group by service type
- Calculate monthly/yearly cost

Response:
Found 6 recurring subscriptions totaling SAR 425.50/month:

**Entertainment** (SAR 165.00/month):
- Netflix: SAR 65.00
- Spotify: SAR 45.00
- Shahid: SAR 55.00

**Utilities** (SAR 260.50/month):
- STC Internet: SAR 199.00
- Mobily Mobile: SAR 61.50

Annual subscription cost: SAR 5,106.00
Consider reviewing if all subscriptions are actively used.
```

## Integration with Insights Dashboard

```python
# When insights button is clicked
def generate_expense_insights():
    """Generate comprehensive expense insights"""
    
    # This runs hidden thinking and multiple queries
    expense_analysis = expense_agent.analyze_for_insights()
    
    # User only sees this clean summary
    return {
        'title': '💸 Expense Analysis',
        'summary': expense_analysis['summary'],
        'stats': {
            'Total Spent': f"SAR {expense_analysis['data']['total']:,.2f}",
            'Daily Average': f"SAR {expense_analysis['data']['stats']['daily_average']:,.2f}",
            'Transactions': expense_analysis['data']['stats']['transaction_count']
        }
    }
```