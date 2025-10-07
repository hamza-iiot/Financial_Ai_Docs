# Phase 7: Testing & Validation

## Overview
Comprehensive testing suite for all components of the Yomnai application.

## Tasks

### 7.1 Unit Test Suite
Create `tests/test_suite.py`:
```python
import unittest
import sys
from pathlib import Path

# Add src to path
sys.path.insert(0, str(Path(__file__).parent.parent))

# Import all test modules
from tests.test_csv_parser import TestCSVParser
from tests.test_vector_store import TestVectorStore
from tests.test_rag_pipeline import TestRAGPipeline
from tests.test_integration import TestIntegration
from tests.test_security import TestSecurity
from tests.test_performance import TestPerformance

def create_test_suite():
    """Create comprehensive test suite"""
    loader = unittest.TestLoader()
    suite = unittest.TestSuite()
    
    # Add all test cases
    suite.addTests(loader.loadTestsFromTestCase(TestCSVParser))
    suite.addTests(loader.loadTestsFromTestCase(TestVectorStore))
    suite.addTests(loader.loadTestsFromTestCase(TestRAGPipeline))
    suite.addTests(loader.loadTestsFromTestCase(TestIntegration))
    suite.addTests(loader.loadTestsFromTestCase(TestSecurity))
    suite.addTests(loader.loadTestsFromTestCase(TestPerformance))
    
    return suite

if __name__ == '__main__':
    runner = unittest.TextTestRunner(verbosity=2)
    runner.run(create_test_suite())
```

### 7.2 CSV Parser Tests
Create `tests/test_csv_parser.py`:
```python
import unittest
import tempfile
import csv
from datetime import datetime
from pathlib import Path

from src.parsers.csv_parser import BankStatementParser
from src.models import Transaction, TransactionType

class TestCSVParser(unittest.TestCase):
    
    def setUp(self):
        """Set up test fixtures"""
        self.parser = BankStatementParser()
        self.temp_dir = tempfile.mkdtemp()
    
    def create_test_csv(self, filename: str, data: list) -> str:
        """Create a test CSV file"""
        file_path = Path(self.temp_dir) / filename
        
        with open(file_path, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=data[0].keys())
            writer.writeheader()
            writer.writerows(data)
        
        return str(file_path)
    
    def test_parse_standard_csv(self):
        """Test parsing standard CSV format"""
        test_data = [
            {
                'Date': '2024-01-15',
                'Description': 'GROCERY STORE',
                'Debit': '150.00',
                'Credit': '',
                'Balance': '4850.00'
            },
            {
                'Date': '2024-01-20',
                'Description': 'SALARY DEPOSIT',
                'Debit': '',
                'Credit': '3000.00',
                'Balance': '7850.00'
            }
        ]
        
        csv_path = self.create_test_csv('standard.csv', test_data)
        transactions = self.parser.parse_file(csv_path)
        
        self.assertEqual(len(transactions), 2)
        self.assertEqual(transactions[0].amount, 150.00)
        self.assertEqual(transactions[0].type, TransactionType.DEBIT)
        self.assertEqual(transactions[1].amount, 3000.00)
        self.assertEqual(transactions[1].type, TransactionType.CREDIT)
    
    def test_parse_with_header_rows(self):
        """Test parsing CSV with extra header rows"""
        csv_content = """
Bank Statement
Account: 123456789
Period: January 2024

Date,Description,Amount,Balance
2024-01-15,PURCHASE,-50.00,950.00
2024-01-16,DEPOSIT,100.00,1050.00
"""
        
        file_path = Path(self.temp_dir) / 'with_headers.csv'
        file_path.write_text(csv_content)
        
        transactions = self.parser.parse_file(str(file_path))
        
        self.assertEqual(len(transactions), 2)
        self.assertEqual(transactions[0].amount, 50.00)
    
    def test_parse_different_date_formats(self):
        """Test parsing different date formats"""
        date_formats = [
            ('2024-01-15', '%Y-%m-%d'),
            ('15/01/2024', '%d/%m/%Y'),
            ('01/15/2024', '%m/%d/%Y'),
            ('15 Jan 2024', '%d %b %Y')
        ]
        
        for date_str, _ in date_formats:
            test_data = [{
                'Date': date_str,
                'Description': 'TEST',
                'Amount': '-100.00'
            }]
            
            csv_path = self.create_test_csv(f'date_{date_str.replace("/", "_")}.csv', test_data)
            transactions = self.parser.parse_file(csv_path)
            
            self.assertEqual(len(transactions), 1)
            self.assertIsInstance(transactions[0].date, datetime)
    
    def test_handle_malformed_csv(self):
        """Test handling of malformed CSV"""
        csv_content = """
Date,Description,Amount
2024-01-15,TEST,NotANumber
2024-01-16,TEST2,
,TEST3,100
"""
        
        file_path = Path(self.temp_dir) / 'malformed.csv'
        file_path.write_text(csv_content)
        
        transactions = self.parser.parse_file(str(file_path))
        
        # Should parse valid transactions only
        self.assertGreater(len(transactions), 0)
    
    def test_currency_symbol_handling(self):
        """Test handling of currency symbols"""
        test_data = [
            {'Date': '2024-01-15', 'Description': 'TEST1', 'Amount': '$100.00'},
            {'Date': '2024-01-16', 'Description': 'TEST2', 'Amount': '£200.00'},
            {'Date': '2024-01-17', 'Description': 'TEST3', 'Amount': '€300.00'},
            {'Date': '2024-01-18', 'Description': 'TEST4', 'Amount': '1,234.56'}
        ]
        
        csv_path = self.create_test_csv('currency.csv', test_data)
        transactions = self.parser.parse_file(csv_path)
        
        self.assertEqual(len(transactions), 4)
        self.assertEqual(transactions[0].amount, 100.00)
        self.assertEqual(transactions[3].amount, 1234.56)
```

