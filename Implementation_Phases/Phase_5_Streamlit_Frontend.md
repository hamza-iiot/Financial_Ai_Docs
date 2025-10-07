# Phase 5: Streamlit Frontend

## Overview
Build the complete Streamlit user interface with file upload, transaction viewing, and chat functionality.

## Tasks

### 5.1 Main Application UI
Update `app.py`:
```python
import streamlit as st
import pandas as pd
from datetime import datetime
import tempfile
import os
from pathlib import Path

from config.settings import settings
from config.logging_config import setup_logging
from src.parsers.csv_parser import BankStatementParser
from src.parsers.validator import TransactionValidator
from src.ai.vector_store import VectorStoreManager
from src.ai.ollama_client import OllamaClient
from src.ai.rag_pipeline import RAGPipeline
from src.ui.components import (
    render_header,
    render_file_upload,
    render_transaction_viewer,
    render_chat_interface,
    render_insights_dashboard
)

# Initialize logging
logger = setup_logging(settings.DEBUG_MODE)

# Page configuration
st.set_page_config(
    page_title="Yomnai - Private Financial Analyst",
    page_icon="👁️",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Initialize session state
def init_session_state():
    """Initialize session state variables"""
    if 'transactions' not in st.session_state:
        st.session_state.transactions = []
    if 'vector_store' not in st.session_state:
        st.session_state.vector_store = None
    if 'rag_pipeline' not in st.session_state:
        st.session_state.rag_pipeline = None
    if 'chat_history' not in st.session_state:
        st.session_state.chat_history = []
    if 'file_uploaded' not in st.session_state:
        st.session_state.file_uploaded = False
    if 'session_id' not in st.session_state:
        st.session_state.session_id = datetime.now().strftime("%Y%m%d_%H%M%S")

def main():
    """Main application entry point"""
    init_session_state()
    
    # Render header
    render_header()
    
    # Sidebar for file upload and controls
    with st.sidebar:
        st.header("📁 Data Upload")
        
        uploaded_file = render_file_upload()
        
        if uploaded_file:
            process_uploaded_file(uploaded_file)
        
        # Display session info
        if st.session_state.file_uploaded:
            st.success("✅ File loaded successfully!")
            st.info(f"📊 {len(st.session_state.transactions)} transactions")
            
            if st.button("🗑️ Clear Data"):
                clear_session_data()
                st.rerun()
        
        # Settings section
        with st.expander("⚙️ Settings"):
            st.slider("Results per query", 5, 20, 10, key="n_results")
            st.slider("Temperature", 0.0, 1.0, 0.1, key="temperature")
    
    # Main content area
    if not st.session_state.file_uploaded:
        # Welcome screen
        render_welcome_screen()
    else:
        # Main application tabs
        tabs = st.tabs(["💬 Chat", "📊 Transactions", "🔍 Insights", "📈 Analytics"])
        
        with tabs[0]:
            render_chat_interface()
        
        with tabs[1]:
            render_transaction_viewer()
        
        with tabs[2]:
            render_insights_dashboard()
        
        with tabs[3]:
            render_analytics()

def process_uploaded_file(uploaded_file):
    """Process the uploaded CSV file"""
    with st.spinner("Processing your bank statement..."):
        try:
            # Save uploaded file temporarily
            with tempfile.NamedTemporaryFile(delete=False, suffix='.csv') as tmp_file:
                tmp_file.write(uploaded_file.getvalue())
                tmp_path = tmp_file.name
            
            # Parse CSV
            parser = BankStatementParser()
            transactions = parser.parse_file(tmp_path)
            
            # Validate transactions
            validator = TransactionValidator()
            validation_result = validator.validate_transactions(transactions)
            
            if not validation_result['valid']:
                st.error("Validation failed:")
                for error in validation_result['errors']:
                    st.error(f"• {error}")
                return
            
            # Show warnings if any
            if validation_result['warnings']:
                with st.expander("⚠️ Warnings"):
                    for warning in validation_result['warnings']:
                        st.warning(warning)
            
            # Initialize vector store
            if st.session_state.vector_store is None:
                st.session_state.vector_store = VectorStoreManager()
            
            # Add transactions to vector store
            st.session_state.vector_store.add_transactions(
                transactions,
                st.session_state.session_id
            )
            
            # Initialize RAG pipeline
            if st.session_state.rag_pipeline is None:
                ollama_client = OllamaClient()
                st.session_state.rag_pipeline = RAGPipeline(
                    st.session_state.vector_store,
                    ollama_client
                )
            
            # Store in session state
            st.session_state.transactions = transactions
            st.session_state.file_uploaded = True
            
            # Clean up temp file
            os.unlink(tmp_path)
            
            st.success(f"Successfully loaded {len(transactions)} transactions!")
            st.balloons()
            
        except Exception as e:
            st.error(f"Error processing file: {str(e)}")
            logger.error(f"File processing error: {str(e)}")

def clear_session_data():
    """Clear all session data"""
    if st.session_state.vector_store:
        st.session_state.vector_store.clear_collection(st.session_state.session_id)
    
    st.session_state.transactions = []
    st.session_state.chat_history = []
    st.session_state.file_uploaded = False
    st.session_state.session_id = datetime.now().strftime("%Y%m%d_%H%M%S")

def render_welcome_screen():
    """Render welcome screen when no file is uploaded"""
    st.markdown("""
    ## Welcome to Yomnai - Your Private Financial Analyst 👁️
    
    Yomnai helps you understand your finances through natural conversation. 
    All processing happens locally on your computer, ensuring complete privacy.
    
    ### 🚀 Getting Started
    1. Upload your bank statement CSV file using the sidebar
    2. Wait for processing (usually takes a few seconds)
    3. Start asking questions about your transactions!
    
    ### 💡 Example Questions You Can Ask:
    - "What were my largest expenses last month?"
    - "How much did I spend on groceries?"
    - "Show me all income transactions"
    - "What's my average daily spending?"
    - "Find all transactions over $500"
    
    ### 🔒 Privacy First
    - ✅ All data stays on your computer
    - ✅ No cloud uploads
    - ✅ No data tracking
    - ✅ Completely private and secure
    
    **Ready? Upload your bank statement to begin!** 👈
    """)

def render_analytics():
    """Render analytics dashboard"""
    st.header("📈 Financial Analytics")
    
    if not st.session_state.transactions:
        st.info("No data available for analytics")
        return
    
    # Convert transactions to DataFrame
    df = pd.DataFrame([t.to_dict() for t in st.session_state.transactions])
    df['date'] = pd.to_datetime(df['date'])
    df['month'] = df['date'].dt.to_period('M')
    
    # Summary metrics
    col1, col2, col3, col4 = st.columns(4)
    
    with col1:
        total_income = df[df['type'] == 'credit']['amount'].sum()
        st.metric("Total Income", f"${total_income:,.2f}")
    
    with col2:
        total_expenses = df[df['type'] == 'debit']['amount'].sum()
        st.metric("Total Expenses", f"${total_expenses:,.2f}")
    
    with col3:
        net_flow = total_income - total_expenses
        st.metric("Net Flow", f"${net_flow:,.2f}")
    
    with col4:
        avg_transaction = df['amount'].mean()
        st.metric("Avg Transaction", f"${avg_transaction:,.2f}")
    
    # Charts
    st.subheader("Transaction Trends")
    
    # Daily spending chart
    daily_spending = df[df['type'] == 'debit'].groupby(df['date'].dt.date)['amount'].sum()
    st.line_chart(daily_spending)
    
    # Transaction type distribution
    st.subheader("Transaction Distribution")
    type_counts = df['type'].value_counts()
    st.bar_chart(type_counts)
    
    # Top transactions
    st.subheader("Top Transactions")
    top_expenses = df[df['type'] == 'debit'].nlargest(10, 'amount')[['date', 'description', 'amount']]
    st.dataframe(top_expenses, use_container_width=True)

if __name__ == "__main__":
    main()
```

