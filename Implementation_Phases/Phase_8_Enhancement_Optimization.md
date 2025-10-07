# Phase 8: Enhancement & Optimization

## Overview
Advanced features and performance optimizations to enhance the Yomnai application.

## Tasks

### 8.1 Advanced Query Features
Create `src/ai/advanced_queries.py`:
```python
from typing import Dict, Any, List, Tuple
import re
from datetime import datetime, timedelta
import pandas as pd

class AdvancedQueryProcessor:
    """Process advanced financial queries"""
    
    def __init__(self, rag_pipeline):
        self.rag = rag_pipeline
        self.query_patterns = self._init_query_patterns()
    
    def _init_query_patterns(self):
        """Initialize query pattern matching"""
        return {
            'comparison': r'compare|versus|vs|difference between',
            'trend': r'trend|over time|pattern|change',
            'forecast': r'predict|forecast|estimate|project',
            'anomaly': r'unusual|strange|odd|anomaly|outlier',
            'budget': r'budget|limit|threshold|exceed',
            'savings': r'save|saving|could have saved',
            'recurring': r'recurring|regular|monthly|weekly|subscription'
        }
    
    def process_advanced_query(self, query: str, transactions: List) -> Dict[str, Any]:
        """Process advanced query types"""
        query_lower = query.lower()
        
        # Identify query type
        for query_type, pattern in self.query_patterns.items():
            if re.search(pattern, query_lower):
                handler = getattr(self, f'_handle_{query_type}_query', None)
                if handler:
                    return handler(query, transactions)
        
        # Default to standard processing
        return self.rag.process_query(query)
    
    def _handle_comparison_query(self, query: str, transactions: List) -> Dict[str, Any]:
        """Handle comparison queries"""
        # Extract periods to compare
        months = re.findall(r'(january|february|march|april|may|june|july|august|september|october|november|december)', query.lower())
        
        if len(months) < 2:
            return {"answer": "Please specify two periods to compare."}
        
        # Convert transactions to DataFrame
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        df['month'] = df['date'].dt.month_name().str.lower()
        
        # Compare periods
        period1_data = df[df['month'] == months[0]]
        period2_data = df[df['month'] == months[1]]
        
        comparison = {
            f"{months[0]}_total": period1_data['amount'].sum(),
            f"{months[1]}_total": period2_data['amount'].sum(),
            f"{months[0]}_count": len(period1_data),
            f"{months[1]}_count": len(period2_data),
            "difference": period2_data['amount'].sum() - period1_data['amount'].sum()
        }
        
        answer = f"""Comparison between {months[0].capitalize()} and {months[1].capitalize()}:
        
{months[0].capitalize()}:
- Total: ${comparison[f'{months[0]}_total']:.2f}
- Transactions: {comparison[f'{months[0]}_count']}

{months[1].capitalize()}:
- Total: ${comparison[f'{months[1]}_total']:.2f}
- Transactions: {comparison[f'{months[1]}_count']}

Difference: ${abs(comparison['difference']):.2f} {'increase' if comparison['difference'] > 0 else 'decrease'}"""
        
        return {"answer": answer, "data": comparison}
    
    def _handle_trend_query(self, query: str, transactions: List) -> Dict[str, Any]:
        """Handle trend analysis queries"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        df['date'] = pd.to_datetime(df['date'])
        
        # Calculate daily spending trend
        daily_spending = df[df['type'] == 'debit'].groupby(df['date'].dt.date)['amount'].sum()
        
        # Calculate trend metrics
        trend_data = {
            "average_daily": daily_spending.mean(),
            "median_daily": daily_spending.median(),
            "max_daily": daily_spending.max(),
            "min_daily": daily_spending.min(),
            "trend_direction": "increasing" if daily_spending.iloc[-7:].mean() > daily_spending.iloc[:7].mean() else "decreasing"
        }
        
        answer = f"""Spending Trend Analysis:
        
- Average daily spending: ${trend_data['average_daily']:.2f}
- Median daily spending: ${trend_data['median_daily']:.2f}
- Highest daily spending: ${trend_data['max_daily']:.2f}
- Lowest daily spending: ${trend_data['min_daily']:.2f}
- Recent trend: {trend_data['trend_direction']}"""
        
        return {"answer": answer, "trend_data": trend_data}
    
    def _handle_anomaly_query(self, query: str, transactions: List) -> Dict[str, Any]:
        """Detect and report anomalies"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        
        # Calculate statistics
        mean_amount = df['amount'].mean()
        std_amount = df['amount'].std()
        
        # Find outliers (2 standard deviations)
        threshold = mean_amount + (2 * std_amount)
        anomalies = df[df['amount'] > threshold]
        
        if anomalies.empty:
            return {"answer": "No unusual transactions detected."}
        
        answer = "Unusual transactions detected:\n\n"
        for _, row in anomalies.iterrows():
            answer += f"- {row['date']}: {row['description']} - ${row['amount']:.2f}\n"
        
        answer += f"\nThese transactions are significantly above the average of ${mean_amount:.2f}"
        
        return {"answer": answer, "anomalies": anomalies.to_dict('records')}
    
    def _handle_recurring_query(self, query: str, transactions: List) -> Dict[str, Any]:
        """Identify recurring transactions"""
        df = pd.DataFrame([t.to_dict() for t in transactions])
        
        # Group by description similarity
        recurring = {}
        
        for _, trans in df.iterrows():
            desc_key = self._normalize_description(trans['description'])
            
            if desc_key not in recurring:
                recurring[desc_key] = []
            
            recurring[desc_key].append({
                'date': trans['date'],
                'amount': trans['amount'],
                'description': trans['description']
            })
        
        # Filter to only recurring (2+ occurrences)
        recurring = {k: v for k, v in recurring.items() if len(v) >= 2}
        
        if not recurring:
            return {"answer": "No recurring transactions identified."}
        
        answer = "Recurring transactions identified:\n\n"
        
        for desc, trans_list in recurring.items():
            avg_amount = sum(t['amount'] for t in trans_list) / len(trans_list)
            answer += f"- {trans_list[0]['description']}\n"
            answer += f"  Frequency: {len(trans_list)} times\n"
            answer += f"  Average amount: ${avg_amount:.2f}\n\n"
        
        return {"answer": answer, "recurring": recurring}
    
    def _normalize_description(self, description: str) -> str:
        """Normalize description for grouping"""
        # Remove numbers and special characters
        normalized = re.sub(r'[0-9#*]', '', description)
        # Take first few words
        words = normalized.split()[:3]
        return ' '.join(words).lower()
```

