# Financial Agents Architecture

## System Overview

The multi-agent financial analysis system transforms Yomnai from a single-prompt system into a sophisticated network of **12 specialized agents** - 6 for transaction analysis and 6 for financial statement analysis. The system uses a lightweight router for intent classification and hidden "chain of thought" reasoning for improved accuracy.

**LATEST UPDATE (January 2025)**:
- Expanded from 6 to 12 agents (6 transaction + 6 financial statement agents)
- All 12 agents now include detailed 7-step "Think deeply" hidden reasoning prompts
- Enhanced Saudi business context integration (Zakat, VAT, GOSI, Ramadan cycles)
- Dual-dashboard architecture support (transaction analysis + financial statements)

## Core Design Principles

1. **Specialization**: Each agent is an expert in one financial domain
2. **Hidden Thinking**: Multi-step reasoning happens behind the scenes
3. **Saudi Context**: All agents optimized for Saudi Riyal (SAR) transactions
4. **Sequential Processing**: For insights, agents run in order (not parallel)
5. **Privacy First**: All processing remains local

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│                   User Interface                     │
│                    (Streamlit)                       │
└──────────┬────────────────────────┬─────────────────┘
           │                        │
           ▼                        ▼
    ┌─────────────┐          ┌──────────────┐
    │    Chat     │          │   Insights    │
    │  Interface  │          │    Button     │
    └──────┬──────┘          └──────┬───────┘
           │                         │
           ▼                         ▼
    ┌─────────────────────────────────────┐
    │       Query Understander             │
    │    (Gemma3:4b - Fast LLM)           │
    │  • Analyzes intent & extracts filters│
    │  • Retrieves relevant transactions   │
    └──────┬──────────────────────────────┘
           │
           ▼
    ┌─────────────────────────────────────┐
    │     Agent Router & Orchestrator      │
    │    (Routes based on query type)      │
    └──────┬──────────────────────────────┘
           │
           ▼
    ┌─────────────────────────────────────┐
    │    Specialized Agents (Analysis)     │
    ├──────────────────────────────────────┤
    │ • Expense Agent                     │
    │ • Income Agent                      │
    │ • Fee Hunter Agent                  │
    │ • Budget Advisor Agent              │
    │ • Trend Analyst Agent                │
    │ • Transaction Investigator Agent      │
    └─────────────┬───────────────────────┘
                  │
    ┌─────────────▼───────────────────────┐
    │         Ollama LLM (Qwen3)          │
    │    (Analysis & Response Gen)        │
    └──────────────────────────────────────┘
```

## Agent Orchestration

### Chat Mode Flow
```python
class AgentOrchestrator:
    """Orchestrates multi-agent interactions"""
    
    def process_chat_query(self, query: str) -> Dict[str, Any]:
        """Process chat query through appropriate agent"""
        
        # Step 1: Understand query and retrieve data (all in one)
        understanding = self.query_understander.analyze_and_retrieve(query)
        intent = understanding['intent']
        transactions = understanding['transactions']
        
        # Step 2: Route to appropriate agent based on query type
        query_type = intent.get('query_type', 'general')
        agent_category = self._map_query_type_to_agent(query_type)
        
        # Step 3: Generate hidden thinking with context
        thinking_context = self.router.process_with_thinking(
            query=query,
            category=agent_category
        )
        
        # Step 4: Execute primary agent with pre-retrieved data
        primary_agent = self.agents[agent_category]
        result = primary_agent.execute_with_data(
            query=query,
            thinking_context=thinking_context,
            transactions=transactions  # Pre-retrieved data
        )
        
        # Step 5: Return clean result to UI
        return {
            'answer': result['final_answer'],
            'sources': result['sources'],
            'agent_used': agent_category,
            'transactions_retrieved': len(transactions)
        }
