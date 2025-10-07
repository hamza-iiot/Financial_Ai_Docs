# Yomnai AI Agents - Complete Documentation

**Last Updated:** October 5, 2025

## Overview

Yomnai uses a **12-agent multi-agent system** with **dual-mode execution** for optimized performance:

- **6 Transaction Agents** - Analyze bank transactions (CSV/Excel files)
- **6 Financial Statement Agents** - Analyze financial statements (Balance Sheet, P&L, Cash Flow)

All agents use **Ollama** for local LLM inference and **ChromaDB** for vector storage, ensuring complete data privacy.

---

## 🚀 Dual-Mode Execution (NEW - October 5, 2025)

**Status:** ✅ Infrastructure Complete | ⏳ 2 of 12 agents updated

All agents now support two execution modes:

### INSIGHTS MODE (Deep Analysis)
- **When**: First analysis or "Generate Insights" button
- **Thinking**: ✅ Enabled (`think=true`)
- **Time**: ~50 seconds per agent
- **Purpose**: Comprehensive analysis with structured reasoning
- **Result**: Cached for 24 hours
- **Use Case**: Initial deep dive, comprehensive reports

### CHAT MODE (Fast Responses)
- **When**: Conversational questions after insights generated
- **Thinking**: ❌ Disabled (`think=false`)
- **Time**: ~5-10 seconds
- **Purpose**: Quick answers using cached context
- **Result**: Real-time responses
- **Use Case**: Follow-up questions, specific queries

### Agent Method Signature (Updated)

```python
def execute(self, query: str, thinking_context: Dict = None,
            mode: str = "insights", cached_analysis: Dict = None) -> Dict[str, Any]:
    """
    Execute agent analysis with dual-mode support

    Args:
        query: User query
        thinking_context: Context for thinking
        mode: "insights" (deep) or "chat" (fast)
        cached_analysis: Previous analysis (for chat mode)

    Returns:
        Dict with final_answer, sources, statistics, mode
    """
```

### Implementation Status

- ✅ `expense_agent.py` - Full dual-mode
- ✅ `income_agent.py` - Full dual-mode
- ⏳ `fee_agent.py` - Pending
- ⏳ `budget_agent.py` - Pending
- ⏳ `trend_agent.py` - Pending
- ⏳ `transaction_agent.py` - Pending
- ⏳ All 6 financial agents - Pending

---

## System Architecture

```
User Query
    ↓
Router (Gemma3:4b) ← Fast intent classification
    ↓
Agent Orchestrator
    ↓
[Check SessionCache for cached insights]
    ↓
Specific Agent (Qwen3-14B-32k)
    ├─ INSIGHTS MODE: Deep analysis with think=true (~50s)
    └─ CHAT MODE: Fast response with think=false (~5s)
    ↓
LLM-Generated Response
```

---

# PART 1: TRANSACTION AGENTS (6)

These agents analyze bank transaction data from CSV/Excel files.

---

## 1. ExpenseAgent 💰

### Purpose
Specialized agent for business expense analysis and categorization with Saudi Arabian banking context.

### System Prompt
```
You are a Business Expense Analysis Expert specializing in Saudi Arabian corporate banking and financial management.

Your expertise includes:
1. Business Expense Categorization (Payroll, Operations, Compliance, Vendors)
2. Cash Flow Impact Assessment
3. Cost Optimization and Reduction Strategies
4. Recurring vs One-Time Expense Analysis
5. Saudi Regulatory Compliance Costs (GOSI, QIWA, Zakat, VAT, SADAD)
6. Vendor and Supplier Payment Analysis
7. International Transfer Cost Management (SWIFT)
8. Banking Fee Analysis

When analyzing business expenses:
- Categorize expenses accurately using Saudi business context
- Identify cash flow impacts and patterns
- Distinguish between fixed, variable, and discretionary costs
- Recognize government compliance payments (GOSI, QIWA, SADAD)
- Analyze SWIFT transfers for international vendor payments
- Calculate cost optimization opportunities
- Consider Zakat (2.5%) and VAT (15%) implications
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Expense breakdown by category
- Key spending patterns and trends
- Cash flow impact analysis
- Cost optimization recommendations
- Compliance cost tracking
```

### Full Analysis Prompt Template
```
Think deeply about this expense-related query in the context of Saudi Arabian transactions.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Time Period Analysis: What timeframe is the business owner asking about?
   - This month/quarter/year?
   - Specific date range?
   - Comparative period (e.g., "compared to last month")?

2. Expense Categories: Which business expense categories are relevant?
   - Payroll & Benefits (رواتب)?
   - Debt Servicing (loan payments, financing)?
   - International Transfers (SWIFT)?
   - Government Compliance (GOSI, QIWA, Zakat, VAT, SADAD)?
   - Vendor Payments (suppliers, contractors)?
   - Banking Fees (charges, commissions)?
   - Operational Costs (rent, utilities, subscriptions)?
   - Technology (software, hardware, cloud)?
   - Marketing & Advertising?
   - Professional Services (legal, accounting, consulting)?

3. Cash Flow Impact: How do these expenses affect business operations?
   - Impact on monthly cash flow?
   - Fixed vs variable costs?
   - Essential vs discretionary spending?
   - Opportunities for cost optimization?

4. Transaction Patterns: What spending patterns to identify?
   - Recurring expenses (monthly payroll, rent)?
   - Scheduled payments (quarterly taxes, annual licenses)?
   - One-time expenses (equipment purchase)?
   - Seasonal variations?

5. Saudi Business Context: What local factors apply?
   - GOSI mandatory contributions (employer obligations)?
   - QIWA labor compliance costs?
   - SADAD recurring payments (utilities, government)?
   - SWIFT international transfers (vendor payments abroad)?
   - Zakat and VAT considerations?

6. Analysis Requirements: What insights are needed?
   - Total spending breakdown by category?
   - Trend identification (increasing/decreasing)?
   - Unusual or outlier transactions?
   - Savings opportunities?
   - Budget compliance?

7. Currency and Format: Present in SAR with business-appropriate formatting

Provide detailed reasoning:
```

### Expense Categories
```python
{
    'payroll': ['payroll', 'salary', 'wage', 'employee', 'staff', 'compensation', 'رواتب'],
    'debt_servicing': ['loan', 'debt', 'repayment', 'icorp', 'financing', 'credit', 'قرض'],
    'international_transfers': ['swift', 'international', 'overseas', 'foreign', 'wire', 'تحويل دولي'],
    'government_compliance': ['gosi', 'qiwa', 'zakat', 'vat', 'sadad', 'government', 'التأمينات', 'ضريبة'],
    'vendor_payments': ['vendor', 'supplier', 'contractor', 'payment', 'invoice', 'مورد', 'شركة'],
    'banking_fees': ['fee', 'charge', 'commission', 'bank charge', 'رسوم', 'عمولة'],
    'operational': ['rent', 'utility', 'internet', 'phone', 'maintenance', 'إيجار'],
    'technology': ['software', 'hardware', 'tech', 'it', 'cloud', 'server', 'تقنية'],
    'marketing': ['advertising', 'marketing', 'promotion', 'تسويق'],
    'professional_services': ['legal', 'accounting', 'consulting', 'audit', 'استشارات']
}
```

### Tools & Methods
- **pandas** - DataFrame analysis, grouping, filtering
- **Vector Store** - Transaction retrieval via ChromaDB
- **LLM** - Pattern analysis and natural language generation
- `_analyze_expenses()` - Deep analysis with category breakdown
- `_categorize_expenses()` - Keyword-based categorization
- `_identify_patterns()` - Day-of-week analysis, recurring detection
- `_format_expense_response()` - LLM-generated summary

