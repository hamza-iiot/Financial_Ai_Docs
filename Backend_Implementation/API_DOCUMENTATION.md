# API Documentation - Yomnai Backend

## Base URL
```
Development: http://localhost:8001
Production: https://api.yomnai.com
```

## API Version
```
Version: 1.0.0
OpenAPI: 3.0.0
```

---

## 🔐 Authentication

### Overview
Yomnai uses email-based whitelist authentication with session management. No passwords required.

### Verify Access
```http
POST /api/auth/verify-access
```

#### Request Body
```json
{
  "email": "user@example.com"
}
```

#### Success Response (200 OK)
```json
{
  "authorized": true,
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "expires_at": "2024-01-02T10:00:00Z"
}
```

#### Unauthorized Response (401)
```json
{
  "authorized": false,
  "message": "Access restricted. Please register at https://forms.google.com/register",
  "registration_url": "https://forms.google.com/register"
}
```

### Validate Session
```http
GET /api/auth/validate-session
Headers:
  X-Session-ID: 550e8400-e29b-41d4-a716-446655440000
```

#### Success Response (200 OK)
```json
{
  "valid": true,
  "email": "user@example.com",
  "expires_at": "2024-01-02T10:00:00Z"
}
```

### Logout
```http
POST /api/auth/logout
Headers:
  X-Session-ID: 550e8400-e29b-41d4-a716-446655440000
```

---

## 📤 File Upload

### Upload Document
```http
POST /api/upload
Headers:
  X-Session-ID: {session_id}
  Content-Type: multipart/form-data
```

#### Request Body (Form Data)
```
file: (binary)
document_type: "auto" | "transactions" | "financial"  (optional)
```

#### Success Response (200 OK)
```json
{
  "upload_id": "upload_123456",
  "filename": "bank_statement.csv",
  "document_type": "transactions",
  "status": "processing",
  "details": {
    "rows": 342,
    "columns": ["Date", "Description", "Amount", "Balance"],
    "date_range": {
      "start": "2024-01-01",
      "end": "2024-01-31"
    },
    "detected_currency": "SAR",
    "parsing_method": "intelligent_csv_parser"
  }
}
```

#### Processing Status
```http
GET /api/upload/{upload_id}/status
Headers:
  X-Session-ID: {session_id}
```

#### Status Response
```json
{
  "upload_id": "upload_123456",
  "status": "completed",  // processing | completed | failed
  "progress": 100,
  "message": "Successfully processed 342 transactions",
  "data": {
    "transaction_count": 342,
    "total_income": 87500.00,
    "total_expenses": 65678.00,
    "categories_detected": ["Payroll", "Vendors", "GOSI", "Utilities"],
    "vector_indexed": true
  }
}
```

---

## 💬 Chat Interface

### Send Message
```http
POST /api/chat/message
Headers:
  X-Session-ID: {session_id}
```

#### Request Body
```json
{
  "message": "What are my top 5 expenses this month?",
  "context": {
    "upload_id": "upload_123456",
    "document_type": "transactions"
  }
}
```

#### Success Response (200 OK)
```json
{
  "message_id": "msg_789",
  "response": "Based on your transactions, here are your top 5 expenses...",
  "agent_used": "ExpenseAgent",
  "metadata": {
    "processing_time": 1.23,
    "confidence": 0.95,
    "tokens_used": 450,
    "data_points_analyzed": 342
  },
  "suggestions": [
    "Would you like to see expense trends?",
    "Want to identify potential savings?"
  ]
}
```

### Stream Chat (WebSocket)
```javascript
ws://localhost:8001/ws/chat/{session_id}

// Send message
{
  "type": "message",
  "content": "Analyze my spending patterns"
}

// Receive chunks
{
  "type": "chunk",
  "content": "I'm analyzing",
  "agent": "ExpenseAgent"
}

// Final response
{
  "type": "complete",
  "message_id": "msg_790",
  "agent": "ExpenseAgent"
}
```

---

## 🤖 Agent Analysis

### Run Full Analysis
```http
POST /api/analysis/full
Headers:
  X-Session-ID: {session_id}
```