```

### Insights Mode Flow (Sequential)
```python
def generate_insights(self) -> Dict[str, Any]:
    """Generate comprehensive insights - SEQUENTIAL execution"""
    
    insights = {}
    
    # Step 1: Expense Analysis (with hidden thinking)
    with st.spinner("Analyzing expenses..."):
        expense_thinking = self._think_about_expenses()  # Hidden
        insights['expenses'] = self.expense_agent.analyze(
            thinking=expense_thinking,
            show_progress=False  # Hide intermediate steps
        )
    
    # Step 2: Income Analysis (after expenses complete)
    with st.spinner("Analyzing income..."):
        income_thinking = self._think_about_income()  # Hidden
        insights['income'] = self.income_agent.analyze(
            thinking=income_thinking,
            show_progress=False
        )
    
    # Step 3: Fee Analysis (after income complete)
    with st.spinner("Detecting fees..."):
        fee_thinking = self._think_about_fees()  # Hidden
        insights['fees'] = self.fee_agent.analyze(
            thinking=fee_thinking,
            show_progress=False
        )
    
    # Return aggregated insights
    return self._format_insights(insights)
```

## Hidden Thinking Process

Each agent performs multi-step reasoning that's hidden from the UI:

```python
class BaseFinancialAgent:
    """Base class for all financial agents"""
    
    def execute_with_data(self, query: str, thinking_context: Dict, 
                         transactions: List[Dict]) -> Dict:
        """Execute with pre-retrieved data and hidden reasoning"""
        
        # Step 1: Internal reasoning (NOT shown to user)
        internal_analysis = self._think_step_by_step(query, thinking_context)
        
        # Step 2: Work with pre-retrieved transactions (no searching needed)
        # Transactions already filtered and retrieved by QueryUnderstander
        
        # Step 3: Analyze pre-retrieved data with specialized prompt (hidden)
        analysis = self._deep_analyze(transactions, internal_analysis)
        
        # Step 4: Generate final answer (shown to user)
        final_answer = self._generate_user_response(analysis, query)
        
        return {
            'final_answer': final_answer,
            'sources': transactions[:5],  # Top 5 for reference
            'thinking': internal_analysis  # Kept but not displayed
        }