### 7.3 Integration Tests
Create `tests/test_integration.py`:
```python
import unittest
import tempfile
from pathlib import Path

from src.core.integration import YomnaiCore, ProcessingResult

class TestIntegration(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.core = YomnaiCore()
        self.temp_dir = tempfile.mkdtemp()
    
    def create_test_file(self) -> str:
        """Create a test CSV file"""
        csv_content = """Date,Description,Debit,Credit,Balance
2024-01-15,GROCERY STORE,150.00,,4850.00
2024-01-20,SALARY DEPOSIT,,3000.00,7850.00
2024-01-25,UTILITY BILL,125.00,,7725.00"""
        
        file_path = Path(self.temp_dir) / 'test_statement.csv'
        file_path.write_text(csv_content)
        return str(file_path)
    
    def test_process_bank_statement(self):
        """Test complete bank statement processing"""
        file_path = self.create_test_file()
        result = self.core.process_bank_statement(file_path)
        
        self.assertIsInstance(result, ProcessingResult)
        self.assertTrue(result.success)
        self.assertEqual(len(result.transactions), 3)
        self.assertIsNotNone(result.processing_time)
    
    def test_initialize_ai_components(self):
        """Test AI component initialization"""
        # This test will only pass if Ollama is running
        try:
            success = self.core.initialize_ai_components()
            if success:
                self.assertIsNotNone(self.core.ollama_client)
                self.assertIsNotNone(self.core.vector_store)
                self.assertIsNotNone(self.core.rag_pipeline)
        except ConnectionError:
            self.skipTest("Ollama not available")
    
    def test_query_without_data(self):
        """Test querying without loaded data"""
        response = self.core.query("What are my expenses?")
        
        self.assertIn("error", response)
        self.assertIn("No", response["answer"])
    
    def test_complete_workflow(self):
        """Test complete workflow from upload to query"""
        # Skip if Ollama not available
        try:
            if not self.core.initialize_ai_components():
                self.skipTest("AI components not available")
        except:
            self.skipTest("AI components not available")
        
        # Process file
        file_path = self.create_test_file()
        result = self.core.process_bank_statement(file_path)
        
        self.assertTrue(result.success)
        
        # Query data
        response = self.core.query("How many transactions do I have?")
        
        self.assertIn("answer", response)
        self.assertNotIn("error", response)
    
    def test_session_management(self):
        """Test session creation and clearing"""
        file_path = self.create_test_file()
        result = self.core.process_bank_statement(file_path)
        
        self.assertTrue(result.success)
        self.assertIsNotNone(self.core.current_session)
        
        # Clear session
        self.core.clear_session()
        self.assertIsNone(self.core.current_session)
```

