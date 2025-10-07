# Income Agent

## Overview
The Income Agent specializes in analyzing income sources, salary patterns, and revenue streams. It understands Saudi employment patterns, identifies regular vs irregular income, and provides insights on income stability and growth.

## Agent Profile

- **Specialization**: Income analysis, salary tracking, deposit identification
- **Model**: qwen3-14b-32k:latest
- **Search Depth**: 15-20 transactions per query
- **Query Count**: 4-5 different search angles

## Implementation

```python
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

class IncomeAgent:
    """Specialized agent for income analysis"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Saudi-specific income keywords
        self.income_keywords = {
            'salary': ['salary', 'راتب', 'payroll', 'wages', 'monthly salary'],
            'bonus': ['bonus', 'مكافأة', 'incentive', 'reward', 'performance'],
            'allowance': ['allowance', 'بدل', 'housing', 'transport', 'cola'],
            'transfer': ['transfer', 'تحويل', 'deposit', 'credit', 'received'],
            'business': ['invoice', 'payment', 'revenue', 'sales', 'income'],
            'investment': ['dividend', 'return', 'profit', 'investment', 'trading'],
            'government': ['hafiz', 'حافز', 'support', 'subsidy', 'grant'],
            'freelance': ['project', 'freelance', 'contract', 'consulting', 'service']
        }
        
        # Saudi salary payment patterns
        self.salary_patterns = {
            'monthly': {'days': [25, 26, 27, 28, 29, 30, 31, 1, 2, 3]},
            'biweekly': {'frequency': 14},
            'weekly': {'frequency': 7}
        }
    
    def execute(self, query: str, thinking_context: Dict = None) -> Dict[str, Any]:
        """Execute income analysis with hidden thinking"""
        
        # Step 1: Hidden thinking process
        internal_analysis = self._think_about_income(query, thinking_context)
        
        # Step 2: Generate specialized income queries
        search_queries = self._generate_income_queries(internal_analysis)
        
        # Step 3: Gather income data
        all_income = self._gather_income_data(search_queries)
        
        # Step 4: Deep income analysis
        income_analysis = self._analyze_income(all_income, internal_analysis)
        
        # Step 5: Generate user response
        final_response = self._format_income_response(income_analysis)
        
        return {
            'final_answer': final_response,
            'sources': income_analysis['primary_income'][:5],
            'statistics': income_analysis['stats'],
            'thinking': internal_analysis  # Hidden from UI
        }
    
    def _think_about_income(self, query: str, context: Dict = None) -> Dict:
        """Hidden reasoning about income patterns"""
        
        thinking_prompt = f"""Analyze this income-related query for Saudi Arabian banking context.
        
        Query: {query}
        
        Consider (your analysis will be hidden):
        1. Income types: Is this about salary, bonuses, or other income?
        2. Regularity: Are they asking about regular or irregular income?
        3. Time period: Specific month, quarter, or general pattern?
        4. Saudi context: Consider end-of-month salary patterns (25th-30th)
        5. Income sources: Single employer or multiple sources?
        6. Currency: All amounts should be in SAR
        
        Detailed reasoning:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are an income analyst expert for Saudi Arabian personal finance. Think systematically.",
            temperature=0.3,
            max_tokens=500
        )
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'query_type': self._classify_income_query(query),
            'context': context
        }
    
    def _generate_income_queries(self, thinking: Dict) -> List[Dict]:
        """Generate multiple search queries for income"""
        
        queries = []
        
        # Query 1: General income/credit search
        queries.append({
            'query': 'salary income credit deposit transfer received',
            'n_results': 20,
            'filters': {'type': 'credit'}
        })
        
        # Query 2: Salary-specific search
        queries.append({
            'query': 'salary payroll wages monthly راتب',
            'n_results': 15,
            'filters': {'type': 'credit', 'amount': {'$gte': 1000}}
        })
        
        # Query 3: Bonus and allowances
        queries.append({
            'query': 'bonus allowance incentive بدل مكافأة extra',
            'n_results': 10,
            'filters': {'type': 'credit'}
        })
        
        # Query 4: Transfers and deposits
        queries.append({
            'query': 'transfer deposit credit تحويل received from',
            'n_results': 15,
            'filters': {'type': 'credit'}
        })
        
        # Query 5: Large deposits (potential income)
        queries.append({
            'query': 'large significant deposit credit',
            'n_results': 10,
            'filters': {'type': 'credit', 'amount': {'$gte': 5000}}
        })
        
        return queries
    
    def _gather_income_data(self, queries: List[Dict]) -> List[Dict]:
        """Execute searches and aggregate income transactions"""
        
        all_income = []
        seen_ids = set()
        
        for query_config in queries:
            try:
                results = self.vector_store.search(
                    query=query_config['query'],
                    n_results=query_config['n_results'],
                    filter_dict=query_config.get('filters')
                )
                
                for trans in results.get('transactions', []):
                    trans_id = f"{trans.get('date')}_{trans.get('amount')}_{trans.get('description')}"
                    if trans_id not in seen_ids:
                        seen_ids.add(trans_id)
                        all_income.append(trans)
                        
            except Exception as e:
                continue
        
        return all_income
    
    def _analyze_income(self, transactions: List[Dict], thinking: Dict) -> Dict:
        """Comprehensive income analysis"""
        
        if not transactions:
            return {
                'total_income': 0,
                'primary_income': [],
                'income_sources': {},
                'patterns': {},
                'stats': {}
            }
        
        df = pd.DataFrame(transactions)
        df['date'] = pd.to_datetime(df['date'])
        
        # Identify income sources
        income_sources = self._identify_income_sources(df)
        
        # Detect salary pattern
        salary_info = self._detect_salary_pattern(df)
        
        # Calculate statistics
        stats = {
            'total_income': float(df['amount'].sum()),
            'average_income': float(df['amount'].mean()),
            'median_income': float(df['amount'].median()),
            'largest_deposit': float(df['amount'].max()),
            'income_count': len(df),
            'monthly_average': self._calculate_monthly_average(df),
            'income_stability': self._calculate_stability_score(df)
        }
        
        # Identify patterns
        patterns = {
            'salary': salary_info,
            'frequency': self._analyze_frequency(df),
            'growth_trend': self._analyze_growth_trend(df),
            'irregular_income': self._find_irregular_income(df)
        }
        
        # Get primary income transactions
        primary_income = df.nlargest(10, 'amount').to_dict('records')
        
        return {
            'total_income': stats['total_income'],
            'primary_income': primary_income,
            'income_sources': income_sources,
            'patterns': patterns,
            'stats': stats
        }
    
    def _identify_income_sources(self, df: pd.DataFrame) -> Dict:
        """Categorize income by source"""
        
        sources = {
            'salary': [],
            'bonus': [],
            'transfer': [],
            'business': [],
            'investment': [],
            'government': [],
            'other': []
        }
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            categorized = False
            
            # Check each income category
            for source, keywords in self.income_keywords.items():
                if any(keyword in desc_lower for keyword in keywords):
                    sources[source].append(row.to_dict())
                    categorized = True
                    break
            
            if not categorized:
                sources['other'].append(row.to_dict())
        
        # Calculate totals
        source_totals = {}
        for source, trans_list in sources.items():
            if trans_list:
                total = sum(t.get('amount', 0) for t in trans_list)
                source_totals[source] = {
                    'total': total,
                    'count': len(trans_list),
                    'average': total / len(trans_list),
                    'percentage': (total / df['amount'].sum()) * 100
                }
        
        return source_totals
    
    def _detect_salary_pattern(self, df: pd.DataFrame) -> Dict:
        """Detect regular salary payments"""
        
        # Look for regular large deposits
        df_sorted = df.sort_values('date')
        
        # Find potential salary (regular amount, monthly frequency)
        amount_counts = df['amount'].value_counts()
        
        potential_salaries = []
        for amount, count in amount_counts.items():
            if count >= 2 and amount >= 3000:  # Minimum salary threshold
                # Check if dates are roughly monthly
                salary_trans = df[df['amount'] == amount].sort_values('date')
                dates = pd.to_datetime(salary_trans['date'])
                
                if len(dates) > 1:
                    days_between = [(dates.iloc[i+1] - dates.iloc[i]).days 
                                   for i in range(len(dates)-1)]
                    avg_days = np.mean(days_between)
                    
                    if 25 <= avg_days <= 35:  # Monthly pattern
                        potential_salaries.append({
                            'amount': float(amount),
                            'frequency': 'monthly',
                            'occurrences': count,
                            'average_interval': avg_days,
                            'last_date': dates.iloc[-1].strftime('%Y-%m-%d')
                        })
        
        # Return the most likely salary (highest amount with regular pattern)
        if potential_salaries:
            return max(potential_salaries, key=lambda x: x['amount'])
        
        return {}
    
    def _calculate_monthly_average(self, df: pd.DataFrame) -> float:
        """Calculate average monthly income"""
        
        if df.empty:
            return 0.0
        
        df['month'] = df['date'].dt.to_period('M')
        monthly_income = df.groupby('month')['amount'].sum()
        
        return float(monthly_income.mean())
    
    def _calculate_stability_score(self, df: pd.DataFrame) -> float:
        """Calculate income stability score (0-100)"""
        
        if len(df) < 2:
            return 0.0
        
        # Factors: regularity, consistency, frequency
        df['month'] = df['date'].dt.to_period('M')
        monthly_income = df.groupby('month')['amount'].sum()
        
        if len(monthly_income) < 2:
            return 50.0
        
        # Calculate coefficient of variation (lower is more stable)
        cv = (monthly_income.std() / monthly_income.mean()) if monthly_income.mean() > 0 else 1
        
        # Convert to 0-100 score (lower CV = higher score)
        stability = max(0, min(100, 100 * (1 - cv)))
        
        return float(stability)
    
    def _analyze_frequency(self, df: pd.DataFrame) -> Dict:
        """Analyze income frequency patterns"""
        
        df_sorted = df.sort_values('date')
        
        if len(df_sorted) < 2:
            return {'pattern': 'insufficient_data'}
        
        # Calculate days between income
        dates = pd.to_datetime(df_sorted['date'])
        intervals = [(dates.iloc[i+1] - dates.iloc[i]).days 
                    for i in range(len(dates)-1)]
        
        if not intervals:
            return {'pattern': 'single_income'}
        
        avg_interval = np.mean(intervals)
        
        if avg_interval <= 7:
            pattern = 'weekly'
        elif avg_interval <= 15:
            pattern = 'biweekly'
        elif avg_interval <= 35:
            pattern = 'monthly'
        elif avg_interval <= 95:
            pattern = 'quarterly'
        else:
            pattern = 'irregular'
        
        return {
            'pattern': pattern,
            'average_interval_days': float(avg_interval),
            'min_interval': min(intervals),
            'max_interval': max(intervals)
        }
    
    def _analyze_growth_trend(self, df: pd.DataFrame) -> Dict:
        """Analyze income growth trend"""
        
        df['month'] = df['date'].dt.to_period('M')
        monthly_income = df.groupby('month')['amount'].sum().sort_index()
        
        if len(monthly_income) < 2:
            return {'trend': 'insufficient_data'}
        
        # Simple linear trend
        x = np.arange(len(monthly_income))
        y = monthly_income.values
        
        if len(x) > 1:
            slope = np.polyfit(x, y, 1)[0]
            
            if slope > 100:
                trend = 'increasing'
            elif slope < -100:
                trend = 'decreasing'
            else:
                trend = 'stable'
            
            return {
                'trend': trend,
                'monthly_change': float(slope),
                'percentage_change': float((slope / y[0]) * 100) if y[0] > 0 else 0
            }
        
        return {'trend': 'stable'}
    
    def _find_irregular_income(self, df: pd.DataFrame) -> List[Dict]:
        """Identify irregular/unexpected income"""
        
        # Find income that doesn't match regular patterns
        median_amount = df['amount'].median()
        std_amount = df['amount'].std()
        
        # Irregular = more than 2 std deviations from median
        threshold = median_amount + (2 * std_amount)
        
        irregular = df[df['amount'] > threshold].nlargest(5, 'amount')
        
        return irregular.to_dict('records') if not irregular.empty else []
    
    def _format_income_response(self, analysis: Dict) -> str:
        """Format analysis into user-friendly response"""
        
        prompt = f"""Based on this income analysis, provide clear insights for Saudi Arabian context.

        Income Analysis:
        - Total Income: SAR {analysis['stats'].get('total_income', 0):,.2f}
        - Monthly Average: SAR {analysis['stats'].get('monthly_average', 0):,.2f}
        - Income Stability: {analysis['stats'].get('income_stability', 0):.0f}%
        - Number of Deposits: {analysis['stats'].get('income_count', 0)}
        
        Income Sources:
        {self._format_income_sources(analysis['income_sources'])}
        
        Salary Information:
        {self._format_salary_info(analysis['patterns'].get('salary', {}))}
        
        Provide insights about:
        1. Primary income source and stability
        2. Income frequency and patterns
        3. Any irregular or bonus income
        4. Recommendations for income management
        
        Use Saudi Riyals (SAR) for all amounts.
        """
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are Yomnai, specializing in Saudi Arabian income analysis.
            Focus on salary patterns, income stability, and growth opportunities.
            Always use SAR for currency. Be specific and actionable.""",
            temperature=0.2,
            max_tokens=500
        )
        
        return response
    
    def _format_income_sources(self, sources: Dict) -> str:
        """Format income sources for prompt"""
        if not sources:
            return "No income sources identified"
        
        lines = []
        for source, data in sorted(sources.items(), 
                                  key=lambda x: x[1].get('total', 0), 
                                  reverse=True)[:4]:
            if data:
                lines.append(
                    f"- {source.title()}: SAR {data['total']:,.2f} "
                    f"({data['percentage']:.1f}% of total, "
                    f"{data['count']} transactions)"
                )
        
        return '\n'.join(lines)
    
    def _format_salary_info(self, salary: Dict) -> str:
        """Format salary information"""
        if not salary:
            return "No regular salary pattern detected"
        
        return f"""Regular Salary Detected:
        - Amount: SAR {salary.get('amount', 0):,.2f}
        - Frequency: {salary.get('frequency', 'Unknown')}
        - Last Received: {salary.get('last_date', 'Unknown')}
        - Occurrences: {salary.get('occurrences', 0)}"""
    
    def _classify_income_query(self, query: str) -> str:
        """Classify the type of income query"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['salary', 'payroll', 'wages']):
            return 'salary'
        elif any(word in query_lower for word in ['bonus', 'incentive', 'extra']):
            return 'bonus'
        elif any(word in query_lower for word in ['all', 'total', 'income']):
            return 'general'
        elif any(word in query_lower for word in ['source', 'where', 'from']):
            return 'sources'
        else:
            return 'general'
    
    def analyze_for_insights(self) -> Dict[str, Any]:
        """Comprehensive income analysis for insights dashboard"""
        
        # Hidden comprehensive thinking
        thinking = self._think_about_income(
            "Provide comprehensive income analysis including salary, bonuses, and all deposits",
            context={'mode': 'insights', 'comprehensive': True}
        )
        
        # Comprehensive income queries
        queries = [
            {'query': 'salary income credit deposit', 'n_results': 40, 'filters': {'type': 'credit'}},
            {'query': 'salary payroll wages monthly', 'n_results': 20, 'filters': {'type': 'credit', 'amount': {'$gte': 3000}}},
            {'query': 'bonus allowance incentive extra', 'n_results': 15, 'filters': {'type': 'credit'}},
            {'query': 'transfer credit received from', 'n_results': 20, 'filters': {'type': 'credit'}},
            {'query': 'large significant major deposit', 'n_results': 10, 'filters': {'type': 'credit', 'amount': {'$gte': 10000}}}
        ]
        
        # Gather all income data
        all_income = self._gather_income_data(queries)
        
        # Comprehensive analysis
        analysis = self._analyze_income(all_income, thinking)
        
        # Generate detailed summary
        summary_prompt = f"""Generate a comprehensive income analysis for Saudi Arabian context:

        Total Income: SAR {analysis['stats'].get('total_income', 0):,.2f}
        Monthly Average: SAR {analysis['stats'].get('monthly_average', 0):,.2f}
        Income Stability: {analysis['stats'].get('income_stability', 0):.0f}%
        
        Sources: {self._format_income_sources(analysis['income_sources'])}
        {self._format_salary_info(analysis['patterns'].get('salary', {}))}
        
        Provide:
        1. Income summary and stability assessment
        2. Primary vs secondary income sources
        3. Growth trend analysis
        4. Recommendations for income optimization
        
        Use SAR for all amounts."""
        
        summary = self.ollama.generate(
            prompt=summary_prompt,
            system_prompt="You are a Saudi income analyst. Provide clear, actionable insights in SAR.",
            temperature=0.2,
            max_tokens=400
        )
        
        return {
            'summary': summary,
            'data': analysis,
            'query_count': len(queries),
            'transactions_analyzed': len(all_income)
        }
```

## Query Examples

### User Query → Agent Response

**Query**: "What is my monthly salary?"
```
Hidden Thinking:
- Looking for regular monthly deposits
- Check for consistent amounts
- Verify monthly pattern (25th-30th typical in Saudi)

Response:
Your regular monthly salary is **SAR 15,000.00**

Details:
- Deposited on the 27th of each month
- Source: ACME Company Payroll
- Received consistently for the past 6 months
- Last salary: October 27, 2024

Your salary represents 75% of your total monthly income.
```

**Query**: "Show me all my income sources"
```
Hidden Thinking:
- Categorize all credit transactions
- Identify regular vs irregular income
- Calculate percentages

Response:
You have 4 income sources totaling SAR 23,450.00 this month:

**Primary Income (85%)**:
• Salary: SAR 15,000.00 - ACME Company (monthly)
• Housing Allowance: SAR 5,000.00 - ACME Company (monthly)

**Secondary Income (15%)**:
• Freelance Projects: SAR 2,750.00 - Various clients (2 payments)
• Investment Returns: SAR 700.00 - Trading account (irregular)

Income stability score: 82/100 (Very Stable)
```