# Database Schema Documentation - Yomnai Backend

## Overview

Yomnai uses a hybrid database approach:
- **SQLite** - For structured data (sessions, users, cache)
- **ChromaDB** - For vector embeddings and semantic search

---

## 🗄️ SQLite Database Schema

### Database: `yomnai.db`
Location: `yomnai_backend/data/yomnai.db`

---

## 📊 Tables

### 1. `authorized_users`
Stores pre-authorized email addresses that can access the system.

```sql
CREATE TABLE authorized_users (
    email TEXT PRIMARY KEY,
    authorized_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    authorized_by TEXT,
    first_access TIMESTAMP,
    last_access TIMESTAMP,
    access_count INTEGER DEFAULT 0,
    status TEXT DEFAULT 'active' CHECK(status IN ('active', 'blocked', 'pending')),
    metadata JSON,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_authorized_users_status ON authorized_users(status);
CREATE INDEX idx_authorized_users_last_access ON authorized_users(last_access);
```

**Sample Data:**
```sql
INSERT INTO authorized_users (email, authorized_by, metadata) VALUES
('john.doe@company.sa', 'admin', '{"source": "google_form", "company": "SABIC"}'),
('sarah.ahmed@firm.sa', 'admin', '{"source": "typeform", "plan": "premium"}');
```

---

### 2. `sessions`
Manages user sessions with 24-hour expiration.

```sql
CREATE TABLE sessions (
    session_id TEXT PRIMARY KEY,
    email TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    last_activity TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ip_address TEXT,
    user_agent TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    terminated_at TIMESTAMP,
    termination_reason TEXT,
    metadata JSON,
    FOREIGN KEY (email) REFERENCES authorized_users(email)
);

CREATE INDEX idx_sessions_email ON sessions(email);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
CREATE INDEX idx_sessions_is_active ON sessions(is_active);
```

---

### 3. `uploads`
Tracks all file uploads and their processing status.

```sql
CREATE TABLE uploads (
    upload_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    email TEXT NOT NULL,
    filename TEXT NOT NULL,
    file_size INTEGER,
    file_hash TEXT,
    document_type TEXT CHECK(document_type IN ('transactions', 'financial', 'unknown')),
    status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'processing', 'completed', 'failed')),
    upload_path TEXT,
    processed_at TIMESTAMP,
    error_message TEXT,
    metadata JSON,
    statistics JSON,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(session_id),
    FOREIGN KEY (email) REFERENCES authorized_users(email)
);

CREATE INDEX idx_uploads_session_id ON uploads(session_id);
CREATE INDEX idx_uploads_email ON uploads(email);
CREATE INDEX idx_uploads_status ON uploads(status);
CREATE INDEX idx_uploads_created_at ON uploads(created_at);
```

**Metadata JSON Structure:**
```json
{
  "rows": 342,
  "columns": ["Date", "Description", "Amount", "Balance"],
  "date_range": {
    "start": "2024-01-01",
    "end": "2024-01-31"
  },
  "currency": "SAR",
  "parser_used": "intelligent_csv_parser",
  "processing_time": 2.34
}
```

---

### 4. `analysis_results`
Stores results from agent analysis for caching and history.

```sql
CREATE TABLE analysis_results (
    analysis_id TEXT PRIMARY KEY,
    upload_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    email TEXT NOT NULL,
    agent_name TEXT NOT NULL,
    status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'processing', 'completed', 'failed')),
    result JSON,
    processing_time REAL,
    confidence_score REAL,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (upload_id) REFERENCES uploads(upload_id),
    FOREIGN KEY (session_id) REFERENCES sessions(session_id),
    FOREIGN KEY (email) REFERENCES authorized_users(email)
);

CREATE INDEX idx_analysis_upload_id ON analysis_results(upload_id);
CREATE INDEX idx_analysis_agent_name ON analysis_results(agent_name);
CREATE INDEX idx_analysis_created_at ON analysis_results(created_at);
```