### Key Capabilities
- Categorize expenses into 10 business categories
- Detect recurring payments
- Calculate category-wise totals and percentages
- Month filtering (e.g., "expenses in January")
- Pattern detection (day-of-week spending)
- Top expense identification

---

## 2. IncomeAgent 📈

### Purpose
Specialized agent for business revenue analysis and income stream tracking.

### System Prompt
```
You are a Business Revenue Analysis Expert specializing in Saudi Arabian corporate finance and income stream management.

Your expertise includes:
1. Revenue Stream Classification (Client Payments, Services, Products, Subscriptions)
2. Payment Collection Analysis and Efficiency
3. Client Concentration Risk Assessment
4. Recurring vs One-Time Revenue Analysis
5. Revenue Growth and Stability Evaluation
6. Saudi Market Revenue Patterns (Government Contracts, B2B, Ramadan Impact)
7. International Revenue Management (SWIFT Transfers)
8. Revenue Quality and Predictability Assessment

When analyzing business income:
- Classify revenue by source and stability
- Assess payment collection patterns and efficiency
- Identify client concentration risks
- Analyze recurring vs one-time revenue mix
- Evaluate revenue growth trajectories
- Consider Saudi business cycles (Ramadan, Hajj, fiscal patterns)
- Track government contract revenue and payment timing
- Analyze revenue quality and predictability
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Revenue breakdown by source/category
- Client concentration analysis
- Revenue stability assessment
- Growth trends and patterns
- Collection efficiency metrics
- Revenue optimization opportunities
```

### Full Analysis Prompt Template
```
Think deeply about this income and revenue analysis query in the context of Saudi Arabian business transactions.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Time Period Analysis: What timeframe is the business owner asking about?
   - Monthly/quarterly/annual revenue analysis?
   - Year-over-year growth comparison?
   - Specific period revenue breakdown?
   - Revenue trend analysis?

2. Revenue Stream Classification: What types of income to identify?
   - Client/customer payments (B2B vs B2C)?
   - Service revenue (professional services, consulting)?
   - Product sales revenue?
   - Subscription or recurring revenue?
   - Project-based revenue (milestone payments)?
   - Government contracts or grants?

3. Payment Patterns: What collection dynamics exist?
   - Payment terms (Net 30, Net 60, immediate)?
   - Recurring vs one-time payments?
   - Advance payments or deposits?
   - Late payment patterns?
   - Seasonal revenue variations?

4. Client Concentration: What revenue stability factors?
   - Dependency on major clients (top 3, top 10)?
   - Revenue diversification across clients?
   - Client retention and churn rates?
   - New vs existing client revenue mix?
   - Risk from client concentration?

5. Saudi Business Context: What local revenue factors?
   - Government contracts and payment cycles?
   - SADAD payment collections (automated)?
   - Vision 2030 sector opportunities?
   - Ramadan impact on business revenue (seasonal)?
   - International revenue (SWIFT transfers)?

6. Revenue Quality Assessment: What stability indicators?
   - Predictable recurring revenue percentage?
   - Revenue growth trajectory (stable, growing, declining)?
   - Payment collection efficiency?
   - Revenue per customer/transaction trends?
   - Profitability by revenue stream?

7. Analysis Requirements: What insights to provide?
   - Total revenue by source/category?
   - Growth trends and patterns?
   - Revenue stability assessment?
   - Client/customer insights?
   - Revenue optimization opportunities?

Provide detailed reasoning:
```

### Income Categories
```python
{
    'client_payments': ['client', 'customer', 'invoice', 'payment received', 'contract payment', 'عميل', 'فاتورة'],
    'service_revenue': ['service', 'consulting', 'project', 'delivery', 'implementation', 'خدمة', 'مشروع'],
    'product_sales': ['sales', 'product', 'goods', 'merchandise', 'order', 'مبيعات', 'منتج'],
    'recurring_revenue': ['subscription', 'monthly', 'recurring', 'retainer', 'maintenance', 'اشتراك', 'شهري'],
    'investment_income': ['dividend', 'return', 'investment', 'interest', 'profit share', 'أرباح'],
    'government_support': ['subsidy', 'grant', 'government', 'support', 'دعم', 'حكومي'],
    'refunds': ['refund', 'reimbursement', 'return', 'cashback', 'استرداد'],
    'inter_company': ['transfer', 'inter-company', 'group', 'subsidiary', 'تحويل'],
    'partnership_income': ['partner', 'joint venture', 'collaboration', 'شريك']
}
```

### Tools & Methods
- **pandas** - Income stream analysis
- **numpy** - Salary pattern detection
- `_analyze_income()` - Income source breakdown
- `_identify_income_sources()` - Advanced B2B payment detection
- `_detect_salary_pattern()` - Monthly salary identification (25-35 days interval)
- `_calculate_monthly_average()` - Average monthly income

### Key Capabilities
- Identify 9 types of income sources
- Detect regular salary patterns
- Client payment recognition (B2B transfers with company keywords)
- Monthly average income calculation
- Source-wise percentage breakdown
- Payment term analysis

### Advanced Features
- **B2B Payment Detection**: Recognizes corporate transfers with patterns like:
  - Company keywords + payment references
  - Bank transfer indicators (SABB, SWIFT, ICORP)
  - Formal payment structures (sent date, exchange rate, value date)

---

## 3. FeeHunterAgent 🔍

### Purpose
Detects and analyzes all types of fees and charges, identifies savings opportunities.

### System Prompt
```
You are a Banking Fee Detection and Optimization Expert specializing in Saudi Arabian corporate banking cost reduction.

Your expertise includes:
1. Banking Fee Identification (Transaction, Account, Wire Transfer, Card Fees)
2. Fee Pattern Recognition and Classification
3. Avoidable vs Unavoidable Fee Analysis
4. Savings Opportunity Calculation
5. Saudi Banking Fee Regulations (VAT 15%, SAMA Guidelines)
6. SWIFT and International Transfer Fee Optimization
7. Fee Negotiation Strategy Development
8. Alternative Banking Solution Recommendations

When analyzing banking fees:
- Identify all fee types accurately (banking, transaction, VAT, penalties)
- Classify fees by frequency (recurring, per-transaction, one-time)
- Assess fee avoidability (avoidable, negotiable, preventable, necessary)
- Calculate total annual fee costs by category
- Estimate potential savings from optimization
- Consider VAT on banking services (15% in Saudi Arabia)
- Analyze SWIFT/international transfer fees for volume discounts
- Identify duplicate or erroneous charges
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Total fees breakdown by type
- Fee frequency analysis (recurring vs one-time)
- Avoidability assessment per category
- Estimated annual savings potential
- Specific fee reduction recommendations
- Alternative banking options for cost reduction
```

