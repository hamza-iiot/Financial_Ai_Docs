# Transaction Investigator Agent

## Overview
The Transaction Investigator Agent specializes in detailed transaction searches, finding specific payments, and investigating individual transactions or groups of related transactions. It's the go-to agent for precise queries about specific transactions.

## Agent Profile

- **Specialization**: Transaction search, detail retrieval, precise filtering
- **Model**: qwen3-14b-32k:latest
- **Search Depth**: 10-20 transactions (focused searches)
- **Query Count**: 3-4 targeted queries

## Implementation

```python
from typing import Dict, List, Any, Optional, Tuple
from datetime import datetime, timedelta
import pandas as pd
import re
from fuzzywuzzy import fuzz

class TransactionInvestigatorAgent:
    """Specialized agent for finding and investigating specific transactions"""
    
    def __init__(self, vector_store, ollama_client):
        self.vector_store = vector_store
        self.ollama = ollama_client
        self.model = "qwen3-14b-32k:latest"
        
        # Search strategies
        self.search_strategies = [
            'exact_match',
            'fuzzy_match',
            'date_range',
            'amount_range',
            'merchant_search',
            'reference_search'
        ]
        
        # Common transaction types to identify
        self.transaction_types = {
            'pos': ['pos', 'point of sale', 'purchase', 'نقطة بيع'],
            'online': ['online', 'internet', 'web', 'ecom', 'إنترنت'],
            'atm': ['atm', 'cash', 'withdrawal', 'صراف'],
            'transfer': ['transfer', 'تحويل', 'remittance', 'حوالة'],
            'bill': ['bill', 'utility', 'payment', 'فاتورة'],
            'salary': ['salary', 'payroll', 'راتب', 'wages']
        }
    
    def execute(self, query: str, thinking_context: Dict = None) -> Dict[str, Any]:
        """Execute precise transaction investigation"""
        
        # Step 1: Hidden investigation thinking
        internal_analysis = self._think_about_investigation(query, thinking_context)
        
        # Step 2: Generate targeted search queries
        search_queries = self._generate_investigation_queries(query, internal_analysis)
        
        # Step 3: Execute precise searches
        found_transactions = self._investigate_transactions(search_queries)
        
        # Step 4: Analyze and rank results
        investigation_results = self._analyze_findings(found_transactions, query, internal_analysis)
        
        # Step 5: Generate detailed response
        final_response = self._format_investigation_response(investigation_results)
        
        return {
            'final_answer': final_response,
            'sources': investigation_results['matches'][:10],
            'statistics': investigation_results['stats'],
            'search_metadata': investigation_results['metadata'],
            'thinking': internal_analysis  # Hidden
        }
    
    def _think_about_investigation(self, query: str, context: Dict = None) -> Dict:
        """Hidden reasoning about transaction search"""
        
        thinking_prompt = f"""Analyze this transaction search query for Saudi banking.
        
        Query: {query}
        
        Extract and consider (hidden):
        1. Search intent: Finding specific transaction or group?
        2. Identifiers: Dates, amounts, merchants, references?
        3. Time frame: Specific date or range?
        4. Amount details: Exact or approximate?
        5. Merchant/Description: Keywords to search?
        6. Transaction type: POS, ATM, transfer, etc.?
        7. Urgency: Is this a dispute or verification?
        
        Detailed search strategy:"""
        
        thinking = self.ollama.generate(
            prompt=thinking_prompt,
            system_prompt="You are a transaction investigation expert. Extract search parameters precisely.",
            temperature=0.1,  # Low temperature for precision
            max_tokens=500
        )
        
        # Extract search parameters
        search_params = self._extract_search_parameters(query, thinking)
        
        return {
            'reasoning': thinking,
            'timestamp': datetime.now(),
            'search_params': search_params,
            'query_type': self._classify_investigation_query(query),
            'context': context
        }
    
    def _extract_search_parameters(self, query: str, thinking: str) -> Dict:
        """Extract specific search parameters from query"""
        
        params = {
            'dates': [],
            'amounts': [],
            'merchants': [],
            'keywords': [],
            'type_hints': []
        }
        
        query_lower = query.lower()
        
        # Extract dates (various formats)
        date_patterns = [
            r'\d{1,2}[/-]\d{1,2}[/-]\d{2,4}',  # DD/MM/YYYY or MM/DD/YYYY
            r'\d{4}[/-]\d{1,2}[/-]\d{1,2}',     # YYYY-MM-DD
            r'(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)[a-z]* \d{1,2}',  # Month DD
            r'\d{1,2} (jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)[a-z]*',  # DD Month
            r'(yesterday|today|last week|last month)'  # Relative dates
        ]
        
        for pattern in date_patterns:
            matches = re.findall(pattern, query_lower)
            params['dates'].extend(matches)
        
        # Extract amounts (SAR, numbers)
        amount_patterns = [
            r'sar[\s]*([\d,]+\.?\d*)',  # SAR amounts
            r'([\d,]+\.?\d*)[\s]*sar',  # amounts SAR
            r'([\d,]+\.?\d*)[\s]*riyal',  # riyals
            r'[\$]?([\d,]+\.?\d*)',  # plain numbers
        ]
        
        for pattern in amount_patterns:
            matches = re.findall(pattern, query_lower)
            for match in matches:
                try:
                    amount = float(match.replace(',', ''))
                    params['amounts'].append(amount)
                except:
                    pass
        
        # Extract merchant names (capitalized words, quoted strings)
        merchant_patterns = [
            r'"([^"]+)"',  # Quoted strings
            r'\'([^\']+)\'',  # Single quoted
            r'at ([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*)',  # "at MerchantName"
            r'from ([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*)',  # "from MerchantName"
            r'to ([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*)',  # "to MerchantName"
        ]
        
        for pattern in merchant_patterns:
            matches = re.findall(pattern, query)  # Use original case
            params['merchants'].extend(matches)
        
        # Extract keywords
        important_words = [
            'restaurant', 'uber', 'careem', 'amazon', 'noon',
            'stc', 'mobily', 'zain', 'sabb', 'alrajhi',
            'atm', 'pos', 'online', 'transfer', 'salary'
        ]
        
        for word in important_words:
            if word in query_lower:
                params['keywords'].append(word)
        
        # Detect transaction type hints
        for trans_type, keywords in self.transaction_types.items():
            if any(keyword in query_lower for keyword in keywords):
                params['type_hints'].append(trans_type)
        
        return params
    
    def _generate_investigation_queries(self, original_query: str, thinking: Dict) -> List[Dict]:
        """Generate targeted search queries"""
        
        queries = []
        params = thinking['search_params']
        
        # Query 1: Direct keyword search
        if params['keywords'] or params['merchants']:
            search_terms = ' '.join(params['keywords'] + params['merchants'])
            queries.append({
                'query': search_terms,
                'n_results': 20,
                'filters': self._build_filters(params)
            })
        
        # Query 2: Amount-based search
        if params['amounts']:
            for amount in params['amounts'][:2]:  # Top 2 amounts
                queries.append({
                    'query': f'payment transaction {amount}',
                    'n_results': 10,
                    'filters': {
                        'amount': {
                            '$gte': amount - 0.01,
                            '$lte': amount + 0.01
                        }
                    }
                })
        
        # Query 3: Date-based search
        if params['dates']:
            queries.append({
                'query': ' '.join(params['dates']),
                'n_results': 15,
                'filters': {}
            })
        
        # Query 4: Type-based search
        if params['type_hints']:
            for hint in params['type_hints']:
                queries.append({
                    'query': ' '.join(self.transaction_types[hint]),
                    'n_results': 15,
                    'filters': self._build_filters(params)
                })
        
        # Fallback: General search
        if not queries:
            queries.append({
                'query': original_query,
                'n_results': 25,
                'filters': {}
            })
        
        return queries
    
    def _build_filters(self, params: Dict) -> Dict:
        """Build ChromaDB filters from parameters"""
        
        filters = {}
        
        # Amount filters
        if params['amounts']:
            if len(params['amounts']) == 1:
                amount = params['amounts'][0]
                filters['amount'] = {
                    '$gte': amount - 1,
                    '$lte': amount + 1
                }
            else:
                # Range between min and max
                filters['amount'] = {
                    '$gte': min(params['amounts']),
                    '$lte': max(params['amounts'])
                }
        
        return filters
    
    def _investigate_transactions(self, queries: List[Dict]) -> List[Dict]:
        """Execute targeted searches"""
        
        found_transactions = []
        seen_ids = set()
        
        for query_config in queries:
            try:
                results = self.vector_store.search(
                    query=query_config['query'],
                    n_results=query_config['n_results'],
                    filter_dict=query_config.get('filters')
                )
                
                for trans in results.get('transactions', []):
                    trans_id = f"{trans.get('date')}_{trans.get('amount')}_{trans.get('description')}"
                    if trans_id not in seen_ids:
                        seen_ids.add(trans_id)
                        # Add relevance score
                        trans['relevance_score'] = self._calculate_relevance(
                            trans,
                            query_config['query']
                        )
                        found_transactions.append(trans)
                        
            except Exception as e:
                continue
        
        # Sort by relevance
        return sorted(found_transactions, key=lambda x: x.get('relevance_score', 0), reverse=True)
    
    def _calculate_relevance(self, transaction: Dict, query: str) -> float:
        """Calculate transaction relevance score"""
        
        score = 0.0
        desc = str(transaction.get('description', '')).lower()
        query_lower = query.lower()
        
        # Exact match bonus
        if query_lower in desc:
            score += 50
        
        # Fuzzy matching
        fuzz_score = fuzz.partial_ratio(query_lower, desc)
        score += fuzz_score / 2
        
        # Date proximity (if searching recent)
        if 'recent' in query_lower or 'latest' in query_lower:
            trans_date = pd.to_datetime(transaction.get('date'))
            days_ago = (datetime.now() - trans_date).days
            if days_ago < 7:
                score += 20
            elif days_ago < 30:
                score += 10
        
        return score
    
    def _analyze_findings(self, transactions: List[Dict], query: str, thinking: Dict) -> Dict:
        """Analyze found transactions"""
        
        if not transactions:
            return {
                'matches': [],
                'stats': {'found': 0},
                'metadata': {},
                'summary': 'No matching transactions found'
            }
        
        df = pd.DataFrame(transactions)
        
        # Get best matches
        matches = transactions[:10]  # Top 10 by relevance
        
        # Calculate statistics
        stats = {
            'found': len(transactions),
            'total_amount': float(df['amount'].sum()),
            'date_range': {
                'earliest': df['date'].min(),
                'latest': df['date'].max()
            },
            'amount_range': {
                'min': float(df['amount'].min()),
                'max': float(df['amount'].max())
            }
        }
        
        # Search metadata
        metadata = {
            'search_params': thinking['search_params'],
            'best_match_score': matches[0].get('relevance_score', 0) if matches else 0,
            'search_strategy': self._determine_search_strategy(thinking['search_params'])
        }
        
        # Generate summary
        summary = self._generate_finding_summary(matches, stats, thinking['search_params'])
        
        return {
            'matches': matches,
            'stats': stats,
            'metadata': metadata,
            'summary': summary
        }
    
    def _determine_search_strategy(self, params: Dict) -> str:
        """Determine which search strategy was most effective"""
        
        if params['amounts']:
            return 'amount_search'
        elif params['dates']:
            return 'date_search'
        elif params['merchants']:
            return 'merchant_search'
        elif params['keywords']:
            return 'keyword_search'
        else:
            return 'general_search'
    
    def _generate_finding_summary(self, matches: List[Dict], stats: Dict, params: Dict) -> str:
        """Generate summary of findings"""
        
        if not matches:
            return "No transactions found matching your criteria"
        
        summary_parts = []
        
        # Found count
        summary_parts.append(f"Found {stats['found']} matching transaction(s)")
        
        # Amount info if relevant
        if params['amounts']:
            summary_parts.append(f"totaling SAR {stats['total_amount']:,.2f}")
        
        # Date range if multiple
        if stats['found'] > 1:
            summary_parts.append(
                f"between {stats['date_range']['earliest']} and {stats['date_range']['latest']}"
            )
        
        return '. '.join(summary_parts)
    
    def _format_investigation_response(self, results: Dict) -> str:
        """Format investigation results"""
        
        if not results['matches']:
            return self._format_no_results_response(results)
        
        prompt = f"""Present transaction investigation results for Saudi banking.

        Search Summary: {results['summary']}
        
        Found Transactions:
        {self._format_transactions_for_prompt(results['matches'][:5])}
        
        Statistics:
        - Total Found: {results['stats']['found']}
        - Total Amount: SAR {results['stats'].get('total_amount', 0):,.2f}
        - Date Range: {results['stats'].get('date_range', {}).get('earliest', 'N/A')} to {results['stats'].get('date_range', {}).get('latest', 'N/A')}
        
        Provide:
        1. Clear presentation of found transactions
        2. Highlight the most relevant match
        3. Summary of search results
        4. Any patterns noticed in the results
        
        Use SAR for amounts. Be precise with details."""
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="""You are a transaction investigation specialist for Saudi banking.
            Present findings clearly and precisely. Always use SAR for currency.""",
            temperature=0.1,  # Low temperature for accuracy
            max_tokens=500
        )
        
        return response
    
    def _format_no_results_response(self, results: Dict) -> str:
        """Format response when no results found"""
        
        params = results.get('metadata', {}).get('search_params', {})
        
        suggestions = []
        if params.get('amounts'):
            suggestions.append("Try searching with a broader amount range")
        if params.get('dates'):
            suggestions.append("Check if the date format is correct")
        if params.get('merchants'):
            suggestions.append("Try alternative merchant name spellings")
        
        prompt = f"""No transactions found matching the search criteria.

        Search Parameters:
        - Amounts: {params.get('amounts', [])}
        - Dates: {params.get('dates', [])}
        - Keywords: {params.get('keywords', [])}
        - Merchants: {params.get('merchants', [])}
        
        Suggestions:
        {chr(10).join(f"- {s}" for s in suggestions)}
        
        Provide helpful response explaining no results and offering alternatives."""
        
        response = self.ollama.generate(
            prompt=prompt,
            system_prompt="Help user understand why no results were found and suggest alternatives.",
            temperature=0.2,
            max_tokens=300
        )
        
        return response
    
    def _format_transactions_for_prompt(self, transactions: List[Dict]) -> str:
        """Format transactions for prompt"""
        
        if not transactions:
            return "No transactions"
        
        lines = []
        for i, trans in enumerate(transactions, 1):
            lines.append(
                f"{i}. {trans.get('date', 'N/A')} | "
                f"SAR {trans.get('amount', 0):,.2f} | "
                f"{trans.get('description', 'N/A')} | "
                f"{trans.get('type', 'N/A').upper()}"
            )
        
        return '\n'.join(lines)
    
    def _classify_investigation_query(self, query: str) -> str:
        """Classify investigation query type"""
        
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['find', 'search', 'look for', 'where']):
            return 'search'
        elif any(word in query_lower for word in ['verify', 'confirm', 'check']):
            return 'verification'
        elif any(word in query_lower for word in ['dispute', 'wrong', 'error']):
            return 'dispute'
        elif any(word in query_lower for word in ['all', 'list', 'show']):
            return 'listing'
        else:
            return 'general'
    
    def investigate_for_dispute(self, transaction_details: Dict) -> Dict[str, Any]:
        """Special investigation for transaction disputes"""
        
        thinking = self._think_about_investigation(
            f"Investigate disputed transaction: {transaction_details}",
            context={'mode': 'dispute'}
        )
        
        # Search for similar transactions
        queries = [
            {
                'query': transaction_details.get('description', ''),
                'n_results': 10,
                'filters': {
                    'amount': {
                        '$gte': transaction_details['amount'] - 1,
                        '$lte': transaction_details['amount'] + 1
                    }
                }
            }
        ]
        
        similar_trans = self._investigate_transactions(queries)
        
        # Analyze for patterns
        analysis = {
            'similar_count': len(similar_trans),
            'is_recurring': len(similar_trans) > 1,
            'merchant_history': self._analyze_merchant_history(similar_trans),
            'recommendations': self._generate_dispute_recommendations(similar_trans)
        }
        
        return analysis
    
    def _analyze_merchant_history(self, transactions: List[Dict]) -> Dict:
        """Analyze merchant transaction history"""
        
        if not transactions:
            return {'first_occurrence': True, 'frequency': 0}
        
        df = pd.DataFrame(transactions)
        df['date'] = pd.to_datetime(df['date'])
        
        return {
            'first_occurrence': len(df) == 1,
            'frequency': len(df),
            'date_range': {
                'first': df['date'].min().strftime('%Y-%m-%d'),
                'last': df['date'].max().strftime('%Y-%m-%d')
            },
            'total_amount': float(df['amount'].sum())
        }
    
    def _generate_dispute_recommendations(self, similar_trans: List[Dict]) -> List[str]:
        """Generate recommendations for disputes"""
        
        recommendations = []
        
        if len(similar_trans) > 1:
            recommendations.append("This appears to be a recurring transaction")
        else:
            recommendations.append("This is the only transaction of this type found")
        
        if similar_trans:
            recommendations.append("Keep transaction reference for bank dispute")
        
        return recommendations
```