### 5.2 UI Components Module
Create `src/ui/components.py`:
```python
import streamlit as st
import pandas as pd
from typing import List, Optional
from datetime import datetime

def render_header():
    """Render application header"""
    col1, col2 = st.columns([3, 1])
    
    with col1:
        st.title("👁️ Yomnai - Your Private Financial Analyst")
    
    with col2:
        st.markdown(
            f"**Session:** {datetime.now().strftime('%H:%M')}"
        )
    
    st.markdown("---")

def render_file_upload():
    """Render file upload component"""
    uploaded_file = st.file_uploader(
        "Choose a CSV file",
        type=['csv'],
        help="Upload your bank statement in CSV format"
    )
    
    if uploaded_file:
        # File size check
        file_size_mb = uploaded_file.size / (1024 * 1024)
        if file_size_mb > 50:
            st.error(f"File too large ({file_size_mb:.1f} MB). Maximum size is 50 MB.")
            return None
    
    return uploaded_file

def render_transaction_viewer():
    """Render transaction data viewer"""
    st.header("📊 Transaction Data")
    
    if not st.session_state.transactions:
        st.info("No transactions loaded")
        return
    
    # Convert to DataFrame
    df = pd.DataFrame([t.to_dict() for t in st.session_state.transactions])
    
    # Add filters
    col1, col2, col3 = st.columns(3)
    
    with col1:
        type_filter = st.selectbox(
            "Transaction Type",
            ["All", "Debit", "Credit"]
        )
    
    with col2:
        min_amount = st.number_input("Min Amount", value=0.0)
    
    with col3:
        max_amount = st.number_input("Max Amount", value=10000.0)
    
    # Apply filters
    filtered_df = df.copy()
    
    if type_filter != "All":
        filtered_df = filtered_df[filtered_df['type'] == type_filter.lower()]
    
    filtered_df = filtered_df[
        (filtered_df['amount'] >= min_amount) & 
        (filtered_df['amount'] <= max_amount)
    ]
    
    # Display options
    col1, col2 = st.columns([2, 1])
    
    with col1:
        search_term = st.text_input("🔍 Search transactions", "")
        if search_term:
            filtered_df = filtered_df[
                filtered_df['description'].str.contains(
                    search_term, case=False, na=False
                )
            ]
    
    with col2:
        show_raw = st.checkbox("Show raw data", False)
    
    # Display data
    if show_raw:
        st.dataframe(filtered_df, use_container_width=True)
    else:
        # Format for display
        display_df = filtered_df.copy()
        display_df['date'] = pd.to_datetime(display_df['date']).dt.strftime('%Y-%m-%d')
        display_df['amount'] = display_df['amount'].apply(lambda x: f"${x:,.2f}")
        display_df['balance'] = display_df['balance'].apply(
            lambda x: f"${x:,.2f}" if pd.notna(x) else "-"
        )
        
        st.dataframe(
            display_df[['date', 'description', 'type', 'amount', 'balance']],
            use_container_width=True,
            hide_index=True
        )
    
    # Summary statistics
    with st.expander("📈 Summary Statistics"):
        col1, col2 = st.columns(2)
        
        with col1:
            st.metric("Total Transactions", len(filtered_df))
            st.metric("Total Amount", f"${filtered_df['amount'].sum():,.2f}")
        
        with col2:
            st.metric("Average Amount", f"${filtered_df['amount'].mean():,.2f}")
            st.metric("Median Amount", f"${filtered_df['amount'].median():,.2f}")

def render_chat_interface():
    """Render chat interface"""
    st.header("💬 Chat with Your Data")
    
    if not st.session_state.rag_pipeline:
        st.info("Please upload a file to start chatting")
        return
    
    # Display chat history
    for message in st.session_state.chat_history:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])
            
            # Show sources if available
            if message.get("sources"):
                with st.expander("📎 Sources"):
                    for source in message["sources"]:
                        st.text(f"• {source.get('description', 'N/A')} - ${source.get('amount', 0):.2f}")
    
    # Chat input
    if prompt := st.chat_input("Ask about your transactions..."):
        # Add user message to chat
        st.session_state.chat_history.append({
            "role": "user",
            "content": prompt
        })
        
        # Display user message
        with st.chat_message("user"):
            st.markdown(prompt)
        
        # Get AI response
        with st.chat_message("assistant"):
            with st.spinner("Thinking..."):
                response = st.session_state.rag_pipeline.process_query(
                    prompt,
                    n_results=st.session_state.get('n_results', 10)
                )
                
                # Display response
                st.markdown(response['answer'])
                
                # Show sources
                if response.get('sources'):
                    with st.expander("📎 Sources"):
                        for source in response['sources'][:5]:
                            st.text(
                                f"• {source.get('description', 'N/A')} - "
                                f"${source.get('amount', 0):.2f}"
                            )
                
                # Add to chat history
                st.session_state.chat_history.append({
                    "role": "assistant",
                    "content": response['answer'],
                    "sources": response.get('sources', [])[:5]
                })
    
    # Quick action buttons
    st.markdown("### 🚀 Quick Questions")
    col1, col2, col3 = st.columns(3)
    
    with col1:
        if st.button("📊 Summary"):
            process_quick_query("Give me a summary of my transactions")
    
    with col2:
        if st.button("💸 Top Expenses"):
            process_quick_query("What were my top 5 expenses?")
    
    with col3:
        if st.button("💰 Income Sources"):
            process_quick_query("What were my sources of income?")

def process_quick_query(query: str):
    """Process a quick query button click"""
    st.session_state.chat_history.append({
        "role": "user",
        "content": query
    })
    st.rerun()

def render_insights_dashboard():
    """Render insights dashboard"""
    st.header("🔍 Financial Insights")
    
    if not st.session_state.rag_pipeline:
        st.info("Please upload a file to see insights")
        return
    
    with st.spinner("Generating insights..."):
        insights = st.session_state.rag_pipeline.get_insights()
    
    # Display insights in cards
    if insights.get('top_expenses'):
        st.subheader("💸 Top Expenses")
        st.info(insights['top_expenses']['answer'])
    
    if insights.get('income_sources'):
        st.subheader("💰 Income Analysis")
        st.success(insights['income_sources']['answer'])
    
    if insights.get('fees_and_charges'):
        st.subheader("🏦 Bank Fees & Charges")
        st.warning(insights['fees_and_charges']['answer'])
    
    # Generate summary
    if st.button("Generate Detailed Summary"):
        with st.spinner("Creating summary..."):
            summary = st.session_state.rag_pipeline.generate_summary()
            st.markdown("### 📝 Transaction Summary")
            st.markdown(summary)
```