### Full Analysis Prompt Template
```
Think deeply about this banking fee and charge analysis query for Saudi Arabian business transactions.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Fee Type Classification: What kinds of fees to identify?
   - Banking service fees (account maintenance, wire transfers)?
   - Transaction fees (ATM, POS, card transactions)?
   - SWIFT/international transfer fees?
   - Currency exchange fees?
   - Credit/financing fees and interest charges?
   - Overdraft or insufficient funds fees?
   - VAT on banking services (15%)?

2. Fee Frequency Analysis: What payment patterns exist?
   - Recurring monthly fees (account maintenance)?
   - Per-transaction fees (each wire transfer)?
   - Annual fees (credit cards, licenses)?
   - One-time fees (account opening, closure)?
   - Conditional fees (minimum balance penalties)?

3. Fee Avoidability Assessment: Which fees could be reduced?
   - Avoidable fees (ATM fees with better card usage)?
   - Negotiable fees (wire transfer fees for volume)?
   - Preventable fees (late payment penalties)?
   - Unnecessary fees (duplicate services)?
   - Optimization opportunities (switching account types)?

4. Savings Calculation: What cost reduction potential exists?
   - Total annual fees by category?
   - Percentage of fees that are avoidable?
   - Estimated savings from optimization?
   - Alternative banking options with lower fees?
   - Negotiation opportunities with bank?

5. Saudi Banking Context: What local fee regulations?
   - VAT on banking services (15% standard rate)?
   - SAMA (Saudi Central Bank) fee regulations?
   - Consumer protection on fee disclosure?
   - Islamic banking fee structures (no interest)?
   - Interbank transfer fees (SARIE system)?

6. Pattern Recognition: What unusual fee patterns?
   - Increasing fee trends over time?
   - Unusually high fees compared to benchmarks?
   - Duplicate or erroneous charges?
   - Seasonal fee variations?
   - Fee concentration in specific categories?

7. Recommendations: What actionable advice to provide?
   - Immediate savings opportunities?
   - Banking behavior changes to reduce fees?
   - Alternative banking products/services?
   - Negotiation strategies with bank?
   - Long-term fee optimization plan?

Provide detailed reasoning:
```

### Fee Categories
```python
{
    'transaction_fees': ['transaction fee', 'processing fee', 'transfer charge', 'رسوم معاملة'],
    'swift_fees': ['swift', 'international transfer', 'wire fee', 'correspondent', 'سويفت'],
    'account_fees': ['account fee', 'maintenance', 'monthly charge', 'service charge', 'رسوم حساب'],
    'corporate_card_fees': ['corporate card', 'business card', 'annual fee', 'card charge'],
    'vat_charges': ['vat', 'value added tax', 'tax charge', 'ضريبة القيمة المضافة', '15%'],
    'government_fees': ['government fee', 'sadad fee', 'ministry charge', 'رسوم حكومية'],
    'penalty_charges': ['penalty', 'late payment', 'overdraft', 'insufficient funds', 'غرامة'],
    'pos_fees': ['pos', 'point of sale', 'merchant fee', 'mada fee', 'نقاط البيع'],
    'forex_fees': ['currency', 'exchange', 'conversion', 'forex', 'تحويل عملة'],
    'digital_banking_fees': ['online', 'digital', 'platform fee', 'subscription', 'إلكتروني']
}
```

### Tools & Methods
- **Fee Detection Logic**: `_is_likely_fee()` - Keyword + amount pattern matching
- **Typical Fee Amounts**: [5, 10, 15, 20, 25, 30, 35, 50, 75, 100, 2.5, 3.75, 7.5]
- `_filter_fees_from_transactions()` - Extract fees from transaction list
- `_find_recurring_fees()` - Identify repeated fee amounts
- `_analyze_fees()` - Calculate total fees and savings potential

### Key Capabilities
- Detect 10 types of fees
- Identify recurring vs one-time fees
- Calculate monthly and annual savings potential
- Fee avoidance recommendations
- Saudi VAT (15%) recognition

---

## 4. BudgetAdvisorAgent 📊

### Purpose
Business cash flow analysis and financial health assessment.

### System Prompt
```
You are a Saudi corporate financial advisor. Provide business budget advice in SAR.
```

### Full Analysis Prompt Template
```
Think deeply about this budget and cash flow query in the context of Saudi Arabian business operations.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Time Period Analysis: What timeframe is the business owner asking about?
   - Monthly/quarterly/annual budget review?
   - Year-over-year comparisons?
   - Future projections or historical analysis?

2. Cash Flow Components: Which elements need analysis?
   - Operating cash flow (revenue vs expenses)?
   - Debt service coverage (loan payments)?
   - Working capital requirements?
   - Reserve fund adequacy?

3. Business Health Metrics: What financial ratios are relevant?
   - Operating margin (net income / revenue)?
   - Expense ratio (expenses / revenue)?
   - Savings rate (surplus / revenue)?
   - Debt-to-income ratio?

4. Saudi Business Context: What compliance/regulatory aspects?
   - GOSI contributions (social insurance)?
   - QIWA compliance (labor regulations)?
   - SADAD recurring payments (utilities, government)?
   - Zakat obligations?

5. Budget Categories: Which spending areas to examine?
   - Fixed costs (rent, salaries, insurance)?
   - Variable costs (utilities, supplies)?
   - Discretionary spending?
   - Capital expenditures?

6. Financial Goals: What recommendations might help?
   - Emergency fund targets (3-6 months expenses)?
   - Debt reduction strategies?
   - Cost optimization opportunities?
   - Revenue growth requirements?

7. Currency and Format: Present in SAR with business-appropriate formatting

Provide detailed reasoning:
```

### Budget Rules
```python
{
    'operating_expenses': 60,  # % of revenue for operations
    'debt_service': 15,        # % for debt payments
    'reserves': 25              # % for cash reserves/growth
}
```

### Tools & Methods
- `_calculate_health_score()` - Financial health scoring (0-100)
- Separates income and expenses for net cash flow analysis
- Calculates:
  - Net cashflow
  - Savings rate (% of income saved)
  - Expense ratio (expenses as % of income)

### Financial Health Score Calculation
```
Base Score: 50

Savings Rate Adjustments:
+30 points: ≥20% savings rate
+20 points: ≥10% savings rate
+10 points: ≥5% savings rate
-20 points: Negative savings (deficit)

Expense Ratio Adjustments:
+20 points: ≤70% expense ratio
+10 points: ≤85% expense ratio
-10 points: >100% (spending more than income)

Final Score: 0-100
```

### Key Capabilities
- Calculate operating margin
- Net cash flow tracking
- Financial health scoring
- Working capital optimization advice
- Debt service coverage analysis

---

## 5. TrendAnalystAgent 📉

### Purpose
Analyzes financial trends, patterns, and growth trajectories.

### System Prompt
```
You are a business intelligence analyst for Saudi companies. Analyze growth patterns, seasonality, and business cycles systematically.
```

### Full Analysis Prompt Template
```
Think deeply about this trend and pattern analysis query in the context of Saudi Arabian business data.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Trend Type Identification: What kind of trend is being requested?
   - Revenue growth trends?
   - Expense increase/decrease patterns?
   - Cash flow seasonality?
   - Transaction volume trends?

2. Time Period Analysis: What timeframe should be examined?
   - Short-term trends (weekly/monthly)?
   - Medium-term patterns (quarterly/seasonal)?
   - Long-term growth (annual/multi-year)?
   - Comparative periods (YoY, MoM)?

3. Pattern Recognition: What patterns might exist?
   - Seasonal business cycles (Ramadan, Hajj season)?
   - Recurring payments (monthly GOSI, SADAD)?
   - Growth trajectories (linear, exponential, declining)?
   - Day-of-week or time-of-month patterns?

4. Statistical Analysis: Which metrics are relevant?
   - Growth rate calculations (percentage change)?
   - Moving averages (smoothing volatility)?
   - Trend direction (increasing/decreasing/stable)?
   - Volatility measures (standard deviation)?

5. Saudi Business Context: What cultural/economic factors?
   - Islamic calendar impacts (Ramadan business slowdown)?
   - Government payment cycles (monthly GOSI)?
   - Vision 2030 economic trends?
   - Oil price impacts on business environment?

6. Actionable Insights: What predictions or recommendations?
   - Future projections based on trends?
   - Opportunities for optimization?
   - Risk indicators (declining revenue)?
   - Budget planning implications?

7. Data Requirements: What historical data is needed?
   - Minimum data points for reliable trends?
   - Need for outlier removal?
   - Aggregation level (daily/weekly/monthly)?

Provide detailed reasoning:
```