### 7.4 Security Tests
Create `tests/test_security.py`:
```python
import unittest
import tempfile
from pathlib import Path

from src.core.security import SecurityManager

class TestSecurity(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.security = SecurityManager()
    
    def test_sanitize_input(self):
        """Test input sanitization"""
        # Test SQL injection attempt
        malicious = "'; DROP TABLE transactions; --"
        sanitized = self.security.sanitize_input(malicious)
        
        self.assertNotIn("DROP", sanitized)
        self.assertNotIn("--", sanitized)
        
        # Test normal input
        normal = "What are my expenses for January?"
        sanitized = self.security.sanitize_input(normal)
        self.assertEqual(sanitized, normal)
    
    def test_validate_file_type(self):
        """Test file type validation"""
        # Create test files
        temp_dir = tempfile.mkdtemp()
        
        # Valid CSV
        csv_file = Path(temp_dir) / "test.csv"
        csv_file.write_text("Date,Amount\n2024-01-15,100")
        self.assertTrue(self.security.validate_file_type(str(csv_file)))
        
        # Invalid executable
        exe_file = Path(temp_dir) / "test.exe"
        exe_file.write_bytes(b"MZ\x90\x00")  # PE header
        self.assertFalse(self.security.validate_file_type(str(exe_file)))
    
    def test_mask_account_numbers(self):
        """Test account number masking"""
        text = "Transfer from account 1234567890123456 to 9876543210"
        masked = self.security.mask_account_numbers(text)
        
        self.assertIn("12**************56", masked)
        self.assertIn("**********", masked)
        self.assertNotIn("1234567890123456", masked)
    
    def test_generate_session_token(self):
        """Test session token generation"""
        token1 = self.security.generate_session_token()
        token2 = self.security.generate_session_token()
        
        self.assertIsInstance(token1, str)
        self.assertGreater(len(token1), 20)
        self.assertNotEqual(token1, token2)  # Should be unique
    
    def test_hash_sensitive_data(self):
        """Test sensitive data hashing"""
        sensitive = "account_number_12345"
        hashed = self.security.hash_sensitive_data(sensitive)
        
        self.assertIsInstance(hashed, str)
        self.assertEqual(len(hashed), 8)
        self.assertNotEqual(hashed, sensitive)
        
        # Same input should produce same hash
        hashed2 = self.security.hash_sensitive_data(sensitive)
        self.assertEqual(hashed, hashed2)
```