### 8.2 Transaction Categorization
Create `src/ai/categorization.py`:
```python
from typing import Dict, List, Optional
import re
from src.models import Transaction

class TransactionCategorizer:
    """Automatically categorize transactions"""
    
    def __init__(self):
        self.categories = self._init_categories()
        self.merchant_cache = {}
    
    def _init_categories(self) -> Dict[str, List[str]]:
        """Initialize category keywords"""
        return {
            'Groceries': [
                'grocery', 'supermarket', 'market', 'food', 'walmart', 'target',
                'kroger', 'safeway', 'whole foods', 'trader joe', 'aldi'
            ],
            'Dining': [
                'restaurant', 'cafe', 'coffee', 'pizza', 'burger', 'sushi',
                'starbucks', 'mcdonalds', 'subway', 'chipotle', 'dining'
            ],
            'Transportation': [
                'gas', 'fuel', 'uber', 'lyft', 'taxi', 'parking', 'toll',
                'shell', 'exxon', 'chevron', 'bp', 'transit', 'auto'
            ],
            'Utilities': [
                'electric', 'gas', 'water', 'internet', 'cable', 'phone',
                'utility', 'verizon', 'at&t', 'comcast', 'power', 'energy'
            ],
            'Entertainment': [
                'movie', 'netflix', 'spotify', 'hulu', 'disney', 'gaming',
                'theater', 'concert', 'music', 'streaming', 'entertainment'
            ],
            'Shopping': [
                'amazon', 'ebay', 'online', 'store', 'shop', 'mall',
                'clothing', 'electronics', 'department', 'retail'
            ],
            'Healthcare': [
                'pharmacy', 'medical', 'doctor', 'hospital', 'clinic',
                'cvs', 'walgreens', 'health', 'dental', 'insurance'
            ],
            'Banking': [
                'fee', 'charge', 'interest', 'atm', 'transfer', 'withdrawal',
                'deposit', 'bank', 'overdraft', 'service charge'
            ],
            'Income': [
                'salary', 'payroll', 'deposit', 'employer', 'income',
                'wage', 'bonus', 'commission', 'payment received'
            ],
            'Bills': [
                'insurance', 'rent', 'mortgage', 'loan', 'payment',
                'monthly', 'subscription', 'membership'
            ]
        }
    
    def categorize_transaction(self, transaction: Transaction) -> str:
        """Categorize a single transaction"""
        description_lower = transaction.description.lower()
        
        # Check cache first
        if description_lower in self.merchant_cache:
            return self.merchant_cache[description_lower]
        
        # Check for income (credits usually)
        if transaction.type.value == 'credit':
            for keyword in self.categories['Income']:
                if keyword in description_lower:
                    category = 'Income'
                    self.merchant_cache[description_lower] = category
                    return category
        
        # Check other categories
        scores = {}
        for category, keywords in self.categories.items():
            if category == 'Income':
                continue
            
            score = sum(1 for keyword in keywords if keyword in description_lower)
            if score > 0:
                scores[category] = score
        
        # Return category with highest score
        if scores:
            category = max(scores, key=scores.get)
            self.merchant_cache[description_lower] = category
            return category
        
        # Default category
        category = 'Other'
        self.merchant_cache[description_lower] = category
        return category
    
    def categorize_all(self, transactions: List[Transaction]) -> Dict[str, List[Transaction]]:
        """Categorize all transactions"""
        categorized = {}
        
        for transaction in transactions:
            category = self.categorize_transaction(transaction)
            transaction.category = category
            
            if category not in categorized:
                categorized[category] = []
            
            categorized[category].append(transaction)
        
        return categorized
    
    def get_category_summary(self, transactions: List[Transaction]) -> Dict[str, Dict]:
        """Get summary by category"""
        categorized = self.categorize_all(transactions)
        summary = {}
        
        for category, trans_list in categorized.items():
            total = sum(t.amount for t in trans_list)
            count = len(trans_list)
            avg = total / count if count > 0 else 0
            
            summary[category] = {
                'total': total,
                'count': count,
                'average': avg,
                'percentage': 0  # Will calculate after
            }
        
        # Calculate percentages
        grand_total = sum(s['total'] for s in summary.values())
        for category in summary:
            if grand_total > 0:
                summary[category]['percentage'] = (summary[category]['total'] / grand_total) * 100
        
        return summary
```