### 5.3 Styling and Theme
Create `src/ui/styles.py`:
```python
def get_custom_css():
    """Get custom CSS for Streamlit app"""
    return """
    <style>
    /* Main container */
    .main {
        padding: 2rem;
    }
    
    /* Headers */
    h1 {
        color: #1f77b4;
        border-bottom: 2px solid #1f77b4;
        padding-bottom: 0.5rem;
    }
    
    h2 {
        color: #2c3e50;
        margin-top: 2rem;
    }
    
    /* Chat messages */
    .stChatMessage {
        padding: 1rem;
        border-radius: 10px;
        margin-bottom: 1rem;
    }
    
    /* Metrics */
    [data-testid="metric-container"] {
        background-color: #f0f2f6;
        padding: 1rem;
        border-radius: 10px;
        border: 1px solid #e0e0e0;
    }
    
    /* File uploader */
    [data-testid="stFileUploader"] {
        background-color: #f8f9fa;
        padding: 2rem;
        border-radius: 10px;
        border: 2px dashed #1f77b4;
    }
    
    /* Buttons */
    .stButton > button {
        background-color: #1f77b4;
        color: white;
        border-radius: 5px;
        padding: 0.5rem 1rem;
        border: none;
        transition: all 0.3s;
    }
    
    .stButton > button:hover {
        background-color: #145a8d;
        transform: translateY(-2px);
    }
    
    /* Success messages */
    .success {
        background-color: #d4edda;
        border-color: #c3e6cb;
        color: #155724;
        padding: 1rem;
        border-radius: 5px;
    }
    
    /* Error messages */
    .error {
        background-color: #f8d7da;
        border-color: #f5c6cb;
        color: #721c24;
        padding: 1rem;
        border-radius: 5px;
    }
    
    /* Sidebar */
    .css-1d391kg {
        background-color: #f8f9fa;
    }
    
    /* Expander */
    .streamlit-expanderHeader {
        background-color: #f0f2f6;
        border-radius: 5px;
    }
    
    /* Tables */
    .dataframe {
        font-size: 14px;
    }
    
    .dataframe thead th {
        background-color: #1f77b4;
        color: white;
        font-weight: bold;
    }
    
    .dataframe tbody tr:hover {
        background-color: #f0f2f6;
    }
    
    /* Loading spinner */
    .stSpinner > div {
        border-color: #1f77b4;
    }
    
    /* Chat input */
    .stChatInput {
        border: 2px solid #1f77b4;
        border-radius: 10px;
    }
    
    /* Tabs */
    .stTabs [data-baseweb="tab-list"] {
        gap: 2rem;
    }
    
    .stTabs [data-baseweb="tab"] {
        font-size: 16px;
        font-weight: 500;
    }
    </style>
    """

def apply_custom_theme():
    """Apply custom theme to Streamlit app"""
    import streamlit as st
    st.markdown(get_custom_css(), unsafe_allow_html=True)
```

