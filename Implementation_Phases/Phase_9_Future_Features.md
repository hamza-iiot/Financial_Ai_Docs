# Phase 9: Future Features (Optional)

## Overview
Advanced capabilities and features for future development of the Yomnai application.

## Future Feature Implementations

### 9.1 Multi-File Analysis
Create `src/ai/multi_file_processor.py`:
```python
from typing import List, Dict, Any
from pathlib import Path
import pandas as pd
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class MultiFileProcessor:
    """Process multiple bank statement files for comprehensive analysis"""
    
    def __init__(self, parser, vector_store):
        self.parser = parser
        self.vector_store = vector_store
        self.consolidated_data = None
    
    def process_multiple_files(self, file_paths: List[str]) -> Dict[str, Any]:
        """Process multiple CSV files and consolidate data"""
        all_transactions = []
        file_summaries = []
        
        for file_path in file_paths:
            try:
                # Parse each file
                transactions = self.parser.parse_file(file_path)
                all_transactions.extend(transactions)
                
                # Create file summary
                file_summaries.append({
                    'file': Path(file_path).name,
                    'transactions': len(transactions),
                    'date_range': self._get_date_range(transactions)
                })
                
                logger.info(f"Processed {Path(file_path).name}: {len(transactions)} transactions")
                
            except Exception as e:
                logger.error(f"Error processing {file_path}: {str(e)}")
                continue
        
        # Sort by date
        all_transactions.sort(key=lambda x: x.date)
        
        # Detect duplicates
        duplicates = self._detect_duplicates(all_transactions)
        
        # Create consolidated DataFrame
        self.consolidated_data = pd.DataFrame([t.to_dict() for t in all_transactions])
        
        return {
            'total_transactions': len(all_transactions),
            'files_processed': len(file_summaries),
            'file_summaries': file_summaries,
            'duplicates_found': len(duplicates),
            'date_range': self._get_date_range(all_transactions)
        }
    
    def _detect_duplicates(self, transactions: List) -> List:
        """Detect potential duplicate transactions across files"""
        seen = {}
        duplicates = []
        
        for trans in transactions:
            key = (trans.date, trans.amount, trans.description[:20])
            
            if key in seen:
                duplicates.append({
                    'original': seen[key],
                    'duplicate': trans
                })
            else:
                seen[key] = trans
        
        return duplicates
    
    def _get_date_range(self, transactions: List) -> Dict[str, str]:
        """Get date range of transactions"""
        if not transactions:
            return {'start': None, 'end': None}
        
        dates = [t.date for t in transactions]
        return {
            'start': min(dates).strftime('%Y-%m-%d'),
            'end': max(dates).strftime('%Y-%m-%d')
        }
    
    def analyze_cross_file_trends(self) -> Dict[str, Any]:
        """Analyze trends across multiple files"""
        if self.consolidated_data is None:
            return {}
        
        df = self.consolidated_data
        df['date'] = pd.to_datetime(df['date'])
        df['year_month'] = df['date'].dt.to_period('M')
        
        # Monthly trends
        monthly_trends = df.groupby(['year_month', 'type'])['amount'].sum().unstack(fill_value=0)
        
        # Year-over-year comparison
        df['year'] = df['date'].dt.year
        yearly_comparison = df.groupby(['year', 'type'])['amount'].sum().unstack(fill_value=0)
        
        return {
            'monthly_trends': monthly_trends.to_dict(),
            'yearly_comparison': yearly_comparison.to_dict(),
            'average_monthly_spending': df[df['type'] == 'debit'].groupby('year_month')['amount'].sum().mean(),
            'average_monthly_income': df[df['type'] == 'credit'].groupby('year_month')['amount'].sum().mean()
        }
```