### 8.3 Export and Reporting
Create `src/core/reporting.py`:
```python
import pandas as pd
from datetime import datetime
from pathlib import Path
import json
from typing import List, Dict, Any
from reportlab.lib import colors
from reportlab.lib.pagesizes import letter, A4
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import inch

class ReportGenerator:
    """Generate various report formats"""
    
    def __init__(self):
        self.styles = getSampleStyleSheet()
        self._init_custom_styles()
    
    def _init_custom_styles(self):
        """Initialize custom PDF styles"""
        self.styles.add(ParagraphStyle(
            name='CustomTitle',
            parent=self.styles['Heading1'],
            fontSize=24,
            textColor=colors.HexColor('#1f77b4'),
            spaceAfter=30
        ))
    
    def generate_pdf_report(self, transactions: List, output_path: str, 
                           summary: Dict[str, Any] = None):
        """Generate PDF report"""
        doc = SimpleDocTemplate(output_path, pagesize=letter)
        story = []
        
        # Title
        title = Paragraph("Yomnai Financial Analysis Report", self.styles['CustomTitle'])
        story.append(title)
        
        # Date
        date_text = Paragraph(f"Generated: {datetime.now().strftime('%B %d, %Y')}", 
                            self.styles['Normal'])
        story.append(date_text)
        story.append(Spacer(1, 12))
        
        # Summary section
        if summary:
            story.append(Paragraph("Executive Summary", self.styles['Heading2']))
            
            summary_data = [
                ['Metric', 'Value'],
                ['Total Transactions', str(summary.get('total_transactions', 0))],
                ['Total Income', f"${summary.get('total_income', 0):,.2f}"],
                ['Total Expenses', f"${summary.get('total_expenses', 0):,.2f}"],
                ['Net Flow', f"${summary.get('net_flow', 0):,.2f}"],
                ['Date Range', f"{summary.get('start_date', '')} to {summary.get('end_date', '')}"]
            ]
            
            summary_table = Table(summary_data)
            summary_table.setStyle(TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
                ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
                ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
                ('FONTSIZE', (0, 0), (-1, 0), 14),
                ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
                ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
                ('GRID', (0, 0), (-1, -1), 1, colors.black)
            ]))
            
            story.append(summary_table)
            story.append(Spacer(1, 20))
        
        # Top transactions
        story.append(Paragraph("Top Transactions", self.styles['Heading2']))
        
        # Convert transactions to table data
        trans_data = [['Date', 'Description', 'Amount', 'Type']]
        
        for trans in transactions[:10]:  # Top 10
            trans_data.append([
                trans.date.strftime('%Y-%m-%d'),
                trans.description[:30],
                f"${trans.amount:.2f}",
                trans.type.value
            ])
        
        trans_table = Table(trans_data)
        trans_table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 12),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, 1), (-1, -1), colors.white),
            ('GRID', (0, 0), (-1, -1), 1, colors.black)
        ]))
        
        story.append(trans_table)
        
        # Build PDF
        doc.build(story)
    
    def generate_excel_report(self, transactions: List, output_path: str,
                            category_summary: Dict = None):
        """Generate Excel report with multiple sheets"""
        with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
            # Transactions sheet
            df_trans = pd.DataFrame([t.to_dict() for t in transactions])
            df_trans.to_excel(writer, sheet_name='Transactions', index=False)
            
            # Summary sheet
            if category_summary:
                df_summary = pd.DataFrame.from_dict(category_summary, orient='index')
                df_summary.to_excel(writer, sheet_name='Category Summary')
            
            # Monthly breakdown
            df_trans['date'] = pd.to_datetime(df_trans['date'])
            df_trans['month'] = df_trans['date'].dt.to_period('M')
            
            monthly = df_trans.groupby(['month', 'type'])['amount'].sum().unstack(fill_value=0)
            monthly.to_excel(writer, sheet_name='Monthly Breakdown')
            
            # Format worksheets
            for sheet_name in writer.sheets:
                worksheet = writer.sheets[sheet_name]
                
                # Adjust column widths
                for column in worksheet.columns:
                    max_length = 0
                    column_letter = column[0].column_letter
                    
                    for cell in column:
                        try:
                            if len(str(cell.value)) > max_length:
                                max_length = len(str(cell.value))
                        except:
                            pass
                    
                    adjusted_width = min(max_length + 2, 50)
                    worksheet.column_dimensions[column_letter].width = adjusted_width
    
    def generate_json_export(self, transactions: List, output_path: str,
                           metadata: Dict = None):
        """Generate JSON export"""
        export_data = {
            "export_date": datetime.now().isoformat(),
            "version": "1.0",
            "metadata": metadata or {},
            "transactions": [t.to_dict() for t in transactions]
        }
        
        with open(output_path, 'w') as f:
            json.dump(export_data, f, indent=2, default=str)
    
    def generate_markdown_report(self, transactions: List, output_path: str,
                                chat_history: List = None):
        """Generate Markdown report"""
        md_content = "# Yomnai Financial Analysis Report\n\n"
        md_content += f"Generated: {datetime.now().strftime('%B %d, %Y')}\n\n"
        
        # Summary statistics
        df = pd.DataFrame([t.to_dict() for t in transactions])
        
        md_content += "## Summary Statistics\n\n"
        md_content += f"- Total Transactions: {len(transactions)}\n"
        md_content += f"- Date Range: {df['date'].min()} to {df['date'].max()}\n"
        md_content += f"- Total Debits: ${df[df['type']=='debit']['amount'].sum():,.2f}\n"
        md_content += f"- Total Credits: ${df[df['type']=='credit']['amount'].sum():,.2f}\n\n"
        
        # Top transactions table
        md_content += "## Top 10 Transactions\n\n"
        md_content += "| Date | Description | Amount | Type |\n"
        md_content += "|------|-------------|--------|------|\n"
        
        for trans in sorted(transactions, key=lambda x: x.amount, reverse=True)[:10]:
            md_content += f"| {trans.date.strftime('%Y-%m-%d')} "
            md_content += f"| {trans.description[:30]} "
            md_content += f"| ${trans.amount:.2f} "
            md_content += f"| {trans.type.value} |\n"
        
        # Chat history if available
        if chat_history:
            md_content += "\n## Analysis Q&A History\n\n"
            
            for entry in chat_history:
                if entry['role'] == 'user':
                    md_content += f"### Q: {entry['content']}\n\n"
                else:
                    md_content += f"**A:** {entry['content']}\n\n"
        
        # Write to file
        with open(output_path, 'w') as f:
            f.write(md_content)
```

