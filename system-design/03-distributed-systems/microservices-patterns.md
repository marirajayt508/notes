# Microservices Patterns

Microservices architecture is a design approach where applications are built as a collection of loosely coupled, independently deployable services. This guide covers essential patterns, best practices, and implementation strategies for building robust microservices systems.

## Table of Contents
- [What are Microservices?](#what-are-microservices)
- [Core Patterns](#core-patterns)
- [Communication Patterns](#communication-patterns)
- [Data Management Patterns](#data-management-patterns)
- [Deployment Patterns](#deployment-patterns)
- [Observability Patterns](#observability-patterns)
- [Security Patterns](#security-patterns)
- [Interview Questions & Answers](#interview-questions--answers)

## What are Microservices?

### Definition
Microservices architecture structures an application as a collection of services that are:
- **Highly maintainable and testable**
- **Loosely coupled**
- **Independently deployable**
- **Organized around business capabilities**
- **Owned by small teams**

### Key Characteristics
- **Single Responsibility**: Each service has one business responsibility
- **Decentralized**: Services manage their own data and business logic
- **Fault Isolation**: Failure in one service doesn't bring down the entire system
- **Technology Diversity**: Different services can use different technologies
- **Independent Scaling**: Services can be scaled independently

### Benefits vs Challenges

**Benefits:**
- Independent development and deployment
- Technology diversity
- Fault isolation
- Scalability
- Team autonomy

**Challenges:**
- Distributed system complexity
- Network latency and failures
- Data consistency
- Testing complexity
- Operational overhead

## Core Patterns

### 1. Service Decomposition Patterns

#### Decompose by Business Capability

```python
# Example: E-commerce system decomposed by business capabilities

class UserService:
    """Handles user management and authentication"""
    
    def __init__(self):
        self.users = {}
        self.sessions = {}
    
    def create_user(self, user_data):
        user_id = f"user_{len(self.users) + 1}"
        self.users[user_id] = {
            'id': user_id,
            'email': user_data['email'],
            'name': user_data['name'],
            'created_at': datetime.now()
        }
        return user_id
    
    def authenticate(self, email, password):
        # Simplified authentication
        for user_id, user in self.users.items():
            if user['email'] == email:
                session_id = f"session_{int(time.time())}"
                self.sessions[session_id] = user_id
                return session_id
        return None
    
    def get_user(self, user_id):
        return self.users.get(user_id)

class ProductCatalogService:
    """Manages product catalog and inventory"""
    
    def __init__(self):
        self.products = {}
        self.inventory = {}
    
    def add_product(self, product_data):
        product_id = f"product_{len(self.products) + 1}"
        self.products[product_id] = {
            'id': product_id,
            'name': product_data['name'],
            'description': product_data['description'],
            'price': product_data['price'],
            'category': product_data['category']
        }
        self.inventory[product_id] = product_data.get('quantity', 0)
        return product_id
    
    def get_product(self, product_id):
        product = self.products.get(product_id)
        if product:
            product['available_quantity'] = self.inventory.get(product_id, 0)
        return product
    
    def search_products(self, query, category=None):
        results = []
        for product in self.products.values():
            if query.lower() in product['name'].lower():
                if not category or product['category'] == category:
                    results.append(product)
        return results
    
    def reserve_inventory(self, product_id, quantity):
        available = self.inventory.get(product_id, 0)
        if available >= quantity:
            self.inventory[product_id] -= quantity
            return True
        return False

class OrderService:
    """Handles order processing and management"""
    
    def __init__(self, product_service, payment_service):
        self.orders = {}
        self.product_service = product_service
        self.payment_service = payment_service
    
    def create_order(self, user_id, items):
        order_id = f"order_{len(self.orders) + 1}"
        
        # Calculate total and validate items
        total_amount = 0
        order_items = []
        
        for item in items:
            product = self.product_service.get_product(item['product_id'])
            if not product:
                raise ValueError(f"Product {item['product_id']} not found")
            
            if not self.product_service.reserve_inventory(item['product_id'], item['quantity']):
                raise ValueError(f"Insufficient inventory for {item['product_id']}")
            
            item_total = product['price'] * item['quantity']
            total_amount += item_total
            
            order_items.append({
                'product_id': item['product_id'],
                'product_name': product['name'],
                'quantity': item['quantity'],
                'unit_price': product['price'],
                'total_price': item_total
            })
        
        order = {
            'id': order_id,
            'user_id': user_id,
            'items': order_items,
            'total_amount': total_amount,
            'status': 'pending',
            'created_at': datetime.now()
        }
        
        self.orders[order_id] = order
        return order_id
    
    def get_order(self, order_id):
        return self.orders.get(order_id)
    
    def update_order_status(self, order_id, status):
        if order_id in self.orders:
            self.orders[order_id]['status'] = status
            return True
        return False

class PaymentService:
    """Handles payment processing"""
    
    def __init__(self):
        self.payments = {}
    
    def process_payment(self, order_id, payment_method, amount):
        payment_id = f"payment_{len(self.payments) + 1}"
        
        # Simulate payment processing
        success = self._process_with_gateway(payment_method, amount)
        
        payment = {
            'id': payment_id,
            'order_id': order_id,
            'amount': amount,
            'payment_method': payment_method,
            'status': 'completed' if success else 'failed',
            'processed_at': datetime.now()
        }
        
        self.payments[payment_id] = payment
        return payment
    
    def _process_with_gateway(self, payment_method, amount):
        # Simulate external payment gateway
        time.sleep(0.1)  # Simulate network delay
        return True  # Simplified - always succeeds
```

#### Decompose by Subdomain (Domain-Driven Design)

```python
# Example: Bounded contexts in an e-commerce domain

class CustomerManagementContext:
    """Customer subdomain - handles customer lifecycle"""
    
    class Customer:
        def __init__(self, customer_id, email, name):
            self.customer_id = customer_id
            self.email = email
            self.name = name
            self.preferences = {}
            self.loyalty_points = 0
    
    def __init__(self):
        self.customers = {}
    
    def register_customer(self, email, name):
        customer_id = f"cust_{len(self.customers) + 1}"
        customer = self.Customer(customer_id, email, name)
        self.customers[customer_id] = customer
        return customer_id
    
    def update_loyalty_points(self, customer_id, points):
        if customer_id in self.customers:
            self.customers[customer_id].loyalty_points += points

class OrderManagementContext:
    """Order subdomain - handles order lifecycle"""
    
    class Order:
        def __init__(self, order_id, customer_id):
            self.order_id = order_id
            self.customer_id = customer_id
            self.line_items = []
            self.status = 'draft'
            self.total = 0
    
    class LineItem:
        def __init__(self, product_id, quantity, price):
            self.product_id = product_id
            self.quantity = quantity
            self.price = price
    
    def __init__(self):
        self.orders = {}
    
    def create_order(self, customer_id):
        order_id = f"ord_{len(self.orders) + 1}"
        order = self.Order(order_id, customer_id)
        self.orders[order_id] = order
        return order_id
    
    def add_line_item(self, order_id, product_id, quantity, price):
        if order_id in self.orders:
            line_item = self.LineItem(product_id, quantity, price)
            self.orders[order_id].line_items.append(line_item)
            self.orders[order_id].total += quantity * price

class InventoryContext:
    """Inventory subdomain - handles stock management"""
    
    class InventoryItem:
        def __init__(self, product_id, quantity, reserved=0):
            self.product_id = product_id
            self.quantity = quantity
            self.reserved = reserved
            self.available = quantity - reserved
    
    def __init__(self):
        self.inventory = {}
    
    def add_stock(self, product_id, quantity):
        if product_id in self.inventory:
            self.inventory[product_id].quantity += quantity
            self.inventory[product_id].available += quantity
        else:
            self.inventory[product_id] = self.InventoryItem(product_id, quantity)
    
    def reserve_stock(self, product_id, quantity):
        if product_id in self.inventory:
            item = self.inventory[product_id]
            if item.available >= quantity:
                item.reserved += quantity
                item.available -= quantity
                return True
        return False
```

### 2. API Gateway Pattern

```python
import json
import time
from typing import Dict, Any, Optional
from abc import ABC, abstractmethod

class APIGateway:
    """Central entry point for all client requests"""
    
    def __init__(self):
        self.services = {}
        self.rate_limiter = RateLimiter()
        self.auth_service = AuthenticationService()
        self.circuit_breakers = {}
        self.request_router = RequestRouter()
    
    def register_service(self, service_name: str, service_instance):
        """Register a microservice with the gateway"""
        self.services[service_name] = service_instance
        self.circuit_breakers[service_name] = CircuitBreaker(service_name)
    
    def handle_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Main request handling pipeline"""
        try:
            # 1. Authentication
            if not self._authenticate_request(request):
                return {'error': 'Unauthorized', 'status': 401}
            
            # 2. Rate limiting
            if not self.rate_limiter.allow_request(request.get('client_id')):
                return {'error': 'Rate limit exceeded', 'status': 429}
            
            # 3. Request routing
            service_name, method_name = self.request_router.route(request['path'])
            
            if service_name not in self.services:
                return {'error': 'Service not found', 'status': 404}
            
            # 4. Circuit breaker check
            circuit_breaker = self.circuit_breakers[service_name]
            if not circuit_breaker.can_execute():
                return {'error': 'Service unavailable', 'status': 503}
            
            # 5. Execute request
            service = self.services[service_name]
            method = getattr(service, method_name, None)
            
            if not method:
                return {'error': 'Method not found', 'status': 404}
            
            start_time = time.time()
            try:
                result = method(**request.get('params', {}))
                circuit_breaker.record_success()
                
                # 6. Response transformation
                response = self._transform_response(result)
                response['execution_time'] = time.time() - start_time
                
                return response
                
            except Exception as e:
                circuit_breaker.record_failure()
                return {'error': str(e), 'status': 500}
        
        except Exception as e:
            return {'error': f'Gateway error: {str(e)}', 'status': 500}
    
    def _authenticate_request(self, request: Dict[str, Any]) -> bool:
        """Authenticate incoming request"""
        token = request.get('headers', {}).get('Authorization')
        if not token:
            return False
        
        return self.auth_service.validate_token(token)
    
    def _transform_response(self, result: Any) -> Dict[str, Any]:
        """Transform service response for client"""
        return {
            'data': result,
            'status': 200,
            'timestamp': time.time()
        }

class RateLimiter:
    """Token bucket rate limiter"""
    
    def __init__(self, requests_per_minute=60):
        self.requests_per_minute = requests_per_minute
        self.buckets = {}  # client_id -> bucket
    
    def allow_request(self, client_id: str) -> bool:
        now = time.time()
        
        if client_id not in self.buckets:
            self.buckets[client_id] = {
                'tokens': self.requests_per_minute,
                'last_refill': now
            }
        
        bucket = self.buckets[client_id]
        
        # Refill tokens
        time_passed = now - bucket['last_refill']
        tokens_to_add = time_passed * (self.requests_per_minute / 60)
        bucket['tokens'] = min(self.requests_per_minute, bucket['tokens'] + tokens_to_add)
        bucket['last_refill'] = now
        
        # Check if request is allowed
        if bucket['tokens'] >= 1:
            bucket['tokens'] -= 1
            return True
        
        return False

class CircuitBreaker:
    """Circuit breaker for service calls"""
    
    def __init__(self, service_name: str, failure_threshold=5, timeout=60):
        self.service_name = service_name
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def can_execute(self) -> bool:
        if self.state == 'CLOSED':
            return True
        elif self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
                return True
            return False
        else:  # HALF_OPEN
            return True
    
    def record_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'

class RequestRouter:
    """Routes requests to appropriate services"""
    
    def __init__(self):
        self.routes = {
            '/users': ('user_service', 'handle_user_request'),
            '/products': ('product_service', 'handle_product_request'),
            '/orders': ('order_service', 'handle_order_request'),
            '/payments': ('payment_service', 'handle_payment_request')
        }
    
    def route(self, path: str) -> tuple:
        """Return (service_name, method_name) for given path"""
        for route_path, (service, method) in self.routes.items():
            if path.startswith(route_path):
                return service, method
        
        raise ValueError(f"No route found for path: {path}")

class AuthenticationService:
    """Handles authentication and token validation"""
    
    def __init__(self):
        self.valid_tokens = {
            'token123': {'user_id': 'user1', 'role': 'customer'},
            'token456': {'user_id': 'user2', 'role': 'admin'}
        }
    
    def validate_token(self, token: str) -> bool:
        return token in self.valid_tokens
    
    def get_user_info(self, token: str) -> Optional[Dict[str, Any]]:
        return self.valid_tokens.get(token)
```

### 3. Service Registry and Discovery

```python
import threading
import time
from typing import Dict, List, Optional
from dataclasses import dataclass
from enum import Enum

class ServiceStatus(Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"
    UNKNOWN = "unknown"

@dataclass
class ServiceInstance:
    service_name: str
    instance_id: str
    host: str
    port: int
    status: ServiceStatus
    metadata: Dict[str, str]
    last_heartbeat: float
    
    def is_healthy(self, heartbeat_timeout: int = 30) -> bool:
        return (time.time() - self.last_heartbeat) < heartbeat_timeout

class ServiceRegistry:
    """Central registry for service discovery"""
    
    def __init__(self, heartbeat_timeout=30):
        self.services = {}  # service_name -> list of instances
        self.heartbeat_timeout = heartbeat_timeout
        self.lock = threading.Lock()
        
        # Start health check thread
        self.health_checker = threading.Thread(target=self._health_check_loop)
        self.health_checker.daemon = True
        self.health_checker.start()
    
    def register_service(self, service_instance: ServiceInstance):
        """Register a new service instance"""
        with self.lock:
            if service_instance.service_name not in self.services:
                self.services[service_instance.service_name] = []
            
            # Remove existing instance with same ID
            self.services[service_instance.service_name] = [
                inst for inst in self.services[service_instance.service_name]
                if inst.instance_id != service_instance.instance_id
            ]
            
            # Add new instance
            service_instance.last_heartbeat = time.time()
            self.services[service_instance.service_name].append(service_instance)
            
            print(f"Registered service: {service_instance.service_name}:{service_instance.instance_id}")
    
    def deregister_service(self, service_name: str, instance_id: str):
        """Deregister a service instance"""
        with self.lock:
            if service_name in self.services:
                self.services[service_name] = [
                    inst for inst in self.services[service_name]
                    if inst.instance_id != instance_id
                ]
                print(f"Deregistered service: {service_name}:{instance_id}")
    
    def discover_service(self, service_name: str) -> List[ServiceInstance]:
        """Discover healthy instances of a service"""
        with self.lock:
            instances = self.services.get(service_name, [])
            return [inst for inst in instances if inst.is_healthy(self.heartbeat_timeout)]
    
    def heartbeat(self, service_name: str, instance_id: str):
        """Update heartbeat for a service instance"""
        with self.lock:
            instances = self.services.get(service_name, [])
            for instance in instances:
                if instance.instance_id == instance_id:
                    instance.last_heartbeat = time.time()
                    instance.status = ServiceStatus.HEALTHY
                    break
    
    def _health_check_loop(self):
        """Background thread to check service health"""
        while True:
            time.sleep(10)  # Check every 10 seconds
            
            with self.lock:
                for service_name, instances in self.services.items():
                    for instance in instances:
                        if not instance.is_healthy(self.heartbeat_timeout):
                            instance.status = ServiceStatus.UNHEALTHY
                            print(f"Service unhealthy: {service_name}:{instance.instance_id}")

class LoadBalancer:
    """Load balancer with multiple strategies"""
    
    def __init__(self, service_registry: ServiceRegistry):
        self.service_registry = service_registry
        self.round_robin_counters = {}
    
    def get_service_instance(self, service_name: str, strategy: str = "round_robin") -> Optional[ServiceInstance]:
        """Get a service instance using specified load balancing strategy"""
        instances = self.service_registry.discover_service(service_name)
        
        if not instances:
            return None
        
        if strategy == "round_robin":
            return self._round_robin(service_name, instances)
        elif strategy == "random":
            return self._random(instances)
        elif strategy == "least_connections":
            return self._least_connections(instances)
        else:
            return instances[0]  # Default to first instance
    
    def _round_robin(self, service_name: str, instances: List[ServiceInstance]) -> ServiceInstance:
        """Round-robin load balancing"""
        if service_name not in self.round_robin_counters:
            self.round_robin_counters[service_name] = 0
        
        index = self.round_robin_counters[service_name] % len(instances)
        self.round_robin_counters[service_name] += 1
        
        return instances[index]
    
    def _random(self, instances: List[ServiceInstance]) -> ServiceInstance:
        """Random load balancing"""
        import random
        return random.choice(instances)
    
    def _least_connections(self, instances: List[ServiceInstance]) -> ServiceInstance:
        """Least connections load balancing (simplified)"""
        # In a real implementation, you'd track active connections
        return min(instances, key=lambda x: int(x.metadata.get('active_connections', '0')))

class ServiceClient:
    """Client for making requests to services through service discovery"""
    
    def __init__(self, service_registry: ServiceRegistry, load_balancer: LoadBalancer):
        self.service_registry = service_registry
        self.load_balancer = load_balancer
    
    def call_service(self, service_name: str, method: str, **kwargs):
        """Make a call to a service"""
        instance = self.load_balancer.get_service_instance(service_name)
        
        if not instance:
            raise Exception(f"No healthy instances found for service: {service_name}")
        
        try:
            # Simulate service call
            result = self._make_http_request(instance, method, **kwargs)
            return result
        
        except Exception as e:
            # Mark instance as unhealthy and retry with another instance
            instance.status = ServiceStatus.UNHEALTHY
            
            # Try another instance
            retry_instance = self.load_balancer.get_service_instance(service_name)
            if retry_instance and retry_instance.instance_id != instance.instance_id:
                return self._make_http_request(retry_instance, method, **kwargs)
            
            raise e
    
    def _make_http_request(self, instance: ServiceInstance, method: str, **kwargs):
        """Simulate HTTP request to service instance"""
        # In real implementation, this would make actual HTTP request
        print(f"Calling {instance.service_name}:{instance.instance_id} -> {method}")
        time.sleep(0.1)  # Simulate network delay
        return {"status": "success", "data": f"Response from {instance.instance_id}"}

# Usage example
def setup_service_discovery():
    # Create service registry
    registry = ServiceRegistry()
    
    # Register service instances
    user_service_1 = ServiceInstance(
        service_name="user-service",
        instance_id="user-1",
        host="localhost",
        port=8001,
        status=ServiceStatus.HEALTHY,
        metadata={"version": "1.0", "active_connections": "5"}
    )
    
    user_service_2 = ServiceInstance(
        service_name="user-service",
        instance_id="user-2",
        host="localhost",
        port=8002,
        status=ServiceStatus.HEALTHY,
        metadata={"version": "1.0", "active_connections": "3"}
    )
    
    registry.register_service(user_service_1)
    registry.register_service(user_service_2)
    
    # Create load balancer and client
    load_balancer = LoadBalancer(registry)
    client = ServiceClient(registry, load_balancer)
    
    # Make service calls
    for i in range(5):
        try:
            result = client.call_service("user-service", "get_user", user_id=f"user_{i}")
            print(f"Call {i}: {result}")
        except Exception as e:
            print(f"Call {i} failed: {e}")
        
        time.sleep(1)
    
    return registry, load_balancer, client
```

## Communication Patterns

### 1. Synchronous Communication

```python
import requests
import time
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class ServiceResponse:
    status_code: int
    data: Any
    error: Optional[str] = None
    execution_time: float = 0

class HTTPServiceClient:
    """HTTP client for synchronous service communication"""
    
    def __init__(self, base_url: str, timeout: int = 30):
        self.base_url = base_url.rstrip('/')
        self.timeout = timeout
        self.session = requests.Session()
    
    def get(self, endpoint: str, params: Dict[str, Any] = None) -> ServiceResponse:
        """Make GET request to service"""
        return self._make_request('GET', endpoint, params=params)
    
    def post(self, endpoint: str, data: Dict[str, Any] = None) -> ServiceResponse:
        """Make POST request to service"""
        return self._make_request('POST', endpoint, json=data)
    
    def put(self, endpoint: str, data: Dict[str, Any] = None) -> ServiceResponse:
        """Make PUT request to service"""
        return self._make_request('PUT', endpoint, json=data)
    
    def delete(self, endpoint: str) -> ServiceResponse:
        """Make DELETE request to service"""
        return self._make_request('DELETE', endpoint)
    
    def _make_request(self, method: str, endpoint: str, **kwargs) -> ServiceResponse:
        """Make HTTP request with error handling and timing"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        start_time = time.time()
        
        try:
            response = self.session.request(
                method=method,
                url=url,
                timeout=self.timeout,
                **kwargs
            )
            
            execution_time = time.time() - start_time
            
            if response.status_code < 400:
                return ServiceResponse(
                    status_code=response.status_code,
                    data=response.json() if response.content else None,
                    execution_time=execution_time
                )
            else:
                return ServiceResponse(
                    status_code=response.status_code,
                    data=None,
                    error=response.text,
                    execution_time=execution_time
                )
        
        except requests.exceptions.Timeout:
            return ServiceResponse(
                status_code=408,
                data=None,
                error="Request timeout",
                execution_time=time.time() - start_time
            )
        
        except requests.exceptions.ConnectionError:
            return ServiceResponse(
                status_code=503,
                data=None,
                error="Service unavailable",
                execution_time=time.time() - start_time
            )
        
        except Exception as e:
            return ServiceResponse(
                status_code=500,
                data=None,
                error=str(e),
                execution_time=time.time() - start_time
            )

class ServiceOrchestrator:
    """Orchestrates calls to multiple services"""
    
    def __init__(self):
        self.clients = {
            'user': HTTPServiceClient('http://user-service:8001'),
            'product': HTTPServiceClient('http://product-service:8002'),
            'order': HTTPServiceClient('http://order-service:8003'),
            'payment': HTTPServiceClient('http://payment-service:8004')
        }
    
    def process_order(self, user_id: str, items: list, payment_info: dict) -> Dict[str, Any]:
        """Process order by orchestrating multiple service calls"""
        
        # Step 1: Validate user
        user_response = self.clients['user'].get(f'/users/{user_id}')
        if user_response.status_code != 200:
            return {'error': 'Invalid user', 'step': 'user_validation'}
        
        user = user_response.data
        
        # Step 2: Validate products and calculate total
        total_amount = 0
        validated_items = []
        
        for item in items:
            product_response = self.clients['product'].get(f'/products/{item["product_id"]}')
            if product_response.status_code != 200:
                return {'error': f'Invalid product: {item["product_id"]}', 'step': 'product_validation'}
            
            product = product_response.data
            item_total = product['price'] * item['quantity']
            total_amount += item_total
            
            validated_items.append({
                'product_id': item['product_id'],
                'product_name': product['name'],
                'quantity': item['quantity'],
                'unit_price': product['price'],
                'total_price': item_total
            })
        
        # Step 3: Create order
        order_data = {
            'user_id': user_id,
            'items': validated_items,
            'total_amount': total_amount
        }
        
        order_response = self.clients['order'].post('/orders', order_data)
        if order_response.status_code != 201:
            return {'error': 'Failed to create order', 'step': 'order_creation'}
        
        order = order_response.data
        
        # Step 4: Process payment
        payment_data = {
            'order_id': order['id'],
            'amount': total_amount,
            'payment_method': payment_info['method'],
            'card_token': payment_info.get('card_token')
        }
        
        payment_response = self.clients['payment'].post('/payments', payment_data)
        if payment_response.status_code != 200:
            # Compensate: Cancel order
            self.clients['order'].delete(f'/orders/{order["id"]}')
            return {'error': 'Payment failed', 'step': 'payment_processing'}
        
        # Step 5: Update order status
        self.clients['order'].put(f'/orders/{order["id"]}', {'status': 'confirmed'})
        
        return {
            'success': True,
            'order_id': order['id'],
            'total_amount': total_amount,
            'payment_id': payment_response.data['id']
        }
```

### 2. Asynchronous Communication

```python
import asyncio
import json
from typing import Dict, Any, Callable
