# Bloom Filters

A Bloom filter is a space-efficient probabilistic data structure designed to test whether an element is a member of a set. It can have false positives but never false negatives, making it ideal for applications where memory efficiency is crucial and occasional false positives are acceptable.

## Table of Contents
- [What is a Bloom Filter?](#what-is-a-bloom-filter)
- [How Bloom Filters Work](#how-bloom-filters-work)
- [Implementation](#implementation)
- [Mathematical Analysis](#mathematical-analysis)
- [Variants and Extensions](#variants-and-extensions)
- [Real-World Applications](#real-world-applications)
- [Trade-offs and Limitations](#trade-offs-and-limitations)

## What is a Bloom Filter?

### Key Properties
- **Space-efficient**: Uses a fixed-size bit array regardless of the number of elements
- **Probabilistic**: Can have false positives but never false negatives
- **Fast operations**: O(k) time complexity for both insertion and lookup, where k is the number of hash functions
- **No deletions**: Standard Bloom filters don't support element removal

### When to Use Bloom Filters
- **Pre-filtering**: Check if expensive operations (database queries, network requests) are worth performing
- **Caching**: Determine if an item might be in cache before checking
- **Distributed systems**: Reduce network calls by filtering out definitely absent items
- **Security**: Detect malicious URLs, spam, or known attacks
- **Big data**: Handle massive datasets where memory is constrained

## How Bloom Filters Work

### Basic Algorithm

1. **Initialize**: Create a bit array of size `m`, all bits set to 0
2. **Add element**: 
   - Hash the element with `k` different hash functions
   - Set the corresponding bits to 1
3. **Query element**:
   - Hash the element with the same `k` hash functions
   - Check if all corresponding bits are 1
   - If any bit is 0: element is definitely NOT in the set
   - If all bits are 1: element MIGHT be in the set

### Visual Example

```
Bit Array (size 10): [0,0,0,0,0,0,0,0,0,0]

Add "apple":
- hash1("apple") = 2
- hash2("apple") = 5  
- hash3("apple") = 8
Result: [0,0,1,0,0,1,0,0,1,0]

Add "banana":
- hash1("banana") = 1
- hash2("banana") = 5  (collision!)
- hash3("banana") = 7
Result: [0,1,1,0,0,1,0,1,1,0]

Query "apple": Check positions 2,5,8 → all are 1 → MIGHT be present
Query "cherry": hash1=3, hash2=6, hash3=9 → position 3 is 0 → DEFINITELY not present
Query "grape": hash1=1, hash2=5, hash3=8 → all are 1 → FALSE POSITIVE!
```

## Implementation

### Basic Bloom Filter

```python
import hashlib
import math
from typing import Any, List

class BloomFilter:
    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        """
        Initialize Bloom Filter
        
        Args:
            expected_items: Expected number of items to be added
            false_positive_rate: Desired false positive probability
        """
        self.expected_items = expected_items
        self.false_positive_rate = false_positive_rate
        
        # Calculate optimal parameters
        self.size = self._calculate_size(expected_items, false_positive_rate)
        self.hash_count = self._calculate_hash_count(self.size, expected_items)
        
        # Initialize bit array
        self.bit_array = [0] * self.size
        self.items_added = 0
    
    def _calculate_size(self, n: int, p: float) -> int:
        """Calculate optimal bit array size"""
        return int(-n * math.log(p) / (math.log(2) ** 2))
    
    def _calculate_hash_count(self, m: int, n: int) -> int:
        """Calculate optimal number of hash functions"""
        return int(m * math.log(2) / n)
    
    def _hash(self, item: Any) -> List[int]:
        """Generate k hash values for an item"""
        # Convert item to string and encode
        item_str = str(item).encode('utf-8')
        
        # Use different hash functions
        hashes = []
        for i in range(self.hash_count):
            # Create different hash functions using salt
            hash_input = item_str + str(i).encode('utf-8')
            hash_value = int(hashlib.md5(hash_input).hexdigest(), 16)
            hashes.append(hash_value % self.size)
        
        return hashes
    
    def add(self, item: Any) -> None:
        """Add an item to the Bloom filter"""
        hash_values = self._hash(item)
        for hash_val in hash_values:
            self.bit_array[hash_val] = 1
        self.items_added += 1
    
    def contains(self, item: Any) -> bool:
        """Check if item might be in the set"""
        hash_values = self._hash(item)
        return all(self.bit_array[hash_val] == 1 for hash_val in hash_values)
    
    def current_false_positive_rate(self) -> float:
        """Calculate current false positive rate"""
        if self.items_added == 0:
            return 0.0
        
        # Probability that a bit is still 0
        prob_zero = (1 - 1/self.size) ** (self.hash_count * self.items_added)
        
        # False positive rate
        return (1 - prob_zero) ** self.hash_count
    
    def __len__(self) -> int:
        """Return number of items added"""
        return self.items_added
    
    def __contains__(self, item: Any) -> bool:
        """Support 'in' operator"""
        return self.contains(item)

# Usage example
bf = BloomFilter(expected_items=1000, false_positive_rate=0.01)

# Add items
bf.add("apple")
bf.add("banana")
bf.add("cherry")

# Check membership
print("apple" in bf)     # True (definitely added)
print("banana" in bf)    # True (definitely added)
print("grape" in bf)     # False or True (might be false positive)
print("orange" in bf)    # False or True (might be false positive)
```

### Optimized Implementation with Multiple Hash Functions

```python
import mmh3  # MurmurHash3 - install with: pip install mmh3

class OptimizedBloomFilter:
    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        self.expected_items = expected_items
        self.false_positive_rate = false_positive_rate
        
        # Calculate parameters
        self.size = int(-expected_items * math.log(false_positive_rate) / (math.log(2) ** 2))
        self.hash_count = int(self.size * math.log(2) / expected_items)
        
        # Use bitarray for better memory efficiency
        try:
            from bitarray import bitarray
            self.bit_array = bitarray(self.size)
            self.bit_array.setall(0)
            self._use_bitarray = True
        except ImportError:
            self.bit_array = [0] * self.size
            self._use_bitarray = False
        
        self.items_added = 0
    
    def _hash(self, item: Any) -> List[int]:
        """Generate hash values using MurmurHash3"""
        item_bytes = str(item).encode('utf-8')
        hashes = []
        
        for i in range(self.hash_count):
            hash_val = mmh3.hash(item_bytes, i) % self.size
            hashes.append(hash_val)
        
        return hashes
    
    def add(self, item: Any) -> None:
        """Add item to filter"""
        for hash_val in self._hash(item):
            if self._use_bitarray:
                self.bit_array[hash_val] = True
            else:
                self.bit_array[hash_val] = 1
        self.items_added += 1
    
    def contains(self, item: Any) -> bool:
        """Check if item might be present"""
        for hash_val in self._hash(item):
            if self._use_bitarray:
                if not self.bit_array[hash_val]:
                    return False
            else:
                if self.bit_array[hash_val] == 0:
                    return False
        return True
    
    def union(self, other: 'OptimizedBloomFilter') -> 'OptimizedBloomFilter':
        """Create union of two Bloom filters"""
        if self.size != other.size or self.hash_count != other.hash_count:
            raise ValueError("Bloom filters must have same parameters for union")
        
        result = OptimizedBloomFilter(self.expected_items, self.false_positive_rate)
        
        for i in range(self.size):
            if self._use_bitarray:
                result.bit_array[i] = self.bit_array[i] or other.bit_array[i]
            else:
                result.bit_array[i] = self.bit_array[i] or other.bit_array[i]
        
        result.items_added = self.items_added + other.items_added
        return result
    
    def intersection(self, other: 'OptimizedBloomFilter') -> 'OptimizedBloomFilter':
        """Create intersection of two Bloom filters"""
        if self.size != other.size or self.hash_count != other.hash_count:
            raise ValueError("Bloom filters must have same parameters for intersection")
        
        result = OptimizedBloomFilter(self.expected_items, self.false_positive_rate)
        
        for i in range(self.size):
            if self._use_bitarray:
                result.bit_array[i] = self.bit_array[i] and other.bit_array[i]
            else:
                result.bit_array[i] = self.bit_array[i] and other.bit_array[i]
        
        # Intersection size is harder to estimate
        result.items_added = min(self.items_added, other.items_added)
        return result
```

## Mathematical Analysis

### Optimal Parameters

Given:
- `n` = number of items to insert
- `p` = desired false positive rate
- `m` = size of bit array
- `k` = number of hash functions

**Optimal bit array size:**
```
m = -n * ln(p) / (ln(2))²
```

**Optimal number of hash functions:**
```
k = (m/n) * ln(2)
```

**Actual false positive rate:**
```
p = (1 - e^(-kn/m))^k
```

### Space Complexity Analysis

```python
def analyze_bloom_filter(n: int, p: float):
    """Analyze Bloom filter parameters"""
    
    # Calculate optimal parameters
    m = int(-n * math.log(p) / (math.log(2) ** 2))
    k = int(m * math.log(2) / n)
    
    # Memory usage
    memory_bits = m
    memory_bytes = m / 8
    memory_kb = memory_bytes / 1024
    
    # Bits per element
    bits_per_element = m / n
    
    print(f"Analysis for {n:,} items with {p:.1%} false positive rate:")
    print(f"Bit array size: {m:,} bits ({memory_kb:.2f} KB)")
    print(f"Hash functions: {k}")
    print(f"Bits per element: {bits_per_element:.2f}")
    print(f"Memory efficiency vs set: {100 * bits_per_element / (64 * 8):.1f}%")

# Example analysis
analyze_bloom_filter(1_000_000, 0.01)
# Output:
# Analysis for 1,000,000 items with 1.0% false positive rate:
# Bit array size: 9,585,059 bits (1,173.46 KB)
# Hash functions: 6
# Bits per element: 9.59
# Memory efficiency vs set: 1.9%
```

## Variants and Extensions

### 1. Counting Bloom Filter

Supports deletions by using counters instead of bits.

```python
class CountingBloomFilter:
    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        self.expected_items = expected_items
        self.false_positive_rate = false_positive_rate
        
        # Calculate parameters (same as regular Bloom filter)
        self.size = int(-expected_items * math.log(false_positive_rate) / (math.log(2) ** 2))
        self.hash_count = int(self.size * math.log(2) / expected_items)
        
        # Use counters instead of bits (4-bit counters are usually sufficient)
        self.counters = [0] * self.size
        self.items_added = 0
    
    def _hash(self, item: Any) -> List[int]:
        """Generate hash values"""
        item_str = str(item).encode('utf-8')
        hashes = []
        for i in range(self.hash_count):
            hash_input = item_str + str(i).encode('utf-8')
            hash_value = int(hashlib.md5(hash_input).hexdigest(), 16)
            hashes.append(hash_value % self.size)
        return hashes
    
    def add(self, item: Any) -> None:
        """Add item to filter"""
        for hash_val in self._hash(item):
            if self.counters[hash_val] < 15:  # Prevent overflow (4-bit counter)
                self.counters[hash_val] += 1
        self.items_added += 1
    
    def remove(self, item: Any) -> bool:
        """Remove item from filter"""
        hash_values = self._hash(item)
        
        # Check if item might be present
        if not all(self.counters[h] > 0 for h in hash_values):
            return False  # Item definitely not present
        
        # Decrement counters
        for hash_val in hash_values:
            if self.counters[hash_val] > 0:
                self.counters[hash_val] -= 1
        
        self.items_added -= 1
        return True
    
    def contains(self, item: Any) -> bool:
        """Check if item might be present"""
        return all(self.counters[h] > 0 for h in self._hash(item))
```

### 2. Scalable Bloom Filter

Automatically grows to maintain false positive rate.

```python
class ScalableBloomFilter:
    def __init__(self, initial_capacity: int = 1000, 
                 false_positive_rate: float = 0.01,
                 growth_factor: int = 2):
        self.initial_capacity = initial_capacity
        self.false_positive_rate = false_positive_rate
        self.growth_factor = growth_factor
        
        # List of Bloom filters
        self.filters = []
        self.current_capacity = initial_capacity
        
        # Create first filter
        self._add_filter()
    
    def _add_filter(self):
        """Add a new Bloom filter to the chain"""
        # Each new filter has tighter false positive rate
        filter_fp_rate = self.false_positive_rate * (0.5 ** len(self.filters))
        new_filter = BloomFilter(self.current_capacity, filter_fp_rate)
        self.filters.append(new_filter)
        self.current_capacity *= self.growth_factor
    
    def add(self, item: Any) -> None:
        """Add item to the most recent filter"""
        current_filter = self.filters[-1]
        
        # If current filter is full, create a new one
        if current_filter.items_added >= current_filter.expected_items:
            self._add_filter()
            current_filter = self.filters[-1]
        
        current_filter.add(item)
    
    def contains(self, item: Any) -> bool:
        """Check if item might be present in any filter"""
        return any(f.contains(item) for f in self.filters)
    
    def __len__(self) -> int:
        """Total items added"""
        return sum(len(f) for f in self.filters)
```

### 3. Cuckoo Filter

Alternative to Bloom filters that supports deletions and has better space efficiency.

```python
import random

class CuckooFilter:
    def __init__(self, capacity: int, bucket_size: int = 4, max_kicks: int = 500):
        self.capacity = capacity
        self.bucket_size = bucket_size
        self.max_kicks = max_kicks
        
        # Create buckets
        self.buckets = [[] for _ in range(capacity)]
        self.size = 0
    
    def _hash1(self, item: Any) -> int:
        """Primary hash function"""
        return hash(str(item)) % self.capacity
    
    def _hash2(self, item: Any, fingerprint: int) -> int:
        """Secondary hash function"""
        return (self._hash1(item) ^ hash(fingerprint)) % self.capacity
    
    def _fingerprint(self, item: Any) -> int:
        """Generate fingerprint for item"""
        return hash(str(item)) % (2**8)  # 8-bit fingerprint
    
    def add(self, item: Any) -> bool:
        """Add item to filter"""
        fingerprint = self._fingerprint(item)
        i1 = self._hash1(item)
        i2 = self._hash2(item, fingerprint)
        
        # Try to insert in either bucket
        if len(self.buckets[i1]) < self.bucket_size:
            self.buckets[i1].append(fingerprint)
            self.size += 1
            return True
        
        if len(self.buckets[i2]) < self.bucket_size:
            self.buckets[i2].append(fingerprint)
            self.size += 1
            return True
        
        # Cuckoo eviction
        i = random.choice([i1, i2])
        for _ in range(self.max_kicks):
            # Randomly evict an item
            if self.buckets[i]:
                evicted = random.choice(self.buckets[i])
                self.buckets[i].remove(evicted)
                self.buckets[i].append(fingerprint)
                
                # Find alternative location for evicted item
                alt_i = i ^ hash(evicted) % self.capacity
                if len(self.buckets[alt_i]) < self.bucket_size:
                    self.buckets[alt_i].append(evicted)
                    self.size += 1
                    return True
                
                # Continue cuckoo process
                fingerprint = evicted
                i = alt_i
        
        return False  # Filter is full
    
    def contains(self, item: Any) -> bool:
        """Check if item might be present"""
        fingerprint = self._fingerprint(item)
        i1 = self._hash1(item)
        i2 = self._hash2(item, fingerprint)
        
        return (fingerprint in self.buckets[i1] or 
                fingerprint in self.buckets[i2])
    
    def remove(self, item: Any) -> bool:
        """Remove item from filter"""
        fingerprint = self._fingerprint(item)
        i1 = self._hash1(item)
        i2 = self._hash2(item, fingerprint)
        
        if fingerprint in self.buckets[i1]:
            self.buckets[i1].remove(fingerprint)
            self.size -= 1
            return True
        
        if fingerprint in self.buckets[i2]:
            self.buckets[i2].remove(fingerprint)
            self.size -= 1
            return True
        
        return False
```

## Real-World Applications

### 1. Web Crawling - Duplicate URL Detection

```python
class WebCrawler:
    def __init__(self, expected_urls: int = 10_000_000):
        self.visited_urls = BloomFilter(expected_urls, 0.01)
        self.url_queue = []
    
    def should_crawl(self, url: str) -> bool:
        """Check if URL should be crawled"""
        if url in self.visited_urls:
            return False  # Might be duplicate (false positive possible)
        
        # Additional check against database for false positives
        if self._check_database(url):
            return False  # Confirmed duplicate
        
        return True
    
    def crawl_url(self, url: str):
        """Crawl URL and mark as visited"""
        if self.should_crawl(url):
            self.visited_urls.add(url)
            # Perform actual crawling
            content = self._fetch_url(url)
            self._process_content(content)
    
    def _check_database(self, url: str) -> bool:
        """Expensive database check for confirmed duplicates"""
        # Only called for potential false positives
        pass
    
    def _fetch_url(self, url: str):
        """Fetch URL content"""
        pass
    
    def _process_content(self, content):
        """Process crawled content"""
        pass
```

### 2. Database Query Optimization

```python
class DatabaseQueryCache:
    def __init__(self, cache_size: int = 100_000):
        self.cache_filter = BloomFilter(cache_size, 0.01)
        self.actual_cache = {}  # LRU cache
        self.max_cache_size = cache_size // 10  # Actual cache is smaller
    
    def get(self, query: str):
        """Get query result with Bloom filter pre-check"""
        
        # Fast check: definitely not in cache
        if query not in self.cache_filter:
            result = self._execute_query(query)
            self._add_to_cache(query, result)
            return result
        
        # Might be in cache, check actual cache
        if query in self.actual_cache:
            return self.actual_cache[query]  # Cache hit
        
        # False positive: not actually in cache
        result = self._execute_query(query)
        self._add_to_cache(query, result)
        return result
    
    def _add_to_cache(self, query: str, result):
        """Add query result to cache"""
        self.cache_filter.add(query)
        
        # Manage actual cache size (LRU eviction)
        if len(self.actual_cache) >= self.max_cache_size:
            # Remove oldest item (simplified LRU)
            oldest_key = next(iter(self.actual_cache))
            del self.actual_cache[oldest_key]
        
        self.actual_cache[query] = result
    
    def _execute_query(self, query: str):
        """Execute expensive database query"""
        # Simulate database query
        import time
        time.sleep(0.1)  # Simulate query time
        return f"Result for {query}"
```

### 3. Distributed Cache Coordination

```python
class DistributedCacheCoordinator:
    def __init__(self, node_id: str, expected_items: int = 1_000_000):
        self.node_id = node_id
        self.local_cache_filter = BloomFilter(expected_items, 0.01)
        self.peer_cache_filters = {}  # node_id -> BloomFilter
        self.local_cache = {}
    
    def get(self, key: str):
        """Get value with distributed cache coordination"""
        
        # Check local cache first
        if key in self.local_cache_filter and key in self.local_cache:
            return self.local_cache[key]
        
        # Check which peers might have the key
        candidate_peers = []
        for peer_id, peer_filter in self.peer_cache_filters.items():
            if key in peer_filter:
                candidate_peers.append(peer_id)
        
        # Query candidate peers
        for peer_id in candidate_peers:
            value = self._query_peer(peer_id, key)
            if value is not None:
                # Cache locally for future use
                self._add_to_local_cache(key, value)
                return value
        
        # Not found in any cache, fetch from source
        value = self._fetch_from_source(key)
        self._add_to_local_cache(key, value)
        return value
    
    def _add_to_local_cache(self, key: str, value):
        """Add to local cache and update filter"""
        self.local_cache[key] = value
        self.local_cache_filter.add(key)
        
        # Broadcast filter update to peers
        self._broadcast_filter_update()
    
    def _query_peer(self, peer_id: str, key: str):
        """Query peer for key (network call)"""
        # Simulate network call
        pass
    
    def _fetch_from_source(self, key: str):
        """Fetch from original data source"""
        # Simulate expensive operation
        pass
    
    def _broadcast_filter_update(self):
        """Broadcast updated filter to peers"""
        # Send filter to all peers
        pass
    
    def receive_peer_filter(self, peer_id: str, peer_filter: BloomFilter):
        """Receive updated filter from peer"""
        self.peer_cache_filters[peer_id] = peer_filter
```

### 4. Content Delivery Network (CDN)

```python
class CDNNode:
    def __init__(self, node_location: str, cache_capacity: int = 10_000):
        self.location = node_location
        self.content_filter = BloomFilter(cache_capacity, 0.01)
        self.content_cache = {}
        self.peer_filters = {}  # location -> BloomFilter
    
    def get_content(self, content_id: str):
        """Get content with CDN optimization"""
        
        # Check local cache
        if content_id in self.content_filter:
            if content_id in self.content_cache:
                return self.content_cache[content_id]
        
        # Find nearest peer that might have content
        nearest_peer = self._find_nearest_peer_with_content(content_id)
        
        if nearest_peer:
            content = self._fetch_from_peer(nearest_peer, content_id)
            if content:
                self._cache_content(content_id, content)
                return content
        
        # Fetch from origin server
        content = self._fetch_from_origin(content_id)
        self._cache_content(content_id, content)
        return content
    
    def _find_nearest_peer_with_content(self, content_id: str):
        """Find nearest peer that might have content"""
        candidates = []
        
        for location, peer_filter in self.peer_filters.items():
            if content_id in peer_filter:
                distance = self._calculate_distance(self.location, location)
                candidates.append((distance, location))
        
        if candidates:
            candidates.sort()  # Sort by distance
            return candidates[0][1]  # Return nearest location
        
        return None
    
    def _cache_content(self, content_id: str, content):
        """Cache content locally"""
        self.content_filter.add(content_id)
        self.content_cache[content_id] = content
        
        # Update peers about new content
        self._notify_peers_of_new_content()
    
    def _calculate_distance(self, loc1: str, loc2: str) -> float:
        """Calculate distance between locations"""
        # Simplified distance calculation
        return abs(hash(loc1) - hash(loc2)) % 1000
    
    def _fetch_from_peer(self, peer_location: str, content_id: str):
        """Fetch content from peer CDN node"""
        pass
    
    def _fetch_from_origin(self, content_id: str):
        """Fetch content from origin server"""
        pass
    
    def _notify_peers_of_new_content(self):
        """Notify peers of updated content filter"""
        pass
```

## Trade-offs and Limitations

### Advantages
- **Space efficient**: Much smaller than hash sets
- **Fast operations**: O(k) time complexity
- **Cache friendly**: Sequential memory access patterns
- **Scalable**: Performance doesn't degrade with size
- **Parallelizable**: Hash functions can be computed in parallel

### Disadvantages
- **False positives**: Can incorrectly report presence
- **No deletions**: Standard version doesn't support removal
- **Fixed size**: Cannot resize after creation
- **Parameter sensitivity**: Poor parameter choice affects performance
- **Hash function dependency**: Quality depends on hash function choice

### Performance Comparison

```python
import time
import sys

def compare_data_structures():
    """Compare Bloom filter with other data structures"""
    
    items = [f"item_{i}" for i in range(100_000)]
    test_items = [f"test_{i}" for i in range(10_000)]
    
    # Test data structures
    structures = {
        'set': set(),
        'bloom_filter': BloomFilter(100_000, 0.01),
        'dict': {},
    }
    
    # Measure insertion time and memory
    results = {}
    
    for name, structure in structures.items():
        start_time = time.time()
        start_memory = sys.getsizeof(structure)
        
        if name == 'bloom_filter':
            for item in items:
                structure.add(item)
        elif name == 'set':
            for item in items:
                structure.add(item)
        elif name == 'dict':
            for item in items:
                structure[item] = True
        
        end_time = time.time()
        end_memory = sys.getsizeof(structure)
        
        # Measure lookup time
        lookup_start = time.time()
        for item in test_items:
            _ = item in structure
        lookup_end = time.time()
        
        results[name] = {
            'insertion_time': end_time - start_time,
            'lookup_time': lookup_end - lookup_start,
            'memory_usage': end_memory - start_memory,
        }
    
    # Print results
    for name, metrics in results.items():
        print(f"{name}:")
        print(f"  Insertion time: {metrics['insertion_time']:.4f}s")
        print(f"  Lookup time: {metrics['lookup_time']:.4f}s")
        print(f"  Memory usage: {metrics['memory_usage']:,} bytes")
        print()

# Run comparison
compare_data_structures()
```

### When NOT to Use Bloom Filters

1. **Exact membership required**: When false positives are unacceptable
2. **Frequent deletions needed**: Standard Bloom filters don't support deletions
3. **Small datasets**: Overhead may not be worth it for small sets
4. **Dynamic sizing**: When the number of elements is highly variable
5. **Complex queries**: Bloom filters only support membership testing

## Interview Questions & Answers

### Basic Questions

**Q1: What is a Bloom filter and what are its key properties?**

**Answer:** A Bloom filter is a space-efficient probabilistic data structure used to test whether an element is a member of a set. Key properties:
- **Space-efficient**: Uses fixed-size bit array regardless of element count
- **Probabilistic**: Can have false positives but never false negatives
- **Fast operations**: O(k) time for insertion and lookup (k = number of hash functions)
- **No deletions**: Standard Bloom filters don't support element removal
- **No element retrieval**: Can only test membership, not retrieve elements

**Q2: Explain the difference between false positives and false negatives in Bloom filters.**

**Answer:**
- **False Positive**: Filter says element is present when it's actually not in the set. This can happen due to hash collisions.
- **False Negative**: Filter says element is not present when it's actually in the set. This NEVER happens in Bloom filters.

**Example**: If we add "apple" to a Bloom filter:
- Querying "apple" will always return True (no false negatives)
- Querying "banana" might return True even if we never added it (false positive)

**Q3: How do you calculate the optimal parameters for a Bloom filter?**

**Answer:** Given n (expected items) and p (desired false positive rate):

```python
import math

def calculate_bloom_parameters(n, p):
    # Optimal bit array size
    m = int(-n * math.log(p) / (math.log(2) ** 2))
    
    # Optimal number of hash functions
    k = int(m * math.log(2) / n)
    
    return m, k

# Example: 1M items, 1% false positive rate
m, k = calculate_bloom_parameters(1_000_000, 0.01)
print(f"Bit array size: {m}, Hash functions: {k}")
# Output: Bit array size: 9585059, Hash functions: 6
```

### Intermediate Questions

**Q4: How would you implement a distributed Bloom filter across multiple servers?**

**Answer:** Several approaches:

1. **Replicated Bloom Filters**: Each server maintains identical copy
```python
class ReplicatedBloomFilter:
    def __init__(self, servers, expected_items, fp_rate):
        self.servers = servers
        self.local_filter = BloomFilter(expected_items, fp_rate)
    
    def add(self, item):
        # Add to local filter
        self.local_filter.add(item)
        
        # Replicate to all servers
        for server in self.servers:
            server.add_to_bloom_filter(item)
    
    def contains(self, item):
        # Check local filter first (fastest)
        return self.local_filter.contains(item)
```

2. **Partitioned Bloom Filters**: Distribute items across servers
```python
class PartitionedBloomFilter:
    def __init__(self, servers, expected_items, fp_rate):
        self.servers = servers
        self.filters = {
            server: BloomFilter(expected_items // len(servers), fp_rate)
            for server in servers
        }
    
    def _get_server(self, item):
        return self.servers[hash(item) % len(self.servers)]
    
    def add(self, item):
        server = self._get_server(item)
        self.filters[server].add(item)
    
    def contains(self, item):
        server = self._get_server(item)
        return self.filters[server].contains(item)
```

3. **Hierarchical Bloom Filters**: Multiple levels with different granularities

**Q5: What are the limitations of Bloom filters and when shouldn't you use them?**

**Answer:** Limitations:
1. **False positives**: Can incorrectly report presence
2. **No deletions**: Standard version doesn't support removal
3. **Fixed size**: Cannot resize after creation
4. **No element retrieval**: Can't get actual elements back
5. **Parameter sensitivity**: Poor parameters affect performance

**Don't use when:**
- Exact membership is required (no false positives acceptable)
- Frequent deletions are needed
- You need to retrieve actual elements
- Dataset size is highly variable
- Memory is not a constraint and hash sets work fine

**Q6: How do Counting Bloom Filters work and when would you use them?**

**Answer:** Counting Bloom Filters use counters instead of bits to support deletions:

```python
class CountingBloomFilter:
    def __init__(self, expected_items, fp_rate, counter_size=4):
        self.size = self._calculate_size(expected_items, fp_rate)
        self.hash_count = self._calculate_hash_count(self.size, expected_items)
        self.counters = [0] * self.size
        self.max_count = (2 ** counter_size) - 1
    
    def add(self, item):
        for hash_val in self._hash(item):
            if self.counters[hash_val] < self.max_count:
                self.counters[hash_val] += 1
    
    def remove(self, item):
        # Check if item might be present
        if not self.contains(item):
            return False
        
        # Decrement counters
        for hash_val in self._hash(item):
            if self.counters[hash_val] > 0:
                self.counters[hash_val] -= 1
        return True
    
    def contains(self, item):
        return all(self.counters[h] > 0 for h in self._hash(item))
```

**Use when:** You need deletion support and can tolerate higher memory usage.

### Advanced Questions

**Q7: Design a Bloom filter for a web crawler to avoid duplicate URLs with 10 billion URLs.**

**Answer:** Architecture considerations:

1. **Scale**: 10B URLs, assume 1% false positive rate acceptable
2. **Memory**: ~115 GB for single Bloom filter (too large)
3. **Solution**: Hierarchical/Distributed approach

```python
class WebCrawlerBloomFilter:
    def __init__(self):
        # Tier 1: In-memory recent URLs (last 24 hours)
        self.recent_filter = BloomFilter(100_000_000, 0.001)  # 100M URLs
        
        # Tier 2: Domain-based partitioned filters
        self.domain_filters = {}  # domain -> BloomFilter
        
        # Tier 3: Persistent storage for long-term deduplication
        self.persistent_storage = PersistentBloomFilter()
    
    def should_crawl(self, url):
        domain = self._extract_domain(url)
        
        # Check recent URLs first (fastest)
        if url in self.recent_filter:
            return False
        
        # Check domain-specific filter
        if domain in self.domain_filters:
            if url in self.domain_filters[domain]:
                return False
        
        # Check persistent storage (slowest)
        if self.persistent_storage.contains(url):
            return False
        
        return True
    
    def mark_crawled(self, url):
        domain = self._extract_domain(url)
        
        # Add to all tiers
        self.recent_filter.add(url)
        
        if domain not in self.domain_filters:
            self.domain_filters[domain] = BloomFilter(10_000_000, 0.01)
        self.domain_filters[domain].add(url)
        
        self.persistent_storage.add_async(url)
```

**Q8: How would you handle Bloom filter serialization and persistence?**

**Answer:** Implementation for persistence:

```python
import pickle
import json
from typing import Dict, Any

class PersistentBloomFilter:
    def __init__(self, expected_items, fp_rate, storage_path):
        self.storage_path = storage_path
        self.bloom_filter = BloomFilter(expected_items, fp_rate)
        self.metadata = {
            'expected_items': expected_items,
            'fp_rate': fp_rate,
            'items_added': 0,
            'created_at': time.time()
        }
        self._load_from_disk()
    
    def save_to_disk(self):
        """Save Bloom filter to disk"""
        data = {
            'bit_array': self.bloom_filter.bit_array,
            'size': self.bloom_filter.size,
            'hash_count': self.bloom_filter.hash_count,
            'metadata': self.metadata
        }
        
        with open(self.storage_path, 'wb') as f:
            pickle.dump(data, f)
    
    def _load_from_disk(self):
        """Load Bloom filter from disk"""
        try:
            with open(self.storage_path, 'rb') as f:
                data = pickle.load(f)
                
            self.bloom_filter.bit_array = data['bit_array']
            self.bloom_filter.size = data['size']
            self.bloom_filter.hash_count = data['hash_count']
            self.metadata = data['metadata']
            
        except FileNotFoundError:
            # File doesn't exist, start fresh
            pass
    
    def add(self, item):
        self.bloom_filter.add(item)
        self.metadata['items_added'] += 1
        
        # Periodic saves to avoid data loss
        if self.metadata['items_added'] % 10000 == 0:
            self.save_to_disk()
    
    def contains(self, item):
        return self.bloom_filter.contains(item)
    
    def get_stats(self):
        current_fp_rate = self.bloom_filter.current_false_positive_rate()
        return {
            **self.metadata,
            'current_fp_rate': current_fp_rate,
            'memory_usage_mb': len(self.bloom_filter.bit_array) / (8 * 1024 * 1024)
        }
```

**Q9: Compare Bloom filters with Cuckoo filters. When would you choose each?**

**Answer:** Comparison:

| Feature | Bloom Filter | Cuckoo Filter |
|---------|-------------|---------------|
| **False Positives** | Yes | Yes |
| **False Negatives** | No | No |
| **Deletions** | No (standard) | Yes |
| **Space Efficiency** | Better for low FP rates | Better for high FP rates |
| **Lookup Performance** | O(k) | O(1) average |
| **Implementation** | Simpler | More complex |

```python
# Choose Bloom Filter when:
class BloomFilterUseCase:
    """
    - Write-heavy workloads (no deletions needed)
    - Very low false positive rates required (< 1%)
    - Simple implementation preferred
    - Memory is extremely constrained
    """
    pass

# Choose Cuckoo Filter when:
class CuckooFilterUseCase:
    """
    - Deletions are required
    - Higher false positive rates acceptable (1-10%)
    - Lookup performance is critical
    - Dynamic datasets with changing membership
    """
    pass
```

**Q10: How would you monitor and optimize Bloom filter performance in production?**

**Answer:** Comprehensive monitoring strategy:

```python
class BloomFilterMonitor:
    def __init__(self, bloom_filter):
        self.bloom_filter = bloom_filter
        self.metrics = {
            'total_adds': 0,
            'total_queries': 0,
            'estimated_false_positives': 0,
            'memory_usage': 0
        }
        self.performance_history = []
    
    def record_add(self, item):
        self.bloom_filter.add(item)
        self.metrics['total_adds'] += 1
        self._update_metrics()
    
    def record_query(self, item, result, actual_presence=None):
        self.metrics['total_queries'] += 1
        
        # Track false positives if we know actual presence
        if actual_presence is not None:
            if result and not actual_presence:
                self.metrics['estimated_false_positives'] += 1
    
    def _update_metrics(self):
        # Calculate current false positive rate
        current_fp_rate = self.bloom_filter.current_false_positive_rate()
        
        # Memory usage
        memory_usage = len(self.bloom_filter.bit_array) / (8 * 1024 * 1024)  # MB
        
        self.performance_history.append({
            'timestamp': time.time(),
            'items_added': self.metrics['total_adds'],
            'fp_rate': current_fp_rate,
            'memory_usage_mb': memory_usage
        })
        
        # Alert if false positive rate exceeds threshold
        if current_fp_rate > self.bloom_filter.false_positive_rate * 1.5:
            self._trigger_alert('high_fp_rate', current_fp_rate)
    
    def _trigger_alert(self, alert_type, value):
        print(f"ALERT: {alert_type} = {value}")
        # In production: send to monitoring system
    
    def get_optimization_recommendations(self):
        recommendations = []
        
        current_fp_rate = self.bloom_filter.current_false_positive_rate()
        target_fp_rate = self.bloom_filter.false_positive_rate
        
        if current_fp_rate > target_fp_rate * 2:
            recommendations.append({
                'type': 'resize_filter',
                'reason': 'False positive rate too high',
                'current_fp': current_fp_rate,
                'target_fp': target_fp_rate
            })
        
        if self.metrics['total_adds'] > self.bloom_filter.expected_items:
            recommendations.append({
                'type': 'scale_up',
                'reason': 'More items than expected',
                'current_items': self.metrics['total_adds'],
                'expected_items': self.bloom_filter.expected_items
            })
        
        return recommendations
```

### System Design Questions

**Q11: Design a content recommendation system using Bloom filters to avoid showing duplicate content.**

**Answer:** Multi-layer Bloom filter architecture:

```python
class ContentRecommendationSystem:
    def __init__(self):
        # Per-user Bloom filters for recently shown content
        self.user_filters = {}  # user_id -> BloomFilter
        
        # Global Bloom filter for trending/popular content
        self.trending_filter = BloomFilter(10_000_000, 0.01)
        
        # Category-specific filters
        self.category_filters = {
            'news': BloomFilter(1_000_000, 0.01),
            'sports': BloomFilter(500_000, 0.01),
            'tech': BloomFilter(800_000, 0.01)
        }
    
    def get_recommendations(self, user_id, category, candidate_content):
        """Filter out content user has already seen"""
        
        # Get or create user filter
        if user_id not in self.user_filters:
            self.user_filters[user_id] = BloomFilter(100_000, 0.01)
        
        user_filter = self.user_filters[user_id]
        category_filter = self.category_filters.get(category)
        
        recommendations = []
        
        for content in candidate_content:
            content_id = content['id']
            
            # Skip if user has seen this content
            if content_id in user_filter:
                continue
            
            # Skip if it's trending (user might have seen elsewhere)
            if content_id in self.trending_filter:
                continue
            
            # Skip if overexposed in category
            if category_filter and content_id in category_filter:
                continue
            
            recommendations.append(content)
        
        return recommendations
    
    def record_content_shown(self, user_id, content_id, category):
        """Record that content was shown to user"""
        
        # Add to user filter
        if user_id in self.user_filters:
            self.user_filters[user_id].add(content_id)
        
        # Add to category filter
        if category in self.category_filters:
            self.category_filters[category].add(content_id)
    
    def mark_trending(self, content_id):
        """Mark content as trending"""
        self.trending_filter.add(content_id)
```

**Q12: How would you use Bloom filters in a distributed caching system?**

**Answer:** Bloom filters for cache coordination:

```python
class DistributedCacheWithBloomFilter:
    def __init__(self, node_id, peer_nodes):
        self.node_id = node_id
        self.peer_nodes = peer_nodes
        
        # Local cache and its Bloom filter
        self.local_cache = {}
        self.local_bloom_filter = BloomFilter(1_000_000, 0.01)
        
        # Bloom filters from peer nodes
        self.peer_bloom_filters = {}
        
        # Negative cache (items definitely not in any cache)
        self.negative_bloom_filter = BloomFilter(10_000_000, 0.01)
    
    def get(self, key):
        """Get value with Bloom filter optimization"""
        
        # Check local cache first
        if key in self.local_cache:
            return self.local_cache[key]
        
        # Check if definitely not cached anywhere
        if key in self.negative_bloom_filter:
            return self._fetch_from_source(key)
        
        # Check which peers might have the key
        candidate_peers = []
        for peer_id, peer_filter in self.peer_bloom_filters.items():
            if key in peer_filter:
                candidate_peers.append(peer_id)
        
        # Query candidate peers
        for peer_id in candidate_peers:
            value = self._query_peer(peer_id, key)
            if value is not None:
                # Cache locally for future use
                self._cache_locally(key, value)
                return value
        
        # Not found in any cache, fetch from source
        value = self._fetch_from_source(key)
        
        # Add to negative filter if not found
        if value is None:
            self.negative_bloom_filter.add(key)
        else:
            self._cache_locally(key, value)
        
        return value
    
    def _cache_locally(self, key, value):
        """Cache item locally and update Bloom filter"""
        self.local_cache[key] = value
        self.local_bloom_filter.add(key)
        
        # Broadcast updated filter to peers
        self._broadcast_bloom_filter_update()
    
    def _broadcast_bloom_filter_update(self):
        """Send updated Bloom filter to all peers"""
        for peer_id in self.peer_nodes:
            self._send_bloom_filter_to_peer(peer_id, self.local_bloom_filter)
    
    def receive_peer_bloom_filter(self, peer_id, bloom_filter):
        """Receive updated Bloom filter from peer"""
        self.peer_bloom_filters[peer_id] = bloom_filter
```

## Best Practices

1. **Choose parameters carefully**: Balance memory usage and false positive rate
2. **Monitor false positive rate**: Track actual vs. expected false positive rate
3. **Use quality hash functions**: Poor hash functions reduce effectiveness
4. **Consider alternatives**: Evaluate Cuckoo filters for deletion support
5. **Implement proper serialization**: For persistent or distributed use cases
6. **Test thoroughly**: Verify behavior under expected load patterns
7. **Document assumptions**: Make false positive implications clear to users