### 8.4 Performance Caching
Create `src/core/cache.py`:
```python
import pickle
import hashlib
from pathlib import Path
from datetime import datetime, timedelta
from typing import Any, Optional
import logging

logger = logging.getLogger(__name__)

class CacheManager:
    """Manage application caching"""
    
    def __init__(self, cache_dir: str = "./data/cache"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        self.cache_ttl = timedelta(hours=24)  # Default TTL
    
    def _get_cache_key(self, key: str) -> str:
        """Generate cache key"""
        return hashlib.md5(key.encode()).hexdigest()
    
    def _get_cache_path(self, cache_key: str) -> Path:
        """Get cache file path"""
        return self.cache_dir / f"{cache_key}.cache"
    
    def get(self, key: str) -> Optional[Any]:
        """Get cached value"""
        cache_key = self._get_cache_key(key)
        cache_path = self._get_cache_path(cache_key)
        
        if not cache_path.exists():
            return None
        
        try:
            with open(cache_path, 'rb') as f:
                cache_data = pickle.load(f)
            
            # Check expiration
            if datetime.now() - cache_data['timestamp'] > self.cache_ttl:
                cache_path.unlink()  # Delete expired cache
                return None
            
            logger.debug(f"Cache hit for key: {key[:20]}...")
            return cache_data['value']
            
        except Exception as e:
            logger.error(f"Error reading cache: {str(e)}")
            return None
    
    def set(self, key: str, value: Any, ttl: Optional[timedelta] = None):
        """Set cached value"""
        cache_key = self._get_cache_key(key)
        cache_path = self._get_cache_path(cache_key)
        
        cache_data = {
            'timestamp': datetime.now(),
            'value': value,
            'ttl': ttl or self.cache_ttl
        }
        
        try:
            with open(cache_path, 'wb') as f:
                pickle.dump(cache_data, f)
            
            logger.debug(f"Cached value for key: {key[:20]}...")
            
        except Exception as e:
            logger.error(f"Error writing cache: {str(e)}")
    
    def clear(self):
        """Clear all cache"""
        for cache_file in self.cache_dir.glob("*.cache"):
            try:
                cache_file.unlink()
            except:
                pass
        
        logger.info("Cache cleared")
    
    def clear_expired(self):
        """Clear expired cache entries"""
        cleared = 0
        
        for cache_file in self.cache_dir.glob("*.cache"):
            try:
                with open(cache_file, 'rb') as f:
                    cache_data = pickle.load(f)
                
                if datetime.now() - cache_data['timestamp'] > cache_data.get('ttl', self.cache_ttl):
                    cache_file.unlink()
                    cleared += 1
                    
            except:
                pass
        
        if cleared > 0:
            logger.info(f"Cleared {cleared} expired cache entries")
    
    def get_cache_size(self) -> Dict[str, Any]:
        """Get cache statistics"""
        cache_files = list(self.cache_dir.glob("*.cache"))
        total_size = sum(f.stat().st_size for f in cache_files)
        
        return {
            'num_entries': len(cache_files),
            'total_size_mb': total_size / (1024 * 1024),
            'cache_dir': str(self.cache_dir)
        }
```

