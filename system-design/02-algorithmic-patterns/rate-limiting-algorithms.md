# Rate Limiting Algorithms

Rate limiting is a crucial technique in system design to control the rate of requests sent or received by a network interface controller. It helps prevent abuse, ensures fair usage, and maintains system stability.

## Table of Contents
- [Why Rate Limiting?](#why-rate-limiting)
- [Common Rate Limiting Algorithms](#common-rate-limiting-algorithms)
- [Implementation Considerations](#implementation-considerations)
- [Comparison Table](#comparison-table)
- [Real-World Examples](#real-world-examples)

## Why Rate Limiting?

### Benefits
- **Prevent DoS attacks**: Protect against malicious users overwhelming the system
- **Cost control**: Limit resource consumption and associated costs
- **Fair usage**: Ensure equitable access to resources among users
- **Quality of service**: Maintain consistent performance for all users
- **Compliance**: Meet SLA requirements and regulatory constraints

### Common Use Cases
- API rate limiting (REST, GraphQL)
- Database connection throttling
- Message queue processing
- Network bandwidth control
- User action limiting (login attempts, file uploads)

## Common Rate Limiting Algorithms

### 1. Token Bucket Algorithm

**Concept**: A bucket holds tokens that are consumed by requests. Tokens are added at a fixed rate.

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()
    
    def consume(self, tokens=1):
        self._refill()
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False
    
    def _refill(self):
        now = time.time()
        tokens_to_add = (now - self.last_refill) * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
```

**Pros**:
- Allows burst traffic up to bucket capacity
- Smooth rate limiting over time
- Memory efficient

**Cons**:
- Complex to implement correctly
- Requires careful tuning of parameters

**Best for**: APIs that need to handle occasional bursts

### 2. Leaky Bucket Algorithm

**Concept**: Requests are processed at a fixed rate, excess requests overflow and are dropped.

```python
class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.queue = []
        self.leak_rate = leak_rate  # requests per second
        self.last_leak = time.time()
    
    def add_request(self, request):
        self._leak()
        if len(self.queue) < self.capacity:
            self.queue.append(request)
            return True
        return False  # Bucket overflow
    
    def _leak(self):
        now = time.time()
        elapsed = now - self.last_leak
        requests_to_process = int(elapsed * self.leak_rate)
        
        for _ in range(min(requests_to_process, len(self.queue))):
            self.queue.pop(0)
        
        self.last_leak = now
```

**Pros**:
- Smooth, predictable output rate
- Good for systems requiring steady processing
- Simple to understand

**Cons**:
- No burst handling
- Can introduce latency
- Memory overhead for queuing

**Best for**: Systems requiring consistent processing rates

### 3. Fixed Window Counter

**Concept**: Count requests in fixed time windows (e.g., per minute).

```python
class FixedWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # in seconds
        self.windows = {}
    
    def is_allowed(self, user_id):
        now = time.time()
        window = int(now // self.window_size)
        
        if user_id not in self.windows:
            self.windows[user_id] = {}
        
        current_count = self.windows[user_id].get(window, 0)
        
        if current_count < self.limit:
            self.windows[user_id][window] = current_count + 1
            return True
        
        return False
```

**Pros**:
- Simple to implement
- Memory efficient
- Easy to understand and debug

**Cons**:
- Boundary problem (burst at window edges)
- Not smooth rate limiting
- Can allow 2x limit at window boundaries

**Best for**: Simple rate limiting with acceptable burst behavior

### 4. Sliding Window Log

**Concept**: Keep a log of request timestamps and count requests in a sliding time window.

```python
class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.requests = {}  # user_id -> list of timestamps
    
    def is_allowed(self, user_id):
        now = time.time()
        
        if user_id not in self.requests:
            self.requests[user_id] = []
        
        # Remove old requests outside the window
        cutoff = now - self.window_size
        self.requests[user_id] = [
            req_time for req_time in self.requests[user_id] 
            if req_time > cutoff
        ]
        
        if len(self.requests[user_id]) < self.limit:
            self.requests[user_id].append(now)
            return True
        
        return False
```

**Pros**:
- Accurate rate limiting
- No boundary issues
- Precise control

**Cons**:
- High memory usage
- Expensive cleanup operations
- Complex implementation

**Best for**: Applications requiring precise rate limiting

### 5. Sliding Window Counter

**Concept**: Hybrid approach combining fixed windows with weighted counting.

```python
class SlidingWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.windows = {}
    
    def is_allowed(self, user_id):
        now = time.time()
        current_window = int(now // self.window_size)
        previous_window = current_window - 1
        
        if user_id not in self.windows:
            self.windows[user_id] = {}
        
        # Calculate weighted count
        current_count = self.windows[user_id].get(current_window, 0)
        previous_count = self.windows[user_id].get(previous_window, 0)
        
        # Weight based on how far we are into current window
        elapsed_in_current = now % self.window_size
        weight = elapsed_in_current / self.window_size
        
        estimated_count = (previous_count * (1 - weight)) + current_count
        
        if estimated_count < self.limit:
            self.windows[user_id][current_window] = current_count + 1
            return True
        
        return False
```

**Pros**:
- Good approximation of sliding window
- Memory efficient
- Reduces boundary issues

**Cons**:
- Approximation, not exact
- More complex than fixed window
- Still has some boundary effects

**Best for**: Balance between accuracy and efficiency

## Implementation Considerations

### Distributed Systems

**Challenges**:
- Shared state across multiple servers
- Network latency and partitions
- Consistency vs. availability trade-offs

**Solutions**:

1. **Centralized Counter** (Redis/Database)
```python
import redis

class DistributedRateLimit:
    def __init__(self, redis_client, limit, window):
        self.redis = redis_client
        self.limit = limit
        self.window = window
    
    def is_allowed(self, user_id):
        pipe = self.redis.pipeline()
        now = time.time()
        window = int(now // self.window)
        key = f"rate_limit:{user_id}:{window}"
        
        pipe.incr(key)
        pipe.expire(key, self.window)
        results = pipe.execute()
        
        return results[0] <= self.limit
```

2. **Distributed Token Bucket** (with Redis Lua script)
```lua
-- Redis Lua script for atomic token bucket operations
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local tokens = tonumber(ARGV[2])
local interval = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local current_tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or 0

local now = redis.call('TIME')
local current_time = tonumber(now[1])

-- Calculate tokens to add
local elapsed = current_time - last_refill
local tokens_to_add = math.floor(elapsed * tokens / interval)
current_tokens = math.min(capacity, current_tokens + tokens_to_add)

if current_tokens >= requested then
    current_tokens = current_tokens - requested
    redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', current_time)
    redis.call('EXPIRE', key, interval * 2)
    return 1
else
    redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', current_time)
    redis.call('EXPIRE', key, interval * 2)
    return 0
end
```

### Storage Considerations

| Storage Type | Pros | Cons | Best For |
|--------------|------|------|----------|
| **In-Memory** | Fast, low latency | Not persistent, single node | High-performance, temporary limits |
| **Redis** | Fast, distributed, persistent | Additional infrastructure | Distributed systems |
| **Database** | Persistent, ACID | Slower, potential bottleneck | Long-term tracking, audit trails |
| **Local + Sync** | Fast reads, eventual consistency | Complex synchronization | Hybrid approaches |

### Error Handling

```python
class RateLimitError(Exception):
    def __init__(self, retry_after=None):
        self.retry_after = retry_after
        super().__init__(f"Rate limit exceeded. Retry after {retry_after} seconds")

def rate_limit_middleware(rate_limiter):
    def decorator(func):
        def wrapper(user_id, *args, **kwargs):
            if not rate_limiter.is_allowed(user_id):
                retry_after = rate_limiter.get_retry_after(user_id)
                raise RateLimitError(retry_after)
            return func(user_id, *args, **kwargs)
        return wrapper
    return decorator
```

## Comparison Table

| Algorithm | Memory Usage | Accuracy | Burst Handling | Implementation Complexity | Use Case |
|-----------|--------------|----------|----------------|---------------------------|----------|
| **Token Bucket** | Low | High | Excellent | Medium | APIs with burst needs |
| **Leaky Bucket** | Medium | High | Poor | Medium | Steady processing systems |
| **Fixed Window** | Low | Low | Poor | Low | Simple rate limiting |
| **Sliding Window Log** | High | Perfect | Good | High | Precise control needed |
| **Sliding Window Counter** | Low | Good | Good | Medium | Balanced approach |

## Real-World Examples

### API Gateway Rate Limiting
```python
class APIGatewayRateLimit:
    def __init__(self):
        self.user_limits = TokenBucket(100, 10)  # 100 requests, 10/sec refill
        self.ip_limits = TokenBucket(1000, 50)   # 1000 requests, 50/sec refill
        self.global_limits = TokenBucket(10000, 500)  # Global limit
    
    def check_limits(self, user_id, ip_address):
        # Check in order of specificity
        if not self.global_limits.consume():
            raise RateLimitError("Global rate limit exceeded")
        
        if not self.ip_limits.consume():
            raise RateLimitError("IP rate limit exceeded")
        
        if not self.user_limits.consume():
            raise RateLimitError("User rate limit exceeded")
        
        return True
```

### Database Connection Throttling
```python
class DatabaseConnectionThrottle:
    def __init__(self, max_connections, refill_rate):
        self.semaphore = asyncio.Semaphore(max_connections)
        self.rate_limiter = TokenBucket(max_connections, refill_rate)
    
    async def get_connection(self):
        if not self.rate_limiter.consume():
            raise RateLimitError("Database connection rate limit exceeded")
        
        await self.semaphore.acquire()
        try:
            # Get database connection
            connection = await get_db_connection()
            return connection
        except:
            self.semaphore.release()
            raise
```

### Multi-Tier Rate Limiting
```python
class MultiTierRateLimit:
    def __init__(self):
        self.tiers = {
            'free': TokenBucket(100, 1),      # 100 requests, 1/sec
            'premium': TokenBucket(1000, 10), # 1000 requests, 10/sec
            'enterprise': TokenBucket(10000, 100)  # 10000 requests, 100/sec
        }
    
    def is_allowed(self, user_id, tier):
        if tier not in self.tiers:
            return False
        
        return self.tiers[tier].consume()
```

## Best Practices

1. **Choose the right algorithm** based on your specific requirements
2. **Monitor and alert** on rate limit violations
3. **Provide clear error messages** with retry information
4. **Use multiple layers** (user, IP, global) for comprehensive protection
5. **Consider graceful degradation** instead of hard blocking
6. **Test thoroughly** under various load conditions
7. **Document rate limits** clearly for API consumers
8. **Implement proper logging** for debugging and analysis

## Interview Questions & Answers

### Basic Questions

**Q1: What is rate limiting and why is it important?**

**Answer:** Rate limiting is a technique to control the rate of requests sent or received by a system. It's important because it:
- Prevents DoS attacks and abuse
- Ensures fair resource usage among users
- Maintains system stability under load
- Controls costs associated with resource consumption
- Helps meet SLA requirements

**Q2: Explain the difference between Token Bucket and Leaky Bucket algorithms.**

**Answer:** 
- **Token Bucket**: Allows burst traffic up to bucket capacity. Tokens are added at a fixed rate, and requests consume tokens. If tokens are available, requests are processed immediately.
- **Leaky Bucket**: Processes requests at a fixed rate regardless of input rate. Excess requests are queued or dropped. No burst handling - output is always smooth.

**Token Bucket is better for APIs that need burst handling, while Leaky Bucket is better for systems requiring steady processing rates.**

**Q3: What are the trade-offs of Fixed Window vs Sliding Window rate limiting?**

**Answer:**
- **Fixed Window**: 
  - Pros: Simple, memory efficient, easy to implement
  - Cons: Boundary problem (can allow 2x limit at window edges), not smooth
- **Sliding Window**: 
  - Pros: Accurate, no boundary issues, smooth rate limiting
  - Cons: Higher memory usage, more complex implementation

### Intermediate Questions

**Q4: How would you implement distributed rate limiting across multiple servers?**

**Answer:** Several approaches:

1. **Centralized Counter (Redis)**:
```python
def distributed_rate_limit(user_id, limit, window):
    current_time = time.time()
    key = f"rate_limit:{user_id}:{int(current_time // window)}"
    
    current_count = redis.incr(key)
    redis.expire(key, window)
    
    return current_count <= limit
```

2. **Sliding Window with Redis Sorted Sets**:
```python
def sliding_window_rate_limit(user_id, limit, window):
    current_time = time.time()
    key = f"rate_limit:{user_id}"
    
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, current_time - window)
    pipe.zcard(key)
    pipe.zadd(key, {str(current_time): current_time})
    pipe.expire(key, window)
    
    results = pipe.execute()
    return results[1] < limit
```

3. **Local + Sync Approach**: Each server maintains local counters and periodically syncs with central store.

**Q5: What happens when your rate limiter fails? How do you handle it?**

**Answer:** Failure handling strategies:

1. **Fail Open**: Allow all requests when rate limiter is down
   - Pros: No service disruption
   - Cons: No protection during outage

2. **Fail Closed**: Block all requests when rate limiter is down
   - Pros: Maintains protection
   - Cons: Service disruption

3. **Graceful Degradation**: 
   - Use local/cached rate limits
   - Implement circuit breaker pattern
   - Have backup rate limiting mechanisms

```python
def resilient_rate_limit(user_id):
    try:
        return primary_rate_limiter.is_allowed(user_id)
    except Exception:
        # Fallback to local rate limiter
        return local_rate_limiter.is_allowed(user_id)
```

**Q6: How would you rate limit different types of users (free, premium, enterprise)?**

**Answer:** Multi-tier rate limiting:

```python
class TieredRateLimit:
    def __init__(self):
        self.limits = {
            'free': {'requests': 100, 'window': 3600},      # 100/hour
            'premium': {'requests': 1000, 'window': 3600},   # 1000/hour
            'enterprise': {'requests': 10000, 'window': 3600} # 10000/hour
        }
        self.rate_limiters = {}
    
    def is_allowed(self, user_id, tier):
        if tier not in self.limits:
            return False
        
        if user_id not in self.rate_limiters:
            limit_config = self.limits[tier]
            self.rate_limiters[user_id] = TokenBucket(
                limit_config['requests'], 
                limit_config['requests'] / limit_config['window']
            )
        
        return self.rate_limiters[user_id].consume()
```

### Advanced Questions

**Q7: Design a rate limiter for a global CDN with edge locations worldwide.**

**Answer:** Multi-layer approach:

1. **Edge-level rate limiting**: Fast local decisions
2. **Regional aggregation**: Sync between nearby edges
3. **Global coordination**: Central coordination for global limits

```python
class GlobalCDNRateLimit:
    def __init__(self, edge_id, region_id):
        self.edge_id = edge_id
        self.region_id = region_id
        self.local_limiter = TokenBucket(1000, 10)  # Local edge limit
        self.regional_limiter = DistributedRateLimit()  # Regional Redis
        self.global_limiter = GlobalRateLimit()  # Global coordination
    
    def is_allowed(self, user_id, request_type):
        # 1. Check local edge limit (fastest)
        if not self.local_limiter.consume():
            return False
        
        # 2. Check regional limit (medium latency)
        if not self.regional_limiter.is_allowed(user_id, 'regional'):
            return False
        
        # 3. Check global limit (highest latency, async)
        # Use eventual consistency for global limits
        self.global_limiter.record_request_async(user_id)
        
        return True
```

**Q8: How would you implement rate limiting for different API endpoints with different limits?**

**Answer:** Hierarchical rate limiting:

```python
class HierarchicalRateLimit:
    def __init__(self):
        self.limiters = {
            'global': TokenBucket(10000, 100),  # Global user limit
            'endpoints': {
                '/api/search': TokenBucket(1000, 10),
                '/api/upload': TokenBucket(100, 1),
                '/api/download': TokenBucket(500, 5)
            }
        }
    
    def is_allowed(self, user_id, endpoint):
        # Check global user limit first
        if not self.limiters['global'].consume():
            return False, "Global rate limit exceeded"
        
        # Check endpoint-specific limit
        if endpoint in self.limiters['endpoints']:
            if not self.limiters['endpoints'][endpoint].consume():
                return False, f"Rate limit exceeded for {endpoint}"
        
        return True, "Allowed"
```

**Q9: How do you handle rate limiting for batch operations or bulk APIs?**

**Answer:** Weight-based rate limiting:

```python
class WeightedRateLimit:
    def __init__(self, capacity, refill_rate):
        self.token_bucket = TokenBucket(capacity, refill_rate)
        self.weights = {
            'single_request': 1,
            'batch_10': 10,
            'batch_100': 100,
            'bulk_operation': 500
        }
    
    def is_allowed(self, operation_type, batch_size=1):
        if operation_type in self.weights:
            tokens_needed = self.weights[operation_type]
        else:
            tokens_needed = batch_size  # Default: 1 token per item
        
        return self.token_bucket.consume(tokens_needed)
```

**Q10: How would you monitor and alert on rate limiting effectiveness?**

**Answer:** Comprehensive monitoring strategy:

```python
class RateLimitMonitor:
    def __init__(self):
        self.metrics = {
            'requests_allowed': 0,
            'requests_blocked': 0,
            'false_positives': 0,  # Legitimate requests blocked
            'false_negatives': 0   # Malicious requests allowed
        }
        self.alerts = []
    
    def record_decision(self, allowed, user_id, endpoint, is_legitimate=None):
        if allowed:
            self.metrics['requests_allowed'] += 1
            if is_legitimate is False:  # Malicious request allowed
                self.metrics['false_negatives'] += 1
                self.trigger_alert('false_negative', user_id, endpoint)
        else:
            self.metrics['requests_blocked'] += 1
            if is_legitimate is True:  # Legitimate request blocked
                self.metrics['false_positives'] += 1
                self.trigger_alert('false_positive', user_id, endpoint)
    
    def get_effectiveness_metrics(self):
        total_requests = self.metrics['requests_allowed'] + self.metrics['requests_blocked']
        if total_requests == 0:
            return {}
        
        return {
            'block_rate': self.metrics['requests_blocked'] / total_requests,
            'false_positive_rate': self.metrics['false_positives'] / total_requests,
            'false_negative_rate': self.metrics['false_negatives'] / total_requests,
            'accuracy': 1 - (self.metrics['false_positives'] + self.metrics['false_negatives']) / total_requests
        }
```

### System Design Questions

**Q11: Design a rate limiter for Twitter's API that handles 500M requests per day.**

**Answer:** Architecture considerations:

1. **Scale**: 500M requests/day = ~5,787 RPS average, ~50K RPS peak
2. **Distribution**: Multiple data centers, edge locations
3. **Storage**: Redis cluster with sharding
4. **Algorithm**: Sliding window counter for accuracy

```python
class TwitterAPIRateLimit:
    def __init__(self):
        self.redis_cluster = RedisCluster()
        self.rate_limits = {
            'user_timeline': {'limit': 300, 'window': 900},    # 300/15min
            'search': {'limit': 180, 'window': 900},           # 180/15min
            'post_tweet': {'limit': 300, 'window': 900}        # 300/15min
        }
    
    def is_allowed(self, user_id, endpoint, app_id=None):
        # Composite key for sharding
        key = f"rate_limit:{endpoint}:{user_id}"
        if app_id:
            key += f":{app_id}"
        
        # Use consistent hashing to determine Redis shard
        shard = self.get_shard(key)
        
        # Sliding window implementation
        return self.sliding_window_check(shard, key, endpoint)
    
    def get_shard(self, key):
        return self.redis_cluster.get_shard(hash(key))
```

**Q12: How would you implement rate limiting for a microservices architecture?**

**Answer:** Service mesh approach:

1. **Sidecar Pattern**: Rate limiter as sidecar proxy
2. **Centralized Policy**: Configuration management
3. **Distributed Enforcement**: Local decisions with global coordination

```python
class MicroserviceRateLimit:
    def __init__(self, service_name):
        self.service_name = service_name
        self.policy_store = PolicyStore()
        self.local_cache = LocalCache()
        self.global_coordinator = GlobalCoordinator()
    
    def check_rate_limit(self, request):
        # Get policies for this service and endpoint
        policies = self.policy_store.get_policies(
            self.service_name, 
            request.endpoint
        )
        
        for policy in policies:
            if not self.evaluate_policy(policy, request):
                return False, f"Rate limit exceeded: {policy.name}"
        
        return True, "Allowed"
    
    def evaluate_policy(self, policy, request):
        # Extract rate limit key (user_id, ip, api_key, etc.)
        key = self.extract_key(policy.key_extractor, request)
        
        # Check local cache first
        if self.local_cache.is_rate_limited(key, policy):
            return False
        
        # Check with global coordinator
        return self.global_coordinator.is_allowed(key, policy)
```

## Common Pitfalls and Solutions

### 1. Clock Synchronization Issues
**Problem**: Distributed systems with unsynchronized clocks
**Solution**: Use logical timestamps or centralized time service

### 2. Race Conditions
**Problem**: Concurrent requests bypassing rate limits
**Solution**: Atomic operations, proper locking, or optimistic concurrency control

### 3. Memory Leaks
**Problem**: Rate limiter state growing indefinitely
**Solution**: Implement proper cleanup, TTL, and garbage collection

### 4. Hot Spotting
**Problem**: Some users/keys consuming disproportionate resources
**Solution**: Consistent hashing, load balancing, and circuit breakers

### 5. Configuration Management
**Problem**: Difficulty updating rate limits without downtime
**Solution**: Dynamic configuration, feature flags, and gradual rollouts

## Further Reading

- [Token Bucket Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Token_bucket)
- [Leaky Bucket Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Leaky_bucket)
- [Rate Limiting Strategies and Techniques - NGINX](https://www.nginx.com/blog/rate-limiting-nginx/)
- [Distributed Rate Limiting with Redis](https://redis.io/docs/manual/patterns/distributed-locks/)
