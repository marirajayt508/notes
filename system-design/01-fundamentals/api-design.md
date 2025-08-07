# API Design - System Design Fundamentals

## Table of Contents
1. [API Design Principles](#api-design-principles)
2. [RESTful API Design](#restful-api-design)
3. [GraphQL vs REST](#graphql-vs-rest)
4. [API Versioning](#api-versioning)
5. [Authentication & Authorization](#authentication--authorization)
6. [Rate Limiting](#rate-limiting)
7. [Error Handling](#error-handling)
8. [API Documentation](#api-documentation)
9. [Performance Optimization](#performance-optimization)
10. [Interview Questions](#interview-questions)

---

## API Design Principles

### 1. Consistency
**Maintain consistent patterns across all endpoints**

```http
# Good: Consistent naming and structure
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{id}
PUT    /api/v1/users/{id}
DELETE /api/v1/users/{id}

GET    /api/v1/posts
POST   /api/v1/posts
GET    /api/v1/posts/{id}
PUT    /api/v1/posts/{id}
DELETE /api/v1/posts/{id}

# Bad: Inconsistent patterns
GET    /api/v1/users
POST   /api/v1/create-user
GET    /api/v1/user/{id}
PUT    /api/v1/update_user/{id}
DELETE /api/v1/remove-user/{id}
```

### 2. Intuitive Resource Naming
**Use nouns for resources, verbs for actions**

```http
# Good: Resource-based URLs
GET    /api/v1/users/{id}/posts           # Get user's posts
POST   /api/v1/users/{id}/posts           # Create post for user
GET    /api/v1/posts/{id}/comments        # Get post comments
POST   /api/v1/posts/{id}/like            # Like a post

# Bad: Action-based URLs
GET    /api/v1/getUserPosts?userId={id}
POST   /api/v1/createPostForUser
GET    /api/v1/getCommentsForPost?postId={id}
POST   /api/v1/likePost?postId={id}
```

### 3. Stateless Design
**Each request should contain all necessary information**

```python
# Good: Stateless request with all context
@app.route('/api/v1/users/<user_id>/posts', methods=['GET'])
@require_auth
def get_user_posts(user_id):
    # All context provided in request
    auth_token = request.headers.get('Authorization')
    user = authenticate_user(auth_token)
    
    if not can_access_user_posts(user, user_id):
        return {'error': 'Forbidden'}, 403
    
    posts = get_posts_for_user(user_id)
    return {'posts': posts}

# Bad: Stateful design relying on server session
@app.route('/api/v1/posts', methods=['GET'])
def get_posts():
    # Relies on server-side session state
    if 'user_id' not in session:
        return {'error': 'Not authenticated'}, 401
    
    user_id = session['user_id']
    posts = get_posts_for_user(user_id)
    return {'posts': posts}
```

### 4. Idempotency
**Same request should produce same result**

```python
# Idempotent operations
@app.route('/api/v1/users/<user_id>', methods=['PUT'])
def update_user(user_id):
    """PUT is idempotent - same request produces same result"""
    user_data = request.json
    user = update_user_by_id(user_id, user_data)
    return {'user': user}

@app.route('/api/v1/users/<user_id>', methods=['DELETE'])
def delete_user(user_id):
    """DELETE is idempotent - deleting same resource multiple times"""
    delete_user_by_id(user_id)  # Should handle already-deleted case
    return {'message': 'User deleted'}, 200

# Non-idempotent operation (by design)
@app.route('/api/v1/posts', methods=['POST'])
def create_post():
    """POST creates new resource each time"""
    post_data = request.json
    post = create_new_post(post_data)
    return {'post': post}, 201
```

---

## RESTful API Design

### HTTP Methods and Their Usage

#### GET - Retrieve Resources
```http
# Get all users (with pagination)
GET /api/v1/users?page=1&limit=20&sort=created_at

# Get specific user
GET /api/v1/users/123

# Get user's posts
GET /api/v1/users/123/posts

# Search users
GET /api/v1/users?search=john&role=admin
```

```python
@app.route('/api/v1/users', methods=['GET'])
def get_users():
    # Parse query parameters
    page = int(request.args.get('page', 1))
    limit = min(int(request.args.get('limit', 20)), 100)  # Max 100
    sort = request.args.get('sort', 'created_at')
    search = request.args.get('search')
    role = request.args.get('role')
    
    # Build query
    query = User.query
    if search:
        query = query.filter(User.name.contains(search))
    if role:
        query = query.filter(User.role == role)
    
    # Apply pagination and sorting
    users = query.order_by(getattr(User, sort)).paginate(
        page=page, per_page=limit, error_out=False
    )
    
    return {
        'users': [user.to_dict() for user in users.items],
        'pagination': {
            'page': page,
            'pages': users.pages,
            'per_page': limit,
            'total': users.total,
            'has_next': users.has_next,
            'has_prev': users.has_prev
        }
    }
```

#### POST - Create Resources
```http
POST /api/v1/users
Content-Type: application/json

{
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
}
```

```python
@app.route('/api/v1/users', methods=['POST'])
def create_user():
    # Validate request
    if not request.is_json:
        return {'error': 'Content-Type must be application/json'}, 400
    
    data = request.json
    
    # Validate required fields
    required_fields = ['name', 'email']
    for field in required_fields:
        if field not in data:
            return {'error': f'Missing required field: {field}'}, 400
    
    # Validate email format
    if not is_valid_email(data['email']):
        return {'error': 'Invalid email format'}, 400
    
    # Check if user already exists
    if User.query.filter_by(email=data['email']).first():
        return {'error': 'User with this email already exists'}, 409
    
    # Create user
    user = User(
        name=data['name'],
        email=data['email'],
        role=data.get('role', 'user')
    )
    
    db.session.add(user)
    db.session.commit()
    
    return {'user': user.to_dict()}, 201
```

#### PUT - Update/Replace Resources
```http
PUT /api/v1/users/123
Content-Type: application/json

{
    "name": "John Smith",
    "email": "john.smith@example.com",
    "role": "admin"
}
```

```python
@app.route('/api/v1/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    user = User.query.get_or_404(user_id)
    data = request.json
    
    # Update all provided fields
    if 'name' in data:
        user.name = data['name']
    if 'email' in data:
        if not is_valid_email(data['email']):
            return {'error': 'Invalid email format'}, 400
        user.email = data['email']
    if 'role' in data:
        user.role = data['role']
    
    user.updated_at = datetime.utcnow()
    db.session.commit()
    
    return {'user': user.to_dict()}
```

#### PATCH - Partial Updates
```http
PATCH /api/v1/users/123
Content-Type: application/json

{
    "name": "John Smith"
}
```

```python
@app.route('/api/v1/users/<int:user_id>', methods=['PATCH'])
def patch_user(user_id):
    user = User.query.get_or_404(user_id)
    data = request.json
    
    # Only update provided fields
    for field in ['name', 'email', 'role']:
        if field in data:
            if field == 'email' and not is_valid_email(data[field]):
                return {'error': 'Invalid email format'}, 400
            setattr(user, field, data[field])
    
    user.updated_at = datetime.utcnow()
    db.session.commit()
    
    return {'user': user.to_dict()}
```

#### DELETE - Remove Resources
```http
DELETE /api/v1/users/123
```

```python
@app.route('/api/v1/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    user = User.query.get_or_404(user_id)
    
    # Soft delete vs hard delete
    if SOFT_DELETE_ENABLED:
        user.deleted_at = datetime.utcnow()
        user.is_active = False
    else:
        db.session.delete(user)
    
    db.session.commit()
    
    return {'message': 'User deleted successfully'}, 200
```

### HTTP Status Codes

#### Success Codes (2xx)
```python
# 200 OK - Successful GET, PUT, PATCH, DELETE
return {'data': result}, 200

# 201 Created - Successful POST
return {'user': new_user}, 201

# 202 Accepted - Request accepted for processing
return {'message': 'Request accepted'}, 202

# 204 No Content - Successful DELETE with no response body
return '', 204
```

#### Client Error Codes (4xx)
```python
# 400 Bad Request - Invalid request format
return {'error': 'Invalid JSON format'}, 400

# 401 Unauthorized - Authentication required
return {'error': 'Authentication required'}, 401

# 403 Forbidden - Insufficient permissions
return {'error': 'Insufficient permissions'}, 403

# 404 Not Found - Resource not found
return {'error': 'User not found'}, 404

# 409 Conflict - Resource already exists
return {'error': 'Email already exists'}, 409

# 422 Unprocessable Entity - Validation errors
return {
    'error': 'Validation failed',
    'details': {
        'email': 'Invalid email format',
        'age': 'Must be at least 18'
    }
}, 422

# 429 Too Many Requests - Rate limit exceeded
return {'error': 'Rate limit exceeded'}, 429
```

#### Server Error Codes (5xx)
```python
# 500 Internal Server Error - Unexpected server error
return {'error': 'Internal server error'}, 500

# 502 Bad Gateway - Upstream server error
return {'error': 'Service temporarily unavailable'}, 502

# 503 Service Unavailable - Server overloaded
return {'error': 'Service unavailable'}, 503
```

### Resource Relationships

#### Nested Resources
```http
# User's posts
GET    /api/v1/users/123/posts
POST   /api/v1/users/123/posts
GET    /api/v1/users/123/posts/456
PUT    /api/v1/users/123/posts/456
DELETE /api/v1/users/123/posts/456

# Post's comments
GET    /api/v1/posts/456/comments
POST   /api/v1/posts/456/comments
```

#### Resource Linking
```json
{
    "user": {
        "id": 123,
        "name": "John Doe",
        "email": "john@example.com",
        "posts": [
            {
                "id": 456,
                "title": "My First Post",
                "url": "/api/v1/posts/456"
            }
        ],
        "links": {
            "self": "/api/v1/users/123",
            "posts": "/api/v1/users/123/posts",
            "followers": "/api/v1/users/123/followers"
        }
    }
}
```

---

## GraphQL vs REST

### REST API Example
```http
# Multiple requests needed
GET /api/v1/users/123
GET /api/v1/users/123/posts
GET /api/v1/posts/456/comments
```

```json
// Response from /api/v1/users/123
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "created_at": "2023-01-01T00:00:00Z"
}

// Response from /api/v1/users/123/posts
{
    "posts": [
        {
            "id": 456,
            "title": "My First Post",
            "content": "Hello World!",
            "created_at": "2023-01-02T00:00:00Z"
        }
    ]
}
```

### GraphQL Example
```graphql
# Single request with exact data needed
query {
    user(id: 123) {
        name
        email
        posts {
            title
            comments {
                content
                author {
                    name
                }
            }
        }
    }
}
```

```json
{
    "data": {
        "user": {
            "name": "John Doe",
            "email": "john@example.com",
            "posts": [
                {
                    "title": "My First Post",
                    "comments": [
                        {
                            "content": "Great post!",
                            "author": {
                                "name": "Jane Smith"
                            }
                        }
                    ]
                }
            ]
        }
    }
}
```

### GraphQL Implementation
```python
import graphene
from graphene_sqlalchemy import SQLAlchemyObjectType

class UserType(SQLAlchemyObjectType):
    class Meta:
        model = User

class PostType(SQLAlchemyObjectType):
    class Meta:
        model = Post

class Query(graphene.ObjectType):
    user = graphene.Field(UserType, id=graphene.Int(required=True))
    users = graphene.List(UserType, limit=graphene.Int(), offset=graphene.Int())
    
    def resolve_user(self, info, id):
        return User.query.get(id)
    
    def resolve_users(self, info, limit=20, offset=0):
        return User.query.offset(offset).limit(limit).all()

class CreateUser(graphene.Mutation):
    class Arguments:
        name = graphene.String(required=True)
        email = graphene.String(required=True)
    
    user = graphene.Field(UserType)
    
    def mutate(self, info, name, email):
        user = User(name=name, email=email)
        db.session.add(user)
        db.session.commit()
        return CreateUser(user=user)

class Mutation(graphene.ObjectType):
    create_user = CreateUser.Field()

schema = graphene.Schema(query=Query, mutation=Mutation)
```

### When to Use GraphQL vs REST

#### Use GraphQL When:
- **Complex data relationships** - Nested resources with varying requirements
- **Mobile applications** - Minimize network requests and data transfer
- **Rapid frontend development** - Frontend teams need flexibility
- **Multiple client types** - Different clients need different data

#### Use REST When:
- **Simple CRUD operations** - Straightforward resource manipulation
- **Caching requirements** - HTTP caching is important
- **File uploads** - Binary data handling
- **Team familiarity** - Team is more comfortable with REST

---

## API Versioning

### URL Versioning
```http
# Version in URL path
GET /api/v1/users
GET /api/v2/users

# Version in subdomain
GET https://v1.api.example.com/users
GET https://v2.api.example.com/users
```

```python
from flask import Blueprint

# Version 1
v1_bp = Blueprint('v1', __name__, url_prefix='/api/v1')

@v1_bp.route('/users', methods=['GET'])
def get_users_v1():
    # Version 1 implementation
    users = User.query.all()
    return {'users': [{'id': u.id, 'name': u.name} for u in users]}

# Version 2
v2_bp = Blueprint('v2', __name__, url_prefix='/api/v2')

@v2_bp.route('/users', methods=['GET'])
def get_users_v2():
    # Version 2 implementation with additional fields
    users = User.query.all()
    return {
        'users': [
            {
                'id': u.id,
                'name': u.name,
                'email': u.email,
                'created_at': u.created_at.isoformat()
            } for u in users
        ]
    }

app.register_blueprint(v1_bp)
app.register_blueprint(v2_bp)
```

### Header Versioning
```http
GET /api/users
Accept: application/vnd.api+json;version=1

GET /api/users
Accept: application/vnd.api+json;version=2
```

```python
@app.route('/api/users', methods=['GET'])
def get_users():
    accept_header = request.headers.get('Accept', '')
    
    if 'version=2' in accept_header:
        return get_users_v2()
    else:
        return get_users_v1()  # Default to v1
```

### Query Parameter Versioning
```http
GET /api/users?version=1
GET /api/users?version=2
```

### Versioning Best Practices
```python
class APIVersionManager:
    def __init__(self):
        self.versions = {
            'v1': {'deprecated': True, 'sunset_date': '2024-12-31'},
            'v2': {'deprecated': False, 'current': True},
            'v3': {'deprecated': False, 'beta': True}
        }
    
    def get_version_info(self, version):
        return self.versions.get(version, {})
    
    def add_version_headers(self, response, version):
        version_info = self.get_version_info(version)
        
        if version_info.get('deprecated'):
            response.headers['Deprecation'] = 'true'
            response.headers['Sunset'] = version_info.get('sunset_date')
        
        response.headers['API-Version'] = version
        return response

# Usage
version_manager = APIVersionManager()

@app.after_request
def add_version_headers(response):
    version = get_api_version_from_request()
    return version_manager.add_version_headers(response, version)
```

---

## Authentication & Authorization

### JWT Authentication
```python
import jwt
from datetime import datetime, timedelta
from functools import wraps

class JWTAuth:
    def __init__(self, secret_key, algorithm='HS256'):
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    def generate_token(self, user_id, expires_in=3600):
        payload = {
            'user_id': user_id,
            'exp': datetime.utcnow() + timedelta(seconds=expires_in),
            'iat': datetime.utcnow()
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token):
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            raise AuthenticationError('Token has expired')
        except jwt.InvalidTokenError:
            raise AuthenticationError('Invalid token')

# Authentication decorator
def require_auth(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        auth_header = request.headers.get('Authorization')
        
        if not auth_header:
            return {'error': 'Authorization header required'}, 401
        
        try:
            token = auth_header.split(' ')[1]  # Bearer <token>
            payload = jwt_auth.verify_token(token)
            request.current_user_id = payload['user_id']
        except (IndexError, AuthenticationError) as e:
            return {'error': str(e)}, 401
        
        return f(*args, **kwargs)
    return decorated_function

# Usage
@app.route('/api/v1/profile', methods=['GET'])
@require_auth
def get_profile():
    user = User.query.get(request.current_user_id)
    return {'user': user.to_dict()}
```

### OAuth 2.0 Implementation
```python
from authlib.integrations.flask_oauth2 import ResourceProtector
from authlib.oauth2.rfc6750 import BearerTokenValidator

class MyBearerTokenValidator(BearerTokenValidator):
    def authenticate_token(self, token_string):
        # Validate token and return token object
        token = Token.query.filter_by(access_token=token_string).first()
        if token and not token.is_expired():
            return token
        return None
    
    def request_invalid(self, request):
        return False
    
    def token_revoked(self, token):
        return token.revoked

require_oauth = ResourceProtector()
require_oauth.register_token_validator(MyBearerTokenValidator())

@app.route('/api/v1/protected', methods=['GET'])
@require_oauth('read')
def protected_resource():
    token = request.oauth_token
    user = token.user
    return {'message': f'Hello {user.name}'}
```

### Role-Based Access Control (RBAC)
```python
from enum import Enum

class Role(Enum):
    USER = 'user'
    ADMIN = 'admin'
    MODERATOR = 'moderator'

class Permission(Enum):
    READ_USERS = 'read:users'
    WRITE_USERS = 'write:users'
    DELETE_USERS = 'delete:users'
    MODERATE_CONTENT = 'moderate:content'

ROLE_PERMISSIONS = {
    Role.USER: [Permission.READ_USERS],
    Role.MODERATOR: [Permission.READ_USERS, Permission.MODERATE_CONTENT],
    Role.ADMIN: [Permission.READ_USERS, Permission.WRITE_USERS, Permission.DELETE_USERS]
}

def require_permission(permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            user = get_current_user()
            user_permissions = ROLE_PERMISSIONS.get(user.role, [])
            
            if permission not in user_permissions:
                return {'error': 'Insufficient permissions'}, 403
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route('/api/v1/users', methods=['DELETE'])
@require_auth
@require_permission(Permission.DELETE_USERS)
def delete_user():
    # Only admins can delete users
    pass
```

---

## Rate Limiting

### Token Bucket Algorithm
```python
import time
from collections import defaultdict

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.time()
    
    def consume(self, tokens=1):
        now = time.time()
        
        # Add tokens based on time passed
        tokens_to_add = (now - self.last_refill) * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
        
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False

class RateLimiter:
    def __init__(self, capacity=100, refill_rate=10):
        self.buckets = defaultdict(lambda: TokenBucket(capacity, refill_rate))
    
    def is_allowed(self, key, tokens=1):
        return self.buckets[key].consume(tokens)

# Flask middleware
rate_limiter = RateLimiter(capacity=100, refill_rate=10)  # 100 requests, 10/second refill

@app.before_request
def check_rate_limit():
    # Use IP address as key (could also use user ID)
    client_ip = request.remote_addr
    
    if not rate_limiter.is_allowed(client_ip):
        return {'error': 'Rate limit exceeded'}, 429

# Per-endpoint rate limiting
def rate_limit(requests_per_minute=60):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            key = f"{request.remote_addr}:{request.endpoint}"
            limiter = RateLimiter(capacity=requests_per_minute, refill_rate=requests_per_minute/60)
            
            if not limiter.is_allowed(key):
                return {'error': 'Rate limit exceeded'}, 429
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/api/v1/upload', methods=['POST'])
@rate_limit(requests_per_minute=10)  # Stricter limit for uploads
def upload_file():
    pass
```

### Sliding Window Rate Limiting
```python
import time
from collections import deque

class SlidingWindowRateLimiter:
    def __init__(self, max_requests, window_size):
        self.max_requests = max_requests
        self.window_size = window_size
        self.requests = defaultdict(deque)
    
    def is_allowed(self, key):
        now = time.time()
        window_start = now - self.window_size
        
        # Remove old requests outside the window
        while self.requests[key] and self.requests[key][0] < window_start:
            self.requests[key].popleft()
        
        # Check if we can allow this request
        if len(self.requests[key]) < self.max_requests:
            self.requests[key].append(now)
            return True
        
        return False

# Usage
sliding_limiter = SlidingWindowRateLimiter(max_requests=100, window_size=3600)  # 100 requests per hour
```

### Redis-Based Distributed Rate Limiting
```python
import redis
import time

class RedisRateLimiter:
    def __init__(self, redis_client, max_requests, window_size):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_size = window_size
    
    def is_allowed(self, key):
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - self.window_size
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count current requests
        pipe.zcard(key)
        
        # Add current request
        pipe.zadd(key, {str(now): now})
        
        # Set expiration
        pipe.expire(key, int(self.window_size) + 1)
        
        results = pipe.execute()
        current_requests = results[1]
        
        return current_requests < self.max_requests

# Usage
redis_client = redis.Redis(host='localhost', port=6379, db=0)
redis_limiter = RedisRateLimiter(redis_client, max_requests=1000, window_size=3600)
```

---

## Error Handling

### Standardized Error Response Format
```python
class APIError(Exception):
    def __init__(self, message, status_code=400, error_code=None, details=None):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code
        self.details = details or {}

class ValidationError(APIError):
    def __init__(self, message, details=None):
        super().__init__(message, status_code=422, error_code='VALIDATION_ERROR', details=details)

class NotFoundError(APIError):
    def __init__(self, resource='Resource'):
        super().__init__(f'{resource} not found', status_code=404, error_code='NOT_FOUND')

class AuthenticationError(APIError):
    def __init__(self, message='Authentication required'):
        super().__init__(message, status_code=401, error_code='AUTHENTICATION_ERROR')

# Global error handler
@app.errorhandler(APIError)
def handle_api_error(error):
    response = {
        'error': {
            'message': error.message,
            'code': error.error_code,
            'details': error.details
        }
    }
    
    # Add request ID for tracking
    response['error']['request_id'] = getattr(request, 'request_id', None)
    
    return response, error.status_code

@app.errorhandler(404)
def handle_not_found(error):
    return {
        'error': {
            'message': 'Endpoint not found',
            'code': 'ENDPOINT_NOT_FOUND',
            'request_id': getattr(request, 'request_id', None)
        }
    }, 404

@app.errorhandler(500)
def handle_internal_error(error):
    # Log the error
    app.logger.error(f'Internal server error: {error}')
    
    return {
        'error': {
            'message': 'Internal server error',
            'code': 'INTERNAL_ERROR',
            'request_id': getattr(request, 'request_id', None)
        }
    }, 500
```

### Input Validation
```python
from marshmallow import Schema, fields, validate, ValidationError

class UserSchema(Schema):
    name = fields.Str(required=True, validate=validate.Length(min=1, max=100))
    email = fields.Email(required=True)
    age = fields.Int(validate=validate.Range(min=0, max=150))
    role = fields.Str(validate=validate.OneOf(['user', 'admin', 'moderator']))

def validate_json(schema_class):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not request.is_json:
                raise APIError('Content-Type must be application/json', status_code=400)
            
            schema = schema_class()
            try:
                validated_data = schema.load(request.json)
                request.validated_data = validated_data
            except ValidationError as err:
                raise ValidationError('Validation failed', details=err.messages)
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route('/api/v1/users', methods=['POST'])
@validate_json(UserSchema
