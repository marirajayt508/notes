# Consistent Hashing - Algorithmic Patterns

## Table of Contents
1. [What is Consistent Hashing?](#what-is-consistent-hashing)
2. [Problems with Traditional Hashing](#problems-with-traditional-hashing)
3. [Consistent Hashing Algorithm](#consistent-hashing-algorithm)
4. [Virtual Nodes](#virtual-nodes)
5. [Implementation Examples](#implementation-examples)
6. [Real-World Applications](#real-world-applications)
7. [Trade-offs and Considerations](#trade-offs-and-considerations)
8. [Interview Questions](#interview-questions)

---

## What is Consistent Hashing?

**Consistent Hashing** is a distributed hashing scheme that operates independently of the number of servers or objects in a distributed hash table. It was introduced by Karger et al. at MIT for use in distributed caching.

### Key Properties:
- **Minimal Redistribution**: When nodes are added/removed, only a small fraction of keys need to be redistributed
- **Load Distribution**: Keys are distributed roughly evenly across nodes
- **Fault Tolerance**: System continues to work when nodes fail
- **Scalability**: Easy to add/remove nodes without major disruption

### Use Cases:
- Distributed caching systems (Memcached, Redis Cluster)
- Content Delivery Networks (CDNs)
- Database sharding
- Load balancing
- Peer-to-peer networks

---

## Problems with Traditional Hashing

### Simple Hash Function Approach
```python
# Traditional hashing approach
def get_server(key, servers):
    hash_value = hash(key)
    server_index = hash_value % len(servers)
    return servers[server_index]

# Example with 3 servers
servers = ['server1', 'server2', 'server3']
keys = ['user1', 'user2', 'user3', 'user4', 'user5']

for key in keys:
    server = get_server(key, servers)
    print(f"{key} -> {server}")

# Output:
# user1 -> server2
# user2 -> server3  
# user3 -> server1
# user4 -> server2
# user5 -> server3
```

### Problems When Scaling
```python
# What happens when we add a server?
servers_new = ['server1', 'server2', 'server3', 'server4']

print("After adding server4:")
for key in keys:
    old_server = get_server(key, servers)
    new_server = get_server(key, servers_new)
    if old_server != new_server:
        print(f"{key}: {old_server} -> {new_server} (MOVED)")
    else:
        print(f"{key}: {old_server} (SAME)")

# Output:
# user1: server2 -> server2 (SAME)
# user2: server3 -> server3 (SAME)  
# user3: server1 -> server3 (MOVED)
# user4: server2 -> server4 (MOVED)
# user5: server3 -> server1 (MOVED)
```

**Issues:**
- **Massive Redistribution**: 60% of keys moved when adding one server
- **Cache Invalidation**: Most cached data becomes invalid
- **Hot Spots**: Uneven distribution during transitions
- **Downtime**: System disruption during rebalancing

---

## Consistent Hashing Algorithm

### Basic Concept
```
Hash Ring (0 to 2^32-1):

    0
    |
2^32-1 ---- 2^30
    |         |
    |    Key3  |
2^31 ---- 2^31.5
    |         |
  Key1      Key2
    |         |
2^30.5 ---- 2^29
    |
  Server1
```

### Algorithm Steps:
1. **Create Hash Ring**: Map hash values to a circle (0 to 2^32-1)
2. **Place Servers**: Hash server identifiers and place on ring
3. **Place Keys**: Hash keys and place on ring
4. **Find Server**: For each key, find the first server clockwise on the ring

### Basic Implementation
```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, nodes=None):
        self.nodes = nodes or []
        self.ring = {}
        self.sorted_keys = []
        self._build_ring()
    
    def _hash(self, key):
        """Generate hash value for a key"""
        return int(hashlib.md5(str(key).encode('utf-8')).hexdigest(), 16)
    
    def _build_ring(self):
        """Build the hash ring with current nodes"""
        self.ring = {}
        self.sorted_keys = []
        
        for node in self.nodes:
            key = self._hash(node)
            self.ring[key] = node
            self.sorted_keys.append(key)
        
        self.sorted_keys.sort()
    
    def add_node(self, node):
        """Add a new node to the ring"""
        if node not in self.nodes:
            self.nodes.append(node)
            key = self._hash(node)
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
    
    def remove_node(self, node):
        """Remove a node from the ring"""
        if node in self.nodes:
            self.nodes.remove(node)
            key = self._hash(node)
            del self.ring[key]
            self.sorted_keys.remove(key)
    
    def get_node(self, key):
        """Get the node responsible for a key"""
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        
        # Find the first node clockwise from the key
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]

# Example usage
ch = ConsistentHash(['server1', 'server2', 'server3'])

keys = ['user1', 'user2', 'user3', 'user4', 'user5']
print("Initial distribution:")
for key in keys:
    server = ch.get_node(key)
    print(f"{key} -> {server}")

print("\nAfter adding server4:")
ch.add_node('server4')
for key in keys:
    server = ch.get_node(key)
    print(f"{key} -> {server}")
```

---

## Virtual Nodes

### Problem with Basic Consistent Hashing
```python
# With only 3 servers, distribution might be uneven
ch = ConsistentHash(['server1', 'server2', 'server3'])

# Simulate 1000 keys
import random
key_distribution = {}
for i in range(1000):
    key = f"key{i}"
    server = ch.get_node(key)
    key_distribution[server] = key_distribution.get(server, 0) + 1

print("Distribution without virtual nodes:")
for server, count in key_distribution.items():
    print(f"{server}: {count} keys ({count/10:.1f}%)")
```

### Virtual Nodes Solution
```python
class ConsistentHashWithVirtualNodes:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.nodes = nodes or []
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self._build_ring()
    
    def _hash(self, key):
        return int(hashlib.md5(str(key).encode('utf-8')).hexdigest(), 16)
    
    def _build_ring(self):
        self.ring = {}
        self.sorted_keys = []
        
        for node in self.nodes:
            for i in range(self.virtual_nodes):
                virtual_key = self._hash(f"{node}:{i}")
                self.ring[virtual_key] = node
                self.sorted_keys.append(virtual_key)
        
        self.sorted_keys.sort()
    
    def add_node(self, node):
        if node not in self.nodes:
            self.nodes.append(node)
            for i in range(self.virtual_nodes):
                virtual_key = self._hash(f"{node}:{i}")
                self.ring[virtual_key] = node
                bisect.insort(self.sorted_keys, virtual_key)
    
    def remove_node(self, node):
        if node in self.nodes:
            self.nodes.remove(node)
            keys_to_remove = []
            for virtual_key, n in self.ring.items():
                if n == node:
                    keys_to_remove.append(virtual_key)
            
            for key in keys_to_remove:
                del self.ring[key]
                self.sorted_keys.remove(key)
            
            self.nodes.remove(node)
    
    def get_node(self, key):
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes(self, key, count=1):
        """Get multiple nodes for replication"""
        if not self.ring or count <= 0:
            return []
        
        hash_key = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        
        nodes = []
        seen = set()
        
        for i in range(len(self.sorted_keys)):
            if len(nodes) >= count:
                break
            
            current_idx = (idx + i) % len(self.sorted_keys)
            node = self.ring[self.sorted_keys[current_idx]]
            
            if node not in seen:
                nodes.append(node)
                seen.add(node)
        
        return nodes

# Test virtual nodes distribution
ch_virtual = ConsistentHashWithVirtualNodes(['server1', 'server2', 'server3'], virtual_nodes=150)

key_distribution = {}
for i in range(1000):
    key = f"key{i}"
    server = ch_virtual.get_node(key)
    key_distribution[server] = key_distribution.get(server, 0) + 1

print("Distribution with virtual nodes:")
for server, count in key_distribution.items():
    print(f"{server}: {count} keys ({count/10:.1f}%)")
```

### Benefits of Virtual Nodes:
- **Better Distribution**: More even key distribution
- **Smoother Scaling**: Adding/removing nodes affects smaller ranges
- **Fault Tolerance**: Node failures have less impact
- **Flexibility**: Can assign different numbers of virtual nodes based on capacity

---

## Implementation Examples

### Redis Cluster Implementation
```python
class RedisClusterHash:
    def __init__(self, nodes):
        self.nodes = nodes
        self.slots = 16384  # Redis uses 16384 hash slots
        self.slot_to_node = {}
        self._assign_slots()
    
    def _assign_slots(self):
        """Assign hash slots to nodes"""
        slots_per_node = self.slots // len(self.nodes)
        
        for i, node in enumerate(self.nodes):
            start_slot = i * slots_per_node
            end_slot = start_slot + slots_per_node - 1
            
            if i == len(self.nodes) - 1:  # Last node gets remaining slots
                end_slot = self.slots - 1
            
            for slot in range(start_slot, end_slot + 1):
                self.slot_to_node[slot] = node
    
    def _get_slot(self, key):
        """Calculate CRC16 hash slot for key"""
        import crc16
        return crc16.crc16xmodem(key.encode()) % self.slots
    
    def get_node(self, key):
        slot = self._get_slot(key)
        return self.slot_to_node[slot]
    
    def add_node(self, node):
        """Add node and rebalance slots"""
        self.nodes.append(node)
        self._rebalance_slots()
    
    def _rebalance_slots(self):
        """Redistribute slots evenly among all nodes"""
        self.slot_to_node = {}
        self._assign_slots()

# Usage
redis_cluster = RedisClusterHash(['redis1:6379', 'redis2:6379', 'redis3:6379'])
```

### Distributed Cache Implementation
```python
class DistributedCache:
    def __init__(self, cache_nodes, replication_factor=2):
        self.consistent_hash = ConsistentHashWithVirtualNodes(cache_nodes)
        self.replication_factor = replication_factor
        self.caches = {node: {} for node in cache_nodes}  # Simulate cache storage
    
    def put(self, key, value):
        """Store key-value pair with replication"""
        nodes = self.consistent_hash.get_nodes(key, self.replication_factor)
        
        success_count = 0
        for node in nodes:
            try:
                self.caches[node][key] = value
                success_count += 1
            except Exception as e:
                print(f"Failed to write to {node}: {e}")
        
        return success_count > 0
    
    def get(self, key):
        """Retrieve value for key"""
        nodes = self.consistent_hash.get_nodes(key, self.replication_factor)
        
        for node in nodes:
            try:
                if key in self.caches[node]:
                    return self.caches[node][key]
            except Exception as e:
                print(f"Failed to read from {node}: {e}")
                continue
        
        return None
    
    def delete(self, key):
        """Delete key from all replicas"""
        nodes = self.consistent_hash.get_nodes(key, self.replication_factor)
        
        for node in nodes:
            try:
                if key in self.caches[node]:
                    del self.caches[node][key]
            except Exception as e:
                print(f"Failed to delete from {node}: {e}")
    
    def add_node(self, node):
        """Add new cache node"""
        self.caches[node] = {}
        self.consistent_hash.add_node(node)
        # In real implementation, would need to migrate data
    
    def remove_node(self, node):
        """Remove cache node"""
        if node in self.caches:
            del self.caches[node]
        self.consistent_hash.remove_node(node)

# Usage
cache = DistributedCache(['cache1', 'cache2', 'cache3'], replication_factor=2)
cache.put('user:123', {'name': 'John', 'email': 'john@example.com'})
user_data = cache.get('user:123')
```

### Load Balancer with Consistent Hashing
```python
class ConsistentHashLoadBalancer:
    def __init__(self, servers, virtual_nodes=150):
        self.consistent_hash = ConsistentHashWithVirtualNodes(servers, virtual_nodes)
        self.server_weights = {server: 1 for server in servers}
        self.server_health = {server: True for server in servers}
    
    def get_server(self, client_id):
        """Get server for client using consistent hashing"""
        # Filter healthy servers
        healthy_servers = [s for s in self.consistent_hash.nodes if self.server_health[s]]
        
        if not healthy_servers:
            return None
        
        # Create temporary hash ring with only healthy servers
        temp_hash = ConsistentHashWithVirtualNodes(healthy_servers)
        return temp_hash.get_node(client_id)
    
    def add_server(self, server, weight=1):
        """Add new server to load balancer"""
        self.server_weights[server] = weight
        self.server_health[server] = True
        
        # Add multiple virtual nodes based on weight
        for i in range(weight * 150):
            self.consistent_hash.add_node(f"{server}:{i}")
    
    def mark_server_unhealthy(self, server):
        """Mark server as unhealthy (health check failure)"""
        self.server_health[server] = False
    
    def mark_server_healthy(self, server):
        """Mark server as healthy"""
        self.server_health[server] = True
    
    def get_server_distribution(self, num_clients=1000):
        """Analyze load distribution across servers"""
        distribution = {}
        
        for i in range(num_clients):
            client_id = f"client_{i}"
            server = self.get_server(client_id)
            if server:
                distribution[server] = distribution.get(server, 0) + 1
        
        return distribution

# Usage
lb = ConsistentHashLoadBalancer(['server1', 'server2', 'server3'])
distribution = lb.get_server_distribution()
print("Load distribution:", distribution)
```

---

## Real-World Applications

### 1. Amazon DynamoDB
```python
# DynamoDB uses consistent hashing for partitioning
class DynamoDBPartitioning:
    def __init__(self, partitions):
        self.partitions = partitions
        self.consistent_hash = ConsistentHashWithVirtualNodes(
            [f"partition_{i}" for i in range(partitions)]
        )
    
    def get_partition(self, partition_key):
        """Get partition for a given partition key"""
        return self.consistent_hash.get_node(partition_key)
    
    def write_item(self, partition_key, sort_key, item):
        partition = self.get_partition(partition_key)
        # Write to the determined partition
        return f"Writing to {partition}: {partition_key}#{sort_key}"
    
    def read_item(self, partition_key, sort_key):
        partition = self.get_partition(partition_key)
        # Read from the determined partition
        return f"Reading from {partition}: {partition_key}#{sort_key}"

# Usage
dynamo = DynamoDBPartitioning(partitions=100)
partition = dynamo.get_partition("user_123")
```

### 2. Content Delivery Network (CDN)
```python
class CDNConsistentHash:
    def __init__(self, edge_servers):
        self.consistent_hash = ConsistentHashWithVirtualNodes(edge_servers)
        self.server_locations = {}  # server -> geographic location
    
    def get_edge_server(self, content_url, client_location=None):
        """Get edge server for content caching"""
        # Primary selection based on content hash
        primary_server = self.consistent_hash.get_node(content_url)
        
        # Secondary selection based on geographic proximity
        if client_location:
            nearby_servers = self.get_nearby_servers(client_location)
            if primary_server in nearby_servers:
                return primary_server
            else:
                # Return closest server that also has good hash distribution
                return self.get_best_nearby_server(content_url, nearby_servers)
        
        return primary_server
    
    def cache_content(self, content_url, content_data):
        """Cache content on appropriate edge servers"""
        # Get multiple servers for redundancy
        servers = self.consistent_hash.get_nodes(content_url, count=3)
        
        for server in servers:
            # Cache content on each server
            self.cache_on_server(server, content_url, content_data)
    
    def get_nearby_servers(self, client_location):
        # Implementation would use geographic distance calculation
        return ['edge_server_1', 'edge_server_2']
    
    def get_best_nearby_server(self, content_url, nearby_servers):
        # Choose best server from nearby options
        temp_hash = ConsistentHashWithVirtualNodes(nearby_servers)
        return temp_hash.get_node(content_url)
    
    def cache_on_server(self, server, url, data):
        # Implementation would cache data on specific server
        pass
```

### 3. Database Sharding
```python
class DatabaseSharding:
    def __init__(self, database_shards):
        self.consistent_hash = ConsistentHashWithVirtualNodes(database_shards)
        self.shard_connections = {}  # shard -> database_connection
    
    def get_shard(self, user_id):
        """Get database shard for user"""
        return self.consistent_hash.get_node(str(user_id))
    
    def insert_user(self, user_id, user_data):
        """Insert user into appropriate shard"""
        shard = self.get_shard(user_id)
        connection = self.shard_connections[shard]
        
        # Execute insert on the correct shard
        return connection.execute(
            "INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)",
            (user_id, user_data['name'], user_data['email'])
        )
    
    def get_user(self, user_id):
        """Get user from appropriate shard"""
        shard = self.get_shard(user_id)
        connection = self.shard_connections[shard]
        
        return connection.execute(
            "SELECT * FROM users WHERE user_id = ?",
            (user_id,)
        ).fetchone()
    
    def add_shard(self, new_shard, connection):
        """Add new database shard"""
        self.shard_connections[new_shard] = connection
        self.consistent_hash.add_node(new_shard)
        
        # In production, would need to migrate data
        self.migrate_data_to_new_shard(new_shard)
    
    def migrate_data_to_new_shard(self, new_shard):
        """Migrate data that should now belong to new shard"""
        # Implementation would:
        # 1. Identify keys that should move to new shard
        # 2. Copy data from old shards to new shard
        # 3. Delete data from old shards
        # 4. Update application routing
        pass
```

---

## Trade-offs and Considerations

### Advantages
```
✅ Minimal Redistribution: Only K/N keys move when adding/removing nodes
✅ Load Distribution: Virtual nodes ensure even distribution
✅ Fault Tolerance: System continues working with node failures
✅ Scalability: Easy to add/remove nodes
✅ No Central Coordinator: Fully distributed algorithm
```

### Disadvantages
```
❌ Range Queries: Cannot efficiently handle range queries
❌ Complexity: More complex than simple hashing
❌ Memory Overhead: Virtual nodes require additional memory
❌ Hot Spots: Popular keys can still create hot spots
❌ Rebalancing Cost: Data migration still required when scaling
```

### Performance Characteristics
```python
# Time Complexity Analysis
class ConsistentHashPerformance:
    """
    Operations Time Complexity:
    - get_node(key): O(log N) where N is number of virtual nodes
    - add_node(): O(V log N) where V is virtual nodes per physical node
    - remove_node(): O(V log N)
    
    Space Complexity: O(N * V) where N is nodes, V is virtual nodes per node
    """
    
    def benchmark_operations(self):
        import time
        
        # Test with different numbers of nodes
        for num_nodes in [10, 100, 1000]:
            nodes = [f"node_{i}" for i in range(num_nodes)]
            ch = ConsistentHashWithVirtualNodes(nodes, virtual_nodes=150)
            
            # Benchmark get_node operation
            start_time = time.time()
            for i in range(10000):
                ch.get_node(f"key_{i}")
            end_time = time.time()
            
            avg_time = (end_time - start_time) / 10000 * 1000000  # microseconds
            print(f"Nodes: {num_nodes}, Avg get_node time: {avg_time:.2f} μs")

# Run benchmark
benchmark = ConsistentHashPerformance()
benchmark.benchmark_operations()
```

### Configuration Guidelines
```python
class ConsistentHashConfig:
    """Best practices for consistent hashing configuration"""
    
    @staticmethod
    def calculate_virtual_nodes(num_physical_nodes, target_balance=0.1):
        """
        Calculate optimal number of virtual nodes
        
        Rule of thumb: 150-200 virtual nodes per physical node
        More virtual nodes = better balance but more memory
        """
        if num_physical_nodes <= 10:
            return 200  # Higher for small clusters
        elif num_physical_nodes <= 100:
            return 150  # Standard configuration
        else:
            return 100  # Lower for large clusters
    
    @staticmethod
    def choose_hash_function():
        """
        Hash function selection:
        - MD5: Good distribution, widely supported
        - SHA-1: Better security, slightly slower
        - CRC32: Faster, but less uniform distribution
        - MurmurHash: Fast and good distribution
        """
        return "MD5"  # Most common choice
    
    @staticmethod
    def replication_factor(consistency_requirement):
        """
        Choose replication factor based on requirements:
        - High consistency: 3+ replicas
        - Balanced: 2 replicas  
        - Performance focused: 1 replica
        """
        if consistency_requirement == "high":
            return 3
        elif consistency_requirement == "medium":
            return 2
        else:
            return 1
```

---

## Interview Questions

### Q1: Explain consistent hashing and why it's better than simple hashing.

**Answer Framework:**
1. **Problem with Simple Hashing**: When nodes change, most keys need redistribution
2. **Consistent Hashing Solution**: Only K/N keys move when adding/removing nodes
3. **Algorithm**: Hash ring, place nodes and keys, find clockwise server
4. **Benefits**: Minimal redistribution, fault tolerance, scalability

### Q2: What are virtual nodes and why are they important?

**Key Points:**
- **Problem**: Uneven distribution with few physical nodes
- **Solution**: Each physical node gets multiple virtual nodes on ring
- **Benefits**: Better load distribution, smoother scaling
- **Trade-off**: More memory usage vs better balance

### Q3: How would you implement consistent hashing for a distributed cache?

**Implementation Steps:**
1. Create hash ring with virtual nodes
2. Implement get_node() method with binary search
3. Add replication for fault tolerance
4. Handle node addition/removal with data migration
5. Include health checking and failover

### Q4: What are the limitations of consistent hashing?

**Limitations:**
1. **Range Queries**: Cannot efficiently handle range queries
2. **Hot Spots**: Popular keys can still overload nodes
3. **Complexity**: More complex than simple hashing
4. **Memory Overhead**: Virtual nodes require extra memory

### Q5: How does consistent hashing work in real systems like DynamoDB or Cassandra?

**Real-World Examples:**
- **DynamoDB**: Uses consistent hashing for partition key distribution
- **Cassandra**: Combines consistent hashing with replication strategies
- **Redis Cluster**: Uses hash slots (16384) instead of pure consistent hashing
- **Amazon Dynamo**: Original paper that popularized consistent hashing

---

## Key Takeaways

1. **Consistent hashing solves the redistribution problem** when scaling distributed systems
2. **Virtual nodes are essential** for even load distribution
3. **Trade-offs exist** between complexity and benefits
4. **Real systems often modify** the basic algorithm for specific needs
5. **Understanding the algorithm** is crucial for system design interviews

Remember: Consistent hashing is a fundamental building block for many distributed systems. Master the concept, implementation, and trade-offs for system design success!
