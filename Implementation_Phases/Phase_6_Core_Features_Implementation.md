# Phase 6: Core Features Implementation

## Overview
Connect all components to create the complete end-to-end workflow for financial analysis.

## Tasks

### 6.1 Complete Integration Module
Create `src/core/integration.py`:
```python
import logging
from typing import Dict, Any, List, Optional
from dataclasses import dataclass
import pandas as pd
from datetime import datetime

from src.parsers.csv_parser import BankStatementParser
from src.parsers.validator import TransactionValidator
from src.ai.vector_store import VectorStoreManager
from src.ai.ollama_client import OllamaClient
from src.ai.rag_pipeline import RAGPipeline
from src.models import Transaction

logger = logging.getLogger(__name__)

@dataclass
class ProcessingResult:
    """Result of file processing"""
    success: bool
    transactions: List[Transaction]
    validation_result: Dict[str, Any]
    error_message: Optional[str] = None
    processing_time: Optional[float] = None

class YomnaiCore:
    """Core integration class for Yomnai application"""
    
    def __init__(self):
        self.parser = BankStatementParser()
        self.validator = TransactionValidator()
        self.vector_store = None
        self.ollama_client = None
        self.rag_pipeline = None
        self.current_session = None
        
    def initialize_ai_components(self) -> bool:
        """Initialize AI components"""
        try:
            # Initialize Ollama client
            self.ollama_client = OllamaClient()
            
            # Initialize vector store
            self.vector_store = VectorStoreManager()
            
            # Initialize RAG pipeline
            self.rag_pipeline = RAGPipeline(
                self.vector_store,
                self.ollama_client
            )
            
            logger.info("AI components initialized successfully")
            return True
            
        except Exception as e:
            logger.error(f"Failed to initialize AI components: {str(e)}")
            return False
    
    def process_bank_statement(self, file_path: str) -> ProcessingResult:
        """Process a bank statement file"""
        start_time = datetime.now()
        
        try:
            # Parse CSV file
            logger.info(f"Parsing file: {file_path}")
            transactions = self.parser.parse_file(file_path)
            
            # Validate transactions
            logger.info("Validating transactions")
            validation_result = self.validator.validate_transactions(transactions)
            
            if not validation_result['valid']:
                return ProcessingResult(
                    success=False,
                    transactions=[],
                    validation_result=validation_result,
                    error_message="Validation failed"
                )
            
            # Create session ID
            self.current_session = datetime.now().strftime("%Y%m%d_%H%M%S")
            
            # Add to vector store if AI is initialized
            if self.vector_store:
                logger.info("Adding transactions to vector store")
                self.vector_store.add_transactions(
                    transactions,
                    self.current_session
                )
            
            processing_time = (datetime.now() - start_time).total_seconds()
            
            return ProcessingResult(
                success=True,
                transactions=transactions,
                validation_result=validation_result,
                processing_time=processing_time
            )
            
        except Exception as e:
            logger.error(f"Error processing bank statement: {str(e)}")
            return ProcessingResult(
                success=False,
                transactions=[],
                validation_result={"valid": False, "errors": [str(e)]},
                error_message=str(e)
            )
    
    def query(self, question: str, **kwargs) -> Dict[str, Any]:
        """Query the processed data"""
        if not self.rag_pipeline:
            return {
                "answer": "AI components not initialized. Please restart the application.",
                "error": "No RAG pipeline"
            }
        
        if not self.current_session:
            return {
                "answer": "No data loaded. Please upload a bank statement first.",
                "error": "No active session"
            }
        
        return self.rag_pipeline.process_query(question, **kwargs)
    
    def get_transaction_summary(self) -> Dict[str, Any]:
        """Get summary of loaded transactions"""
        if not self.vector_store:
            return {}
        
        stats = self.vector_store.get_statistics()
        
        # Get transaction breakdown
        if self.rag_pipeline:
            summary = self.rag_pipeline.generate_summary()
        else:
            summary = "AI not available for summary generation"
        
        return {
            "statistics": stats,
            "summary": summary
        }
    
    def export_analysis(self, output_path: str, format: str = "json") -> bool:
        """Export analysis results"""
        try:
            if format == "json":
                # Export as JSON
                import json
                export_data = {
                    "session_id": self.current_session,
                    "timestamp": datetime.now().isoformat(),
                    "statistics": self.vector_store.get_statistics() if self.vector_store else {},
                    "transactions_count": len(self.current_transactions) if hasattr(self, 'current_transactions') else 0
                }
                
                with open(output_path, 'w') as f:
                    json.dump(export_data, f, indent=2)
            
            elif format == "csv":
                # Export as CSV
                if hasattr(self, 'current_transactions'):
                    df = pd.DataFrame([t.to_dict() for t in self.current_transactions])
                    df.to_csv(output_path, index=False)
            
            logger.info(f"Exported analysis to {output_path}")
            return True
            
        except Exception as e:
            logger.error(f"Export failed: {str(e)}")
            return False
    
    def clear_session(self):
        """Clear current session data"""
        if self.vector_store and self.current_session:
            self.vector_store.clear_collection(self.current_session)
        
        self.current_session = None
        if hasattr(self, 'current_transactions'):
            delattr(self, 'current_transactions')
```

