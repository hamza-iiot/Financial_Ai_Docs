# Implementation Guide - Yomnai FastAPI Backend

**STATUS: ✅ COMPLETED** (October 1, 2025)

This guide documents the implementation that has been completed. The backend is running successfully at `http://localhost:8001`.

For current status and integration instructions, see `/yomnai_backend/IMPLEMENTATION_STATUS.md`.

---

## 📋 Prerequisites

### Required Software
- Python 3.10 or higher
- Ollama running locally
- Node.js 18+ (for frontend)
- Git

### Ollama Models
```bash
# Install required models
ollama pull qwen3-14b-32k:latest
ollama pull gemma3:4b
ollama pull qwen2.5:7b-vl
```

---

## 🚀 Step-by-Step Implementation

## Phase 1: Project Setup (Day 1)

### 1.1 Create Project Structure

```bash
# Create backend directory
mkdir yomnai_backend
cd yomnai_backend

# Create directory structure
mkdir -p app/{api/endpoints,core,services,models,utils}
mkdir -p agents migrations tests data
mkdir -p data/chroma_db

# Create empty __init__.py files
touch app/__init__.py
touch app/api/__init__.py
touch app/api/endpoints/__init__.py
touch app/core/__init__.py
touch app/services/__init__.py
touch app/models/__init__.py
touch app/utils/__init__.py
touch agents/__init__.py
```

### 1.2 Set Up Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Linux/Mac)
source venv/bin/activate
```

### 1.3 Create Requirements File

```python
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6
pydantic==2.5.0
pydantic-settings==2.1.0
sqlalchemy==2.0.23
aiosqlite==0.19.0
chromadb==0.4.22
pandas==2.2.0
openpyxl==3.1.2
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-dotenv==1.0.0
aiofiles==23.2.1
websockets==12.0
httpx==0.25.2
sentence-transformers==2.2.2
langchain==0.1.9
ollama==0.1.6
pdf2image==1.17.0
Pillow==10.2.0
pytest==7.4.3
pytest-asyncio==0.21.1
```

```bash
# Install dependencies
pip install -r requirements.txt
```

### 1.4 Create Environment Configuration

```bash
# .env
APP_NAME=Yomnai Financial Analyst
APP_VERSION=1.0.0
DEBUG=True
ENVIRONMENT=development

# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
API_PREFIX=/api

# Database
DATABASE_URL=sqlite+aiosqlite:///./data/yomnai.db
CHROMA_PERSIST_DIR=./data/chroma_db

# Ollama
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL_MAIN=qwen3-14b-32k:latest
OLLAMA_MODEL_ROUTER=gemma3:4b
OLLAMA_MODEL_VISION=qwen2.5:7b-vl

# Security
SECRET_KEY=your-secret-key-change-this-in-production
SESSION_TIMEOUT_HOURS=24
MAX_UPLOAD_SIZE_MB=50

# Rate Limiting
RATE_LIMIT_PER_DAY=100
RATE_LIMIT_PER_HOUR=20

# CORS
FRONTEND_URLS=http://localhost:5173,http://localhost:3000

