# Load Balancing - System Design Fundamentals

## Table of Contents
1. [What is Load Balancing?](#what-is-load-balancing)
2. [Types of Load Balancers](#types-of-load-balancers)
3. [Load Balancing Algorithms](#load-balancing-algorithms)
4. [Layer 4 vs Layer 7 Load Balancing](#layer-4-vs-layer-7-load-balancing)
5. [Health Checks and Failover](#health-checks-and-failover)
6. [Session Persistence](#session-persistence)
7. [Global Load Balancing](#global-load-balancing)
8. [Implementation Examples](#implementation-examples)
9. [Interview Questions](#interview-questions)

---

## What is Load Balancing?

**Load balancing** is the process of distributing incoming network traffic across multiple servers to ensure no single server becomes overwhelmed, improving application availability, scalability, and performance.

### Key Benefits:
- **High Availability**: Eliminates single points of failure
- **Scalability**: Distributes load across multiple servers
- **Performance**: Reduces response time by distributing requests
- **Flexibility**: Easy to add/remove servers
- **Fault Tolerance**: Automatic failover to healthy servers

### Basic Load Balancer Architecture:
```
[Client Requests] → [Load Balancer] → [Server Pool]
                                    ├── Server 1
                                    ├── Server 2
                                    ├── Server 3
                                    └── Server N
```

---

## Types of Load Balancers

### 1. Hardware Load Balancers
**Physical appliances** dedicated to load balancing.

**Examples**: F5 BIG-IP, Citrix NetScaler, A10 Networks

**Pros:**
- High performance and throughput
- Advanced features (SSL offloading, compression)
- Dedicated hardware optimization
- Enterprise support

**Cons:**
- Expensive initial cost
- Limited scalability
- Vendor lock-in
- Complex configuration

**Use Cases:**
- Large enterprises with high traffic
- Applications requiring specialized features
- Environments with strict performance requirements

### 2. Software Load Balancers
**Software-based** solutions running on standard hardware.

**Examples**: NGINX, HAProxy, Apache HTTP Server, Traefik

**Pros:**
- Cost-effective
- Flexible and customizable
- Easy to scale horizontally
- Open-source options available

**Cons:**
- Performance limited by underlying hardware
- Requires more management overhead
- May need additional monitoring

**Use Cases:**
- Startups and small to medium businesses
- Cloud-native applications
- Microservices architectures

### 3. Cloud Load Balancers
**Managed services** provided by cloud providers.

**Examples**: 
- AWS: Application Load Balancer (ALB), Network Load Balancer (NLB)
- Google Cloud: Cloud Load Balancing
- Azure: Azure Load Balancer, Application Gateway

**Pros:**
- Fully managed service
- Auto-scaling capabilities
- Integration with cloud services
- Pay-as-you-use pricing

**Cons:**
- Vendor lock-in
- Less control over configuration
- Potential latency in multi-cloud setups

**Use Cases:**
- Cloud-first applications
- Applications requiring auto-scaling
- Teams preferring managed services

---

## Load Balancing Algorithms

### 1. Round Robin
Distributes requests sequentially across servers.

```python
class RoundRobinBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.current = 0
    
    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server

# Example usage
balancer = RoundRobinBalancer(['server1', 'server2', 'server3'])
print(balancer.get_server())  # server1
print(balancer.get_server())  # server2
print(balancer.get_server())  # server3
print(balancer.get_server())  # server1 (cycles back)
```

**Pros:**
- Simple to implement
- Equal distribution of requests
- Good for servers with similar capacity

**Cons:**
- Doesn't consider server load or capacity
- May not be optimal for varying request processing times

**Best For:** Homogeneous server environments with similar request patterns

### 2. Weighted Round Robin
Assigns different weights to servers based on their capacity.

```python
class WeightedRoundRobinBalancer:
    def __init__(self, servers_weights):
        self.servers_weights = servers_weights  # [('server1', 3), ('server2', 2), ('server3', 1)]
        self.current_weights = [0] * len(servers_weights)
        self.total_weight = sum(weight for _, weight in servers_weights)
    
    def get_server(self):
        # Find server with highest current weight
        max_weight_index = 0
        for i in range(1, len(self.current_weights)):
            if self.current_weights[i] > self.current_weights[max_weight_index]:
                max_weight_index = i
        
        # Select server and adjust weights
        server, weight = self.servers_weights[max_weight_index]
        self.current_weights[max_weight_index] -= self.total_weight
        
        # Add original weight to all servers
        for i in range(len(self.current_weights)):
            self.current_weights[i] += self.servers_weights[i][1]
        
        return server
```

**Use Cases:**
- Servers with different capacities
- Gradual traffic shifting during deployments
- A/B testing with different traffic ratios

### 3. Least Connections
Routes requests to the server with the fewest active connections.

```python
class LeastConnectionsBalancer:
    def __init__(self, servers):
        self.servers = {server: 0 for server in servers}  # server -> connection_count
    
    def get_server(self):
        # Find server with minimum connections
        return min(self.servers.keys(), key=lambda s: self.servers[s])
    
    def add_connection(self, server):
        self.servers[server] += 1
    
    def remove_connection(self, server):
        self.servers[server] = max(0, self.servers[server] - 1)
```

**Pros:**
- Better for long-lived connections
- Adapts to varying request processing times
- More even distribution under varying loads

**Cons:**
- Requires tracking connection state
- More complex implementation
- Overhead of connection counting

**Best For:** Applications with long-lived connections (WebSockets, database connections)

### 4. Least Response Time
Routes to the server with the lowest average response time.

```python
import time
from collections import deque

class LeastResponseTimeBalancer:
    def __init__(self, servers, window_size=100):
        self.servers = servers
        self.response_times = {server: deque(maxlen=window_size) for server in servers}
        self.active_requests = {server: 0 for server in servers}
    
    def get_server(self):
        best_server = None
        best_score = float('inf')
        
        for server in self.servers:
            # Calculate average response time
            if self.response_times[server]:
                avg_response_time = sum(self.response_times[server]) / len(self.response_times[server])
            else:
                avg_response_time = 0
            
            # Factor in active connections
            score = avg_response_time * (1 + self.active_requests[server])
            
            if score < best_score:
                best_score = score
                best_server = server
        
        self.active_requests[best_server] += 1
        return best_server
    
    def record_response_time(self, server, response_time):
        self.response_times[server].append(response_time)
        self.active_requests[server] = max(0, self.active_requests[server] - 1)
```

**Best For:** Applications where response time varies significantly between servers

### 5. IP Hash
Routes requests based on client IP hash.

```python
import hashlib

class IPHashBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self, client_ip):
        # Hash client IP and map to server
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        return self.servers[hash_value % len(self.servers)]
```

**Pros:**
- Session persistence without server-side state
- Consistent routing for same client
- Simple implementation

**Cons:**
- Uneven distribution if client IPs are not well distributed
- Difficult to handle server failures
- Not adaptive to server load

**Best For:** Applications requiring session affinity

### 6. Random
Randomly selects a server for each request.

```python
import random

class RandomBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self):
        return random.choice(self.servers)
```

**Pros:**
- Simple implementation
- No state to maintain
- Good distribution over time

**Cons:**
- May have short-term imbalances
- Doesn't consider server capacity or load

**Best For:** Stateless applications with homogeneous servers

---

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer) Load Balancing

**Operates at the transport layer** (TCP/UDP level).

```
Client → [L4 Load Balancer] → Server
         (Routes based on IP + Port)
```

**Characteristics:**
- Routes based on IP address and port
- No inspection of application data
- Lower latency and higher throughput
- Protocol agnostic (TCP, UDP)

**Example Configuration (NGINX):**
```nginx
stream {
    upstream backend {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
        server 192.168.1.12:8080;
    }
    
    server {
        listen 80;
        proxy_pass backend;
        proxy_timeout 1s;
        proxy_responses 1;
    }
}
```

**Use Cases:**
- High-performance applications
- Non-HTTP protocols
- Simple load distribution
- When application content inspection is not needed

### Layer 7 (Application Layer) Load Balancing

**Operates at the application layer** (HTTP/HTTPS level).

```
Client → [L7 Load Balancer] → Server
         (Routes based on HTTP headers, URL, content)
```

**Characteristics:**
- Routes based on application data (HTTP headers, URLs, cookies)
- Can modify requests/responses
- SSL termination capabilities
- Content-based routing

**Example Configuration (NGINX):**
```nginx
http {
    upstream api_servers {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
    }
    
    upstream web_servers {
        server 192.168.1.20:8080;
        server 192.168.1.21:8080;
    }
    
    server {
        listen 80;
        
        location /api/ {
            proxy_pass http://api_servers;
        }
        
        location / {
            proxy_pass http://web_servers;
        }
    }
}
```

**Advanced L7 Features:**
```nginx
# Content-based routing
location /images/ {
    proxy_pass http://image_servers;
}

location /videos/ {
    proxy_pass http://video_servers;
}

# Header-based routing
if ($http_user_agent ~* "mobile") {
    proxy_pass http://mobile_servers;
}

# Geographic routing
if ($geoip_country_code = "US") {
    proxy_pass http://us_servers;
}
```

**Use Cases:**
- Web applications requiring content-based routing
- Microservices architectures
- SSL termination
- Request/response modification

### Comparison Table

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| **Performance** | Higher throughput, lower latency | Lower throughput, higher latency |
| **Routing Logic** | IP + Port | HTTP headers, URL, content |
| **Protocol Support** | Any TCP/UDP | HTTP/HTTPS primarily |
| **SSL Termination** | Pass-through only | Full SSL termination |
| **Content Inspection** | No | Yes |
| **Caching** | No | Yes |
| **Complexity** | Lower | Higher |
| **Cost** | Lower | Higher |

---

## Health Checks and Failover

### Health Check Types

#### 1. Passive Health Checks
Monitor actual request failures to determine server health.

```python
class PassiveHealthCheck:
    def __init__(self, failure_threshold=3, recovery_threshold=2):
        self.failure_threshold = failure_threshold
        self.recovery_threshold = recovery_threshold
        self.server_stats = {}  # server -> {'failures': 0, 'successes': 0, 'healthy': True}
    
    def record_request(self, server, success):
        if server not in self.server_stats:
            self.server_stats[server] = {'failures': 0, 'successes': 0, 'healthy': True}
        
        stats = self.server_stats[server]
        
        if success:
            stats['successes'] += 1
            stats['failures'] = 0  # Reset failure count on success
            
            # Mark as healthy if enough successes
            if not stats['healthy'] and stats['successes'] >= self.recovery_threshold:
                stats['healthy'] = True
                print(f"Server {server} marked as healthy")
        else:
            stats['failures'] += 1
            stats['successes'] = 0  # Reset success count on failure
            
            # Mark as unhealthy if too many failures
            if stats['healthy'] and stats['failures'] >= self.failure_threshold:
                stats['healthy'] = False
                print(f"Server {server} marked as unhealthy")
    
    def is_healthy(self, server):
        return self.server_stats.get(server, {}).get('healthy', True)
```

#### 2. Active Health Checks
Proactively send health check requests to servers.

```python
import asyncio
import aiohttp
import time

class ActiveHealthCheck:
    def __init__(self, servers, check_interval=30, timeout=5):
        self.servers = servers
        self.check_interval = check_interval
        self.timeout = timeout
        self.server_health = {server: True for server in servers}
        self.last_check = {server: 0 for server in servers}
    
    async def check_server_health(self, server):
        try:
            async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=self.timeout)) as session:
                async with session.get(f"http://{server}/health") as response:
                    if response.status == 200:
                        self.server_health[server] = True
                        return True
        except Exception as e:
            print(f"Health check failed for {server}: {e}")
        
        self.server_health[server] = False
        return False
    
    async def run_health_checks(self):
        while True:
            tasks = []
            for server in self.servers:
                if time.time() - self.last_check[server] >= self.check_interval:
                    tasks.append(self.check_server_health(server))
                    self.last_check[server] = time.time()
            
            if tasks:
                await asyncio.gather(*tasks)
            
            await asyncio.sleep(1)  # Check every second for due health checks
    
    def get_healthy_servers(self):
        return [server for server, healthy in self.server_health.items() if healthy]
```

### Health Check Endpoints

#### Simple Health Check
```python
from flask import Flask, jsonify
import psutil
import time

app = Flask(__name__)

@app.route('/health')
def health_check():
    return jsonify({
        'status': 'healthy',
        'timestamp': time.time(),
        'version': '1.0.0'
    })

@app.route('/health/detailed')
def detailed_health_check():
    # Check various system metrics
    cpu_percent = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    
    # Define health thresholds
    healthy = (
        cpu_percent < 80 and
        memory.percent < 85 and
        disk.percent < 90
    )
    
    return jsonify({
        'status': 'healthy' if healthy else 'unhealthy',
        'timestamp': time.time(),
        'metrics': {
            'cpu_percent': cpu_percent,
            'memory_percent': memory.percent,
            'disk_percent': disk.percent
        },
        'checks': {
            'database': check_database_connection(),
            'cache': check_cache_connection(),
            'external_api': check_external_api()
        }
    })

def check_database_connection():
    try:
        # Attempt database connection
        # db.execute("SELECT 1")
        return {'status': 'healthy', 'response_time_ms': 5}
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}

def check_cache_connection():
    try:
        # Attempt cache connection
        # cache.ping()
        return {'status': 'healthy', 'response_time_ms': 2}
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}

def check_external_api():
    try:
        # Check external API dependency
        return {'status': 'healthy', 'response_time_ms': 100}
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}
```

---

## Session Persistence

### Sticky Sessions (Session Affinity)
Route requests from the same client to the same server.

#### Cookie-Based Persistence
```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        
        # Enable sticky sessions based on cookie
        sticky cookie srv_id expires=1h domain=.example.com path=/;
    }
}
```

#### IP-Based Persistence
```nginx
upstream backend {
    ip_hash;  # Route based on client IP
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

### Session Sharing
Store session data externally so any server can handle requests.

#### Redis Session Store
```python
import redis
import json
from flask import Flask, session, request

app = Flask(__name__)
redis_client = redis.Redis(host='redis-server', port=6379, db=0)

class RedisSessionInterface:
    def __init__(self, redis_client, key_prefix='session:'):
        self.redis = redis_client
        self.key_prefix = key_prefix
    
    def get_session_data(self, session_id):
        key = f"{self.key_prefix}{session_id}"
        data = self.redis.get(key)
        return json.loads(data) if data else {}
    
    def set_session_data(self, session_id, data, ttl=3600):
        key = f"{self.key_prefix}{session_id}"
        self.redis.setex(key, ttl, json.dumps(data))
    
    def delete_session(self, session_id):
        key = f"{self.key_prefix}{session_id}"
        self.redis.delete(key)

# Usage in application
session_store = RedisSessionInterface(redis_client)

@app.route('/login', methods=['POST'])
def login():
    username = request.json.get('username')
    # Authenticate user...
    
    session_id = generate_session_id()
    session_data = {
        'user_id': user_id,
        'username': username,
        'login_time': time.time()
    }
    
    session_store.set_session_data(session_id, session_data)
    return jsonify({'session_id': session_id})
```

---

## Global Load Balancing

### DNS-Based Load Balancing
Use DNS to distribute traffic across multiple data centers.

```python
# DNS configuration example
# A records for different regions
www.example.com.    300    IN    A    1.2.3.4      # US East
www.example.com.    300    IN    A    5.6.7.8      # US West
www.example.com.    300    IN    A    9.10.11.12   # Europe
www.example.com.    300    IN    A    13.14.15.16  # Asia
```

#### GeoDNS Implementation
```python
import geoip2.database
from dns import resolver, rrset, rdatatype

class GeoDNSLoadBalancer:
    def __init__(self, geoip_db_path):
        self.reader = geoip2.database.Reader(geoip_db_path)
        self.region_servers = {
            'US': ['1.2.3.4', '5.6.7.8'],
            'EU': ['9.10.11.12'],
            'AS': ['13.14.15.16']
        }
    
    def get_servers_for_ip(self, client_ip):
        try:
            response = self.reader.city(client_ip)
            continent = response.continent.code
            
            # Return servers for client's region, with fallback
            return self.region_servers.get(continent, self.region_servers['US'])
        except Exception:
            # Fallback to US servers
            return self.region_servers['US']
    
    def resolve_dns_query(self, client_ip, domain):
        servers = self.get_servers_for_ip(client_ip)
        # Return DNS response with appropriate servers
        return servers
```

### Anycast Routing
Use the same IP address in multiple locations, routing to the nearest one.

```
Client Request → Internet → Nearest Anycast Node
                          ├── US East (1.2.3.4)
                          ├── US West (1.2.3.4)
                          ├── Europe (1.2.3.4)
                          └── Asia (1.2.3.4)
```

**Benefits:**
- Automatic failover
- Reduced latency
- DDoS mitigation
- Simplified DNS management

---

## Implementation Examples

### HAProxy Configuration
```haproxy
global
    daemon
    maxconn 4096
    log stdout local0

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

# Frontend configuration
frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem
    
    # Redirect HTTP to HTTPS
    redirect scheme https if !{ ssl_fc }
    
    # Route based on path
    acl is_api path_beg /api/
    acl is_static path_beg /static/
    
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

# Backend configurations
backend web_servers
    balance roundrobin
    option httpchk GET /health
    
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check

backend api_servers
    balance leastconn
    option httpchk GET /api/health
    
    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check

backend static_servers
    balance roundrobin
    
    server static1 192.168.1.30:8080 check
    server static2 192.168.1.31:8080 check

# Statistics page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
```

### NGINX Load Balancer
```nginx
http {
    # Define upstream server groups
    upstream web_servers {
        least_conn;  # Use least connections algorithm
        
        server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
        server 192.168.1.11:8080 weight=2 max_fails=3 fail_timeout=30s;
        server 192.168.1.12:8080 weight=1 max_fails=3 fail_timeout=30s;
        
        # Backup server
        server 192.168.1.99:8080 backup;
    }
    
    upstream api_servers {
        hash $request_uri consistent;  # Consistent hashing
        
        server 192.168.1.20:8080;
        server 192.168.1.21:8080;
        server 192.168.1.22:8080;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    server {
        listen 80;
        server_name example.com;
        
        # Health check endpoint
        location /nginx-health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # API routes with rate limiting
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://api_servers;
            
            # Proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }
        
        # Web application routes
        location / {
            proxy_pass http://web_servers;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### AWS Application Load Balancer (Terraform)
```hcl
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets           = aws_subnet.public[*].id

  enable_deletion_protection = false

  tags = {
    Environment = "production"
  }
}

# Target Groups
resource "aws_lb_target_group" "web" {
  name     = "web-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  tags = {
    Name = "web-target-group"
  }
}

resource "aws_lb_target_group" "api" {
  name     = "api-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/api/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
}

# Listeners
resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_listener" "web_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# Listener Rules
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.web_https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

---

## Interview Questions

### Q1: How would you design a load balancer for a high-traffic web application?

**Answer Framework:**
1. **Requirements Analysis**
   - Expected traffic volume (RPS)
   - Geographic distribution
   - Availability requirements
   - Budget constraints

2. **Architecture Design**
   - Layer 7 ALB for HTTP/HTTPS traffic
   - Multiple availability zones
   - Auto-scaling target groups
   - Health checks and failover

3. **Algorithm Selection**
   - Least connections for varying request times
   - Weighted round-robin for different server capacities
   - Geographic routing for global users

4. **Implementation Considerations**
   - SSL termination at load balancer
   - Session persistence strategy
   - Monitoring and alerting
   - Security (DDoS protection, WAF)

### Q2: What's the difference between Layer 4 and Layer 7 load balancing?

**Key Points:**
- **Layer 4**: Transport layer, routes based on IP/port, higher performance
- **Layer 7**: Application layer, routes based on HTTP content, more features
- **Trade-offs**: Performance vs functionality
- **Use cases**: When to choose each approach

### Q3: How do you handle session persistence in a load-balanced environment?

**Solutions:**
1. **Sticky Sessions**: Route same client to same server
2. **Session Replication**: Sync session data across servers
3. **External Session Store**: Redis/database for session storage
4. **Stateless Design**: JWT tokens, no server-side sessions

### Q4: How would you implement health checks for your load balancer?

**Implementation:**
1. **Active Health Checks**: Periodic HTTP requests to health endpoints
2. **Passive Health Checks**: Monitor actual request failures
3. **Multi-level Checks**: Application, database, external dependencies
4. **Graceful Degradation**: Gradual traffic reduction for unhealthy servers

### Q5: Design a global load balancing solution for a worldwide application.

**Architecture:**
1. **DNS-based Load Balancing**: GeoDNS for regional routing
2. **CDN Integration**: Edge locations for static content
3. **Regional Load Balancers**: Local load balancing within regions
4. **Failover Strategy**: Cross-