### 9.2 Predictive Analytics
Create `src/ai/predictive_analytics.py`:
```python
import numpy as np
import pandas as pd
from typing import Dict, List, Tuple
from datetime import datetime, timedelta
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures

class PredictiveAnalytics:
    """Predictive analytics for financial forecasting"""
    
    def __init__(self):
        self.model = None
        self.poly_features = PolynomialFeatures(degree=2)
    
    def predict_future_spending(self, transactions: List, days_ahead: int = 30) -> Dict[str, Any]:
        """Predict future spending based on historical data"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        
        # Aggregate daily spending
        daily_spending = df[df['type'] == 'debit'].groupby(df['date'].dt.date)['amount'].sum()
        
        if len(daily_spending) < 7:
            return {"error": "Insufficient data for prediction"}
        
        # Prepare data for regression
        X = np.arange(len(daily_spending)).reshape(-1, 1)
        y = daily_spending.values
        
        # Fit polynomial regression
        X_poly = self.poly_features.fit_transform(X)
        self.model = LinearRegression()
        self.model.fit(X_poly, y)
        
        # Predict future
        future_X = np.arange(len(daily_spending), len(daily_spending) + days_ahead).reshape(-1, 1)
        future_X_poly = self.poly_features.transform(future_X)
        predictions = self.model.predict(future_X_poly)
        
        # Generate dates
        last_date = daily_spending.index[-1]
        future_dates = [last_date + timedelta(days=i+1) for i in range(days_ahead)]
        
        return {
            'predictions': {
                str(date): max(0, pred)  # Ensure non-negative
                for date, pred in zip(future_dates, predictions)
            },
            'total_predicted': max(0, sum(predictions)),
            'average_predicted': max(0, np.mean(predictions)),
            'confidence': self._calculate_confidence(X_poly, y)
        }
    
    def _calculate_confidence(self, X, y) -> float:
        """Calculate model confidence score"""
        if self.model is None:
            return 0.0
        
        score = self.model.score(X, y)
        return min(max(score * 100, 0), 100)  # Convert to percentage
    
    def detect_spending_anomalies(self, transactions: List) -> List[Dict]:
        """Detect anomalies in spending patterns"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        
        # Calculate rolling statistics
        df = df.sort_values('date')
        df['rolling_mean'] = df['amount'].rolling(window=7, min_periods=1).mean()
        df['rolling_std'] = df['amount'].rolling(window=7, min_periods=1).std()
        
        # Detect anomalies (2 standard deviations)
        df['is_anomaly'] = abs(df['amount'] - df['rolling_mean']) > (2 * df['rolling_std'])
        
        anomalies = []
        for _, row in df[df['is_anomaly']].iterrows():
            anomalies.append({
                'date': row['date'].strftime('%Y-%m-%d'),
                'description': row['description'],
                'amount': row['amount'],
                'expected_range': (
                    max(0, row['rolling_mean'] - 2 * row['rolling_std']),
                    row['rolling_mean'] + 2 * row['rolling_std']
                )
            })
        
        return anomalies
    
    def predict_cash_flow(self, transactions: List, days: int = 30) -> Dict[str, Any]:
        """Predict future cash flow"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        
        # Get latest balance
        latest_balance = df.iloc[-1]['balance'] if 'balance' in df.columns else 0
        
        # Predict income and expenses
        spending_prediction = self.predict_future_spending(transactions, days)
        income_prediction = self._predict_income(df, days)
        
        # Calculate projected balance
        projected_expenses = spending_prediction.get('total_predicted', 0)
        projected_income = income_prediction.get('total_predicted', 0)
        projected_balance = latest_balance + projected_income - projected_expenses
        
        return {
            'current_balance': latest_balance,
            'projected_balance': projected_balance,
            'projected_income': projected_income,
            'projected_expenses': projected_expenses,
            'net_change': projected_income - projected_expenses,
            'days_until_zero': self._calculate_days_until_zero(
                latest_balance, projected_income, projected_expenses, days
            )
        }
    
    def _predict_income(self, df: pd.DataFrame, days: int) -> Dict[str, Any]:
        """Predict future income"""
        # Identify income patterns
        income_df = df[df['type'] == 'credit']
        
        if income_df.empty:
            return {'total_predicted': 0}
        
        # Calculate average income frequency and amount
        income_dates = pd.to_datetime(income_df['date'])
        
        if len(income_dates) > 1:
            # Calculate average days between income
            date_diffs = income_dates.diff().dt.days.dropna()
            avg_frequency = date_diffs.mean()
            
            # Calculate average income amount
            avg_amount = income_df['amount'].mean()
            
            # Predict number of income events in next N days
            predicted_events = days / avg_frequency if avg_frequency > 0 else 0
            total_predicted = predicted_events * avg_amount
        else:
            total_predicted = 0
        
        return {'total_predicted': total_predicted}
    
    def _calculate_days_until_zero(self, balance: float, income: float, expenses: float, max_days: int) -> int:
        """Calculate days until balance reaches zero"""
        if balance <= 0:
            return 0
        
        daily_net = (income - expenses) / max_days if max_days > 0 else 0
        
        if daily_net >= 0:
            return -1  # Balance increasing or stable
        
        days = abs(balance / daily_net)
        return min(int(days), max_days)
```