### Tools & Methods
- **numpy** - Linear trend calculation (`polyfit`)
- `_analyze_trends()` - Time-series analysis
- Month-over-month and period-over-period comparisons
- Day-of-week pattern analysis

### Trend Detection
```python
# Linear trend slope calculation
slope = np.polyfit(x, y, 1)[0]

Direction Classification:
- "increasing": slope > 100 SAR/month
- "decreasing": slope < -100 SAR/month
- "stable": -100 ≤ slope ≤ 100
```

### Key Capabilities
- Revenue/expense trend direction
- Monthly change rate calculation
- Day-of-week spending patterns
- Growth trajectory assessment
- Seasonal pattern identification

---

## 6. TransactionInvestigatorAgent 🕵️

### Purpose
Find and investigate specific transactions using semantic search.

### System Prompt
```
You are a corporate transaction investigator for Saudi business banking. Find and analyze specific business transactions systematically.
```

### Full Analysis Prompt Template
```
Think deeply about this transaction search and investigation query for Saudi Arabian business banking.

Query: {query}
Extracted Parameters: {params}

Consider these aspects (your thinking will be hidden from the user):
1. Search Intent: What is the user trying to find?
   - Specific transaction by amount or merchant?
   - Transaction category or type?
   - Time-based search (recent, specific date)?
   - Pattern investigation (recurring, unusual)?

2. Search Parameters Analysis: What clues are in the query?
   - Amounts mentioned (exact or approximate)?
   - Merchant/vendor names (quoted strings)?
   - Keywords (salary, rent, GOSI, SWIFT)?
   - Date references (last month, January, yesterday)?

3. Transaction Types: Which categories might match?
   - Payroll and employee compensation?
   - Government payments (GOSI, QIWA, SADAD)?
   - Vendor and supplier payments?
   - International transfers (SWIFT)?
   - Banking fees and charges?
   - Operational expenses?

4. Saudi Banking Context: What banking patterns exist?
   - SADAD payment system transactions?
   - SWIFT/international wire codes?
   - IBAN-based local transfers?
   - Government entity identifiers?
   - Arabic transaction descriptions?

5. Search Strategy: How to find the best matches?
   - Exact amount matching (±1 SAR tolerance)?
   - Semantic description search?
   - Date range filtering?
   - Transaction type filtering (debit/credit)?
   - Relevance ranking criteria?

6. Investigation Depth: What level of detail needed?
   - Single transaction details?
   - Multiple related transactions?
   - Pattern across transactions?
   - Anomaly detection?

7. Result Presentation: How to present findings?
   - Most relevant transactions first?
   - Grouped by category or date?
   - Include context (related transactions)?
   - Highlight matching criteria?

Provide detailed reasoning:
```

### Tools & Methods
- **regex** - Amount and merchant extraction
- `_extract_search_parameters()` - Parse amounts, keywords, merchants
- `_rank_transactions()` - Relevance scoring
- Semantic vector search via ChromaDB

### Search Parameter Extraction
```python
# Amount patterns
r'sar[\s]*([\d,]+\.?\d*)'  # "SAR 1,500"
r'([\d,]+\.?\d*)[\s]*sar'  # "1500 SAR"

# Merchant patterns
r'"([^"]+)"'  # Quoted strings
r'at ([A-Z][a-z]+)'  # "at Starbucks"

# Keywords
['restaurant', 'uber', 'amazon', 'atm', 'transfer', 'salary']
```

### Relevance Scoring
```
+20 points: Keyword match
+30 points: Merchant name match
+50 points: Exact amount match (±1 SAR tolerance)
```

### Key Capabilities
- Amount-based search (e.g., "find 1500 SAR transaction")
- Merchant-based search (e.g., "find Starbucks payments")
- Keyword search (e.g., "find all uber rides")
- Date-based filtering
- Duplicate transaction detection
- Anomaly identification

---

# PART 2: FINANCIAL STATEMENT AGENTS (6)

These agents analyze financial statements (Balance Sheet, P&L, Cash Flow).

---

## 7. RatioAnalystAgent 📐

### Purpose
Calculate and interpret key financial ratios including liquidity, profitability, leverage, and efficiency.

### System Prompt
```
You are a Financial Ratio Analysis Expert specializing in calculating and interpreting key financial metrics.

Your expertise includes:
1. Liquidity Ratios (Current, Quick, Cash)
2. Profitability Ratios (ROE, ROA, Net Margin, Gross Margin)
3. Leverage Ratios (Debt-to-Equity, Interest Coverage, Debt-to-Assets)
4. Efficiency Ratios (Asset Turnover, Inventory Turnover, DSO)
5. Market Ratios (P/E, EPS, Dividend Yield)

When analyzing financial statements:
- Calculate all relevant ratios accurately
- Compare ratios to industry benchmarks
- Identify strengths and weaknesses
- Provide actionable insights
- Consider Saudi Arabian business context (Zakat, VAT implications)
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Key ratios with values
- Health assessment (Excellent/Good/Fair/Poor)
- Comparison to benchmarks
- Recommendations for improvement
```

### Full Analysis Prompt Template
```
Think deeply about this financial ratio analysis query for Saudi Arabian business financial statements.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Ratio Categories: Which financial ratios are most relevant?
   - Liquidity ratios (Current, Quick, Cash)?
   - Profitability ratios (ROE, ROA, Net Margin, Gross Margin)?
   - Leverage ratios (Debt-to-Equity, Interest Coverage)?
   - Efficiency ratios (Asset Turnover, Inventory Turnover)?

2. Financial Health Assessment: What does the user want to know?
   - Overall financial health score?
   - Specific ratio interpretation?
   - Comparison to industry benchmarks?
   - Trend analysis over time?

3. Balance Sheet Analysis: What metrics to examine?
   - Asset composition (current vs non-current)?
   - Liability structure (short-term vs long-term debt)?
   - Equity position and shareholder value?
   - Working capital adequacy?

4. Income Statement Analysis: What profitability metrics?
   - Revenue and gross profit margins?
   - Operating efficiency (EBIT, EBITDA)?
   - Net profitability and return ratios?
   - Cost structure analysis?

5. Saudi Business Context: What regulatory considerations?
   - Zakat impact on net profit calculations?
   - VAT implications on revenue/expenses?
   - SAMA banking regulations (if applicable)?
   - Vision 2030 sector considerations?

6. Benchmark Comparisons: What standards to use?
   - Industry-specific benchmarks?
   - Historical company performance?
   - Regional market averages?
   - International standards (adjusted for Saudi context)?

7. Actionable Insights: What recommendations needed?
   - Strengths to leverage?
   - Weaknesses requiring attention?
   - Opportunities for improvement?
   - Risk mitigation strategies?

Provide detailed reasoning:
```

### Calculated Ratios

**Liquidity Ratios:**
```
Current Ratio = Current Assets / Current Liabilities
Quick Ratio = (Current Assets - Inventory) / Current Liabilities
Cash Ratio = Cash / Current Liabilities
```

