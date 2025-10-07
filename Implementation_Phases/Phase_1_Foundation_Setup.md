# Phase 1: Foundation Setup

## Overview
Set up the core project structure and install all required dependencies for the Yomnai financial analyst application.

## Tasks

### 1.1 Project Structure Creation
```bash
# Create project directories
mkdir -p src
mkdir -p src/parsers
mkdir -p src/ai
mkdir -p src/ui
mkdir -p src/utils
mkdir -p data/temp
mkdir -p tests
mkdir -p config
```

### 1.2 Requirements File
Create `requirements.txt`:
```txt
streamlit==1.32.0
pandas==2.2.0
chromadb==0.4.22
langchain==0.1.9
langchain-community==0.0.24
ollama==0.1.7
python-dotenv==1.0.0
numpy==1.26.3
openpyxl==3.1.2
```

### 1.3 Environment Configuration
Create `.env`:
```env
# Ollama Configuration
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
```

### 1.4 Configuration Module
Create `config/settings.py`:
```python
import os
from pathlib import Path
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

class Settings:
    # Paths
    BASE_DIR = Path(__file__).parent.parent
    DATA_DIR = BASE_DIR / "data"
    TEMP_DIR = DATA_DIR / "temp"
    
    # Ollama Settings
    OLLAMA_HOST = os.getenv("OLLAMA_HOST", "localhost")
    OLLAMA_PORT = int(os.getenv("OLLAMA_PORT", 11434))
    OLLAMA_MODEL = os.getenv("OLLAMA_MODEL", "llama3")
    OLLAMA_URL = f"http://{OLLAMA_HOST}:{OLLAMA_PORT}"
    
    # ChromaDB Settings
    CHROMA_PERSIST_DIR = os.getenv("CHROMA_PERSIST_DIRECTORY", "./data/chroma_db")
    CHROMA_COLLECTION = os.getenv("CHROMA_COLLECTION_NAME", "bank_transactions")
    
    # Application Settings
    MAX_FILE_SIZE_MB = int(os.getenv("MAX_FILE_SIZE_MB", 50))
    CHUNK_SIZE = int(os.getenv("CHUNK_SIZE", 1000))
    DEBUG_MODE = os.getenv("DEBUG_MODE", "False").lower() == "true"
    
    # Ensure directories exist
    TEMP_DIR.mkdir(parents=True, exist_ok=True)
    Path(CHROMA_PERSIST_DIR).mkdir(parents=True, exist_ok=True)

settings = Settings()
```

### 1.5 Logging Configuration
Create `config/logging_config.py`:
```python
import logging
import sys
from pathlib import Path

def setup_logging(debug=False):
    """Configure application logging"""
    log_level = logging.DEBUG if debug else logging.INFO
    
    # Create logs directory
    log_dir = Path("logs")
    log_dir.mkdir(exist_ok=True)
    
    # Configure formatter
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    
    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    console_handler.setLevel(log_level)
    
    # File handler
    file_handler = logging.FileHandler(log_dir / "argus.log")
    file_handler.setFormatter(formatter)
    file_handler.setLevel(log_level)
    
    # Configure root logger
    root_logger = logging.getLogger()
    root_logger.setLevel(log_level)
    root_logger.addHandler(console_handler)
    root_logger.addHandler(file_handler)
    
    return root_logger
```

### 1.6 Main Application Entry Point
Create `app.py`:
```python
import streamlit as st
from config.settings import settings
from config.logging_config import setup_logging

# Initialize logging
logger = setup_logging(settings.DEBUG_MODE)

def main():
    st.set_page_config(
        page_title="Yomnai - Private Financial Analyst",
        page_icon="👁️",
        layout="wide",
        initial_sidebar_state="expanded"
    )
    
    st.title("👁️ Yomnai - Your Private Financial Analyst")
    st.markdown("---")
    
    # Placeholder for main application
    st.info("🚧 Application under development. Phase 1: Foundation Setup Complete.")
    
    # Debug info
    if settings.DEBUG_MODE:
        with st.expander("Debug Information"):
            st.json({
                "Ollama URL": settings.OLLAMA_URL,
                "Model": settings.OLLAMA_MODEL,
                "ChromaDB Directory": settings.CHROMA_PERSIST_DIR,
                "Max File Size": f"{settings.MAX_FILE_SIZE_MB} MB"
            })

if __name__ == "__main__":
    main()
```

### 1.7 Utility Functions
Create `src/utils/__init__.py`:
```python
"""Utility functions for Yomnai application"""

def format_currency(amount: float) -> str:
    """Format amount as currency"""
    return f"${amount:,.2f}"

def sanitize_filename(filename: str) -> str:
    """Sanitize filename for safe storage"""
    import re
    return re.sub(r'[^a-zA-Z0-9._-]', '_', filename)

def calculate_file_hash(file_path: str) -> str:
    """Calculate SHA256 hash of file"""
    import hashlib
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()
```

## Implementation Status: ✅ COMPLETED

### Completed Tasks
- ✅ Project structure and directories created
- ✅ requirements.txt created with all dependencies
- ✅ .env file configured with application settings
- ✅ Config modules (settings.py, logging_config.py) implemented
- ✅ Main app.py created with basic Streamlit interface
- ✅ Utility functions module created
- ✅ All directories created with proper structure

### Files Created
- `/requirements.txt` - Python dependencies
- `/.env` - Environment configuration
- `/config/settings.py` - Application settings module
- `/config/logging_config.py` - Logging configuration
- `/app.py` - Main Streamlit application
- `/src/utils/__init__.py` - Utility functions

## Validation Checklist
- [x] Directory structure created
- [x] Environment variables configured
- [x] Requirements file ready for installation
- [x] Basic application structure in place
- [x] Virtual environment activated
- [x] Dependencies installed via `pip install -r requirements.txt`
- [x] Basic Streamlit app runs with `streamlit run app.py` ✅ TESTED AND WORKING

## Success Criteria
- Application structure ready for development
- Configuration system in place
- Logging system configured
- All directories created with proper permissions

## Next Phase
Phase 2: Data Processing Pipeline - Building the CSV parser and transaction extraction system