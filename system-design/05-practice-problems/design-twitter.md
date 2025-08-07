# Design Twitter - System Design Practice Problem

## Table of Contents
1. [Problem Statement](#problem-statement)
2. [Requirements Gathering](#requirements-gathering)
3. [Capacity Estimation](#capacity-estimation)
4. [High-Level Design](#high-level-design)
5. [Detailed Component Design](#detailed-component-design)
6. [Database Design](#database-design)
7. [API Design](#api-design)
8. [Scalability Considerations](#scalability-considerations)
9. [Advanced Features](#advanced-features)
10. [Interview Discussion Points](#interview-discussion-points)

---

## Problem Statement

**Design a simplified version of Twitter that supports:**
- Users can post tweets (280 characters)
- Users can follow other users
- Users can view their timeline (tweets from people they follow)
- Users can view their own tweets
- Basic user authentication

**This is a classic system design interview question that tests:**
- Understanding of social media architecture
- Database design and sharding strategies
- Caching mechanisms
- Feed generation algorithms
- Scalability patterns

---

## Requirements Gathering

### Functional Requirements
1. **User Management**
   - User registration and authentication
   - User profiles (username, bio, profile picture)
   - Follow/unfollow other users

2. **Tweet Operations**
   - Post tweets (text, images, videos)
   - Delete tweets
   - Like/unlike tweets
   - Retweet functionality

3. **Timeline/Feed**
   - Home timeline (tweets from followed users)
   - User timeline (user's own tweets)
   - Real-time updates

4. **Search**
   - Search tweets by keywords
   - Search users by username

### Non-Functional Requirements
1. **Scale**
   - 300M monthly active users
   - 200M daily active users
   - 100M tweets per day
   - 500M timeline views per day

2. **Performance**
   - Timeline load time < 200ms
   - Tweet posting < 100ms
   - 99.9% availability

3. **Consistency**
   - Eventual consistency for timeline
   - Strong consistency for user data

---

## Capacity Estimation

### User Statistics
```
Monthly Active Users (MAU): 300M
Daily Active Users (DAU): 200M (67% of MAU)
Average tweets per user per day: 0.5
Total tweets per day: 200M × 0.5 = 100M tweets/day
Peak tweets per second: 100M / (24 × 3600) × 5 = ~5,800 TPS
```

### Storage Estimation
```
Tweet Storage:
- Average tweet size: 280 chars × 2 bytes = 560 bytes
- Metadata (user_id, timestamp, etc.): 200 bytes
- Total per tweet: ~800 bytes
- Daily storage: 100M × 800 bytes = 80GB/day
- Annual storage: 80GB × 365 = ~29TB/year

User Data:
- 300M users × 1KB per user = 300GB

Media Storage:
- Assume 10% tweets have media
- Average media size: 1MB
- Daily media: 10M × 1MB = 10TB/day
- Annual media: 10TB × 365 = 3.65PB/year
```

### Bandwidth Estimation
```
Read:Write Ratio = 100:1 (typical for social media)

Write Bandwidth:
- 5,800 tweets/sec × 800 bytes = 4.6MB/s

Read Bandwidth:
- Timeline reads: 500M/day = 5,800 reads/sec
- Average timeline: 20 tweets × 800 bytes = 16KB
- Read bandwidth: 5,800 × 16KB = 92.8MB/s
```

---

## High-Level Design

### System Architecture
```
[Mobile Apps] ──┐
[Web Client]  ──┼── [Load Balancer] ── [API Gateway]
[3rd Party]   ──┘                           │
                                            │
    ┌───────────────────────────────────────┼───────────────────────────────────────┐
    │                                       │                                       │
    ▼                                       ▼                                       ▼
[User Service]                      [Tweet Service]                        [Timeline Service]
    │                                       │                                       │
    ▼                                       ▼                                       ▼
[User DB]                           [Tweet DB]                            [Timeline Cache]
                                        │                                       │
                                        ▼                                       ▼
                                [Media Storage]                         [Timeline DB]
                                        │
                                        ▼
                                    [CDN]
```

### Core Services
1. **User Service**: User registration, authentication, profiles, follow relationships
2. **Tweet Service**: Tweet creation, deletion, media handling
3. **Timeline Service**: Feed generation and caching
4. **Notification Service**: Real-time notifications
5. **Search Service**: Tweet and user search functionality

---

## Detailed Component Design

### 1. User Service
```python
class UserService:
    def __init__(self, user_db, cache):
        self.user_db = user_db
        self.cache = cache
    
    def create_user(self, username, email, password):
        # Validate input
        if self.user_exists(username):
            raise UserExistsError()
        
        # Hash password
        password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
        
        # Create user
        user = User(
            username=username,
            email=email,
            password_hash=password_hash,
            created_at=datetime.now()
        )
        
        user_id = self.user_db.insert(user)
        self.cache.set(f"user:{user_id}", user, ttl=3600)
        return user_id
    
    def follow_user(self, follower_id, followee_id):
        # Check if users exist
        if not self.user_exists(follower_id) or not self.user_exists(followee_id):
            raise UserNotFoundError()
        
        # Add follow relationship
        self.user_db.add_follow(follower_id, followee_id)
        
        # Invalidate cache
        self.cache.delete(f"followers:{followee_id}")
        self.cache.delete(f"following:{follower_id}")
    
    def get_followers(self, user_id):
        # Try cache first
        cache_key = f"followers:{user_id}"
        followers = self.cache.get(cache_key)
        
        if followers is None:
            followers = self.user_db.get_followers(user_id)
            self.cache.set(cache_key, followers, ttl=1800)
        
        return followers
```

### 2. Tweet Service
```python
class TweetService:
    def __init__(self, tweet_db, media_service, timeline_service):
        self.tweet_db = tweet_db
        self.media_service = media_service
        self.timeline_service = timeline_service
    
    def create_tweet(self, user_id, content, media_urls=None):
        # Validate content
        if len(content) > 280:
            raise TweetTooLongError()
        
        # Process media
        processed_media = []
        if media_urls:
            for url in media_urls:
                processed_url = self.media_service.process_media(url)
                processed_media.append(processed_url)
        
        # Create tweet
        tweet = Tweet(
            user_id=user_id,
            content=content,
            media_urls=processed_media,
            created_at=datetime.now(),
            tweet_id=self.generate_tweet_id()
        )
        
        # Store tweet
        self.tweet_db.insert(tweet)
        
        # Trigger timeline update (async)
        self.timeline_service.fanout_tweet(tweet)
        
        return tweet.tweet_id
    
    def get_user_tweets(self, user_id, limit=20, offset=0):
        return self.tweet_db.get_tweets_by_user(user_id, limit, offset)
    
    def generate_tweet_id(self):
        # Snowflake-like ID generation
        timestamp = int(time.time() * 1000)
        machine_id = self.get_machine_id()
        sequence = self.get_sequence_number()
        
        return (timestamp << 22) | (machine_id << 12) | sequence
```

### 3. Timeline Service
```python
class TimelineService:
    def __init__(self, timeline_cache, user_service, tweet_service):
        self.timeline_cache = timeline_cache
        self.user_service = user_service
        self.tweet_service = tweet_service
    
    def get_home_timeline(self, user_id, limit=20):
        # Try cache first
        cache_key = f"timeline:{user_id}"
        timeline = self.timeline_cache.get(cache_key)
        
        if timeline is None:
            timeline = self.generate_timeline(user_id, limit)
            self.timeline_cache.set(cache_key, timeline, ttl=300)  # 5 min TTL
        
        return timeline[:limit]
    
    def generate_timeline(self, user_id, limit):
        # Get users that this user follows
        following = self.user_service.get_following(user_id)
        
        # Get recent tweets from followed users
        all_tweets = []
        for followed_user_id in following:
            tweets = self.tweet_service.get_user_tweets(followed_user_id, limit=50)
            all_tweets.extend(tweets)
        
        # Sort by timestamp and return top tweets
        all_tweets.sort(key=lambda x: x.created_at, reverse=True)
        return all_tweets[:limit]
    
    def fanout_tweet(self, tweet):
        # Get followers of the tweet author
        followers = self.user_service.get_followers(tweet.user_id)
        
        # Add tweet to each follower's timeline cache
        for follower_id in followers:
            cache_key = f"timeline:{follower_id}"
            
            # Get current timeline
            timeline = self.timeline_cache.get(cache_key) or []
            
            # Add new tweet to the beginning
            timeline.insert(0, tweet)
            
            # Keep only recent tweets (e.g., last 100)
            timeline = timeline[:100]
            
            # Update cache
            self.timeline_cache.set(cache_key, timeline, ttl=300)
```

---

## Database Design

### User Table
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    profile_image_url VARCHAR(500),
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    tweet_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

### Tweets Table
```sql
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    media_urls JSON,
    reply_to_tweet_id BIGINT,
    retweet_of_tweet_id BIGINT,
    like_count INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    reply_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (reply_to_tweet_id) REFERENCES tweets(tweet_id),
    FOREIGN KEY (retweet_of_tweet_id) REFERENCES tweets(tweet_id)
);

CREATE INDEX idx_tweets_user_id ON tweets(user_id);
CREATE INDEX idx_tweets_created_at ON tweets(created_at);
CREATE INDEX idx_tweets_reply_to ON tweets(reply_to_tweet_id);
```

### Follows Table
```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (followee_id) REFERENCES users(user_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_followee ON follows(followee_id);
```

### Likes Table
```sql
CREATE TABLE likes (
    user_id BIGINT NOT NULL,
    tweet_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id)
);

CREATE INDEX idx_likes_tweet_id ON likes(tweet_id);
CREATE INDEX idx_likes_user_id ON likes(user_id);
```

---

## API Design

### RESTful API Endpoints

#### User APIs
```http
POST /api/v1/users
Content-Type: application/json
{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "secure_password"
}

GET /api/v1/users/{user_id}
Response: {
    "user_id": 12345,
    "username": "john_doe",
    "display_name": "John Doe",
    "bio": "Software Engineer",
    "follower_count": 150,
    "following_count": 200,
    "tweet_count": 50
}

POST /api/v1/users/{user_id}/follow
Content-Type: application/json
{
    "followee_id": 67890
}

GET /api/v1/users/{user_id}/followers?limit=20&offset=0
GET /api/v1/users/{user_id}/following?limit=20&offset=0
```

#### Tweet APIs
```http
POST /api/v1/tweets
Content-Type: application/json
Authorization: Bearer <token>
{
    "content": "Hello, Twitter!",
    "media_urls": ["https://example.com/image.jpg"]
}

GET /api/v1/tweets/{tweet_id}
Response: {
    "tweet_id": 98765,
    "user_id": 12345,
    "username": "john_doe",
    "content": "Hello, Twitter!",
    "media_urls": ["https://cdn.example.com/image.jpg"],
    "like_count": 10,
    "retweet_count": 5,
    "created_at": "2023-10-11T10:30:00Z"
}

DELETE /api/v1/tweets/{tweet_id}
Authorization: Bearer <token>

POST /api/v1/tweets/{tweet_id}/like
Authorization: Bearer <token>
```

#### Timeline APIs
```http
GET /api/v1/timeline/home?limit=20&offset=0
Authorization: Bearer <token>
Response: {
    "tweets": [...],
    "next_offset": 20,
    "has_more": true
}

GET /api/v1/timeline/user/{user_id}?limit=20&offset=0
Response: {
    "tweets": [...],
    "next_offset": 20,
    "has_more": true
}
```

---

## Scalability Considerations

### 1. Database Sharding

#### User Sharding
```python
def get_user_shard(user_id):
    return user_id % NUM_USER_SHARDS

# Shard 0: user_ids ending in 0, 3, 6, 9
# Shard 1: user_ids ending in 1, 4, 7
# Shard 2: user_ids ending in 2, 5, 8
```

#### Tweet Sharding
```python
def get_tweet_shard(tweet_id):
    # Use timestamp-based sharding for better query performance
    timestamp = extract_timestamp_from_tweet_id(tweet_id)
    return timestamp % NUM_TWEET_SHARDS

# Alternative: Hash-based sharding
def get_tweet_shard_by_hash(tweet_id):
    return hash(tweet_id) % NUM_TWEET_SHARDS
```

### 2. Caching Strategy

#### Multi-Level Caching
```python
class CacheManager:
    def __init__(self):
        self.l1_cache = LRUCache(1000)      # In-memory cache
        self.l2_cache = RedisCache()        # Distributed cache
        self.l3_cache = Database()          # Database
    
    def get(self, key):
        # Try L1 cache first
        value = self.l1_cache.get(key)
        if value is not None:
            return value
        
        # Try L2 cache
        value = self.l2_cache.get(key)
        if value is not None:
            self.l1_cache.put(key, value)
            return value
        
        # Fallback to database
        value = self.l3_cache.get(key)
        if value is not None:
            self.l1_cache.put(key, value)
            self.l2_cache.put(key, value, ttl=3600)
        
        return value
```

#### Cache Keys Strategy
```python
# User cache keys
user_profile = f"user:profile:{user_id}"
user_followers = f"user:followers:{user_id}"
user_following = f"user:following:{user_id}"

# Tweet cache keys
tweet_detail = f"tweet:{tweet_id}"
user_tweets = f"tweets:user:{user_id}:page:{page}"

# Timeline cache keys
home_timeline = f"timeline:home:{user_id}"
user_timeline = f"timeline:user:{user_id}"
```

### 3. Timeline Generation Strategies

#### Push Model (Fanout on Write)
```python
def fanout_on_write(tweet):
    """
    Pros: Fast read (timeline pre-computed)
    Cons: Slow write for users with many followers
    """
    followers = get_followers(tweet.user_id)
    
    for follower_id in followers:
        timeline_key = f"timeline:{follower_id}"
        timeline = cache.get(timeline_key) or []
        timeline.insert(0, tweet)
        timeline = timeline[:100]  # Keep recent tweets
        cache.set(timeline_key, timeline)
```

#### Pull Model (Fanout on Read)
```python
def fanout_on_read(user_id):
    """
    Pros: Fast write, handles celebrity users well
    Cons: Slow read (timeline computed on demand)
    """
    following = get_following(user_id)
    
    all_tweets = []
    for followed_user_id in following:
        tweets = get_recent_tweets(followed_user_id, limit=20)
        all_tweets.extend(tweets)
    
    # Sort and return top tweets
    all_tweets.sort(key=lambda x: x.created_at, reverse=True)
    return all_tweets[:20]
```

#### Hybrid Model
```python
def hybrid_fanout(tweet):
    """
    Push for users with < 1M followers
    Pull for celebrity users with > 1M followers
    """
    follower_count = get_follower_count(tweet.user_id)
    
    if follower_count < 1_000_000:
        fanout_on_write(tweet)
    else:
        # Store in celebrity tweets cache
        celebrity_tweets_key = f"celebrity_tweets:{tweet.user_id}"
        cache.lpush(celebrity_tweets_key, tweet)
        cache.ltrim(celebrity_tweets_key, 0, 99)  # Keep recent 100
```

### 4. Load Balancing

#### Geographic Load Balancing
```
US East: us-east-1.twitter.com
US West: us-west-1.twitter.com
Europe: eu-west-1.twitter.com
Asia: ap-southeast-1.twitter.com
```

#### Service-Level Load Balancing
```
Read Replicas: 
- Timeline reads → Read replicas
- User profile reads → Read replicas

Write Masters:
- Tweet creation → Write master
- User updates → Write master
```

---

## Advanced Features

### 1. Real-Time Updates
```python
# WebSocket implementation for real-time timeline updates
class TimelineWebSocket:
    def __init__(self):
        self.connections = {}  # user_id -> websocket_connection
    
    def connect(self, user_id, websocket):
        self.connections[user_id] = websocket
    
    def broadcast_tweet(self, tweet):
        # Get followers of tweet author
        followers = get_followers(tweet.user_id)
        
        # Send real-time update to online followers
        for follower_id in followers:
            if follower_id in self.connections:
                websocket = self.connections[follower_id]
                websocket.send(json.dumps({
                    'type': 'new_tweet',
                    'tweet': tweet.to_dict()
                }))
```

### 2. Trending Topics
```python
class TrendingService:
    def __init__(self):
        self.hashtag_counter = Counter()
        self.time_window = 3600  # 1 hour window
    
    def extract_hashtags(self, tweet_content):
        return re.findall(r'#\w+', tweet_content)
    
    def update_trends(self, tweet):
        hashtags = self.extract_hashtags(tweet.content)
        
        for hashtag in hashtags:
            # Increment counter with timestamp
            key = f"{hashtag}:{int(time.time() // self.time_window)}"
            self.hashtag_counter[key] += 1
    
    def get_trending_topics(self, limit=10):
        current_window = int(time.time() // self.time_window)
        
        # Get counts for current window
        current_trends = {}
        for key, count in self.hashtag_counter.items():
            hashtag, window = key.split(':')
            if int(window) == current_window:
                current_trends[hashtag] = current_trends.get(hashtag, 0) + count
        
        # Return top trending hashtags
        return sorted(current_trends.items(), key=lambda x: x[1], reverse=True)[:limit]
```

### 3. Content Recommendation
```python
class RecommendationService:
    def __init__(self, ml_model):
        self.ml_model = ml_model
    
    def get_recommended_tweets(self, user_id, limit=10):
        # Get user's interaction history
        user_likes = get_user_likes(user_id)
        user_retweets = get_user_retweets(user_id)
        following = get_following(user_id)
        
        # Feature extraction
        features = {
            'liked_topics': extract_topics(user_likes),
            'retweeted_topics': extract_topics(user_retweets),
            'following_interests': get_following_interests(following),
            'time_of_day': datetime.now().hour,
            'day_of_week': datetime.now().weekday()
        }
        
        # Get candidate tweets
        candidate_tweets = get_candidate_tweets(user_id)
        
        # Score tweets using ML model
        scored_tweets = []
        for tweet in candidate_tweets:
            tweet_features = extract_tweet_features(tweet)
            score = self.ml_model.predict(features, tweet_features)
            scored_tweets.append((tweet, score))
        
        # Return top scored tweets
        scored_tweets.sort(key=lambda x: x[1], reverse=True)
        return [tweet for tweet, score in scored_tweets[:limit]]
```

---

## Interview Discussion Points

### 1. Trade-offs Discussion

**Consistency vs Availability**
- **Strong Consistency**: User data, follow relationships
- **Eventual Consistency**: Timeline, like counts, follower counts
- **Why**: Timeline can tolerate slight delays, but user auth cannot

**Push vs Pull vs Hybrid Timeline**
- **Push**: Fast reads, slow writes for celebrities
- **Pull**: Fast writes, slow reads
- **Hybrid**: Best of both worlds, more complex

### 2. Bottleneck Analysis

**Database Bottlenecks**
- Hot partitions (celebrity users)
- Write-heavy workload during peak hours
- Complex timeline queries

**Solutions**
- Read replicas for timeline queries
- Caching layer for hot data
- Async processing for non-critical operations

### 3. Failure Scenarios

**Database Failure**
- Master-slave replication
- Automatic failover
- Data backup and recovery

**Cache Failure**
- Cache warming strategies
- Graceful degradation
- Circuit breaker pattern

### 4. Monitoring and Metrics

**Key Metrics**
- Tweet creation rate
- Timeline load time
- Cache hit ratio
- Database query performance
- User engagement metrics

**Alerting**
- High error rates
- Slow response times
- Database connection pool exhaustion
- Cache memory usage

### 5. Security Considerations

**Authentication & Authorization**
- JWT tokens for API access
- OAuth for third-party integrations
- Rate limiting per user/IP

**Content Security**
- Input validation and sanitization
- Spam detection algorithms
- Content moderation systems

---

## Scaling Numbers Validation

### Final Architecture Capacity
```
Load Balancers: 10 instances
API Servers: 100 instances (1000 RPS each)
Database Shards: 20 shards
Cache Clusters: 10 Redis clusters
CDN: Global distribution

Total Capacity:
- 100,000 RPS for reads
- 10,000 RPS for writes
- 50TB storage (with replication)
- 99.9% availability
```

### Cost Estimation (AWS)
```
Compute: $50,000/month
Storage: $30,000/month
Network: $20,000/month
Cache: $15,000/month
CDN: $10,000/month
Total: ~$125,000/month
```

---

## Key Takeaways

1. **Start with requirements** - Clarify functional and non-functional requirements
2. **Estimate capacity** - Calculate storage, bandwidth, and QPS requirements
3. **Design for scale** - Use sharding, caching, and load balancing
4. **Consider trade-offs** - Consistency vs availability, push vs pull
5. **Plan for failures** - Implement redundancy and graceful degradation
6. **Monitor everything** - Track key metrics and set up alerting

**Remember**: This is a complex system that evolved over many years. Start simple and add complexity as needed. Focus on the core functionality first, then discuss advanced features and optimizations.