**Profitability Ratios:**
```
Net Margin = (Net Profit / Revenue) × 100
Gross Margin = (Gross Profit / Revenue) × 100
ROA = (Net Profit / Total Assets) × 100
ROE = (Net Profit / Total Equity) × 100
```

**Leverage Ratios:**
```
Debt-to-Equity = Total Debt / Total Equity
Debt-to-Assets = Total Debt / Total Assets
Interest Coverage = EBIT / Interest Expense
```

**Efficiency Ratios:**
```
Asset Turnover = Revenue / Total Assets
```

### Health Score Calculation
```
Base Score: 50

Liquidity Contribution (max 20 points):
+20: Current Ratio ≥ 2
+15: Current Ratio ≥ 1.5
+10: Current Ratio ≥ 1

Profitability Contribution (max 20 points):
+20: ROE ≥ 15%
+15: ROE ≥ 10%
+10: ROE ≥ 5%

Leverage Contribution (max 10 points):
+10: Debt-to-Equity between 0.3-1.0
+5: Debt-to-Equity < 0.3

Final Score: 0-100
```

### Tools & Methods
- `_calculate_ratios()` - Comprehensive ratio calculations
- `_calculate_health_score()` - 0-100 financial health scoring
- `_extract_key_findings()` - Automated insights
- `_generate_recommendations()` - Actionable advice

### Key Capabilities
- Calculate 13+ financial ratios
- Health score (0-100)
- Benchmark comparisons
- Strength/weakness identification
- Actionable recommendations

---

## 8. ProfitabilityAgent 💹

### Purpose
Analyze profit margins, trends, and optimization strategies.

### System Prompt
```
You are a Profitability Analysis Expert specializing in margin optimization and profit trends.

Your expertise includes:
1. Gross Profit Analysis (Revenue - COGS)
2. Operating Profit Analysis (EBIT, EBITDA)
3. Net Profit Margins and Trends
4. Cost Structure Optimization
5. Revenue Growth Strategies
6. Margin Improvement Opportunities

When analyzing profitability:
- Calculate all profit metrics (Gross, Operating, Net)
- Analyze margin trends over time
- Compare to industry benchmarks
- Identify cost reduction opportunities
- Suggest revenue enhancement strategies
- Consider Saudi business context (Zakat impact on net profit)
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Profit metrics and margins
- Year-over-year comparisons
- Cost breakdown analysis
- Margin improvement recommendations
```

### Full Analysis Prompt Template
```
Think deeply about this profitability and margin analysis query for Saudi Arabian business financial performance.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Profit Metrics: Which profitability measures are relevant?
   - Gross Profit (Revenue - COGS)?
   - Operating Profit (EBIT, EBITDA)?
   - Net Profit (after all expenses and taxes)?
   - Contribution margins by product/service?

2. Margin Analysis: What margin trends to examine?
   - Gross margin percentage trends?
   - Operating margin sustainability?
   - Net profit margin efficiency?
   - EBITDA margin for operational performance?

3. Revenue Analysis: What revenue aspects to consider?
   - Total revenue growth (YoY, MoM)?
   - Revenue diversification and stability?
   - Pricing power and market position?
   - Revenue per customer or per unit?

4. Cost Structure: What cost components to analyze?
   - COGS as percentage of revenue?
   - Operating expenses efficiency?
   - Fixed vs variable cost breakdown?
   - Cost reduction opportunities?

5. Saudi Business Context: What local factors matter?
   - Zakat deduction impact on net profit (2.5% of adjusted profit)?
   - VAT impact on revenue calculations (15%)?
   - Industry-specific margin norms in Saudi market?
   - Vision 2030 sector growth opportunities?

6. Year-over-Year Comparisons: What trends to identify?
   - Revenue growth rate consistency?
   - Margin expansion or contraction?
   - Cost control effectiveness?
   - Profitability improvement trajectory?

7. Optimization Strategies: What recommendations to provide?
   - Revenue enhancement opportunities?
   - Cost reduction strategies?
   - Pricing optimization potential?
   - Operational efficiency improvements?

Provide detailed reasoning:
```

### Calculated Metrics

**Margins:**
```
Gross Margin = (Gross Profit / Revenue) × 100
Operating Margin = (Operating Profit / Revenue) × 100
EBITDA Margin = (EBITDA / Revenue) × 100
Net Margin = (Net Profit / Revenue) × 100
```

**Growth Metrics:**
```
Revenue Growth YoY = ((Current - Prior) / Prior) × 100
Profit Growth YoY = ((Current Profit - Prior Profit) / Prior Profit) × 100
Margin Improvement = Current Net Margin - Prior Net Margin
```

**Cost Ratios:**
```
COGS Ratio = (COGS / Revenue) × 100
OpEx Ratio = (Operating Expenses / Revenue) × 100
```

### Margin Health Assessment
```
Score System (0-4 checks):

1. Gross Margin ≥30% ✓
2. Operating Margin ≥10% ✓
3. Net Margin ≥8% ✓
4. Profit Growth >0% ✓

Health Rating:
- 4/4 or 3/4: "Excellent"
- 2/4: "Good"
- 1/4: "Fair"
- 0/4: "Needs Improvement"
```

### Improvement Opportunities Identified
- COGS optimization (if >60% of revenue)
- Operating expense reduction (if >30% of revenue)
- Price optimization (if gross margin <35%)
- Economies of scale leverage (if revenue growing >15%)
- EBITDA improvement (if EBITDA margin <20%)

### Tools & Methods
- `_calculate_profit_metrics()` - All profit calculations
- `_assess_margin_health()` - 4-point health check
- `_identify_opportunities()` - Cost reduction & revenue optimization
- YoY comparison formatting

---

## 9. LiquidityAgent 💧

### Purpose
Evaluate short-term and long-term financial stability, working capital management.

### System Prompt
```
You are a Liquidity & Solvency Expert specializing in cash flow and financial stability analysis.

Your expertise includes:
1. Liquidity Analysis (Current, Quick, Cash ratios)
2. Working Capital Management
3. Cash Conversion Cycle
4. Solvency Assessment
5. Cash Flow Analysis
6. Short-term Financial Health

When analyzing liquidity:
- Calculate all liquidity metrics
- Assess cash position and trends
- Evaluate working capital efficiency
- Analyze payment capacity
- Identify liquidity risks and opportunities
- Consider Saudi context (payment terms, SADAD cycles)
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Liquidity metrics and ratios
- Cash position analysis
- Working capital assessment
- Risk indicators
- Recommendations for improvement
```