### 8.5 Quick Start Script
Create `quickstart.py`:
```python
#!/usr/bin/env python3
"""
Yomnai Quick Start Script
Automatically sets up and launches the application
"""

import os
import sys
import subprocess
from pathlib import Path

def check_python_version():
    """Check Python version"""
    if sys.version_info < (3, 10):
        print("❌ Python 3.10+ required")
        print(f"Current version: {sys.version}")
        return False
    print(f"✅ Python {sys.version_info.major}.{sys.version_info.minor} detected")
    return True

def check_ollama():
    """Check if Ollama is installed and running"""
    try:
        result = subprocess.run(['ollama', 'list'], capture_output=True, text=True)
        if result.returncode == 0:
            print("✅ Ollama is installed")
            
            # Check if llama3 model is available
            if 'llama3' not in result.stdout.lower():
                print("📥 Pulling llama3 model...")
                subprocess.run(['ollama', 'pull', 'llama3'])
            else:
                print("✅ Llama3 model available")
            
            return True
    except FileNotFoundError:
        print("❌ Ollama not found. Please install from: https://ollama.ai")
        return False

def install_dependencies():
    """Install Python dependencies"""
    print("\n📦 Installing dependencies...")
    
    requirements_file = Path("requirements.txt")
    
    if not requirements_file.exists():
        print("Creating requirements.txt...")
        requirements = """streamlit==1.32.0
pandas==2.2.0
chromadb==0.4.22
langchain==0.1.9
langchain-community==0.0.24
ollama==0.1.7
python-dotenv==1.0.0
numpy==1.26.3
openpyxl==3.1.2
reportlab==4.0.7
psutil==5.9.6
pytest==7.4.3
"""
        requirements_file.write_text(requirements)
    
    # Install dependencies
    subprocess.run([sys.executable, '-m', 'pip', 'install', '-r', 'requirements.txt'])
    print("✅ Dependencies installed")

def create_directories():
    """Create necessary directories"""
    directories = [
        "src", "src/parsers", "src/ai", "src/ui", "src/core",
        "data", "data/temp", "data/cache", "data/sessions",
        "config", "logs", "tests", "docs"
    ]
    
    for dir_path in directories:
        Path(dir_path).mkdir(parents=True, exist_ok=True)
    
    print("✅ Directory structure created")

def create_env_file():
    """Create .env file if not exists"""
    env_file = Path(".env")
    
    if not env_file.exists():
        env_content = """# Ollama Configuration
OLLAMA_HOST=localhost
OLLAMA_PORT=11434
OLLAMA_MODEL=llama3

# ChromaDB Configuration
CHROMA_PERSIST_DIRECTORY=./data/chroma_db
CHROMA_COLLECTION_NAME=bank_transactions

# Application Settings
MAX_FILE_SIZE_MB=50
CHUNK_SIZE=1000
DEBUG_MODE=False
"""
        env_file.write_text(env_content)
        print("✅ Environment file created")

def start_ollama():
    """Start Ollama service"""
    print("\n🚀 Starting Ollama service...")
    
    # Start Ollama in background
    subprocess.Popen(['ollama', 'serve'], 
                    stdout=subprocess.DEVNULL, 
                    stderr=subprocess.DEVNULL)
    
    # Wait a moment for service to start
    import time
    time.sleep(3)
    
    print("✅ Ollama service started")

def launch_app():
    """Launch Streamlit application"""
    print("\n🎯 Launching Yomnai application...")
    print("-" * 50)
    print("The application will open in your browser.")
    print("To stop, press Ctrl+C in this terminal.")
    print("-" * 50)
    
    subprocess.run([
        sys.executable, '-m', 'streamlit', 'run', 'app.py',
        '--server.port=8501',
        '--server.address=localhost'
    ])

def main():
    """Main quick start process"""
    print("=" * 50)
    print("ARGUS QUICK START")
    print("=" * 50)
    
    # Check requirements
    if not check_python_version():
        return
    
    # Create directories
    create_directories()
    
    # Create env file
    create_env_file()
    
    # Install dependencies
    install_dependencies()
    
    # Check and start Ollama
    if check_ollama():
        start_ollama()
    else:
        print("\n⚠️ Ollama not available - AI features will be limited")
        response = input("Continue anyway? (y/n): ")
        if response.lower() != 'y':
            return
    
    # Launch application
    print("\n" + "=" * 50)
    print("Setup complete! Launching Yomnai...")
    print("=" * 50)
    
    try:
        launch_app()
    except KeyboardInterrupt:
        print("\n\n👋 Yomnai stopped. Goodbye!")

if __name__ == "__main__":
    main()
```

## Validation Checklist
- [ ] Advanced query processing works correctly
- [ ] Transaction categorization is accurate
- [ ] PDF report generation succeeds
- [ ] Excel export includes all sheets
- [ ] Caching improves performance
- [ ] Quick start script sets up environment

## Success Criteria
- Advanced queries provide meaningful insights
- Categories cover 90%+ of transactions
- Reports are professional and complete
- Cache reduces repeated query time by 50%+
- Quick start works on fresh installation

## Next Phase
Phase 9: Future Features (Optional) - Advanced capabilities for future development