```

## Specialized Agents

### Overview
Yomnai employs **12 specialized AI agents** organized into two categories:
- **6 Transaction Agents**: For analyzing bank statements and transaction data
- **6 Financial Statement Agents**: For analyzing balance sheets, income statements, and cash flows

All agents use the Qwen3-14b-32k model for analysis and implement hidden "chain of thought" reasoning for accuracy.

### Transaction Analysis Agents

#### 1. **[Expense Agent](./Expense_Agent.md)**
- **Purpose**: Analyzes business expenses, categorizes spending, identifies cost-saving opportunities
- **Specialties**: Payroll analysis, vendor payments, operational costs, compliance expenses
- **Key Features**:
  - Business expense categorization (Saudi context)
  - Spending pattern detection
  - Cost optimization recommendations
- **Model**: qwen3-14b-32k:latest

#### 2. **[Income Agent](./Income_Agent.md)**
- **Purpose**: Tracks revenue streams, analyzes income stability, identifies payment patterns
- **Specialties**: Client payments, recurring revenue, service income, contract payments
- **Key Features**:
  - Income source identification
  - Revenue trend analysis
  - Payment regularity detection
- **Model**: qwen3-14b-32k:latest

#### 3. **[Fee Hunter Agent](./Fee_Hunter_Agent.md)**
- **Purpose**: Detects and analyzes all fees, service charges, and hidden costs
- **Specialties**: Bank fees, SWIFT charges, VAT, government fees, penalties
- **Key Features**:
  - Comprehensive fee detection
  - Recurring fee identification
  - Annual savings calculation
- **Model**: qwen3-14b-32k:latest

#### 4. **[Budget Advisor Agent](./Budget_Advisor_Agent.md)**
- **Purpose**: Provides budget health assessment and cash flow recommendations
- **Specialties**: Cash flow analysis, operating margins, financial health scoring
- **Key Features**:
  - Health score calculation (0-100)
  - Budget rule application
  - Reserve recommendations
- **Model**: qwen3-14b-32k:latest

#### 5. **[Trend Analyst Agent](./Trend_Analyst_Agent.md)**
- **Purpose**: Identifies patterns, seasonal trends, and provides predictive insights
- **Specialties**: Time-series analysis, spending patterns, growth rate calculations
- **Key Features**:
  - Trend direction analysis
  - Pattern recognition
  - Seasonal cycle detection
- **Model**: qwen3-14b-32k:latest

#### 6. **[Transaction Investigator Agent](./Transaction_Investigator_Agent.md)**
- **Purpose**: Finds specific transactions based on complex search criteria
- **Specialties**: Transaction search, amount matching, merchant identification
- **Key Features**:
  - Multi-parameter search
  - Relevance ranking
  - Flexible query parsing
- **Model**: qwen3-14b-32k:latest

### Financial Statement Analysis Agents

#### 7. **Ratio Analyst Agent**
- **Purpose**: Calculates and interprets key financial ratios
- **Specialties**: Liquidity ratios, profitability ratios, leverage ratios, efficiency ratios
- **Key Features**:
  - Comprehensive ratio calculation
  - Industry benchmark comparison
  - Health assessment scoring
  - Trend analysis
- **Model**: qwen3-14b-32k:latest

#### 8. **Profitability Agent**
- **Purpose**: Analyzes profit trends and margin optimization
- **Specialties**: Gross margin, net margin, EBITDA analysis, cost structure
- **Key Features**:
  - Margin evolution tracking
  - YoY profit comparison
  - Cost optimization opportunities
  - Revenue enhancement strategies
- **Model**: qwen3-14b-32k:latest

#### 9. **Liquidity Agent**
- **Purpose**: Evaluates short-term and long-term financial stability
- **Specialties**: Working capital, cash conversion cycle, solvency assessment
- **Key Features**:
  - Cash position analysis
  - Working capital efficiency
  - Liquidity risk assessment
  - Cash flow forecasting
- **Model**: qwen3-14b-32k:latest

#### 10. **Financial Trend Agent**
- **Purpose**: Identifies patterns in financial performance over time
- **Specialties**: Revenue growth trends, seasonal patterns, performance trajectories
- **Key Features**:
  - CAGR calculations
  - Quarterly performance analysis
  - Trend reversal detection
  - Forward-looking projections
- **Model**: qwen3-14b-32k:latest

#### 11. **Risk Assessment Agent**
- **Purpose**: Evaluates financial risks and compliance status
- **Specialties**: Credit risk, market risk, compliance monitoring (Zakat, VAT, GOSI)
- **Key Features**:
  - Risk scoring (1-10 scale)
  - Covenant monitoring
  - Saudi compliance tracking
  - Early warning indicators
- **Model**: qwen3-14b-32k:latest

#### 12. **Efficiency Agent**
- **Purpose**: Analyzes operational metrics and efficiency ratios
- **Specialties**: Asset turnover, inventory management, productivity metrics
- **Key Features**:
  - Operational efficiency scoring
  - Bottleneck identification
  - Capacity utilization
  - Productivity per employee
- **Model**: qwen3-14b-32k:latest

### Orchestration & Routing

The agent system is coordinated by two key components:

- **[Agent Router System](./Agent_Router_System.md)**: Lightweight query classification using small models (phi3/tinyllama)
- **Agent Orchestrator**: Manages agent execution, data flow, and response aggregation

## Agent Registry

```python
# Transaction Analysis Agents
TRANSACTION_AGENTS = {
    'expense': {
        'class': 'ExpenseAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'spending patterns, expense categories, cost analysis',
        'search_depth': 20,
        'queries_per_analysis': 5
    },
    'income': {
        'class': 'IncomeAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'income sources, salary, deposits, revenue streams',
        'search_depth': 15,
        'queries_per_analysis': 4
    },
    'fee': {
        'class': 'FeeHunterAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'bank fees, service charges, hidden costs',
        'search_depth': 30,
        'queries_per_analysis': 6
    },
    'budget': {
        'class': 'BudgetAdvisorAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'budgeting, savings, financial planning',
        'search_depth': 25,
        'queries_per_analysis': 7
    },
    'trend': {
        'class': 'TrendAnalystAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'patterns, trends, time-based analysis',
        'search_depth': 50,
        'queries_per_analysis': 8
    },
    'transaction': {
        'class': 'TransactionInvestigatorAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'specific transactions, detailed searches',
        'search_depth': 10,
        'queries_per_analysis': 3
    }
}

