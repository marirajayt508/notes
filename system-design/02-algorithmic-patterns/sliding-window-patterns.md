# Sliding Window Patterns

Sliding window is a powerful algorithmic technique used to solve problems involving arrays, strings, or sequences where you need to find or calculate something among all contiguous subarrays/substrings of a specific size or condition. This pattern is essential in system design for implementing rate limiting, monitoring, analytics, and data processing systems.

## Table of Contents
- [What is Sliding Window?](#what-is-sliding-window)
- [Types of Sliding Window](#types-of-sliding-window)
- [Implementation Patterns](#implementation-patterns)
- [System Design Applications](#system-design-applications)
- [Advanced Techniques](#advanced-techniques)
- [Performance Optimization](#performance-optimization)
- [Real-World Examples](#real-world-examples)

## What is Sliding Window?

### Core Concept
The sliding window technique involves creating a "window" that slides over data structures to examine subsets of elements. Instead of recalculating everything for each position, we maintain the window state and update it incrementally.

### Key Benefits
- **Time Complexity**: Reduces O(n²) or O(n³) solutions to O(n)
- **Space Efficiency**: Often uses O(1) extra space
- **Real-time Processing**: Enables streaming data analysis
- **Memory Locality**: Better cache performance due to sequential access

### When to Use Sliding Window
- Finding maximum/minimum in subarrays
- Calculating running averages or sums
- Pattern matching in strings
- Rate limiting and monitoring
- Real-time analytics
- Data stream processing

## Types of Sliding Window

### 1. Fixed Size Window

The window size remains constant throughout the process.

```python
def fixed_window_maximum(arr, k):
    """Find maximum in every window of size k"""
    if not arr or k <= 0:
        return []
    
    result = []
    window_max = float('-inf')
    
    # Initialize first window
    for i in range(k):
        window_max = max(window_max, arr[i])
    result.append(window_max)
    
    # Slide the window
    for i in range(k, len(arr)):
        # Remove element going out of window
        # Add element coming into window
        # Recalculate max (naive approach)
        window_max = max(arr[i-k+1:i+1])
        result.append(window_max)
    
    return result

# Optimized version using deque
from collections import deque

def optimized_fixed_window_maximum(arr, k):
    """Optimized O(n) solution using deque"""
    if not arr or k <= 0:
        return []
    
    dq = deque()  # Store indices
    result = []
    
    for i in range(len(arr)):
        # Remove indices outside current window
        while dq and dq[0] <= i - k:
            dq.popleft()
        
        # Remove indices of smaller elements
        while dq and arr[dq[-1]] <= arr[i]:
            dq.pop()
        
        dq.append(i)
        
        # Add to result if window is complete
        if i >= k - 1:
            result.append(arr[dq[0]])
    
    return result
```

### 2. Variable Size Window

The window size changes based on certain conditions.

```python
def longest_substring_without_repeating(s):
    """Find longest substring without repeating characters"""
    if not s:
        return 0
    
    char_set = set()
    left = 0
    max_length = 0
    
    for right in range(len(s)):
        # Shrink window until no duplicates
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1
        
        # Add current character
        char_set.add(s[right])
        max_length = max(max_length, right - left + 1)
    
    return max_length

def minimum_window_substring(s, t):
    """Find minimum window substring containing all characters of t"""
    if not s or not t:
        return ""
    
    # Count characters in t
    dict_t = {}
    for char in t:
        dict_t[char] = dict_t.get(char, 0) + 1
    
    required = len(dict_t)
    left = right = 0
    formed = 0
    
    window_counts = {}
    ans = float('inf'), None, None
    
    while right < len(s):
        # Add character from right to window
        char = s[right]
        window_counts[char] = window_counts.get(char, 0) + 1
        
        if char in dict_t and window_counts[char] == dict_t[char]:
            formed += 1
        
        # Try to contract window
        while left <= right and formed == required:
            char = s[left]
            
            # Update answer if this window is smaller
            if right - left + 1 < ans[0]:
                ans = (right - left + 1, left, right)
            
            # Remove from left
            window_counts[char] -= 1
            if char in dict_t and window_counts[char] < dict_t[char]:
                formed -= 1
            
            left += 1
        
        right += 1
    
    return "" if ans[0] == float('inf') else s[ans[1]:ans[2] + 1]
```

### 3. Multi-Window Patterns

Using multiple windows simultaneously.

```python
class MultiWindowAnalyzer:
    """Analyze data using multiple sliding windows"""
    
    def __init__(self, window_sizes):
        self.window_sizes = window_sizes
        self.windows = {size: deque() for size in window_sizes}
        self.sums = {size: 0 for size in window_sizes}
    
    def add_value(self, value):
        """Add value to all windows"""
        results = {}
        
        for size in self.window_sizes:
            window = self.windows[size]
            
            # Add new value
            window.append(value)
            self.sums[size] += value
            
            # Remove old values if window is full
            if len(window) > size:
                old_value = window.popleft()
                self.sums[size] -= old_value
            
            # Calculate metrics
            if len(window) == size:
                results[f'avg_{size}'] = self.sums[size] / size
                results[f'sum_{size}'] = self.sums[size]
                results[f'max_{size}'] = max(window)
                results[f'min_{size}'] = min(window)
        
        return results

# Usage example
analyzer = MultiWindowAnalyzer([5, 10, 20])
for i in range(25):
    metrics = analyzer.add_value(i)
    if metrics:
        print(f"Value {i}: {metrics}")
```

## Implementation Patterns

### 1. Two Pointer Technique

```python
class TwoPointerSlidingWindow:
    """Template for two-pointer sliding window problems"""
    
    @staticmethod
    def find_subarray_with_sum(arr, target_sum):
        """Find subarray with given sum"""
        left = 0
        current_sum = 0
        
        for right in range(len(arr)):
            current_sum += arr[right]
            
            # Shrink window if sum exceeds target
            while current_sum > target_sum and left <= right:
                current_sum -= arr[left]
                left += 1
            
            # Check if we found the target
            if current_sum == target_sum:
                return arr[left:right + 1]
        
        return None
    
    @staticmethod
    def max_sum_subarray_size_k(arr, k):
        """Maximum sum of subarray of size k"""
        if len(arr) < k:
            return None
        
        # Calculate sum of first window
        window_sum = sum(arr[:k])
        max_sum = window_sum
        
        # Slide the window
        for i in range(k, len(arr)):
            window_sum = window_sum - arr[i - k] + arr[i]
            max_sum = max(max_sum, window_sum)
        
        return max_sum
```

### 2. Hash Map Based Window

```python
class HashMapSlidingWindow:
    """Using hash maps to track window state"""
    
    @staticmethod
    def longest_substring_k_distinct(s, k):
        """Longest substring with at most k distinct characters"""
        if not s or k == 0:
            return 0
        
        char_count = {}
        left = 0
        max_length = 0
        
        for right in range(len(s)):
            # Add character to window
            char = s[right]
            char_count[char] = char_count.get(char, 0) + 1
            
            # Shrink window if too many distinct characters
            while len(char_count) > k:
                left_char = s[left]
                char_count[left_char] -= 1
                if char_count[left_char] == 0:
                    del char_count[left_char]
                left += 1
            
            max_length = max(max_length, right - left + 1)
        
        return max_length
    
    @staticmethod
    def character_replacement(s, k):
        """Longest substring with at most k character replacements"""
        char_count = {}
        left = 0
        max_length = 0
        max_count = 0
        
        for right in range(len(s)):
            char = s[right]
            char_count[char] = char_count.get(char, 0) + 1
            max_count = max(max_count, char_count[char])
            
            # If window size - max_count > k, shrink window
            if right - left + 1 - max_count > k:
                left_char = s[left]
                char_count[left_char] -= 1
                left += 1
            
            max_length = max(max_length, right - left + 1)
        
        return max_length
```

### 3. Deque-Based Window

```python
from collections import deque

class DequeSlidingWindow:
    """Using deque for efficient window operations"""
    
    def __init__(self, window_size):
        self.window_size = window_size
        self.window = deque()
        self.min_deque = deque()  # Indices of potential minimums
        self.max_deque = deque()  # Indices of potential maximums
    
    def add_element(self, value):
        """Add element and maintain window"""
        index = len(self.window)
        self.window.append(value)
        
        # Maintain min deque
        while self.min_deque and self.window[self.min_deque[-1]] >= value:
            self.min_deque.pop()
        self.min_deque.append(index)
        
        # Maintain max deque
        while self.max_deque and self.window[self.max_deque[-1]] <= value:
            self.max_deque.pop()
        self.max_deque.append(index)
        
        # Remove elements outside window
        if len(self.window) > self.window_size:
            self.window.popleft()
            
            # Update deques
            if self.min_deque and self.min_deque[0] <= index - self.window_size:
                self.min_deque.popleft()
            if self.max_deque and self.max_deque[0] <= index - self.window_size:
                self.max_deque.popleft()
    
    def get_min(self):
        """Get minimum in current window"""
        if self.min_deque:
            return self.window[self.min_deque[0]]
        return None
    
    def get_max(self):
        """Get maximum in current window"""
        if self.max_deque:
            return self.window[self.max_deque[0]]
        return None
    
    def get_range(self):
        """Get range (max - min) in current window"""
        min_val = self.get_min()
        max_val = self.get_max()
        if min_val is not None and max_val is not None:
            return max_val - min_val
        return None
```

## System Design Applications

### 1. Rate Limiting with Sliding Window

```python
import time
from collections import deque

class SlidingWindowRateLimit:
    """Rate limiter using sliding window"""
    
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.requests = deque()  # Store request timestamps
    
    def is_allowed(self, user_id=None):
        """Check if request is allowed"""
        current_time = time.time()
        
        # Remove old requests outside window
        while self.requests and self.requests[0] <= current_time - self.window_size:
            self.requests.popleft()
        
        # Check if we can accept new request
        if len(self.requests) < self.max_requests:
            self.requests.append(current_time)
            return True
        
        return False
    
    def get_retry_after(self):
        """Get seconds until next request is allowed"""
        if not self.requests:
            return 0
        
        current_time = time.time()
        oldest_request = self.requests[0]
        return max(0, self.window_size - (current_time - oldest_request))

# Distributed version using Redis
import redis
import json

class DistributedSlidingWindowRateLimit:
    """Distributed rate limiter using Redis"""
    
    def __init__(self, redis_client, max_requests, window_size_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_size = window_size_seconds
    
    def is_allowed(self, user_id):
        """Check if request is allowed for user"""
        current_time = time.time()
        key = f"rate_limit:{user_id}"
        
        # Use Redis pipeline for atomic operations
        pipe = self.redis.pipeline()
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, current_time - self.window_size)
        
        # Count current requests
        pipe.zcard(key)
        
        # Add current request
        pipe.zadd(key, {str(current_time): current_time})
        
        # Set expiration
        pipe.expire(key, int(self.window_size) + 1)
        
        results = pipe.execute()
        request_count = results[1]
        
        return request_count < self.max_requests
```

### 2. Real-Time Analytics with Sliding Windows

```python
import time
from collections import defaultdict, deque
from threading import Lock

class RealTimeAnalytics:
    """Real-time analytics using multiple sliding windows"""
    
    def __init__(self):
        self.windows = {
            '1m': {'size': 60, 'data': deque()},
            '5m': {'size': 300, 'data': deque()},
            '1h': {'size': 3600, 'data': deque()},
        }
        self.metrics = defaultdict(lambda: defaultdict(float))
        self.lock = Lock()
    
    def record_event(self, event_type, value=1, timestamp=None):
        """Record an event with timestamp"""
        if timestamp is None:
            timestamp = time.time()
        
        event = {
            'type': event_type,
            'value': value,
            'timestamp': timestamp
        }
        
        with self.lock:
            for window_name, window_config in self.windows.items():
                window_data = window_config['data']
                window_size = window_config['size']
                
                # Add new event
                window_data.append(event)
                
                # Remove old events
                cutoff_time = timestamp - window_size
                while window_data and window_data[0]['timestamp'] < cutoff_time:
                    window_data.popleft()
    
    def get_metrics(self, window='1m'):
        """Get aggregated metrics for a time window"""
        if window not in self.windows:
            return {}
        
        with self.lock:
            window_data = self.windows[window]['data']
            
            metrics = defaultdict(lambda: {
                'count': 0,
                'sum': 0,
                'avg': 0,
                'max': float('-inf'),
                'min': float('inf')
            })
            
            for event in window_data:
                event_type = event['type']
                value = event['value']
                
                metrics[event_type]['count'] += 1
                metrics[event_type]['sum'] += value
                metrics[event_type]['max'] = max(metrics[event_type]['max'], value)
                metrics[event_type]['min'] = min(metrics[event_type]['min'], value)
            
            # Calculate averages
            for event_type in metrics:
                if metrics[event_type]['count'] > 0:
                    metrics[event_type]['avg'] = (
                        metrics[event_type]['sum'] / metrics[event_type]['count']
                    )
                else:
                    metrics[event_type]['min'] = 0
                    metrics[event_type]['max'] = 0
            
            return dict(metrics)
    
    def get_rate(self, event_type, window='1m'):
        """Get rate of events per second"""
        metrics = self.get_metrics(window)
        if event_type in metrics:
            window_size = self.windows[window]['size']
            return metrics[event_type]['count'] / window_size
        return 0

# Usage example
analytics = RealTimeAnalytics()

# Record events
analytics.record_event('page_view', 1)
analytics.record_event('api_call', 1)
analytics.record_event('error', 1)

# Get metrics
print(analytics.get_metrics('1m'))
print(f"Page view rate: {analytics.get_rate('page_view', '1m')} per second")
```

### 3. System Monitoring with Sliding Windows

```python
import time
import statistics
from collections import deque
from dataclasses import dataclass
from typing import List, Dict, Optional

@dataclass
class MetricPoint:
    timestamp: float
    value: float
    tags: Dict[str, str] = None

class SlidingWindowMonitor:
    """System monitoring using sliding windows"""
    
    def __init__(self, window_sizes: List[int]):
        self.window_sizes = window_sizes
        self.metrics = {}  # metric_name -> window_size -> deque
        self.alerts = {}   # metric_name -> alert_config
    
    def add_metric_point(self, metric_name: str, value: float, 
                        timestamp: Optional[float] = None, 
                        tags: Optional[Dict[str, str]] = None):
        """Add a metric point"""
        if timestamp is None:
            timestamp = time.time()
        
        point = MetricPoint(timestamp, value, tags or {})
        
        if metric_name not in self.metrics:
            self.metrics[metric_name] = {
                size: deque() for size in self.window_sizes
            }
        
        # Add to all windows
        for window_size in self.window_sizes:
            window = self.metrics[metric_name][window_size]
            window.append(point)
            
            # Remove old points
            cutoff_time = timestamp - window_size
            while window and window[0].timestamp < cutoff_time:
                window.popleft()
        
        # Check alerts
        self._check_alerts(metric_name, timestamp)
    
    def get_statistics(self, metric_name: str, window_size: int) -> Dict:
        """Get statistics for a metric in a time window"""
        if (metric_name not in self.metrics or 
            window_size not in self.metrics[metric_name]):
            return {}
        
        window = self.metrics[metric_name][window_size]
        if not window:
            return {}
        
        values = [point.value for point in window]
        
        return {
            'count': len(values),
            'sum': sum(values),
            'avg': statistics.mean(values),
            'median': statistics.median(values),
            'min': min(values),
            'max': max(values),
            'std': statistics.stdev(values) if len(values) > 1 else 0,
            'p95': statistics.quantiles(values, n=20)[18] if len(values) >= 20 else max(values),
            'p99': statistics.quantiles(values, n=100)[98] if len(values) >= 100 else max(values)
        }
    
    def set_alert(self, metric_name: str, condition: str, 
                  threshold: float, window_size: int):
        """Set alert condition for a metric"""
        self.alerts[metric_name] = {
            'condition': condition,  # 'avg_gt', 'max_gt', 'count_gt', etc.
            'threshold': threshold,
            'window_size': window_size
        }
    
    def _check_alerts(self, metric_name: str, timestamp: float):
        """Check if any alerts should be triggered"""
        if metric_name not in self.alerts:
            return
        
        alert_config = self.alerts[metric_name]
        stats = self.get_statistics(metric_name, alert_config['window_size'])
        
        if not stats:
            return
        
        condition = alert_config['condition']
        threshold = alert_config['threshold']
        
        triggered = False
        
        if condition == 'avg_gt' and stats['avg'] > threshold:
            triggered = True
        elif condition == 'max_gt' and stats['max'] > threshold:
            triggered = True
        elif condition == 'count_gt' and stats['count'] > threshold:
            triggered = True
        elif condition == 'p95_gt' and stats['p95'] > threshold:
            triggered = True
        
        if triggered:
            self._trigger_alert(metric_name, condition, stats, timestamp)
    
    def _trigger_alert(self, metric_name: str, condition: str, 
                      stats: Dict, timestamp: float):
        """Trigger an alert"""
        print(f"ALERT: {metric_name} {condition} - {stats} at {timestamp}")
        # In real implementation, send to alerting system

# Usage example
monitor = SlidingWindowMonitor([60, 300, 3600])  # 1m, 5m, 1h windows

# Set alerts
monitor.set_alert('cpu_usage', 'avg_gt', 80.0, 300)  # Alert if avg CPU > 80% in 5m
monitor.set_alert('error_rate', 'count_gt', 100, 60)  # Alert if > 100 errors in 1m

# Add metrics
for i in range(100):
    monitor.add_metric_point('cpu_usage', 75 + i * 0.1)
    monitor.add_metric_point('memory_usage', 60 + i * 0.2)
    monitor.add_metric_point('error_rate', 1 if i % 10 == 0 else 0)
    time.sleep(0.1)

# Get statistics
print(monitor.get_statistics('cpu_usage', 60))
```

## Advanced Techniques

### 1. Segment Tree for Range Queries

```python
class SegmentTree:
    """Segment tree for efficient range queries"""
    
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self.build(arr, 0, 0, self.n - 1)
    
    def build(self, arr, node, start, end):
        """Build segment tree"""
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self.build(arr, 2 * node + 1, start, mid)
            self.build(arr, 2 * node + 2, mid + 1, end)
            self.tree[node] = max(self.tree[2 * node + 1], self.tree[2 * node + 2])
    
    def query(self, node, start, end, l, r):
        """Query maximum in range [l, r]"""
        if r < start or end < l:
            return float('-inf')
        if l <= start and end <= r:
            return self.tree[node]
        
        mid = (start + end) // 2
        left_max = self.query(2 * node + 1, start, mid, l, r)
        right_max = self.query(2 * node + 2, mid + 1, end, l, r)
        return max(left_max, right_max)
    
    def range_max(self, l, r):
        """Get maximum in range [l, r]"""
        return self.query(0, 0, self.n - 1, l, r)

class SlidingWindowWithSegmentTree:
    """Sliding window using segment tree for O(log n) queries"""
    
    def __init__(self, arr):
        self.arr = arr
        self.seg_tree = SegmentTree(arr)
    
    def max_in_windows(self, k):
        """Get maximum in all windows of size k"""
        result = []
        for i in range(len(self.arr) - k + 1):
            window_max = self.seg_tree.range_max(i, i + k - 1)
            result.append(window_max)
        return result
```

### 2. Sparse Table for Static Range Queries

```python
import math

class SparseTable:
    """Sparse table for O(1) range maximum queries"""
    
    def __init__(self, arr):
        self.arr = arr
        self.n = len(arr)
        self.k = int(math.log2(self.n)) + 1
        self.st = [[0] * self.k for _ in range(self.n)]
        self.build()
    
    def build(self):
        """Build sparse table"""
        # Initialize for intervals of length 1
        for i in range(self.n):
            self.st[i][0] = self.arr[i]
        
        # Build for intervals of length 2^j
        j = 1
        while (1 << j) <= self.n:
            i = 0
            while (i + (1 << j) - 1) < self.n:
                self.st[i][j] = max(self.st[i][j-1], 
                                   self.st[i + (1 << (j-1))][j-1])
                i += 1
            j += 1
    
    def query(self, l, r):
        """Query maximum in range [l, r] in O(1)"""
        j = int(math.log2(r - l + 1))
        return max(self.st[l][j], self.st[r - (1 << j) + 1][j])

class OptimizedSlidingWindow:
    """Optimized sliding window using sparse table"""
    
    def __init__(self, arr):
        self.arr = arr
        self.sparse_table = SparseTable(arr)
    
    def max_in_windows(self, k):
        """Get maximum in all windows of size k in O(n)"""
        result = []
        for i in range(len(self.arr) - k + 1):
            window_max = self.sparse_table.query(i, i + k - 1)
            result.append(window_max)
        return result
```

## Performance Optimization

### 1. Memory-Efficient Sliding Window

```python
class MemoryEfficientSlidingWindow:
    """Memory-efficient sliding window for large datasets"""
    
    def __init__(self, window_size, aggregation_func):
        self.window_size = window_size
        self.aggregation_func = aggregation_func
        self.buffer = []
        self.current_result = None
    
    def add_value(self, value):
        """Add value with memory management"""
        self.buffer.append(value)
        
        # Keep only necessary values
        if len(self.buffer) > self.window_size:
            self.buffer.pop(0)
        
        # Calculate result only when window is full
        if len(self.buffer) == self.window_size:
            self.current_result = self.aggregation_func(self.buffer)
        
        return self.current_result
    
    def get_current_result(self):
        """Get current aggregation result"""
        return self.current_result

# Specialized for different aggregations
class IncrementalSum:
    """Incremental sum calculation"""
    
    def __init__(self, window_size):
        self.window_size = window_size
        self.window = deque()
        self.current_sum = 0
    
    def add_value(self, value):
        """Add value and update sum incrementally"""
        self.window.append(value)
        self.current_sum += value
        
        if len(self.window) > self.window_size:
            old_value = self.window.popleft()
            self.current_sum -= old_value
        
        return self.current_sum if len(self.window) == self.window_size else None

class IncrementalAverage:
    """Incremental average calculation"""
    
    def __init__(self, window_size):
        self.sum_calculator = IncrementalSum(window_size)
        self.window_size = window_size
    
    def add_value(self, value):
        """Add value and update average incrementally"""
        current_sum = self.sum_calculator.add_value(value)
        if current_sum is not None:
            return current_sum / self.window_size
        return None
```

### 2. Parallel Sliding Window Processing

```python
import concurrent.futures
from typing import List, Callable, Any

class ParallelSlidingWindow:
    """Parallel processing of sliding windows"""
    
    def __init__(self, num_workers=4):
        self.num_workers = num_workers
    
    def process_windows(self, data: List[Any], window_size: int, 
                       process_func: Callable, overlap: int = 0) -> List[Any]:
        """Process sliding windows in parallel"""
        
        # Calculate chunk size for each worker
        total_windows = len(data) - window_size + 1
        chunk_size = max(1, total_windows // self.num_workers)
        
        # Create chunks with overlap
        chunks = []
        for i in range(0, total_windows, chunk_size):
            start_idx = i
            end_idx = min(i + chunk_size + window_size - 1, len(data))
            chunk_data = data[start_idx:end_idx]
            chunks.append((chunk_data, window_size, start_idx))
        
        # Process chunks in parallel
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_workers) as executor:
            futures = [
                executor.submit(self._process_chunk, chunk_data, window_size, 
                              start_idx, process_func)
                for chunk_data, window_size, start_idx in chunks
            ]
            
            results = []
            for future in concurrent.futures.as_completed(futures):
                chunk_results = future.result()
                results.extend(chunk_results)
        
        # Sort results by original index
        results.sort(key=lambda x: x[0])
        return [result[1] for result in results]
    
    def _process_chunk(self, chunk_data: List[Any], window_size: int, 
                      start_idx: int, process_func: Callable) -> List[tuple]:
        """Process a chunk of data"""
        results = []
        for i in range(len(chunk_data) - window_size + 1):
            window = chunk_data[i:i + window_size]
            result = process_func(window)
            results.append((start_idx + i, result))
        return results

# Usage example
def calculate_window_stats(window):
    """Calculate statistics for a window"""
    return {
        'sum': sum(window),
        'avg': sum(window) / len(window),
        'max': max(window),
        'min': min(window)
    }

# Process large dataset in parallel
parallel_processor = ParallelSlidingWindow(num_workers=4)
large_data = list(range(10000))
results = parallel_processor.process_windows(
    large_data, window_size=100, process_func=calculate_window_stats
)
```

## Real-World Examples

### 1. Stock Price Analysis

```python
import time
from collections import deque
from typing import List, Dict, Optional

class StockPriceAnalyzer:
    """Real-time stock price analysis using sliding windows"""
    
    def __init__(self, symbol: str):
        self.symbol = symbol
        self.prices = deque()
        self.volumes = deque()
        self.timestamps = deque()
        
        # Different time windows for analysis
        self.windows = {
            '1m': 60,
            '5m': 300,
            '15m': 900,
            '1h': 3600
        }
    
    def add_price_point(self, price: float, volume: int, timestamp: Optional[float] = None):
        """Add a new price point"""
        if timestamp is None:
            timestamp = time.time()
        
        self.prices.append(price)
        self.volumes.append(volume)
        self.timestamps.append(timestamp)
        
        # Keep only last hour of data
        cutoff_time = timestamp - 3600
        while self.timestamps and self.timestamps[0] < cutoff_time:
            self.prices.popleft()
            self.volumes.popleft()
            self.timestamps.popleft()
    
    def get_moving_average(self, window: str) -> Optional[float]:
        """Calculate moving average for specified window"""
        if window not in self.windows:
            return None
        
        window_size = self.windows[window]
        current_time = time.time()
        cutoff_time = current_time - window_size
        
        # Find prices within window
        window_prices = []
        for i, timestamp in enumerate(self.timestamps):
            if timestamp >= cutoff_time:
                window_prices.extend(list(self.prices)[i:])
                break
        
        if not window_prices:
            return None
        
        return sum(window_prices) / len(window_prices)
    
    def get_price_volatility(self, window: str) -> Optional[float]:
        """Calculate price volatility (standard deviation)"""
        if window not in self.windows:
            return None
        
        window_size = self.windows[window]
        current_time = time.time()
        cutoff_time = current_time - window_size
        
        # Find prices within window
        window_prices = []
        for i, timestamp in enumerate(self.timestamps):
            if timestamp >= cutoff_time:
                window_prices.extend(list(self.prices)[i:])
                break
        
        if len(window_prices) < 2:
            return None
        
        avg = sum(window_prices) / len(window_prices)
        variance = sum((price - avg) ** 2 for price in window_prices) / len(window_prices)
        return variance ** 0.5
    
    def detect_price_breakout(self, threshold_percent: float = 5.0) -> Dict:
        """Detect price breakouts using multiple timeframes"""
        current_price = self.prices[-1] if self.prices else None
        if not current_price:
            return {}
        
        breakouts = {}
        
        for window in ['5m', '15m', '1h']:
            ma = self.get_moving_average(window)
            if ma:
                percent_change = ((current_price - ma) / ma) * 100
                if abs(percent_change) > threshold_percent:
                    breakouts[window] = {
                        'direction': 'up' if percent_change > 0 else 'down',
                        'percent_change': percent_change,
                        'current_price': current_price,
                        'moving_average': ma
                    }
        
        return breakouts

# Usage example
analyzer = StockPriceAnalyzer('AAPL')

# Simulate real-time price updates
import random
base_price = 150.0
for i in range(1000):
    # Simulate price movement
    price_change = random.uniform(-2, 2)
    new_price = base_price + price_change
    volume = random.randint(1000, 10000)
    
    analyzer.add_price_point(new_price, volume)
    base_price = new_price
    
    # Check for breakouts every 10 updates
    if i % 10 == 0:
        breakouts = analyzer.detect_price_breakout()
        if breakouts:
            print(f"Breakouts detected: {breakouts}")
    
    time.sleep(0.1)
```

### 2. Network Traffic Monitoring

```python
import time
from collections import deque, defaultdict
from dataclasses import dataclass
from typing import Dict, List, Optional

@dataclass
class NetworkPacket:
    timestamp: float
    source_ip: str
    dest_ip: str
    protocol: str
    size: int
    port: int

class NetworkTrafficMonitor:
    """Monitor network traffic using sliding windows"""
    
    def __init__(self):
        self.packets = deque()
        self.window_sizes = {
            '1m': 60,
            '5m': 300,
            '15m': 900
        }
    
    def add_packet(self, packet: NetworkPacket):
        """Add a network packet"""
        self.packets.append(packet)
        
        # Keep only last 15 minutes of data
        cutoff_time = packet.timestamp - 900
        while self.packets and self.packets[0].timestamp < cutoff_time:
            self.packets.popleft()
    
    def get_traffic_stats(self, window: str) -> Dict:
        """Get traffic statistics for a time window"""
        if window not in self.window_sizes:
            return {}
        
        window_size = self.window_sizes[window]
        current_time = time.time()
        cutoff_time = current_time - window_size
        
        # Filter packets in window
        window_packets = [p for p in self.packets if p.timestamp >= cutoff_time]
        
        if not window_packets:
            return {}
        
        # Calculate statistics
        total_packets = len(window_packets)
        total_bytes = sum(p.size for p in window_packets)
        
        # Protocol distribution
        protocol_counts = defaultdict(int)
        for packet in window_packets:
            protocol_counts[packet.protocol] += 1
        
        # Top source IPs
        source_ip_counts = defaultdict(int)
        for packet in window_packets:
            source_ip_counts[packet.source_ip] += 1
        
        top_sources = sorted(source_ip_counts.items(), 
                           key=lambda x: x[1], reverse=True)[:10]
        
        # Port distribution
        port_counts = defaultdict(int)
        for packet in window_packets:
            port_counts[packet.port] += 1
        
        top_ports = sorted(port_counts.items(), 
                         key=lambda x: x[1], reverse=True)[:10]
        
        return {
            'total_packets': total_packets,
            'total_bytes': total_bytes,
            'packets_per_second': total_packets / window_size,
            'bytes_per_second': total_bytes / window_size,
            'protocol_distribution': dict(protocol_counts),
            'top_source_ips': top_sources,
            'top_ports': top_ports
        }
    
    def detect_anomalies(self) -> List[Dict]:
        """Detect traffic anomalies"""
        anomalies = []
        
        # Get current stats
        current_stats = self.get_traffic_stats('1m')
        baseline_stats = self.get_traffic_stats('15m')
        
        if not current_stats or not baseline_stats:
            return anomalies
        
        # Check for traffic spikes
        current_pps = current_stats['packets_per_second']
        baseline_pps = baseline_stats['packets_per_second']
        
        if current_pps > baseline_pps * 3:  # 3x normal traffic
            anomalies.append({
                'type': 'traffic_spike',
                'severity': 'high',
                'current_pps': current_pps,
                'baseline_pps': baseline_pps,
                'multiplier': current_pps / baseline_pps
            })
        
        # Check for unusual source IP activity
        current_top_sources = dict(current_stats['top_source_ips'])
        for ip, count in current_top_sources.items():
            if count > 100:  # More than 100 packets per minute from single IP
                anomalies.append({
                    'type': 'suspicious_source',
                    'severity': 'medium',
                    'source_ip': ip,
                    'packet_count': count
                })
        
        return anomalies
    
    def get_bandwidth_utilization(self, window: str) -> Optional[float]:
        """Calculate bandwidth utilization percentage"""
        stats = self.get_traffic_stats(window)
        if not stats:
            return None
        
        # Assume 1 Gbps link capacity
        link_capacity_bps = 1_000_000_000  # 1 Gbps in bits per second
        current_bps = stats['bytes_per_second'] * 8  # Convert bytes to bits
        
        return (current_bps / link_capacity_bps) * 100

# Usage example
monitor = NetworkTrafficMonitor()

# Simulate network packets
import random
protocols = ['TCP', 'UDP', 'ICMP']
ips = [f"192.168.1.{i}" for i in range(1, 255)]

for i in range(10000):
    packet = NetworkPacket(
        timestamp=time.time(),
        source_ip=random.choice(ips),
        dest_ip=random.choice(ips),
        protocol=random.choice(protocols),
        size=random.randint(64, 1500),
        port=random.randint(1, 65535)
    )
    
    monitor.add_packet(packet)
    
    # Check for anomalies every 100 packets
    if i % 100 == 0:
        anomalies = monitor.detect_anomalies()
        if anomalies:
            print(f"Anomalies detected: {anomalies}")
        
        # Print traffic stats
        stats = monitor.get_traffic_stats('1m')
        if stats:
            print(f"Traffic: {stats['packets_per_second']:.2f} pps, "
                  f"{stats['bytes_per_second']:.2f} Bps")
    
    time.sleep(0.01)
```

### 3. Application Performance Monitoring (APM)

```python
import time
import statistics
from collections import deque, defaultdict
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum

class LogLevel(Enum):
    DEBUG = 1
    INFO = 2
    WARN = 3
    ERROR = 4
    CRITICAL = 5

@dataclass
class LogEntry:
    timestamp: float
    level: LogLevel
    service: str
    endpoint: str
    response_time: float
    status_code: int
    user_id: Optional[str] = None
    error_message: Optional[str] = None

class APMMonitor:
    """Application Performance Monitoring using sliding windows"""
    
    def __init__(self):
        self.logs = deque()
        self.window_sizes = {
            '1m': 60,
            '5m': 300,
            '15m': 900,
            '1h': 3600
        }
        self.sla_thresholds = {
            'response_time_p95': 500,  # 500ms
            'error_rate': 0.01,        # 1%
            'availability': 0.999      # 99.9%
        }
    
    def add_log_entry(self, entry: LogEntry):
        """Add a log entry"""
        self.logs.append(entry)
        
        # Keep only last hour of data
        cutoff_time = entry.timestamp - 3600
        while self.logs and self.logs[0].timestamp < cutoff_time:
            self.logs.popleft()
    
    def get_performance_metrics(self, window: str, service: Optional[str] = None) -> Dict:
        """Get performance metrics for a time window"""
        if window not in self.window_sizes:
            return {}
        
        window_size = self.window_sizes[window]
        current_time = time.time()
        cutoff_time = current_time - window_size
        
        # Filter logs in window
        window_logs = [log for log in self.logs if log.timestamp >= cutoff_time]
        
        if service:
            window_logs = [log for log in window_logs if log.service == service]
        
        if not window_logs:
            return {}
        
        # Calculate metrics
        response_times = [log.response_time for log in window_logs]
        status_codes = [log.status_code for log in window_logs]
        
        # Error rate calculation
        error_count = sum(1 for code in status_codes if code >= 400)
        total_requests = len(status_codes)
        error_rate = error_count / total_requests if total_requests > 0 else 0
        
        # Response time percentiles
        response_times.sort()
        p50 = statistics.median(response_times) if response_times else 0
        p95 = statistics.quantiles(response_times, n=20)[18] if len(response_times) >= 20 else (max(response_times) if response_times else 0)
        p99 = statistics.quantiles(response_times, n=100)[98] if len(response_times) >= 100 else (max(response_times) if response_times else 0)
        
        # Throughput
        throughput = total_requests / window_size
        
        # Service breakdown
        service_stats = defaultdict(lambda: {'count': 0, 'errors': 0, 'response_times': []})
        for log in window_logs:
            service_stats[log.service]['count'] += 1
            if log.status_code >= 400:
                service_stats[log.service]['errors'] += 1
            service_stats[log.service]['response_times'].append(log.response_time)
        
        # Calculate per-service metrics
        for service_name in service_stats:
            stats = service_stats[service_name]
            stats['error_rate'] = stats['errors'] / stats['count'] if stats['count'] > 0 else 0
            stats['avg_response_time'] = statistics.mean(stats['response_times']) if stats['response_times'] else 0
        
        return {
            'total_requests': total_requests,
            'error_count': error_count,
            'error_rate': error_rate,
            'throughput_rps': throughput,
            'response_time': {
                'avg': statistics.mean(response_times) if response_times else 0,
                'p50': p50,
                'p95': p95,
                'p99': p99,
                'max': max(response_times) if response_times else 0
            },
            'service_breakdown': dict(service_stats)
        }
    
    def check_sla_violations(self, window: str = '5m') -> List[Dict]:
        """Check for SLA violations"""
        violations = []
        metrics = self.get_performance_metrics(window)
        
        if not metrics:
            return violations
        
        # Check response time SLA
        if metrics['response_time']['p95'] > self.sla_thresholds['response_time_p95']:
            violations.append({
                'type': 'response_time_sla',
                'severity': 'high',
                'current_p95': metrics['response_time']['p95'],
                'threshold': self.sla_thresholds['response_time_p95'],
                'window': window
            })
        
        # Check error rate SLA
        if metrics['error_rate'] > self.sla_thresholds['error_rate']:
            violations.append({
                'type': 'error_rate_sla',
                'severity': 'critical',
                'current_error_rate': metrics['error_rate'],
                'threshold': self.sla_thresholds['error_rate'],
                'window': window
            })
        
        return violations
    
    def detect_performance_anomalies(self) -> List[Dict]:
        """Detect performance anomalies"""
        anomalies = []
        
        # Compare current 1m window with 15m baseline
        current_metrics = self.get_performance_metrics('1m')
        baseline_metrics = self.get_performance_metrics('15m')
        
        if not current_metrics or not baseline_metrics:
            return anomalies
        
        # Response time anomaly
        current_p95 = current_metrics['response_time']['p95']
        baseline_p95 = baseline_metrics['response_time']['p95']
        
        if current_p95 > baseline_p95 * 2:  # 2x slower than baseline
            anomalies.append({
                'type': 'response_time_spike',
                'severity': 'high',
                'current_p95': current_p95,
                'baseline_p95': baseline_p95,
                'multiplier': current_p95 / baseline_p95 if baseline_p95 > 0 else float('inf')
            })
        
        # Error rate anomaly
        current_error_rate = current_metrics['error_rate']
        baseline_error_rate = baseline_metrics['error_rate']
        
        if current_error_rate > baseline_error_rate * 5:  # 5x more errors
            anomalies.append({
                'type': 'error_rate_spike',
                'severity': 'critical',
                'current_error_rate': current_error_rate,
                'baseline_error_rate': baseline_error_rate
            })
        
        return anomalies

# Usage example
apm = APMMonitor()

# Simulate application logs
services = ['user-service', 'order-service', 'payment-service', 'inventory-service']
endpoints = ['/api/users', '/api/orders', '/api/payments', '/api/inventory']

for i in range(10000):
    # Simulate log entry
    entry = LogEntry(
        timestamp=time.time(),
        level=random.choice(list(LogLevel)),
        service=random.choice(services),
        endpoint=random.choice(endpoints),
        response_time=random.uniform(50, 1000),  # 50ms to 1s
        status_code=random.choices([200, 201, 400, 404, 500], 
                                 weights=[85, 10, 2, 2, 1])[0],
        user_id=f"user_{random.randint(1, 1000)}"
    )
    
    apm.add_log_entry(entry)
    
    # Check for violations and anomalies every 100 entries
    if i % 100 == 0:
        violations = apm.check_sla_violations()
        anomalies = apm.detect_performance_anomalies()
        
        if violations:
            print(f"SLA Violations: {violations}")
        
        if anomalies:
            print(f"Performance Anomalies: {anomalies}")
        
        # Print current metrics
        metrics = apm.get_performance_metrics('1m')
        if metrics:
            print(f"Metrics: {metrics['throughput_rps']:.2f} RPS, "
                  f"P95: {metrics['response_time']['p95']:.2f}ms, "
                  f"Error Rate: {metrics['error_rate']:.3f}")
    
    time.sleep(0.01)
```

## Best Practices and Tips

### 1. Choosing the Right Window Type
- **Fixed windows**: Use for regular reporting and simple aggregations
- **Variable windows**: Use for pattern matching and condition-based analysis
- **Multi-windows**: Use for comparative analysis across different time scales

### 2. Memory Management
- Implement proper cleanup for old data
- Use circular buffers for fixed-size windows
- Consider data compression for long-term storage

### 3. Performance Optimization
- Use appropriate data structures (deque for FIFO, heaps for min/max)
- Implement incremental calculations when possible
- Consider parallel processing for large datasets

### 4. Error Handling
- Handle edge cases (empty windows, single elements)
- Implement proper validation for window parameters
- Add monitoring for window performance metrics

### 5. Testing Strategies
- Test with various window sizes and data patterns
- Verify correctness with known datasets
- Performance test with realistic data volumes
- Test edge cases and error conditions

## Interview Questions & Answers

### Basic Questions

**Q1: What is the sliding window technique and what problems does it solve?**

**Answer:** The sliding window technique is an algorithmic approach that maintains a "window" of elements that slides over a data structure to examine subsets efficiently. It solves problems by:

- **Reducing time complexity**: From O(n²) or O(n³) to O(n) for many problems
- **Optimizing space usage**: Often uses O(1) extra space
- **Enabling real-time processing**: Allows streaming data analysis
- **Improving cache performance**: Sequential memory access patterns

**Common problems solved:**
- Finding maximum/minimum in subarrays
- Calculating running averages or sums
- Pattern matching in strings
- Rate limiting and monitoring
- Real-time analytics

**Q2: Explain the difference between fixed-size and variable-size sliding windows.**

**Answer:**

**Fixed-size window:**
- Window size remains constant
- Example: Find maximum in every subarray of size k
```python
def max_in_fixed_window(arr, k):
    result = []
    for i in range(len(arr) - k + 1):
        window_max = max(arr[i:i+k])
        result.append(window_max)
    return result
```

**Variable-size window:**
- Window size changes based on conditions
- Example: Longest substring without repeating characters
```python
def longest_unique_substring(s):
    char_set = set()
    left = 0
    max_length = 0
    
    for right in range(len(s)):
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1
        char_set.add(s[right])
        max_length = max(max_length, right - left + 1)
    
    return max_length
```

**Q3: How would you implement a sliding window maximum using a deque?**

**Answer:** Use a deque to store indices of potential maximums:

```python
from collections import deque

def sliding_window_maximum(arr, k):
    dq = deque()  # Store indices
    result = []
    
    for i in range(len(arr)):
        # Remove indices outside current window
        while dq and dq[0] <= i - k:
            dq.popleft()
        
        # Remove indices of smaller elements
        while dq and arr[dq[-1]] <= arr[i]:
            dq.pop()
        
        dq.append(i)
        
        # Add to result if window is complete
        if i >= k - 1:
            result.append(arr[dq[0]])
    
    return result
```

**Time Complexity:** O(n) - each element is added and removed at most once
**Space Complexity:** O(k) - deque stores at most k elements

### Intermediate Questions

**Q4: Design a rate limiter using sliding window approach.**

**Answer:** Implementation using sliding window log:

```python
import time
from collections import deque

class SlidingWindowRateLimit:
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.requests = deque()  # Store request timestamps
    
    def is_allowed(self, user_id=None):
        current_time = time.time()
        
        # Remove old requests outside window
        while self.requests and self.requests[0] <= current_time - self.window_size:
            self.requests.popleft()
        
        # Check if we can accept new request
        if len(self.requests) < self.max_requests:
            self.requests.append(current_time)
            return True
        
        return False
    
    def get_retry_after(self):
        if not self.requests:
            return 0
        
        current_time = time.time()
        oldest_request = self.requests[0]
        return max(0, self.window_size - (current_time - oldest_request))

# Usage
rate_limiter = SlidingWindowRateLimit(100, 60)  # 100 requests per minute
print(rate_limiter.is_allowed())  # True or False
```

**Q5: How would you implement real-time analytics using multiple sliding windows?**

**Answer:** Multi-window analytics system:

```python
import time
from collections import deque, defaultdict

class RealTimeAnalytics:
    def __init__(self):
        self.windows = {
            '1m': {'size': 60, 'data': deque()},
            '5m': {'size': 300, 'data': deque()},
            '1h': {'size': 3600, 'data': deque()},
        }
    
    def record_event(self, event_type, value=1, timestamp=None):
        if timestamp is None:
            timestamp = time.time()
        
        event = {
            'type': event_type,
            'value': value,
            'timestamp': timestamp
        }
        
        for window_name, window_config in self.windows.items():
            window_data = window_config['data']
            window_size = window_config['size']
            
            # Add new event
            window_data.append(event)
            
            # Remove old events
            cutoff_time = timestamp - window_size
            while window_data and window_data[0]['timestamp'] < cutoff_time:
                window_data.popleft()
    
    def get_metrics(self, window='1m'):
        window_data = self.windows[window]['data']
        
        metrics = defaultdict(lambda: {
            'count': 0, 'sum': 0, 'avg': 0,
            'max': float('-inf'), 'min': float('inf')
        })
        
        for event in window_data:
            event_type = event['type']
            value = event['value']
            
            metrics[event_type]['count'] += 1
            metrics[event_type]['sum'] += value
            metrics[event_type]['max'] = max(metrics[event_type]['max'], value)
            metrics[event_type]['min'] = min(metrics[event_type]['min'], value)
        
        # Calculate averages
        for event_type in metrics:
            if metrics[event_type]['count'] > 0:
                metrics[event_type]['avg'] = (
                    metrics[event_type]['sum'] / metrics[event_type]['count']
                )
        
        return dict(metrics)
```

**Q6: What are the trade-offs between different sliding window implementations?**

**Answer:** Comparison of approaches:

| Implementation | Time Complexity | Space Complexity | Use Case |
|----------------|----------------|------------------|----------|
| **Naive (recalculate)** | O(nk) | O(1) | Small windows, simple logic |
| **Deque-based** | O(n) | O(k) | Min/max queries, optimal performance |
| **Hash map** | O(n) | O(k) | Character/element counting |
| **Segment Tree** | O(n log k) | O(k) | Complex range queries |
| **Two Pointers** | O(n) | O(1) | Sum-based problems |

**Trade-offs:**
- **Memory vs. Time**: More memory often leads to better time complexity
- **Complexity vs. Performance**: Simple solutions may be slower but easier to maintain
- **Flexibility vs. Optimization**: Generic solutions vs. problem-specific optimizations

### Advanced Questions

**Q7: Design a system monitoring solution using sliding windows for anomaly detection.**

**Answer:** Comprehensive monitoring system:

```python
import time
import statistics
from collections import deque
from dataclasses import dataclass
from typing import Dict, List, Optional

@dataclass
class MetricPoint:
    timestamp: float
    value: float
    tags: Dict[str, str] = None

class AnomalyDetectionSystem:
    def __init__(self, window_sizes: List[int]):
        self.window_sizes = window_sizes
        self.metrics = {}  # metric_name -> window_size -> deque
        self.baselines = {}  # metric_name -> baseline_stats
        self.alerts = []
    
    def add_metric_point(self, metric_name: str, value: float, 
                        timestamp: Optional[float] = None, 
                        tags: Optional[Dict[str, str]] = None):
        if timestamp is None:
            timestamp = time.time()
        
        point = MetricPoint(timestamp, value, tags or {})
        
        if metric_name not in self.metrics:
            self.metrics[metric_name] = {
                size: deque() for size in self.window_sizes
            }
        
        # Add to all windows
        for window_size in self.window_sizes:
            window = self.metrics[metric_name][window_size]
            window.append(point)
            
            # Remove old points
            cutoff_time = timestamp - window_size
            while window and window[0].timestamp < cutoff_time:
                window.popleft()
        
        # Check for anomalies
        self._detect_anomalies(metric_name, timestamp)
    
    def _detect_anomalies(self, metric_name: str, timestamp: float):
        # Compare current 1-minute window with 1-hour baseline
        current_stats = self._get_window_stats(metric_name, 60)
        baseline_stats = self._get_window_stats(metric_name, 3600)
        
        if not current_stats or not baseline_stats:
            return
        
        # Statistical anomaly detection
        current_avg = current_stats['avg']
        baseline_avg = baseline_stats['avg']
        baseline_std = baseline_stats['std']
        
        # Z-score based detection
        if baseline_std > 0:
            z_score = abs(current_avg - baseline_avg) / baseline_std
            if z_score > 3:  # 3-sigma rule
                self._trigger_alert(metric_name, 'statistical_anomaly', {
                    'z_score': z_score,
                    'current_avg': current_avg,
                    'baseline_avg': baseline_avg,
                    'timestamp': timestamp
                })
        
        # Threshold-based detection
        if current_avg > baseline_avg * 2:  # 2x increase
            self._trigger_alert(metric_name, 'spike_detected', {
                'current_avg': current_avg,
                'baseline_avg': baseline_avg,
                'multiplier': current_avg / baseline_avg,
                'timestamp': timestamp
            })
    
    def _get_window_stats(self, metric_name: str, window_size: int) -> Dict:
        if (metric_name not in self.metrics or 
            window_size not in self.metrics[metric_name]):
            return {}
        
        window = self.metrics[metric_name][window_size]
        if not window:
            return {}
        
        values = [point.value for point in window]
        
        return {
            'count': len(values),
            'avg': statistics.mean(values),
            'std': statistics.stdev(values) if len(values) > 1 else 0,
            'min': min(values),
            'max': max(values),
            'p95': statistics.quantiles(values, n=20)[18] if len(values) >= 20 else max(values)
        }
    
    def _trigger_alert(self, metric_name: str, alert_type: str, details: Dict):
        alert = {
            'metric': metric_name,
            'type': alert_type,
            'details': details,
            'timestamp': time.time()
        }
        self.alerts.append(alert)
        print(f"ALERT: {alert}")
```

**Q8: How would you optimize sliding window operations for very large datasets?**

**Answer:** Several optimization strategies:

1. **Memory-Efficient Implementation:**
```python
class MemoryEfficientSlidingWindow:
    def __init__(self, window_size, aggregation_func):
        self.window_size = window_size
        self.aggregation_func = aggregation_func
        self.circular_buffer = [None] * window_size
        self.head = 0
        self.count = 0
        self.current_result = None
    
    def add_value(self, value):
        # Overwrite old value in circular buffer
        self.circular_buffer[self.head] = value
        self.head = (self.head + 1) % self.window_size
        
        if self.count < self.window_size:
            self.count += 1
        
        # Calculate result only when window is full
        if self.count == self.window_size:
            active_values = self.circular_buffer[:self.count]
            self.current_result = self.aggregation_func(active_values)
        
        return self.current_result
```

2. **Incremental Computation:**
```python
class IncrementalSlidingWindow:
    def __init__(self, window_size):
        self.window_size = window_size
        self.window = deque()
        self.current_sum = 0
        self.current_count = 0
    
    def add_value(self, value):
        # Add new value
        self.window.append(value)
        self.current_sum += value
        self.current_count += 1
        
        # Remove old value if window is full
        if len(self.window) > self.window_size:
            old_value = self.window.popleft()
            self.current_sum -= old_value
            self.current_count -= 1
        
        return {
            'sum': self.current_sum,
            'avg': self.current_sum / len(self.window),
            'count': len(self.window)
        }
```

3. **Parallel Processing:**
```python
import concurrent.futures
from typing import List, Callable

class ParallelSlidingWindow:
    def __init__(self, num_workers=4):
        self.num_workers = num_workers
    
    def process_windows(self, data: List, window_size: int, 
                       process_func: Callable) -> List:
        # Split data into chunks for parallel processing
        chunk_size = len(data) // self.num_workers
        chunks = [
            data[i:i + chunk_size + window_size - 1] 
            for i in range(0, len(data), chunk_size)
        ]
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_workers) as executor:
            futures = [
                executor.submit(self._process_chunk, chunk, window_size, process_func)
                for chunk in chunks
            ]
            
            results = []
            for future in concurrent.futures.as_completed(futures):
                results.extend(future.result())
        
        return results
    
    def _process_chunk(self, chunk, window_size, process_func):
        results = []
        for i in range(len(chunk) - window_size + 1):
            window = chunk[i:i + window_size]
            result = process_func(window)
            results.append(result)
        return results
```

**Q9: Design a distributed sliding window system for a global application.**

**Answer:** Distributed architecture with coordination:

```python
import time
import json
from typing import Dict, List
from collections import deque

class DistributedSlidingWindow:
    def __init__(self, node_id: str, window_size: int, sync_interval: int = 10):
        self.node_id = node_id
        self.window_size = window_size
        self.sync_interval = sync_interval
        
        # Local sliding window
        self.local_window = deque()
        self.local_metrics = {}
        
        # Peer coordination
        self.peer_nodes = {}  # node_id -> last_sync_time
        self.global_state = {}
        
        # Synchronization
        self.last_sync = time.time()
    
    def add_event(self, event_data: Dict):
        """Add event to local window"""
        timestamp = time.time()
        event = {
            'timestamp': timestamp,
            'node_id': self.node_id,
            'data': event_data
        }
        
        self.local_window.append(event)
        
        # Remove old events
        cutoff_time = timestamp - self.window_size
        while self.local_window and self.local_window[0]['timestamp'] < cutoff_time:
            self.local_window.popleft()
        
        # Periodic synchronization
        if timestamp - self.last_sync > self.sync_interval:
            self._sync_with_peers()
    
    def get_global_metrics(self) -> Dict:
        """Get aggregated metrics across all nodes"""
        current_time = time.time()
        cutoff_time = current_time - self.window_size
        
        # Combine local and peer data
        all_events = list(self.local_window)
        
        for node_id, node_data in self.global_state.items():
            if node_id != self.node_id:
                for event in node_data.get('events', []):
                    if event['timestamp'] > cutoff_time:
                        all_events.append(event)
        
        # Calculate global metrics
        if not all_events:
            return {}
        
        return {
            'total_events': len(all_events),
            'events_per_second': len(all_events) / self.window_size,
            'nodes_active': len(set(event['node_id'] for event in all_events)),
            'time_range': {
                'start': min(event['timestamp'] for event in all_events),
                'end': max(event['timestamp'] for event in all_events)
            }
        }
    
    def _sync_with_peers(self):
        """Synchronize state with peer nodes"""
        # Prepare local state for sharing
        local_state = {
            'node_id': self.node_id,
            'timestamp': time.time(),
            'events': list(self.local_window)[-100:]  # Share last 100 events
        }
        
        # Send to all peers (simplified - would use actual network calls)
        self._broadcast_state(local_state)
        
        self.last_sync = time.time()
    
    def _broadcast_state(self, state: Dict):
        """Broadcast state to peer nodes"""
        # In real implementation, this would use network protocols
        # like gRPC, HTTP, or message queues
        pass
    
    def receive_peer_state(self, peer_state: Dict):
        """Receive and process state from peer node"""
        peer_id = peer_state['node_id']
        self.global_state[peer_id] = peer_state
        self.peer_nodes[peer_id] = peer_state['timestamp']
    
    def get_local_metrics(self) -> Dict:
        """Get metrics for local node only"""
        if not self.local_window:
            return {}
        
        return {
            'local_events': len(self.local_window),
            'events_per_second': len(self.local_window) / self.window_size,
            'oldest_event': self.local_window[0]['timestamp'],
            'newest_event': self.local_window[-1]['timestamp']
        }
```

### System Design Questions

**Q10: Design a real-time fraud detection system using sliding windows.**

**Answer:** Multi-layer fraud detection system:

```python
import time
from collections import deque, defaultdict
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum

class RiskLevel(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

@dataclass
class Transaction:
    user_id: str
    amount: float
    merchant: str
    location: str
    timestamp: float
    card_type: str

class FraudDetectionSystem:
    def __init__(self):
        # Multiple sliding windows for different time periods
        self.windows = {
            '1m': deque(),
            '5m': deque(),
            '1h': deque(),
            '24h': deque()
        }
        
        self.window_sizes = {
            '1m': 60,
            '5m': 300,
            '1h': 3600,
            '24h': 86400
        }
        
        # User-specific sliding windows
        self.user_windows = defaultdict(lambda: {
            '1m': deque(),
            '5m': deque(),
            '1h': deque()
        })
        
        # Risk thresholds
        self.thresholds = {
            'velocity': {'1m': 5, '5m': 20, '1h': 100},  # transactions per period
            'amount': {'1m': 10000, '5m': 50000, '1h': 200000},  # total amount
            'unique_merchants': {'1h': 10},  # unique merchants per hour
            'location_changes': {'1h': 3}  # location changes per hour
        }
    
    def process_transaction(self, transaction: Transaction) -> Dict:
        """Process transaction and return risk assessment"""
        
        # Add to global windows
        self._add_to_windows(transaction)
        
        # Add to user-specific windows
        self._add_to_user_windows(transaction)
        
        # Perform fraud checks
        risk_factors = self._analyze_risk_factors(transaction)
        
        # Calculate overall risk score
        risk_score = self._calculate_risk_score(risk_factors)
        
        # Determine action
        action = self._determine_action(risk_score)
        
        return {
            'transaction_id': f"{transaction.user_id}_{transaction.timestamp}",
            'risk_score': risk_score,
            'risk_level': self._get_risk_level(risk_score),
            'risk_factors': risk_factors,
            'action': action,
            'timestamp': transaction.timestamp
        }
    
    def _add_to_windows(self, transaction: Transaction):
        """Add transaction to global sliding windows"""
        for window_name, window in self.windows.items():
            window.append(transaction)
            
            # Remove old transactions
            window_size = self.window_sizes[window_name]
            cutoff_time = transaction.timestamp - window_size
            
            while window and window[0].timestamp < cutoff_time:
                window.popleft()
    
    def _add_to_user_windows(self, transaction: Transaction):
        """Add transaction to user-specific sliding windows"""
        user_windows = self.user_windows[transaction.user_id]
        
        for window_name, window in user_windows.items():
            window.append(transaction)
            
            # Remove old transactions
            window_size = self.window_sizes[window_name]
            cutoff_time = transaction.timestamp - window_size
            
            while window and window[0].timestamp < cutoff_time:
                window.popleft()
    
    def _analyze_risk_factors(self, transaction: Transaction) -> Dict:
        """Analyze various risk factors using sliding windows"""
        user_windows = self.user_windows[transaction.user_id]
        risk_factors = {}
        
        # 1. Transaction velocity check
        for period in ['1m', '5m', '1h']:
            count = len(user_windows[period])
            threshold = self.thresholds['velocity'][period]
            
            if count > threshold:
                risk_factors[f'high_velocity_{period}'] = {
                    'count': count,
                    'threshold': threshold,
                    'severity': min(count / threshold, 3.0)
                }
        
        # 2. Amount velocity check
        for period in ['1m', '5m', '1h']:
            total_amount = sum(t.amount for t in user_windows[period])
            threshold = self.thresholds['amount'][period]
            
            if total_amount > threshold:
                risk_factors[f'high_amount_{period}'] = {
                    'total_amount': total_amount,
                    'threshold': threshold,
                    'severity': min(total_amount / threshold, 3.0)
                }
        
        # 3. Unique merchants check
        merchants_1h = set(t.merchant for t in user_windows['1h'])
        if len(merchants_1h) > self.thresholds['unique_merchants']['1h']:
            risk_factors['too_many_merchants'] = {
                'count': len(merchants_1h),
                'threshold': self.thresholds['unique_merchants']['1h'],
                'severity': len(merchants_1h) / self.thresholds['unique_merchants']['1h']
            }
        
        # 4. Location changes check
        locations_1h = [t.location for t in user_windows['1h']]
        location_changes = len(set(locations_1h))
        
        if location_changes > self.thresholds['location_changes']['1h']:
            risk_factors['rapid_location_changes'] = {
                'changes': location_changes,
                'threshold': self.thresholds['location_changes']['1h'],
                'severity': location_changes / self.thresholds['location_changes']['1h']
            }
        
        # 5. Unusual amount pattern
        if user_windows['1h']:
            amounts_1h = [t.amount for t in user_windows['1h']]
            avg_amount = sum(amounts_1h) / len(amounts_1h)
            
            if transaction.amount > avg_amount * 5:  # 5x average
                risk_factors['unusual_amount'] = {
                    'current_amount': transaction.amount,
                    'average_amount': avg_amount,
                    'severity': transaction.amount / avg_amount
                }
        
        return risk_factors
    
    def _calculate_risk_score(self, risk_factors: Dict) -> float:
        """Calculate overall risk score from individual factors"""
        if not risk_factors:
            return 0.0
        
        # Weighted risk calculation
        weights = {
            'high_velocity': 0.3,
            'high_amount': 0.25,
            'too_many_merchants': 0.2,
            'rapid_location_changes': 0.15,
            'unusual_amount': 0.1
        }
        
        total_score = 0.0
        total_weight = 0.0
        
        for factor_name, factor_data in risk_factors.items():
            # Find matching weight pattern
            weight = 0.1  # default weight
            for pattern, w in weights.items():
                if pattern in factor_name:
                    weight = w
                    break
            
            severity = factor_data.get('severity', 1.0)
            total_score += weight * severity
            total_weight += weight
        
        return min(total_score, 10.0)  # Cap at 10.0
    
    def _get_risk_level(self, risk_score: float) -> RiskLevel:
        """Convert risk score to risk level"""
        if risk_score < 2.0:
            return RiskLevel.LOW
        elif risk_score < 5.0:
            return RiskLevel.MEDIUM
        elif risk_score < 8.0:
            return RiskLevel.HIGH
        else:
            return RiskLevel.CRITICAL
    
    def _determine_action(self, risk_score: float) -> str:
        """Determine action based on risk score"""
        if risk_score < 2.0:
            return "approve"
        elif risk_score < 5.0:
            return "review"
        elif risk_score < 8.0:
            return "challenge"  # Require additional authentication
        else:
            return "block"

# Usage example
fraud_detector = FraudDetectionSystem()

# Process transactions
transactions = [
    Transaction("user123", 100.0, "Amazon", "NYC", time.time(), "credit"),
    Transaction("user123", 200.0, "Walmart", "NYC", time.time() + 30, "credit"),
    Transaction("user123", 5000.0, "Jewelry Store", "LA", time.time() + 60, "credit"),
]

for transaction in transactions:
    result = fraud_detector.process_transaction(transaction)
    print(f"Transaction: ${transaction.amount} - Risk: {result['risk_level']} - Action: {result['action']}")
```

## Best Practices and Tips

### 1. Choosing the Right Window Type
- **Fixed windows**: Use for regular reporting and simple aggregations
- **Variable windows**: Use for pattern matching and condition-based analysis
- **Multi-windows**: Use for comparative analysis across different time scales

### 2. Memory Management
- Implement proper cleanup for old data
- Use circular buffers for fixed-size windows
- Consider data compression for long-term storage

### 3. Performance Optimization
- Use appropriate data structures (deque for FIFO, heaps for min/max)
- Implement incremental calculations when possible
- Consider parallel processing for large datasets

### 4. Error Handling
- Handle edge cases (empty windows, single elements)
- Implement proper validation for window parameters
- Add monitoring for window performance metrics

### 5. Testing Strategies
- Test with various window sizes and data patterns
- Verify correctness with known datasets
- Performance test with realistic data volumes
- Test edge cases and error conditions

## Conclusion

Sliding window patterns are fundamental to many system design problems, particularly in:
- Rate limiting and throttling
- Real-time analytics and monitoring
- Performance optimization
- Data stream processing
- Anomaly detection

The key to successful implementation is choosing the right window type, optimizing for your specific use case, and properly handling edge cases. These patterns enable efficient processing of continuous data streams while maintaining bounded memory usage and predictable performance characteristics.