#### Request Body
```json
{
  "upload_id": "upload_123456",
  "document_type": "transactions" | "financial",  // Determines which agent set to use
  "agents": ["all"] // or specific agent names
}
```

#### Success Response (200 OK)

For **Transaction Analysis** (document_type: "transactions"):
```json
{
  "analysis_id": "analysis_456",
  "status": "completed",
  "document_type": "transactions",
  "results": {
    "ExpenseAgent": {
      "status": "completed",
      "findings": {
        "total_expenses": 65678.00,
        "top_categories": [
          {"category": "Payroll", "amount": 35000, "percentage": 53.3},
          {"category": "Vendors", "amount": 18500, "percentage": 28.2}
        ],
        "insights": "High payroll concentration suggests...",
        "recommendations": ["Consider vendor payment optimization", "Review recurring subscriptions"]
      }
    },
    "IncomeAgent": {
      "status": "completed",
      "findings": {
        "total_income": 87500.00,
        "income_streams": [
          {"source": "Product Sales", "amount": 62000, "percentage": 70.9},
          {"source": "Services", "amount": 18500, "percentage": 21.1}
        ],
        "stability_score": 8.5,
        "growth_trend": "positive"
      }
    },
    "FeeHunterAgent": {
      "status": "completed",
      "findings": {
        "total_fees": 1050.00,
        "fee_breakdown": [
          {"type": "Bank Fees", "amount": 450},
          {"type": "SADAD Fees", "amount": 300},
          {"type": "Transfer Fees", "amount": 300}
        ],
        "savings_opportunities": 350.00
      }
    },
    "BudgetAdvisorAgent": {
      "status": "completed",
      "findings": {
        "cash_flow": 19322.00,
        "burn_rate": 7613.00,
        "runway_months": 8.5,
        "health_score": 78
      }
    },
    "TrendAnalystAgent": {
      "status": "completed",
      "findings": {
        "peak_spending_day": "Thursday",
        "seasonal_patterns": ["Q2 higher by 23%", "Q4 higher by 18%"],
        "anomalies_detected": 2,
        "predictions": {
          "next_month_estimate": 68500.00,
          "confidence": 0.82
        }
      }
    },
    "TransactionInvestigatorAgent": {
      "status": "completed",
      "findings": {
        "duplicate_payments": 2,
        "suspicious_transactions": 0,
        "compliance_status": "GOSI payments on time",
        "audit_flags": []
      }
    }
  },
  "summary": {
    "overall_health_score": 78,
    "key_insights": [
      "Positive cash flow of SAR 19,322",
      "8.5 months of runway available",
      "Opportunity to save SAR 350 in fees"
    ],
    "priority_actions": [
      "Review duplicate payments worth SAR 850",
      "Optimize Thursday spending patterns",
      "Negotiate better bank fee structure"
    ]
  }
}
```

