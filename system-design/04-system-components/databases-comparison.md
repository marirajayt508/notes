# Database Systems Comparison

This guide provides a comprehensive comparison of different database systems, their use cases, trade-offs, and implementation considerations for system design interviews and real-world applications.

## Table of Contents
- [Database Categories](#database-categories)
- [Relational Databases (RDBMS)](#relational-databases-rdbms)
- [NoSQL Databases](#nosql-databases)
- [NewSQL Databases](#newsql-databases)
- [Specialized Databases](#specialized-databases)
- [Database Selection Framework](#database-selection-framework)
- [Interview Questions & Answers](#interview-questions--answers)

## Database Categories

### Overview of Database Types

```python
from enum import Enum
from typing import Dict, List, Any
from dataclasses import dataclass

class DatabaseType(Enum):
    RELATIONAL = "relational"
    DOCUMENT = "document"
    KEY_VALUE = "key_value"
    COLUMN_FAMILY = "column_family"
    GRAPH = "graph"
    TIME_SERIES = "time_series"
    SEARCH = "search"
    CACHE = "cache"

@dataclass
class DatabaseCharacteristics:
    consistency: str  # Strong, Eventual, Tunable
    availability: str  # High, Medium, Low
    partition_tolerance: str  # Yes, No
    scalability: str  # Vertical, Horizontal, Both
    query_complexity: str  # Simple, Medium, Complex
    schema_flexibility: str  # Rigid, Flexible, Schema-less
    acid_compliance: str  # Full, Partial, None
    use_cases: List[str]
    examples: List[str]

class DatabaseComparison:
    def __init__(self):
        self.databases = {
            DatabaseType.RELATIONAL: DatabaseCharacteristics(
                consistency="Strong",
                availability="Medium",
                partition_tolerance="No",
                scalability="Vertical",
                query_complexity="Complex",
                schema_flexibility="Rigid",
                acid_compliance="Full",
                use_cases=[
                    "Financial transactions",
                    "E-commerce orders",
                    "User management",
                    "Inventory systems"
                ],
                examples=["PostgreSQL", "MySQL", "Oracle", "SQL Server"]
            ),
            DatabaseType.DOCUMENT: DatabaseCharacteristics(
                consistency="Eventual/Tunable",
                availability="High",
                partition_tolerance="Yes",
                scalability="Horizontal",
                query_complexity="Medium",
                schema_flexibility="Flexible",
                acid_compliance="Partial",
                use_cases=[
                    "Content management",
                    "Product catalogs",
                    "User profiles",
                    "Configuration data"
                ],
                examples=["MongoDB", "CouchDB", "Amazon DocumentDB"]
            ),
            DatabaseType.KEY_VALUE: DatabaseCharacteristics(
                consistency="Eventual/Tunable",
                availability="High",
                partition_tolerance="Yes",
                scalability="Horizontal",
                query_complexity="Simple",
                schema_flexibility="Schema-less",
                acid_compliance="None",
                use_cases=[
                    "Caching",
                    "Session storage",
                    "Shopping carts",
                    "Real-time recommendations"
                ],
                examples=["Redis", "DynamoDB", "Riak", "Voldemort"]
            ),
            DatabaseType.COLUMN_FAMILY: DatabaseCharacteristics(
                consistency="Tunable",
                availability="High",
                partition_tolerance="Yes",
                scalability="Horizontal",
                query_complexity="Medium",
                schema_flexibility="Flexible",
                acid_compliance="Partial",
                use_cases=[
                    "Time-series data",
                    "IoT sensor data",
                    "Analytics",
                    "Logging systems"
                ],
                examples=["Cassandra", "HBase", "Amazon SimpleDB"]
            )
        }
    
    def compare_databases(self, db_types: List[DatabaseType]) -> Dict[str, Any]:
        """Compare multiple database types"""
        comparison = {}
        
        for db_type in db_types:
            if db_type in self.databases:
                comparison[db_type.value] = self.databases[db_type]
        
        return comparison
    
    def recommend_database(self, requirements: Dict[str, Any]) -> List[DatabaseType]:
        """Recommend database types based on requirements"""
        recommendations = []
        
        for db_type, characteristics in self.databases.items():
            score = self._calculate_match_score(requirements, characteristics)
            if score > 0.7:  # 70% match threshold
                recommendations.append((db_type, score))
        
        # Sort by score descending
        recommendations.sort(key=lambda x: x[1], reverse=True)
        return [db_type for db_type, _ in recommendations]
    
    def _calculate_match_score(self, requirements: Dict[str, Any], 
                              characteristics: DatabaseCharacteristics) -> float:
        """Calculate how well a database matches requirements"""
        score = 0.0
        total_weight = 0.0
        
        # Define weights for different factors
        weights = {
            'consistency': 0.2,
            'availability': 0.2,
            'scalability': 0.15,
            'query_complexity': 0.15,
            'schema_flexibility': 0.1,
            'acid_compliance': 0.1,
            'partition_tolerance': 0.1
        }
        
        for factor, weight in weights.items():
            if factor in requirements:
                required_value = requirements[factor]
                actual_value = getattr(characteristics, factor)
                
                # Simple string matching (in practice, would be more sophisticated)
                if required_value.lower() in actual_value.lower():
                    score += weight
                
                total_weight += weight
        
        return score / total_weight if total_weight > 0 else 0.0
```

## Relational Databases (RDBMS)

### Core Concepts

```python
import sqlite3
from typing import List, Dict, Any, Optional
from dataclasses import dataclass
from datetime import datetime

@dataclass
class User:
    id: Optional[int]
    email: str
    name: str
    created_at: datetime
    
@dataclass
class Order:
    id: Optional[int]
    user_id: int
    total_amount: float
    status: str
    created_at: datetime

@dataclass
class OrderItem:
    id: Optional[int]
    order_id: int
    product_id: int
    quantity: int
    unit_price: float

class RelationalDatabaseExample:
    """Example of relational database operations with ACID properties"""
    
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.conn.row_factory = sqlite3.Row
        self._create_tables()
    
    def _create_tables(self):
        """Create normalized tables with relationships"""
        cursor = self.conn.cursor()
        
        # Users table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                email TEXT UNIQUE NOT NULL,
                name TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        # Orders table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                total_amount DECIMAL(10,2) NOT NULL,
                status TEXT NOT NULL DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users (id)
            )
        ''')
        
        # Order items table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS order_items (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                order_id INTEGER NOT NULL,
                product_id INTEGER NOT NULL,
                quantity INTEGER NOT NULL,
                unit_price DECIMAL(10,2) NOT NULL,
                FOREIGN KEY (order_id) REFERENCES orders (id)
            )
        ''')
        
        # Indexes for performance
        cursor.execute('CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id)')
        cursor.execute('CREATE INDEX IF NOT EXISTS idx_order_items_order_id ON order_items(order_id)')
        
        self.conn.commit()
    
    def create_user(self, email: str, name: str) -> int:
        """Create a new user"""
        cursor = self.conn.cursor()
        cursor.execute(
            'INSERT INTO users (email, name) VALUES (?, ?)',
            (email, name)
        )
        self.conn.commit()
        return cursor.lastrowid
    
    def create_order_with_items(self, user_id: int, items: List[Dict[str, Any]]) -> int:
        """Create order with items using transaction (ACID)"""
        cursor = self.conn.cursor()
        
        try:
            # Start transaction
            cursor.execute('BEGIN TRANSACTION')
            
            # Calculate total amount
            total_amount = sum(item['quantity'] * item['unit_price'] for item in items)
            
            # Create order
            cursor.execute(
                'INSERT INTO orders (user_id, total_amount) VALUES (?, ?)',
                (user_id, total_amount)
            )
            order_id = cursor.lastrowid
            
            # Create order items
            for item in items:
                cursor.execute(
                    'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (?, ?, ?, ?)',
                    (order_id, item['product_id'], item['quantity'], item['unit_price'])
                )
            
            # Commit transaction
            cursor.execute('COMMIT')
            return order_id
            
        except Exception as e:
            # Rollback on error
            cursor.execute('ROLLBACK')
            raise e
    
    def get_user_orders(self, user_id: int) -> List[Dict[str, Any]]:
        """Get user orders with items (JOIN operation)"""
        cursor = self.conn.cursor()
        
        query = '''
            SELECT 
                o.id as order_id,
                o.total_amount,
                o.status,
                o.created_at,
                oi.product_id,
                oi.quantity,
                oi.unit_price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.user_id = ?
            ORDER BY o.created_at DESC, oi.id
        '''
        
        cursor.execute(query, (user_id,))
        rows = cursor.fetchall()
        
        # Group results by order
        orders = {}
        for row in rows:
            order_id = row['order_id']
            if order_id not in orders:
                orders[order_id] = {
                    'id': order_id,
                    'total_amount': row['total_amount'],
                    'status': row['status'],
                    'created_at': row['created_at'],
                    'items': []
                }
            
            if row['product_id']:  # Has items
                orders[order_id]['items'].append({
                    'product_id': row['product_id'],
                    'quantity': row['quantity'],
                    'unit_price': row['unit_price']
                })
        
        return list(orders.values())
    
    def get_order_analytics(self) -> Dict[str, Any]:
        """Complex analytical query"""
        cursor = self.conn.cursor()
        
        query = '''
            SELECT 
                COUNT(DISTINCT o.id) as total_orders,
                COUNT(DISTINCT o.user_id) as unique_customers,
                AVG(o.total_amount) as avg_order_value,
                SUM(o.total_amount) as total_revenue,
                MAX(o.total_amount) as max_order_value,
                MIN(o.total_amount) as min_order_value
            FROM orders o
            WHERE o.status = 'completed'
        '''
        
        cursor.execute(query)
        result = cursor.fetchone()
        
        return {
            'total_orders': result['total_orders'],
            'unique_customers': result['unique_customers'],
            'avg_order_value': float(result['avg_order_value'] or 0),
            'total_revenue': float(result['total_revenue'] or 0),
            'max_order_value': float(result['max_order_value'] or 0),
            'min_order_value': float(result['min_order_value'] or 0)
        }

# Usage example
def demonstrate_rdbms():
    db = RelationalDatabaseExample()
    
    # Create users
    user1_id = db.create_user("john@example.com", "John Doe")
    user2_id = db.create_user("jane@example.com", "Jane Smith")
    
    # Create orders with items
    order1_items = [
        {'product_id': 1, 'quantity': 2, 'unit_price': 29.99},
        {'product_id': 2, 'quantity': 1, 'unit_price': 49.99}
    ]
    
    order1_id = db.create_order_with_items(user1_id, order1_items)
    
    # Get user orders
    orders = db.get_user_orders(user1_id)
    print(f"User {user1_id} orders: {orders}")
    
    # Get analytics
    analytics = db.get_order_analytics()
    print(f"Order analytics: {analytics}")
```

### RDBMS Strengths and Weaknesses

**Strengths:**
- **ACID Compliance**: Strong consistency guarantees
- **Complex Queries**: Rich SQL query capabilities
- **Data Integrity**: Foreign keys, constraints, triggers
- **Mature Ecosystem**: Well-established tools and practices
- **Standardization**: SQL is widely known

**Weaknesses:**
- **Vertical Scaling**: Limited horizontal scaling options
- **Schema Rigidity**: Difficult to change schema
- **Performance**: Can be slower for simple operations
- **Impedance Mismatch**: Object-relational mapping complexity

## NoSQL Databases

### Document Databases

```python
import json
from typing import Dict, Any, List, Optional
from datetime import datetime
import uuid

class DocumentDatabase:
    """Simplified document database implementation"""
    
    def __init__(self):
        self.collections = {}  # collection_name -> documents
        self.indexes = {}      # collection_name -> field_name -> value -> document_ids
    
    def create_collection(self, collection_name: str):
        """Create a new collection"""
        if collection_name not in self.collections:
            self.collections[collection_name] = {}
            self.indexes[collection_name] = {}
    
    def insert_document(self, collection_name: str, document: Dict[str, Any]) -> str:
        """Insert a document into collection"""
        self.create_collection(collection_name)
        
        # Generate document ID if not provided
        doc_id = document.get('_id', str(uuid.uuid4()))
        document['_id'] = doc_id
        document['_created_at'] = datetime.now().isoformat()
        
        # Store document
        self.collections[collection_name][doc_id] = document.copy()
        
        # Update indexes
        self._update_indexes(collection_name, doc_id, document)
        
        return doc_id
    
    def find_document(self, collection_name: str, doc_id: str) -> Optional[Dict[str, Any]]:
        """Find document by ID"""
        if collection_name in self.collections:
            return self.collections[collection_name].get(doc_id)
        return None
    
    def find_documents(self, collection_name: str, query: Dict[str, Any]) -> List[Dict[str, Any]]:
        """Find documents matching query"""
        if collection_name not in self.collections:
            return []
        
        results = []
        
        # Simple query implementation
        for doc_id, document in self.collections[collection_name].items():
            if self._matches_query(document, query):
                results.append(document.copy())
        
        return results
    
    def update_document(self, collection_name: str, doc_id: str, 
                       update: Dict[str, Any]) -> bool:
        """Update document"""
        if collection_name in self.collections and doc_id in self.collections[collection_name]:
            document = self.collections[collection_name][doc_id]
            
            # Remove old indexes
            self._remove_from_indexes(collection_name, doc_id, document)
            
            # Apply update
            if '$set' in update:
                document.update(update['$set'])
            elif '$inc' in update:
                for field, value in update['$inc'].items():
                    document[field] = document.get(field, 0) + value
            else:
                document.update(update)
            
            document['_updated_at'] = datetime.now().isoformat()
            
            # Update indexes
            self._update_indexes(collection_name, doc_id, document)
            
            return True
        
        return False
    
    def delete_document(self, collection_name: str, doc_id: str) -> bool:
        """Delete document"""
        if collection_name in self.collections and doc_id in self.collections[collection_name]:
            document = self.collections[collection_name][doc_id]
            
            # Remove from indexes
            self._remove_from_indexes(collection_name, doc_id, document)
            
            # Delete document
            del self.collections[collection_name][doc_id]
            
            return True
        
        return False
    
    def create_index(self, collection_name: str, field_name: str):
        """Create index on field"""
        self.create_collection(collection_name)
        
        if field_name not in self.indexes[collection_name]:
            self.indexes[collection_name][field_name] = {}
            
            # Build index for existing documents
            for doc_id, document in self.collections[collection_name].items():
                if field_name in document:
                    value = document[field_name]
                    if value not in self.indexes[collection_name][field_name]:
                        self.indexes[collection_name][field_name][value] = set()
                    self.indexes[collection_name][field_name][value].add(doc_id)
    
    def _matches_query(self, document: Dict[str, Any], query: Dict[str, Any]) -> bool:
        """Check if document matches query"""
        for field, expected_value in query.items():
            if field not in document:
                return False
            
            actual_value = document[field]
            
            if isinstance(expected_value, dict):
                # Handle operators like $gt, $lt, etc.
                for operator, value in expected_value.items():
                    if operator == '$gt' and not (actual_value > value):
                        return False
                    elif operator == '$lt' and not (actual_value < value):
                        return False
                    elif operator == '$gte' and not (actual_value >= value):
                        return False
                    elif operator == '$lte' and not (actual_value <= value):
                        return False
                    elif operator == '$ne' and not (actual_value != value):
                        return False
                    elif operator == '$in' and actual_value not in value:
                        return False
            else:
                if actual_value != expected_value:
                    return False
        
        return True
    
    def _update_indexes(self, collection_name: str, doc_id: str, document: Dict[str, Any]):
        """Update indexes for document"""
        if collection_name in self.indexes:
            for field_name, field_index in self.indexes[collection_name].items():
                if field_name in document:
                    value = document[field_name]
                    if value not in field_index:
                        field_index[value] = set()
                    field_index[value].add(doc_id)
    
    def _remove_from_indexes(self, collection_name: str, doc_id: str, document: Dict[str, Any]):
        """Remove document from indexes"""
        if collection_name in self.indexes:
            for field_name, field_index in self.indexes[collection_name].items():
                if field_name in document:
                    value = document[field_name]
                    if value in field_index:
                        field_index[value].discard(doc_id)
                        if not field_index[value]:
                            del field_index[value]

# Usage example
def demonstrate_document_db():
    db = DocumentDatabase()
    
    # Insert user documents
    user1_id = db.insert_document('users', {
        'name': 'John Doe',
        'email': 'john@example.com',
        'age': 30,
        'preferences': {
            'theme': 'dark',
            'notifications': True
        },
        'tags': ['premium', 'active']
    })
    
    user2_id = db.insert_document('users', {
        'name': 'Jane Smith',
        'email': 'jane@example.com',
        'age': 25,
        'preferences': {
            'theme': 'light',
            'notifications': False
        },
        'tags': ['new', 'trial']
    })
    
    # Create index on age
    db.create_index('users', 'age')
    
    # Query documents
    young_users = db.find_documents('users', {'age': {'$lt': 28}})
    print(f"Young users: {young_users}")
    
    # Update document
    db.update_document('users', user1_id, {
        '$set': {'last_login': datetime.now().isoformat()},
        '$inc': {'login_count': 1}
    })
    
    # Find updated document
    updated_user = db.find_document('users', user1_id)
    print(f"Updated user: {updated_user}")
```

### Key-Value Databases

```python
import time
import threading
from typing import Any, Optional, Dict
from datetime import datetime, timedelta

class KeyValueStore:
    """In-memory key-value store with TTL and persistence"""
    
    def __init__(self):
        self.data = {}  # key -> {'value': value, 'expires_at': timestamp}
        self.lock = threading.RLock()
        
        # Start cleanup thread
        self.cleanup_thread = threading.Thread(target=self._cleanup_expired, daemon=True)
        self.cleanup_thread.start()
    
    def set(self, key: str, value: Any, ttl: Optional[int] = None) -> bool:
        """Set key-value pair with optional TTL (seconds)"""
        with self.lock:
            expires_at = None
            if ttl:
                expires_at = time.time() + ttl
            
            self.data[key] = {
                'value': value,
                'expires_at': expires_at,
                'created_at': time.time()
            }
            
            return True
    
    def get(self, key: str) -> Optional[Any]:
        """Get value by key"""
        with self.lock:
            if key not in self.data:
                return None
            
            entry = self.data[key]
            
            # Check if expired
            if entry['expires_at'] and time.time() > entry['expires_at']:
                del self.data[key]
                return None
            
            return entry['value']
    
    def delete(self, key: str) -> bool:
        """Delete key"""
        with self.lock:
            if key in self.data:
                del self.data[key]
                return True
            return False
    
    def exists(self, key: str) -> bool:
        """Check if key exists"""
        return self.get(key) is not None
    
    def increment(self, key: str, amount: int = 1) -> int:
        """Increment numeric value"""
        with self.lock:
            current_value = self.get(key)
            if current_value is None:
                current_value = 0
            elif not isinstance(current_value, (int, float)):
                raise ValueError("Value is not numeric")
            
            new_value = current_value + amount
            self.set(key, new_value)
            return new_value
    
    def expire(self, key: str, ttl: int) -> bool:
        """Set TTL for existing key"""
        with self.lock:
            if key not in self.data:
                return False
            
            self.data[key]['expires_at'] = time.time() + ttl
            return True
    
    def ttl(self, key: str) -> Optional[int]:
        """Get remaining TTL for key"""
        with self.lock:
            if key not in self.data:
                return None
            
            entry = self.data[key]
            if entry['expires_at'] is None:
                return -1  # No expiration
            
            remaining = entry['expires_at'] - time.time()
            return max(0, int(remaining))
    
    def keys(self, pattern: str = "*") -> list:
        """Get all keys matching pattern"""
        with self.lock:
            if pattern == "*":
                return list(self.data.keys())
            
            # Simple pattern matching (just prefix for now)
            if pattern.endswith("*"):
                prefix = pattern[:-1]
                return [key for key in self.data.keys() if key.startswith(prefix)]
            
            return [key for key in self.data.keys() if key == pattern]
    
    def flush_all(self):
        """Clear all data"""
        with self.lock:
            self.data.clear()
    
    def info(self) -> Dict[str, Any]:
        """Get database info"""
        with self.lock:
            total_keys = len(self.data)
            expired_keys = sum(1 for entry in self.data.values() 
                             if entry['expires_at'] and time.time() > entry['expires_at'])
            
            return {
                'total_keys': total_keys,
                'expired_keys': expired_keys,
                'memory_usage': sum(len(str(entry)) for entry in self.data.values())
            }
    
    def _cleanup_expired(self):
        """Background thread to clean up expired keys"""
        while True:
            time.sleep(60)  # Run every minute
            
            with self.lock:
                current_time = time.time()
                expired_keys = [
                    key for key, entry in self.data.items()
                    if entry['expires_at'] and current_time > entry['expires_at']
                ]
                
                for key in expired_keys:
                    del self.data[key]
                
                if expired_keys:
                    print(f"Cleaned up {len(expired_keys)} expired keys")

# Redis-like operations
class RedisLikeOperations:
    """Redis-like operations on key-value store"""
    
    def __init__(self, kv_store: KeyValueStore):
        self.kv = kv_store
    
    def lpush(self, key: str, *values) -> int:
        """Push values to left of list"""
        current_list = self.kv.get(key) or []
        if not isinstance(current_list, list):
            raise ValueError("Key does not contain a list")
        
        for value in reversed(values):
            current_list.insert(0, value)
        
        self.kv.set(key, current_list)
        return len(current_list)
    
    def rpush(self, key: str, *values) -> int:
        """Push values to right of list"""
        current_list = self.kv.get(key) or []
        if not isinstance(current_list, list):
            raise ValueError("Key does not contain a list")
        
        current_list.extend(values)
        self.kv.set(key, current_list)
        return len(current_list)
    
    def lpop(self, key: str) -> Optional[Any]:
        """Pop value from left of list"""
        current_list = self.kv.get(key)
        if not current_list or not isinstance(current_list, list):
            return None
        
        if current_list:
            value = current_list.pop(0)
            self.kv.set(key, current_list)
            return value
        
        return None
    
    def lrange(self, key: str, start: int, stop: int) -> list:
        """Get range of list elements"""
        current_list = self.kv.get(key)
        if not current_list or not isinstance(current_list, list):
            return []
        
        return current_list[start:stop+1]
    
    def sadd(self, key: str, *members) -> int:
        """Add members to set"""
        current_set = self.kv.get(key) or set()
        if not isinstance(current_set, set):
            raise ValueError("Key does not contain a set")
        
        added = 0
        for member in members:
            if member not in current_set:
                current_set.add(member)
                added += 1
        
        self.kv.set(key, current_set)
        return added
    
    def srem(self, key: str, *members) -> int:
        """Remove members from set"""
        current_set = self.kv.get(key)
        if not current_set or not isinstance(current_set, set):
            return 0
        
        removed = 0
        for member in members:
            if member in current_set:
                current_set.remove(member)
                removed += 1
        
        self.kv.set(key, current_set)
        return removed
    
    def smembers(self, key: str) -> set:
        """Get all set members"""
        current_set = self.kv.get(key)
        if not current_set or not isinstance(current_set, set):
            return set()
        
        return current_set.copy()
    
    def hset(self, key: str, field: str, value: Any) -> bool:
        """Set hash field"""
        current_hash = self.kv.get(key) or {}
        if not isinstance(current_hash, dict):
            raise ValueError("Key does not contain a hash")
        
        is_new_field = field not in current_hash
        current_hash[field] = value
        self.kv.set(key, current_hash)
        
        return is_new_field
    
    def hget(self, key: str, field: str) -> Optional[Any]:
        """Get hash field value"""
        current_hash = self.kv.get(key)
        if not current_hash or not isinstance(current_hash, dict):
            return None
        
        return current_hash.get(field)
    
    def hgetall(self, key: str) -> dict:
        """Get all hash fields and values"""
        current_hash = self.kv.get(key)
        if not current_hash or not isinstance(current_hash, dict):
            return {}
        
        return current_hash.copy()

# Usage example
def demonstrate_key_value_db():
    kv = KeyValueStore()
    redis_ops = RedisLikeOperations(kv)
    
    # Basic operations
    kv.set("user:1:name", "John Doe")
    kv.set("user:1:email", "john@example.com")
    kv.set("session:abc123", {"user_id": 1, "expires": "2024-01-01"}, ttl=3600)
    
    print(f"User name: {kv.get('user:1:name')}")
    print(f"Session TTL: {kv.ttl('session:abc123')} seconds")
    
    # List operations
    redis_ops.lpush("recent_orders", "order_1", "order_2")
    redis_ops.rpush("recent_orders", "order_3")
    
    print(f"Recent orders: {redis_ops.lrange('recent_orders', 0, -1)}")
    
    # Set operations
    redis_ops.sadd("user_tags", "premium", "active", "verified")
    print(f"User tags: {redis_ops.smembers('user_tags')}")
    
    # Hash operations
    redis_ops.hset("user:1", "name", "John Doe")
    redis_ops.hset("user:1", "email", "john@example.com")
    print(f"User hash: {redis_ops.hgetall('user:1')}")
```

### Column-Family Databases

```python
from typing import Dict, Any, List, Optional
from collections import defaultdict
import time

class ColumnFamily:
    """Represents a column family in a column-family database"""
    
    def __init__(self, name: str):
        self.name = name
        self.rows = {}  # row_key -> {column_name: {timestamp: value}}
        self.column_metadata = {}  # column_name -> metadata
    
    def put(self, row_key: str, column_name: str, value: Any, timestamp: int = None):
        """Put a value in a specific row and column"""
        if timestamp is None:
            timestamp = int(time.time() * 1000)  # milliseconds
        
        if row_key not in self.rows:
            self.rows[row_key] = {}
        
        if column_name not in self.rows[row_key]:
            self.rows[row_key][column_name] = {}
        
        self.rows[row_key][column_name][timestamp] = value
    
    def get(self, row_key: str, column_name: str = None, 
            max_versions: int = 1) -> Optional[Any]:
        """Get value(s) from row and column"""
        if row_key not in self.rows:
            return None
        
        row = self.rows[row_key]
        
        if column_name:
            if column_name not in row:
                return None
            
            # Get latest versions
            column_data = row[column_name]
            sorted_timestamps = sorted(column_data.keys(), reverse=True)
            
            if max_versions == 1:
                return column_data[sorted_timestamps[0]]
            else:
                return [column_data[ts] for ts in sorted_timestamps[:max_versions]]
        else:
            # Get all columns for row
            result = {}
            for col_name, col_data in row.items():
                latest_timestamp = max(col_data.keys())
                result[col_name] = col_data[latest_timestamp]
            return result
    
    def delete(self, row_key: str, column_name: str = None, timestamp: int = None):
        """Delete row, column, or specific version"""
        if row_key not in self.rows:
            return False
        
        if column_name is None:
            # Delete entire row
            del self.rows[row_key]
            return True
        
        if column_name not in self.rows[row_key]:
            return False
        
        if timestamp is None:
            # Delete entire column
            del self.rows[row_key][column_name]
        else:
            # Delete specific version
            if timestamp in self.rows[row_key][column_name]:
                del self.rows[row_key][column_name][timestamp]
        
        return True
    
    def scan(self, start_row: str = None, end_row: str = None, 
             column_filter: List[str] = None) -> List[Dict[str, Any]]:
        """Scan range of rows"""
        results = []
        
        for row_key in sorted(self.rows.keys()):
            # Apply row key filters
            if start_row and row_key < start_row:
                continue
            if end_row and row_key > end_row:
                break
            
            row_data = {'row_key': row_key}
            
            for col_name, col_data in self.rows[row_key].items():
                # Apply column filter
                if column_filter and col_name not in column_filter:
                    continue
                
                # Get latest value
                latest_timestamp = max(col_data.keys())
                row_data[col_name] = col_data[latest_timestamp]
            
            results.append(row_data)
        
        return results

class ColumnFamilyDatabase:
    """Simplified column-family database like Cassandra/HBase"""
    
    def __init__(self):
        self.column_families = {}
        self.keyspace = "default"
    
    def create_column_family(self, cf_name: str):
        """Create a new column family"""
        if cf_name not in self.column_families:
            self.column_families[cf_name] = ColumnFamily(cf_name)
    
    def get_column_family(self, cf_name: str) -> Optional[ColumnFamily]:
        """Get column family by name"""
        return self.column_families.get(cf_name)
    
    def put(self, cf_name: str, row_key: str, columns: Dict[str, Any]):
        """Put multiple columns in a row"""
        if cf_name not in self.column_families:
            self.create_column_family(cf_name)
        
        cf = self.column_families[cf_name]
        timestamp = int(time.time() * 1000)
        
        for column_name, value in columns.items():
            cf.put(row_key, column_name, value, timestamp)
    
    def get(self, cf_name: str, row_key: str, columns: List[str] = None) -> Optional[Dict[str, Any]]:
        """Get row from column family"""
        if cf_name not in self.column_families:
            return None
        
        cf = self.column_families[cf_name]
        
        if columns:
            result = {}
            for column in columns:
                value = cf.get(row_key, column)
                if value is not None:
                    result[column] = value
            return result if result else None
        else:
            return cf.get(row_key)

# Time-series data example using column-family
class TimeSeriesStore:
    """Time-series data store using column-family database"""
    
    def __init__(self):
        self.db = ColumnFamilyDatabase()
        self.db.create_column_family("metrics")
        self.db.create_column_family("events")
    
    def record_metric(self, metric_name: str, value: float, 
                     timestamp: int = None, tags: Dict[str, str] = None):
        """Record a metric value"""
        if timestamp is None:
            timestamp = int(time.time())
        
        # Create row key: metric_name:YYYYMMDD
        date_str = time.strftime("%Y%m%d", time.gmtime(timestamp))
        row_key = f"{metric_name}:{date_str}"
        
        # Column name includes timestamp and tags
        tag_str = ""
        if tags:
            tag_str = ":" + ":".join(f"{k}={v}" for k, v in sorted(tags.items()))
        
        column_name = f"{timestamp}{tag_str}"
        
        self.db.put("metrics", row_key, {column_name: value})
    
    def get_metrics(self, metric_name: str, start_time: int, end_time: int) -> List[Dict[str, Any]]:
        """Get metric values for time range"""
        results = []
        
        # Generate date range
        current_time = start_time
        while current_time <= end_time:
            date_str = time.strftime("%Y%m%d", time.gmtime(current_time))
            row_key = f"{metric_name}:{date_str}"
            
            cf = self.db.get_column_family("metrics")
            if cf and row_key in cf.rows:
                row_data = cf.rows[row_key]
                
                for column_name, versions in row_data.items():
                    # Parse timestamp from column name
                    timestamp_str = column_name.split(':')[0]
                    timestamp = int(timestamp_str)
                    
                    if start_time <= timestamp <= end_time:
                        latest_version = max(versions.keys())
                        value = versions[latest_version]
                        
                        results.append({
                            'timestamp': timestamp,
                            'value': value,
                            'metric': metric_name
                        })
            
            current_time += 86400  # Next day
        
        return sorted(results, key=lambda x: x['timestamp'])

# Usage example
def demonstrate_column_family_db():
    # Basic column-family operations
    db = ColumnFamilyDatabase()
    
    # Store user data
    db.put("users", "user:1", {
        "name": "John Doe",
        "email": "john@example.com",
        "created_at": "2024-01-01",
        "last_login": "2024-01-15"
    })
    
    db.put("users", "user:2", {
        "name": "Jane Smith", 
        "email": "jane@example.com",
        "created_at": "2024-01-02"
    })
    
    # Get user data
    user1 = db.get("users", "user:1")
    print(f"User 1: {user1}")
    
    # Get specific columns
    user1_contact = db.get("users", "user:1", ["name", "email"])
    print(f"User 1 contact: {user1_contact}")
    
    # Time-series example
    ts_store = TimeSeriesStore()
    
    # Record some metrics
    base_time = int(time.time()) - 3600  # 1 hour ago
    for i in range(60):  # 60 data points
        ts_store.record_metric(
            "cpu_usage", 
            50 + (i % 20), 
            base_time + i * 60,  # Every minute
            {"host": "server1", "region": "us-east"}
        )
    
    # Query metrics
    metrics = ts_store.get_metrics("cpu_usage", base_time, int(time.time()))
    print(f"Retrieved {len(metrics)} metric points")
```

## NewSQL Databases

### Characteristics and Examples

```python
from typing import Dict, Any, List
import threading
import time

class NewSQLDatabase:
    """Simplified NewSQL database with ACID + horizontal scaling"""
    
    def __init__(self, node_id: str, cluster_nodes: List[str]):
        self.node_id = node_id
        self.cluster_nodes = cluster_nodes
        self.local_data = {}  # Shard data stored on this node
        self.transaction_log = []
        self.lock = threading.RLock()
        
        # Consistent hashing for data distribution
        self.hash_ring = self._build_hash_ring()
    
    def _build_hash_ring(self) -> Dict[int, str]:
        """Build consistent hash ring for data distribution"""
        ring = {}
        for node in self.cluster_nodes:
            # Multiple virtual nodes per physical node
            for i in range(100):
                hash_key = hash(f"{node}:{i}") % (2**32)
                ring[hash_key] = node
        return ring
    
    def _get_node_for_key(self, key: str) -> str:
        """Determine which node should store this key"""
        key_hash = hash(key) % (2**32)
        
        # Find the first node in the ring >= key_hash
        for ring_position in sorted(self.hash_ring.keys()):
            if ring_position >= key_hash:
                return self.hash_ring[ring_position]
        
        # Wrap around to first node
        first_position = min(self.hash_ring.keys())
        return self.hash_ring[first_position]
    
    def begin_transaction(self) -> str:
        """Begin a distributed transaction"""
        transaction_id = f"txn_{self.node_id}_{int(time.time() * 1000)}"
        return transaction_id
    
    def put(self, transaction_id: str, key: str, value: Any) -> bool:
        """Put key-value in transaction"""
        target_node = self._get_node_for_key(key)
        
        if target_node == self.node_id:
            # Local operation
            with self.lock:
                if transaction_id not in self.local_data:
                    self.local_data[transaction_id] = {}
                self.local_data[transaction_id][key] = value
                return True
        else:
            # Remote operation (would be network call in real implementation)
            return self._remote_put(target_node, transaction_id, key, value)
    
    def get(self, key: str) -> Any:
        """Get value by key (reads from committed data)"""
        target_node = self._get_node_for_key(key)
        
        if target_node == self.node_id:
            # Local read
            with self.lock:
                # Look in committed data (simplified)
                for txn_data in self.local_data.values():
                    if key in txn_data:
                        return txn_data[key]
                return None
        else:
            # Remote read
            return self._remote_get(target_node, key)
    
    def commit_transaction(self, transaction_id: str) -> bool:
        """Commit transaction using 2-phase commit"""
        try:
            # Phase 1: Prepare
            if not self._prepare_transaction(transaction_id):
                return False
            
            # Phase 2: Commit
            return self._commit_transaction(transaction_id)
            
        except Exception as e:
            # Abort transaction
            self._abort_transaction(transaction_id)
            return False
    
    def _prepare_transaction(self, transaction_id: str) -> bool:
        """Prepare phase of 2PC"""
        # In real implementation, would coordinate with all involved nodes
        with self.lock:
            if transaction_id in self.local_data:
                # Validate transaction (check constraints, etc.)
                return True
            return False
    
    def _commit_transaction(self, transaction_id: str) -> bool:
        """Commit phase of 2PC"""
        with self.lock:
            if transaction_id in self.local_data:
                # Move data from transaction buffer to committed storage
                # (simplified - in reality would write to persistent storage)
                self.transaction_log.append({
                    'transaction_id': transaction_id,
                    'operation': 'commit',
                    'timestamp': time.time(),
                    'data': self.local_data[transaction_id]
                })
                return True
            return False
    
    def _abort_transaction(self, transaction_id: str):
        """Abort transaction"""
        with self.lock:
            if transaction_id in self.local_data:
                del self.local_data[transaction_id]
    
    def _remote_put(self, target_node: str, transaction_id: str, key: str, value: Any) -> bool:
        """Simulate remote put operation"""
        # In real implementation, would make network call
        print(f"Remote PUT to {target_node}: {key} = {value}")
        return True
    
    def _remote_get(self, target_node: str, key: str) -> Any:
        """Simulate remote get operation"""
        # In real implementation, would make network call
        print(f"Remote GET from {target_node}: {key}")
        return None

# Usage example
def demonstrate_newsql():
    # Create cluster of NewSQL nodes
    cluster_nodes = ["node1", "node2", "node3"]
    node1 = NewSQLDatabase("node1", cluster_nodes)
    
    # Distributed transaction
    txn_id = node1.begin_transaction()
    
    # Put data (will be distributed across nodes)
    node1.put(txn_id, "user:1:name", "John Doe")
    node1.put(txn_id, "user:1:email", "john@example.com")
    node1.put(txn_id, "user:2:name", "Jane Smith")
    
    # Commit transaction
    success = node1.commit_transaction(txn_id)
    print(f"Transaction committed: {success}")
    
    # Read data
    name = node1.get("user:1:name")
    print(f"Retrieved name: {name}")
```

## Specialized Databases

### Graph Databases

```python
from typing import Dict, Any, List, Set, Optional
from collections import defaultdict, deque

class GraphNode:
    def __init__(self, node_id: str, properties: Dict[str, Any] = None):
        self.id = node_id
        self.properties = properties or {}
        self.labels = set()
    
    def add_label(self, label: str):
        self.labels.add(label)
    
    def has_label(self, label: str) -> bool:
        return label in self.labels

class GraphEdge:
    def __init__(self, edge_id: str, from_node: str, to_node: str, 
                 relationship_type: str, properties: Dict[str, Any] = None):
        self.id = edge_id
        self.from_node = from_node
        self.to_node = to_node
        self.relationship_type = relationship_type
        self.properties = properties or {}

class GraphDatabase:
    """Simplified graph database implementation"""
    
    def __init__(self):
        self.nodes = {}  # node_id -> GraphNode
        self.edges = {}  # edge_id -> GraphEdge
        self.outgoing_edges = defaultdict(list)  # node_id -> [edge_ids]
        self.incoming_edges = defaultdict(list)  # node_id -> [edge_ids]
        self.node_labels = defaultdict(set)  # label -> {node_ids}
        self.relationship_types = defaultdict(list)  # type -> [edge_ids]
    
    def create_node(self, node_id: str, properties: Dict[str, Any] = None, 
                   labels: List[str] = None) -> GraphNode:
        """Create a new node"""
        node = GraphNode(node_id, properties)
        
        if labels:
            for label in labels:
                node.add_label(label)
                self.node_labels[label].add(node_id)
        
        self.nodes[node_id] = node
        return node
    
    def create_edge(self, edge_id: str, from_node: str, to_node: str,
                   relationship_type: str, properties: Dict[str, Any] = None) -> GraphEdge:
        """Create a new edge"""
        if from_node not in self.nodes or to_node not in self.nodes:
            raise ValueError("Both nodes must exist before creating edge")
        
        edge = GraphEdge(edge_id, from_node, to_node, relationship_type, properties)
        
        self.edges[edge_id] = edge
        self.outgoing_edges[from_node].append(edge_id)
        self.incoming_edges[to_node].append(edge_id)
        self.relationship_types[relationship_type].append(edge_id)
        
        return edge
    
    def get_node(self, node_id: str) -> Optional[GraphNode]:
        """Get node by ID"""
        return self.nodes.get(node_id)
    
    def get_nodes_by_label(self, label: str) -> List[GraphNode]:
        """Get all nodes with specific label"""
        node_ids = self.node_labels.get(label, set())
        return [self.nodes[node_id] for node_id in node_ids]
    
    def get_neighbors(self, node_id: str, direction: str = "both", 
                     relationship_type: str = None) -> List[GraphNode]:
        """Get neighboring nodes"""
        neighbors = []
        
        if direction in ["out", "both"]:
            for edge_id in self.outgoing_edges[node_id]:
                edge = self.edges[edge_id]
                if not relationship_type or edge.relationship_type == relationship_type:
                    neighbors.append(self.nodes[edge.to_node])
        
        if direction in ["in", "both"]:
            for edge_id in self.incoming_edges[node_id]:
                edge = self.edges[edge_id]
                if not relationship_type or edge.relationship_type == relationship_type:
                    neighbors.append(self.nodes[edge.from_node])
        
        return neighbors
    
    def shortest_path(self, start_node: str, end_node: str) -> Optional[List[str]]:
        """Find shortest path between two nodes using BFS"""
        if start_node not in self.nodes or end_node not in self.nodes:
            return None
        
        if start_node == end_node:
            return [start_node]
        
        queue = deque([(start_node, [start_node])])
        visited = {start_node}
        
        while queue:
            current_node, path = queue.popleft()
            
            # Get all neighbors
            neighbors = self.get_neighbors(current_node)
            
            for neighbor in neighbors:
                if neighbor.id not in visited:
                    new_path = path + [neighbor.id]
                    
                    if neighbor.id == end_node:
                        return new_path
                    
                    visited.add(neighbor.id)
                    queue.append((neighbor.id, new_path))
        
        return None  # No path found
    
    def find_paths(self, start_node: str, end_node: str, max_depth: int = 5) -> List[List[str]]:
        """Find all paths between two nodes up to max_depth"""
        if start_node not in self.nodes or end_node not in self.nodes:
            return []
        
        paths = []
        
        def dfs(current_node: str, target_node: str, path: List[str], visited: Set[str], depth: int):
            if depth > max_depth:
                return
            
            if current_node == target_node:
                paths.append(path.copy())
                return
            
            neighbors = self.get_neighbors(current_node)
            for neighbor in neighbors:
                if neighbor.id not in visited:
                    visited.add(neighbor.id)
                    path.append(neighbor.id)
                    dfs(neighbor.id, target_node, path, visited, depth + 1)
                    path.pop()
                    visited.remove(neighbor.id)
        
        dfs(start_node, end_node, [start_node], {start_node}, 0)
        return paths
    
    def cypher_like_query(self, pattern: str) -> List[Dict[str, Any]]:
        """Simplified Cypher-like query language"""
        # Very basic pattern matching - in reality would need full parser
        # Example: "MATCH (u:User)-[:FOLLOWS]->(f:User) RETURN u.name, f.name"
        
        results = []
        
        if "User)-[:FOLLOWS]->" in pattern:
            # Find all FOLLOWS relationships between Users
            user_nodes = self.get_nodes_by_label("User")
            
            for user in user_nodes:
                followers = self.get_neighbors(user.id, "out", "FOLLOWS")
                for follower in followers:
                    if follower.has_label("User"):
                        results.append({
                            'u.name': user.properties.get('name'),
                            'f.name': follower.properties.get('name')
                        })
        
        return results

# Usage example
def demonstrate_graph_db():
    graph = GraphDatabase()
    
    # Create nodes
    alice = graph.create_node("alice", {"name": "Alice", "age": 30}, ["User"])
    bob = graph.create_node("bob", {"name": "Bob", "age": 25}, ["User"])
    charlie = graph.create_node("charlie", {"name": "Charlie", "age": 35}, ["User"])
    
    # Create relationships
    graph.create_edge("e1", "alice", "bob", "FOLLOWS")
    graph.create_edge("e2", "bob", "charlie", "FOLLOWS")
    graph.create_edge("e3", "alice", "charlie", "KNOWS")
    
    # Query relationships
    alice_follows = graph.get_neighbors("alice", "out", "FOLLOWS")
    print(f"Alice follows: {[n.properties['name'] for n in alice_follows]}")
    
    # Find shortest path
    path = graph.shortest_path("alice", "charlie")
    print(f"Shortest path from Alice to Charlie: {path}")
    
    # Find all paths
    all_paths = graph.find_paths("alice", "charlie", max_depth=3)
    print(f"All paths from Alice to Charlie: {all_paths}")
    
    # Cypher-like query
    results = graph.cypher_like_query("MATCH (u:User)-[:FOLLOWS]->(f:User) RETURN u.name, f.name")
    print(f"FOLLOWS relationships: {results}")
```

## Database Selection Framework

### Decision Matrix

```python
from typing import Dict, List, Tuple
from dataclasses import dataclass
from enum import Enum

class RequirementPriority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

@dataclass
class SystemRequirement:
    name: str
    priority: RequirementPriority
    description: str

class DatabaseSelectionFramework:
    """Framework for selecting appropriate database technology"""
    
    def __init__(self):
        self.requirements = []
        self.database_scores = {}
    
    def add_requirement(self, requirement: SystemRequirement):
        """Add a system requirement"""
        self.requirements.append(requirement)
    
    def evaluate_database(self, db_type: DatabaseType, scores: Dict[str, int]) -> float:
        """Evaluate how well a database meets requirements"""
        total_score = 0.0
        total_weight = 0.0
        
        for requirement in self.requirements:
            if requirement.name in scores:
                score = scores[requirement.name]  # 1-5 scale
                weight = requirement.priority.value
                
                total_score += score * weight
                total_weight += weight
        
        return total_score / total_weight if total_weight > 0 else 0.0
    
    def recommend_databases(self) -> List[Tuple[DatabaseType, float]]:
        """Get ranked list of database recommendations"""
        
        # Define how well each database type meets different requirements
        database_evaluations = {
            DatabaseType.RELATIONAL: {
                'consistency': 5,
                'complex_queries': 5,
                'transactions': 5,
                'scalability': 2,
                'flexibility': 2,
                'performance': 3,
                'availability': 3
            },
            DatabaseType.DOCUMENT: {
                'consistency': 3,
                'complex_queries': 3,
                'transactions': 3,
                'scalability': 4,
                'flexibility': 5,
                'performance': 4,
                'availability': 4
            },
            DatabaseType.KEY_VALUE: {
                'consistency': 3,
                'complex_queries': 1,
                'transactions': 2,
                'scalability': 5,
                'flexibility': 4,
                'performance': 5,
                'availability': 5
            },
            DatabaseType.COLUMN_FAMILY: {
                'consistency': 4,
                'complex_queries': 3,
                'transactions': 3,
                'scalability': 5,
                'flexibility': 4,
                'performance': 4,
                'availability': 4
            },
            DatabaseType.GRAPH: {
                'consistency': 4,
                'complex_queries': 5,
                'transactions': 4,
                'scalability': 3,
                'flexibility': 3,
                'performance': 3,
                'availability': 3
            }
        }
        
        recommendations = []
        
        for db_type, scores in database_evaluations.items():
            score = self.evaluate_database(db_type, scores)
            recommendations.append((db_type, score))
        
        # Sort by score descending
        recommendations.sort(key=lambda x: x[1], reverse=True)
        
        return recommendations
    
    def generate_report(self) -> Dict[str, Any]:
        """Generate detailed recommendation report"""
        recommendations = self.recommend_databases()
        
        report = {
            'requirements': [
                {
                    'name': req.name,
                    'priority': req.priority.name,
                    'description': req.description
                }
                for req in self.requirements
            ],
            'recommendations': [
                {
                    'database_type': db_type.value,
                    'score': round(score, 2),
                    'rank': idx + 1
                }
                for idx, (db_type, score) in enumerate(recommendations)
            ],
            'top_choice': {
                'database_type': recommendations[0][0].value,
                'score': round(recommendations[0][1], 2),
                'reasoning': self._generate_reasoning(recommendations[0][0])
            }
        }
        
        return report
    
    def _generate_reasoning(self, db_type: DatabaseType) -> str:
        """Generate reasoning for database choice"""
        reasoning_map = {
            DatabaseType.RELATIONAL: "Strong consistency, ACID transactions, and complex query support make this ideal for financial and transactional systems.",
            DatabaseType.DOCUMENT: "Flexible schema and good scalability make this suitable for content management and rapidly evolving applications.",
            DatabaseType.KEY_VALUE: "Excellent performance and scalability make this perfect for caching, session storage, and simple data models.",
            DatabaseType.COLUMN_FAMILY: "Great for time-series data, analytics, and write-heavy workloads with predictable query patterns.",
            DatabaseType.GRAPH: "Optimal for relationship-heavy data and complex traversal queries in social networks and recommendation systems."
        }
        
        return reasoning_map.get(db_type, "General-purpose database suitable for various use cases.")

# Usage example
def demonstrate_database_selection():
    framework = DatabaseSelectionFramework()
    
    # Define system requirements
    framework.add_requirement(SystemRequirement(
        "consistency", 
        RequirementPriority.CRITICAL,
        "Strong consistency required for financial transactions"
    ))
    
    framework.add_requirement(SystemRequirement(
        "scalability",
        RequirementPriority.HIGH,
        "Must handle 10x growth in traffic"
    ))
    
    framework.add_requirement(SystemRequirement(
        "complex_queries",
        RequirementPriority.MEDIUM,
        "Need to support analytical queries"
    ))
    
    framework.add_requirement(SystemRequirement(
        "flexibility",
        RequirementPriority.LOW,
        "Schema changes should be manageable"
    ))
    
    # Generate recommendations
    report = framework.generate_report()
    
    print("Database Selection Report:")
    print(f"Top recommendation: {report['top_choice']['database_type']} (Score: {report['top_choice']['score']})")
    print(f"Reasoning: {report['top_choice']['reasoning']}")
    
    print("\nAll recommendations:")
    for rec in report['recommendations']:
        print(f"{rec['rank
