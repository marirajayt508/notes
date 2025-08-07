# CAP Theorem

The CAP theorem, also known as Brewer's theorem, is a fundamental principle in distributed systems that states it is impossible for a distributed data store to simultaneously provide more than two out of the following three guarantees: Consistency, Availability, and Partition tolerance.

## Table of Contents
- [Understanding CAP](#understanding-cap)
- [The Three Properties](#the-three-properties)
- [CAP Trade-offs](#cap-trade-offs)
- [Real-World Examples](#real-world-examples)
- [Beyond CAP: PACELC](#beyond-cap-pacelc)
- [Practical Implications](#practical-implications)
- [Interview Questions & Answers](#interview-questions--answers)

## Understanding CAP

### Historical Context
- **Proposed by**: Eric Brewer in 2000
- **Formalized by**: Seth Gilbert and Nancy Lynch in 2002
- **Core insight**: In the presence of network partitions, you must choose between consistency and availability

### The Theorem Statement
> "In a distributed system, you can have at most two of the following three properties:
> - **Consistency (C)**: All nodes see the same data simultaneously
> - **Availability (A)**: System remains operational and responsive
> - **Partition Tolerance (P)**: System continues despite network failures"

## The Three Properties

### 1. Consistency (C)

**Definition**: All nodes in the system return the same, most recent data for any given piece of information.

**Characteristics**:
- **Strong Consistency**: All reads receive the most recent write
- **Eventual Consistency**: System will become consistent over time
- **Weak Consistency**: No guarantees about when data will be consistent

**Example**:
```python
# Strong Consistency Example
class StronglyConsistentDatabase:
    def __init__(self):
        self.data = {}
        self.version = 0
        self.nodes = []
    
    def write(self, key, value):
        # Must update ALL nodes before confirming write
        self.version += 1
        new_record = {
            'value': value,
            'version': self.version,
            'timestamp': time.time()
        }
        
        # Synchronous replication to all nodes
        for node in self.nodes:
            if not node.update(key, new_record):
                raise Exception("Write failed - consistency violated")
        
        self.data[key] = new_record
        return True
    
    def read(self, key):
        # All nodes must return the same value
        values = [node.get(key) for node in self.nodes]
        
        # Verify consistency
        if not all(v['version'] == values[0]['version'] for v in values):
            raise Exception("Inconsistent state detected")
        
        return values[0]['value']
```

### 2. Availability (A)

**Definition**: Every request receives a response, without guarantee that it contains the most recent data.

**Characteristics**:
- System remains operational even during failures
- Requests are always processed (though data might be stale)
- No single point of failure

**Example**:
```python
# High Availability Example
class HighlyAvailableDatabase:
    def __init__(self):
        self.primary_node = DatabaseNode("primary")
        self.replica_nodes = [
            DatabaseNode("replica1"),
            DatabaseNode("replica2"),
            DatabaseNode("replica3")
        ]
    
    def write(self, key, value):
        # Write to primary, replicate asynchronously
        try:
            self.primary_node.write(key, value)
            
            # Async replication (fire and forget)
            for replica in self.replica_nodes:
                replica.async_write(key, value)
            
            return True
        except Exception:
            # If primary fails, promote a replica
            self.failover_to_replica()
            return self.write(key, value)
    
    def read(self, key):
        # Try primary first, fallback to replicas
        try:
            return self.primary_node.read(key)
        except Exception:
            # Try replicas if primary is down
            for replica in self.replica_nodes:
                try:
                    return replica.read(key)
                except Exception:
                    continue
            
            raise Exception("All nodes unavailable")
    
    def failover_to_replica(self):
        # Promote first available replica to primary
        for replica in self.replica_nodes:
            if replica.is_healthy():
                self.primary_node = replica
                self.replica_nodes.remove(replica)
                break
```

### 3. Partition Tolerance (P)

**Definition**: The system continues to operate despite arbitrary message loss or failure of part of the system.

**Characteristics**:
- Network can be unreliable
- Messages between nodes may be lost or delayed
- System must handle network splits gracefully

**Example**:
```python
# Partition Tolerant Example
class PartitionTolerantDatabase:
    def __init__(self, node_id, cluster_nodes):
        self.node_id = node_id
        self.cluster_nodes = cluster_nodes
        self.data = {}
        self.vector_clock = {}
        self.partition_detected = False
    
    def detect_partition(self):
        """Detect if this node is partitioned from others"""
        reachable_nodes = 0
        for node in self.cluster_nodes:
            if self.can_reach(node):
                reachable_nodes += 1
        
        # If we can't reach majority, we're likely partitioned
        self.partition_detected = reachable_nodes < len(self.cluster_nodes) // 2
        return self.partition_detected
    
    def write(self, key, value):
        if self.detect_partition():
            # During partition, continue accepting writes
            # but mark them as potentially conflicting
            self.data[key] = {
                'value': value,
                'timestamp': time.time(),
                'node_id': self.node_id,
                'partition_write': True
            }
            return True
        else:
            # Normal operation - try to replicate
            return self.replicated_write(key, value)
    
    def read(self, key):
        # Always serve reads, even during partitions
        if key in self.data:
            return self.data[key]['value']
        return None
    
    def resolve_conflicts_after_partition(self):
        """Resolve conflicts when partition heals"""
        for node in self.cluster_nodes:
            if self.can_reach(node):
                # Exchange data and resolve conflicts
                self.merge_data_with_node(node)
```

## CAP Trade-offs

### CP Systems (Consistency + Partition Tolerance)
**Choose consistency over availability during partitions**

**Examples**: MongoDB, HBase, Redis Cluster, RDBMS with ACID

**Characteristics**:
- Strong consistency guarantees
- May become unavailable during network partitions
- Suitable for financial systems, inventory management

```python
class CPSystem:
    def __init__(self):
        self.nodes = []
        self.quorum_size = len(self.nodes) // 2 + 1
    
    def write(self, key, value):
        # Must get acknowledgment from majority (quorum)
        acks = 0
        for node in self.nodes:
            try:
                if node.write(key, value):
                    acks += 1
            except NetworkPartitionException:
                continue
        
        if acks >= self.quorum_size:
            return True
        else:
            # Not enough nodes reachable - reject write
            raise UnavailableException("Cannot maintain consistency")
    
    def read(self, key):
        # Read from majority to ensure consistency
        responses = []
        for node in self.nodes:
            try:
                responses.append(node.read(key))
            except NetworkPartitionException:
                continue
        
        if len(responses) >= self.quorum_size:
            # Return most recent version
            return max(responses, key=lambda x: x.version)
        else:
            raise UnavailableException("Cannot guarantee consistency")
```

### AP Systems (Availability + Partition Tolerance)
**Choose availability over consistency during partitions**

**Examples**: Cassandra, DynamoDB, CouchDB, DNS

**Characteristics**:
- Always available for reads and writes
- May return stale or conflicting data
- Suitable for social media, content delivery, analytics

```python
class APSystem:
    def __init__(self):
        self.nodes = []
        self.replication_factor = 3
    
    def write(self, key, value):
        # Write to any available nodes
        successful_writes = 0
        for node in self.nodes[:self.replication_factor]:
            try:
                node.async_write(key, value)
                successful_writes += 1
            except NetworkPartitionException:
                continue
        
        # Accept write if at least one node succeeded
        if successful_writes > 0:
            return True
        else:
            # Try other nodes if primary replicas are down
            return self.write_to_any_available_node(key, value)
    
    def read(self, key):
        # Return data from first available node
        for node in self.nodes:
            try:
                return node.read(key)
            except NetworkPartitionException:
                continue
        
        # If all nodes in partition are down, return cached value
        return self.get_cached_value(key)
```

### CA Systems (Consistency + Availability)
**Cannot tolerate partitions**

**Examples**: Traditional RDBMS (single node), LDAP

**Characteristics**:
- Perfect for single-node systems
- Not truly distributed
- Fails completely during network partitions

```python
class CASystem:
    def __init__(self):
        # Single node system - no partitions possible
        self.data = {}
        self.lock = threading.Lock()
    
    def write(self, key, value):
        with self.lock:
            self.data[key] = {
                'value': value,
                'timestamp': time.time()
            }
        return True
    
    def read(self, key):
        with self.lock:
            return self.data.get(key, {}).get('value')
    
    # Note: This system fails completely if the single node goes down
    # It cannot handle partitions because there's only one node
```

## Real-World Examples

### 1. Amazon DynamoDB (AP System)

```python
class DynamoDBExample:
    """
    DynamoDB chooses Availability and Partition tolerance
    - Eventually consistent reads by default
    - Strongly consistent reads available but may fail during partitions
    - Always accepts writes (with conflict resolution)
    """
    
    def __init__(self):
        self.preference_list = []  # Ordered list of nodes
        self.n = 3  # Replication factor
        self.w = 2  # Write quorum
        self.r = 2  # Read quorum
    
    def put(self, key, value):
        """DynamoDB-style put operation"""
        coordinator = self.get_coordinator(key)
        preference_list = self.get_preference_list(key, self.n)
        
        # Send write to N nodes
        responses = []
        for node in preference_list:
            try:
                response = node.put(key, value, context=self.generate_context())
                responses.append(response)
            except Exception:
                continue
        
        # Return success if W nodes respond
        if len(responses) >= self.w:
            return True
        else:
            # Still return success for availability
            # Will be eventually consistent
            return True
    
    def get(self, key, consistent_read=False):
        """DynamoDB-style get operation"""
        preference_list = self.get_preference_list(key, self.n)
        
        if consistent_read:
            # Strong consistency - may fail during partitions
            responses = []
            for node in preference_list:
                try:
                    responses.append(node.get(key))
                except Exception:
                    continue
            
            if len(responses) >= self.r:
                return self.resolve_conflicts(responses)
            else:
                raise Exception("Cannot guarantee consistency")
        else:
            # Eventually consistent - always available
            for node in preference_list:
                try:
                    return node.get(key)
                except Exception:
                    continue
            return None
```

### 2. MongoDB (CP System)

```python
class MongoDBExample:
    """
    MongoDB chooses Consistency and Partition tolerance
    - Primary-secondary replication
    - Writes only to primary
    - Becomes unavailable if primary is unreachable
    """
    
    def __init__(self):
        self.primary = None
        self.secondaries = []
        self.replica_set_config = {}
    
    def write(self, document):
        """MongoDB-style write operation"""
        if not self.primary or not self.primary.is_reachable():
            # Trigger election if primary is down
            self.elect_new_primary()
        
        if not self.primary:
            # No primary available - reject write for consistency
            raise Exception("No primary available - write rejected")
        
        # Write to primary
        result = self.primary.insert(document)
        
        # Optionally wait for replication based on write concern
        if self.write_concern_majority():
            self.wait_for_majority_replication(document)
        
        return result
    
    def read(self, query, read_preference="primary"):
        """MongoDB-style read operation"""
        if read_preference == "primary":
            if not self.primary or not self.primary.is_reachable():
                raise Exception("Primary not available")
            return self.primary.find(query)
        
        elif read_preference == "secondary":
            for secondary in self.secondaries:
                if secondary.is_reachable():
                    return secondary.find(query)
            raise Exception("No secondary available")
        
        elif read_preference == "primaryPreferred":
            try:
                return self.read(query, "primary")
            except Exception:
                return self.read(query, "secondary")
    
    def elect_new_primary(self):
        """Raft-like election process"""
        # Simplified election logic
        for node in self.secondaries:
            if node.is_reachable() and node.has_majority_support():
                self.primary = node
                self.secondaries.remove(node)
                break
```

### 3. Cassandra (AP System)

```python
class CassandraExample:
    """
    Cassandra chooses Availability and Partition tolerance
    - Tunable consistency
    - Always accepts reads and writes
    - Uses eventual consistency and conflict resolution
    """
    
    def __init__(self):
        self.nodes = []
        self.replication_factor = 3
        self.consistency_level = "QUORUM"
    
    def write(self, key, value, consistency_level=None):
        """Cassandra-style write operation"""
        cl = consistency_level or self.consistency_level
        replicas = self.get_replicas(key)
        
        # Send write to all replicas
        successful_writes = 0
        for replica in replicas:
            try:
                replica.write(key, value, timestamp=time.time())
                successful_writes += 1
            except Exception:
                # Continue even if some nodes are down
                continue
        
        # Check if consistency level is satisfied
        required_acks = self.get_required_acks(cl, len(replicas))
        
        if successful_writes >= required_acks:
            return True
        elif cl in ["ANY", "ONE"] and successful_writes > 0:
            return True
        else:
            # For availability, we might still accept the write
            # and rely on hinted handoff and repair
            self.store_hint(key, value, failed_nodes)
            return True
    
    def read(self, key, consistency_level=None):
        """Cassandra-style read operation"""
        cl = consistency_level or self.consistency_level
        replicas = self.get_replicas(key)
        
        responses = []
        for replica in replicas:
            try:
                response = replica.read(key)
                responses.append(response)
            except Exception:
                continue
        
        required_responses = self.get_required_acks(cl, len(replicas))
        
        if len(responses) >= required_responses:
            # Perform read repair if needed
            return self.resolve_read_conflicts(responses)
        elif len(responses) > 0:
            # Return best available data for availability
            return max(responses, key=lambda x: x.timestamp)
        else:
            return None
    
    def get_required_acks(self, consistency_level, replica_count):
        """Calculate required acknowledgments"""
        if consistency_level == "ONE":
            return 1
        elif consistency_level == "QUORUM":
            return replica_count // 2 + 1
        elif consistency_level == "ALL":
            return replica_count
        elif consistency_level == "ANY":
            return 1
        else:
            return 1
```

## Beyond CAP: PACELC

### PACELC Theorem Extension
**"In case of network Partitioning (P) in a distributed computer system, one has to choose between Availability (A) and Consistency (C) (as per the CAP theorem), but Else (E), even when the system is running normally in the absence of partitions, one has to choose between Latency (L) and Consistency (C)."**

### PACELC Classifications

```python
class PACELCAnalysis:
    """
    Analyze systems using PACELC framework
    """
    
    @staticmethod
    def classify_system(system_name):
        classifications = {
            # During Partition: choose A or C
            # During Normal operation: choose L or C
            
            "DynamoDB": "PA/EL",      # Partition: Availability, Normal: Latency
            "Cassandra": "PA/EL",     # Partition: Availability, Normal: Latency
            "MongoDB": "PC/EC",       # Partition: Consistency, Normal: Consistency
            "HBase": "PC/EC",         # Partition: Consistency, Normal: Consistency
            "CouchDB": "PA/EL",       # Partition: Availability, Normal: Latency
            "BigTable": "PC/EL",      # Partition: Consistency, Normal: Latency
            "VoltDB": "PC/EC",        # Partition: Consistency, Normal: Consistency
        }
        
        return classifications.get(system_name, "Unknown")
    
    @staticmethod
    def explain_classification(classification):
        """Explain what a PACELC classification means"""
        partition_choice, normal_choice = classification.split('/')
        
        explanations = {
            "PA": "During partitions, chooses Availability over Consistency",
            "PC": "During partitions, chooses Consistency over Availability",
            "EL": "During normal operation, chooses Latency over Consistency",
            "EC": "During normal operation, chooses Consistency over Latency"
        }
        
        return {
            "partition": explanations[partition_choice],
            "normal": explanations[normal_choice]
        }
```

## Practical Implications

### 1. System Design Decisions

```python
class SystemDesignDecisions:
    """
    Framework for making CAP-informed design decisions
    """
    
    def __init__(self, requirements):
        self.requirements = requirements
    
    def recommend_cap_choice(self):
        """Recommend CAP trade-off based on requirements"""
        
        # Analyze requirements
        needs_strong_consistency = any([
            self.requirements.get('financial_transactions', False),
            self.requirements.get('inventory_management', False),
            self.requirements.get('user_authentication', False)
        ])
        
        needs_high_availability = any([
            self.requirements.get('global_scale', False),
            self.requirements.get('24_7_uptime', False),
            self.requirements.get('social_media', False)
        ])
        
        partition_tolerance_required = any([
            self.requirements.get('distributed', False),
            self.requirements.get('multi_datacenter', False),
            self.requirements.get('cloud_deployment', False)
        ])
        
        # Make recommendation
        if not partition_tolerance_required:
            return {
                'choice': 'CA',
                'reasoning': 'Single node system can provide both consistency and availability',
                'examples': ['Traditional RDBMS', 'In-memory databases']
            }
        elif needs_strong_consistency:
            return {
                'choice': 'CP',
                'reasoning': 'Strong consistency required, accept availability trade-offs',
                'examples': ['MongoDB', 'HBase', 'Redis Cluster']
            }
        elif needs_high_availability:
            return {
                'choice': 'AP',
                'reasoning': 'High availability required, accept eventual consistency',
                'examples': ['Cassandra', 'DynamoDB', 'CouchDB']
            }
        else:
            return {
                'choice': 'Depends on specific use case',
                'reasoning': 'Need more specific requirements to make recommendation'
            }
```

### 2. Consistency Models

```python
class ConsistencyModels:
    """
    Different consistency models and their trade-offs
    """
    
    @staticmethod
    def strong_consistency():
        """
        Linearizability - strongest consistency model
        All operations appear to execute atomically
        """
        return {
            'guarantees': [
                'All nodes see the same data at the same time',
                'Reads always return the most recent write',
                'Operations appear to execute in real-time order'
            ],
            'trade_offs': [
                'Higher latency',
                'Lower availability during partitions',
                'More complex implementation'
            ],
            'use_cases': [
                'Financial systems',
                'Inventory management',
                'Configuration management'
            ]
        }
    
    @staticmethod
    def eventual_consistency():
        """
        System will become consistent over time
        No guarantees about when
        """
        return {
            'guarantees': [
                'All nodes will eventually converge to same state',
                'No conflicts if no new updates',
                'System remains available during updates'
            ],
            'trade_offs': [
                'Temporary inconsistencies possible',
                'Complex conflict resolution needed',
                'Application must handle stale data'
            ],
            'use_cases': [
                'Social media feeds',
                'Content delivery networks',
                'Analytics systems'
            ]
        }
    
    @staticmethod
    def causal_consistency():
        """
        Causally related operations are seen in same order
        """
        return {
            'guarantees': [
                'Causally related writes seen in same order',
                'Concurrent writes may be seen in different orders',
                'Stronger than eventual, weaker than strong'
            ],
            'implementation': 'Vector clocks or logical timestamps',
            'use_cases': [
                'Collaborative editing',
                'Social media with comments/replies',
                'Distributed version control'
            ]
        }
```

### 3. Partition Handling Strategies

```python
class PartitionHandlingStrategies:
    """
    Different strategies for handling network partitions
    """
    
    def __init__(self):
        self.strategies = {
            'fail_stop': self.fail_stop_strategy,
            'fail_over': self.fail_over_strategy,
            'quorum': self.quorum_strategy,
            'gossip': self.gossip_strategy,
            'vector_clocks': self.vector_clock_strategy
        }
    
    def fail_stop_strategy(self):
        """Stop serving requests during partition"""
        return {
            'description': 'Stop all operations until partition heals',
            'pros': ['Maintains strong consistency', 'Simple to implement'],
            'cons': ['Poor availability', 'User experience degradation'],
            'example': 'Traditional RDBMS cluster'
        }
    
    def fail_over_strategy(self):
        """Failover to backup systems"""
        return {
            'description': 'Switch to backup nodes/datacenters',
            'pros': ['Maintains availability', 'Transparent to users'],
            'cons': ['Potential data loss', 'Complex failover logic'],
            'example': 'Master-slave replication with automatic failover'
        }
    
    def quorum_strategy(self):
        """Use majority consensus"""
        return {
            'description': 'Require majority of nodes for operations',
            'pros': ['Balances consistency and availability', 'Well understood'],
            'cons': ['Requires odd number of nodes', 'Can become unavailable'],
            'example': 'Raft consensus, MongoDB replica sets'
        }
    
    def gossip_strategy(self):
        """Use gossip protocol for eventual consistency"""
        return {
            'description': 'Nodes exchange information periodically',
            'pros': ['High availability', 'Self-healing', 'Scalable'],
            'cons': ['Eventual consistency only', 'Convergence time uncertain'],
            'example': 'Cassandra, Amazon DynamoDB'
        }
    
    def vector_clock_strategy(self):
        """Use vector clocks for conflict resolution"""
        return {
            'description': 'Track causality with vector clocks',
            'pros': ['Handles concurrent updates', 'Preserves causality'],
            'cons': ['Complex implementation', 'Clock size grows'],
            'example': 'Riak, some versions of DynamoDB'
        }
```

## Interview Questions & Answers

### Basic Questions

**Q1: What is the CAP theorem and why is it important?**

**Answer:** The CAP theorem states that in a distributed system, you can only guarantee two out of three properties:
- **Consistency**: All nodes see the same data simultaneously
- **Availability**: System remains operational and responsive
- **Partition Tolerance**: System continues despite network failures

**Importance:**
- Helps architects make informed trade-offs
- Explains why different databases behave differently
- Guides system design decisions based on requirements
- Prevents unrealistic expectations about distributed systems

**Q2: Can you give examples of CP, AP, and CA systems?**

**Answer:**

**CP Systems (Consistency + Partition Tolerance):**
- MongoDB, HBase, Redis Cluster
- Choose consistency over availability during partitions
- May become unavailable but data remains consistent

**AP Systems (Availability + Partition Tolerance):**
- Cassandra, DynamoDB, CouchDB
- Choose availability over consistency during partitions
- Always responsive but may return stale data

**CA Systems (Consistency + Availability):**
- Traditional RDBMS (single node), LDAP
- Cannot handle partitions
- Perfect consistency and availability in non-distributed scenarios

**Q3: What happens during a network partition in CP vs AP systems?**

**Answer:**

**CP Systems during partition:**
```python
# CP System behavior
def handle_partition_cp():
    if not can_reach_majority_of_nodes():
        # Reject writes to maintain consistency
        raise UnavailableException("Cannot guarantee consistency")
    else:
        # Continue normal operations
        return process_request()
```

**AP Systems during partition:**
```python
# AP System behavior  
def handle_partition_ap():
    # Always accept requests
    try:
        return process_request_locally()
    except Exception:
        # Use cached data or default values
        return get_best_available_response()
```

### Intermediate Questions

**Q4: How does eventual consistency work and what are its challenges?**

**Answer:** Eventual consistency guarantees that if no new updates are made, all nodes will eventually converge to the same state.

**How it works:**
1. **Asynchronous replication**: Updates propagate in background
2. **Conflict resolution**: Handle concurrent updates
3. **Anti-entropy**: Periodic synchronization between nodes

**Implementation example:**
```python
class EventuallyConsistentStore:
    def __init__(self):
        self.data = {}
        self.vector_clock = {}
        self.pending_updates = []
    
    def write(self, key, value):
        # Accept write immediately
        timestamp = time.time()
        self.data[key] = {
            'value': value,
            'timestamp': timestamp,
            'node_id': self.node_id
        }
        
        # Queue for async replication
        self.pending_updates.append({
            'key': key,
            'value': value,
            'timestamp': timestamp
        })
        
        return True  # Always succeeds
    
    def read(self, key):
        # Return local value (might be stale)
        return self.data.get(key, {}).get('value')
    
    def sync_with_peers(self):
        # Periodic synchronization
        for update in self.pending_updates:
            self.replicate_to_peers(update)
        self.pending_updates.clear()
```

**Challenges:**
- **Read-your-writes consistency**: User might not see their own updates
- **Monotonic read consistency**: Subsequent reads might return older data
- **Conflict resolution**: Multiple concurrent updates to same data
- **Convergence time**: No guarantees on when consistency is achieved

**Q5: What is the PACELC theorem and how does it extend CAP?**

**Answer:** PACELC extends CAP by considering the trade-off between Latency and Consistency even when there are no partitions.

**PACELC Statement:**
"In case of network Partitioning (P), choose between Availability (A) and Consistency (C), but Else (E), even when the system is running normally, choose between Latency (L) and Consistency (C)."

**Examples:**
```python
# PA/EL System (like Cassandra)
class PAELSystem:
    def read(self, key):
        # During partition: Choose availability
        # During normal operation: Choose latency (return immediately)
        return self.get_from_nearest_node(key)

# PC/EC System (like MongoDB)  
class PCECSystem:
    def read(self, key):
        # During partition: Choose consistency (may fail)
        # During normal operation: Choose consistency (wait for primary)
        return self.get_from_primary_with_confirmation(key)
```

**Q6: How do you handle split-brain scenarios in distributed systems?**

**Answer:** Split-brain occurs when network partition causes multiple nodes to think they're the primary/leader.

**Prevention strategies:**

1. **Quorum-based approach:**
```python
class QuorumBasedLeader:
    def __init__(self, nodes):
        self.nodes = nodes
        self.quorum_size = len(nodes) // 2 + 1
    
    def can_be_leader(self):
        reachable_nodes = sum(1 for node in self.nodes if node.is_reachable())
        return reachable_nodes >= self.quorum_size
    
    def perform_operation(self):
        if not self.can_be_leader():
            raise Exception("Cannot perform operation - no quorum")
        return self.execute_operation()
```

2. **Witness/Tiebreaker nodes:**
```python
class WitnessBasedSystem:
    def __init__(self, primary, secondary, witness):
        self.primary = primary
        self.secondary = secondary
        self.witness = witness
    
    def determine_leader(self):
        if self.primary.can_reach(self.witness):
            return self.primary
        elif self.secondary.can_reach(self.witness):
            return self.secondary
        else:
            # No one can reach witness - system unavailable
            return None
```

3. **Fencing mechanisms:**
```python
class FencingMechanism:
    def __init__(self):
        self.shared_storage = SharedStorage()
        self.generation_number = 0
    
    def acquire_leadership(self):
        # Atomic increment of generation number
        new_generation = self.shared_storage.increment_generation()
        self.generation_number = new_generation
        return True
    
    def perform_operation(self, operation):
        # Check if still the leader
        current_generation = self.shared_storage.get_generation()
        if current_generation != self.generation_number:
            raise Exception("Leadership lost - operation rejected")
        
        return operation.execute()
```

### Advanced Questions

**Q7: Design a distributed system that needs to handle both financial transactions and social media feeds. How would you apply CAP theorem?**

**Answer:** This requires a hybrid approach with different subsystems making different CAP trade-offs:

```python
class HybridSystem:
    def __init__(self):
        # CP subsystem for financial transactions
        self.financial_db = CPSystem(
            consistency_level="STRONG",
            replication_factor=3,
            quorum_size=2
        )
        
        # AP subsystem for social media feeds
        self.social_db = APSystem(
            consistency_level="EVENTUAL",
            replication_factor=5,
            availability_target=99.99
        )
        
        # Coordination layer
        self.transaction_coordinator = TransactionCoordinator()
    
    def process_financial_transaction(self, transaction):
        """Handle financial transaction with strong consistency"""
        try:
            # Use CP system - may fail during partitions
            return self.financial_db.execute_transaction(transaction)
        except UnavailableException:
            # Queue for later processing when partition heals
            self.queue_for_retry(transaction)
            raise Exception("Transaction temporarily unavailable")
    
    def post_social_update(self, user_id, content):
        """Handle social media post with high availability"""
        # Use AP system - always succeeds
        post_id = self.social_db.create_post(user_id, content)
        
        # Async propagation to followers
        self.async_propagate_to_followers(user_id, post_id)
        
        return post_id
    
    def get_user_feed(self, user_id):
        """Get social media feed - eventual consistency OK"""
        return self.social_db.get_feed(user_id)
```

**Design decisions:**
- **Financial transactions**: CP system (MongoDB/PostgreSQL cluster)
- **Social feeds**: AP system (Cassandra/DynamoDB)
- **User profiles**: Hybrid with caching layer
- **Cross-system consistency**: Event sourcing and SAGA patterns

**Q8: How would you implement a globally distributed system that needs to comply with data sovereignty laws?**

**Answer:** Use regional partitioning with different CAP choices per region:

```python
class GloballyDistributedSystem:
    def __init__(self):
        self.regions = {
            'us-east': Region('us-east', data_residency='US'),
            'eu-west': Region('eu-west', data_residency='EU'),
            'asia-pacific': Region('asia-pacific', data_residency='APAC')
        }
        self.global_coordinator = GlobalCoordinator()
    
    def store_user_data(self, user_id, data, user_region):
        """Store data in appropriate region"""
        target_region = self.regions[user_region]
        
        # Store in regional CP system for compliance
        result = target_region.store_with_strong_consistency(user_id, data)
        
        # Replicate metadata globally (AP system)
        self.replicate_metadata_globally(user_id, user_region)
        
        return result
    
    def get_user_data(self, user_id, requesting_region):
        """Get user data respecting data sovereignty"""
        user_region = self.get_user_region(user_id)
        
        if user_region == requesting_region:
            # Local access - use CP system
            return self.regions[user_region].get_local_data(user_id)
        else:
            # Cross-region access - check compliance
            if self.is_cross_region_access_allowed(user_region, requesting_region):
                return self.regions[user_region].get_data_via_secure_channel(user_id)
            else:
                raise ComplianceException("Cross-region access not allowed")
```

**Q9: Explain how vector clocks help with conflict resolution in AP systems.**

**Answer:** Vector clocks track causality between events in distributed systems:

```python
class VectorClock:
    def __init__(self, node_id, nodes):
        self.node_id = node_id
        self.clock = {node: 0 for node in nodes}
    
    def tick(self):
        """Increment local clock"""
        self.clock[self.node_id] += 1
        return self.clock.copy()
    
    def update(self, other_clock):
        """Update clock when receiving message"""
        for node in self.clock:
            self.clock[node] = max(self.clock[node], other_clock.get(node, 0))
        self.tick()  # Increment local clock
    
    def compare(self, other_clock):
        """Compare two vector clocks"""
        self_greater = any(self.clock[node] > other_clock.get(node, 0) 
                          for node in self.clock)
        other_greater = any(other_clock.get(node, 0) > self.clock[node] 
                           for node in self.clock)
        
        if self_greater and not other_greater:
            return "after"  # self happened after other
        elif other_greater and not self_greater:
            return "before"  # self happened before other
        elif not self_greater and not other_greater:
            return "equal"  # concurrent events
        else:
            return "concurrent"  # concurrent events

class APSystemWithVectorClocks:
    def __init__(self, node_id, nodes):
        self.node_id = node_id
        self.vector_clock = VectorClock(node_id, nodes)
        self.data = {}
    
    def write(self, key, value):
        """Write with vector clock"""
        timestamp = self.vector_clock.tick()
        
        self.data[key] = {
            'value': value,
            'vector_clock': timestamp,
            'node_id': self.node_id
        }
        
        # Replicate to other nodes
        self.replicate_to_peers(key, value, timestamp)
        
        return True
    
    def resolve_conflicts(self, key, conflicting_values):
        """Resolve conflicts using vector clocks"""
        if len(conflicting_values) == 1:
            return conflicting_values[0]
        
        # Find values that are not superseded by others
        candidates = []
        for value in conflicting_values:
            is_superseded = False
            for other_value in conflicting_values:
                if value != other_value:
                    comparison = self.compare_vector_clocks(
                        value['vector_clock'], 
                        other_value['vector_clock']
                    )
                    if comparison == "before":
                        is_superseded = True
                        break
            
            if not is_superseded:
                candidates.append(value)
        
        if len(candidates) == 1:
            return candidates[0]
        else:
            # Multiple concurrent updates - application-specific resolution
            return self.application_specific_resolution(candidates)
```

**Q10: How do you test CAP theorem trade-offs in a distributed system?**

**Answer:** Comprehensive testing strategy for CAP properties:

```python
class CAPTester:
    def __init__(self, system_under_test):
        self.system = system_under_test
        self.network_partitioner = NetworkPartitioner()
        self.consistency_checker = ConsistencyChecker()
        self.availability_monitor = AvailabilityMonitor()
    
    def test_consistency_during_partition(self):
        """Test if system maintains consistency during partition"""
        # Write data before partition
        self.system.write("key1", "value1")
        
        # Create network partition
        self.network_partitioner.partition_nodes([1, 2], [3, 4, 5])
        
        # Try to write to both sides
        try:
            partition1_result = self.system.write_to_partition([1, 2], "key1", "value2")
            partition2_result = self.system.write_to_partition([3, 4, 5], "key1", "value3")
            
            # Check if both writes succeeded (AP system)
            if partition1_result and partition2_result:
                print("System chose Availability over Consistency")
                return "AP"
            else:
                print("System chose Consistency over Availability")
                return "CP"
                
        except Exception as e:
            print(f"System unavailable during partition: {e}")
            return "CP"
    
    def test_availability_during_partition(self):
        """Test system availability during partition"""
        # Create partition
        self.network_partitioner.partition_nodes([1, 2], [3, 4, 5])
        
        # Monitor availability
        availability_results = []
        
        for i in range(100):  # Test 100 requests
            try:
                result = self.system.read("key1")
                availability_results.append(True)
            except Exception:
                availability_results.append(False)
            
            time.sleep(0.1)
        
        availability_percentage = sum(availability_results) / len(availability_results)
        
        if availability_percentage > 0.95:  # 95% availability
            print("System maintained high availability")
            return "Available"
        else:
            print(f"System availability dropped to {availability_percentage:.2%}")
            return "Unavailable"
    
    def test_partition_tolerance(self):
        """Test how system handles various partition scenarios"""
        scenarios = [
            {"partition": [[1], [2, 3, 4, 5]], "description": "Single node isolated"},
            {"partition": [[1, 2], [3, 4, 5]], "description": "Minority partition"},
            {"partition": [[1, 2, 3], [4, 5]], "description": "Majority partition"},
            {"partition": [[1], [2], [3], [4], [5]], "description": "Complete partition"}
        ]
        
        results = {}
        
        for scenario in scenarios:
            print(f"Testing: {scenario['description']}")
            
            # Create partition
            self.network_partitioner.create_partition(scenario['partition'])
            
            # Test system behavior
            consistency_result = self.test_consistency_during_partition()
            availability_result = self.test_availability_during_partition()
            
            results[scenario['description']] = {
                'consistency': consistency_result,
                'availability': availability_result
            }
            
            # Heal partition
            self.network_partitioner.heal_partition()
            time.sleep(5)  # Allow system to recover
        
        return results
    
    def generate_cap_report(self):
        """Generate comprehensive CAP analysis report"""
        test_results = self.test_partition_tolerance()
        
        report = {
            'system_classification': self.classify_system(test_results),
            'detailed_results': test_results,
            'recommendations': self.generate_recommendations(test_results)
        }
        
        return report
    
    def classify_system(self, test_results):
        """Classify system based on test results"""
        consistency_count = sum(1 for result in test_results.values() 
                              if result['consistency'] == 'CP')
        availability_count = sum(1 for result in test_results.values() 
                               if result['availability'] == 'Available')
        
        if consistency_count > availability_count:
            return "CP System"
        elif availability_count > consistency_count:
            return "AP System"
        else:
            return "Hybrid System"
```

## Best Practices

1. **Understand your requirements**: Analyze whether you need strong consistency or high availability
2. **Design for partition tolerance**: Assume network failures will happen
3. **Use appropriate consistency models**: Choose the weakest consistency model that meets your needs
4. **Implement proper monitoring**: Track consistency, availability, and partition events
5. **Test partition scenarios**: Regularly test how your system behaves during network failures
6. **Document trade-offs**: Make CAP decisions explicit in your architecture documentation
7. **Consider PACELC**: Think about latency vs consistency trade-offs during normal operation

## Further Reading

- [Brewer's CAP Theorem](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)
- [PACELC Theorem](https://en.wikipedia.org/wiki/PACELC_theorem)
- [Consistency Models in Distributed Systems](https://jepsen.io/consistency)
- [Amazon's Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Google's Spanner Paper](https://research.google/pubs/pub39966/)