### 6.2 Error Handling and Recovery
Create `src/core/error_handler.py`:
```python
import logging
import traceback
from typing import Callable, Any
from functools import wraps

logger = logging.getLogger(__name__)

class YomnaiError(Exception):
    """Base exception for Yomnai application"""
    pass

class FileProcessingError(YomnaiError):
    """Error during file processing"""
    pass

class AIServiceError(YomnaiError):
    """Error with AI service"""
    pass

class ValidationError(YomnaiError):
    """Data validation error"""
    pass

def handle_errors(default_return=None, log_errors=True):
    """Decorator for error handling"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            try:
                return func(*args, **kwargs)
            except Exception as e:
                if log_errors:
                    logger.error(
                        f"Error in {func.__name__}: {str(e)}\n"
                        f"Traceback: {traceback.format_exc()}"
                    )
                
                # Return default value or re-raise
                if default_return is not None:
                    return default_return
                raise
        
        return wrapper
    return decorator

class ErrorRecovery:
    """Handle error recovery strategies"""
    
    @staticmethod
    def recover_from_ollama_error():
        """Attempt to recover from Ollama connection error"""
        recovery_steps = [
            "1. Check if Ollama is running: 'ollama serve'",
            "2. Verify the model is installed: 'ollama list'",
            "3. Pull the model if needed: 'ollama pull llama3'",
            "4. Check firewall settings for port 11434",
            "5. Restart the Ollama service"
        ]
        return recovery_steps
    
    @staticmethod
    def recover_from_csv_error(error_msg: str):
        """Provide guidance for CSV parsing errors"""
        if "encoding" in error_msg.lower():
            return [
                "File encoding issue detected:",
                "- Try saving the CSV file as UTF-8",
                "- Open in Excel and re-save as CSV",
                "- Check for special characters"
            ]
        elif "header" in error_msg.lower():
            return [
                "Header detection issue:",
                "- Ensure the CSV has column headers",
                "- Remove any extra rows before headers",
                "- Check that headers include Date, Description, Amount"
            ]
        else:
            return [
                "General CSV parsing error:",
                "- Verify the file is a valid CSV format",
                "- Check for corrupted data",
                "- Ensure consistent column structure"
            ]
    
    @staticmethod
    def recover_from_memory_error():
        """Handle out of memory errors"""
        return [
            "Memory limit exceeded:",
            "- Try processing a smaller file",
            "- Close other applications",
            "- Restart the application",
            "- Consider upgrading system RAM"
        ]
```

### 6.3 Performance Optimization
Create `src/core/performance.py`:
```python
import time
import psutil
import logging
from functools import lru_cache, wraps
from typing import Callable, Any
import asyncio

logger = logging.getLogger(__name__)

def measure_performance(func: Callable) -> Callable:
    """Decorator to measure function performance"""
    @wraps(func)
    def wrapper(*args, **kwargs) -> Any:
        start_time = time.time()
        start_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB
        
        result = func(*args, **kwargs)
        
        end_time = time.time()
        end_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB
        
        execution_time = end_time - start_time
        memory_used = end_memory - start_memory
        
        logger.debug(
            f"{func.__name__} - Time: {execution_time:.2f}s, "
            f"Memory: {memory_used:.2f}MB"
        )
        
        return result
    
    return wrapper

class PerformanceOptimizer:
    """Optimize application performance"""
    
    @staticmethod
    @lru_cache(maxsize=128)
    def cached_embedding_generation(text: str) -> list:
        """Cache embedding generation for repeated queries"""
        # This would be replaced with actual embedding generation
        return [0.0] * 768  # Placeholder
    
    @staticmethod
    def batch_process_transactions(transactions: list, batch_size: int = 100):
        """Process transactions in batches for memory efficiency"""
        for i in range(0, len(transactions), batch_size):
            batch = transactions[i:i + batch_size]
            yield batch
    
    @staticmethod
    async def async_process_queries(queries: list) -> list:
        """Process multiple queries asynchronously"""
        async def process_single_query(query):
            # Simulate async processing
            await asyncio.sleep(0.1)
            return f"Processed: {query}"
        
        tasks = [process_single_query(q) for q in queries]
        results = await asyncio.gather(*tasks)
        return results
    
    @staticmethod
    def optimize_dataframe(df):
        """Optimize pandas DataFrame memory usage"""
        import pandas as pd
        
        for col in df.columns:
            col_type = df[col].dtype
            
            if col_type != 'object':
                if col_type == 'float64':
                    df[col] = pd.to_numeric(df[col], downcast='float')
                elif col_type == 'int64':
                    df[col] = pd.to_numeric(df[col], downcast='integer')
        
        return df
    
    @staticmethod
    def get_system_resources() -> dict:
        """Get current system resource usage"""
        return {
            "cpu_percent": psutil.cpu_percent(interval=1),
            "memory_percent": psutil.virtual_memory().percent,
            "memory_available_mb": psutil.virtual_memory().available / 1024 / 1024,
            "disk_usage_percent": psutil.disk_usage('/').percent
        }
```

