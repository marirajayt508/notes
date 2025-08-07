# LRU Cache Implementation - Algorithmic Patterns

## Table of Contents
1. [What is LRU Cache?](#what-is-lru-cache)
2. [Implementation Approaches](#implementation-approaches)
3. [HashMap + Doubly Linked List](#hashmap--doubly-linked-list)
4. [Python OrderedDict Implementation](#python-ordereddict-implementation)
5. [Thread-Safe Implementation](#thread-safe-implementation)
6. [Distributed LRU Cache](#distributed-lru-cache)
7. [Interview Questions](#interview-questions)

---

## What is LRU Cache?

**LRU (Least Recently Used)** is a cache eviction policy that removes the least recently used items first when the cache reaches its capacity limit.

### Key Properties:
- **Fixed capacity**: Cache has a maximum size
- **Fast access**: O(1) get and put operations
- **Eviction policy**: Remove least recently used item when full
- **Update on access**: Both get and put operations update recency

### Use Cases:
- CPU cache management
- Operating system page replacement
- Database buffer pools
- Web browser cache
- CDN cache eviction
- Application-level caching (Redis, Memcached)

---

## Implementation Approaches

### Approach 1: Array/List (Naive)
```python
# Time Complexity: O(n) for both get and put
# Space Complexity: O(capacity)
class LRUCacheNaive:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = []  # [(key, value), ...]
    
    def get(self, key):
        for i, (k, v) in enumerate(self.cache):
            if k == key:
                # Move to front (most recent)
                item = self.cache.pop(i)
                self.cache.insert(0, item)
                return v
        return -1
    
    def put(self, key, value):
        # Check if key exists
        for i, (k, v) in enumerate(self.cache):
            if k == key:
                self.cache.pop(i)
                self.cache.insert(0, (key, value))
                return
        
        # Add new item
        if len(self.cache) >= self.capacity:
            self.cache.pop()  # Remove least recent
        self.cache.insert(0, (key, value))
```

**Problems:**
- O(n) time complexity
- Inefficient for large caches
- Not suitable for production use

### Approach 2: HashMap + Doubly Linked List (Optimal)
```python
# Time Complexity: O(1) for both get and put
# Space Complexity: O(capacity)
```

---

## HashMap + Doubly Linked List

This is the **optimal solution** used in production systems.

### Data Structure Design:
```
HashMap: key → Node
Doubly Linked List: head ↔ node1 ↔ node2 ↔ ... ↔ tail

Most Recent (head) ←→ ... ←→ Least Recent (tail)
```

### Complete Implementation:

```python
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> node
        
        # Create dummy head and tail nodes
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _add_node(self, node):
        """Add node right after head"""
        node.prev = self.head
        node.next = self.head.next
        
        self.head.next.prev = node
        self.head.next = node
    
    def _remove_node(self, node):
        """Remove an existing node from the linked list"""
        prev_node = node.prev
        next_node = node.next
        
        prev_node.next = next_node
        next_node.prev = prev_node
    
    def _move_to_head(self, node):
        """Move node to head (mark as most recently used)"""
        self._remove_node(node)
        self._add_node(node)
    
    def _pop_tail(self):
        """Remove the last node (least recently used)"""
        last_node = self.tail.prev
        self._remove_node(last_node)
        return last_node
    
    def get(self, key: int) -> int:
        node = self.cache.get(key)
        
        if node:
            # Move to head (mark as recently used)
            self._move_to_head(node)
            return node.value
        
        return -1
    
    def put(self, key: int, value: int) -> None:
        node = self.cache.get(key)
        
        if node:
            # Update existing node
            node.value = value
            self._move_to_head(node)
        else:
            # Add new node
            new_node = Node(key, value)
            
            if len(self.cache) >= self.capacity:
                # Remove least recently used
                tail = self._pop_tail()
                del self.cache[tail.key]
            
            self.cache[key] = new_node
            self._add_node(new_node)

# Usage Example
lru = LRUCache(2)
lru.put(1, 1)  # cache: {1=1}
lru.put(2, 2)  # cache: {1=1, 2=2}
print(lru.get(1))  # returns 1, cache: {2=2, 1=1}
lru.put(3, 3)  # evicts key 2, cache: {1=1, 3=3}
print(lru.get(2))  # returns -1 (not found)
```

### Visual Representation:
```
Initial: head ↔ tail

After put(1, 1):
head ↔ [1,1] ↔ tail

After put(2, 2):
head ↔ [2,2] ↔ [1,1] ↔ tail

After get(1):
head ↔ [1,1] ↔ [2,2] ↔ tail

After put(3, 3) (capacity=2):
head ↔ [3,3] ↔ [1,1] ↔ tail
(node [2,2] evicted)
```

---

## Python OrderedDict Implementation

Python's `OrderedDict` maintains insertion order and provides O(1) operations:

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key: int) -> int:
        if key in self.cache:
            # Move to end (most recent)
            self.cache.move_to_end(key)
            return self.cache[key]
        return -1
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            # Update existing key
            self.cache.move_to_end(key)
        elif len(self.cache) >= self.capacity:
            # Remove least recent (first item)
            self.cache.popitem(last=False)
        
        self.cache[key] = value

# Even more concise version
class LRUCacheSimple:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            del self.cache[key]
        elif len(self.cache) >= self.capacity:
            self.cache.popitem(last=False)
        self.cache[key] = value
```

**Pros:**
- Much simpler code
- Built-in Python optimization
- Less error-prone

**Cons:**
- Language-specific (not available in all languages)
- Less control over implementation details
- May not demonstrate algorithmic thinking in interviews

---

## Thread-Safe Implementation

For multi-threaded environments:

```python
import threading
from collections import OrderedDict

class ThreadSafeLRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = threading.RLock()  # Reentrant lock
    
    def get(self, key: int) -> int:
        with self.lock:
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)
            return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        with self.lock:
            if key in self.cache:
                del self.cache[key]
            elif len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)
            self.cache[key] = value
    
    def size(self) -> int:
        with self.lock:
            return len(self.cache)
    
    def clear(self) -> None:
        with self.lock:
            self.cache.clear()

# Usage with multiple threads
import concurrent.futures

cache = ThreadSafeLRUCache(100)

def worker(thread_id):
    for i in range(1000):
        cache.put(f"{thread_id}_{i}", i)
        cache.get(f"{thread_id}_{i//2}")

with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(worker, i) for i in range(10)]
    concurrent.futures.wait(futures)
```

---

## Distributed LRU Cache

For distributed systems, LRU becomes more complex:

### Challenges:
1. **Global LRU order**: Hard to maintain across nodes
2. **Network latency**: Affects performance
3. **Consistency**: Cache coherence across nodes
4. **Partitioning**: How to distribute keys

### Approach 1: Consistent Hashing + Local LRU
```python
import hashlib

class DistributedLRUCache:
    def __init__(self, nodes, capacity_per_node):
        self.nodes = nodes  # List of cache nodes
        self.capacity_per_node = capacity_per_node
        self.local_caches = {
            node: LRUCache(capacity_per_node) 
            for node in nodes
        }
    
    def _hash_key(self, key):
        """Consistent hashing to determine node"""
        hash_value = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return self.nodes[hash_value % len(self.nodes)]
    
    def get(self, key):
        node = self._hash_key(key)
        return self.local_caches[node].get(key)
    
    def put(self, key, value):
        node = self._hash_key(key)
        self.local_caches[node].put(key, value)
```

### Approach 2: Global LRU with Timestamps
```python
import time
from collections import OrderedDict

class GlobalLRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> (value, timestamp)
        self.access_times = OrderedDict()  # timestamp -> key
    
    def get(self, key):
        if key not in self.cache:
            return -1
        
        value, old_timestamp = self.cache[key]
        
        # Update timestamp
        new_timestamp = time.time()
        self.cache[key] = (value, new_timestamp)
        
        # Update access order
        del self.access_times[old_timestamp]
        self.access_times[new_timestamp] = key
        
        return value
    
    def put(self, key, value):
        timestamp = time.time()
        
        if key in self.cache:
            # Update existing
            old_timestamp = self.cache[key][1]
            del self.access_times[old_timestamp]
        elif len(self.cache) >= self.capacity:
            # Evict least recent
            oldest_timestamp = next(iter(self.access_times))
            oldest_key = self.access_times[oldest_timestamp]
            del self.cache[oldest_key]
            del self.access_times[oldest_timestamp]
        
        self.cache[key] = (value, timestamp)
        self.access_times[timestamp] = key
```

---

## Advanced Variations

### 1. LRU with TTL (Time To Live)
```python
import time

class LRUCacheWithTTL:
    def __init__(self, capacity, default_ttl=3600):
        self.capacity = capacity
        self.default_ttl = default_ttl
        self.cache = OrderedDict()  # key -> (value, expiry_time)
    
    def _is_expired(self, key):
        if key not in self.cache:
            return True
        _, expiry_time = self.cache[key]
        return time.time() > expiry_time
    
    def get(self, key):
        if self._is_expired(key):
            if key in self.cache:
                del self.cache[key]
            return -1
        
        self.cache.move_to_end(key)
        return self.cache[key][0]
    
    def put(self, key, value, ttl=None):
        if ttl is None:
            ttl = self.default_ttl
        
        expiry_time = time.time() + ttl
        
        if key in self.cache:
            del self.cache[key]
        elif len(self.cache) >= self.capacity:
            self.cache.popitem(last=False)
        
        self.cache[key] = (value, expiry_time)
```

### 2. Weighted LRU Cache
```python
class WeightedLRUCache:
    def __init__(self, max_weight):
        self.max_weight = max_weight
        self.current_weight = 0
        self.cache = OrderedDict()  # key -> (value, weight)
    
    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key][0]
    
    def put(self, key, value, weight=1):
        if key in self.cache:
            old_weight = self.cache[key][1]
            self.current_weight -= old_weight
            del self.cache[key]
        
        # Evict items until we have enough space
        while (self.current_weight + weight > self.max_weight 
               and self.cache):
            _, (_, evicted_weight) = self.cache.popitem(last=False)
            self.current_weight -= evicted_weight
        
        if weight <= self.max_weight:
            self.cache[key] = (value, weight)
            self.current_weight += weight
```

---

## Interview Questions

### Q1: Implement an LRU Cache with O(1) operations.

**Key Points:**
- Use HashMap + Doubly Linked List
- Explain why both data structures are needed
- Walk through get() and put() operations
- Handle edge cases (capacity 0, duplicate keys)

### Q2: How would you implement LRU cache in a distributed system?

**Answer Framework:**
1. **Challenges**: Global ordering, network latency, consistency
2. **Solutions**: 
   - Consistent hashing with local LRU
   - Global timestamps with eventual consistency
   - Approximate LRU with periodic synchronization

### Q3: What are the trade-offs between LRU and other cache eviction policies?

**Comparison:**
- **LRU**: Good temporal locality, complex implementation
- **FIFO**: Simple, but ignores access patterns
- **LFU**: Good for frequency-based patterns, more memory overhead
- **Random**: Simple, surprisingly effective in some cases

### Q4: How would you test an LRU cache implementation?

**Test Cases:**
1. Basic functionality (get/put)
2. Capacity limits and eviction
3. Update existing keys
4. Edge cases (capacity 0, single item)
5. Performance tests
6. Thread safety (if applicable)

```python
def test_lru_cache():
    # Test basic functionality
    cache = LRUCache(2)
    
    # Test put and get
    cache.put(1, 1)
    cache.put(2, 2)
    assert cache.get(1) == 1
    
    # Test eviction
    cache.put(3, 3)  # Should evict key 2
    assert cache.get(2) == -1
    assert cache.get(3) == 3
    assert cache.get(1) == 1
    
    # Test update
    cache.put(1, 10)
    assert cache.get(1) == 10
    
    print("All tests passed!")
```

---

## Real-World Applications

### 1. Redis LRU Implementation
Redis uses an approximation of LRU:
- Samples random keys instead of maintaining perfect order
- Uses 24-bit timestamps for efficiency
- Configurable sample size for accuracy vs performance trade-off

### 2. CPU Cache
- L1, L2, L3 caches use LRU or pseudo-LRU
- Hardware implementation for speed
- Critical for processor performance

### 3. Operating System Page Replacement
- Virtual memory management
- LRU approximation algorithms (Clock, Second Chance)
- Balance between accuracy and overhead

### 4. Database Buffer Pools
- MySQL InnoDB buffer pool
- PostgreSQL shared buffers
- Cache frequently accessed pages

---

## Performance Analysis

### Time Complexity:
- **get()**: O(1)
- **put()**: O(1)
- **Space**: O(capacity)

### Space Optimization:
```python
# Memory-efficient node structure
class CompactNode:
    __slots__ = ['key', 'value', 'prev', 'next']  # Reduces memory overhead
    
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None
```

### Benchmarking:
```python
import time
import random

def benchmark_lru_cache():
    cache = LRUCache(1000)
    
    # Warm up
    for i in range(1000):
        cache.put(i, i)
    
    # Benchmark gets
    start_time = time.time()
    for _ in range(100000):
        key = random.randint(0, 999)
        cache.get(key)
    get_time = time.time() - start_time
    
    # Benchmark puts
    start_time = time.time()
    for _ in range(100000):
        key = random.randint(1000, 1999)
        cache.put(key, key)
    put_time = time.time() - start_time
    
    print(f"Get operations: {get_time:.4f}s")
    print(f"Put operations: {put_time:.4f}s")
```

---

## Key Takeaways

1. **LRU Cache is a fundamental data structure** in system design
2. **HashMap + Doubly Linked List** provides O(1) operations
3. **Thread safety** requires careful synchronization
4. **Distributed LRU** involves trade-offs between consistency and performance
5. **Real systems often use approximations** for better performance
6. **Understanding the implementation details** is crucial for interviews

Remember: LRU cache appears in many system design problems - from designing a web cache to implementing a database buffer pool. Master this pattern!