### 7.5 Performance Tests
Create `tests/test_performance.py`:
```python
import unittest
import time
import tempfile
from pathlib import Path

from src.core.performance import PerformanceOptimizer, measure_performance

class TestPerformance(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.optimizer = PerformanceOptimizer()
    
    def test_batch_processing(self):
        """Test batch processing of transactions"""
        # Create large list of mock transactions
        transactions = list(range(1000))
        
        batches = list(self.optimizer.batch_process_transactions(
            transactions, batch_size=100
        ))
        
        self.assertEqual(len(batches), 10)
        self.assertEqual(len(batches[0]), 100)
        self.assertEqual(sum(len(b) for b in batches), 1000)
    
    def test_performance_measurement(self):
        """Test performance measurement decorator"""
        @measure_performance
        def slow_function():
            time.sleep(0.1)
            return "done"
        
        result = slow_function()
        self.assertEqual(result, "done")
    
    def test_system_resources(self):
        """Test system resource monitoring"""
        resources = self.optimizer.get_system_resources()
        
        self.assertIn("cpu_percent", resources)
        self.assertIn("memory_percent", resources)
        self.assertIn("memory_available_mb", resources)
        
        self.assertGreaterEqual(resources["cpu_percent"], 0)
        self.assertLessEqual(resources["cpu_percent"], 100)
    
    def test_dataframe_optimization(self):
        """Test DataFrame memory optimization"""
        import pandas as pd
        
        # Create test DataFrame
        df = pd.DataFrame({
            'int_col': range(1000),
            'float_col': [1.0] * 1000,
            'str_col': ['test'] * 1000
        })
        
        # Get original memory usage
        original_memory = df.memory_usage(deep=True).sum()
        
        # Optimize
        optimized_df = self.optimizer.optimize_dataframe(df)
        optimized_memory = optimized_df.memory_usage(deep=True).sum()
        
        # Should use less or equal memory
        self.assertLessEqual(optimized_memory, original_memory)
    
    def test_caching(self):
        """Test caching mechanism"""
        # First call - not cached
        text = "test embedding text"
        embedding1 = self.optimizer.cached_embedding_generation(text)
        
        # Second call - should be cached
        embedding2 = self.optimizer.cached_embedding_generation(text)
        
        self.assertEqual(embedding1, embedding2)
```