### 9.3 Budget Management
Create `src/core/budget_manager.py`:
```python
from typing import Dict, List, Optional
from datetime import datetime, timedelta
import json
from pathlib import Path

class BudgetManager:
    """Manage budgets and spending limits"""
    
    def __init__(self, config_path: str = "./config/budgets.json"):
        self.config_path = Path(config_path)
        self.budgets = self._load_budgets()
    
    def _load_budgets(self) -> Dict:
        """Load budget configuration"""
        if self.config_path.exists():
            with open(self.config_path, 'r') as f:
                return json.load(f)
        return {}
    
    def _save_budgets(self):
        """Save budget configuration"""
        self.config_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self.config_path, 'w') as f:
            json.dump(self.budgets, f, indent=2)
    
    def set_budget(self, category: str, amount: float, period: str = "monthly"):
        """Set budget for a category"""
        if category not in self.budgets:
            self.budgets[category] = {}
        
        self.budgets[category] = {
            'amount': amount,
            'period': period,
            'created_at': datetime.now().isoformat(),
            'alerts_enabled': True
        }
        
        self._save_budgets()
    
    def check_budget_status(self, transactions: List, current_date: Optional[datetime] = None) -> Dict:
        """Check current status against budgets"""
        if not current_date:
            current_date = datetime.now()
        
        status = {}
        
        for category, budget_config in self.budgets.items():
            # Filter transactions for period
            period_transactions = self._filter_by_period(
                transactions, 
                budget_config['period'], 
                current_date
            )
            
            # Calculate spending
            spent = sum(t.amount for t in period_transactions 
                       if t.category == category and t.type.value == 'debit')
            
            budget_amount = budget_config['amount']
            remaining = budget_amount - spent
            percentage = (spent / budget_amount * 100) if budget_amount > 0 else 0
            
            status[category] = {
                'budget': budget_amount,
                'spent': spent,
                'remaining': remaining,
                'percentage': percentage,
                'status': self._get_budget_status(percentage),
                'alert': percentage > 80
            }
        
        return status
    
    def _filter_by_period(self, transactions: List, period: str, current_date: datetime) -> List:
        """Filter transactions by budget period"""
        if period == "monthly":
            start_date = current_date.replace(day=1)
        elif period == "weekly":
            start_date = current_date - timedelta(days=current_date.weekday())
        elif period == "yearly":
            start_date = current_date.replace(month=1, day=1)
        else:
            start_date = current_date - timedelta(days=30)
        
        return [t for t in transactions if t.date >= start_date]
    
    def _get_budget_status(self, percentage: float) -> str:
        """Get budget status based on percentage used"""
        if percentage < 50:
            return "good"
        elif percentage < 80:
            return "warning"
        elif percentage < 100:
            return "critical"
        else:
            return "exceeded"
    
    def get_budget_recommendations(self, transactions: List) -> List[str]:
        """Get budget recommendations based on spending patterns"""
        recommendations = []
        
        # Analyze spending by category
        from src.ai.categorization import TransactionCategorizer
        categorizer = TransactionCategorizer()
        category_summary = categorizer.get_category_summary(transactions)
        
        for category, stats in category_summary.items():
            if category not in self.budgets:
                avg_monthly = stats['total'] / 30 * 30  # Rough monthly estimate
                recommendations.append(
                    f"Consider setting a budget for {category}: "
                    f"${avg_monthly:.2f}/month based on current spending"
                )
            else:
                budget = self.budgets[category]['amount']
                if stats['total'] > budget * 1.2:
                    recommendations.append(
                        f"Budget for {category} may be too low. "
                        f"Consider increasing from ${budget:.2f} to ${stats['total']:.2f}"
                    )
        
        return recommendations
```