### 6.4 Security Module
Create `src/core/security.py`:
```python
import hashlib
import secrets
from pathlib import Path
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class SecurityManager:
    """Manage security aspects of the application"""
    
    @staticmethod
    def sanitize_input(text: str) -> str:
        """Sanitize user input to prevent injection attacks"""
        # Remove potential SQL injection patterns
        dangerous_patterns = [
            "DROP", "DELETE", "INSERT", "UPDATE", "SELECT",
            "--", "/*", "*/", "xp_", "sp_", "exec", "execute"
        ]
        
        sanitized = text
        for pattern in dangerous_patterns:
            sanitized = sanitized.replace(pattern, "")
        
        # Limit length
        max_length = 1000
        if len(sanitized) > max_length:
            sanitized = sanitized[:max_length]
        
        return sanitized.strip()
    
    @staticmethod
    def validate_file_type(file_path: str) -> bool:
        """Validate that file is safe to process"""
        allowed_extensions = ['.csv', '.txt']
        file_ext = Path(file_path).suffix.lower()
        
        if file_ext not in allowed_extensions:
            logger.warning(f"Rejected file with extension: {file_ext}")
            return False
        
        # Check file signature (magic bytes) for CSV
        try:
            with open(file_path, 'rb') as f:
                header = f.read(512)
                
            # Basic check for text-based file
            try:
                header.decode('utf-8')
                return True
            except UnicodeDecodeError:
                # Try other common encodings
                for encoding in ['latin-1', 'cp1252']:
                    try:
                        header.decode(encoding)
                        return True
                    except UnicodeDecodeError:
                        continue
                
                return False
                
        except Exception as e:
            logger.error(f"Error validating file: {str(e)}")
            return False
    
    @staticmethod
    def hash_sensitive_data(data: str) -> str:
        """Hash sensitive data for logging"""
        return hashlib.sha256(data.encode()).hexdigest()[:8]
    
    @staticmethod
    def generate_session_token() -> str:
        """Generate secure session token"""
        return secrets.token_urlsafe(32)
    
    @staticmethod
    def mask_account_numbers(text: str) -> str:
        """Mask potential account numbers in text"""
        import re
        
        # Pattern for potential account numbers (8-16 digits)
        pattern = r'\b\d{8,16}\b'
        
        def mask_number(match):
            number = match.group()
            if len(number) <= 10:
                return '*' * len(number)
            else:
                # Show first 2 and last 2 digits
                return number[:2] + '*' * (len(number) - 4) + number[-2:]
        
        return re.sub(pattern, mask_number, text)
    
    @staticmethod
    def check_data_privacy(transactions: list) -> dict:
        """Check for privacy concerns in transaction data"""
        privacy_report = {
            "contains_personal_info": False,
            "sensitive_merchants": [],
            "recommendations": []
        }
        
        sensitive_keywords = [
            'medical', 'hospital', 'clinic', 'pharmacy',
            'legal', 'attorney', 'court',
            'therapy', 'counseling'
        ]
        
        for transaction in transactions:
            desc_lower = transaction.description.lower()
            for keyword in sensitive_keywords:
                if keyword in desc_lower:
                    privacy_report["contains_personal_info"] = True
                    privacy_report["sensitive_merchants"].append(
                        SecurityManager.mask_account_numbers(transaction.description)
                    )
                    break
        
        if privacy_report["contains_personal_info"]:
            privacy_report["recommendations"].append(
                "Sensitive transaction data detected. Ensure data remains local."
            )
        
        return privacy_report
```

