# Budget Advisor Agent

## Overview
The Budget Advisor Agent provides comprehensive budget analysis, spending vs income comparisons, and personalized financial recommendations. It combines insights from expense and income agents to deliver holistic budget advice.

## Agent Profile

- **Specialization**: Budget planning, financial health assessment, savings recommendations
- **Model**: qwen3-14b-32k:latest
- **Search Depth**: 25-35 transactions per query
- **Query Count**: 7-8 different perspectives

## Implementation

```python
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

class BudgetAdvisorAgent:
    """Specialized agent for budget analysis and recommendations"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Saudi budget guidelines (50/30/20 rule adapted)
        self.budget_rules = {
            'needs': 50,  # Essential expenses
            'wants': 30,  # Discretionary spending
            'savings': 20  # Savings and debt payment
        }
        
        # Saudi-specific budget categories
        self.budget_categories = {
            'needs': [
                'rent', 'mortgage', 'utilities', 'electricity', 'water',
                'internet', 'phone', 'groceries', 'insurance', 'medical',
                'transport', 'petrol', 'maintenance'
            ],
            'wants': [
                'restaurant', 'entertainment', 'shopping', 'travel',
                'hobbies', 'gym', 'subscriptions', 'cinema', 'cafe'
            ],
            'savings': [
                'savings', 'investment', 'emergency', 'retirement',
                'debt payment', 'loan', 'credit'
            ]
        }
    
    def execute(self, query: str, thinking_context: Dict = None) -> Dict[str, Any]:
        """Execute budget analysis with hidden thinking"""
        
        # Step 1: Hidden budget analysis thinking
        internal_analysis = self._think_about_budget(query, thinking_context)
        
        # Step 2: Generate comprehensive budget queries
        search_queries = self._generate_budget_queries(internal_analysis)
        
        # Step 3: Gather all financial data
        financial_data = self._gather_financial_data(search_queries)
        
        # Step 4: Deep budget analysis
        budget_analysis = self._analyze_budget(financial_data, internal_analysis)
        
        # Step 5: Generate personalized recommendations
        final_response = self._format_budget_response(budget_analysis)
        
        return {
            'final_answer': final_response,
            'sources': financial_data[:5],
            'statistics': budget_analysis['stats'],
            'recommendations': budget_analysis['recommendations'],
            'thinking': internal_analysis  # Hidden
        }
    
    def _think_about_budget(self, query: str, context: Dict = None) -> Dict:
        """Hidden reasoning about budget and financial health"""
        
        thinking_prompt = f"""Analyze this budget-related query for Saudi Arabian personal finance.
        
        Query: {query}
        
        Consider (hidden analysis):
        1. Financial health: Income vs expenses ratio
        2. Budget balance: Are they overspending or saving?
        3. Category analysis: Needs vs wants vs savings
        4. Saudi context: Cost of living, typical salaries
        5. Improvement areas: Where can they optimize?
        6. Emergency fund: Do they have financial cushion?
        7. Debt situation: Any concerning debt patterns?
        
        Detailed budget analysis:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are a Saudi financial advisor expert in budgeting and financial planning.",
            temperature=0.3,
            max_tokens=500
        )
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'query_type': self._classify_budget_query(query),
            'context': context
        }
    
    def _generate_budget_queries(self, thinking: Dict) -> List[Dict]:
        """Generate queries for comprehensive budget analysis"""
        
        queries = []
        
        # Income queries
        queries.append({
            'query': 'salary income credit deposit transfer',
            'n_results': 30,
            'filters': {'type': 'credit'}
        })
        
        # Essential expenses (needs)
        queries.append({
            'query': 'rent mortgage utilities electricity water phone groceries',
            'n_results': 35,
            'filters': {'type': 'debit'}
        })
        
        # Discretionary spending (wants)
        queries.append({
            'query': 'restaurant shopping entertainment travel cinema cafe mall',
            'n_results': 35,
            'filters': {'type': 'debit'}
        })
        
        # Savings and investments
        queries.append({
            'query': 'savings investment transfer deposit emergency',
            'n_results': 20,
            'filters': {'type': 'debit'}
        })
        
        # Large transactions (budget impact)
        queries.append({
            'query': 'large significant major payment expense',
            'n_results': 20,
            'filters': {'amount': {'$gte': 1000}}
        })
        
        # Recurring payments
        queries.append({
            'query': 'monthly recurring subscription regular payment',
            'n_results': 25,
            'filters': {'type': 'debit'}
        })
        
        # Debt and loan payments
        queries.append({
            'query': 'loan payment installment credit debt',
            'n_results': 15,
            'filters': {'type': 'debit'}
        })
        
        return queries
    
    def _gather_financial_data(self, queries: List[Dict]) -> List[Dict]:
        """Gather comprehensive financial data"""
        
        all_transactions = []
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
                        all_transactions.append(trans)
                        
            except Exception as e:
                continue
        
        return all_transactions
    
    def _analyze_budget(self, transactions: List[Dict], thinking: Dict) -> Dict:
        """Comprehensive budget analysis"""
        
        if not transactions:
            return {
                'stats': {},
                'budget_breakdown': {},
                'recommendations': [],
                'health_score': 0
            }
        
        df = pd.DataFrame(transactions)
        df['date'] = pd.to_datetime(df['date'])
        
        # Separate income and expenses
        income_df = df[df['type'] == 'credit']
        expense_df = df[df['type'] == 'debit']
        
        # Calculate totals
        total_income = float(income_df['amount'].sum()) if not income_df.empty else 0
        total_expenses = float(expense_df['amount'].sum()) if not expense_df.empty else 0
        
        # Categorize expenses
        categorized = self._categorize_budget_items(expense_df)
        
        # Calculate statistics
        stats = {
            'total_income': total_income,
            'total_expenses': total_expenses,
            'net_cashflow': total_income - total_expenses,
            'savings_rate': ((total_income - total_expenses) / total_income * 100) if total_income > 0 else 0,
            'expense_ratio': (total_expenses / total_income * 100) if total_income > 0 else 0,
            'daily_spending': total_expenses / max(1, (df['date'].max() - df['date'].min()).days),
            'monthly_income': self._calculate_monthly_average(income_df),
            'monthly_expenses': self._calculate_monthly_average(expense_df)
        }
        
        # Budget breakdown
        budget_breakdown = self._calculate_budget_breakdown(categorized, total_income)
        
        # Financial health score
        health_score = self._calculate_health_score(stats, budget_breakdown)
        
        # Generate recommendations
        recommendations = self._generate_recommendations(stats, budget_breakdown, categorized)
        
        return {
            'stats': stats,
            'budget_breakdown': budget_breakdown,
            'recommendations': recommendations,
            'health_score': health_score,
            'categorized': categorized
        }
    
    def _categorize_budget_items(self, df: pd.DataFrame) -> Dict:
        """Categorize transactions into needs/wants/savings"""
        
        categories = {
            'needs': [],
            'wants': [],
            'savings': [],
            'uncategorized': []
        }
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            categorized = False
            
            for category, keywords in self.budget_categories.items():
                if any(keyword in desc_lower for keyword in keywords):
                    categories[category].append(row.to_dict())
                    categorized = True
                    break
            
            if not categorized:
                categories['uncategorized'].append(row.to_dict())
        
        # Calculate totals
        category_totals = {}
        for cat, trans_list in categories.items():
            if trans_list:
                total = sum(t.get('amount', 0) for t in trans_list)
                category_totals[cat] = {
                    'total': total,
                    'count': len(trans_list),
                    'average': total / len(trans_list),
                    'items': trans_list[:5]  # Top 5 examples
                }
        
        return category_totals
    
    def _calculate_budget_breakdown(self, categorized: Dict, total_income: float) -> Dict:
        """Calculate budget percentage breakdown"""
        
        breakdown = {}
        
        for category in ['needs', 'wants', 'savings']:
            if category in categorized:
                amount = categorized[category]['total']
                percentage = (amount / total_income * 100) if total_income > 0 else 0
                
                breakdown[category] = {
                    'amount': amount,
                    'percentage': percentage,
                    'target': self.budget_rules[category],
                    'difference': percentage - self.budget_rules[category],
                    'status': self._get_budget_status(percentage, self.budget_rules[category])
                }
        
        return breakdown
    
    def _get_budget_status(self, actual: float, target: float) -> str:
        """Determine budget category status"""
        
        if actual <= target * 0.8:
            return 'excellent'
        elif actual <= target:
            return 'good'
        elif actual <= target * 1.2:
            return 'warning'
        else:
            return 'over-budget'
    
    def _calculate_health_score(self, stats: Dict, breakdown: Dict) -> float:
        """Calculate financial health score (0-100)"""
        
        score = 50  # Base score
        
        # Savings rate factor (max 30 points)
        savings_rate = stats.get('savings_rate', 0)
        if savings_rate >= 20:
            score += 30
        elif savings_rate >= 10:
            score += 20
        elif savings_rate >= 5:
            score += 10
        elif savings_rate < 0:
            score -= 20
        
        # Budget adherence factor (max 20 points)
        for category, data in breakdown.items():
            if data['status'] in ['excellent', 'good']:
                score += 7
            elif data['status'] == 'warning':
                score += 3
        
        # Expense ratio factor
        expense_ratio = stats.get('expense_ratio', 100)
        if expense_ratio <= 70:
            score += 10
        elif expense_ratio <= 85:
            score += 5
        elif expense_ratio > 100:
            score -= 10
        
        return max(0, min(100, score))
    
    def _calculate_monthly_average(self, df: pd.DataFrame) -> float:
        """Calculate monthly average"""
        
        if df.empty:
            return 0.0
        
        df['month'] = df['date'].dt.to_period('M')
        monthly = df.groupby('month')['amount'].sum()
        
        return float(monthly.mean())
    
    def _generate_recommendations(self, stats: Dict, breakdown: Dict, categorized: Dict) -> List[str]:
        """Generate personalized budget recommendations"""
        
        recommendations = []
        
        # Savings rate recommendations
        savings_rate = stats.get('savings_rate', 0)
        if savings_rate < 10:
            recommendations.append(
                f"Increase savings to at least 10% of income (currently {savings_rate:.1f}%)"
            )
        
        # Category-specific recommendations
        for category, data in breakdown.items():
            if data['status'] == 'over-budget':
                recommendations.append(
                    f"Reduce {category} spending by SAR {data['difference'] * stats['total_income'] / 100:.0f}"
                )
        
        # Expense ratio
        if stats['expense_ratio'] > 90:
            recommendations.append(
                "High expense ratio - review all discretionary spending"
            )
        
        return recommendations
    
    def _format_budget_response(self, analysis: Dict) -> str:
        """Format budget analysis into actionable response"""
        
        prompt = f"""Provide comprehensive budget analysis and advice for Saudi context.

        Financial Summary:
        - Monthly Income: SAR {analysis['stats'].get('monthly_income', 0):,.2f}
        - Monthly Expenses: SAR {analysis['stats'].get('monthly_expenses', 0):,.2f}
        - Net Cashflow: SAR {analysis['stats'].get('net_cashflow', 0):,.2f}
        - Savings Rate: {analysis['stats'].get('savings_rate', 0):.1f}%
        - Financial Health Score: {analysis.get('health_score', 0):.0f}/100
        
        Budget Breakdown:
        {self._format_budget_breakdown(analysis['budget_breakdown'])}
        
        Key Recommendations:
        {self._format_recommendations(analysis['recommendations'])}
        
        Provide:
        1. Budget health assessment
        2. Specific areas needing attention
        3. Actionable steps to improve financial health
        4. Savings opportunities
        
        Use SAR for all amounts. Be specific and practical."""
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are a Saudi financial advisor specializing in personal budgeting.
            Provide clear, actionable budget advice considering local cost of living.
            Always use SAR. Focus on practical improvements.""",
            temperature=0.2,
            max_tokens=500
        )
        
        return response
    
    def _format_budget_breakdown(self, breakdown: Dict) -> str:
        """Format budget breakdown"""
        lines = []
        
        for category, data in breakdown.items():
            status_emoji = {
                'excellent': '✅',
                'good': '👍',
                'warning': '⚠️',
                'over-budget': '❌'
            }.get(data['status'], '❓')
            
            lines.append(
                f"- {category.title()}: {data['percentage']:.1f}% "
                f"(Target: {data['target']}%) {status_emoji}"
            )
        
        return '\n'.join(lines)
    
    def _format_recommendations(self, recommendations: List[str]) -> str:
        """Format recommendations"""
        if not recommendations:
            return "- Maintain current budget discipline"
        
        return '\n'.join(f"- {rec}" for rec in recommendations[:4])
    
    def _classify_budget_query(self, query: str) -> str:
        """Classify budget query type"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['save', 'saving', 'cut', 'reduce']):
            return 'savings'
        elif any(word in query_lower for word in ['budget', 'plan', 'allocate']):
            return 'planning'
        elif any(word in query_lower for word in ['health', 'score', 'assessment']):
            return 'health'
        elif any(word in query_lower for word in ['compare', 'vs', 'ratio']):
            return 'comparison'
        else:
            return 'general'
    
    def analyze_for_insights(self) -> Dict[str, Any]:
        """Comprehensive budget analysis for insights dashboard"""
        
        # Deep budget thinking
        thinking = self._think_about_budget(
            "Provide complete budget health assessment",
            context={'mode': 'insights', 'comprehensive': True}
        )
        
        # Comprehensive queries
        queries = self._generate_budget_queries(thinking)
        
        # Gather all data
        all_data = self._gather_financial_data(queries)
        
        # Full budget analysis
        analysis = self._analyze_budget(all_data, thinking)
        
        # Generate budget report
        summary_prompt = f"""Generate a budget health report for Saudi personal finance:

        Financial Health Score: {analysis['health_score']:.0f}/100
        
        Monthly Income: SAR {analysis['stats'].get('monthly_income', 0):,.2f}
        Monthly Expenses: SAR {analysis['stats'].get('monthly_expenses', 0):,.2f}
        Savings Rate: {analysis['stats'].get('savings_rate', 0):.1f}%
        
        Budget Status:
        {self._format_budget_breakdown(analysis['budget_breakdown'])}
        
        Provide:
        1. Overall financial health assessment
        2. Budget strengths and weaknesses
        3. Top 3 actions for improvement
        4. Realistic monthly savings target
        
        Use SAR. Be encouraging but realistic."""
        
        summary = self.ollama.generate(
            prompt=summary_prompt,
            system_prompt="You are a supportive Saudi financial advisor. Provide practical budget guidance.",
            temperature=0.2,
            max_tokens=400
        )
        
        return {
            'summary': summary,
            'data': analysis,
            'health_score': analysis['health_score'],
            'recommendations': analysis['recommendations']
        }
```

## Query Examples

### User Query → Agent Response

**Query**: "How is my budget looking?"
```
Response:
Your budget health score is **72/100** (Good)

**Monthly Overview:**
• Income: SAR 15,000
• Expenses: SAR 12,350 (82% of income)
• Savings: SAR 2,650 (18%)

**Budget Breakdown:**
• Needs: 48% (Target: 50%) ✅
• Wants: 34% (Target: 30%) ⚠️
• Savings: 18% (Target: 20%) 👍

**Action Items:**
1. Reduce discretionary spending by SAR 600 to meet savings target
2. Your restaurant expenses (SAR 1,850) could be cut by 30%
3. Consider canceling unused subscriptions (SAR 245/month identified)

You're doing well but have room for improvement!
```