### 5.4 Session Management
Create `src/ui/session_manager.py`:
```python
import streamlit as st
from typing import Any, Dict
import json
from pathlib import Path
from datetime import datetime

class SessionManager:
    """Manage user sessions and data persistence"""
    
    def __init__(self, session_dir: str = "./data/sessions"):
        self.session_dir = Path(session_dir)
        self.session_dir.mkdir(parents=True, exist_ok=True)
    
    def save_session(self, session_id: str, data: Dict[str, Any]):
        """Save session data to disk"""
        session_file = self.session_dir / f"{session_id}.json"
        
        session_data = {
            "session_id": session_id,
            "timestamp": datetime.now().isoformat(),
            "data": data
        }
        
        with open(session_file, 'w') as f:
            json.dump(session_data, f, indent=2, default=str)
    
    def load_session(self, session_id: str) -> Dict[str, Any]:
        """Load session data from disk"""
        session_file = self.session_dir / f"{session_id}.json"
        
        if not session_file.exists():
            return {}
        
        with open(session_file, 'r') as f:
            session_data = json.load(f)
        
        return session_data.get('data', {})
    
    def list_sessions(self) -> List[Dict[str, str]]:
        """List all available sessions"""
        sessions = []
        
        for session_file in self.session_dir.glob("*.json"):
            with open(session_file, 'r') as f:
                data = json.load(f)
                sessions.append({
                    "session_id": data['session_id'],
                    "timestamp": data['timestamp']
                })
        
        return sorted(sessions, key=lambda x: x['timestamp'], reverse=True)
    
    def delete_session(self, session_id: str):
        """Delete a session"""
        session_file = self.session_dir / f"{session_id}.json"
        if session_file.exists():
            session_file.unlink()
    
    def export_chat_history(self, chat_history: List[Dict]) -> str:
        """Export chat history as markdown"""
        markdown = "# Yomnai Chat History\n\n"
        markdown += f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
        
        for message in chat_history:
            role = message['role'].capitalize()
            content = message['content']
            markdown += f"## {role}\n{content}\n\n"
        
        return markdown
```

## Validation Checklist
- [ ] Streamlit app launches successfully
- [ ] File upload accepts CSV files
- [ ] Transactions display in formatted table
- [ ] Chat interface accepts queries
- [ ] Responses appear in chat window
- [ ] Quick action buttons work
- [ ] Analytics charts render correctly

## Success Criteria
- Clean, intuitive user interface
- Smooth file upload experience
- Real-time chat responses
- Clear data visualization
- Responsive design
- Session persistence

## Next Phase
Phase 6: Core Features Implementation - Connecting all components and implementing the complete workflow