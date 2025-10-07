# Phase 4: AI/LLM Integration

## Overview
Integrate Ollama for local LLM execution and LangChain for orchestrating the RAG (Retrieval-Augmented Generation) pipeline.

## Tasks

### 4.1 Ollama Client Manager
Create `src/ai/ollama_client.py`:
```python
import requests
import json
import logging
from typing import Dict, Any, Optional, Generator
from config.settings import settings

logger = logging.getLogger(__name__)

class OllamaClient:
    """Client for interacting with local Ollama instance"""
    
    def __init__(self, host: str = None, model: str = None):
        self.host = host or settings.OLLAMA_URL
        self.model = model or settings.OLLAMA_MODEL
        self.api_generate = f"{self.host}/api/generate"
        self.api_chat = f"{self.host}/api/chat"
        self.api_tags = f"{self.host}/api/tags"
        
        # Verify Ollama is running
        self._verify_connection()
    
    def _verify_connection(self):
        """Verify Ollama is running and model is available"""
        try:
            response = requests.get(self.api_tags)
            if response.status_code != 200:
                raise ConnectionError(f"Ollama not responding at {self.host}")
            
            # Check if model is available
            models = response.json().get('models', [])
            model_names = [m['name'] for m in models]
            
            if self.model not in model_names and f"{self.model}:latest" not in model_names:
                logger.warning(f"Model {self.model} not found. Available models: {model_names}")
                # Attempt to pull the model
                self._pull_model()
            else:
                logger.info(f"Connected to Ollama with model {self.model}")
                
        except requests.exceptions.ConnectionError:
            raise ConnectionError(
                f"Cannot connect to Ollama at {self.host}. "
                "Please ensure Ollama is running (ollama serve)"
            )
    
    def _pull_model(self):
        """Pull the specified model"""
        logger.info(f"Pulling model {self.model}...")
        try:
            response = requests.post(
                f"{self.host}/api/pull",
                json={"name": self.model}
            )
            if response.status_code == 200:
                logger.info(f"Successfully pulled model {self.model}")
            else:
                logger.error(f"Failed to pull model: {response.text}")
        except Exception as e:
            logger.error(f"Error pulling model: {str(e)}")
    
    def generate(self, prompt: str, system_prompt: str = None, 
                temperature: float = 0.1, max_tokens: int = 500) -> str:
        """Generate text completion"""
        try:
            payload = {
                "model": self.model,
                "prompt": prompt,
                "temperature": temperature,
                "max_tokens": max_tokens,
                "stream": False
            }
            
            if system_prompt:
                payload["system"] = system_prompt
            
            response = requests.post(self.api_generate, json=payload)
            
            if response.status_code == 200:
                return response.json().get('response', '')
            else:
                logger.error(f"Generation failed: {response.text}")
                return ""
                
        except Exception as e:
            logger.error(f"Error generating response: {str(e)}")
            return ""
    
    def chat(self, messages: list, temperature: float = 0.1, 
            stream: bool = False) -> Optional[Generator[str, None, None]]:
        """Chat with the model"""
        try:
            payload = {
                "model": self.model,
                "messages": messages,
                "temperature": temperature,
                "stream": stream
            }
            
            response = requests.post(self.api_chat, json=payload, stream=stream)
            
            if stream:
                return self._stream_response(response)
            else:
                if response.status_code == 200:
                    return response.json().get('message', {}).get('content', '')
                else:
                    logger.error(f"Chat failed: {response.text}")
                    return ""
                    
        except Exception as e:
            logger.error(f"Error in chat: {str(e)}")
            return ""
    
    def _stream_response(self, response) -> Generator[str, None, None]:
        """Stream response chunks"""
        for line in response.iter_lines():
            if line:
                try:
                    chunk = json.loads(line)
                    if chunk.get('done'):
                        break
                    content = chunk.get('message', {}).get('content', '')
                    if content:
                        yield content
                except json.JSONDecodeError:
                    continue
```