**Result JSON Structure:**
```json
{
  "findings": {
    "total_expenses": 65678.00,
    "categories": [...],
    "insights": "...",
    "recommendations": [...]
  },
  "metrics": {
    "tokens_used": 450,
    "data_points": 342
  }
}
```

---

### 5. `chat_messages`
Stores chat history for context and analysis.

```sql
CREATE TABLE chat_messages (
    message_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    upload_id TEXT,
    email TEXT NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    agent_used TEXT,
    metadata JSON,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(session_id),
    FOREIGN KEY (upload_id) REFERENCES uploads(upload_id),
    FOREIGN KEY (email) REFERENCES authorized_users(email)
);

CREATE INDEX idx_chat_session_id ON chat_messages(session_id);
CREATE INDEX idx_chat_upload_id ON chat_messages(upload_id);
CREATE INDEX idx_chat_created_at ON chat_messages(created_at);
```

---

### 6. `access_logs`
Audit trail for all system access and actions.

```sql
CREATE TABLE access_logs (
    log_id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT,
    session_id TEXT,
    action TEXT NOT NULL,
    endpoint TEXT,
    method TEXT,
    status_code INTEGER,
    ip_address TEXT,
    user_agent TEXT,
    request_data JSON,
    response_time REAL,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_access_logs_email ON access_logs(email);
CREATE INDEX idx_access_logs_action ON access_logs(action);
CREATE INDEX idx_access_logs_created_at ON access_logs(created_at);
```

**Actions Include:**
- `login_attempt`
- `login_success`
- `login_failed`
- `file_upload`
- `analysis_request`
- `chat_message`
- `logout`
- `session_expired`

---

### 7. `cache_entries`
Query result caching for performance optimization.

```sql
CREATE TABLE cache_entries (
    cache_key TEXT PRIMARY KEY,
    cache_value JSON NOT NULL,
    cache_type TEXT,
    upload_id TEXT,
    expires_at TIMESTAMP,
    hit_count INTEGER DEFAULT 0,
    last_accessed TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (upload_id) REFERENCES uploads(upload_id)
);

CREATE INDEX idx_cache_expires_at ON cache_entries(expires_at);
CREATE INDEX idx_cache_upload_id ON cache_entries(upload_id);
```

---

### 8. `rate_limits`
Track API usage for rate limiting.

```sql
CREATE TABLE rate_limits (
    email TEXT NOT NULL,
    endpoint TEXT NOT NULL,
    window_start TIMESTAMP NOT NULL,
    request_count INTEGER DEFAULT 1,
    PRIMARY KEY (email, endpoint, window_start)
);

CREATE INDEX idx_rate_limits_window_start ON rate_limits(window_start);
```

---

### 9. `admin_actions`
Log administrative actions for audit.

```sql
CREATE TABLE admin_actions (
    action_id INTEGER PRIMARY KEY AUTOINCREMENT,
    admin_email TEXT NOT NULL,
    action_type TEXT NOT NULL,
    target_email TEXT,
    description TEXT,
    metadata JSON,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_admin_actions_created_at ON admin_actions(created_at);
```

---

## 🔄 Views

### `v_active_sessions`
```sql
CREATE VIEW v_active_sessions AS
SELECT
    s.session_id,
    s.email,
    s.created_at,
    s.expires_at,
    s.last_activity,
    u.access_count,
    u.status as user_status
FROM sessions s
JOIN authorized_users u ON s.email = u.email
WHERE s.is_active = TRUE
  AND s.expires_at > datetime('now');
```

### `v_user_activity`
```sql
CREATE VIEW v_user_activity AS
SELECT
    u.email,
    u.status,
    u.first_access,
    u.last_access,
    u.access_count,
    COUNT(DISTINCT s.session_id) as total_sessions,
    COUNT(DISTINCT up.upload_id) as total_uploads,
    COUNT(DISTINCT cm.message_id) as total_messages
FROM authorized_users u
LEFT JOIN sessions s ON u.email = s.email
LEFT JOIN uploads up ON u.email = up.email
LEFT JOIN chat_messages cm ON u.email = cm.email
GROUP BY u.email;
```