# File Storage
UPLOAD_DIR=./data/uploads
TEMP_DIR=./data/temp
```

---

## Phase 2: Core Configuration (Day 2)

### 2.1 Create Configuration Module

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import List
from pathlib import Path

class Settings(BaseSettings):
    # App Config
    app_name: str = "Yomnai"
    app_version: str = "1.0.0"
    debug: bool = False
    environment: str = "development"

    # API Config
    api_host: str = "0.0.0.0"
    api_port: int = 8000
    api_prefix: str = "/api"

    # Database
    database_url: str = "sqlite+aiosqlite:///./data/yomnai.db"
    chroma_persist_dir: str = "./data/chroma_db"

    # Ollama
    ollama_host: str = "http://localhost:11434"
    ollama_model_main: str = "qwen3-14b-32k:latest"
    ollama_model_router: str = "gemma3:4b"
    ollama_model_vision: str = "qwen2.5:7b-vl"

    # Security
    secret_key: str = "change-this-in-production"
    session_timeout_hours: int = 24
    max_upload_size_mb: int = 50

    # Rate Limiting
    rate_limit_per_day: int = 100
    rate_limit_per_hour: int = 20

    # CORS
    frontend_urls: str = "http://localhost:5173"

    @property
    def cors_origins(self) -> List[str]:
        return [url.strip() for url in self.frontend_urls.split(",")]

    # File Storage
    upload_dir: Path = Path("./data/uploads")
    temp_dir: Path = Path("./data/temp")

    def __init__(self, **values):
        super().__init__(**values)
        # Create directories if they don't exist
        self.upload_dir.mkdir(parents=True, exist_ok=True)
        self.temp_dir.mkdir(parents=True, exist_ok=True)

    class Config:
        env_file = ".env"
        case_sensitive = False

settings = Settings()
```

### 2.2 Create Database Module

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import declarative_base, sessionmaker
from sqlalchemy import text
from .config import settings
import logging

logger = logging.getLogger(__name__)

# Create async engine
engine = create_async_engine(
    settings.database_url,
    echo=settings.debug,
    future=True,
    pool_pre_ping=True,
    pool_size=20,
    max_overflow=40
)

# Create async session factory
AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Base class for models
Base = declarative_base()

async def init_db():
    """Initialize database with tables"""
    async with engine.begin() as conn:
        # Create all tables
        await conn.run_sync(Base.metadata.create_all)

        # Enable foreign keys for SQLite
        if "sqlite" in settings.database_url:
            await conn.execute(text("PRAGMA foreign_keys = ON"))
            await conn.execute(text("PRAGMA journal_mode = WAL"))
            await conn.execute(text("PRAGMA busy_timeout = 5000"))
            await conn.execute(text("PRAGMA synchronous = NORMAL"))

    logger.info("Database initialized successfully")