### 9.4 Automated Insights Engine
Create `src/ai/insights_engine.py`:
```python
from typing import Dict, List, Any
import pandas as pd
from datetime import datetime, timedelta

class InsightsEngine:
    """Generate automated financial insights"""
    
    def __init__(self):
        self.insight_templates = self._init_templates()
    
    def _init_templates(self):
        """Initialize insight templates"""
        return {
            'spending_increase': "Your {category} spending increased by {percentage}% compared to last month",
            'new_merchant': "You started using a new service: {merchant}",
            'savings_opportunity': "You could save ${amount} by reducing {category} expenses by {percentage}%",
            'positive_trend': "Great! Your {metric} improved by {percentage}% this month",
            'unusual_time': "Unusual activity detected: {count} transactions after midnight",
            'fee_alert': "You paid ${amount} in fees this month - consider ways to reduce them"
        }
    
    def generate_insights(self, transactions: List) -> List[Dict[str, Any]]:
        """Generate comprehensive insights"""
        insights = []
        
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        
        # Spending trends
        insights.extend(self._analyze_spending_trends(df))
        
        # Time-based patterns
        insights.extend(self._analyze_time_patterns(df))
        
        # Fee analysis
        insights.extend(self._analyze_fees(df))
        
        # Merchant analysis
        insights.extend(self._analyze_merchants(df))
        
        # Savings opportunities
        insights.extend(self._identify_savings_opportunities(df))
        
        return insights
    
    def _analyze_spending_trends(self, df: pd.DataFrame) -> List[Dict]:
        """Analyze spending trends"""
        insights = []
        
        # Compare current month to previous
        current_month = df['date'].max().month
        prev_month = current_month - 1 if current_month > 1 else 12
        
        current_spending = df[(df['date'].dt.month == current_month) & 
                             (df['type'] == 'debit')]['amount'].sum()
        prev_spending = df[(df['date'].dt.month == prev_month) & 
                          (df['type'] == 'debit')]['amount'].sum()
        
        if prev_spending > 0:
            change_pct = ((current_spending - prev_spending) / prev_spending) * 100
            
            if abs(change_pct) > 20:
                insights.append({
                    'type': 'spending_trend',
                    'priority': 'high' if change_pct > 0 else 'medium',
                    'message': f"Spending {'increased' if change_pct > 0 else 'decreased'} by {abs(change_pct):.1f}% compared to last month",
                    'data': {
                        'current': current_spending,
                        'previous': prev_spending,
                        'change': change_pct
                    }
                })
        
        return insights
    
    def _analyze_time_patterns(self, df: pd.DataFrame) -> List[Dict]:
        """Analyze transaction time patterns"""
        insights = []
        
        df['hour'] = pd.to_datetime(df['date']).dt.hour
        
        # Late night transactions
        late_night = df[(df['hour'] >= 0) & (df['hour'] < 6)]
        if len(late_night) > 5:
            insights.append({
                'type': 'time_pattern',
                'priority': 'low',
                'message': f"{len(late_night)} transactions made during late night hours (midnight-6am)",
                'data': {
                    'count': len(late_night),
                    'total_amount': late_night['amount'].sum()
                }
            })
        
        # Weekend vs weekday spending
        df['weekday'] = pd.to_datetime(df['date']).dt.dayofweek
        weekend_spending = df[df['weekday'].isin([5, 6])]['amount'].sum()
        weekday_spending = df[~df['weekday'].isin([5, 6])]['amount'].sum()
        
        if weekend_spending > weekday_spending * 0.4:  # If weekend is >40% of weekday
            insights.append({
                'type': 'time_pattern',
                'priority': 'medium',
                'message': "High weekend spending detected - consider budgeting for leisure activities",
                'data': {
                    'weekend': weekend_spending,
                    'weekday': weekday_spending
                }
            })
        
        return insights
    
    def _analyze_fees(self, df: pd.DataFrame) -> List[Dict]:
        """Analyze bank fees and charges"""
        insights = []
        
        fee_keywords = ['fee', 'charge', 'penalty', 'interest', 'overdraft']
        fee_mask = df['description'].str.lower().str.contains('|'.join(fee_keywords), na=False)
        fees = df[fee_mask]
        
        if not fees.empty:
            total_fees = fees['amount'].sum()
            
            insights.append({
                'type': 'fees',
                'priority': 'high' if total_fees > 50 else 'medium',
                'message': f"You paid ${total_fees:.2f} in fees - consider ways to avoid these charges",
                'data': {
                    'total_fees': total_fees,
                    'fee_count': len(fees),
                    'largest_fee': fees['amount'].max()
                }
            })
        
        return insights
    
    def _analyze_merchants(self, df: pd.DataFrame) -> List[Dict]:
        """Analyze merchant patterns"""
        insights = []
        
        # Find most frequent merchants
        merchant_counts = df['description'].value_counts().head(5)
        
        if not merchant_counts.empty:
            top_merchant = merchant_counts.index[0]
            top_count = merchant_counts.iloc[0]
            
            if top_count > 10:
                insights.append({
                    'type': 'merchant_pattern',
                    'priority': 'low',
                    'message': f"Frequent transactions at {top_merchant} ({top_count} times)",
                    'data': {
                        'merchant': top_merchant,
                        'count': int(top_count),
                        'total_spent': df[df['description'] == top_merchant]['amount'].sum()
                    }
                })
        
        return insights
    
    def _identify_savings_opportunities(self, df: pd.DataFrame) -> List[Dict]:
        """Identify potential savings"""
        insights = []
        
        # Subscription detection
        potential_subscriptions = self._detect_subscriptions(df)
        
        if potential_subscriptions:
            total_subscription_cost = sum(s['amount'] for s in potential_subscriptions)
            
            insights.append({
                'type': 'savings_opportunity',
                'priority': 'medium',
                'message': f"Review {len(potential_subscriptions)} recurring subscriptions totaling ${total_subscription_cost:.2f}/month",
                'data': {
                    'subscriptions': potential_subscriptions,
                    'total_cost': total_subscription_cost
                }
            })
        
        # Small frequent purchases
        small_purchases = df[(df['amount'] < 10) & (df['type'] == 'debit')]
        if len(small_purchases) > 20:
            total_small = small_purchases['amount'].sum()
            
            insights.append({
                'type': 'savings_opportunity',
                'priority': 'low',
                'message': f"Small purchases add up: {len(small_purchases)} transactions under $10 total ${total_small:.2f}",
                'data': {
                    'count': len(small_purchases),
                    'total': total_small,
                    'average': small_purchases['amount'].mean()
                }
            })
        
        return insights
    
    def _detect_subscriptions(self, df: pd.DataFrame) -> List[Dict]:
        """Detect recurring subscriptions"""
        subscriptions = []
        
        # Group by similar descriptions and amounts
        for description in df['description'].unique():
            desc_transactions = df[df['description'] == description]
            
            if len(desc_transactions) >= 2:
                # Check if amounts are consistent
                amounts = desc_transactions['amount'].values
                if len(set(amounts)) == 1:  # Same amount each time
                    # Check if dates are roughly monthly
                    dates = pd.to_datetime(desc_transactions['date'])
                    date_diffs = dates.diff().dt.days.dropna()
                    
                    if not date_diffs.empty:
                        avg_diff = date_diffs.mean()
                        if 25 <= avg_diff <= 35:  # Roughly monthly
                            subscriptions.append({
                                'description': description,
                                'amount': float(amounts[0]),
                                'frequency': 'monthly'
                            })
        
        return subscriptions
```