### `v_daily_statistics`
```sql
CREATE VIEW v_daily_statistics AS
SELECT
    DATE(created_at) as date,
    COUNT(DISTINCT email) as unique_users,
    COUNT(DISTINCT session_id) as sessions,
    COUNT(CASE WHEN action = 'file_upload' THEN 1 END) as uploads,
    COUNT(CASE WHEN action = 'analysis_request' THEN 1 END) as analyses,
    COUNT(CASE WHEN action = 'chat_message' THEN 1 END) as chat_messages
FROM access_logs
GROUP BY DATE(created_at);
```

---

## 🗂️ ChromaDB Collections

### Collection: `transactions`

**Purpose:** Store transaction embeddings for semantic search

**Schema:**
```python
{
    "id": "txn_uuid",
    "embedding": [...],  # 384-dimensional vector
    "metadata": {
        "session_id": "550e8400-e29b-41d4",
        "upload_id": "upload_123456",
        "email": "user@example.com",
        "date": "2024-01-15",
        "description": "GOSI Payment",
        "amount": -3500.00,
        "category": "Government",
        "type": "debit"
    },
    "document": "2024-01-15 GOSI Payment SAR -3500.00 Government"
}
```

### Collection: `financial_statements`

**Purpose:** Store financial statement data for analysis

**Schema:**
```python
{
    "id": "stmt_uuid",
    "embedding": [...],  # 384-dimensional vector
    "metadata": {
        "session_id": "550e8400-e29b-41d4",
        "upload_id": "upload_789456",
        "email": "user@example.com",
        "statement_type": "balance_sheet",
        "period": "2024-Q1",
        "metric": "total_assets",
        "value": 5200000
    },
    "document": "Balance Sheet Q1 2024 Total Assets SAR 5,200,000"
}
```

### Collection: `analysis_cache`

**Purpose:** Cache analysis results for quick retrieval

**Schema:**
```python
{
    "id": "cache_uuid",
    "embedding": [...],  # Query embedding
    "metadata": {
        "query": "What are my top expenses?",
        "upload_id": "upload_123456",
        "agent": "ExpenseAgent",
        "timestamp": "2024-01-15T10:30:00Z"
    },
    "document": "Response text from agent..."
}
```

---

## 🔧 Database Initialization Script

```sql
-- init_db.sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;

-- Create all tables (as defined above)
-- ...

-- Insert default admin user
INSERT INTO authorized_users (email, authorized_by, status, metadata)
VALUES ('admin@yomnai.com', 'system', 'active', '{"role": "admin"}');

-- Create triggers for updated_at
CREATE TRIGGER update_authorized_users_timestamp
AFTER UPDATE ON authorized_users
BEGIN
    UPDATE authorized_users SET updated_at = CURRENT_TIMESTAMP
    WHERE email = NEW.email;
END;

-- Clean up expired sessions daily
CREATE TRIGGER cleanup_expired_sessions
AFTER INSERT ON sessions
BEGIN
    DELETE FROM sessions
    WHERE expires_at < datetime('now', '-1 day')
      AND is_active = TRUE;
END;
```

---

## 📈 Performance Indexes

### Recommended Indexes for Optimization
```sql
-- For fast session lookups
CREATE INDEX idx_sessions_lookup ON sessions(session_id, email, is_active);

-- For upload history queries
CREATE INDEX idx_uploads_history ON uploads(email, created_at DESC);

-- For chat context retrieval
CREATE INDEX idx_chat_context ON chat_messages(session_id, upload_id, created_at);

-- For rate limiting checks
CREATE INDEX idx_rate_limits_check ON rate_limits(email, endpoint, window_start DESC);

-- For access log analysis
CREATE INDEX idx_access_logs_analysis ON access_logs(email, action, created_at DESC);
```

---

## 🔄 Migration from Streamlit

### Data Migration Strategy