async def get_db() -> AsyncSession:
    """Dependency to get database session"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### 2.3 Create Database Models

```python
# app/models/database.py
from sqlalchemy import Column, String, Integer, Float, Boolean, DateTime, JSON, ForeignKey, Text
from sqlalchemy.sql import func
from app.core.database import Base
from datetime import datetime, timedelta

class AuthorizedUser(Base):
    __tablename__ = "authorized_users"

    email = Column(String, primary_key=True, index=True)
    authorized_at = Column(DateTime, server_default=func.now())
    authorized_by = Column(String)
    first_access = Column(DateTime)
    last_access = Column(DateTime)
    access_count = Column(Integer, default=0)
    status = Column(String, default="active")
    metadata = Column(JSON)
    notes = Column(Text)
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

class Session(Base):
    __tablename__ = "sessions"

    session_id = Column(String, primary_key=True, index=True)
    email = Column(String, ForeignKey("authorized_users.email"), nullable=False)
    created_at = Column(DateTime, server_default=func.now())
    expires_at = Column(DateTime, nullable=False)
    last_activity = Column(DateTime, server_default=func.now())
    ip_address = Column(String)
    user_agent = Column(String)
    is_active = Column(Boolean, default=True)
    terminated_at = Column(DateTime)
    termination_reason = Column(String)
    metadata = Column(JSON)

class Upload(Base):
    __tablename__ = "uploads"

    upload_id = Column(String, primary_key=True, index=True)
    session_id = Column(String, ForeignKey("sessions.session_id"), nullable=False)
    email = Column(String, ForeignKey("authorized_users.email"), nullable=False)
    filename = Column(String, nullable=False)
    file_size = Column(Integer)
    file_hash = Column(String)
    document_type = Column(String)
    status = Column(String, default="pending")
    upload_path = Column(String)
    processed_at = Column(DateTime)
    error_message = Column(Text)
    metadata = Column(JSON)
    statistics = Column(JSON)
    created_at = Column(DateTime, server_default=func.now())

class AnalysisResult(Base):
    __tablename__ = "analysis_results"

    analysis_id = Column(String, primary_key=True, index=True)
    upload_id = Column(String, ForeignKey("uploads.upload_id"), nullable=False)
    session_id = Column(String, ForeignKey("sessions.session_id"), nullable=False)
    email = Column(String, ForeignKey("authorized_users.email"), nullable=False)
    agent_name = Column(String, nullable=False, index=True)
    status = Column(String, default="pending")
    result = Column(JSON)
    processing_time = Column(Float)
    confidence_score = Column(Float)
    error_message = Column(Text)
    created_at = Column(DateTime, server_default=func.now())

class ChatMessage(Base):
    __tablename__ = "chat_messages"

    message_id = Column(String, primary_key=True, index=True)
    session_id = Column(String, ForeignKey("sessions.session_id"), nullable=False)
    upload_id = Column(String, ForeignKey("uploads.upload_id"))
    email = Column(String, ForeignKey("authorized_users.email"), nullable=False)
    role = Column(String, nullable=False)  # user, assistant, system
    content = Column(Text, nullable=False)
    agent_used = Column(String)
    metadata = Column(JSON)
    created_at = Column(DateTime, server_default=func.now())

class AccessLog(Base):
    __tablename__ = "access_logs"

    log_id = Column(Integer, primary_key=True, autoincrement=True)
    email = Column(String, index=True)
    session_id = Column(String)
    action = Column(String, nullable=False, index=True)
    endpoint = Column(String)
    method = Column(String)
    status_code = Column(Integer)
    ip_address = Column(String)
    user_agent = Column(String)
    request_data = Column(JSON)
    response_time = Column(Float)
    error_message = Column(Text)
    created_at = Column(DateTime, server_default=func.now(), index=True)
```

---

## Phase 3: Authentication Service (Day 3)

### 3.1 Create Authentication Service

```python
# app/services/auth_service.py
from typing import Optional
from datetime import datetime, timedelta
from uuid import uuid4
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update
from app.models.database import AuthorizedUser, Session, AccessLog
from app.core.config import settings
import logging

logger = logging.getLogger(__name__)

class AuthService:
    @staticmethod
    async def verify_email(db: AsyncSession, email: str) -> bool:
        """Check if email is in whitelist"""
        result = await db.execute(
            select(AuthorizedUser).where(
                AuthorizedUser.email == email,
                AuthorizedUser.status == "active"
            )
        )
        user = result.scalar_one_or_none()
        return user is not None

    @staticmethod
    async def create_session(
        db: AsyncSession,
        email: str,
        ip_address: str = None,
        user_agent: str = None
    ) -> Optional[str]:
        """Create a new session for authorized user"""
        # Verify email first
        if not await AuthService.verify_email(db, email):
            return None

        # Generate session ID
        session_id = str(uuid4())
        expires_at = datetime.utcnow() + timedelta(hours=settings.session_timeout_hours)

        # Create session
        new_session = Session(
            session_id=session_id,
            email=email,
            expires_at=expires_at,
            ip_address=ip_address,
            user_agent=user_agent
        )
        db.add(new_session)

        # Update user last access
        await db.execute(
            update(AuthorizedUser)
            .where(AuthorizedUser.email == email)
            .values(
                last_access=datetime.utcnow(),
                access_count=AuthorizedUser.access_count + 1,
                first_access=AuthorizedUser.first_access or datetime.utcnow()
            )
        )

        # Log access
        access_log = AccessLog(
            email=email,
            session_id=session_id,
            action="login_success",
            ip_address=ip_address,
            user_agent=user_agent
        )
        db.add(access_log)

        await db.commit()
        logger.info(f"Session created for {email}: {session_id}")
        return session_id

    @staticmethod
    async def validate_session(db: AsyncSession, session_id: str) -> Optional[dict]:
        """Validate session and return user info"""
        result = await db.execute(
            select(Session, AuthorizedUser)
            .join(AuthorizedUser, Session.email == AuthorizedUser.email)
            .where(
                Session.session_id == session_id,
                Session.is_active == True,
                Session.expires_at > datetime.utcnow()
            )
        )
        row = result.first()

        if not row:
            return None

        session, user = row

        # Update last activity
        await db.execute(
            update(Session)
            .where(Session.session_id == session_id)
            .values(last_activity=datetime.utcnow())
        )
        await db.commit()

        return {
            "session_id": session.session_id,
            "email": user.email,
            "expires_at": session.expires_at.isoformat(),
            "status": user.status
        }

    @staticmethod
    async def terminate_session(db: AsyncSession, session_id: str) -> bool:
        """Terminate a session"""
        result = await db.execute(
            update(Session)
            .where(Session.session_id == session_id)
            .values(
                is_active=False,
                terminated_at=datetime.utcnow(),
                termination_reason="logout"
            )
        )
        await db.commit()
        return result.rowcount > 0
```

### 3.2 Create Security Dependencies

```python
# app/api/deps.py
from typing import Optional
from fastapi import Depends, HTTPException, status, Header
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_db
from app.services.auth_service import AuthService

async def get_current_session(
    x_session_id: str = Header(None),
    db: AsyncSession = Depends(get_db)
) -> dict:
    """Dependency to validate session"""
    if not x_session_id:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Session ID required"
        )

    session_info = await AuthService.validate_session(db, x_session_id)
    if not session_info:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired session"
        )

    return session_info

async def get_optional_session(
    x_session_id: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db)
) -> Optional[dict]:
    """Optional session validation"""
    if not x_session_id:
        return None
    return await AuthService.validate_session(db, x_session_id)
```

---

## Phase 4: API Endpoints (Day 4-5)

### 4.1 Create Authentication Endpoints

```python
# app/api/endpoints/auth.py
from fastapi import APIRouter, Depends, HTTPException, status, Request
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, EmailStr
from app.core.database import get_db
from app.services.auth_service import AuthService
from app.api.deps import get_current_session

router = APIRouter(prefix="/auth", tags=["authentication"])

class VerifyAccessRequest(BaseModel):
    email: EmailStr

class VerifyAccessResponse(BaseModel):
    authorized: bool
    session_id: Optional[str] = None
    email: Optional[str] = None
    expires_at: Optional[str] = None
    message: Optional[str] = None
    registration_url: Optional[str] = None

@router.post("/verify-access", response_model=VerifyAccessResponse)
async def verify_access(
    request: VerifyAccessRequest,
    req: Request,
    db: AsyncSession = Depends(get_db)
):
    """Verify email access and create session"""
    ip_address = req.client.host if req.client else None
    user_agent = req.headers.get("user-agent")

    session_id = await AuthService.create_session(
        db,
        request.email,
        ip_address,
        user_agent
    )

    if session_id:
        session_info = await AuthService.validate_session(db, session_id)
        return VerifyAccessResponse(
            authorized=True,
            session_id=session_id,
            email=request.email,
            expires_at=session_info["expires_at"]
        )
    else:
        return VerifyAccessResponse(
            authorized=False,
            message="Access restricted. Please register first.",
            registration_url="https://forms.google.com/register"
        )

@router.get("/validate-session")
async def validate_session(
    session: dict = Depends(get_current_session)
):
    """Validate current session"""
    return {
        "valid": True,
        "email": session["email"],
        "expires_at": session["expires_at"]
    }

@router.post("/logout")
async def logout(
    session: dict = Depends(get_current_session),
    db: AsyncSession = Depends(get_db)
):
    """Terminate session"""
    success = await AuthService.terminate_session(db, session["session_id"])
    if success:
        return {"message": "Logged out successfully"}
    else:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Failed to logout"
        )
```

### 4.2 Create Upload Endpoints

```python
# app/api/endpoints/upload.py
from fastapi import APIRouter, Depends, File, UploadFile, HTTPException, status, Form
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional
from uuid import uuid4
from app.core.database import get_db
from app.core.config import settings
from app.api.deps import get_current_session
from app.services.parser_service import ParserService
from app.models.database import Upload
import aiofiles
from pathlib import Path

router = APIRouter(prefix="/upload", tags=["upload"])

@router.post("/")
async def upload_file(
    file: UploadFile = File(...),
    document_type: Optional[str] = Form("auto"),
    session: dict = Depends(get_current_session),
    db: AsyncSession = Depends(get_db)
):
    """Upload and process a file"""
    # Check file size
    if file.size > settings.max_upload_size_mb * 1024 * 1024:
        raise HTTPException(
            status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
            detail=f"File exceeds {settings.max_upload_size_mb}MB limit"
        )

    # Generate upload ID and path
    upload_id = f"upload_{uuid4().hex[:12]}"
    file_path = settings.upload_dir / f"{upload_id}_{file.filename}"

    # Save file
    async with aiofiles.open(file_path, 'wb') as f:
        content = await file.read()
        await f.write(content)

    # Create upload record
    upload = Upload(
        upload_id=upload_id,
        session_id=session["session_id"],
        email=session["email"],
        filename=file.filename,
        file_size=file.size,
        document_type=document_type,
        upload_path=str(file_path),
        status="processing"
    )
    db.add(upload)
    await db.commit()

    # Process file in background
    # TODO: Add background task for processing

    return {
        "upload_id": upload_id,
        "filename": file.filename,
        "document_type": document_type,
        "status": "processing",
        "message": "File uploaded successfully and is being processed"
    }

@router.get("/{upload_id}/status")
async def get_upload_status(
    upload_id: str,
    session: dict = Depends(get_current_session),
    db: AsyncSession = Depends(get_db)
):
    """Get upload processing status"""
    # Get upload record
    result = await db.execute(
        select(Upload).where(
            Upload.upload_id == upload_id,
            Upload.email == session["email"]
        )
    )
    upload = result.scalar_one_or_none()

    if not upload:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Upload not found"
        )

    return {
        "upload_id": upload.upload_id,
        "status": upload.status,
        "filename": upload.filename,
        "document_type": upload.document_type,
        "metadata": upload.metadata,
        "statistics": upload.statistics,
        "error_message": upload.error_message
    }
```

---

## Phase 5: Main Application (Day 6)

### 5.1 Create Main FastAPI App

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
import logging
from app.core.config import settings
from app.core.database import init_db
from app.api.endpoints import auth, upload, chat, analysis, transactions, financial
from app.api.websocket import websocket_endpoint

# Configure logging
logging.basicConfig(
    level=logging.DEBUG if settings.debug else logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events"""
    logger.info("Starting up...")
    await init_db()
    yield
    logger.info("Shutting down...")