### Full Analysis Prompt Template
```
Think deeply about this liquidity and cash flow analysis query for Saudi Arabian business financial stability.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Liquidity Ratios: Which metrics are most critical?
   - Current Ratio (Current Assets / Current Liabilities)?
   - Quick Ratio (Liquid Assets / Current Liabilities)?
   - Cash Ratio (Cash & Equivalents / Current Liabilities)?
   - Operating Cash Flow Ratio?

2. Working Capital: What working capital aspects to examine?
   - Net working capital sufficiency (Current Assets - Current Liabilities)?
   - Working capital turnover efficiency?
   - Day Sales Outstanding (DSO) for receivables?
   - Days Payable Outstanding (DPO) for payables?

3. Cash Position: What cash metrics matter?
   - Cash and cash equivalents balance?
   - Cash burn rate or cash generation?
   - Free cash flow availability?
   - Cash conversion cycle duration?

4. Short-term Obligations: What payment capacity to assess?
   - Immediate payment obligations (next 30-90 days)?
   - Debt service coverage ability?
   - Payroll and operational expense coverage?
   - Buffer for unexpected expenses?

5. Saudi Business Context: What local factors apply?
   - SADAD payment cycles (monthly utilities, government)?
   - GOSI monthly contributions (employer obligations)?
   - Typical payment terms in Saudi business (30-90 days)?
   - Ramadan cash flow impacts (seasonal)?

6. Solvency Assessment: What long-term stability indicators?
   - Debt-to-equity ratio sustainability?
   - Interest coverage adequacy?
   - Long-term debt maturity schedule?
   - Asset base supporting liabilities?

7. Risk Identification: What liquidity risks exist?
   - Cash shortage risk in near term?
   - Over-reliance on credit facilities?
   - Seasonal cash flow volatility?
   - Recommendations for liquidity improvement?

Provide detailed reasoning:
```

### Calculated Metrics

**Liquidity Ratios:**
```
Current Ratio = Current Assets / Current Liabilities
Quick Ratio = (Current Assets - Inventory) / Current Liabilities
Cash Ratio = (Cash + Marketable Securities) / Current Liabilities
```

**Working Capital:**
```
Working Capital = Current Assets - Current Liabilities
Total Liquid Assets = Cash + Marketable Securities
```

**Cash Conversion Cycle:**
```
Days Inventory Outstanding (DIO) = (Inventory / COGS) × 365
Days Sales Outstanding (DSO) = (Receivables / Revenue) × 365
Days Payables Outstanding (DPO) = (Payables / COGS) × 365

Cash Conversion Cycle = DIO + DSO - DPO
```

### Liquidity Status Assessment
```
Excellent: Current Ratio ≥2, Quick Ratio ≥1.5, Positive Working Capital
Good: Current Ratio ≥1.5, Quick Ratio ≥1
Fair: Current Ratio ≥1, Quick Ratio ≥0.8
Poor: Current Ratio <1 or Negative Working Capital
```

### Tools & Methods
- `_calculate_liquidity_metrics()` - All liquidity calculations
- `_assess_liquidity_status()` - Status determination
- `_identify_liquidity_risks()` - Risk flagging
- Cash conversion cycle analysis

### Key Capabilities
- Working capital analysis
- Cash position trends
- Payment capacity assessment
- Liquidity risk identification
- Saudi payment cycle consideration (SADAD, Net 30 terms)

---

## 10. FinancialTrendAgent 📊

### Purpose
Identify patterns in financial performance over time, growth trajectories, seasonal patterns.

### System Prompt
```
You are a Financial Trend Analysis Expert specializing in identifying patterns and trajectories in financial performance.

Your expertise includes:
1. Revenue Growth Trends (YoY, QoQ, MoM)
2. Expense Pattern Analysis
3. Margin Evolution Over Time
4. Seasonal Performance Patterns
5. Working Capital Trends
6. Cash Flow Trajectories

When analyzing trends:
- Calculate growth rates and compound annual growth rate (CAGR)
- Identify seasonal patterns and cyclicality
- Detect trend reversals and inflection points
- Compare performance across periods
- Project future trajectories based on historical patterns
- Consider Saudi market dynamics (Ramadan effects, Hajj season, fiscal year patterns)
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Growth metrics and trends
- Seasonal pattern identification
- Performance trajectory assessment
- Forward-looking insights
- Risk factors and opportunities
```

### Full Analysis Prompt Template
```
Think deeply about this financial trend analysis query for Saudi Arabian business performance patterns.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Trend Type: What kind of financial trend is being analyzed?
   - Revenue growth trajectory (YoY, QoQ, MoM)?
   - Profitability evolution over time?
   - Expense pattern changes?
   - Cash flow trends and stability?

2. Time Period Analysis: What timeframe is most relevant?
   - Short-term trends (monthly/quarterly)?
   - Medium-term patterns (annual)?
   - Long-term growth (multi-year CAGR)?
   - Comparative period analysis (YoY, QoQ)?

3. Growth Metrics: Which growth calculations to perform?
   - Revenue CAGR (Compound Annual Growth Rate)?
   - Profit growth consistency?
   - Margin expansion/contraction rates?
   - Asset and equity growth rates?

4. Seasonal Patterns: What cyclical factors to identify?
   - Ramadan impact on business activity (30-day cycle)?
   - Hajj season effects (annual pilgrimage period)?
   - Government fiscal year patterns (Gregorian calendar)?
   - Quarterly performance volatility?

5. Saudi Business Context: What market dynamics apply?
   - Vision 2030 sector transformation impacts?
   - Oil price correlation (for affected sectors)?
   - Government spending cycles?
   - Islamic calendar business cycles?

6. Trend Direction: What trajectory patterns exist?
   - Consistent growth, decline, or stability?
   - Acceleration or deceleration?
   - Trend reversals or inflection points?
   - Volatility and predictability assessment?

7. Forward-Looking Insights: What projections to provide?
   - Extrapolation of current trends?
   - Growth sustainability assessment?
   - Risk factors for trend continuation?
   - Opportunities emerging from trends?

Provide detailed reasoning:
```

### Calculated Metrics

**Growth Rates:**
```
Revenue YoY Growth = ((Current - Prior Year) / Prior Year) × 100
Profit YoY Growth = ((Current Profit - Prior Profit) / Prior Profit) × 100
EBITDA YoY Growth = ((Current EBITDA - Prior EBITDA) / Prior EBITDA) × 100
```

**Margin Trends:**
```
Margin Improvement = Current Net Margin - Prior Net Margin (in percentage points)

Trend Direction:
- "expanding" if margin improvement > 0
- "contracting" if margin improvement < 0
- "stable" if margin change ≈ 0
```

**Quarterly Analysis:**
```
QoQ Growth = ((Current Quarter - Previous Quarter) / Previous Quarter) × 100
```

### Tools & Methods
- `_calculate_trend_metrics()` - Multi-period calculations
- `_determine_trend_direction()` - Trajectory classification
- `_identify_seasonal_patterns()` - Quarterly pattern detection
- `_generate_projections()` - Forward-looking estimates

### Saudi Market Considerations
- Ramadan seasonality (reduced business activity)
- Hajj season impact (tourism, hospitality)
- Fiscal year patterns
- Government payment cycles

---

## 11. RiskAssessmentAgent ⚠️

### Purpose
Evaluate financial risks, compliance status, and risk mitigation strategies.

### System Prompt
```
You are a Financial Risk & Compliance Expert specializing in risk assessment and regulatory compliance.

Your expertise includes:
1. Credit Risk Assessment (Default risk, creditworthiness)
2. Liquidity Risk Analysis
3. Market Risk Evaluation (Currency exposure, interest rate risk)
4. Operational Risk Assessment
5. Compliance Monitoring (Zakat, VAT, GOSI, regulatory requirements)
6. Covenant Analysis and Monitoring

When assessing risks:
- Identify and quantify key financial risks
- Evaluate compliance with Saudi regulations (Zakat, VAT, GOSI)
- Assess debt covenants and restrictions
- Calculate risk scores and ratings
- Identify early warning indicators
- Provide risk mitigation recommendations
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Risk score (1-10, where 1 is low risk)
- Key risk factors identified
- Compliance status summary
- Covenant analysis
- Risk mitigation recommendations
```

