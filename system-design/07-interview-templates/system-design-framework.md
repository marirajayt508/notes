# System Design Interview Framework

## Table of Contents
1. [Interview Process Overview](#interview-process-overview)
2. [Step-by-Step Framework](#step-by-step-framework)
3. [Requirements Gathering Template](#requirements-gathering-template)
4. [Capacity Estimation Template](#capacity-estimation-template)
5. [High-Level Design Template](#high-level-design-template)
6. [Deep Dive Template](#deep-dive-template)
7. [Common Patterns & Solutions](#common-patterns--solutions)
8. [Interview Tips & Best Practices](#interview-tips--best-practices)

---

## Interview Process Overview

### Typical Timeline (45-60 minutes)
```
5-10 min: Requirements gathering and clarification
10-15 min: Capacity estimation and constraints
15-20 min: High-level system design
15-20 min: Detailed component design and deep dive
5-10 min: Scaling, monitoring, and wrap-up
```

### What Interviewers Look For
1. **Structured thinking** - Systematic approach to problem-solving
2. **Trade-off analysis** - Understanding of different design choices
3. **Scalability awareness** - Ability to design for growth
4. **Real-world experience** - Practical knowledge of systems
5. **Communication skills** - Clear explanation of complex concepts

---

## Step-by-Step Framework

### Phase 1: Clarify Requirements (5-10 minutes)

#### Questions to Ask
```
Functional Requirements:
- What are the core features we need to support?
- Who are the users of this system?
- What are the main use cases?
- Are there any specific features to prioritize?

Non-Functional Requirements:
- How many users do we expect?
- What's the expected read/write ratio?
- What are the latency requirements?
- What's the availability requirement?
- Are there any consistency requirements?
- Any specific technology constraints?
```

#### Template Response
```
"Let me clarify the requirements:

Functional Requirements:
1. [Feature 1] - [Brief description]
2. [Feature 2] - [Brief description]
3. [Feature 3] - [Brief description]

Non-Functional Requirements:
- Scale: [X] users, [Y] requests/day
- Performance: [Z]ms latency, [A]% availability
- Consistency: [Strong/Eventual] for [specific data]

Is this understanding correct?"
```

### Phase 2: Capacity Estimation (10-15 minutes)

#### Estimation Framework
```
1. User Statistics
   - Daily Active Users (DAU)
   - Peak usage patterns
   - Geographic distribution

2. Traffic Estimation
   - Requests per second (RPS)
   - Read vs Write ratio
   - Peak traffic multiplier

3. Storage Estimation
   - Data per user/request
   - Growth rate
   - Retention period

4. Bandwidth Estimation
   - Average request/response size
   - Total bandwidth requirements
```

#### Sample Calculations
```python
# Example: Social Media Platform
DAU = 100_000_000
avg_posts_per_user_per_day = 2
read_write_ratio = 100  # 100 reads per 1 write

# Write Traffic
writes_per_day = DAU * avg_posts_per_user_per_day
writes_per_second = writes_per_day / (24 * 3600)
peak_writes_per_second = writes_per_second * 3  # 3x peak factor

# Read Traffic  
reads_per_second = writes_per_second * read_write_ratio
peak_reads_per_second = reads_per_second * 3

# Storage
avg_post_size = 1000  # bytes
daily_storage = writes_per_day * avg_post_size
annual_storage = daily_storage * 365

print(f"Peak writes: {peak_writes_per_second:,.0f} RPS")
print(f"Peak reads: {peak_reads_per_second:,.0f} RPS") 
print(f"Annual storage: {annual_storage / (1024**4):.1f} TB")
```

### Phase 3: High-Level Design (15-20 minutes)

#### Design Components Checklist
```
□ Load Balancer
□ API Gateway
□ Application Servers
□ Databases (Primary/Replica)
□ Caching Layer
□ Message Queues
□ File Storage
□ CDN
□ Monitoring & Logging
```

#### Architecture Template
```
[Client] → [Load Balancer] → [API Gateway] → [Services]
                                                ↓
[Cache] ← [Application Servers] → [Message Queue]
   ↓              ↓                      ↓
[Database] → [File Storage] ← [Background Workers]
   ↓              ↓
[Monitoring] ← [CDN]
```

### Phase 4: Detailed Design (15-20 minutes)

#### Deep Dive Areas
1. **Database Design**
   - Schema design
   - Indexing strategy
   - Sharding approach
   - Replication setup

2. **API Design**
   - RESTful endpoints
   - Request/response formats
   - Authentication/authorization
   - Rate limiting

3. **Caching Strategy**
   - Cache levels
   - Cache keys
   - Invalidation strategy
   - Cache-aside vs Write-through

4. **Scalability Patterns**
   - Horizontal vs vertical scaling
   - Microservices decomposition
   - Async processing
   - Circuit breakers

---

## Requirements Gathering Template

### Functional Requirements Checklist

#### User Management
```
□ User registration/authentication
□ User profiles and settings
□ User roles and permissions
□ Social features (follow/friend)
```

#### Core Features
```
□ Create/Read/Update/Delete operations
□ Search functionality
□ Filtering and sorting
□ Real-time updates
□ Notifications
```

#### Content Management
```
□ Text content
□ Media content (images/videos)
□ File uploads
□ Content moderation
```

### Non-Functional Requirements Template

#### Scale Requirements
```
Users:
- Monthly Active Users (MAU): ___
- Daily Active Users (DAU): ___
- Peak concurrent users: ___

Traffic:
- Requests per day: ___
- Peak requests per second: ___
- Read/Write ratio: ___

Data:
- Data per user: ___
- Total data volume: ___
- Growth rate: ___% per year
```

#### Performance Requirements
```
Latency:
- API response time: < ___ ms
- Page load time: < ___ ms
- Search response time: < ___ ms

Throughput:
- Requests per second: ___
- Concurrent connections: ___

Availability:
- Uptime requirement: ___%
- Recovery time objective (RTO): ___
- Recovery point objective (RPO): ___
```

---

## Capacity Estimation Template

### Traffic Estimation Formulas

#### Basic Calculations
```python
# Users and Activity
DAU = monthly_active_users * 0.3  # Typical conversion
peak_concurrent_users = DAU * 0.1  # 10% online simultaneously

# Request Volume
daily_requests = DAU * avg_requests_per_user_per_day
requests_per_second = daily_requests / (24 * 3600)
peak_rps = requests_per_second * peak_multiplier  # Usually 2-5x

# Read/Write Split
write_rps = peak_rps / (read_write_ratio + 1)
read_rps = peak_rps - write_rps
```

#### Storage Calculations
```python
# Data Storage
user_data_size = avg_user_profile_size * total_users
content_data_size = avg_content_size * total_content_items
metadata_size = content_data_size * 0.1  # 10% overhead

total_storage = user_data_size + content_data_size + metadata_size
storage_with_replication = total_storage * replication_factor
annual_growth = storage_with_replication * growth_rate
```

#### Bandwidth Calculations
```python
# Network Bandwidth
avg_request_size = 1024  # 1KB
avg_response_size = 5120  # 5KB

ingress_bandwidth = write_rps * avg_request_size
egress_bandwidth = read_rps * avg_response_size
total_bandwidth = ingress_bandwidth + egress_bandwidth
```

### Quick Reference Numbers

#### Typical System Metrics
```
Web Application:
- Response time: < 200ms
- Availability: 99.9% (8.77 hours downtime/year)
- Concurrent users: 10% of DAU

Mobile Application:
- Response time: < 100ms
- Offline capability required
- Push notifications

Real-time System:
- Response time: < 50ms
- WebSocket connections
- Event streaming
```

#### Storage Estimates
```
Text Data:
- Tweet: ~280 bytes
- Blog post: ~5KB
- User profile: ~1KB

Media Data:
- Profile image: ~100KB
- Photo: ~1MB
- Video (1 min): ~10MB

Database Overhead:
- Indexes: 20-30% of data size
- Replication: 2-3x storage
- Backup: 1x additional storage
```

---

## High-Level Design Template

### Standard Architecture Patterns

#### Three-Tier Architecture
```
Presentation Tier:
- Web browsers
- Mobile apps
- API clients

Application Tier:
- Load balancers
- Web servers
- Application servers
- API gateways

Data Tier:
- Databases
- File storage
- Caching systems
```

#### Microservices Architecture
```
Client Layer:
- Web/Mobile clients
- Third-party integrations

Gateway Layer:
- API Gateway
- Load balancer
- Authentication service

Service Layer:
- User service
- Content service
- Notification service
- Search service

Data Layer:
- Service-specific databases
- Shared cache
- Message queues
```

### Component Selection Guide

#### Load Balancing
```
Layer 4 (Transport):
- AWS ALB, NGINX
- Simple, fast
- TCP/UDP load balancing

Layer 7 (Application):
- AWS ALB, HAProxy
- Content-based routing
- SSL termination
```

#### Databases
```
Relational (ACID):
- PostgreSQL, MySQL
- Complex queries
- Strong consistency

NoSQL Document:
- MongoDB, DynamoDB
- Flexible schema
- Horizontal scaling

NoSQL Key-Value:
- Redis, DynamoDB
- Simple operations
- High performance

NoSQL Graph:
- Neo4j, Amazon Neptune
- Relationship queries
- Social networks
```

#### Caching
```
In-Memory:
- Redis, Memcached
- Sub-millisecond latency
- Session storage

CDN:
- CloudFront, Cloudflare
- Geographic distribution
- Static content

Application Cache:
- Caffeine, Guava
- Local caching
- Reduced network calls
```

---

## Deep Dive Template

### Database Design Deep Dive

#### Schema Design Process
```
1. Identify Entities
   - Users, Posts, Comments, etc.
   - Define attributes for each entity

2. Define Relationships
   - One-to-one, one-to-many, many-to-many
   - Foreign key constraints

3. Normalization
   - Eliminate data redundancy
   - Balance between normalization and performance

4. Indexing Strategy
   - Primary keys, foreign keys
   - Query-based indexes
   - Composite indexes
```

#### Sharding Strategies
```python
# Hash-based Sharding
def get_shard(key, num_shards):
    return hash(key) % num_shards

# Range-based Sharding
def get_shard_by_range(key, ranges):
    for i, range_end in enumerate(ranges):
        if key <= range_end:
            return i
    return len(ranges) - 1

# Directory-based Sharding
shard_directory = {
    'user_1_1000': 'shard_1',
    'user_1001_2000': 'shard_2',
    # ...
}
```

### API Design Deep Dive

#### RESTful API Design
```http
# Resource-based URLs
GET    /api/v1/users           # List users
POST   /api/v1/users           # Create user
GET    /api/v1/users/{id}      # Get user
PUT    /api/v1/users/{id}      # Update user
DELETE /api/v1/users/{id}      # Delete user

# Nested resources
GET    /api/v1/users/{id}/posts     # User's posts
POST   /api/v1/users/{id}/posts     # Create post for user

# Query parameters
GET    /api/v1/posts?limit=20&offset=0&sort=created_at
```

#### Error Handling
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "request_id": "req_12345"
  }
}
```

### Caching Deep Dive

#### Cache Patterns
```python
# Cache-Aside (Lazy Loading)
def get_user(user_id):
    # Try cache first
    user = cache.get(f"user:{user_id}")
    if user is None:
        # Cache miss - load from database
        user = database.get_user(user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
    return user

# Write-Through
def update_user(user_id, user_data):
    # Update database
    database.update_user(user_id, user_data)
    # Update cache
    cache.set(f"user:{user_id}", user_data, ttl=3600)

# Write-Behind (Write-Back)
def update_user_async(user_id, user_data):
    # Update cache immediately
    cache.set(f"user:{user_id}", user_data, ttl=3600)
    # Queue database update
    queue.enqueue('update_user_db', user_id, user_data)
```

#### Cache Invalidation Strategies
```python
# TTL-based
cache.set(key, value, ttl=3600)  # Expire after 1 hour

# Event-based
def on_user_update(user_id):
    cache.delete(f"user:{user_id}")
    cache.delete(f"user_posts:{user_id}")

# Version-based
def get_user_with_version(user_id):
    version = get_user_version(user_id)
    cache_key = f"user:{user_id}:v{version}"
    return cache.get(cache_key)
```

---

## Common Patterns & Solutions

### Scalability Patterns

#### Horizontal Scaling
```
Stateless Services:
- No server-side session state
- Load balancer can route to any server
- Easy to add/remove servers

Database Sharding:
- Partition data across multiple databases
- Each shard handles subset of data
- Requires shard key selection

Microservices:
- Decompose monolith into services
- Independent scaling and deployment
- Service mesh for communication
```

#### Vertical Scaling
```
Hardware Upgrades:
- More CPU cores
- More RAM
- Faster storage (SSD)

Software Optimization:
- Query optimization
- Code profiling and optimization
- Connection pooling
```

### Reliability Patterns

#### Circuit Breaker
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

#### Retry with Exponential Backoff
```python
import time
import random

def retry_with_backoff(func, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            
            # Exponential backoff with jitter
            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)
```

### Performance Patterns

#### Connection Pooling
```python
class ConnectionPool:
    def __init__(self, max_connections=10):
        self.max_connections = max_connections
        self.pool = queue.Queue(maxsize=max_connections)
        self.active_connections = 0
    
    def get_connection(self):
        try:
            return self.pool.get_nowait()
        except queue.Empty:
            if self.active_connections < self.max_connections:
                self.active_connections += 1
                return self.create_connection()
            else:
                return self.pool.get()  # Wait for available connection
    
    def return_connection(self, conn):
        self.pool.put(conn)
```

#### Async Processing
```python
# Message Queue Pattern
def handle_user_signup(user_data):
    # Immediate response
    user_id = create_user(user_data)
    
    # Async tasks
    queue.enqueue('send_welcome_email', user_id)
    queue.enqueue('create_user_profile', user_id)
    queue.enqueue('update_analytics', user_id)
    
    return {'user_id': user_id, 'status': 'created'}

# Background Worker
def background_worker():
    while True:
        task = queue.dequeue()
        if task:
            process_task(task)
        else:
            time.sleep(1)
```

---

## Interview Tips & Best Practices

### Communication Tips

#### Start with Clarification
```
"Before I start designing, let me clarify a few things:
1. What are the core features we need to support?
2. What's the expected scale in terms of users and requests?
3. Are there any specific constraints or requirements?"
```

#### Think Out Loud
```
"I'm thinking about the database design here. We have a few options:
1. Single relational database - simple but may not scale
2. Sharded database - better scalability but more complex
3. NoSQL - flexible but eventual consistency

Given our scale requirements, I think sharding would be the best approach because..."
```

#### Acknowledge Trade-offs
```
"There are trade-offs with this approach:
- Pros: Fast reads, good user experience
- Cons: More complex writes, potential consistency issues
- Alternative: We could use [alternative approach] which would give us [different trade-offs]"
```

### Technical Tips

#### Start Simple, Then Scale
```
1. Begin with monolithic architecture
2. Identify bottlenecks
3. Add caching layer
4. Introduce load balancing
5. Consider microservices
6. Add monitoring and alerting
```

#### Use Numbers to Validate Design
```
"Let me validate this design:
- We estimated 10,000 RPS peak load
- Each server can handle 1,000 RPS
- So we need at least 10 servers
- With 2x redundancy, that's 20 servers
- This seems reasonable for our scale"
```

#### Consider Failure Scenarios
```
"What happens if:
1. Database goes down? → Use read replicas and failover
2. Cache fails? → Graceful degradation to database
3. Service becomes unavailable? → Circuit breaker pattern
4. Network partition? → Design for eventual consistency"
```

### Common Mistakes to Avoid

#### Don't Jump to Solutions
```
❌ "We need microservices and Kubernetes"
✅ "Let's start with the requirements and see what architecture fits best"
```

#### Don't Over-Engineer
```
❌ Complex distributed system for 1000 users
✅ Simple architecture that can evolve with growth
```

#### Don't Ignore Non-Functional Requirements
```
❌ Focus only on features
✅ Consider scalability, reliability, and performance
```

#### Don't Forget to Validate
```
❌ Design without checking if it meets requirements
✅ Walk through key scenarios and validate capacity
```

### Time Management

#### 45-Minute Interview Breakdown
```
0-5 min:   Requirements clarification
5-15 min:  Capacity estimation and constraints
15-30 min: High-level design and API design
30-40 min: Deep dive into 1-2 components
40-45 min: Scaling, monitoring, wrap-up
```

#### 60-Minute Interview Breakdown
```
0-10 min:  Requirements clarification
10-20 min: Capacity estimation and constraints
20-35 min: High-level design and API design
35-50 min: Deep dive into 2-3 components
50-60 min: Scaling, monitoring, failure scenarios
```

---

## Key Takeaways

### Essential Principles
1. **Clarify before designing** - Understand the problem fully
2. **Start simple, evolve complexity** - Don't over-engineer initially
3. **Think about trade-offs** - Every decision has pros and cons
4. **Validate with numbers** - Use capacity estimation to guide design
5. **Consider failure scenarios** - Design for reliability
6. **Communicate clearly** - Explain your thought process

### Success Factors
1. **Structured approach** - Follow a consistent framework
2. **Practical experience** - Draw from real-world knowledge
3. **Balanced design** - Consider all aspects (performance, scalability, reliability)
4. **Clear communication** - Make complex concepts understandable
5. **Adaptability** - Adjust design based on feedback and constraints

Remember: System design interviews are about demonstrating your ability to think through complex problems systematically. Use this framework as a guide, but be prepared to adapt based on the specific problem and interviewer feedback.