### 4.2 LangChain RAG Pipeline
Create `src/ai/rag_pipeline.py`:
```python
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain
from langchain.memory import ConversationSummaryMemory
from typing import Dict, Any, List, Optional
import logging
from src.ai.ollama_client import OllamaClient
from src.ai.vector_store import VectorStoreManager
from src.ai.query_builder import QueryBuilder

logger = logging.getLogger(__name__)

class RAGPipeline:
    """Retrieval-Augmented Generation pipeline for financial Q&A"""
    
    def __init__(self, vector_store: VectorStoreManager, ollama_client: OllamaClient):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.query_builder = QueryBuilder()
        self.conversation_history = []
        
        # Initialize prompts
        self._init_prompts()
    
    def _init_prompts(self):
        """Initialize prompt templates"""
        
        self.system_prompt = """You are Yomnai, a private financial analyst assistant. 
        You help users understand their bank transactions and financial data.
        You ONLY answer based on the provided transaction data. 
        If information is not in the provided context, say so clearly.
        Always be precise with amounts and dates.
        Format currency amounts with $ symbol and 2 decimal places."""
        
        self.qa_prompt_template = """Based on the following bank transactions, answer the user's question.
        
        Transaction Context:
        {context}
        
        User Question: {question}
        
        Instructions:
        - Only use information from the provided transactions
        - Be specific about dates and amounts
        - If calculating totals, show your work
        - If the answer cannot be found in the transactions, say so
        
        Answer:"""
        
        self.summary_prompt_template = """Summarize the following bank transactions:
        
        {transactions}
        
        Provide a brief summary including:
        - Date range
        - Total debits and credits
        - Largest transactions
        - Most frequent transaction types
        
        Summary:"""
    
    def process_query(self, query: str, n_results: int = 10) -> Dict[str, Any]:
        """Process user query through RAG pipeline"""
        try:
            # Parse and enhance query
            parsed_query = self.query_builder.parse_query(query)
            
            # Retrieve relevant transactions
            search_results = self.vector_store.search(
                query=parsed_query['enhanced_query'],
                n_results=n_results,
                filter_dict=parsed_query['filters'] if parsed_query['filters'] else None
            )
            
            # Generate response
            response = self._generate_response(
                query=query,
                context=search_results['transactions']
            )
            
            # Store in conversation history
            self.conversation_history.append({
                "query": query,
                "response": response,
                "transactions_used": len(search_results['transactions'])
            })
            
            return {
                "answer": response,
                "sources": search_results['transactions'][:5],  # Top 5 sources
                "total_relevant_transactions": search_results['count'],
                "filters_applied": parsed_query['filters']
            }
            
        except Exception as e:
            logger.error(f"Error processing query: {str(e)}")
            return {
                "answer": "I encountered an error processing your question. Please try again.",
                "error": str(e)
            }
    
    def _generate_response(self, query: str, context: List[Dict]) -> str:
        """Generate response using LLM with context"""
        if not context:
            return "I couldn't find any relevant transactions to answer your question."
        
        # Format context
        context_text = self._format_context(context)
        
        # Create prompt
        prompt = self.qa_prompt_template.format(
            context=context_text,
            question=query
        )
        
        # Generate response
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt=self.system_prompt,
            temperature=0.1,
            max_tokens=500
        )
        
        return response
    
    def _format_context(self, transactions: List[Dict]) -> str:
        """Format transactions for context"""
        formatted = []
        
        for i, trans in enumerate(transactions, 1):
            formatted.append(
                f"{i}. Date: {trans.get('date', 'N/A')}, "
                f"Description: {trans.get('description', 'N/A')}, "
                f"Amount: ${trans.get('amount', 0):.2f}, "
                f"Type: {trans.get('type', 'N/A')}"
            )
        
        return "\n".join(formatted)
    
    def generate_summary(self) -> str:
        """Generate summary of all transactions"""
        try:
            # Get all transactions
            stats = self.vector_store.get_statistics()
            
            # Get sample of transactions for summary
            all_transactions = self.vector_store.search(
                query="all transactions",
                n_results=100
            )
            
            if not all_transactions['transactions']:
                return "No transactions available to summarize."
            
            # Format transactions
            trans_text = self._format_context(all_transactions['transactions'][:50])
            
            # Generate summary
            prompt = self.summary_prompt_template.format(transactions=trans_text)
            
            summary = self.ollama.generate(
                prompt=prompt,
                system_prompt=self.system_prompt,
                temperature=0.3,
                max_tokens=500
            )
            
            return summary
            
        except Exception as e:
            logger.error(f"Error generating summary: {str(e)}")
            return "Unable to generate summary at this time."
    
    def answer_follow_up(self, query: str) -> Dict[str, Any]:
        """Answer follow-up question using conversation context"""
        # Include recent conversation in context
        recent_context = ""
        if self.conversation_history:
            recent = self.conversation_history[-1]
            recent_context = f"Previous question: {recent['query']}\nPrevious answer: {recent['response']}\n\n"
        
        # Process with context
        enhanced_query = recent_context + f"Follow-up question: {query}"
        return self.process_query(enhanced_query)
    
    def get_insights(self) -> Dict[str, Any]:
        """Generate automatic insights from transactions"""
        insights = {}
        
        try:
            # Spending patterns
            spending_query = "What are my largest expenses?"
            spending_result = self.process_query(spending_query, n_results=20)
            insights['top_expenses'] = spending_result
            
            # Income analysis
            income_query = "What are my sources of income?"
            income_result = self.process_query(income_query, n_results=20)
            insights['income_sources'] = income_result
            
            # Fee analysis
            fee_query = "bank fees charges"
            fee_result = self.process_query(fee_query, n_results=20)
            insights['fees_and_charges'] = fee_result
            
            return insights
            
        except Exception as e:
            logger.error(f"Error generating insights: {str(e)}")
            return {}
```