### 6.5 Configuration Management
Create `src/core/config_manager.py`:
```python
import json
import yaml
from pathlib import Path
from typing import Dict, Any, Optional
import logging

logger = logging.getLogger(__name__)

class ConfigManager:
    """Manage application configuration"""
    
    def __init__(self, config_path: str = "./config/app_config.yaml"):
        self.config_path = Path(config_path)
        self.config = self._load_config()
        self.user_preferences = self._load_user_preferences()
    
    def _load_config(self) -> Dict[str, Any]:
        """Load application configuration"""
        if not self.config_path.exists():
            return self._get_default_config()
        
        try:
            with open(self.config_path, 'r') as f:
                if self.config_path.suffix == '.yaml':
                    return yaml.safe_load(f)
                elif self.config_path.suffix == '.json':
                    return json.load(f)
        except Exception as e:
            logger.error(f"Error loading config: {str(e)}")
            return self._get_default_config()
    
    def _get_default_config(self) -> Dict[str, Any]:
        """Get default configuration"""
        return {
            "app": {
                "name": "Yomnai Financial Analyst",
                "version": "1.0.0",
                "debug": False
            },
            "processing": {
                "max_file_size_mb": 50,
                "chunk_size": 1000,
                "batch_size": 100
            },
            "ai": {
                "model": "llama3",
                "temperature": 0.1,
                "max_tokens": 500,
                "n_results": 10
            },
            "ui": {
                "theme": "light",
                "show_advanced_options": False,
                "auto_save_session": True
            }
        }
    
    def _load_user_preferences(self) -> Dict[str, Any]:
        """Load user preferences"""
        pref_path = Path("./config/user_preferences.json")
        
        if not pref_path.exists():
            return {}
        
        try:
            with open(pref_path, 'r') as f:
                return json.load(f)
        except Exception as e:
            logger.error(f"Error loading preferences: {str(e)}")
            return {}
    
    def save_config(self):
        """Save current configuration"""
        try:
            self.config_path.parent.mkdir(parents=True, exist_ok=True)
            
            with open(self.config_path, 'w') as f:
                if self.config_path.suffix == '.yaml':
                    yaml.dump(self.config, f, default_flow_style=False)
                elif self.config_path.suffix == '.json':
                    json.dump(self.config, f, indent=2)
            
            logger.info("Configuration saved successfully")
            
        except Exception as e:
            logger.error(f"Error saving config: {str(e)}")
    
    def get(self, key_path: str, default: Any = None) -> Any:
        """Get configuration value by dot-separated path"""
        keys = key_path.split('.')
        value = self.config
        
        for key in keys:
            if isinstance(value, dict) and key in value:
                value = value[key]
            else:
                return default
        
        return value
    
    def set(self, key_path: str, value: Any):
        """Set configuration value by dot-separated path"""
        keys = key_path.split('.')
        config = self.config
        
        for key in keys[:-1]:
            if key not in config:
                config[key] = {}
            config = config[key]
        
        config[keys[-1]] = value
        self.save_config()
    
    def update_user_preference(self, key: str, value: Any):
        """Update user preference"""
        self.user_preferences[key] = value
        
        pref_path = Path("./config/user_preferences.json")
        pref_path.parent.mkdir(parents=True, exist_ok=True)
        
        try:
            with open(pref_path, 'w') as f:
                json.dump(self.user_preferences, f, indent=2)
        except Exception as e:
            logger.error(f"Error saving preferences: {str(e)}")
```

### 6.6 Main Execution Script
Create `main.py`:
```python
#!/usr/bin/env python3
"""
Yomnai - Private Financial Analyst
Main execution script
"""

import sys
import logging
from pathlib import Path

# Add src to path
sys.path.insert(0, str(Path(__file__).parent))

from config.settings import settings
from config.logging_config import setup_logging
from src.core.integration import YomnaiCore
from src.core.config_manager import ConfigManager
from src.core.security import SecurityManager

# Initialize logging
logger = setup_logging(settings.DEBUG_MODE)

def main():
    """Main entry point"""
    logger.info("Starting Yomnai Financial Analyst")
    
    try:
        # Load configuration
        config_manager = ConfigManager()
        
        # Initialize core
        core = YomnaiCore()
        
        # Initialize AI components
        if not core.initialize_ai_components():
            logger.error("Failed to initialize AI components")
            logger.info("Running in limited mode (no AI features)")
        
        # Launch Streamlit app
        import streamlit.web.cli as stcli
        import sys
        
        sys.argv = [
            "streamlit",
            "run",
            "app.py",
            "--server.port=8501",
            "--server.address=localhost",
            "--server.headless=false",
            "--browser.gatherUsageStats=false"
        ]
        
        sys.exit(stcli.main())
        
    except KeyboardInterrupt:
        logger.info("Application terminated by user")
        sys.exit(0)
    except Exception as e:
        logger.error(f"Fatal error: {str(e)}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## Validation Checklist
- [ ] All components integrate seamlessly
- [ ] Error handling works for all failure modes
- [ ] Performance optimization reduces latency
- [ ] Security measures protect user data
- [ ] Configuration management allows customization
- [ ] Session management maintains state

## Success Criteria
- Complete end-to-end workflow functions correctly
- Errors are handled gracefully with recovery options
- Performance meets <3 second response time goal
- Security measures prevent data leakage
- Configuration persists between sessions

## Next Phase
Phase 7: Testing & Validation - Comprehensive testing of all components