For **Financial Statement Analysis** (document_type: "financial"):
```json
{
  "analysis_id": "analysis_789",
  "status": "completed",
  "document_type": "financial",
  "results": {
    "RatioAnalystAgent": {
      "status": "completed",
      "findings": {
        "current_ratio": 2.75,
        "quick_ratio": 1.89,
        "debt_to_equity": 0.59,
        "roe": 18.5,
        "roa": 14.0,
        "key_ratios": [
          {"ratio": "Current Ratio", "value": 2.75, "benchmark": 2.0, "status": "Excellent"},
          {"ratio": "ROE", "value": 18.5, "benchmark": 12.0, "status": "Above Average"}
        ],
        "health_assessment": "Strong liquidity and profitability"
      }
    },
    "ProfitabilityAgent": {
      "status": "completed",
      "findings": {
        "gross_margin": 40.0,
        "net_margin": 15.1,
        "ebitda_margin": 21.0,
        "operating_margin": 18.2,
        "margin_trend": "improving",
        "yoy_improvement": 1.9,
        "opportunities": ["Cost optimization in operations", "Revenue mix enhancement"]
      }
    },
    "LiquidityAgent": {
      "status": "completed",
      "findings": {
        "cash_position": 850000,
        "working_capital": 1450000,
        "cash_conversion_cycle": 38,
        "days_sales_outstanding": 45,
        "liquidity_status": "Excellent",
        "recommendations": ["Optimize collection cycle", "Negotiate payment terms"]
      }
    },
    "FinancialTrendAgent": {
      "status": "completed",
      "findings": {
        "revenue_growth_yoy": 11.5,
        "revenue_cagr_3y": 9.8,
        "profit_growth": 15.2,
        "trend_direction": "Strong Positive",
        "seasonal_patterns": ["Q2 peak revenue", "Q4 strong margins"],
        "projections": {"next_year_growth": 10.5}
      }
    },
    "RiskAssessmentAgent": {
      "status": "completed",
      "findings": {
        "risk_score": 2,
        "risk_level": "Low",
        "compliance_status": {
          "zakat": "Compliant",
          "vat": "Compliant",
          "gosi": "Compliant"
        },
        "covenant_status": "All covenants met",
        "early_warnings": [],
        "mitigation_strategies": ["Maintain current leverage", "Build cash reserves"]
      }
    },
    "EfficiencyAgent": {
      "status": "completed",
      "findings": {
        "asset_turnover": 1.55,
        "inventory_turnover": 8.2,
        "receivables_turnover": 9.6,
        "efficiency_score": 78,
        "bottlenecks": ["DSO could be improved"],
        "improvement_opportunities": ["Reduce collection cycle by 10 days", "Optimize inventory levels"]
      }
    }
  },
  "summary": {
    "overall_health_score": 86,
    "key_insights": [
      "Strong financial position with excellent liquidity",
      "Profitability margins above industry average",
      "Low risk profile with full compliance",
      "Operational efficiency at 78%"
    ],
    "priority_actions": [
      "Accelerate receivables collection",
      "Optimize inventory management",
      "Maintain conservative leverage"
    ]
  }
}
```

### Get Specific Agent Analysis
```http
GET /api/analysis/agent/{agent_name}
Headers:
  X-Session-ID: {session_id}

Params:
  upload_id: upload_123456
```

---

## 📊 Data Retrieval

### Get Transactions
```http
GET /api/transactions
Headers:
  X-Session-ID: {session_id}

Query Parameters:
  upload_id: upload_123456
  page: 1
  limit: 50
  sort_by: date | amount | description
  order: asc | desc
  filter_type: all | credit | debit
  search: "GOSI"
  date_from: 2024-01-01
  date_to: 2024-01-31
  min_amount: 100
  max_amount: 10000
```

#### Success Response (200 OK)
```json
{
  "transactions": [
    {
      "id": "txn_001",
      "date": "2024-01-15",
      "description": "GOSI Payment",
      "amount": -3500.00,
      "balance": 121500.00,
      "category": "Government",
      "type": "debit",
      "tags": ["gosi", "compliance", "monthly"]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 342,
    "pages": 7
  },
  "summary": {
    "total_credits": 87500.00,
    "total_debits": 65678.00,
    "net": 21822.00,
    "count": 342
  }
}
```

### Get Financial Statements
```http
GET /api/financial/statements
Headers:
  X-Session-ID: {session_id}

Query Parameters:
  upload_id: upload_789456
  statement_type: balance_sheet | income_statement | cash_flow | all (default: all)
```

#### Success Response (200 OK)

**IMPORTANT:** The API returns the parsed financial statement data directly (not wrapped). The structure varies by file:

**Full Structure (when statement_type=all):**
```json
{
  "company_info": {
    "name": "International Human Resources Co.",
    "symbol": "9545 | SA15K0H4L4H6",
    "audit_status": "Audited",
    "rounding": "Actuals"
  },
  "periods": {
    "current": "2024",
    "prior": "2023"
  },
  "balance_sheet": {
    "assets": {
      "current": {
        "Bank balances and cash": {"current": 13394557, "prior": 8374914},
        "Total": {"current": 64124865, "prior": 40763658},
        "total": {"current": 64124865, "prior": 40763658}
      },
      "non_current": {
        "total": {"current": 21875013, "prior": 18573102}
      },
      "total": {
        "Total assets": {"current": 85999878, "prior": 59336760}
      }
    },
    "liabilities": {
      "current": {
        "Accrued expenses": {"current": 8931387, "prior": 3331359},
        "Total": {"current": 26395611, "prior": 10315045},
        "total": {"current": 26395611, "prior": 10315045}
      },
      "non_current": {
        "total": {"current": 10185079, "prior": 7542057}
      },
      "total": {
        "Total liabilities": {"current": 36580690, "prior": 17857102}
      }
    },
    "equity": {
      "Total equity": {"current": 49419188, "prior": 41479658},
      "Retained earnings (accumulated losses)": {"current": 11597809, "prior": 3611919}
    }
  },
  "income_statement": {
    "revenue": {
      "Total revenue": {"current": 192258806, "prior": 112087213}
    },
    "expenses": {
      "Other operating expenses": {"current": 161408921, "prior": 94128306},
      "Other expenses": {"current": 16677754, "prior": 12432532},
      "Zakat expenses on continuing operations for period": {"current": 1028072, "prior": 942745}
    },
    "profit_metrics": {
      "Profit (loss) for period": {"current": 8302790, "prior": 2534161},
      "Profit (loss) before zakat and income tax from continuing operations": {"current": 9330862, "prior": 3476906}
    }
  },
  "cash_flow": {
    "operating": {
      "Net cash flows from (used in) operating activities": {"current": -1544397, "prior": 3110815}
    },
    "investing": {
      "Net cash flows from (used in) investing activities": {"current": -2016639, "prior": -2999620}
    },
    "financing": {
      "Net cash flows from (used in) financing activities": {"current": 8580679, "prior": -1422335},
      "Net increase (decrease) in cash and cash equivalents": {"current": 5019643, "prior": -1311140}
    }
  },
  "ratios": {
    "current_ratio": 2.43,
    "quick_ratio": 2.43,
    "debt_to_equity": 0.74,
    "debt_ratio": 0.43,
    "profit_margin": 1.89,
    "roa": 4.22,
    "roe": 7.34
  }
}
```

**Note on File Structure Variations:**
- Different Excel files have different structures
- Some files include current/non-current breakdowns, others don't
- The parser extracts what's available in each file
- Frontend must handle both cases gracefully
- Field names are dynamic based on Excel column headers
- All monetary values use `.current` and `.prior` for two-period comparison

### Get Financial Ratios
```http
GET /api/financial/ratios
Headers:
  X-Session-ID: {session_id}

Query Parameters:
  upload_id: upload_789456
```

#### Success Response (200 OK)

**IMPORTANT:** The API returns ratios directly (not wrapped in `{ratios: ...}`). Values are already in percentage format (e.g., 2.43 means 2.43%, NOT 0.0243).

```json
{
  "current_ratio": 2.43,
  "quick_ratio": 2.43,
  "debt_to_equity": 0.74,
  "debt_ratio": 0.43,
  "profit_margin": 1.89,
  "roa": 4.22,
  "roe": 7.34
}
```

**Note on Ratio Format:**
- All ratios are calculated by the backend parser
- Percentage values are already multiplied (e.g., ROA: 4.22 = 4.22%)
- Frontend should display as-is, NOT multiply by 100 again
- Ratios correspond to the "current" period
- Some files may have additional ratios depending on available data

---

## 📊 Frontend Integration Notes

### Financial Statements Display

The frontend `FinancialOverview.tsx` component handles variable file structures:

**Data Normalization:**
```typescript
// Extract values with multiple fallbacks for different file structures
const currentAssets = bs.assets?.current?.total?.current
  || bs.assets?.current?.Total?.current
  || bs.assets?.current_assets?.total
  || 0

const totalAssets = bs.assets?.total?.['Total assets']?.current
  || bs.assets?.total?.total?.current
  || (currentAssets + fixedAssets)
  || 0
```

**Graceful Degradation:**
- Shows asset/liability breakdowns when available
- Falls back to totals-only display when breakdown is missing
- Handles missing ratios with default values
- Displays year-over-year data (current vs prior)