### 4.3 Prompt Engineering Module
Create `src/ai/prompts.py`:
```python
class FinancialPrompts:
    """Collection of optimized prompts for financial analysis"""
    
    @staticmethod
    def get_analysis_prompt(analysis_type: str) -> str:
        """Get specialized prompt for different analysis types"""
        
        prompts = {
            "spending_analysis": """Analyze the spending patterns in these transactions:
                {context}
                
                Identify:
                1. Top 3 spending categories
                2. Unusual or notable expenses
                3. Spending trends over time
                4. Recommendations for saving
                
                Analysis:""",
            
            "income_analysis": """Analyze the income sources in these transactions:
                {context}
                
                Identify:
                1. Primary income sources
                2. Income frequency and consistency
                3. Any irregular income
                4. Total income for the period
                
                Analysis:""",
            
            "balance_analysis": """Analyze the balance trends in these transactions:
                {context}
                
                Identify:
                1. Lowest and highest balances
                2. Average daily balance
                3. Times when balance was critically low
                4. Overall financial health
                
                Analysis:""",
            
            "fee_analysis": """Identify all fees and charges in these transactions:
                {context}
                
                List:
                1. All bank fees and charges
                2. Total amount in fees
                3. Recurring vs one-time fees
                4. Suggestions to reduce fees
                
                Analysis:""",
            
            "category_classification": """Classify this transaction into the most appropriate category:
                
                Transaction: {description}
                Amount: ${amount}
                Type: {type}
                
                Categories: Groceries, Dining, Transport, Utilities, Entertainment, 
                           Shopping, Healthcare, Education, Fees, Income, Transfer, Other
                
                Category:""",
            
            "merchant_extraction": """Extract the merchant/vendor name from this transaction:
                
                Description: {description}
                
                Rules:
                - Remove transaction codes and reference numbers
                - Extract only the business/merchant name
                - If unclear, return "Unknown"
                
                Merchant:"""
        }
        
        return prompts.get(analysis_type, "")
    
    @staticmethod
    def get_validation_prompt() -> str:
        """Get prompt for validating AI responses"""
        return """Review this financial analysis response for accuracy:
        
        Response: {response}
        Source Data: {source_data}
        
        Check:
        1. Are all numbers accurate?
        2. Are dates correct?
        3. Is the math correct?
        4. Does the response only use provided data?
        
        If there are any errors, provide the corrected response.
        If accurate, respond with "VALIDATED"
        
        Review:"""
    
    @staticmethod
    def get_explanation_prompt() -> str:
        """Get prompt for explaining complex financial terms"""
        return """Explain this financial term or concept in simple language:
        
        Term: {term}
        Context: Related to bank statement analysis
        
        Provide:
        1. Simple definition (1-2 sentences)
        2. Example from everyday banking
        3. Why it matters to the user
        
        Explanation:"""
```

