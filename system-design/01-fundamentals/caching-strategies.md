# Caching Strategies - System Design Fundamentals

## Table of Contents
1. [What is Caching?](#what-is-caching)
2. [Cache Patterns with Redis](#cache-patterns-with-redis)
3. [Cache Levels](#cache-levels)
4. [Cache Invalidation](#cache-invalidation)
5. [Distributed Caching with Redis](#distributed-caching-with-redis)
6. [Cache Replacement Policies](#cache-replacement-policies)
7. [Redis Implementation Examples](#redis-implementation-examples)
8. [Performance Considerations](#performance-considerations)
9. [Interview Questions](#interview-questions)

---

## What is Caching?

**Caching** is a technique to store frequently accessed data in a fast storage layer (cache) to reduce latency and improve system performance by avoiding expensive operations like database queries or API calls.

### Key Benefits:
- **Reduced Latency**: Faster data access from memory vs disk/network
- **Improved Throughput**: Handle more requests with same resources
- **Reduced Load**: Less pressure on backend systems (databases, APIs)
- **Cost Savings**: Fewer expensive operations and resource usage
- **Better User Experience**: Faster response times

### Cache Hit vs Cache Miss:
```
Cache Hit: Data found in cache → Fast response
Cache Miss: Data not in cache → Fetch from source + Store in cache
```

---

## Cache Patterns with Redis

### 1. Cache-Aside (Lazy Loading) with Redis
**Application manages cache explicitly**

```python
import redis
import json
import time

# Redis connection
redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

def get_user_cache_aside(user_id):
    cache_key = f"user:{user_id}"
    
    # Try to get from Redis cache first
    cached_user = redis_client.get(cache_key)
    
    if cached_user:  # Cache hit
        print(f"Cache HIT for user {user_id}")
        return json.loads(cached_user)
    
    # Cache miss - load from database
    print(f"Cache MISS for user {user_id}")
    user = database.get_user(user_id)
    
    if user:
        # Store in Redis cache with TTL (1 hour)
        redis_client.setex(cache_key, 3600, json.dumps(user))
    
    return user

def update_user_cache_aside(user_id, user_data):
    # Update database first
    database.update_user(user_id, user_data)
    
    # Invalidate cache to ensure consistency
    cache_key = f"user:{user_id}"
    redis_client.delete(cache_key)
    
    print(f"Cache invalidated for user {user_id}")

# Example usage
user = get_user_cache_aside(123)  # First call: Cache miss, loads from DB
user = get_user_cache_aside(123)  # Second call: Cache hit, loads from Redis
```

**Pros:**
- Simple to understand and implement
- Cache only contains requested data
- Cache failures don't affect application

**Cons:**
- Cache miss penalty (extra latency)
- Potential for stale data
- Application manages cache logic

### 2. Write-Through with Redis
**Write to cache and database simultaneously**

```python
class WriteThroughCache:
    def __init__(self, redis_client, database):
        self.cache = redis_client
        self.database = database
        self.default_ttl = 3600  # 1 hour
    
    def get(self, key):
        # Always try cache first
        cached_data = self.cache.get(key)
        if cached_data:
            return json.loads(cached_data)
        
        # Load from database and cache it
        data = self.database.get(key)
        if data:
            self.cache.setex(key, self.default_ttl, json.dumps(data))
        return data
    
    def set(self, key, value):
        # Write to both database and cache simultaneously
        self.database.set(key, value)
        self.cache.setex(key, self.default_ttl, json.dumps(value))
        return True

# Usage
cache_db = WriteThroughCache(redis_client, mysql_database)

# This writes to both MySQL and Redis
cache_db.set("user:456", {"name": "John", "email": "john@example.com"})

# This reads from Redis (fast) or MySQL (slower) if not cached
user = cache_db.get("user:456")
```

**Pros:**
- Data consistency between cache and database
- Read performance is good
- Simple for applications

**Cons:**
- Write latency (writes to both systems)
- Wasted cache space for rarely read data
- Cache failure affects writes

### 3. Write-Behind (Write-Back) with Redis
**Write to cache immediately, database asynchronously**

```python
import asyncio
import threading
from queue import Queue

class WriteBehindCache:
    def __init__(self, redis_client, database):
        self.cache = redis_client
        self.database = database
        self.write_queue = Queue()
        self.default_ttl = 3600
        
        # Start background writer thread
        self.writer_thread = threading.Thread(target=self._background_writer, daemon=True)
        self.writer_thread.start()
    
    def get(self, key):
        cached_data = self.cache.get(key)
        if cached_data:
            return json.loads(cached_data)
        
        # Load from database
        data = self.database.get(key)
        if data:
            self.cache.setex(key, self.default_ttl, json.dumps(data))
        return data
    
    def set(self, key, value):
        # Write to cache immediately
        self.cache.setex(key, self.default_ttl, json.dumps(value))
        
        # Queue database write for background processing
        self.write_queue.put((key, value))
        return True
    
    def _background_writer(self):
        """Background thread to write to database"""
        while True:
            try:
                key, value = self.write_queue.get(timeout=1)
                self.database.set(key, value)
                print(f"Background write completed for {key}")
                self.write_queue.task_done()
            except:
                continue

# Usage
write_behind_cache = WriteBehindCache(redis_client, mysql_database)

# Fast write (only to Redis)
write_behind_cache.set("user:789", {"name": "Jane", "email": "jane@example.com"})

# Database write happens asynchronously in background
```

**Pros:**
- Very fast writes (only to cache)
- Good for write-heavy workloads
- Reduces database load

**Cons:**
- Risk of data loss if cache fails
- Complex implementation
- Eventual consistency

### 4. Refresh-Ahead with Redis
**Proactively refresh cache before expiration**

```python
import time
import threading

class RefreshAheadCache:
    def __init__(self, redis_client, database):
        self.cache = redis_client
        self.database = database
        self.default_ttl = 3600
        self.refresh_threshold = 0.8  # Refresh when 80% of TTL elapsed
    
    def get(self, key):
        cached_data = self.cache.get(key)
        
        if cached_data:
            # Check if we need to refresh
            ttl = self.cache.ttl(key)
            if ttl > 0 and ttl < (self.default_ttl * (1 - self.refresh_threshold)):
                # Trigger background refresh
                threading.Thread(target=self._refresh_cache, args=(key,), daemon=True).start()
            
            return json.loads(cached_data)
        
        # Cache miss - load and cache
        data = self.database.get(key)
        if data:
            self.cache.setex(key, self.default_ttl, json.dumps(data))
        return data
    
    def _refresh_cache(self, key):
        """Background refresh of cache data"""
        try:
            fresh_data = self.database.get(key)
            if fresh_data:
                self.cache.setex(key, self.default_ttl, json.dumps(fresh_data))
                print(f"Cache refreshed for {key}")
        except Exception as e:
            print(f"Failed to refresh cache for {key}: {e}")

# Usage
refresh_cache = RefreshAheadCache(redis_client, mysql_database)
user = refresh_cache.get("user:999")  # May trigger background refresh
```

**Pros:**
- Reduced cache miss penalty
- Always fresh data
- Good user experience

**Cons:**
- Complex implementation
- Additional background processing
- May refresh unused data

---

## Cache Levels

### 1. Browser Cache
```http
# HTTP headers for browser caching
Cache-Control: public, max-age=3600
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
```

### 2. CDN Cache
```python
# CDN cache configuration example
cdn_config = {
    "cache_behaviors": [
        {
            "path_pattern": "/api/static/*",
            "ttl": 86400,  # 24 hours
            "compress": True
        },
        {
            "path_pattern": "/api/dynamic/*",
            "ttl": 300,    # 5 minutes
            "compress": False
        }
    ]
}
```

### 3. Application Cache (Redis)
```python
# Multi-level caching with Redis
class MultiLevelCache:
    def __init__(self):
        self.l1_cache = {}  # In-memory cache
        self.l2_cache = redis.Redis(host='localhost', port=6379, db=0)
        self.l3_database = mysql_database
    
    def get(self, key):
        # L1: In-memory cache (fastest)
        if key in self.l1_cache:
            print(f"L1 cache hit for {key}")
            return self.l1_cache[key]
        
        # L2: Redis cache
        cached_data = self.l2_cache.get(key)
        if cached_data:
            print(f"L2 cache hit for {key}")
            data = json.loads(cached_data)
            self.l1_cache[key] = data  # Populate L1
            return data
        
        # L3: Database
        print(f"Cache miss for {key}, loading from database")
        data = self.l3_database.get(key)
        if data:
            # Populate both cache levels
            self.l1_cache[key] = data
            self.l2_cache.setex(key, 3600, json.dumps(data))
        
        return data
    
    def invalidate(self, key):
        # Invalidate all cache levels
        if key in self.l1_cache:
            del self.l1_cache[key]
        self.l2_cache.delete(key)

# Usage
multi_cache = MultiLevelCache()
user = multi_cache.get("user:123")
```

---

## Cache Invalidation

### 1. TTL-Based Expiration
```python
# Redis TTL examples
redis_client.setex("user:123", 3600, json.dumps(user_data))  # Expires in 1 hour
redis_client.expire("user:123", 1800)  # Update TTL to 30 minutes

# Check TTL
ttl = redis_client.ttl("user:123")
print(f"Key expires in {ttl} seconds")

# Set expiration at specific time
import datetime
expire_time = datetime.datetime(2024, 1, 1, 0, 0, 0)
redis_client.expireat("user:123", int(expire_time.timestamp()))
```

### 2. Event-Based Invalidation
```python
class EventBasedCache:
    def __init__(self, redis_client):
        self.cache = redis_client
        self.event_handlers = {}
    
    def register_invalidation_handler(self, event_type, handler):
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)
    
    def trigger_event(self, event_type, data):
        if event_type in self.event_handlers:
            for handler in self.event_handlers[event_type]:
                handler(data)
    
    def invalidate_user_cache(self, user_data):
        user_id = user_data['user_id']
        # Invalidate user-related cache keys
        patterns = [
            f"user:{user_id}",
            f"user:{user_id}:posts",
            f"user:{user_id}:profile",
            f"timeline:{user_id}"
        ]
        
        for pattern in patterns:
            self.cache.delete(pattern)
        print(f"Invalidated cache for user {user_id}")

# Usage
event_cache = EventBasedCache(redis_client)
event_cache.register_invalidation_handler('user_updated', event_cache.invalidate_user_cache)

# When user is updated
event_cache.trigger_event('user_updated', {'user_id': 123})
```

### 3. Tag-Based Invalidation
```python
class TagBasedCache:
    def __init__(self, redis_client):
        self.cache = redis_client
    
    def set_with_tags(self, key, value, tags, ttl=3600):
        # Store the actual data
        self.cache.setex(key, ttl, json.dumps(value))
        
        # Associate key with tags
        for tag in tags:
            tag_key = f"tag:{tag}"
            self.cache.sadd(tag_key, key)
            self.cache.expire(tag_key, ttl + 60)  # Tag expires slightly later
    
    def invalidate_by_tag(self, tag):
        tag_key = f"tag:{tag}"
        
        # Get all keys associated with this tag
        keys = self.cache.smembers(tag_key)
        
        if keys:
            # Delete all associated keys
            self.cache.delete(*keys)
            # Delete the tag itself
            self.cache.delete(tag_key)
            print(f"Invalidated {len(keys)} keys with tag '{tag}'")

# Usage
tag_cache = TagBasedCache(redis_client)

# Cache user data with tags
user_data = {"name": "John", "department": "Engineering"}
tag_cache.set_with_tags(
    "user:123", 
    user_data, 
    tags=["user", "department:engineering", "active_users"]
)

# Invalidate all engineering department data
tag_cache.invalidate_by_tag("department:engineering")
```

---

## Distributed Caching with Redis

### 1. Redis Cluster Setup
```python
from rediscluster import RedisCluster

# Redis Cluster configuration
startup_nodes = [
    {"host": "redis-node1", "port": "7000"},
    {"host": "redis-node2", "port": "7000"},
    {"host": "redis-node3", "port": "7000"},
    {"host": "redis-node4", "port": "7000"},
    {"host": "redis-node5", "port": "7000"},
    {"host": "redis-node6", "port": "7000"}
]

# Create cluster client
redis_cluster = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)

class DistributedCache:
    def __init__(self, cluster_client):
        self.cluster = cluster_client
    
    def get(self, key):
        try:
            cached_data = self.cluster.get(key)
            return json.loads(cached_data) if cached_data else None
        except Exception as e:
            print(f"Cache get error: {e}")
            return None
    
    def set(self, key, value, ttl=3600):
        try:
            return self.cluster.setex(key, ttl, json.dumps(value))
        except Exception as e:
            print(f"Cache set error: {e}")
            return False
    
    def delete(self, key):
        try:
            return self.cluster.delete(key)
        except Exception as e:
            print(f"Cache delete error: {e}")
            return False

# Usage
distributed_cache = DistributedCache(redis_cluster)
distributed_cache.set("user:123", {"name": "John"})
```

### 2. Redis Sentinel for High Availability
```python
from redis.sentinel import Sentinel

# Sentinel configuration
sentinels = [
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379)
]

sentinel = Sentinel(sentinels, socket_timeout=0.1)

class HighAvailabilityCache:
    def __init__(self, sentinel, service_name='mymaster'):
        self.sentinel = sentinel
        self.service_name = service_name
    
    def get_master(self):
        return self.sentinel.master_for(self.service_name, socket_timeout=0.1)
    
    def get_slave(self):
        return self.sentinel.slave_for(self.service_name, socket_timeout=0.1)
    
    def set(self, key, value, ttl=3600):
        master = self.get_master()
        return master.setex(key, ttl, json.dumps(value))
    
    def get(self, key):
        # Read from slave for better performance
        slave = self.get_slave()
        try:
            cached_data = slave.get(key)
            return json.loads(cached_data) if cached_data else None
        except:
            # Fallback to master if slave fails
            master = self.get_master()
            cached_data = master.get(key)
            return json.loads(cached_data) if cached_data else None

# Usage
ha_cache = HighAvailabilityCache(sentinel)
ha_cache.set("user:456", {"name": "Jane"})
user = ha_cache.get("user:456")
```

---

## Cache Replacement Policies

### 1. LRU (Least Recently Used) in Redis
```python
# Redis LRU configuration
# In redis.conf:
# maxmemory 2gb
# maxmemory-policy allkeys-lru

class LRUCache:
    def __init__(self, redis_client, max_size=1000):
        self.cache = redis_client
        self.max_size = max_size
        self.access_times = {}
    
    def get(self, key):
        data = self.cache.get(key)
        if data:
            # Update access time
            self.cache.zadd("access_times", {key: time.time()})
            return json.loads(data)
        return None
    
    def set(self, key, value, ttl=3600):
        # Check if we need to evict
        current_size = self.cache.dbsize()
        if current_size >= self.max_size:
            self._evict_lru()
        
        # Set the data
        self.cache.setex(key, ttl, json.dumps(value))
        self.cache.zadd("access_times", {key: time.time()})
    
    def _evict_lru(self):
        # Get least recently used key
        lru_keys = self.cache.zrange("access_times", 0, 0)
        if lru_keys:
            lru_key = lru_keys[0]
            self.cache.delete(lru_key)
            self.cache.zrem("access_times", lru_key)
            print(f"Evicted LRU key: {lru_key}")
```

### 2. LFU (Least Frequently Used)
```python
class LFUCache:
    def __init__(self, redis_client, max_size=1000):
        self.cache = redis_client
        self.max_size = max_size
    
    def get(self, key):
        data = self.cache.get(key)
        if data:
            # Increment frequency counter
            self.cache.zincrby("frequencies", 1, key)
            return json.loads(data)
        return None
    
    def set(self, key, value, ttl=3600):
        current_size = self.cache.dbsize()
        if current_size >= self.max_size:
            self._evict_lfu()
        
        self.cache.setex(key, ttl, json.dumps(value))
        self.cache.zadd("frequencies", {key: 1})
    
    def _evict_lfu(self):
        # Get least frequently used key
        lfu_keys = self.cache.zrange("frequencies", 0, 0)
        if lfu_keys:
            lfu_key = lfu_keys[0]
            self.cache.delete(lfu_key)
            self.cache.zrem("frequencies", lfu_key)
            print(f"Evicted LFU key: {lfu_key}")
```

---

## Redis Implementation Examples

### 1. Session Caching
```python
import uuid
import hashlib

class SessionCache:
    def __init__(self, redis_client):
        self.cache = redis_client
        self.session_ttl = 1800  # 30 minutes
    
    def create_session(self, user_id, user_data):
        session_id = str(uuid.uuid4())
        session_key = f"session:{session_id}"
        
        session_data = {
            'user_id': user_id,
            'user_data': user_data,
            'created_at': time.time(),
            'last_accessed': time.time()
        }
        
        self.cache.setex(session_key, self.session_ttl, json.dumps(session_data))
        return session_id
    
    def get_session(self, session_id):
        session_key = f"session:{session_id}"
        session_data = self.cache.get(session_key)
        
        if session_data:
            session = json.loads(session_data)
            # Update last accessed time
            session['last_accessed'] = time.time()
            self.cache.setex(session_key, self.session_ttl, json.dumps(session))
            return session
        
        return None
    
    def destroy_session(self, session_id):
        session_key = f"session:{session_id}"
        return self.cache.delete(session_key)

# Usage
session_cache = SessionCache(redis_client)
session_id = session_cache.create_session(123, {"name": "John", "role": "admin"})
session = session_cache.get_session(session_id)
```

### 2. Rate Limiting with Redis
```python
class RedisRateLimiter:
    def __init__(self, redis_client):
        self.cache = redis_client
    
    def is_allowed(self, key, limit, window):
        """
        Sliding window rate limiter
        key: identifier (user_id, ip_address, etc.)
        limit: max requests allowed
        window: time window in seconds
        """
        now = time.time()
        pipeline = self.cache.pipeline()
        
        # Remove old entries outside the window
        pipeline.zremrangebyscore(key, 0, now - window)
        
        # Count current requests
        pipeline.zcard(key)
        
        # Add current request
        pipeline.zadd(key, {str(now): now})
        
        # Set expiration
        pipeline.expire(key, int(window))
        
        results = pipeline.execute()
        current_requests = results[1]
        
        return current_requests < limit

# Usage
rate_limiter = RedisRateLimiter(redis_client)

def api_endpoint():
    user_id = get_current_user_id()
    
    if not rate_limiter.is_allowed(f"rate_limit:{user_id}", limit=100, window=3600):
        return {"error": "Rate limit exceeded"}, 429
    
    # Process request
    return {"data": "success"}
```

### 3. Leaderboard with Redis
```python
class Leaderboard:
    def __init__(self, redis_client, name):
        self.cache = redis_client
        self.leaderboard_key = f"leaderboard:{name}"
    
    def add_score(self, user_id, score):
        """Add or update user score"""
        return self.cache.zadd(self.leaderboard_key, {user_id: score})
    
    def get_top_users(self, count=10):
        """Get top N users"""
        return self.cache.zrevrange(self.leaderboard_key, 0, count-1, withscores=True)
    
    def get_user_rank(self, user_id):
        """Get user's rank (1-based)"""
        rank = self.cache.zrevrank(self.leaderboard_key, user_id)
        return rank + 1 if rank is not None else None
    
    def get_user_score(self, user_id):
        """Get user's score"""
        return self.cache.zscore(self.leaderboard_key, user_id)
    
    def get_users_around(self, user_id, count=5):
        """Get users around a specific user"""
        rank = self.cache.zrevrank(self.leaderboard_key, user_id)
        if rank is None:
            return []
        
        start = max(0, rank - count // 2)
        end = start + count - 1
        
        return self.cache.zrevrange(self.leaderboard_key, start, end, withscores=True)

# Usage
leaderboard = Leaderboard(redis_client, "game_scores")
leaderboard.add_score("user123", 1500)
leaderboard.add_score("user456", 2000)

top_players = leaderboard.get_top_users(10)
user_rank = leaderboard.get_user_rank("user123")
```

---

## Performance Considerations

### 1. Cache Warming
```python
class CacheWarmer:
    def __init__(self, redis_client, database):
        self.cache = redis_client
        self.database = database
    
    def warm_user_cache(self, user_ids):
        """Pre-populate cache with user data"""
        pipeline = self.cache.pipeline()
        
        for user_id in user_ids:
            user_data = self.database.get_user(user_id)
            if user_data:
                cache_key = f"user:{user_id}"
                pipeline.setex(cache_key, 3600, json.dumps(user_data))
        
        pipeline.execute()
        print(f"Warmed cache for {len(user_ids)} users")
    
    def warm_popular_content(self):
        """Pre-populate cache with popular content"""
        popular_posts = self.database.get_popular_posts(limit=100)
        
        pipeline = self.cache.pipeline()
        for post in popular_posts:
            cache_key = f"post:{post['id']}"
            pipeline.setex(cache_key, 1800, json.dumps(post))  # 30 min TTL
        
        pipeline.execute()
        print(f"Warmed cache for {len(popular_posts)} popular posts")

# Usage - typically run during deployment or low-traffic periods
cache_warmer = CacheWarmer(redis_client, database)
cache_warmer.warm_user_cache([1, 2, 3, 4, 5])
cache_warmer.warm_popular_content()
```

### 2. Cache Monitoring
```python
class CacheMonitor:
    def __init__(self, redis_client):
        self.cache = redis_client
        self.stats = {
            'hits': 0,
            'misses': 0,
            'sets': 0,
            'deletes': 0
        }
    
    def record_hit(self):
        self.stats['hits'] += 1
    
    def record_miss(self):
        self.stats['misses'] += 1
    
    def record_set(self):
        self.stats['sets'] += 1
    
    def record_delete(self):
        self.stats['deletes'] += 1
    
    def get_hit_ratio(self):
        total = self.stats['hits'] + self.stats['misses']
        return self.stats['hits'] / total if total > 0 else 0
    
    def get_redis_info(self):
        """Get Redis server information"""
        info = self.cache.info()
        return {
            'used_memory': info['used_memory_human'],
            'connected_clients': info['connected_clients'],
            'total_commands_processed': info['total_commands_processed'],
            'keyspace_hits': info['keyspace_hits'],
            'keyspace_misses': info['keyspace_misses'],
            'redis_hit_ratio': info['keyspace_hits'] / (info['keyspace_hits'] + info['keyspace_misses']) if (info['keyspace_hits'] + info['keyspace_misses']) > 0 else 0
        }
    
    def print_stats(self):
        redis_info = self.get_redis_info()
        print(f"""
Cache Statistics:
- Application Hit Ratio: {self.get_hit_ratio():.2%}
- Redis Hit Ratio: {redis_info['redis_hit_ratio']:.2%}
- Memory Usage: {redis_info['used_memory']}
- Connected Clients: {redis_info['connected_clients']}
- Total Commands: {redis_info['total_commands_processed']}
        """)

# Usage
monitor = CacheMonitor(redis_client)

def monitored_cache_get(key):
    data = redis_client.get(key)
    if data:
        monitor.record_hit()
        return json.loads(data)
    else:
        monitor.record_miss()
        return None

# Print stats periodically
monitor.print_stats()
```

---

## Interview Questions

### Q1: Explain different caching patterns and when to use each.

**Answer Framework:**
1. **Cache-Aside**: Application manages cache, good for read-heavy workloads
2. **Write-Through**: Writes to cache and DB simultaneously, ensures consistency
3. **Write-Behind**: Writes to cache immediately, DB asynchronously, best performance
4. **Refresh-Ahead**: Proactively refreshes cache, prevents cache miss penalty

### Q2: How would you implement a distributed cache with Redis?

**Key Points:**
- **Redis Cluster**: Automatic sharding across multiple nodes
- **Redis Sentinel**: High availability with master-slave replication
- **Consistent Hashing**: For custom sharding logic
- **Replication**: Data redundancy and read scaling

### Q3: How do you handle cache invalidation in a distributed system?

**Solutions:**
1. **TTL-based**: Simple expiration times
2. **Event-driven**: Invalidate on data changes
3. **Tag-based**: Group related cache entries
4. **Version-based**: Use versioned cache keys

### Q4: What are the trade-offs between different cache replacement policies?

**Comparison:**
- **LRU**: Good for temporal locality, complex to implement
- **LFU**: Good for frequency patterns, requires frequency tracking
- **FIFO**: Simple but ignores access patterns
- **Random**: Simple, surprisingly effective in some cases

### Q5: How would you design a caching strategy for a social media application?

**Architecture:**
1. **Multi-level caching**: Browser → CDN → Application → Database
2. **User data**: Cache-aside with Redis
3. **Timeline**: Write-through with background refresh
4. **Popular content**: Proactive caching with higher TTL
5.