**Key Learnings:**
1. Field names vary between Excel files (e.g., "Total" vs "total")
2. Some files lack current/non-current breakdowns
3. Ratios are already percentages - don't multiply by 100
4. Frontend must try multiple field paths for each value

---

## 🤖 Agent Analysis (continued)

### Health Indicators

```json
  "health_indicators": {
    "overall_score": 86,
    "strengths": ["Excellent liquidity", "Strong profitability", "Conservative leverage"],
    "concerns": ["DSO could be improved", "Inventory turnover below benchmark"],
    "peer_comparison": "Above industry average"
  }
}
```

---

## 📈 Analytics & Insights

### Get Trends
```http
GET /api/analytics/trends
Headers:
  X-Session-ID: {session_id}

Query Parameters:
  upload_id: upload_123456
  period: daily | weekly | monthly
  metric: expenses | income | cash_flow | all
```

#### Success Response (200 OK)
```json
{
  "trends": {
    "expenses": [
      {"date": "2024-01-01", "value": 2150.00},
      {"date": "2024-01-02", "value": 3200.00}
    ],
    "income": [
      {"date": "2024-01-01", "value": 5500.00},
      {"date": "2024-01-02", "value": 4200.00}
    ],
    "cash_flow": [
      {"date": "2024-01-01", "value": 3350.00},
      {"date": "2024-01-02", "value": 1000.00}
    ]
  },
  "statistics": {
    "average_daily_expense": 2183.00,
    "average_daily_income": 2916.67,
    "volatility_score": 6.8,
    "trend_direction": "stable"
  }
}
```

### Get Categories
```http
GET /api/analytics/categories
Headers:
  X-Session-ID: {session_id}

Query Parameters:
  upload_id: upload_123456
```

#### Success Response (200 OK)
```json
{
  "categories": [
    {
      "name": "Payroll",
      "amount": 35000.00,
      "percentage": 40.0,
      "transaction_count": 12,
      "trend": "stable"
    },
    {
      "name": "Vendors",
      "amount": 18500.00,
      "percentage": 21.1,
      "transaction_count": 45,
      "trend": "increasing"
    },
    {
      "name": "GOSI",
      "amount": 3500.00,
      "percentage": 4.0,
      "transaction_count": 1,
      "trend": "stable"
    }
  ],
  "uncategorized": {
    "amount": 8678.00,
    "percentage": 9.9,
    "transaction_count": 28
  }
}
```

---

## 🔄 WebSocket Events

### Connection
```javascript
const ws = new WebSocket('ws://localhost:8001/ws/{session_id}');
```

### Event Types

#### Agent Analysis Progress
```json
{
  "type": "analysis_progress",
  "data": {
    "agent": "ExpenseAgent",
    "status": "processing",
    "progress": 45,
    "message": "Categorizing transactions..."
  }
}
```

#### Real-time Insights
```json
{
  "type": "insight",
  "data": {
    "agent": "TrendAnalystAgent",
    "insight": "Unusual spending pattern detected today",
    "severity": "info",
    "action": "Review transaction #txn_456"
  }
}
```

#### Chat Streaming
```json
{
  "type": "chat_stream",
  "data": {
    "chunk": "Based on my analysis",
    "agent": "BudgetAdvisorAgent",
    "complete": false
  }
}
```

---

## 🚨 Error Responses

### Standard Error Format
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Session expired or invalid",
    "details": {
      "session_id": "550e8400-e29b-41d4-a716-446655440000",
      "expired_at": "2024-01-01T10:00:00Z"
    },
    "request_id": "req_abc123",
    "timestamp": "2024-01-01T11:00:00Z"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Invalid or expired session |
| `FORBIDDEN` | 403 | Email not in whitelist |
| `NOT_FOUND` | 404 | Resource not found |
| `BAD_REQUEST` | 400 | Invalid request parameters |
| `FILE_TOO_LARGE` | 413 | Upload exceeds 50MB limit |
| `UNSUPPORTED_FORMAT` | 415 | File format not supported |
| `RATE_LIMITED` | 429 | Too many requests |
| `PROCESSING_ERROR` | 422 | Failed to process file |
| `AGENT_ERROR` | 500 | Agent processing failed |
| `SERVER_ERROR` | 500 | Internal server error |