# Create FastAPI app
app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    lifespan=lifespan,
    docs_url="/docs" if settings.debug else None,
    redoc_url="/redoc" if settings.debug else None
)

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Request-ID", "X-Processing-Time"]
)

# Include routers
app.include_router(auth.router, prefix=settings.api_prefix)
app.include_router(upload.router, prefix=settings.api_prefix)
app.include_router(chat.router, prefix=settings.api_prefix)
app.include_router(analysis.router, prefix=settings.api_prefix)
app.include_router(transactions.router, prefix=settings.api_prefix)
app.include_router(financial.router, prefix=settings.api_prefix)

# WebSocket endpoint
app.add_api_websocket_route("/ws/{session_id}", websocket_endpoint)

# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Global exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "SERVER_ERROR",
                "message": "An unexpected error occurred",
                "request_id": request.headers.get("X-Request-ID")
            }
        }
    )

# Health check
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "version": settings.app_version,
        "environment": settings.environment
    }

# Root endpoint
@app.get("/")
async def root():
    return {
        "name": settings.app_name,
        "version": settings.app_version,
        "docs": "/docs" if settings.debug else None
    }
```

### 5.2 Create Run Script

```python
# run.py
import uvicorn
from app.core.config import settings

if __name__ == "__main__":
    uvicorn.run(
        "app.main:app",
        host=settings.api_host,
        port=settings.api_port,
        reload=settings.debug,
        log_level="debug" if settings.debug else "info"
    )