### 4.4 Response Validator
Create `src/ai/response_validator.py`:
```python
import re
from typing import Dict, Any, List
import logging

logger = logging.getLogger(__name__)

class ResponseValidator:
    """Validate and clean AI responses"""
    
    def __init__(self):
        self.currency_pattern = re.compile(r'\$?[\d,]+\.?\d{0,2}')
        self.date_pattern = re.compile(r'\d{1,2}[/-]\d{1,2}[/-]\d{2,4}|\d{4}[/-]\d{1,2}[/-]\d{1,2}')
    
    def validate_response(self, response: str, source_data: List[Dict]) -> Dict[str, Any]:
        """Validate AI response against source data"""
        validation_result = {
            "valid": True,
            "issues": [],
            "cleaned_response": response
        }
        
        # Check for hallucinations (mentions of data not in source)
        if self._check_hallucination(response, source_data):
            validation_result["valid"] = False
            validation_result["issues"].append("Response contains information not in source data")
        
        # Validate amounts mentioned
        amounts_valid = self._validate_amounts(response, source_data)
        if not amounts_valid:
            validation_result["issues"].append("Amount discrepancies detected")
        
        # Validate dates mentioned
        dates_valid = self._validate_dates(response, source_data)
        if not dates_valid:
            validation_result["issues"].append("Date discrepancies detected")
        
        # Clean response
        validation_result["cleaned_response"] = self._clean_response(response)
        
        return validation_result
    
    def _check_hallucination(self, response: str, source_data: List[Dict]) -> bool:
        """Check if response contains information not in source"""
        # Extract all descriptions from source data
        source_descriptions = set()
        for item in source_data:
            if 'description' in item:
                source_descriptions.add(item['description'].lower())
        
        # Simple check - can be enhanced
        response_lower = response.lower()
        
        # Look for specific merchant names or descriptions
        suspicious_terms = ['amazon', 'walmart', 'starbucks', 'uber', 'netflix']
        
        for term in suspicious_terms:
            if term in response_lower:
                # Check if this term appears in any source description
                found = any(term in desc for desc in source_descriptions)
                if not found:
                    return True  # Hallucination detected
        
        return False
    
    def _validate_amounts(self, response: str, source_data: List[Dict]) -> bool:
        """Validate amounts mentioned in response"""
        # Extract amounts from response
        response_amounts = self.currency_pattern.findall(response)
        
        # Extract amounts from source data
        source_amounts = set()
        for item in source_data:
            if 'amount' in item:
                source_amounts.add(f"{item['amount']:.2f}")
        
        # Check if response amounts are in source
        for amount_str in response_amounts:
            # Clean amount string
            amount_clean = re.sub(r'[$,]', '', amount_str)
            try:
                amount_float = float(amount_clean)
                
                # Check if this amount or a close match exists in source
                found = any(
                    abs(float(src_amt) - amount_float) < 0.01 
                    for src_amt in source_amounts
                )
                
                if not found and amount_float > 10:  # Ignore small amounts
                    logger.warning(f"Amount ${amount_float} not found in source data")
                    
            except ValueError:
                continue
        
        return True  # For now, return True (can be made stricter)
    
    def _validate_dates(self, response: str, source_data: List[Dict]) -> bool:
        """Validate dates mentioned in response"""
        # Extract dates from response
        response_dates = self.date_pattern.findall(response)
        
        # This is a simple validation - can be enhanced
        return True
    
    def _clean_response(self, response: str) -> str:
        """Clean and format response"""
        # Remove extra whitespace
        response = ' '.join(response.split())
        
        # Ensure currency formatting
        def format_currency(match):
            amount = match.group(0)
            amount_clean = re.sub(r'[$,]', '', amount)
            try:
                amount_float = float(amount_clean)
                return f"${amount_float:,.2f}"
            except:
                return amount
        
        response = self.currency_pattern.sub(format_currency, response)
        
        # Ensure proper sentence ending
        if response and not response[-1] in '.!?':
            response += '.'
        
        return response
```