### Full Analysis Prompt Template
```
Think deeply about this financial risk assessment query for Saudi Arabian business risk evaluation.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Risk Categories: Which financial risks to assess?
   - Credit risk (default probability, debt burden)?
   - Liquidity risk (cash shortage, payment capacity)?
   - Market risk (FX exposure, interest rate sensitivity)?
   - Operational risk (business continuity, fraud)?

2. Leverage & Debt Risk: What debt-related risks exist?
   - Debt-to-equity ratio sustainability?
   - Interest coverage adequacy (EBIT / Interest)?
   - Debt maturity schedule concentration?
   - Covenant compliance (financial ratio restrictions)?

3. Liquidity Risk: What cash flow risks are present?
   - Current ratio sufficiency (<1.0 = high risk)?
   - Cash burn rate vs cash reserves?
   - Working capital volatility?
   - Dependency on external financing?

4. Profitability Risk: What earnings stability concerns?
   - Profit margin consistency?
   - Revenue concentration (customer dependency)?
   - Cost structure rigidity (high fixed costs)?
   - Earnings volatility and predictability?

5. Saudi Compliance Risks: What regulatory obligations?
   - Zakat compliance (2.5% on adjusted capital)?
   - VAT compliance (15% collection and remittance)?
   - GOSI contributions (employer 12%, employee 10%)?
   - SAMA regulations (if financial institution)?

6. Governance & Operational Risks: What control concerns?
   - Financial reporting quality and transparency?
   - Internal control effectiveness?
   - Management continuity and succession?
   - Fraud indicators or irregularities?

7. Risk Scoring & Mitigation: What recommendations to provide?
   - Overall risk score (1-10 scale)?
   - Critical risk factors requiring immediate attention?
   - Early warning indicators to monitor?
   - Risk mitigation strategies and action plan?

Provide detailed reasoning:
```

### Risk Metrics Calculated

**Leverage Risks:**
```
Debt-to-Equity Ratio = Total Debt / Total Equity
Debt-to-Assets Ratio = Total Debt / Total Assets
Interest Coverage Ratio = EBIT / Interest Expense
Equity Ratio = Total Equity / Total Assets
```

**Liquidity Risks:**
```
Current Ratio = Current Assets / Current Liabilities
Quick Ratio = (Current Assets - Inventory) / Current Liabilities
Cash-to-Debt Ratio = Cash / Total Debt
```

**Profitability Indicators:**
```
Net Margin = (Net Profit / Revenue) × 100
ROA = (Net Profit / Total Assets) × 100
```

### Risk Score Calculation (1-10 Scale)
```
Base Score: 5 (Medium Risk)

Risk Increases (+points):
+2: Debt-to-Equity > 2
+2: Current Ratio < 1
+1: Interest Coverage < 2
+1: Negative cash flow
+1: Net margin < 3%

Risk Decreases (-points):
-2: Debt-to-Equity < 0.5
-1: Current Ratio > 2
-1: Interest Coverage > 5
-1: Strong cash flow
-1: Net margin > 15%

Final Score: 1 (Low Risk) to 10 (High Risk)
```

### Saudi Compliance Checklist
- **Zakat Compliance**: 2.5% of net worth
- **VAT Compliance**: 15% on taxable supplies
- **GOSI**: Social insurance contributions
- **SAMA Regulations**: Banking regulations
- **Corporate Governance**: BOD requirements

### Early Warning Indicators
- Declining current ratio (<1.5)
- Rising debt-to-equity (>1.5)
- Falling interest coverage (<3x)
- Negative operating cash flow
- Margin compression (>2pp decline)

### Tools & Methods
- `_calculate_risk_metrics()` - Comprehensive risk calculations
- `_calculate_risk_score()` - 1-10 risk scoring
- `_assess_compliance()` - Saudi regulatory compliance check
- `_identify_early_warnings()` - Red flag detection
- `_generate_mitigation_strategies()` - Risk reduction recommendations

---

## 12. EfficiencyAgent ⚡

### Purpose
Analyze operational metrics, asset utilization, and productivity.

### System Prompt
```
You are an Operational Efficiency Expert specializing in productivity and asset utilization analysis.

Your expertise includes:
1. Asset Turnover Analysis (Total, Fixed, Working Capital)
2. Inventory Management Metrics (Turnover, Days Inventory)
3. Receivables Management (DSO, Collection Efficiency)
4. Payables Management (DPO, Payment Terms Optimization)
5. Operating Cycle Efficiency
6. Cost Efficiency Metrics (Cost per unit, Overhead ratios)

When analyzing efficiency:
- Calculate all turnover and efficiency ratios
- Analyze the cash conversion cycle
- Assess asset utilization effectiveness
- Identify operational bottlenecks
- Compare to industry benchmarks
- Provide improvement recommendations
- Consider Saudi business practices (payment terms, collection cycles)
- Use SAR (Saudi Riyals) for all amounts

Format your response with:
- Efficiency metrics and ratios
- Asset utilization analysis
- Working capital efficiency
- Operational bottlenecks
- Improvement opportunities
```

### Full Analysis Prompt Template
```
Think deeply about this operational efficiency analysis query for Saudi Arabian business performance optimization.

Query: {query}

Consider these aspects (your thinking will be hidden from the user):
1. Asset Utilization: How effectively are assets generating revenue?
   - Total Asset Turnover (Revenue / Total Assets)?
   - Fixed Asset Turnover (Revenue / Fixed Assets)?
   - Working Capital Turnover (Revenue / Working Capital)?
   - Return on Assets (ROA) efficiency?

2. Inventory Efficiency: How well is inventory managed?
   - Inventory Turnover Ratio (COGS / Average Inventory)?
   - Days Inventory Outstanding (DIO = 365 / Inventory Turnover)?
   - Obsolete or slow-moving inventory risk?
   - Just-in-time vs safety stock balance?

3. Receivables Management: How efficient is collections?
   - Days Sales Outstanding (DSO = AR / (Revenue / 365))?
   - Collection effectiveness index?
   - Bad debt provision adequacy?
   - Customer payment terms (typical 30-90 days in Saudi)?

4. Payables Management: How optimal is payment timing?
   - Days Payable Outstanding (DPO = AP / (COGS / 365))?
   - Supplier payment terms optimization?
   - Early payment discount opportunities?
   - Cash flow vs relationship balance?

5. Cash Conversion Cycle: What is the operating cycle efficiency?
   - CCC = DIO + DSO - DPO (lower is better)?
   - Working capital tied up in operations?
   - Seasonal fluctuations in cycle?
   - Comparison to industry norms?

6. Saudi Business Context: What local practices apply?
   - Typical payment terms in Saudi B2B (30-60 days)?
   - Ramadan impact on collection cycles?
   - Government contract payment delays (common issue)?
   - SADAD payment system efficiency?

7. Improvement Opportunities: What optimizations to recommend?
   - Asset utilization enhancement strategies?
   - Inventory reduction opportunities?
   - Receivables acceleration tactics?
   - Payables optimization (without harming relationships)?

Provide detailed reasoning:
```

### Calculated Metrics

**Asset Turnover:**
```
Asset Turnover = Revenue / Total Assets
Fixed Asset Turnover = Revenue / Fixed Assets
Working Capital Turnover = Revenue / Working Capital
Return on Assets = (Net Profit / Total Assets) × 100
```

**Inventory Efficiency:**
```
Inventory Turnover = COGS / Average Inventory
Days Inventory Outstanding = (Inventory / COGS) × 365
```

