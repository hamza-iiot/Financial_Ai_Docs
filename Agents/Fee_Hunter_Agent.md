# Fee Hunter Agent

## Overview
The Fee Hunter Agent specializes in detecting and analyzing bank fees, service charges, and hidden costs. It's designed to identify unnecessary expenses and help users save money by finding and eliminating avoidable fees in Saudi banking context.

## Agent Profile

- **Specialization**: Fee detection, charge analysis, cost optimization
- **Model**: qwen3-14b-32k:latest  
- **Search Depth**: 30-40 transactions per query
- **Query Count**: 6-8 different search angles

## Implementation

```python
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
import pandas as pd
import re

class FeeHunterAgent:
    """Specialized agent for detecting and analyzing fees"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Saudi banking fee keywords
        self.fee_keywords = {
            'bank_fees': [
                'fee', 'رسوم', 'charge', 'commission', 'عمولة',
                'service charge', 'رسوم خدمة', 'admin', 'إدارية'
            ],
            'atm_fees': [
                'atm', 'withdrawal fee', 'cash advance', 'atm charge',
                'صراف', 'سحب نقدي', 'رسوم صراف'
            ],
            'transfer_fees': [
                'transfer fee', 'wire fee', 'remittance', 'swift',
                'رسوم تحويل', 'حوالة', 'سويفت'
            ],
            'card_fees': [
                'card fee', 'annual fee', 'renewal', 'issuance',
                'رسوم بطاقة', 'رسوم سنوية', 'تجديد', 'إصدار'
            ],
            'vat_fees': [
                'vat', 'ضريبة', 'value added tax', 'tax',
                'ضريبة القيمة المضافة'
            ],
            'penalty_fees': [
                'penalty', 'late fee', 'overdraft', 'insufficient',
                'غرامة', 'رسوم تأخير', 'سحب على المكشوف'
            ],
            'subscription_fees': [
                'subscription', 'monthly charge', 'service fee',
                'اشتراك', 'رسوم شهرية', 'خدمة'
            ],
            'international_fees': [
                'international', 'foreign', 'currency conversion',
                'دولي', 'عملة أجنبية', 'تحويل عملة'
            ]
        }
        
        # Common Saudi banks for pattern matching
        self.saudi_banks = [
            'ncb', 'الأهلي', 'alrajhi', 'الراجحي', 'samba', 'سامبا',
            'riyad', 'الرياض', 'alinma', 'الإنماء', 'alawwal', 'الأول',
            'aljazira', 'الجزيرة', 'anb', 'العربي', 'saib', 'الاستثمار',
            'albilad', 'البلاد', 'sabb', 'ساب'
        ]
    
    def execute(self, query: str, thinking_context: Dict = None) -> Dict[str, Any]:
        """Execute fee detection with hidden analysis"""
        
        # Step 1: Hidden thinking about fees
        internal_analysis = self._think_about_fees(query, thinking_context)
        
        # Step 2: Generate comprehensive fee search queries
        search_queries = self._generate_fee_queries(internal_analysis)
        
        # Step 3: Hunt for all fees (extensive search)
        all_fees = self._hunt_for_fees(search_queries)
        
        # Step 4: Deep fee analysis and categorization
        fee_analysis = self._analyze_fees(all_fees, internal_analysis)
        
        # Step 5: Generate actionable response
        final_response = self._format_fee_response(fee_analysis)
        
        return {
            'final_answer': final_response,
            'sources': fee_analysis['all_fees'][:5],
            'statistics': fee_analysis['stats'],
            'savings_potential': fee_analysis['savings'],
            'thinking': internal_analysis  # Hidden
        }
    
    def _think_about_fees(self, query: str, context: Dict = None) -> Dict:
        """Hidden reasoning about fees and charges"""
        
        thinking_prompt = f"""Analyze this fee-related query for Saudi Arabian banking.
        
        Query: {query}
        
        Think about (hidden from user):
        1. Fee types: Bank fees, ATM fees, transfer fees, penalties, VAT?
        2. Patterns: Recurring vs one-time fees?
        3. Avoidability: Which fees could be avoided?
        4. Saudi context: VAT (15%), SAMA regulations, local bank practices
        5. Hidden fees: Small recurring charges that add up?
        6. International: Foreign transaction or currency conversion fees?
        
        Detailed analysis:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are a fee detection expert for Saudi banking. Identify all types of fees and charges.",
            temperature=0.3,
            max_tokens=500
        )
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'query_type': self._classify_fee_query(query),
            'context': context
        }
    
    def _generate_fee_queries(self, thinking: Dict) -> List[Dict]:
        """Generate comprehensive fee search queries"""
        
        queries = []
        
        # Query 1: General fee search
        queries.append({
            'query': 'fee charge commission رسوم عمولة service',
            'n_results': 40,
            'filters': {'type': 'debit'}
        })
        
        # Query 2: Bank fees specifically
        queries.append({
            'query': 'bank fee admin charge service رسوم بنكية إدارية',
            'n_results': 30,
            'filters': {'type': 'debit', 'amount': {'$lte': 100}}
        })
        
        # Query 3: ATM and withdrawal fees
        queries.append({
            'query': 'atm withdrawal cash fee صراف سحب رسوم',
            'n_results': 25,
            'filters': {'type': 'debit'}
        })
        
        # Query 4: VAT and taxes
        queries.append({
            'query': 'vat tax ضريبة القيمة المضافة',
            'n_results': 20,
            'filters': {'type': 'debit'}
        })
        
        # Query 5: Small recurring fees (often hidden)
        queries.append({
            'query': 'monthly subscription service recurring شهري اشتراك',
            'n_results': 25,
            'filters': {'type': 'debit', 'amount': {'$lte': 50}}
        })
        
        # Query 6: Transfer and international fees
        queries.append({
            'query': 'transfer wire international foreign swift تحويل دولي',
            'n_results': 20,
            'filters': {'type': 'debit'}
        })
        
        # Query 7: Penalties and late fees
        queries.append({
            'query': 'penalty late overdraft insufficient غرامة تأخير',
            'n_results': 15,
            'filters': {'type': 'debit'}
        })
        
        # Query 8: Card fees
        queries.append({
            'query': 'card annual renewal issuance بطاقة سنوية تجديد',
            'n_results': 15,
            'filters': {'type': 'debit'}
        })
        
        return queries
    
    def _hunt_for_fees(self, queries: List[Dict]) -> List[Dict]:
        """Aggressively hunt for all types of fees"""
        
        all_fees = []
        seen_ids = set()
        
        for query_config in queries:
            try:
                results = self.vector_store.search(
                    query=query_config['query'],
                    n_results=query_config['n_results'],
                    filter_dict=query_config.get('filters')
                )
                
                for trans in results.get('transactions', []):
                    # Additional fee detection logic
                    if self._is_likely_fee(trans):
                        trans_id = f"{trans.get('date')}_{trans.get('amount')}_{trans.get('description')}"
                        if trans_id not in seen_ids:
                            seen_ids.add(trans_id)
                            all_fees.append(trans)
                            
            except Exception as e:
                continue
        
        return all_fees
    
    def _is_likely_fee(self, transaction: Dict) -> bool:
        """Determine if transaction is likely a fee"""
        
        desc_lower = str(transaction.get('description', '')).lower()
        amount = transaction.get('amount', 0)
        
        # Check for fee keywords
        for fee_type, keywords in self.fee_keywords.items():
            if any(keyword in desc_lower for keyword in keywords):
                return True
        
        # Check for typical fee amounts (small debits)
        typical_fees = [5, 10, 15, 20, 25, 30, 35, 50, 75, 100, 2.5, 3.75, 7.5]
        if amount in typical_fees:
            # Additional context check
            if any(bank in desc_lower for bank in self.saudi_banks):
                return True
        
        # Check for VAT pattern (15% in Saudi)
        if 'vat' in desc_lower or 'ضريبة' in desc_lower:
            return True
        
        # Pattern matching for fee descriptions
        fee_patterns = [
            r'^\d+\.\d{2}$',  # Just an amount (often fees)
            r'service.*charge',
            r'admin.*fee',
            r'monthly.*fee'
        ]
        
        for pattern in fee_patterns:
            if re.search(pattern, desc_lower):
                return True
        
        return False
    
    def _analyze_fees(self, fees: List[Dict], thinking: Dict) -> Dict:
        """Comprehensive fee analysis"""
        
        if not fees:
            return {
                'total_fees': 0,
                'all_fees': [],
                'categories': {},
                'patterns': {},
                'stats': {},
                'savings': {}
            }
        
        df = pd.DataFrame(fees)
        df['date'] = pd.to_datetime(df['date'])
        
        # Categorize fees
        categorized = self._categorize_fees(df)
        
        # Calculate statistics
        stats = {
            'total_fees': float(df['amount'].sum()),
            'average_fee': float(df['amount'].mean()),
            'median_fee': float(df['amount'].median()),
            'largest_fee': float(df['amount'].max()),
            'fee_count': len(df),
            'monthly_average': float(df['amount'].sum() / max(1, df['date'].dt.month.nunique())),
            'daily_average': float(df['amount'].sum() / max(1, (df['date'].max() - df['date'].min()).days))
        }
        
        # Identify patterns
        patterns = {
            'recurring': self._find_recurring_fees(df),
            'avoidable': self._find_avoidable_fees(df),
            'by_bank': self._analyze_by_bank(df),
            'trends': self._analyze_fee_trends(df)
        }
        
        # Calculate savings potential
        savings = self._calculate_savings_potential(df, patterns)
        
        return {
            'total_fees': stats['total_fees'],
            'all_fees': df.to_dict('records'),
            'categories': categorized,
            'patterns': patterns,
            'stats': stats,
            'savings': savings
        }
    
    def _categorize_fees(self, df: pd.DataFrame) -> Dict:
        """Categorize fees by type"""
        
        categories = {cat: [] for cat in self.fee_keywords.keys()}
        categories['other'] = []
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            categorized = False
            
            for category, keywords in self.fee_keywords.items():
                if any(keyword in desc_lower for keyword in keywords):
                    categories[category].append(row.to_dict())
                    categorized = True
                    break
            
            if not categorized:
                categories['other'].append(row.to_dict())
        
        # Calculate totals
        category_totals = {}
        for cat, trans_list in categories.items():
            if trans_list:
                total = sum(t.get('amount', 0) for t in trans_list)
                category_totals[cat] = {
                    'total': total,
                    'count': len(trans_list),
                    'average': total / len(trans_list),
                    'percentage': (total / df['amount'].sum()) * 100,
                    'transactions': trans_list[:3]  # Top 3 examples
                }
        
        return category_totals
    
    def _find_recurring_fees(self, df: pd.DataFrame) -> List[Dict]:
        """Identify recurring fees"""
        
        recurring = []
        
        # Group by amount and description similarity
        amount_groups = df.groupby('amount')
        
        for amount, group in amount_groups:
            if len(group) >= 2:
                # Check if dates suggest recurring pattern
                dates = pd.to_datetime(group['date']).sort_values()
                if len(dates) > 1:
                    intervals = [(dates.iloc[i+1] - dates.iloc[i]).days 
                               for i in range(len(dates)-1)]
                    avg_interval = sum(intervals) / len(intervals)
                    
                    # Monthly recurring (25-35 days)
                    if 25 <= avg_interval <= 35:
                        recurring.append({
                            'amount': float(amount),
                            'frequency': 'monthly',
                            'occurrences': len(group),
                            'description': group.iloc[0]['description'],
                            'total_cost': float(amount * len(group)),
                            'annual_projection': float(amount * 12)
                        })
        
        return sorted(recurring, key=lambda x: x['total_cost'], reverse=True)
    
    def _find_avoidable_fees(self, df: pd.DataFrame) -> List[Dict]:
        """Identify potentially avoidable fees"""
        
        avoidable = []
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            
            # ATM fees from other banks
            if 'atm' in desc_lower and any(bank in desc_lower for bank in self.saudi_banks):
                avoidable.append({
                    'type': 'ATM fee',
                    'amount': row['amount'],
                    'description': row['description'],
                    'suggestion': 'Use your own bank ATMs to avoid fees'
                })
            
            # International transaction fees
            elif 'international' in desc_lower or 'foreign' in desc_lower:
                avoidable.append({
                    'type': 'International fee',
                    'amount': row['amount'],
                    'description': row['description'],
                    'suggestion': 'Use cards with no foreign transaction fees'
                })
            
            # Account maintenance fees
            elif 'maintenance' in desc_lower or 'admin' in desc_lower:
                avoidable.append({
                    'type': 'Maintenance fee',
                    'amount': row['amount'],
                    'description': row['description'],
                    'suggestion': 'Maintain minimum balance or switch to fee-free account'
                })
        
        return avoidable
    
    def _analyze_by_bank(self, df: pd.DataFrame) -> Dict:
        """Analyze fees by bank"""
        
        bank_fees = {}
        
        for _, row in df.iterrows():
            desc_lower = str(row.get('description', '')).lower()
            
            for bank in self.saudi_banks:
                if bank in desc_lower:
                    if bank not in bank_fees:
                        bank_fees[bank] = {
                            'total': 0,
                            'count': 0,
                            'fees': []
                        }
                    
                    bank_fees[bank]['total'] += row['amount']
                    bank_fees[bank]['count'] += 1
                    bank_fees[bank]['fees'].append(row.to_dict())
                    break
        
        return bank_fees
    
    def _analyze_fee_trends(self, df: pd.DataFrame) -> Dict:
        """Analyze fee trends over time"""
        
        df['month'] = df['date'].dt.to_period('M')
        monthly_fees = df.groupby('month')['amount'].sum()
        
        if len(monthly_fees) < 2:
            return {'trend': 'insufficient_data'}
        
        # Calculate trend
        first_half = monthly_fees.iloc[:len(monthly_fees)//2].mean()
        second_half = monthly_fees.iloc[len(monthly_fees)//2:].mean()
        
        if second_half > first_half * 1.1:
            trend = 'increasing'
        elif second_half < first_half * 0.9:
            trend = 'decreasing'
        else:
            trend = 'stable'
        
        return {
            'trend': trend,
            'monthly_average': float(monthly_fees.mean()),
            'highest_month': {
                'period': str(monthly_fees.idxmax()),
                'amount': float(monthly_fees.max())
            },
            'lowest_month': {
                'period': str(monthly_fees.idxmin()),
                'amount': float(monthly_fees.min())
            }
        }
    
    def _calculate_savings_potential(self, df: pd.DataFrame, patterns: Dict) -> Dict:
        """Calculate potential savings from eliminating fees"""
        
        savings = {
            'monthly_potential': 0,
            'annual_potential': 0,
            'recommendations': []
        }
        
        # Recurring fees savings
        for fee in patterns.get('recurring', []):
            if fee['frequency'] == 'monthly':
                savings['monthly_potential'] += fee['amount']
                savings['recommendations'].append(
                    f"Review recurring {fee['description']}: SAR {fee['amount']}/month"
                )
        
        # Avoidable fees savings
        avoidable_total = sum(f['amount'] for f in patterns.get('avoidable', []))
        if avoidable_total > 0:
            savings['monthly_potential'] += avoidable_total / max(1, df['date'].dt.month.nunique())
            savings['recommendations'].append(
                f"Avoid unnecessary fees: SAR {avoidable_total:.2f} total"
            )
        
        savings['annual_potential'] = savings['monthly_potential'] * 12
        
        return savings
    
    def _format_fee_response(self, analysis: Dict) -> str:
        """Format fee analysis into actionable response"""
        
        prompt = f"""Provide a comprehensive fee analysis for Saudi Arabian banking context.

        Fee Analysis:
        - Total Fees Paid: SAR {analysis['stats'].get('total_fees', 0):,.2f}
        - Number of Fees: {analysis['stats'].get('fee_count', 0)}
        - Monthly Average: SAR {analysis['stats'].get('monthly_average', 0):,.2f}
        
        Fee Categories:
        {self._format_fee_categories(analysis['categories'])}
        
        Recurring Fees:
        {self._format_recurring_fees(analysis['patterns'].get('recurring', []))}
        
        Savings Potential:
        - Monthly: SAR {analysis['savings'].get('monthly_potential', 0):,.2f}
        - Annual: SAR {analysis['savings'].get('annual_potential', 0):,.2f}
        
        Provide:
        1. Summary of fees paid
        2. Most expensive fee categories
        3. Specific recommendations to reduce fees
        4. Priority actions for immediate savings
        
        Use SAR for all amounts. Be specific and actionable."""
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are a fee optimization expert for Saudi banking.
            Focus on identifying unnecessary fees and providing specific actions to eliminate them.
            Always use SAR. Prioritize high-impact savings.""",
            temperature=0.2,
            max_tokens=500
        )
        
        return response
    
    def _format_fee_categories(self, categories: Dict) -> str:
        """Format fee categories"""
        if not categories:
            return "No fees categorized"
        
        lines = []
        for cat, data in sorted(categories.items(), 
                               key=lambda x: x[1].get('total', 0), 
                               reverse=True)[:5]:
            if data and data['count'] > 0:
                cat_name = cat.replace('_', ' ').title()
                lines.append(
                    f"- {cat_name}: SAR {data['total']:,.2f} "
                    f"({data['count']} fees, {data['percentage']:.1f}%)"
                )
        
        return '\n'.join(lines) if lines else "No significant fee categories"
    
    def _format_recurring_fees(self, recurring: List[Dict]) -> str:
        """Format recurring fees"""
        if not recurring:
            return "No recurring fees detected"
        
        lines = []
        for fee in recurring[:3]:
            lines.append(
                f"- {fee['description']}: SAR {fee['amount']:,.2f}/{fee['frequency']} "
                f"(Annual: SAR {fee['annual_projection']:,.2f})"
            )
        
        return '\n'.join(lines)
    
    def _classify_fee_query(self, query: str) -> str:
        """Classify fee query type"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['all', 'total', 'how much']):
            return 'total'
        elif any(word in query_lower for word in ['avoid', 'reduce', 'save']):
            return 'optimization'
        elif any(word in query_lower for word in ['recurring', 'monthly', 'subscription']):
            return 'recurring'
        elif any(word in query_lower for word in ['bank', 'atm', 'service']):
            return 'bank_fees'
        else:
            return 'general'
    
    def analyze_for_insights(self) -> Dict[str, Any]:
        """Comprehensive fee analysis for insights dashboard"""
        
        # Deep thinking about all fees
        thinking = self._think_about_fees(
            "Find ALL fees, charges, and hidden costs",
            context={'mode': 'insights', 'comprehensive': True}
        )
        
        # Exhaustive fee search queries
        queries = self._generate_fee_queries(thinking)
        
        # Hunt for all fees
        all_fees = self._hunt_for_fees(queries)
        
        # Comprehensive analysis
        analysis = self._analyze_fees(all_fees, thinking)
        
        # Generate actionable summary
        summary_prompt = f"""Generate a fee optimization report for Saudi banking:

        Total Fees: SAR {analysis['stats'].get('total_fees', 0):,.2f}
        Monthly Average: SAR {analysis['stats'].get('monthly_average', 0):,.2f}
        Fee Count: {analysis['stats'].get('fee_count', 0)}
        
        Categories: {self._format_fee_categories(analysis['categories'])}
        
        Savings Potential: SAR {analysis['savings'].get('annual_potential', 0):,.2f}/year
        
        Provide:
        1. Total fee impact assessment
        2. Top 3 most expensive fee types
        3. Immediate actions to reduce fees
        4. Estimated annual savings if recommendations followed
        
        Use SAR. Be specific with amounts and actions."""
        
        summary = self.ollama.generate(
            prompt=summary_prompt,
            system_prompt="You are a Saudi fee optimization expert. Provide clear actions to eliminate fees.",
            temperature=0.2,
            max_tokens=400
        )
        
        return {
            'summary': summary,
            'data': analysis,
            'query_count': len(queries),
            'fees_found': len(all_fees),
            'savings_potential': analysis['savings']
        }
```

## Query Examples

### User Query → Agent Response

**Query**: "How much am I paying in fees?"
```
Hidden Thinking:
- Search for all fee types
- Calculate total and breakdown
- Identify biggest fee categories
- Find savings opportunities

Response:
You're paying **SAR 487.50** in fees this month:

**Breakdown by Type:**
• Bank Service Fees: SAR 195.00 (40%)
  - Account maintenance: SAR 50.00
  - SMS alerts: SAR 25.00
  - Card annual fee: SAR 120.00

• ATM Fees: SAR 157.50 (32%)
  - Other bank ATMs: SAR 157.50 (21 withdrawals)

• Transfer Fees: SAR 135.00 (28%)
  - International transfers: SAR 100.00
  - Local transfers: SAR 35.00

**Action Required:** Switch to your bank's ATMs to save SAR 157.50/month (SAR 1,890/year)!
```