### 7.6 End-to-End Test Script
Create `tests/test_e2e.py`:
```python
#!/usr/bin/env python3
"""
End-to-end test script for Yomnai application
"""

import sys
import time
import tempfile
from pathlib import Path
import subprocess

def create_test_csv():
    """Create a test CSV file"""
    csv_content = """Date,Description,Debit,Credit,Balance
2024-01-01,Opening Balance,,,5000.00
2024-01-05,GROCERY STORE ABC,125.50,,4874.50
2024-01-10,EMPLOYER SALARY DEPOSIT,,3500.00,8374.50
2024-01-15,ELECTRIC UTILITY COMPANY,150.00,,8224.50
2024-01-20,RESTAURANT XYZ,75.25,,8149.25
2024-01-25,ATM WITHDRAWAL,200.00,,7949.25
2024-01-28,ONLINE TRANSFER FROM SAVINGS,,500.00,8449.25
2024-01-30,SUBSCRIPTION SERVICE,14.99,,8434.26"""
    
    temp_file = tempfile.NamedTemporaryFile(
        mode='w', suffix='.csv', delete=False
    )
    temp_file.write(csv_content)
    temp_file.close()
    
    return temp_file.name

def test_csv_parsing():
    """Test CSV parsing functionality"""
    print("Testing CSV parsing...")
    
    from src.parsers.csv_parser import BankStatementParser
    
    csv_file = create_test_csv()
    parser = BankStatementParser()
    
    try:
        transactions = parser.parse_file(csv_file)
        assert len(transactions) == 7, f"Expected 7 transactions, got {len(transactions)}"
        print(f"✓ Successfully parsed {len(transactions)} transactions")
        return True
    except Exception as e:
        print(f"✗ CSV parsing failed: {str(e)}")
        return False
    finally:
        Path(csv_file).unlink()

def test_vector_store():
    """Test vector store functionality"""
    print("Testing vector store...")
    
    from src.ai.vector_store import VectorStoreManager
    from src.models import Transaction, TransactionType
    from datetime import datetime
    
    try:
        vector_store = VectorStoreManager(
            persist_directory="./test_vectors",
            collection_name="test_e2e"
        )
        
        # Create test transaction
        test_transaction = Transaction(
            date=datetime(2024, 1, 15),
            description="TEST TRANSACTION",
            amount=100.00,
            type=TransactionType.DEBIT
        )
        
        # Add to vector store
        vector_store.add_transactions([test_transaction], "test_session")
        
        # Search
        results = vector_store.search("test", n_results=5)
        
        assert len(results['transactions']) > 0, "No search results found"
        print("✓ Vector store operations successful")
        
        # Cleanup
        vector_store.clear_collection()
        return True
        
    except Exception as e:
        print(f"✗ Vector store test failed: {str(e)}")
        return False

def test_ollama_connection():
    """Test Ollama connection"""
    print("Testing Ollama connection...")
    
    try:
        from src.ai.ollama_client import OllamaClient
        
        client = OllamaClient()
        response = client.generate(
            "Say 'test successful' and nothing else",
            temperature=0.1
        )
        
        if "test successful" in response.lower():
            print("✓ Ollama connection successful")
            return True
        else:
            print(f"✗ Unexpected Ollama response: {response}")
            return False
            
    except ConnectionError:
        print("✗ Ollama not running. Start with: ollama serve")
        return False
    except Exception as e:
        print(f"✗ Ollama test failed: {str(e)}")
        return False

def test_streamlit_launch():
    """Test Streamlit app launch"""
    print("Testing Streamlit launch...")
    
    try:
        # Try to import streamlit
        import streamlit as st
        print("✓ Streamlit installed")
        
        # Check if app.py exists
        if not Path("app.py").exists():
            print("✗ app.py not found")
            return False
        
        print("✓ App file exists")
        return True
        
    except ImportError:
        print("✗ Streamlit not installed")
        return False

def run_all_tests():
    """Run all end-to-end tests"""
    print("=" * 50)
    print("ARGUS END-TO-END TEST SUITE")
    print("=" * 50)
    
    tests = [
        ("CSV Parsing", test_csv_parsing),
        ("Vector Store", test_vector_store),
        ("Ollama Connection", test_ollama_connection),
        ("Streamlit Setup", test_streamlit_launch)
    ]
    
    results = []
    
    for test_name, test_func in tests:
        print(f"\n{test_name}")
        print("-" * 30)
        success = test_func()
        results.append((test_name, success))
        time.sleep(1)
    
    # Print summary
    print("\n" + "=" * 50)
    print("TEST SUMMARY")
    print("=" * 50)
    
    passed = sum(1 for _, success in results if success)
    total = len(results)
    
    for test_name, success in results:
        status = "✓ PASSED" if success else "✗ FAILED"
        print(f"{test_name}: {status}")
    
    print(f"\nTotal: {passed}/{total} tests passed")
    
    if passed == total:
        print("\n🎉 All tests passed! Application is ready to use.")
        return 0
    else:
        print(f"\n⚠️ {total - passed} test(s) failed. Please check the errors above.")
        return 1

if __name__ == "__main__":
    sys.exit(run_all_tests())
```

## Validation Checklist
- [ ] All unit tests pass
- [ ] Integration tests complete successfully
- [ ] Security tests validate protections
- [ ] Performance tests meet requirements
- [ ] End-to-end workflow functions correctly
- [ ] Error handling tested for edge cases

## Success Criteria
- 100% of critical path tests pass
- CSV parsing handles various formats
- Vector store operations are reliable
- Security measures block malicious input
- Performance meets <3 second query response
- End-to-end workflow completes without errors

## Test Execution Commands
```bash
# Run all tests
python -m pytest tests/ -v

# Run specific test module
python tests/test_csv_parser.py

# Run end-to-end tests
python tests/test_e2e.py

# Run with coverage
python -m pytest tests/ --cov=src --cov-report=html

# Run performance tests only
python -m pytest tests/test_performance.py -v
```

## Next Phase
Phase 8: Enhancement & Optimization - Advanced features and performance improvements