### Rate Limiting
```http
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 95
  X-RateLimit-Reset: 1704153600
```

---

## 🔒 Security Headers

### Required Headers
```http
X-Session-ID: {session_id}  # All authenticated endpoints
Content-Type: application/json  # For JSON payloads
```

### Response Headers
```http
X-Request-ID: req_abc123  # For tracking
X-Processing-Time: 1234  # In milliseconds
X-Agent-Used: ExpenseAgent  # When applicable
```

---

## 📝 Request/Response Examples

### Complete Upload + Analysis Flow

#### 1. Verify Access
```bash
curl -X POST http://localhost:8001/api/auth/verify-access \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'
```

#### 2. Upload File
```bash
curl -X POST http://localhost:8001/api/upload \
  -H "X-Session-ID: 550e8400-e29b-41d4-a716-446655440000" \
  -F "file=@bank_statement.csv"
```

#### 3. Run Analysis
```bash
curl -X POST http://localhost:8001/api/analysis/full \
  -H "X-Session-ID: 550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -d '{"upload_id": "upload_123456", "agents": ["all"]}'
```

#### 4. Chat Query
```bash
curl -X POST http://localhost:8001/api/chat/message \
  -H "X-Session-ID: 550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What are my top expenses?",
    "context": {"upload_id": "upload_123456"}
  }'
```

---

## 🧪 Testing Endpoints

### Health Check
```http
GET /health

Response:
{
  "status": "healthy",
  "version": "1.0.0",
  "services": {
    "database": "connected",
    "chromadb": "connected",
    "ollama": "connected"
  }
}
```

### API Documentation
```http
GET /docs         # Swagger UI
GET /redoc        # ReDoc
GET /openapi.json # OpenAPI Schema
```

---

## 📊 Pagination

All list endpoints support pagination:

```http
GET /api/transactions?page=2&limit=50

Response includes:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 50,
    "total": 342,
    "pages": 7,
    "has_next": true,
    "has_prev": true
  }
}
```

---

## 🔍 Filtering & Sorting

### Filter Operators
- `eq` - Equals
- `ne` - Not equals
- `gt` - Greater than
- `gte` - Greater than or equal
- `lt` - Less than
- `lte` - Less than or equal
- `like` - Pattern matching
- `in` - In list

### Example
```http
GET /api/transactions?amount[gte]=1000&amount[lte]=5000&description[like]=%GOSI%
```

---

## 🌍 CORS Configuration

### Allowed Origins
```
Development: http://localhost:5173
Production: https://app.yomnai.com
```

### Allowed Methods
```
GET, POST, PUT, DELETE, OPTIONS
```

### Allowed Headers
```
Content-Type, X-Session-ID, Authorization
```

---

## 📚 SDK Examples

### JavaScript/TypeScript
```typescript
class YomnaiAPI {
  private baseURL = 'http://localhost:8001';
  private sessionId: string | null = null;

  async login(email: string): Promise<boolean> {
    const response = await fetch(`${this.baseURL}/api/auth/verify-access`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email })
    });

    const data = await response.json();
    if (data.authorized) {
      this.sessionId = data.session_id;
      return true;
    }
    return false;
  }

  async uploadFile(file: File): Promise<any> {
    const formData = new FormData();
    formData.append('file', file);

    return fetch(`${this.baseURL}/api/upload`, {
      method: 'POST',
      headers: { 'X-Session-ID': this.sessionId! },
      body: formData
    }).then(res => res.json());
  }

  async runAnalysis(uploadId: string): Promise<any> {
    return fetch(`${this.baseURL}/api/analysis/full`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': this.sessionId!
      },
      body: JSON.stringify({ upload_id: uploadId, agents: ['all'] })
    }).then(res => res.json());
  }
}
```

---

*This API documentation provides comprehensive details for integrating with the Yomnai backend. For implementation details, see [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md).*