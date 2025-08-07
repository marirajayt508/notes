# Database Design - System Design Fundamentals

## Table of Contents
1. [Database Design Principles](#database-design-principles)
2. [SQL vs NoSQL](#sql-vs-nosql)
3. [Database Scaling Strategies](#database-scaling-strategies)
4. [ACID Properties](#acid-properties)
5. [CAP Theorem](#cap-theorem)
6. [Database Sharding](#database-sharding)
7. [Replication Strategies](#replication-strategies)
8. [Indexing Strategies](#indexing-strategies)
9. [Schema Design Patterns](#schema-design-patterns)
10. [Interview Questions](#interview-questions)

---

## Database Design Principles

### 1. Data Modeling Fundamentals

#### Entity-Relationship (ER) Modeling
```
Entity: User
Attributes: user_id, username, email, created_at

Entity: Post  
Attributes: post_id, user_id, content, created_at

Relationship: User (1) ←→ (Many) Post
```

#### Normalization Levels
```sql
-- 1NF: Atomic values, no repeating groups
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20)  -- Single phone number
);

-- 2NF: 1NF + No partial dependencies
CREATE TABLE orders (
    order_id INT,
    product_id INT,
    quantity INT,
    product_name VARCHAR(100),  -- Depends only on product_id
    PRIMARY KEY (order_id, product_id)
);

-- Better 2NF design
CREATE TABLE orders (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2)
);

-- 3NF: 2NF + No transitive dependencies
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(50),  -- Transitively dependent
    department_location VARCHAR(100)  -- Transitively dependent
);

-- Better 3NF design
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(50),
    location VARCHAR(100)
);
```

### 2. Denormalization for Performance
```sql
-- Normalized design (3NF)
SELECT u.username, COUNT(p.post_id) as post_count
FROM users u
LEFT JOIN posts p ON u.user_id = p.user_id
GROUP BY u.user_id, u.username;

-- Denormalized design (faster reads)
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    post_count INT DEFAULT 0,  -- Denormalized field
    last_post_date TIMESTAMP
);

-- Update trigger to maintain consistency
CREATE TRIGGER update_user_stats
AFTER INSERT ON posts
FOR EACH ROW
BEGIN
    UPDATE users 
    SET post_count = post_count + 1,
        last_post_date = NEW.created_at
    WHERE user_id = NEW.user_id;
END;
```

---

## SQL vs NoSQL

### SQL Databases (RDBMS)

#### Characteristics
- **ACID compliance**: Atomicity, Consistency, Isolation, Durability
- **Schema-based**: Fixed structure, strong typing
- **SQL queries**: Standardized query language
- **Joins**: Complex relationships between tables
- **Vertical scaling**: Scale up with better hardware

#### Popular SQL Databases
```
PostgreSQL: Advanced features, JSON support, extensible
MySQL: Fast, reliable, widely adopted
Oracle: Enterprise features, high performance
SQL Server: Microsoft ecosystem, business intelligence
SQLite: Embedded, serverless, lightweight
```

#### Use Cases
- **Financial systems**: Banking, accounting, transactions
- **E-commerce**: Order management, inventory
- **CRM systems**: Customer relationships, sales
- **Analytics**: Complex queries, reporting

#### Example Schema Design
```sql
-- E-commerce database schema
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_category_id INT REFERENCES categories(category_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category_id INT REFERENCES categories(category_id),
    stock_quantity INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL
);

-- Indexes for performance
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### NoSQL Databases

#### Document Databases
**Examples**: MongoDB, CouchDB, Amazon DocumentDB

```javascript
// MongoDB document example
{
  "_id": ObjectId("..."),
  "username": "john_doe",
  "email": "john@example.com",
  "profile": {
    "firstName": "John",
    "lastName": "Doe",
    "age": 30,
    "address": {
      "street": "123 Main St",
      "city": "New York",
      "zipCode": "10001"
    }
  },
  "posts": [
    {
      "title": "My First Post",
      "content": "Hello World!",
      "tags": ["intro", "hello"],
      "createdAt": ISODate("2023-01-01T00:00:00Z")
    }
  ],
  "createdAt": ISODate("2023-01-01T00:00:00Z")
}

// Query examples
db.users.find({"profile.age": {$gte: 18}})
db.users.find({"posts.tags": "intro"})
db.users.updateOne(
  {"username": "john_doe"},
  {$push: {"posts": newPost}}
)
```

**Use Cases:**
- Content management systems
- User profiles and preferences
- Product catalogs
- Real-time analytics

#### Key-Value Stores
**Examples**: Redis, Amazon DynamoDB, Riak

```python
# Redis examples
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Simple key-value operations
r.set('user:1001:name', 'John Doe')
r.set('user:1001:email', 'john@example.com')
r.expire('user:1001:session', 3600)  # TTL

# Hash operations
r.hset('user:1001', mapping={
    'name': 'John Doe',
    'email': 'john@example.com',
    'age': 30
})

# List operations (for timelines, queues)
r.lpush('user:1001:timeline', 'post:123')
r.lpush('user:1001:timeline', 'post:124')
timeline = r.lrange('user:1001:timeline', 0, 9)  # Get 10 recent posts

# Set operations (for tags, followers)
r.sadd('user:1001:followers', 'user:1002', 'user:1003')
r.sadd('user:1002:following', 'user:1001')
mutual_friends = r.sinter('user:1001:followers', 'user:1002:followers')
```

**Use Cases:**
- Session storage
- Caching layer
- Real-time recommendations
- Shopping carts

#### Column-Family
**Examples**: Cassandra, HBase, Amazon SimpleDB

```cql
-- Cassandra schema example
CREATE KEYSPACE social_media 
WITH REPLICATION = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};

CREATE TABLE user_timeline (
    user_id UUID,
    post_id TIMEUUID,
    content TEXT,
    author_id UUID,
    author_name TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

CREATE TABLE post_by_author (
    author_id UUID,
    post_id TIMEUUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (author_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- Queries
SELECT * FROM user_timeline 
WHERE user_id = ? 
LIMIT 20;

SELECT * FROM post_by_author 
WHERE author_id = ? 
AND post_id > ?
LIMIT 10;
```

**Use Cases:**
- Time-series data
- IoT sensor data
- Log aggregation
- Social media feeds

#### Graph Databases
**Examples**: Neo4j, Amazon Neptune, ArangoDB

```cypher
// Neo4j Cypher examples
// Create nodes
CREATE (john:User {name: 'John Doe', email: 'john@example.com'})
CREATE (jane:User {name: 'Jane Smith', email: 'jane@example.com'})
CREATE (company:Company {name: 'Tech Corp'})

// Create relationships
CREATE (john)-[:FOLLOWS]->(jane)
CREATE (john)-[:WORKS_AT]->(company)
CREATE (jane)-[:WORKS_AT]->(company)

// Find mutual connections
MATCH (user1:User)-[:FOLLOWS]->(mutual:User)<-[:FOLLOWS]-(user2:User)
WHERE user1.name = 'John Doe' AND user2.name = 'Jane Smith'
RETURN mutual.name

// Find shortest path
MATCH path = shortestPath((john:User)-[*]-(jane:User))
WHERE john.name = 'John Doe' AND jane.name = 'Jane Smith'
RETURN path

// Recommend friends (friends of friends)
MATCH (user:User)-[:FOLLOWS]->(friend)-[:FOLLOWS]->(recommendation)
WHERE user.name = 'John Doe' 
AND NOT (user)-[:FOLLOWS]->(recommendation)
AND user <> recommendation
RETURN recommendation.name, COUNT(*) as mutual_friends
ORDER BY mutual_friends DESC
LIMIT 5
```

**Use Cases:**
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs

### SQL vs NoSQL Comparison

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Schema** | Fixed, predefined | Flexible, dynamic |
| **Scalability** | Vertical (scale up) | Horizontal (scale out) |
| **ACID** | Full ACID compliance | Eventual consistency |
| **Queries** | Complex SQL queries | Simple queries, some complex |
| **Relationships** | Strong relationships (JOINs) | Weak relationships |
| **Consistency** | Strong consistency | Eventual consistency |
| **Use Cases** | Complex transactions | Large scale, flexible data |

---

## Database Scaling Strategies

### 1. Vertical Scaling (Scale Up)
```
Single Database Server
├── Increase CPU cores
├── Add more RAM
├── Upgrade to SSD storage
└── Optimize database configuration

Pros:
- Simple to implement
- No application changes
- Strong consistency

Cons:
- Hardware limits
- Single point of failure
- Expensive at scale
```

### 2. Horizontal Scaling (Scale Out)

#### Read Replicas
```
Master Database (Writes)
├── Read Replica 1 (Reads)
├── Read Replica 2 (Reads)
└── Read Replica 3 (Reads)

Application Logic:
- Route writes to master
- Route reads to replicas
- Handle replication lag
```

```python
# Database routing example
class DatabaseRouter:
    def __init__(self, master_db, replica_dbs):
        self.master_db = master_db
        self.replica_dbs = replica_dbs
        self.replica_index = 0
    
    def get_write_db(self):
        return self.master_db
    
    def get_read_db(self):
        # Round-robin selection of read replicas
        db = self.replica_dbs[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replica_dbs)
        return db
    
    def execute_write(self, query, params=None):
        return self.get_write_db().execute(query, params)
    
    def execute_read(self, query, params=None):
        return self.get_read_db().execute(query, params)

# Usage
router = DatabaseRouter(master_db, [replica1, replica2, replica3])

# Write operations
router.execute_write("INSERT INTO users (name, email) VALUES (?, ?)", 
                    ("John", "john@example.com"))

# Read operations
users = router.execute_read("SELECT * FROM users WHERE active = ?", (True,))
```

#### Database Sharding
```
Shard 1: Users A-H
Shard 2: Users I-P  
Shard 3: Users Q-Z

Shard Key: user_id or username
```

```python
# Sharding implementation
import hashlib

class DatabaseSharding:
    def __init__(self, shards):
        self.shards = shards  # List of database connections
        self.num_shards = len(shards)
    
    def get_shard_key(self, key):
        # Hash-based sharding
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards
    
    def get_shard(self, key):
        shard_index = self.get_shard_key(key)
        return self.shards[shard_index]
    
    def insert_user(self, user_id, user_data):
        shard = self.get_shard(user_id)
        return shard.execute(
            "INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)",
            (user_id, user_data['name'], user_data['email'])
        )
    
    def get_user(self, user_id):
        shard = self.get_shard(user_id)
        return shard.execute(
            "SELECT * FROM users WHERE user_id = ?",
            (user_id,)
        ).fetchone()
    
    def get_user_posts(self, user_id):
        # Posts are sharded by user_id to maintain locality
        shard = self.get_shard(user_id)
        return shard.execute(
            "SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC",
            (user_id,)
        ).fetchall()

# Cross-shard queries (expensive)
def get_all_active_users(sharding):
    all_users = []
    for shard in sharding.shards:
        users = shard.execute("SELECT * FROM users WHERE active = ?", (True,))
        all_users.extend(users.fetchall())
    return all_users
```

---

## ACID Properties

### Atomicity
**All operations in a transaction succeed or fail together.**

```sql
-- Bank transfer example
BEGIN TRANSACTION;

UPDATE accounts 
SET balance = balance - 100 
WHERE account_id = 'A001';

UPDATE accounts 
SET balance = balance + 100 
WHERE account_id = 'A002';

-- If any operation fails, entire transaction is rolled back
COMMIT;
```

```python
# Python implementation with error handling
def transfer_money(from_account, to_account, amount):
    connection = get_db_connection()
    try:
        connection.begin()
        
        # Check sufficient balance
        balance = connection.execute(
            "SELECT balance FROM accounts WHERE account_id = ?",
            (from_account,)
        ).fetchone()[0]
        
        if balance < amount:
            raise InsufficientFundsError()
        
        # Debit from source account
        connection.execute(
            "UPDATE accounts SET balance = balance - ? WHERE account_id = ?",
            (amount, from_account)
        )
        
        # Credit to destination account
        connection.execute(
            "UPDATE accounts SET balance = balance + ? WHERE account_id = ?",
            (amount, to_account)
        )
        
        # Log transaction
        connection.execute(
            "INSERT INTO transactions (from_account, to_account, amount, timestamp) VALUES (?, ?, ?, ?)",
            (from_account, to_account, amount, datetime.now())
        )
        
        connection.commit()
        return True
        
    except Exception as e:
        connection.rollback()
        raise e
    finally:
        connection.close()
```

### Consistency
**Database remains in a valid state before and after transactions.**

```sql
-- Constraints ensure consistency
CREATE TABLE accounts (
    account_id VARCHAR(10) PRIMARY KEY,
    balance DECIMAL(10,2) NOT NULL CHECK (balance >= 0),  -- No negative balances
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('checking', 'savings')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Triggers for additional consistency
CREATE TRIGGER check_daily_withdrawal_limit
BEFORE UPDATE ON accounts
FOR EACH ROW
BEGIN
    DECLARE daily_withdrawals DECIMAL(10,2);
    
    SELECT COALESCE(SUM(amount), 0) INTO daily_withdrawals
    FROM transactions 
    WHERE account_id = NEW.account_id 
    AND transaction_type = 'withdrawal'
    AND DATE(created_at) = CURDATE();
    
    IF daily_withdrawals > 1000 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Daily withdrawal limit exceeded';
    END IF;
END;
```

### Isolation
**Concurrent transactions don't interfere with each other.**

```sql
-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- Dirty reads possible
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;    -- No dirty reads
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- No dirty/non-repeatable reads
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;      -- Full isolation

-- Example of isolation issues
-- Transaction 1
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'A001';  -- Returns 1000
-- ... some processing time ...
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A001';
COMMIT;

-- Transaction 2 (concurrent)
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'A001';  -- May return different values
UPDATE accounts SET balance = balance - 50 WHERE account_id = 'A001';
COMMIT;
```

### Durability
**Committed transactions persist even after system failures.**

```sql
-- Write-ahead logging (WAL)
-- Changes are logged before being applied to data files
-- Recovery process replays logs after crash

-- Example recovery process
-- 1. Read transaction log
-- 2. Identify committed vs uncommitted transactions
-- 3. Redo committed transactions
-- 4. Undo uncommitted transactions
```

---

## CAP Theorem

**You can only guarantee 2 out of 3: Consistency, Availability, Partition Tolerance**

### Consistency (C)
All nodes see the same data simultaneously.

```python
# Strong consistency example
class StronglyConsistentCache:
    def __init__(self, nodes):
        self.nodes = nodes
    
    def write(self, key, value):
        # Write to all nodes before returning success
        success_count = 0
        for node in self.nodes:
            try:
                node.write(key, value)
                success_count += 1
            except Exception:
                pass
        
        # Require all nodes to succeed
        if success_count == len(self.nodes):
            return True
        else:
            # Rollback on failure
            for node in self.nodes:
                try:
                    node.delete(key)
                except Exception:
                    pass
            return False
    
    def read(self, key):
        # Read from majority of nodes
        values = []
        for node in self.nodes:
            try:
                value = node.read(key)
                values.append(value)
            except Exception:
                pass
        
        # Return most common value
        if values:
            return max(set(values), key=values.count)
        return None
```

### Availability (A)
System remains operational and responsive.

```python
# High availability with eventual consistency
class EventuallyConsistentCache:
    def __init__(self, nodes):
        self.nodes = nodes
    
    def write(self, key, value):
        # Write to any available node
        for node in self.nodes:
            try:
                node.write(key, value)
                # Asynchronously replicate to other nodes
                self.async_replicate(key, value, exclude=node)
                return True
            except Exception:
                continue
        return False
    
    def read(self, key):
        # Read from any available node
        for node in self.nodes:
            try:
                return node.read(key)
            except Exception:
                continue
        return None
    
    def async_replicate(self, key, value, exclude=None):
        # Background replication
        for node in self.nodes:
            if node != exclude:
                try:
                    node.write(key, value)
                except Exception:
                    # Log for retry later
                    self.log_failed_replication(node, key, value)
```

### Partition Tolerance (P)
System continues despite network failures.

```python
# Partition-tolerant system
class PartitionTolerantSystem:
    def __init__(self, nodes):
        self.nodes = nodes
        self.partitions = self.detect_partitions()
    
    def detect_partitions(self):
        # Network partition detection logic
        partitions = []
        for node in self.nodes:
            if self.can_communicate(node):
                partitions.append(node)
        return partitions
    
    def write(self, key, value):
        # Write to majority partition
        majority_size = len(self.nodes) // 2 + 1
        available_nodes = self.detect_partitions()
        
        if len(available_nodes) >= majority_size:
            # Can maintain consistency
            return self.consistent_write(key, value, available_nodes)
        else:
            # Choose availability over consistency
            return self.available_write(key, value, available_nodes)
```

### CAP Theorem Examples

#### CP Systems (Consistency + Partition Tolerance)
```
Examples: MongoDB, HBase, Redis Cluster
- Sacrifice availability during partitions
- Strong consistency guarantees
- May become unavailable during network splits
```

#### AP Systems (Availability + Partition Tolerance)
```
Examples: Cassandra, DynamoDB, CouchDB
- Always available for reads/writes
- Eventual consistency
- May return stale data during partitions
```

#### CA Systems (Consistency + Availability)
```
Examples: Traditional RDBMS (MySQL, PostgreSQL)
- Strong consistency and high availability
- Cannot handle network partitions
- Suitable for single data center deployments
```

---

## Database Sharding

### Sharding Strategies

#### 1. Range-Based Sharding
```python
class RangeBasedSharding:
    def __init__(self, shards, ranges):
        self.shards = shards
        self.ranges = ranges  # [(0, 1000), (1001, 2000), (2001, 3000)]
    
    def get_shard(self, key):
        for i, (start, end) in enumerate(self.ranges):
            if start <= key <= end:
                return self.shards[i]
        raise ValueError(f"No shard found for key {key}")
    
    def rebalance(self, new_ranges):
        # Migrate data when ranges change
        old_ranges = self.ranges
        self.ranges = new_ranges
        
        for shard_index, (old_range, new_range) in enumerate(zip(old_ranges, new_ranges)):
            if old_range != new_range:
                self.migrate_data(shard_index, old_range, new_range)

# Usage
sharding = RangeBasedSharding(
    shards=[shard1, shard2, shard3],
    ranges=[(1, 1000), (1001, 2000), (2001, 3000)]
)
```

**Pros:**
- Range queries are efficient
- Easy to understand and implement
- Good for time-series data

**Cons:**
- Hot spots if data is not evenly distributed
- Difficult to rebalance
- May require manual range adjustments

#### 2. Hash-Based Sharding
```python
import hashlib
import bisect

class ConsistentHashSharding:
    def __init__(self, shards, virtual_nodes=150):
        self.shards = shards
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self._build_ring()
    
    def _hash(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)
    
    def _build_ring(self):
        for shard in self.shards:
            for i in range(self.virtual_nodes):
                virtual_key = self._hash(f"{shard}:{i}")
                self.ring[virtual_key] = shard
                self.sorted_keys.append(virtual_key)
        self.sorted_keys.sort()
    
    def get_shard(self, key):
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        
        # Find the first node clockwise from the hash
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def add_shard(self, shard):
        self.shards.append(shard)
        for i in range(self.virtual_nodes):
            virtual_key = self._hash(f"{shard}:{i}")
            self.ring[virtual_key] = shard
            bisect.insort(self.sorted_keys, virtual_key)
    
    def remove_shard(self, shard):
        self.shards.remove(shard)
        keys_to_remove = []
        for virtual_key, s in self.ring.items():
            if s == shard:
                keys_to_remove.append(virtual_key)
        
        for key in keys_to_remove:
            del self.ring[key]
            self.sorted_keys.remove(key)
```

**Pros:**
- Even distribution of data
- Easy to add/remove shards
- Minimal data movement during rebalancing

**Cons:**
- Range queries require querying all shards
- More complex implementation
- Hash collisions possible

#### 3. Directory-Based Sharding
```python
class DirectoryBasedSharding:
    def __init__(self, shards):
        self.shards = shards
        self.directory = {}  # key_range -> shard_id
        self.shard_map = {i: shard for i, shard in enumerate(shards)}
    
    def register_range(self, start_key, end_key, shard_id):
        self.directory[(start_key, end_key)] = shard_id
    
    def get_shard(self, key):
        for (start, end), shard_id in self.directory.items():
            if start <= key <= end:
                return self.shard_map[shard_id]
        raise ValueError(f"No shard found for key {key}")
    
    def migrate_range(self, start_key, end_key, from_shard_id, to_shard_id):
        # Update directory
        del self.directory[(start_key, end_key)]
        self.directory[(start_key, end_key)] = to_shard_id
        
        # Migrate data
        from_shard = self.shard_map[from_shard_id]
        to_shard = self.shard_map[to_shard_id]
        
        data = from_shard.get_range(start_key, end_key)
        to_shard.insert_batch(data)
        from_shard.delete_range(start_key, end_key)

# Usage
sharding = DirectoryBasedSharding([shard1, shard2, shard3])
sharding.register_range(1, 1000, 0)
sharding.register_range(1001, 2000, 1)
sharding.register_range(2001, 3000, 2)
```

**Pros:**
- Flexible shard assignment
- Easy to rebalance
- Can optimize for access patterns

**Cons:**
- Additional lookup overhead
- Directory service becomes bottleneck
- More complex architecture

### Sharding Challenges

#### Cross-Shard Queries
```python
class CrossShardQuery:
    def __init__(self, sharding):
        self.sharding = sharding
    
    def aggregate_query(self, query, aggregation_func):
        results = []
        for shard in self.sharding.shards:
            try:
                result = shard.execute(query)
                results.extend(result)
            except Exception as e:
                # Handle shard failures
                logging.error(f"Shard query failed: {e}")
        
        return aggregation_func(results)
    
    def join_query(self, table1, table2, join_condition):
        # This is expensive and should be avoided
        all_table1_data = self.get_all_data(table1)
        all_table2_data = self.get_all_data(table2)
        
        # Perform join in application layer
        return self.perform_join(all_table1_data, all_table2_data, join_condition)
    
    def get_all_data(self, table):
        all_data = []
        for shard in self.sharding.shards:
            data = shard.execute(f"SELECT * FROM {table}")
            all_data.extend(data)
        return all_data
```

#### Rebalancing
```python
class ShardRebalancer:
    def __init__(self, sharding):
        self.sharding = sharding
    
    def rebalance(self):
        # Analyze shard sizes and load
        shard_stats = self.analyze_shards()
        
        # Identify overloaded and underloaded shards
        overloaded = [s for s in shard_stats if s['load'] > self.max_load_threshold]
        underloaded = [s for s in shard_stats if s['load'] < self.min_load_threshold]
        
        # Plan data migration
        migration_plan = self.create_migration_plan(overloaded, underloaded)
        
        # Execute migration
        self.execute_migration(migration_plan