# Financial Statement Analysis Agents
FINANCIAL_AGENTS = {
    'ratio': {
        'class': 'RatioAnalystAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'financial ratios, liquidity, leverage, profitability metrics',
        'analysis_depth': 'comprehensive'
    },
    'profitability': {
        'class': 'ProfitabilityAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'profit margins, EBITDA, cost optimization, revenue analysis',
        'analysis_depth': 'detailed'
    },
    'liquidity': {
        'class': 'LiquidityAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'cash management, working capital, solvency, cash flow',
        'analysis_depth': 'thorough'
    },
    'financial_trend': {
        'class': 'FinancialTrendAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'growth trends, seasonal patterns, performance trajectory',
        'analysis_depth': 'historical'
    },
    'risk': {
        'class': 'RiskAssessmentAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'risk assessment, compliance, covenants, early warnings',
        'analysis_depth': 'critical'
    },
    'efficiency': {
        'class': 'EfficiencyAgent',
        'model': 'qwen3-14b-32k:latest',
        'expertise': 'operational efficiency, asset utilization, productivity',
        'analysis_depth': 'operational'
    }
}
```

## Implementation Structure

```
src/
├── ai/
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base_agent.py          # Base class for all agents
│   │   ├── router.py               # Query router (thinking generation)
│   │   ├── orchestrator.py         # Agent orchestrator
│   │   ├── expense_agent.py        # Expense specialist
│   │   ├── income_agent.py         # Income specialist
│   │   ├── fee_agent.py            # Fee hunter
│   │   ├── budget_agent.py         # Budget advisor
│   │   ├── trend_agent.py          # Trend analyst
│   │   └── transaction_agent.py    # Transaction investigator
│   ├── query_understander.py      # LLM-based query understanding & retrieval
│   ├── rag_pipeline.py            # Enhanced with multi-agent
│   └── prompts/
│       ├── expense_prompts.py      # Expense agent prompts
│       ├── income_prompts.py       # Income agent prompts
│       └── ... (other prompt files)
```

## Benefits of This Architecture

1. **Accuracy**: Specialized agents give better domain-specific answers
2. **Hidden Complexity**: Users see clean results, not the reasoning
3. **Comprehensive Analysis**: Multiple queries gather complete data
4. **Saudi Context**: Each agent understands SAR and local banking
5. **Maintainable**: Easy to add new agents or modify existing ones
6. **Performance**: Router uses small model for fast classification

## Integration Example

```python
# In app.py - Enhanced chat interface
def render_chat_interface():
    """Render chat interface with multi-agent support"""
    
    if prompt := st.chat_input("Ask about your transactions..."):
        # Add user message
        with st.chat_message("user"):
            st.markdown(prompt)
        
        # Process through multi-agent system
        with st.chat_message("assistant"):
            # This spinner is all the user sees
            with st.spinner("Analyzing..."):
                # Hidden: Router → Thinking → Agent → Multiple Searches
                response = st.session_state.orchestrator.process_chat_query(prompt)
            
            # Show only the final answer
            st.markdown(response['answer'])
            
            # Subtle indicator of which agent responded
            st.caption(f"Analysis by: {response['agent_used'].title()} Agent")

# In app.py - Enhanced insights with sequential processing
def render_insights_dashboard():
    """Sequential agent processing for comprehensive insights"""
    
    if st.button("🔮 Generate Financial Insights", type="primary"):
        insights_container = st.container()
        
        with insights_container:
            # Sequential processing with status updates
            status = st.empty()
            results = {}
            
            # Process each agent in sequence
            status.info("📊 Analyzing expenses...")
            results['expenses'] = st.session_state.expense_agent.analyze()
            
            status.info("💰 Analyzing income...")
            results['income'] = st.session_state.income_agent.analyze()
            
            status.info("🏦 Detecting fees...")
            results['fees'] = st.session_state.fee_agent.analyze()
            
            status.success("✅ Analysis complete!")
            
            # Display results
            col1, col2, col3 = st.columns(3)
            
            with col1:
                st.subheader("💸 Expense Analysis")
                st.markdown(results['expenses']['summary'])
            
            with col2:
                st.subheader("💰 Income Analysis")
                st.markdown(results['income']['summary'])
            
            with col3:
                st.subheader("🏦 Fees & Charges")
                st.markdown(results['fees']['summary'])
```