## Implementation Roadmap

### Phase 1: Core Features (Weeks 1-2)
- Basic CSV parsing
- Vector store setup
- Simple chat interface
- Basic Q&A functionality

### Phase 2: Enhanced Analysis (Weeks 3-4)
- Advanced query processing
- Transaction categorization
- Basic reporting (PDF, Excel)
- Performance optimization

### Phase 3: Predictive Features (Weeks 5-6)
- Spending predictions
- Anomaly detection
- Budget management
- Automated insights

### Phase 4: Advanced Features (Weeks 7-8)
- Multi-file analysis
- Cash flow forecasting
- Subscription detection
- Advanced visualizations

## Success Metrics

### User Engagement
- Average session duration > 10 minutes
- Questions per session > 5
- Return user rate > 60%

### Technical Performance
- Query response time < 2 seconds
- CSV processing < 5 seconds for 1000 transactions
- 99% uptime for local service

### Analysis Quality
- Category accuracy > 85%
- Prediction confidence > 70%
- User satisfaction > 4/5 stars

## Deployment Options

### 1. Local Installation
```bash
# One-line installer
curl -sSL https://argus.app/install.sh | bash
```

### 2. Docker Container
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["streamlit", "run", "app.py"]
```

### 3. Desktop Application
- Package with PyInstaller
- Create installer for Windows/Mac/Linux
- Auto-update mechanism

## Conclusion

The Yomnai Financial Analyst represents a comprehensive solution for private, intelligent financial analysis. With these nine phases of development, the application can evolve from a simple CSV analyzer to a sophisticated financial assistant that provides predictive insights, budget management, and automated recommendations—all while maintaining complete data privacy.

The modular architecture ensures that each component can be developed, tested, and deployed independently, allowing for flexible development timelines and easy maintenance. The focus on local processing ensures user privacy while the AI-powered analysis provides professional-grade financial insights.

Future development can continue to add features based on user feedback and emerging technologies, while the solid foundation ensures scalability and reliability.