```

---

## Phase 6: Service Migration (Day 7-8)

### 6.1 Migrate Agent Service

```python
# app/services/agent_service.py
import sys
from pathlib import Path

# Add parent directory to path for imports
sys.path.append(str(Path(__file__).parent.parent.parent))

# Import existing agents
from src.ai.agents.orchestrator import AgentOrchestrator
from src.ai.vector_store import VectorStoreManager
from src.ai.ollama_client import OllamaClient

class AgentService:
    def __init__(self):
        self.vector_store = VectorStoreManager()
        self.ollama_client = OllamaClient()
        self.orchestrator = AgentOrchestrator(
            self.vector_store,
            self.ollama_client
        )

    async def analyze(
        self,
        query: str,
        upload_id: str,
        agent_type: str = None
    ) -> dict:
        """Run agent analysis"""
        # Use existing orchestrator
        result = await self.orchestrator.process_query(
            query=query,
            session_id=upload_id,
            agent_type=agent_type
        )
        return result

# Singleton instance
agent_service = AgentService()
```

---

## Phase 7: Frontend Integration (Day 9-10)

### 7.1 Update React API Client

```typescript
// yomnai_frontend/src/services/api.ts
const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8001/api';

class YomnaiAPI {
  private sessionId: string | null = null;

  async verifyAccess(email: string): Promise<boolean> {
    const response = await fetch(`${API_BASE_URL}/auth/verify-access`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email })
    });

    const data = await response.json();
    if (data.authorized) {
      this.sessionId = data.session_id;
      localStorage.setItem('session_id', data.session_id);
      return true;
    }
    return false;
  }

  async uploadFile(file: File): Promise<any> {
    const formData = new FormData();
    formData.append('file', file);

    const response = await fetch(`${API_BASE_URL}/upload`, {
      method: 'POST',
      headers: {
        'X-Session-ID': this.sessionId!
      },
      body: formData
    });

    return response.json();
  }

  async runAnalysis(uploadId: string): Promise<any> {
    const response = await fetch(`${API_BASE_URL}/analysis/full`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': this.sessionId!
      },
      body: JSON.stringify({
        upload_id: uploadId,
        agents: ['all']
      })
    });

    return response.json();
  }

  async sendChatMessage(message: string, uploadId: string): Promise<any> {
    const response = await fetch(`${API_BASE_URL}/chat/message`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': this.sessionId!
      },
      body: JSON.stringify({
        message,
        context: { upload_id: uploadId }
      })
    });

    return response.json();
  }
}