**Receivables Efficiency:**
```
Receivables Turnover = Revenue / Average Receivables
Days Sales Outstanding (DSO) = (Receivables / Revenue) × 365
Collection Efficiency = (Cash Collected / Credit Sales) × 100
```

**Payables Efficiency:**
```
Payables Turnover = COGS / Average Payables
Days Payables Outstanding (DPO) = (Payables / COGS) × 365
```

**Cash Conversion Cycle:**
```
Cash Conversion Cycle = DIO + DSO - DPO
```

**Cost Efficiency:**
```
Overhead Ratio = (Operating Expenses / Revenue) × 100
Cost per Revenue = (Total Costs / Revenue) × 100
```

### Efficiency Score Calculation
```
Score System (0-100):

Base: 50

Asset Utilization (+20 points):
+20: Asset Turnover ≥1.5
+10: Asset Turnover ≥1.0

Inventory Management (+15 points):
+15: Inventory Turnover >6 (60 days)
+10: Inventory Turnover >4 (90 days)

Receivables Management (+15 points):
+15: DSO <30 days
+10: DSO <45 days
+5: DSO <60 days

Cash Conversion Cycle (+10 points):
+10: CCC <30 days
+5: CCC <60 days

Final Score: 0-100
```

### Bottleneck Identification
- **Inventory bottleneck**: DIO >90 days
- **Collection bottleneck**: DSO >60 days
- **Asset underutilization**: Asset Turnover <1
- **High overhead**: Overhead ratio >30%

### Improvement Opportunities
- Inventory reduction strategies (if DIO >90)
- Collection acceleration (if DSO >60)
- Asset productivity enhancement (if turnover <1)
- Overhead cost reduction (if ratio >30%)
- Payment term optimization

### Tools & Methods
- `_calculate_efficiency_metrics()` - All efficiency calculations
- `_calculate_efficiency_score()` - 0-100 scoring
- `_identify_bottlenecks()` - Problem area detection
- `_identify_improvements()` - Optimization recommendations

---

# Technical Implementation Details

## LLM Configuration

**Primary Model**: `qwen3-14b-32k:latest`
- Context window: 32,000 tokens
- Temperature: 0.1-0.3 (deterministic)
- Max tokens: 400-1000 depending on agent

**Router Model**: `gemma3:4b`
- Fast intent classification
- Low latency (<200ms)

## Vector Store (ChromaDB)

**Embedding Model**: `all-MiniLM-L6-v2`
- Dimensions: 384
- Local inference via SentenceTransformers

**Search Strategy**:
- Semantic similarity search
- Metadata filtering (type, date, amount)
- n_results: 10-100 depending on query

## Agent Execution Flow

```python
1. Router classifies query intent (Gemma3:4b)
2. Orchestrator selects appropriate agent
3. Agent performs hidden thinking (Chain-of-Thought)
4. Agent retrieves data from ChromaDB
5. Agent analyzes data (pandas/numpy)
6. Agent generates response (Qwen3-14B)
7. Response returned to user
```

## Data Privacy

- ✅ 100% local processing (no cloud APIs)
- ✅ Local LLM via Ollama
- ✅ Local vector store via ChromaDB
- ✅ No data leaves user's machine

---

# Agent Selection Logic

## Router Decision Tree

```
Query → Keyword Detection → Agent Selection

Transaction Queries:
- "expense", "spent", "payment" → ExpenseAgent
- "income", "salary", "revenue" → IncomeAgent
- "fee", "charge", "commission" → FeeHunterAgent
- "budget", "cash flow", "saving" → BudgetAdvisorAgent
- "trend", "pattern", "over time" → TrendAnalystAgent
- "find", "search", "specific" → TransactionInvestigatorAgent

Financial Statement Queries:
- "ratio", "liquidity", "current ratio" → RatioAnalystAgent
- "profit", "margin", "EBITDA" → ProfitabilityAgent
- "liquidity", "working capital", "cash" → LiquidityAgent
- "trend", "growth", "YoY" → FinancialTrendAgent
- "risk", "debt", "compliance" → RiskAssessmentAgent
- "efficiency", "turnover", "DSO" → EfficiencyAgent
```

---

# Performance Metrics

| Metric | Value |
|--------|-------|
| Average query response time | 2-3 seconds |
| Transaction processing capacity | 10,000+ per session |
| LLM inference time | 1-2 seconds |
| Vector search time | 100-300ms |
| Context window | 32,000 tokens |
| Embedding dimensions | 384 |

---

# Example Queries by Agent

## ExpenseAgent
- "What were my top expenses in January?"
- "How much did I spend on payroll?"
- "Show me all GOSI payments"
- "Analyze my vendor payments"

## IncomeAgent
- "What are my main revenue streams?"
- "Did I receive my salary?"
- "Show me all client payments"
- "What's my monthly average income?"

## FeeHunterAgent
- "How much did I pay in fees?"
- "Find all bank charges"
- "What are my recurring fees?"
- "Can I reduce my banking fees?"

## BudgetAdvisorAgent
- "What's my financial health score?"
- "Am I saving enough?"
- "Analyze my cash flow"
- "What's my expense ratio?"

## TrendAnalystAgent
- "Are my expenses increasing?"
- "What's my spending trend?"
- "Show me patterns over the last 3 months"

## TransactionInvestigatorAgent
- "Find the 5000 SAR payment to Acme Corp"
- "Show me all Uber transactions"
- "Find payments between 1000-2000 SAR"

## RatioAnalystAgent
- "Calculate all financial ratios"
- "What's my current ratio?"
- "Analyze liquidity and profitability"

## ProfitabilityAgent
- "What are my profit margins?"
- "How is my EBITDA?"
- "Analyze margin trends"

## LiquidityAgent
- "Do I have enough working capital?"
- "What's my cash position?"
- "Analyze my cash conversion cycle"

## FinancialTrendAgent
- "What's my revenue growth?"
- "Show me YoY performance"
- "Identify seasonal patterns"

## RiskAssessmentAgent
- "What's my risk score?"
- "Am I compliant with Saudi regulations?"
- "Assess my debt levels"

## EfficiencyAgent
- "What's my asset turnover?"
- "How efficient is my inventory management?"
- "Calculate my cash conversion cycle"

---

# File Structure

```
yomnai_backend/agents/
├── __init__.py                 # Agent exports
├── config.py                   # Agent configuration
├── models.py                   # Transaction data models
│
├── Core:
│   ├── orchestrator.py         # Multi-agent coordination
│   ├── router.py               # Intent classification
│   ├── base_agent.py           # Base class for all agents
│   ├── vector_store.py         # ChromaDB manager
│   ├── ollama_client.py        # LLM client
│   └── query_understander.py   # Query analysis
│
├── Transaction Agents:
│   ├── expense_agent.py
│   ├── income_agent.py
│   ├── fee_agent.py
│   ├── budget_agent.py
│   ├── trend_agent.py
│   └── transaction_agent.py
│
└── Financial Agents:
    ├── ratio_analyst_agent.py
    ├── profitability_agent.py
    ├── liquidity_agent.py
    ├── financial_trend_agent.py
    ├── risk_assessment_agent.py
    └── efficiency_agent.py
```

---

# Dependencies

```
fastapi>=0.104.0
uvicorn>=0.24.0
chromadb>=0.4.22
sentence-transformers>=2.2.2
pandas>=2.1.0
numpy>=1.24.0
python-dotenv>=1.0.0
requests>=2.31.0
```

---

**Document Version**: 1.0
**Last Updated**: October 2025
**Yomnai Backend**: Fully self-contained with all 12 agents
