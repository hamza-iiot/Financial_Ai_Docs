# Trend Analyst Agent

## Overview
The Trend Analyst Agent specializes in identifying patterns, trends, and anomalies in financial data over time. It provides insights on spending habits, seasonal variations, and predictive analysis for future financial behavior.

## Agent Profile

- **Specialization**: Pattern recognition, trend analysis, predictive insights
- **Model**: qwen3-14b-32k:latest
- **Search Depth**: 50-100 transactions (needs historical data)
- **Query Count**: 8-10 different time-based queries

## Implementation

```python
from typing import Dict, List, Any, Optional, Tuple
from datetime import datetime, timedelta
import pandas as pd
import numpy as np
from scipy import stats

class TrendAnalystAgent:
    """Specialized agent for trend and pattern analysis"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Saudi-specific patterns
        self.saudi_patterns = {
            'ramadan': {'months': [3, 4], 'impact': 'increased_food_charity'},
            'eid': {'months': [4, 7], 'impact': 'increased_gifts_travel'},
            'summer': {'months': [6, 7, 8], 'impact': 'increased_travel_ac'},
            'back_to_school': {'months': [8, 9], 'impact': 'increased_education'},
            'year_end': {'months': [12], 'impact': 'increased_shopping'}
        }
        
        # Trend types to analyze
        self.trend_types = [
            'daily', 'weekly', 'monthly', 'seasonal',
            'category', 'merchant', 'amount_distribution'
        ]
    
    def execute(self, query: str, thinking_context: Dict = None) -> Dict[str, Any]:
        """Execute trend analysis with hidden thinking"""
        
        # Step 1: Hidden trend analysis thinking
        internal_analysis = self._think_about_trends(query, thinking_context)
        
        # Step 2: Generate time-based queries
        search_queries = self._generate_trend_queries(internal_analysis)
        
        # Step 3: Gather historical data
        historical_data = self._gather_historical_data(search_queries)
        
        # Step 4: Deep trend analysis
        trend_analysis = self._analyze_trends(historical_data, internal_analysis)
        
        # Step 5: Generate insightful response
        final_response = self._format_trend_response(trend_analysis)
        
        return {
            'final_answer': final_response,
            'sources': historical_data[:5],
            'statistics': trend_analysis['stats'],
            'patterns': trend_analysis['patterns'],
            'predictions': trend_analysis.get('predictions', {}),
            'thinking': internal_analysis  # Hidden
        }
    
    def _think_about_trends(self, query: str, context: Dict = None) -> Dict:
        """Hidden reasoning about trends and patterns"""
        
        thinking_prompt = f"""Analyze this trend-related query for Saudi Arabian financial patterns.
        
        Query: {query}
        
        Consider (hidden analysis):
        1. Time period: What timeframe should we analyze?
        2. Pattern type: Daily, weekly, monthly, or seasonal?
        3. Comparison: Are they comparing periods?
        4. Anomalies: Should we look for unusual patterns?
        5. Saudi context: Ramadan, Eid, summer patterns
        6. Predictions: Do they want future projections?
        7. Categories: Specific spending categories to focus on?
        
        Detailed trend analysis:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are a data analyst specializing in Saudi financial trends and patterns.",
            temperature=0.3,
            max_tokens=500
        )
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'query_type': self._classify_trend_query(query),
            'context': context
        }
    
    def _generate_trend_queries(self, thinking: Dict) -> List[Dict]:
        """Generate queries for historical trend analysis"""
        
        queries = []
        
        # Get all historical data (large search)
        queries.append({
            'query': 'transaction payment expense income all',
            'n_results': 100,
            'filters': {}
        })
        
        # Monthly patterns
        queries.append({
            'query': 'monthly recurring regular pattern',
            'n_results': 50,
            'filters': {}
        })
        
        # Large transactions (trend breakers)
        queries.append({
            'query': 'large significant unusual exceptional',
            'n_results': 30,
            'filters': {'amount': {'$gte': 2000}}
        })
        
        # Weekend spending
        queries.append({
            'query': 'weekend friday saturday shopping entertainment',
            'n_results': 40,
            'filters': {'type': 'debit'}
        })
        
        # Category-specific trends
        for category in ['food', 'transport', 'shopping', 'entertainment']:
            queries.append({
                'query': f'{category} expense spending',
                'n_results': 30,
                'filters': {'type': 'debit'}
            })
        
        return queries
    
    def _gather_historical_data(self, queries: List[Dict]) -> List[Dict]:
        """Gather extensive historical data for trend analysis"""
        
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
        
        return sorted(all_transactions, key=lambda x: x.get('date', ''))
    
    def _analyze_trends(self, transactions: List[Dict], thinking: Dict) -> Dict:
        """Comprehensive trend analysis"""
        
        if not transactions:
            return {
                'stats': {},
                'patterns': {},
                'trends': {},
                'anomalies': [],
                'predictions': {}
            }
        
        df = pd.DataFrame(transactions)
        df['date'] = pd.to_datetime(df['date'])
        df = df.sort_values('date')
        
        # Calculate statistics
        stats = self._calculate_trend_statistics(df)
        
        # Identify patterns
        patterns = {
            'daily': self._analyze_daily_patterns(df),
            'weekly': self._analyze_weekly_patterns(df),
            'monthly': self._analyze_monthly_patterns(df),
            'seasonal': self._analyze_seasonal_patterns(df),
            'category': self._analyze_category_trends(df)
        }
        
        # Detect anomalies
        anomalies = self._detect_anomalies(df)
        
        # Generate predictions
        predictions = self._generate_predictions(df, patterns)
        
        # Identify key trends
        trends = self._identify_key_trends(df, patterns)
        
        return {
            'stats': stats,
            'patterns': patterns,
            'trends': trends,
            'anomalies': anomalies,
            'predictions': predictions
        }
    
    def _calculate_trend_statistics(self, df: pd.DataFrame) -> Dict:
        """Calculate trend statistics"""
        
        # Separate by type
        expenses = df[df['type'] == 'debit']
        income = df[df['type'] == 'credit']
        
        # Time range
        date_range = (df['date'].max() - df['date'].min()).days
        
        return {
            'date_range_days': date_range,
            'total_transactions': len(df),
            'expense_trend': self._calculate_linear_trend(expenses),
            'income_trend': self._calculate_linear_trend(income),
            'volatility': self._calculate_volatility(df),
            'consistency_score': self._calculate_consistency(df)
        }
    
    def _calculate_linear_trend(self, df: pd.DataFrame) -> Dict:
        """Calculate linear trend (increasing/decreasing)"""
        
        if df.empty or len(df) < 2:
            return {'direction': 'insufficient_data', 'slope': 0}
        
        # Group by month
        df['month'] = df['date'].dt.to_period('M')
        monthly = df.groupby('month')['amount'].sum()
        
        if len(monthly) < 2:
            return {'direction': 'insufficient_data', 'slope': 0}
        
        # Linear regression
        x = np.arange(len(monthly))
        y = monthly.values
        slope, intercept = np.polyfit(x, y, 1)
        
        # Determine direction
        if slope > 100:
            direction = 'increasing'
        elif slope < -100:
            direction = 'decreasing'
        else:
            direction = 'stable'
        
        return {
            'direction': direction,
            'slope': float(slope),
            'monthly_change': float(slope),
            'percentage_change': float((slope / y[0] * 100)) if y[0] > 0 else 0
        }
    
    def _calculate_volatility(self, df: pd.DataFrame) -> float:
        """Calculate spending volatility"""
        
        daily_spending = df[df['type'] == 'debit'].groupby(df['date'].dt.date)['amount'].sum()
        
        if len(daily_spending) < 2:
            return 0.0
        
        return float(daily_spending.std() / daily_spending.mean() * 100) if daily_spending.mean() > 0 else 0.0
    
    def _calculate_consistency(self, df: pd.DataFrame) -> float:
        """Calculate transaction consistency score"""
        
        # Check how regular transactions are
        dates = pd.to_datetime(df['date'])
        if len(dates) < 2:
            return 0.0
        
        intervals = [(dates.iloc[i+1] - dates.iloc[i]).days for i in range(len(dates)-1)]
        
        if not intervals:
            return 0.0
        
        # Lower std deviation = more consistent
        consistency = 100 - min(100, stats.variation(intervals) * 100) if len(intervals) > 1 else 50.0
        
        return float(consistency)
    
    def _analyze_daily_patterns(self, df: pd.DataFrame) -> Dict:
        """Analyze daily spending patterns"""
        
        df['day_of_week'] = df['date'].dt.day_name()
        df['day_num'] = df['date'].dt.dayofweek
        
        # Saudi weekend is Friday-Saturday
        df['is_weekend'] = df['day_num'].isin([4, 5])  # Thursday night + Friday + Saturday
        
        daily_spending = df[df['type'] == 'debit'].groupby('day_of_week')['amount'].agg(['sum', 'mean', 'count'])
        
        weekend_spending = df[df['is_weekend'] & (df['type'] == 'debit')]['amount'].sum()
        weekday_spending = df[~df['is_weekend'] & (df['type'] == 'debit')]['amount'].sum()
        
        return {
            'by_day': daily_spending.to_dict() if not daily_spending.empty else {},
            'weekend_vs_weekday': {
                'weekend_total': float(weekend_spending),
                'weekday_total': float(weekday_spending),
                'weekend_ratio': float(weekend_spending / (weekend_spending + weekday_spending)) if (weekend_spending + weekday_spending) > 0 else 0
            },
            'highest_day': daily_spending.idxmax()['sum'] if not daily_spending.empty else None,
            'lowest_day': daily_spending.idxmin()['sum'] if not daily_spending.empty else None
        }
    
    def _analyze_weekly_patterns(self, df: pd.DataFrame) -> Dict:
        """Analyze weekly patterns"""
        
        df['week'] = df['date'].dt.isocalendar().week
        weekly_spending = df[df['type'] == 'debit'].groupby('week')['amount'].sum()
        
        if weekly_spending.empty:
            return {}
        
        return {
            'average_weekly': float(weekly_spending.mean()),
            'highest_week': {
                'week': int(weekly_spending.idxmax()),
                'amount': float(weekly_spending.max())
            },
            'lowest_week': {
                'week': int(weekly_spending.idxmin()),
                'amount': float(weekly_spending.min())
            },
            'weekly_volatility': float(weekly_spending.std())
        }
    
    def _analyze_monthly_patterns(self, df: pd.DataFrame) -> Dict:
        """Analyze monthly patterns including Saudi-specific events"""
        
        df['month'] = df['date'].dt.month
        df['month_name'] = df['date'].dt.month_name()
        
        monthly_spending = df[df['type'] == 'debit'].groupby('month_name')['amount'].sum()
        
        # Check for Saudi patterns (Ramadan, Eid, etc.)
        pattern_impacts = {}
        for pattern_name, pattern_info in self.saudi_patterns.items():
            pattern_months = pattern_info['months']
            pattern_spending = df[df['month'].isin(pattern_months) & (df['type'] == 'debit')]['amount'].sum()
            other_spending = df[~df['month'].isin(pattern_months) & (df['type'] == 'debit')]['amount'].sum()
            
            if pattern_spending > 0 and other_spending > 0:
                pattern_impacts[pattern_name] = {
                    'total': float(pattern_spending),
                    'impact': pattern_info['impact'],
                    'percentage_of_total': float(pattern_spending / (pattern_spending + other_spending) * 100)
                }
        
        return {
            'by_month': monthly_spending.to_dict() if not monthly_spending.empty else {},
            'saudi_patterns': pattern_impacts,
            'highest_month': monthly_spending.idxmax() if not monthly_spending.empty else None,
            'lowest_month': monthly_spending.idxmin() if not monthly_spending.empty else None
        }
    
    def _analyze_seasonal_patterns(self, df: pd.DataFrame) -> Dict:
        """Analyze seasonal patterns"""
        
        df['quarter'] = df['date'].dt.quarter
        df['month'] = df['date'].dt.month
        
        # Define Saudi seasons
        def get_season(month):
            if month in [12, 1, 2]:
                return 'winter'
            elif month in [3, 4, 5]:
                return 'spring'
            elif month in [6, 7, 8]:
                return 'summer'
            else:
                return 'fall'
        
        df['season'] = df['month'].apply(get_season)
        
        seasonal_spending = df[df['type'] == 'debit'].groupby('season')['amount'].agg(['sum', 'mean', 'count'])
        
        return {
            'by_season': seasonal_spending.to_dict() if not seasonal_spending.empty else {},
            'highest_season': seasonal_spending.idxmax()['sum'] if not seasonal_spending.empty else None,
            'summer_vs_winter': self._compare_seasons(df, 'summer', 'winter')
        }
    
    def _compare_seasons(self, df: pd.DataFrame, season1: str, season2: str) -> Dict:
        """Compare two seasons"""
        
        s1_spending = df[(df['season'] == season1) & (df['type'] == 'debit')]['amount'].sum()
        s2_spending = df[(df['season'] == season2) & (df['type'] == 'debit')]['amount'].sum()
        
        return {
            f'{season1}_total': float(s1_spending),
            f'{season2}_total': float(s2_spending),
            'difference': float(s1_spending - s2_spending),
            'ratio': float(s1_spending / s2_spending) if s2_spending > 0 else 0
        }
    
    def _analyze_category_trends(self, df: pd.DataFrame) -> Dict:
        """Analyze trends by spending category"""
        
        categories = {
            'food': ['restaurant', 'grocery', 'cafe', 'food'],
            'transport': ['uber', 'careem', 'taxi', 'petrol'],
            'shopping': ['mall', 'store', 'amazon', 'noon'],
            'entertainment': ['cinema', 'game', 'netflix']
        }
        
        category_trends = {}
        
        for cat_name, keywords in categories.items():
            cat_df = df[df['description'].str.lower().str.contains('|'.join(keywords), na=False)]
            
            if not cat_df.empty:
                cat_expenses = cat_df[cat_df['type'] == 'debit']
                trend = self._calculate_linear_trend(cat_expenses)
                
                category_trends[cat_name] = {
                    'total': float(cat_expenses['amount'].sum()),
                    'count': len(cat_expenses),
                    'trend': trend['direction'],
                    'monthly_change': trend['monthly_change']
                }
        
        return category_trends
    
    def _detect_anomalies(self, df: pd.DataFrame) -> List[Dict]:
        """Detect anomalous transactions or patterns"""
        
        anomalies = []
        
        # Amount-based anomalies (outliers)
        expenses = df[df['type'] == 'debit']['amount']
        if len(expenses) > 3:
            z_scores = stats.zscore(expenses)
            outliers = df[df['type'] == 'debit'][np.abs(z_scores) > 2.5]
            
            for _, row in outliers.iterrows():
                anomalies.append({
                    'type': 'amount_outlier',
                    'date': row['date'].strftime('%Y-%m-%d'),
                    'amount': float(row['amount']),
                    'description': row['description'],
                    'reason': 'Unusually high amount'
                })
        
        # Frequency anomalies
        daily_counts = df.groupby(df['date'].dt.date).size()
        if len(daily_counts) > 3:
            z_scores = stats.zscore(daily_counts)
            high_activity_days = daily_counts[z_scores > 2].index
            
            for day in high_activity_days:
                day_trans = df[df['date'].dt.date == day]
                anomalies.append({
                    'type': 'frequency_anomaly',
                    'date': str(day),
                    'transaction_count': len(day_trans),
                    'total_amount': float(day_trans['amount'].sum()),
                    'reason': 'Unusually high transaction count'
                })
        
        return anomalies[:10]  # Top 10 anomalies
    
    def _identify_key_trends(self, df: pd.DataFrame, patterns: Dict) -> List[str]:
        """Identify and summarize key trends"""
        
        trends = []
        
        # Overall spending trend
        expense_trend = self._calculate_linear_trend(df[df['type'] == 'debit'])
        if expense_trend['direction'] == 'increasing':
            trends.append(f"Spending increasing by SAR {expense_trend['monthly_change']:.0f}/month")
        elif expense_trend['direction'] == 'decreasing':
            trends.append(f"Spending decreasing by SAR {abs(expense_trend['monthly_change']):.0f}/month")
        
        # Weekend pattern
        if patterns.get('daily', {}).get('weekend_vs_weekday', {}).get('weekend_ratio', 0) > 0.4:
            trends.append("High weekend spending (>40% of total)")
        
        # Seasonal pattern
        if patterns.get('seasonal', {}).get('highest_season'):
            trends.append(f"Highest spending in {patterns['seasonal']['highest_season']}")
        
        return trends
    
    def _generate_predictions(self, df: pd.DataFrame, patterns: Dict) -> Dict:
        """Generate future predictions based on trends"""
        
        predictions = {}
        
        # Next month prediction
        monthly_spending = df[df['type'] == 'debit'].groupby(df['date'].dt.to_period('M'))['amount'].sum()
        
        if len(monthly_spending) >= 3:
            # Simple linear prediction
            x = np.arange(len(monthly_spending))
            y = monthly_spending.values
            slope, intercept = np.polyfit(x, y, 1)
            
            next_month_prediction = slope * len(monthly_spending) + intercept
            predictions['next_month_spending'] = max(0, float(next_month_prediction))
        
        # Pattern-based predictions
        if patterns.get('monthly', {}).get('saudi_patterns'):
            predictions['seasonal_impacts'] = patterns['monthly']['saudi_patterns']
        
        return predictions
    
    def _format_trend_response(self, analysis: Dict) -> str:
        """Format trend analysis into insightful response"""
        
        prompt = f"""Provide comprehensive trend analysis for Saudi financial data.

        Trend Statistics:
        - Date Range: {analysis['stats'].get('date_range_days', 0)} days
        - Spending Trend: {analysis['stats'].get('expense_trend', {}).get('direction', 'unknown')}
        - Monthly Change: SAR {analysis['stats'].get('expense_trend', {}).get('monthly_change', 0):,.2f}
        - Volatility: {analysis['stats'].get('volatility', 0):.1f}%
        
        Key Patterns:
        {self._format_patterns(analysis['patterns'])}
        
        Anomalies Detected: {len(analysis.get('anomalies', []))}
        
        Predictions:
        - Next Month: SAR {analysis.get('predictions', {}).get('next_month_spending', 0):,.2f}
        
        Provide:
        1. Summary of key trends
        2. Notable patterns or changes
        3. Anomalies or concerns
        4. Forward-looking insights
        
        Use SAR for all amounts. Focus on actionable insights."""
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are a financial trend analyst for Saudi personal finance.
            Identify patterns, explain trends, and provide predictive insights.
            Always use SAR. Make complex patterns easy to understand.""",
            temperature=0.2,
            max_tokens=500
        )
        
        return response
    
    def _format_patterns(self, patterns: Dict) -> str:
        """Format patterns for prompt"""
        lines = []
        
        # Daily patterns
        if patterns.get('daily', {}).get('highest_day'):
            lines.append(f"- Highest spending day: {patterns['daily']['highest_day']}")
        
        # Weekend ratio
        weekend_ratio = patterns.get('daily', {}).get('weekend_vs_weekday', {}).get('weekend_ratio', 0)
        lines.append(f"- Weekend spending: {weekend_ratio*100:.1f}% of total")
        
        # Monthly patterns
        if patterns.get('monthly', {}).get('highest_month'):
            lines.append(f"- Peak month: {patterns['monthly']['highest_month']}")
        
        # Category trends
        for cat, data in patterns.get('category', {}).items():
            if data.get('trend') == 'increasing':
                lines.append(f"- {cat.title()} spending increasing")
        
        return '\n'.join(lines[:5])
    
    def _classify_trend_query(self, query: str) -> str:
        """Classify trend query type"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['pattern', 'trend', 'change']):
            return 'pattern'
        elif any(word in query_lower for word in ['compare', 'vs', 'versus']):
            return 'comparison'
        elif any(word in query_lower for word in ['predict', 'future', 'next']):
            return 'prediction'
        elif any(word in query_lower for word in ['anomaly', 'unusual', 'strange']):
            return 'anomaly'
        else:
            return 'general'
    
    def analyze_for_insights(self) -> Dict[str, Any]:
        """Comprehensive trend analysis for insights dashboard"""
        
        # Deep trend thinking
        thinking = self._think_about_trends(
            "Analyze all financial trends and patterns",
            context={'mode': 'insights', 'comprehensive': True}
        )
        
        # Comprehensive queries
        queries = self._generate_trend_queries(thinking)
        
        # Gather all historical data
        all_data = self._gather_historical_data(queries)
        
        # Full trend analysis
        analysis = self._analyze_trends(all_data, thinking)
        
        # Generate trend report
        summary_prompt = f"""Generate a trend analysis report for Saudi personal finance:

        Overall Trend: {analysis['stats'].get('expense_trend', {}).get('direction', 'stable')}
        Volatility: {analysis['stats'].get('volatility', 0):.1f}%
        
        Key Patterns:
        {self._format_patterns(analysis['patterns'])}
        
        Anomalies: {len(analysis.get('anomalies', []))} detected
        
        Future Outlook:
        Next Month Projection: SAR {analysis.get('predictions', {}).get('next_month_spending', 0):,.2f}
        
        Provide:
        1. Trend summary
        2. Most significant pattern
        3. Risk factors or concerns
        4. Recommendation based on trends
        
        Use SAR. Be insightful but clear."""
        
        summary = self.ollama.generate(
            prompt=summary_prompt,
            system_prompt="You are a Saudi financial trend analyst. Provide clear trend insights.",
            temperature=0.2,
            max_tokens=400
        )
        
        return {
            'summary': summary,
            'data': analysis,
            'key_trends': analysis.get('trends', []),
            'anomalies': analysis.get('anomalies', [])[:5]
        }
```

## Query Examples

### User Query → Agent Response

**Query**: "How is my spending changing over time?"
```
Response:
Your spending is **increasing by SAR 450/month** (8% growth rate)

**Trend Analysis:**
• 3 months ago: SAR 11,200/month
• Last month: SAR 12,550/month  
• Current month: SAR 13,100 (projected)

**Key Patterns:**
• Weekend spending accounts for 45% of total
• Restaurant expenses increasing fastest (+SAR 200/month)
• Thursday-Friday have highest daily spending

**Seasonal Impact:**
Summer months show 25% higher spending due to travel and AC costs

**Recommendation:** Your spending growth rate exceeds income growth (3%). Consider reducing weekend entertainment expenses.
```