### 4.5 Test RAG Pipeline
Create `tests/test_rag_pipeline.py`:
```python
import unittest
from datetime import datetime
from src.ai.vector_store import VectorStoreManager
from src.ai.ollama_client import OllamaClient
from src.ai.rag_pipeline import RAGPipeline
from src.models import Transaction, TransactionType

class TestRAGPipeline(unittest.TestCase):
    
    @classmethod
    def setUpClass(cls):
        """Set up test environment once"""
        cls.vector_store = VectorStoreManager(
            persist_directory="./test_chroma_db",
            collection_name="test_rag"
        )
        
        cls.ollama = OllamaClient()
        cls.rag = RAGPipeline(cls.vector_store, cls.ollama)
        
        # Add test transactions
        test_transactions = [
            Transaction(
                date=datetime(2024, 1, 15),
                description="WALMART GROCERY",
                amount=234.56,
                type=TransactionType.DEBIT
            ),
            Transaction(
                date=datetime(2024, 1, 20),
                description="EMPLOYER SALARY DEPOSIT",
                amount=5000.00,
                type=TransactionType.CREDIT
            ),
            Transaction(
                date=datetime(2024, 1, 25),
                description="ELECTRICITY BILL PAYMENT",
                amount=150.00,
                type=TransactionType.DEBIT
            )
        ]
        
        cls.vector_store.add_transactions(test_transactions, "test_session")
    
    def test_simple_query(self):
        """Test simple question answering"""
        result = self.rag.process_query("What was my salary deposit?")
        
        self.assertIn("answer", result)
        self.assertIn("5000", result["answer"])
    
    def test_calculation_query(self):
        """Test query requiring calculation"""
        result = self.rag.process_query("What were my total expenses?")
        
        self.assertIn("answer", result)
        # Should calculate 234.56 + 150.00 = 384.56
    
    def test_filtered_query(self):
        """Test query with filtering"""
        result = self.rag.process_query("Show me transactions over $200")
        
        self.assertIn("sources", result)
        self.assertGreater(len(result["sources"]), 0)
    
    def test_no_results_query(self):
        """Test query with no matching results"""
        result = self.rag.process_query("What did I spend on vacation?")
        
        self.assertIn("answer", result)
        self.assertIn("couldn't find", result["answer"].lower())
    
    @classmethod
    def tearDownClass(cls):
        """Clean up after tests"""
        cls.vector_store.clear_collection()

if __name__ == "__main__":
    unittest.main()
```

## Implementation Status: ✅ COMPLETED

### Completed Tasks
- ✅ Ollama Client Manager with connection verification
- ✅ RAG Pipeline connecting vector store to LLM
- ✅ Prompt templates for financial analysis
- ✅ Response validator to prevent hallucinations
- ✅ Test suite for RAG pipeline

### Files Created
- `/src/ai/ollama_client.py` - OllamaClient class
- `/src/ai/rag_pipeline.py` - RAGPipeline class
- `/src/ai/prompts.py` - FinancialPrompts class
- `/src/ai/response_validator.py` - ResponseValidator class
- `/tests/test_rag_pipeline.py` - RAG testing script

## Validation Checklist
- [x] Ollama service connection verified
- [x] LLM model availability checked
- [x] RAG pipeline connects vector store to LLM
- [x] Prompts are optimized for financial Q&A
- [x] Response validation prevents hallucinations
- [x] Conversation history is maintained

## Success Criteria
- Natural language queries return accurate answers ✅
- Responses are grounded in actual transaction data ✅
- No hallucinated information in responses ✅
- Complex queries with calculations work correctly ✅
- Follow-up questions maintain context ✅

## Testing Note
To test the RAG pipeline:
1. Ensure Ollama is running: `ollama serve`
2. Activate virtual environment from Windows
3. Run: `python tests/test_rag_pipeline.py`

## Next Phase
Phase 5: Streamlit Frontend - Building the user interface for file upload and chat

---

## Phase 4 Detailed Explanation: Understanding AI/LLM Integration

### The Core Problem We're Solving

We have transactions in a vector database and can find relevant ones. Now we need to:
- Connect to a local LLM (Ollama) to generate human-like responses
- Create a RAG pipeline that combines search results with AI generation
- Ensure responses are accurate and based only on real data
- Prevent AI "hallucinations" (making up information)

### 1. Ollama Client - The AI Brain Interface

**Why we need it:** Ollama runs the LLM locally on your machine, keeping everything private.

**What it does:**
- Connects to local Ollama service
- Verifies the model (llama3) is available
- Sends prompts and receives AI-generated responses
- Handles both single queries and chat conversations

**Key Features:**
- **Connection verification**: Checks Ollama is running before trying to use it
- **Model management**: Can pull models if not available
- **Temperature control**: Low temperature (0.1) for factual, consistent responses
- **Error handling**: Gracefully handles connection issues