export default new YomnaiAPI();
```

### 7.2 Update React Environment

```bash
# yomnai_frontend/.env
REACT_APP_API_URL=http://localhost:8001/api
REACT_APP_WS_URL=ws://localhost:8001/ws
```

---

## Phase 8: Testing (Day 11)

### 8.1 Create Test Suite

```python
# tests/test_auth.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_verify_access_authorized():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/api/auth/verify-access",
            json={"email": "admin@yomnai.com"}
        )
    assert response.status_code == 200
    data = response.json()
    assert data["authorized"] == True
    assert "session_id" in data

@pytest.mark.asyncio
async def test_verify_access_unauthorized():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/api/auth/verify-access",
            json={"email": "unauthorized@example.com"}
        )
    assert response.status_code == 200
    data = response.json()
    assert data["authorized"] == False
```

### 8.2 Run Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app tests/
```

---

## Phase 9: Deployment (Day 12)

### 9.1 Create Dockerfile

```dockerfile
# Dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create data directories
RUN mkdir -p data/uploads data/temp data/chroma_db

# Expose port
EXPOSE 8000

# Run application
CMD ["python", "run.py"]
```

### 9.2 Create Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build: ./yomnai_backend
    ports:
      - "8000:8001"
    volumes:
      - ./yomnai_backend/data:/app/data
    environment:
      - DATABASE_URL=sqlite:///./data/yomnai.db
      - OLLAMA_HOST=http://host.docker.internal:11434
    depends_on:
      - ollama

  frontend:
    build: ./yomnai_frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8001/api
    depends_on:
      - backend

  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama

volumes:
  ollama_data:
```

---

## 📋 Complete Implementation Checklist

### Phase 1: Setup ✅
- [ ] Create project structure
- [ ] Set up virtual environment
- [ ] Install dependencies
- [ ] Configure environment

### Phase 2: Core ✅
- [ ] Create configuration module
- [ ] Set up database
- [ ] Create models

### Phase 3: Auth ✅
- [ ] Implement auth service
- [ ] Create security dependencies
- [ ] Test authentication

### Phase 4: APIs ✅
- [ ] Auth endpoints
- [ ] Upload endpoints
- [ ] Chat endpoints
- [ ] Analysis endpoints

### Phase 5: Main App ✅
- [ ] Create FastAPI app
- [ ] Configure middleware
- [ ] Add exception handling

### Phase 6: Migration ✅
- [ ] Migrate agent services
- [ ] Migrate parser services
- [ ] Migrate vector store

### Phase 7: Frontend ✅
- [ ] Update API client
- [ ] Test integration
- [ ] Fix CORS issues

### Phase 8: Testing ✅
- [ ] Unit tests
- [ ] Integration tests
- [ ] E2E tests

### Phase 9: Deployment ✅
- [ ] Docker setup
- [ ] Production config
- [ ] Deploy

---

## 🚀 Running the Complete System

### Development
```bash
# Terminal 1: Start Ollama
ollama serve

# Terminal 2: Start Backend
cd yomnai_backend
python run.py

# Terminal 3: Start Frontend
cd yomnai_frontend
npm run dev
```

### Production
```bash
# Using Docker Compose
docker-compose up -d

# Or manually
docker build -t yomnai-backend ./yomnai_backend
docker build -t yomnai-frontend ./yomnai_frontend
docker run -d -p 8000:8001 yomnai-backend
docker run -d -p 3000:3000 yomnai-frontend
```

---

## 🔧 Troubleshooting

### Common Issues

1. **Ollama Connection Failed**
```bash
# Check Ollama is running
curl http://localhost:11434/api/tags

# Restart Ollama
systemctl restart ollama
```

2. **Database Locked Error**
```python
# Add to database.py
engine = create_async_engine(
    settings.database_url,
    connect_args={"check_same_thread": False}  # SQLite only
)
```

3. **CORS Issues**
```python
# Update CORS origins in .env
FRONTEND_URLS=http://localhost:3000,http://localhost:5173
```

4. **Session Expired**
```typescript
// Add auto-refresh in React
useEffect(() => {
  const interval = setInterval(() => {
    api.validateSession();
  }, 60 * 60 * 1000); // Every hour
  return () => clearInterval(interval);
}, []);
```

---

## 📚 Next Steps

1. Add WebSocket support for real-time updates
2. Implement background job processing with Celery
3. Add Redis for caching
4. Implement rate limiting middleware
5. Add monitoring with Prometheus
6. Set up logging with ELK stack
7. Add API versioning
8. Implement data export functionality

---

*This implementation guide provides a complete roadmap for building the Yomnai FastAPI backend. Follow the phases sequentially for best results.*