```python
# migrate_from_streamlit.py
import sqlite3
import chromadb
from pathlib import Path

def migrate_data():
    # 1. Create new SQLite database
    conn = sqlite3.connect('yomnai.db')

    # 2. Execute schema creation
    with open('init_db.sql', 'r') as f:
        conn.executescript(f.read())

    # 3. Migrate ChromaDB collections
    old_client = chromadb.PersistentClient(path="./data/chroma_db")
    new_client = chromadb.PersistentClient(path="./yomnai_backend/data/chroma_db")

    # Copy collections
    for collection_name in ['transactions', 'financial_statements']:
        old_collection = old_client.get_collection(collection_name)
        new_collection = new_client.create_collection(collection_name)

        # Copy all data
        data = old_collection.get()
        if data['ids']:
            new_collection.add(
                ids=data['ids'],
                embeddings=data['embeddings'],
                metadatas=data['metadatas'],
                documents=data['documents']
            )

    print("Migration completed successfully")
```

---

## 🎯 Query Examples

### Get User Activity Summary
```sql
SELECT
    email,
    last_access,
    access_count,
    (SELECT COUNT(*) FROM uploads WHERE email = au.email) as total_uploads,
    (SELECT COUNT(*) FROM chat_messages WHERE email = au.email) as total_messages
FROM authorized_users au
WHERE status = 'active'
ORDER BY last_access DESC;
```

### Find Heavy Users
```sql
SELECT
    email,
    COUNT(*) as action_count,
    COUNT(DISTINCT DATE(created_at)) as active_days
FROM access_logs
WHERE created_at >= datetime('now', '-30 days')
GROUP BY email
HAVING action_count > 100
ORDER BY action_count DESC;
```

### Session Analytics
```sql
SELECT
    DATE(created_at) as date,
    COUNT(*) as sessions,
    COUNT(DISTINCT email) as unique_users,
    AVG((julianday(last_activity) - julianday(created_at)) * 24 * 60) as avg_duration_minutes
FROM sessions
WHERE created_at >= datetime('now', '-7 days')
GROUP BY DATE(created_at);
```

### Cache Performance
```sql
SELECT
    cache_type,
    COUNT(*) as entries,
    SUM(hit_count) as total_hits,
    AVG(hit_count) as avg_hits,
    COUNT(CASE WHEN expires_at > datetime('now') THEN 1 END) as active_entries
FROM cache_entries
GROUP BY cache_type;
```

---

## 🔐 Security Considerations

### SQL Injection Prevention
- Use parameterized queries always
- Never concatenate user input into SQL
- Validate all inputs with Pydantic

### Data Privacy
- No sensitive data in logs
- Encrypt session IDs
- Regular cleanup of old data
- Audit trail for compliance

### Access Control
```sql
-- Example: Check if user can access upload
SELECT 1 FROM uploads
WHERE upload_id = ?
  AND email = ?
  AND status = 'completed';
```

---

## 📊 Database Maintenance

### Daily Cleanup Job
```sql
-- Remove expired sessions
DELETE FROM sessions
WHERE expires_at < datetime('now', '-1 day');

-- Clean old cache entries
DELETE FROM cache_entries
WHERE expires_at < datetime('now');

-- Archive old access logs (keep 90 days)
DELETE FROM access_logs
WHERE created_at < datetime('now', '-90 days');

-- Update database statistics
ANALYZE;
VACUUM;
```

### Backup Strategy
```bash
# Daily backup script
#!/bin/bash
BACKUP_DIR="/backups/yomnai"
DATE=$(date +%Y%m%d)

# Backup SQLite
sqlite3 yomnai.db ".backup ${BACKUP_DIR}/yomnai_${DATE}.db"

# Backup ChromaDB
tar -czf ${BACKUP_DIR}/chromadb_${DATE}.tar.gz data/chroma_db/
```

---

## 📈 Monitoring Queries

### System Health Check
```sql
SELECT
    'Active Sessions' as metric,
    COUNT(*) as value
FROM v_active_sessions
UNION ALL
SELECT
    'Uploads Today',
    COUNT(*)
FROM uploads
WHERE DATE(created_at) = DATE('now')
UNION ALL
SELECT
    'Total Users',
    COUNT(*)
FROM authorized_users
WHERE status = 'active';
```

---

*This schema provides a robust foundation for the Yomnai backend with proper normalization, indexing, and security considerations.*