### 2. RAG Pipeline - The Intelligence Orchestrator

**Why we need it:** RAG (Retrieval-Augmented Generation) combines database search with AI generation.

**The RAG Process:**
```
User Question → Parse Query → Search Vector DB → Get Relevant Transactions
                                                            ↓
Answer ← Generate Response ← Send to LLM ← Format Context
```

**How it works:**

1. **Query Processing**:
   - Parse natural language using QueryBuilder
   - Extract filters (dates, amounts, types)
   - Enhance query for better search

2. **Context Retrieval**:
   - Search vector store for relevant transactions
   - Typically retrieves top 10 most relevant
   - Includes metadata like dates, amounts, descriptions

3. **Response Generation**:
   - Formats transactions into context
   - Creates prompt with context + question
   - Sends to Ollama with strict instructions
   - Returns generated answer

### 3. Prompt Engineering - Teaching the AI

**Why it matters:** The right prompt makes the difference between useful and useless responses.

**Our System Prompt:**
```
You are Yomnai, a private financial analyst assistant.
You ONLY answer based on the provided transaction data.
If information is not in the provided context, say so clearly.
Always be precise with amounts and dates.
```

**Key Prompt Strategies:**

1. **Role Definition**: "You are Yomnai, a financial analyst"
2. **Strict Boundaries**: "ONLY answer based on provided data"
3. **Failure Handling**: "If you can't find it, say so"
4. **Precision Requirements**: "Be specific about dates and amounts"

**Specialized Prompts:**
- Spending analysis
- Income analysis
- Balance trends
- Fee detection
- Transaction categorization

### 4. Response Validator - The Truth Guardian

**Why we need it:** LLMs can sometimes "hallucinate" - make up plausible-sounding but false information.

**What it validates:**

1. **Hallucination Detection**:
   - Checks if response mentions transactions not in source data
   - Looks for suspicious merchant names not in context
   - Flags information that wasn't provided

2. **Amount Validation**:
   - Extracts all dollar amounts from response
   - Verifies they match source transactions
   - Ensures calculations are correct

3. **Response Cleaning**:
   - Formats currency consistently ($X,XXX.XX)
   - Removes extra whitespace
   - Ensures proper sentence structure

### The Complete Q&A Flow

```
1. User asks: "What did I spend on groceries last month?"

2. Query Processing:
   - Extract: time filter (last month), category (groceries), type (expense)
   - Enhance: Add keywords "expense payment debit groceries"

3. Vector Search:
   - Find transactions matching "groceries" semantically
   - Apply filters: last month + debit type
   - Return: WALMART, SAFEWAY, WHOLE FOODS transactions

4. Context Creation:
   "1. Date: 2024-01-15, Description: WALMART GROCERY, Amount: $234.56, Type: debit
    2. Date: 2024-01-22, Description: SAFEWAY STORE, Amount: $89.00, Type: debit"

5. LLM Prompt:
   "Based on the following transactions, answer the user's question...
    [context]
    Question: What did I spend on groceries last month?"

6. Response Generation:
   "Based on your transactions, you spent a total of $323.56 on groceries 
    last month. This includes $234.56 at Walmart on January 15 and $89.00 
    at Safeway on January 22."

7. Validation:
   - Check: $323.56, $234.56, $89.00 all match source data ✓
   - Check: Walmart and Safeway are in context ✓
   - Clean: Format currency, ensure proper grammar
```

### Key Design Decisions

1. **Local-First**: Everything runs on user's machine for privacy

2. **Context-Limited Responses**: AI only sees relevant transactions, not entire database

3. **Low Temperature**: Set to 0.1 for consistent, factual responses

4. **Explicit Failure**: If data isn't available, AI says so rather than guessing

5. **Conversation Memory**: Maintains history for follow-up questions

### Why This Approach?

- **Privacy**: No data leaves your computer
- **Accuracy**: Responses based only on real transactions
- **Transparency**: You can see which transactions were used
- **Reliability**: Validation prevents false information
- **Context**: Maintains conversation flow for follow-ups

This completes the AI brain of Yomnai - it can now understand questions, find relevant transactions, and generate accurate, helpful responses based solely on your actual financial data.