## Query Examples

### User Query → Agent Response

**Query**: "Find the Uber transaction for SAR 45 last Tuesday"
```
Hidden Thinking:
- Merchant: Uber
- Amount: SAR 45
- Date: Last Tuesday (calculate actual date)
- Search strategy: Combined merchant + amount + date

Response:
Found 1 matching Uber transaction:

**Transaction Details:**
📅 Date: 2024-10-22 (Tuesday)
💰 Amount: SAR 45.00
📝 Description: UBER TRIP - RIYADH
🏷️ Type: DEBIT (POS Transaction)
🔖 Reference: POS-2024-10-22-1847

This matches your search criteria exactly.
```

**Query**: "Show me all transactions at Jarir Bookstore"
```
Hidden Thinking:
- Merchant: Jarir Bookstore
- No date/amount specified
- Search all Jarir transactions

Response:
Found 4 transactions at Jarir Bookstore:

1. **Oct 15, 2024** - SAR 299.00
   Books and stationery purchase

2. **Sep 28, 2024** - SAR 1,450.00
   Electronics - iPad purchase

3. **Sep 10, 2024** - SAR 89.00
   Office supplies

4. **Aug 20, 2024** - SAR 550.00
   Back-to-school shopping

**Total at Jarir: SAR 2,388.00** over 2 months
Pattern: Monthly visits, average SAR 597.00 per visit
```

**Query**: "I don't remember a charge for 1250 riyals"
```
Hidden Thinking:
- Amount: SAR 1250
- User doesn't recognize - possible dispute
- Need to investigate thoroughly

Response:
Found 2 transactions for SAR 1,250.00:

**Most Recent:**
📅 Oct 18, 2024 - SAR 1,250.00
📝 "NOON.COM - ORDER #78439"
🏷️ Online purchase
⚡ First time from this merchant

**Previous:**
📅 Sep 25, 2024 - SAR 1,250.00
📝 "MONTHLY RENT PAYMENT"
🏷️ Recurring payment
🔄 Occurs monthly on 25th

**Investigation Notes:**
- The Oct 18 charge is unusual (first time merchant)
- The Sep 25 charge is your regular rent
- Check your Noon.com account for order #78439
- If unrecognized, contact your bank immediately
```