# Consensus Algorithms

Consensus algorithms are fundamental protocols in distributed systems that enable multiple nodes to agree on a single value or state, even in the presence of failures. They are essential for maintaining consistency and coordination in distributed environments.

## Table of Contents
- [What is Consensus?](#what-is-consensus)
- [The Consensus Problem](#the-consensus-problem)
- [Major Consensus Algorithms](#major-consensus-algorithms)
- [Practical Applications](#practical-applications)
- [Performance Comparison](#performance-comparison)
- [Implementation Considerations](#implementation-considerations)
- [Interview Questions & Answers](#interview-questions--answers)

## What is Consensus?

### Definition
Consensus is the process by which distributed nodes agree on a single value or decision, ensuring that all correct nodes reach the same conclusion despite potential failures or network issues.

### Key Properties
1. **Agreement**: All correct nodes decide on the same value
2. **Validity**: The decided value must be proposed by some node
3. **Termination**: All correct nodes eventually decide
4. **Integrity**: Each node decides at most once

### Why Consensus Matters
- **Leader Election**: Choosing a single leader among distributed nodes
- **State Machine Replication**: Keeping replicas synchronized
- **Atomic Broadcast**: Ensuring all nodes receive messages in the same order
- **Configuration Management**: Coordinating system-wide configuration changes

## The Consensus Problem

### FLP Impossibility Result
The Fischer-Lynch-Paterson theorem proves that in an asynchronous network with even one faulty node, no deterministic consensus algorithm can guarantee termination.

**Implications**:
- Perfect consensus is impossible in asynchronous systems
- Practical algorithms use timeouts and failure detectors
- Trade-offs between safety and liveness are necessary

### Byzantine vs Non-Byzantine Failures

**Non-Byzantine (Crash) Failures**:
- Nodes either work correctly or stop completely
- No malicious behavior
- Easier to handle and detect

**Byzantine Failures**:
- Nodes can behave arbitrarily or maliciously
- Can send conflicting messages
- Requires more complex algorithms

## Major Consensus Algorithms

### 1. Raft Consensus Algorithm

Raft is designed to be understandable and provides strong consistency through leader-based replication.

#### Core Concepts

```python
import time
import random
from enum import Enum
from typing import List, Dict, Optional

class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

class LogEntry:
    def __init__(self, term: int, command: str, index: int):
        self.term = term
        self.command = command
        self.index = index

class RaftNode:
    def __init__(self, node_id: str, peers: List[str]):
        self.node_id = node_id
        self.peers = peers
        self.state = NodeState.FOLLOWER
        
        # Persistent state
        self.current_term = 0
        self.voted_for = None
        self.log = []  # List of LogEntry
        
        # Volatile state
        self.commit_index = 0
        self.last_applied = 0
        
        # Leader state
        self.next_index = {}  # For each server, index of next log entry to send
        self.match_index = {}  # For each server, index of highest log entry known to be replicated
        
        # Timing
        self.last_heartbeat = time.time()
        self.election_timeout = random.uniform(150, 300)  # milliseconds
        self.heartbeat_interval = 50  # milliseconds
    
    def start_election(self):
        """Start leader election process"""
        self.state = NodeState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        self.last_heartbeat = time.time()
        
        # Vote for self
        votes_received = 1
        
        # Request votes from all peers
        for peer in self.peers:
            if self.request_vote(peer):
                votes_received += 1
        
        # Check if won election
        if votes_received > len(self.peers) // 2:
            self.become_leader()
        else:
            self.state = NodeState.FOLLOWER
    
    def request_vote(self, peer_id: str) -> bool:
        """Request vote from a peer"""
        # In real implementation, this would be a network call
        request = {
            'term': self.current_term,
            'candidate_id': self.node_id,
            'last_log_index': len(self.log) - 1 if self.log else -1,
            'last_log_term': self.log[-1].term if self.log else 0
        }
        
        # Simulate network call
        response = self.send_vote_request(peer_id, request)
        return response.get('vote_granted', False)
    
    def handle_vote_request(self, request: Dict) -> Dict:
        """Handle incoming vote request"""
        term = request['term']
        candidate_id = request['candidate_id']
        last_log_index = request['last_log_index']
        last_log_term = request['last_log_term']
        
        # Update term if necessary
        if term > self.current_term:
            self.current_term = term
            self.voted_for = None
            self.state = NodeState.FOLLOWER
        
        # Check if can vote for candidate
        vote_granted = False
        
        if (term == self.current_term and 
            (self.voted_for is None or self.voted_for == candidate_id) and
            self.is_log_up_to_date(last_log_index, last_log_term)):
            
            self.voted_for = candidate_id
            vote_granted = True
            self.last_heartbeat = time.time()
        
        return {
            'term': self.current_term,
            'vote_granted': vote_granted
        }
    
    def is_log_up_to_date(self, last_log_index: int, last_log_term: int) -> bool:
        """Check if candidate's log is at least as up-to-date as ours"""
        if not self.log:
            return True
        
        our_last_term = self.log[-1].term
        our_last_index = len(self.log) - 1
        
        if last_log_term > our_last_term:
            return True
        elif last_log_term == our_last_term:
            return last_log_index >= our_last_index
        else:
            return False
    
    def become_leader(self):
        """Transition to leader state"""
        self.state = NodeState.LEADER
        
        # Initialize leader state
        for peer in self.peers:
            self.next_index[peer] = len(self.log)
            self.match_index[peer] = 0
        
        # Send initial heartbeats
        self.send_heartbeats()
    
    def send_heartbeats(self):
        """Send heartbeat messages to all followers"""
        for peer in self.peers:
            self.send_append_entries(peer, heartbeat=True)
    
    def send_append_entries(self, peer_id: str, heartbeat: bool = False):
        """Send append entries RPC to a peer"""
        prev_log_index = self.next_index[peer_id] - 1
        prev_log_term = 0
        
        if prev_log_index >= 0 and prev_log_index < len(self.log):
            prev_log_term = self.log[prev_log_index].term
        
        entries = []
        if not heartbeat and self.next_index[peer_id] < len(self.log):
            entries = self.log[self.next_index[peer_id]:]
        
        request = {
            'term': self.current_term,
            'leader_id': self.node_id,
            'prev_log_index': prev_log_index,
            'prev_log_term': prev_log_term,
            'entries': entries,
            'leader_commit': self.commit_index
        }
        
        response = self.send_append_entries_rpc(peer_id, request)
        self.handle_append_entries_response(peer_id, response)
    
    def handle_append_entries_response(self, peer_id: str, response: Dict):
        """Handle response to append entries RPC"""
        if response['term'] > self.current_term:
            self.current_term = response['term']
            self.state = NodeState.FOLLOWER
            self.voted_for = None
            return
        
        if response['success']:
            # Update next_index and match_index for follower
            self.match_index[peer_id] = response.get('match_index', 0)
            self.next_index[peer_id] = self.match_index[peer_id] + 1
            
            # Update commit index if majority has replicated
            self.update_commit_index()
        else:
            # Decrement next_index and retry
            self.next_index[peer_id] = max(0, self.next_index[peer_id] - 1)
    
    def update_commit_index(self):
        """Update commit index based on majority replication"""
        for n in range(self.commit_index + 1, len(self.log)):
            if self.log[n].term == self.current_term:
                # Count how many servers have replicated this entry
                count = 1  # Count self
                for peer in self.peers:
                    if self.match_index[peer] >= n:
                        count += 1
                
                # If majority has replicated, commit
                if count > len(self.peers) // 2:
                    self.commit_index = n
                else:
                    break
    
    def append_entry(self, command: str) -> bool:
        """Append a new entry to the log (leader only)"""
        if self.state != NodeState.LEADER:
            return False
        
        entry = LogEntry(self.current_term, command, len(self.log))
        self.log.append(entry)
        
        # Replicate to followers
        for peer in self.peers:
            self.send_append_entries(peer)
        
        return True
```

#### Raft Properties

**Leader Election**:
- Only one leader per term
- Leaders send periodic heartbeats
- Followers become candidates if no heartbeat received

**Log Replication**:
- All changes go through the leader
- Leader replicates entries to followers
- Entries are committed when replicated to majority

**Safety Properties**:
- **Election Safety**: At most one leader per term
- **Leader Append-Only**: Leaders never overwrite or delete entries
- **Log Matching**: If two logs contain an entry with same index and term, they are identical up to that point
- **Leader Completeness**: If a log entry is committed in a given term, it will be present in all higher-numbered terms
- **State Machine Safety**: If a server has applied a log entry at a given index, no other server will apply a different log entry for the same index

### 2. PBFT (Practical Byzantine Fault Tolerance)

PBFT handles Byzantine failures and can tolerate up to f faulty nodes out of 3f+1 total nodes.

```python
import hashlib
from typing import Set, Dict, List
from dataclasses import dataclass

@dataclass
class Message:
    view: int
    sequence: int
    digest: str
    node_id: str
    message_type: str
    content: str = ""

class PBFTNode:
    def __init__(self, node_id: str, total_nodes: int):
        self.node_id = node_id
        self.total_nodes = total_nodes
        self.f = (total_nodes - 1) // 3  # Maximum faulty nodes
        
        self.view = 0
        self.sequence = 0
        self.primary_id = "0"  # Primary for current view
        
        # Message logs
        self.message_log = []
        self.prepare_log = {}  # sequence -> set of prepare messages
        self.commit_log = {}   # sequence -> set of commit messages
        
        # State
        self.prepared = {}     # sequence -> bool
        self.committed = {}    # sequence -> bool
        self.executed = {}     # sequence -> bool
    
    def is_primary(self) -> bool:
        """Check if this node is the primary for current view"""
        return self.node_id == self.primary_id
    
    def client_request(self, request: str) -> bool:
        """Handle client request (primary only)"""
        if not self.is_primary():
            return False
        
        # Create pre-prepare message
        digest = hashlib.sha256(request.encode()).hexdigest()
        message = Message(
            view=self.view,
            sequence=self.sequence,
            digest=digest,
            node_id=self.node_id,
            message_type="pre-prepare",
            content=request
        )
        
        # Send pre-prepare to all backups
        self.broadcast_message(message)
        self.sequence += 1
        
        return True
    
    def handle_pre_prepare(self, message: Message):
        """Handle pre-prepare message from primary"""
        # Verify message is from current primary
        if message.node_id != self.primary_id:
            return
        
        # Verify view and sequence
        if message.view != self.view:
            return
        
        # Verify digest
        expected_digest = hashlib.sha256(message.content.encode()).hexdigest()
        if message.digest != expected_digest:
            return
        
        # Accept pre-prepare and send prepare
        self.message_log.append(message)
        
        prepare_msg = Message(
            view=self.view,
            sequence=message.sequence,
            digest=message.digest,
            node_id=self.node_id,
            message_type="prepare"
        )
        
        self.broadcast_message(prepare_msg)
    
    def handle_prepare(self, message: Message):
        """Handle prepare message from backup"""
        seq = message.sequence
        
        if seq not in self.prepare_log:
            self.prepare_log[seq] = set()
        
        self.prepare_log[seq].add(message.node_id)
        
        # Check if we have enough prepare messages (2f)
        if len(self.prepare_log[seq]) >= 2 * self.f and seq not in self.prepared:
            self.prepared[seq] = True
            
            # Send commit message
            commit_msg = Message(
                view=self.view,
                sequence=seq,
                digest=message.digest,
                node_id=self.node_id,
                message_type="commit"
            )
            
            self.broadcast_message(commit_msg)
    
    def handle_commit(self, message: Message):
        """Handle commit message"""
        seq = message.sequence
        
        if seq not in self.commit_log:
            self.commit_log[seq] = set()
        
        self.commit_log[seq].add(message.node_id)
        
        # Check if we have enough commit messages (2f+1)
        if (len(self.commit_log[seq]) >= 2 * self.f + 1 and 
            seq not in self.committed and 
            seq in self.prepared):
            
            self.committed[seq] = True
            self.execute_request(seq)
    
    def execute_request(self, sequence: int):
        """Execute the committed request"""
        if sequence in self.executed:
            return
        
        # Find the request with this sequence number
        for msg in self.message_log:
            if msg.sequence == sequence and msg.message_type == "pre-prepare":
                # Execute the request
                print(f"Node {self.node_id} executing: {msg.content}")
                self.executed[sequence] = True
                break
    
    def broadcast_message(self, message: Message):
        """Broadcast message to all other nodes"""
        # In real implementation, this would send over network
        for node_id in range(self.total_nodes):
            if str(node_id) != self.node_id:
                self.send_message(str(node_id), message)
    
    def send_message(self, target_node: str, message: Message):
        """Send message to specific node"""
        # Simulate network send
        pass
    
    def view_change(self, new_view: int):
        """Initiate view change protocol"""
        self.view = new_view
        self.primary_id = str(new_view % self.total_nodes)
        
        # Send view-change message
        view_change_msg = Message(
            view=new_view,
            sequence=0,
            digest="",
            node_id=self.node_id,
            message_type="view-change"
        )
        
        self.broadcast_message(view_change_msg)
```

### 3. Paxos Algorithm

Paxos is a family of protocols for solving consensus in a network of unreliable processors.

```python
from typing import Optional, Tuple
import time

class PaxosProposer:
    def __init__(self, proposer_id: str, acceptors: List[str]):
        self.proposer_id = proposer_id
        self.acceptors = acceptors
        self.proposal_number = 0
    
    def propose(self, value: str) -> bool:
        """Propose a value using Paxos protocol"""
        self.proposal_number += 1
        proposal_id = f"{self.proposal_number}-{self.proposer_id}"
        
        # Phase 1: Prepare
        promises = self.send_prepare(proposal_id)
        
        if len(promises) <= len(self.acceptors) // 2:
            return False  # Not enough promises
        
        # Find highest proposal number from promises
        highest_proposal = None
        proposed_value = value
        
        for promise in promises:
            if promise.get('accepted_proposal'):
                if (highest_proposal is None or 
                    promise['accepted_proposal']['id'] > highest_proposal):
                    highest_proposal = promise['accepted_proposal']['id']
                    proposed_value = promise['accepted_proposal']['value']
        
        # Phase 2: Accept
        accepts = self.send_accept(proposal_id, proposed_value)
        
        if len(accepts) > len(self.acceptors) // 2:
            # Majority accepted - value is chosen
            self.send_commit(proposed_value)
            return True
        
        return False
    
    def send_prepare(self, proposal_id: str) -> List[Dict]:
        """Send prepare messages to acceptors"""
        promises = []
        
        for acceptor_id in self.acceptors:
            response = self.send_prepare_to_acceptor(acceptor_id, proposal_id)
            if response.get('promise'):
                promises.append(response)
        
        return promises
    
    def send_accept(self, proposal_id: str, value: str) -> List[Dict]:
        """Send accept messages to acceptors"""
        accepts = []
        
        for acceptor_id in self.acceptors:
            response = self.send_accept_to_acceptor(acceptor_id, proposal_id, value)
            if response.get('accepted'):
                accepts.append(response)
        
        return accepts
    
    def send_prepare_to_acceptor(self, acceptor_id: str, proposal_id: str) -> Dict:
        """Send prepare message to specific acceptor"""
        # Simulate network call
        return {'promise': True, 'accepted_proposal': None}
    
    def send_accept_to_acceptor(self, acceptor_id: str, proposal_id: str, value: str) -> Dict:
        """Send accept message to specific acceptor"""
        # Simulate network call
        return {'accepted': True}
    
    def send_commit(self, value: str):
        """Notify learners of chosen value"""
        for acceptor_id in self.acceptors:
            self.notify_learner(acceptor_id, value)

class PaxosAcceptor:
    def __init__(self, acceptor_id: str):
        self.acceptor_id = acceptor_id
        self.promised_proposal = None
        self.accepted_proposal = None
        self.accepted_value = None
    
    def handle_prepare(self, proposal_id: str) -> Dict:
        """Handle prepare message from proposer"""
        if (self.promised_proposal is None or 
            proposal_id > self.promised_proposal):
            
            self.promised_proposal = proposal_id
            
            return {
                'promise': True,
                'accepted_proposal': {
                    'id': self.accepted_proposal,
                    'value': self.accepted_value
                } if self.accepted_proposal else None
            }
        
        return {'promise': False}
    
    def handle_accept(self, proposal_id: str, value: str) -> Dict:
        """Handle accept message from proposer"""
        if (self.promised_proposal is None or 
            proposal_id >= self.promised_proposal):
            
            self.promised_proposal = proposal_id
            self.accepted_proposal = proposal_id
            self.accepted_value = value
            
            return {'accepted': True}
        
        return {'accepted': False}

class MultiPaxos:
    """Multi-Paxos for multiple consensus instances"""
    
    def __init__(self, node_id: str, nodes: List[str]):
        self.node_id = node_id
        self.nodes = nodes
        self.leader = None
        self.instances = {}  # instance_id -> PaxosInstance
        self.next_instance = 0
    
    def propose_value(self, value: str) -> bool:
        """Propose value in next available instance"""
        if self.leader != self.node_id:
            return False  # Only leader can propose
        
        instance_id = self.next_instance
        self.next_instance += 1
        
        # Create Paxos instance
        proposer = PaxosProposer(self.node_id, self.nodes)
        success = proposer.propose(value)
        
        if success:
            self.instances[instance_id] = {
                'value': value,
                'status': 'chosen'
            }
        
        return success
    
    def elect_leader(self):
        """Simple leader election for Multi-Paxos"""
        # Use highest node ID as leader (simplified)
        self.leader = max(self.nodes)
```

### 4. Proof of Work (Blockchain Consensus)

Used in Bitcoin and other cryptocurrencies for distributed consensus without a central authority.

```python
import hashlib
import time
import json
from typing import List, Optional

class Block:
    def __init__(self, index: int, transactions: List[str], 
                 previous_hash: str, timestamp: float = None):
        self.index = index
        self.transactions = transactions
        self.previous_hash = previous_hash
        self.timestamp = timestamp or time.time()
        self.nonce = 0
        self.hash = None
    
    def calculate_hash(self) -> str:
        """Calculate hash of the block"""
        block_string = json.dumps({
            'index': self.index,
            'transactions': self.transactions,
            'previous_hash': self.previous_hash,
            'timestamp': self.timestamp,
            'nonce': self.nonce
        }, sort_keys=True)
        
        return hashlib.sha256(block_string.encode()).hexdigest()
    
    def mine_block(self, difficulty: int):
        """Mine the block by finding valid nonce"""
        target = "0" * difficulty
        
        while self.hash is None or not self.hash.startswith(target):
            self.nonce += 1
            self.hash = self.calculate_hash()
        
        print(f"Block mined: {self.hash}")

class ProofOfWorkBlockchain:
    def __init__(self, difficulty: int = 4):
        self.chain = [self.create_genesis_block()]
        self.difficulty = difficulty
        self.pending_transactions = []
        self.mining_reward = 100
    
    def create_genesis_block(self) -> Block:
        """Create the first block in the chain"""
        genesis = Block(0, ["Genesis Block"], "0")
        genesis.mine_block(self.difficulty)
        return genesis
    
    def get_latest_block(self) -> Block:
        """Get the most recent block"""
        return self.chain[-1]
    
    def add_transaction(self, transaction: str):
        """Add transaction to pending pool"""
        self.pending_transactions.append(transaction)
    
    def mine_pending_transactions(self, mining_reward_address: str) -> Block:
        """Mine a new block with pending transactions"""
        # Add mining reward transaction
        reward_transaction = f"Mining reward to {mining_reward_address}: {self.mining_reward}"
        transactions = self.pending_transactions + [reward_transaction]
        
        # Create new block
        block = Block(
            index=len(self.chain),
            transactions=transactions,
            previous_hash=self.get_latest_block().hash
        )
        
        # Mine the block
        block.mine_block(self.difficulty)
        
        # Add to chain and clear pending transactions
        self.chain.append(block)
        self.pending_transactions = []
        
        return block
    
    def is_chain_valid(self) -> bool:
        """Validate the entire blockchain"""
        for i in range(1, len(self.chain)):
            current_block = self.chain[i]
            previous_block = self.chain[i - 1]
            
            # Check if current block's hash is valid
            if current_block.hash != current_block.calculate_hash():
                return False
            
            # Check if current block points to previous block
            if current_block.previous_hash != previous_block.hash:
                return False
        
        return True
    
    def resolve_conflicts(self, other_chains: List['ProofOfWorkBlockchain']) -> bool:
        """Resolve conflicts by adopting longest valid chain"""
        longest_chain = None
        max_length = len(self.chain)
        
        for chain in other_chains:
            if len(chain.chain) > max_length and chain.is_chain_valid():
                max_length = len(chain.chain)
                longest_chain = chain.chain
        
        if longest_chain:
            self.chain = longest_chain
            return True
        
        return False

# Usage example
blockchain = ProofOfWorkBlockchain(difficulty=2)

# Add some transactions
blockchain.add_transaction("Alice sends 10 coins to Bob")
blockchain.add_transaction("Bob sends 5 coins to Charlie")

# Mine a block
print("Mining block...")
block = blockchain.mine_pending_transactions("Miner1")

print(f"Blockchain valid: {blockchain.is_chain_valid()}")
```

## Practical Applications

### 1. Distributed Database Replication

```python
class DistributedDatabase:
    def __init__(self, node_id: str, nodes: List[str]):
        self.node_id = node_id
        self.raft_node = RaftNode(node_id, nodes)
        self.state_machine = {}  # Key-value store
        self.last_applied_index = 0
    
    def put(self, key: str, value: str) -> bool:
        """Put key-value pair (goes through consensus)"""
        command = f"PUT {key} {value}"
        return self.raft_node.append_entry(command)
    
    def get(self, key: str) -> Optional[str]:
        """Get value for key (local read)"""
        return self.state_machine.get(key)
    
    def apply_committed_entries(self):
        """Apply committed log entries to state machine"""
        while self.last_applied_index < self.raft_node.commit_index:
            self.last_applied_index += 1
            entry = self.raft_node.log[self.last_applied_index]
            self.apply_command(entry.command)
    
    def apply_command(self, command: str):
        """Apply a single command to state machine"""
        parts = command.split(' ', 2)
        if len(parts) == 3 and parts[0] == "PUT":
            key, value = parts[1], parts[2]
            self.state_machine[key] = value
            print(f"Applied: {command}")
```

### 2. Configuration Management Service

```python
class ConfigurationService:
    def __init__(self, node_id: str, nodes: List[str]):
        self.node_id = node_id
        self.consensus = RaftNode(node_id, nodes)
        self.config = {}
        self.watchers = {}  # key -> list of callbacks
    
    def set_config(self, key: str, value: str) -> bool:
        """Set configuration value"""
        command = f"SET_CONFIG {key} {value}"
        success = self.consensus.append_entry(command)
        
        if success:
            # Notify watchers when config changes
            self.notify_watchers(key, value)
        
        return success
    
    def get_config(self, key: str) -> Optional[str]:
        """Get configuration value"""
        return self.config.get(key)
    
    def watch_config(self, key: str, callback):
        """Watch for changes to a configuration key"""
        if key not in self.watchers:
            self.watchers[key] = []
        self.watchers[key].append(callback)
    
    def notify_watchers(self, key: str, value: str):
        """Notify watchers of configuration changes"""
        if key in self.watchers:
            for callback in self.watchers[key]:
                callback(key, value)
    
    def apply_config_change(self, command: str):
        """Apply configuration change from consensus"""
        parts = command.split(' ', 2)
        if len(parts) == 3 and parts[0] == "SET_CONFIG":
            key, value = parts[1], parts[2]
            old_value = self.config.get(key)
            self.config[key] = value
            
            if old_value != value:
                self.notify_watchers(key, value)
```

### 3. Leader Election Service

```python
class LeaderElectionService:
    def __init__(self, node_id: str, nodes: List[str]):
        self.node_id = node_id
        self.nodes = nodes
        self.current_leader = None
        self.leader_callbacks = []
        
        # Use Raft for leader election
        self.raft = RaftNode(node_id, nodes)
    
    def register_leader_callback(self, callback):
        """Register callback for leader changes"""
        self.leader_callbacks.append(callback)
    
    def get_current_leader(self) -> Optional[str]:
        """Get current leader"""
        if self.raft.state == NodeState.LEADER:
            return self.node_id
        else:
            # In real implementation, would track leader from heartbeats
            return self.current_leader
    
    def is_leader(self) -> bool:
        """Check if this node is the leader"""
        return self.raft.state == NodeState.LEADER
    
    def step_down(self):
        """Step down from leadership"""
        if self.is_leader():
            self.raft.state = NodeState.FOLLOWER
            self.notify_leader_change(None)
    
    def notify_leader_change(self, new_leader: str):
        """Notify callbacks of leader change"""
        if new_leader != self.current_leader:
            self.current_leader = new_leader
            for callback in self.leader_callbacks:
                callback(new_leader)
```

## Performance Comparison

### Throughput and Latency

| Algorithm | Throughput | Latency | Fault Tolerance | Network Overhead |
|-----------|------------|---------|-----------------|------------------|
| **Raft** | High | Low | f+1 nodes survive f failures | Moderate |
| **PBFT** | Medium | Medium | 3f+1 nodes survive f Byzantine failures | High |
| **Paxos** | Medium | Medium | f+1 nodes survive f failures | High |
| **PoW** | Low | High | 51% attack threshold | Very High |

### Trade-offs Analysis

```python
class ConsensusComparison:
    @staticmethod
    def compare_algorithms():
        return {
            'raft': {
                'pros': [
                    'Easy to understand and implement',
                    'Strong consistency guarantees',
                    'Efficient leader-based approach',
                    'Good performance in normal conditions'
                ],
                'cons': [
                    'Single point of failure (leader)',
                    'Cannot handle Byzantine failures',
                    'Network partitions can cause unavailability'
                ],
                'use_cases': [
                    'Distributed databases (etcd, MongoDB)',
                    'Configuration management',
                    'Log replication'
                ]
            },
            'pbft': {
                'pros': [
                    'Handles Byzantine failures',
                    'Deterministic finality',
                    'Good for permissioned networks'
                ],
                'cons': [
                    'High message complexity O(n²)',
                    'Requires 3f+1 nodes for f failures',
                    'View change protocol is complex'
                ],
                'use_cases': [
                    'Blockchain consensus',
                    'Critical infrastructure',
                    'Financial systems'
                ]
            },
            'paxos': {
                'pros': [
                    'Theoretical foundation for consensus',
                    'Handles network partitions well',
                    'Flexible and extensible'
                ],
                'cons': [
                    'Complex to understand and implement',
                    'Can have liveness issues',
                    'Multiple rounds of communication'
                ],
                'use_cases': [
                    'Google Spanner',
                    'Apache Cassandra (for metadata)',
                    'Academic research'
                ]
            },
            'proof_of_work': {
                'pros': [
                    'No central authority needed',
                    'Handles Byzantine failures',
                    'Proven in Bitcoin'
                ],
                'cons': [
                    'Extremely energy intensive',
                    'Slow confirmation times',
                    'Probabilistic finality'
                ],
                'use_cases': [
                    'Public blockchains',
                    'Cryptocurrency systems',
                    'Decentralized networks'
                ]
            }
        }
```

## Implementation Considerations

### 1. Network Communication

```python
import asyncio
import json
from typing import Dict, Any

class NetworkLayer:
    def __init__(self, node_id: str, peers: Dict[str, str]):
        self.node_id = node_id
        self.peers = peers  # peer_id -> address
        self.message_handlers = {}
        self.server = None
    
    async def start_server(self, port: int):
        """Start network server for receiving messages"""
        self.server = await asyncio.start_server(
            self.handle_connection, 'localhost', port
        )
        print(f"Node {self.node_id} listening on port {port}")
    
    async def handle_connection(self, reader, writer):
        """Handle incoming network connection"""
        try:
            data = await reader.read(1024)
            message = json.loads(data.decode())
            
            # Route message to appropriate handler
            msg_type = message.get('type')
            if msg_type in self.message_handlers:
                response = await self.message_handlers[msg_type](message)
                if response:
                    writer.write(json.dumps(response).encode())
                    await writer.drain()
        
        except Exception as e:
            print(f"Error handling connection: {e}")
        finally:
            writer.close()
    
    async def send_message(self, peer_id: str, message: Dict[str, Any]) -> Dict[str, Any]:
        """Send message to peer and wait for response"""
        if peer_id not in self.peers:
            raise ValueError(f"Unknown peer: {peer_id}")
        
        try:
            reader, writer = await asyncio.open_connection(
                'localhost', self.peers[peer_id]
            )
            
            # Send message
            writer.write(json.dumps(message).encode())
            await writer.drain()
            
            # Wait for response
            data = await reader.read(1024)
            response = json.loads(data.decode()) if data else {}
            
            writer.close()
            return response
        
        except Exception as e:
            print(f"Error sending message to {peer_id}: {e}")
            return {}
    
    def register_handler(self, message_type: str, handler):
        """Register handler for message type"""
        self.message_handlers[message_type] = handler
```

### 2. Persistence and Recovery

```python
import pickle
import os
from typing import Any

class PersistentState:
    def __init__(self, node_id: str, state_file: str = None):
        self.node_id = node_id
        self.state_file = state_file or f"node_{node_id}_state.pkl"
        self.state = {}
    
    def save_state(self, key: str, value: Any):
        """Save state to persistent storage"""
        self.state[key] = value
        self._write_to_disk()
    
    def load_state(self, key: str, default: Any = None) -> Any:
        """Load state from persistent storage"""
        if not self.state and os.path.exists(self.state_file):
            self._read_from_disk()
        return self.state.get(key, default)
    
    def _write_to_disk(self):
        """Write state to disk"""
        try:
            with open(self.state_file, 'wb') as f:
                pickle.dump(self.state, f)
        except Exception as e:
            print(f"Error writing state to disk: {e}")
    
    def _read_from_disk(self):
        """Read state from disk"""
        try:
            with open(self.state_file, 'rb') as f:
                self.state = pickle.load(f)
        except Exception as e:
            print(f"Error reading state from disk: {e}")
            self.state = {}

class RecoverableRaftNode(RaftNode):
    def __init__(self, node_id: str, peers: List[str]):
        super().__init__(node_id, peers)
        self.persistent_state = PersistentState(node_id)
        self.recover_state()
    
    def recover_state(self):
        """Recover state from persistent storage"""
        self.current_term = self.persistent_state.load_state('current_term', 0)
        self.voted_for = self.persistent_state.load_state('voted_for', None)
        self.log = self.persistent_state.load_state('log', [])
    
    def save_persistent_state(self):
        """Save persistent state"""
        self.persistent_state.save_state('current_term', self.current_term)
        self.persistent_state.save_state('voted_for', self.voted_for)
        self.persistent_state.save_state('log', self.log)
    
    def append_entry(self, command: str) -> bool:
        """Override to save state after appending"""
        result = super().append_entry(command)
        if result:
            self.save_persistent_state()
        return result
```

### 3. Testing and Simulation

```python
import random
import time
from typing import List, Dict

class ConsensusSimulator:
    def __init__(self, num_nodes: int, algorithm: str = 'raft'):
        self.num_nodes = num_nodes
        self.algorithm = algorithm
        self.nodes = []
        self.network_partitions = []
        self.failed_nodes = set()
    
    def setup_cluster(self):
        """Setup cluster of consensus nodes"""
        node_ids = [str(i) for i in range(self.num_nodes)]
        
        if self.algorithm == 'raft':
            for i in range(self.num_nodes):
                peers = [nid for nid in node_ids if nid != str(i)]
                node = RaftNode(str(i), peers)
                self.nodes.append(node)
        
        elif self.algorithm == 'pbft':
            for i in range(self.num_nodes):
                node = PBFTNode(str(i), self.num_nodes)
                self.nodes.append(node)
    
    def simulate_client_requests(self, num_requests: int):
        """Simulate client requests"""
        results = []
        
        for i in range(num_requests):
            # Find leader (for Raft) or primary (for PBFT)
            leader = self.find_leader()
            
            if leader:
                request = f"request_{i}"
                success = leader.client_request(request)
                results.append(success)
                time.sleep(0.1)  # Small delay between requests
        
        return results
    
    def find_leader(self):
        """Find current leader/primary"""
        for node in self.nodes:
            if hasattr(node, 'state') and node.state == NodeState.LEADER:
                return node
            elif hasattr(node, 'is_primary') and node.is_primary():
                return node
        return None
    
    def simulate_node_failure(self, node_id: str):
        """Simulate node failure"""
        self.failed_nodes.add(node_id)
        print(f"Node {node_id} failed")
    
    def simulate_node_recovery(self, node_id: str):
        """Simulate node recovery"""
        if node_id in self.failed_nodes:
            self.failed_nodes.remove(node_id)
            print(f"Node {node_id} recovered")
    
    def simulate_network_partition(self, partition1: List[str], partition2: List[str]):
        """Simulate network partition"""
        self.network_partitions = [partition1, partition2]
        print(f"Network partitioned: {partition1} | {partition2}")
    
    def heal_network_partition(self):
        """Heal network partition"""
        self.network_partitions = []
        print("Network partition healed")
    
    def run_simulation(self, duration: int):
        """Run consensus simulation"""
        print(f"Starting {self.algorithm} simulation with {self.num_nodes} nodes")
        
        self.setup_cluster()
        start_time = time.time()
        
        # Simulate various scenarios
        scenarios = [
            (5, lambda: self.simulate_client_requests(10)),
            (10, lambda: self.simulate_node_failure("1")),
            (15, lambda: self.simulate_client_requests(5)),
            (20, lambda: self.simulate_node_recovery("1")),
            (25, lambda: self.simulate_network_partition(["0", "1"], ["2", "3", "4"])),
            (30, lambda: self.simulate_client_requests(5)),
            (35, lambda: self.heal_network_partition()),
            (40, lambda: self.simulate_client_requests(10))
        ]
        
        for trigger_time, action in scenarios:
            while time.time() - start_time < trigger_time:
                time.sleep(0.1)
            action()
        
        print("Simulation completed")

# Run simulation
simulator = ConsensusSimulator(5, 'raft')
simulator.run_simulation(45)
```

## Interview Questions & Answers

### Basic Questions

**Q1: What is consensus and why is it important in distributed systems?**

**Answer:** Consensus is the process by which distributed nodes agree on a single value or decision. It's crucial because:

- **Consistency**: Ensures all nodes have the same view of data
- **Coordination**: Enables distributed systems to act as a single entity
- **Fault tolerance**: Allows systems to continue operating despite failures
- **State machine replication**: Keeps replicas synchronized

**Key properties:**
- **Agreement**: All correct nodes decide the same value
- **Validity**: Decided value must be proposed by some node
- **Termination**: All correct nodes eventually decide
- **Integrity**: Each node decides at most once

**Q2: Explain the difference between Raft and Paxos.**

**Answer:**

**Raft:**
- Designed for understandability
- Leader-based approach
- Strong leader that handles all client requests
- Simpler to implement and reason about
- Clear separation of leader election and log replication

**Paxos:**
- More theoretical and flexible
- Can have multiple proposers
- More complex but handles edge cases better
- Basis for many other consensus algorithms
- Requires careful implementation to ensure liveness

**Code comparison:**
```python
# Raft - Leader-based
class RaftConsensus:
    def propose(self, value):
        if self.state != LEADER:
            return False  # Only leader can propose
        return self.replicate_to_majority(value)

# Paxos - Multi-proposer
class PaxosConsensus:
    def propose(self, value):
        # Any node can propose
        proposal_id = self.generate_proposal_id()
        return self.run_paxos_protocol(proposal_id, value)
```

**Q3: What is the Byzantine Generals Problem and how does PBFT solve it?**

**Answer:** The Byzantine Generals Problem illustrates the challenge of achieving consensus when some nodes may act maliciously or unpredictably.

**Problem:** Multiple generals must coordinate an attack, but some may be traitors sending conflicting messages.

**PBFT Solution:**
- Requires 3f+1 nodes to tolerate f Byzantine failures
- Three-phase protocol: pre-prepare, prepare, commit
- Each phase requires 2f+1 matching messages

```python
def pbft_consensus_phases():
    # Phase 1: Primary sends pre-prepare
    primary.broadcast_pre_prepare(request)
    
    # Phase 2: Backups send prepare after validating
    for backup in backups:
        if backup.validate_pre_prepare(request):
            backup.broadcast_prepare(request)
    
    # Phase 3: All nodes send commit after 2f prepares
    for node in all_nodes:
        if node.received_2f_prepares(request):
            node.broadcast_commit(request)
    
    # Execute after 2f+1 commits
    if node.received_2f_plus_1_commits(request):
        node.execute_request(request)
```

### Intermediate Questions

**Q4: How does leader election work in Raft?**

**Answer:** Raft leader election uses randomized timeouts and majority voting:

**Process:**
1. **Follower timeout**: If no heartbeat received, become candidate
2. **Start election**: Increment term, vote for self, request votes
3. **Majority wins**: If >50% votes received, become leader
4. **Send heartbeats**: Prevent other elections

```python
def raft_leader_election():
    # Follower times out
    if time_since_last_heartbeat() > election_timeout:
        self.state = CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        
        votes = 1  # Vote for self
        
        # Request votes from all peers
        for peer in self.peers:
            if self.request_vote_from(peer):
                votes += 1
        
        # Check if won election
        if votes > len(self.peers) // 2:
            self.state = LEADER
            self.send_heartbeats_to_all()
```

**Key features:**
- **Randomized timeouts**: Prevent split votes
- **Term numbers**: Detect stale leaders
- **Log completeness**: Only up-to-date candidates can win

**Q5: What are the trade-offs between different consensus algorithms?**

**Answer:**

| Aspect | Raft | PBFT | Paxos | PoW |
|--------|------|------|-------|-----|
| **Fault Model** | Crash failures | Byzantine failures | Crash failures | Byzantine failures |
| **Throughput** | High | Medium | Medium | Very Low |
| **Latency** | Low | Medium | Medium | Very High |
| **Complexity** | Low | High | Very High | Medium |
| **Energy** | Low | Low | Low | Extremely High |
| **Scalability** | Good | Poor (O(n²)) | Good | Poor |

**When to use each:**
- **Raft**: Distributed databases, configuration services
- **PBFT**: Permissioned blockchains, critical systems
- **Paxos**: Google Spanner, academic research
- **PoW**: Public blockchains, decentralized systems

**Q6: How do you handle network partitions in consensus algorithms?**

**Answer:** Different algorithms handle partitions differently:

**Raft approach:**
```python
def handle_partition_raft():
    # Majority partition continues operating
    if can_reach_majority():
        continue_as_leader()
    else:
        # Minority partition becomes unavailable
        step_down_to_follower()
        reject_client_requests()
```

**PBFT approach:**
```python
def handle_partition_pbft():
    # Need 2f+1 nodes for any progress
    if reachable_nodes >= 2 * f + 1:
        continue_normal_operation()
    else:
        # Trigger view change to find new primary
        initiate_view_change()
```

**Strategies:**
- **Quorum-based**: Require majority for operations
- **View changes**: Elect new leader when current one fails
- **Failure detection**: Use timeouts to detect partitions
- **Recovery protocols**: Sync state when partition heals

### Advanced Questions

**Q7: Design a consensus system for a global distributed database.**

**Answer:** Multi-region consensus system with hierarchical approach:

```python
class GlobalConsensusSystem:
    def __init__(self):
        self.regions = {
            'us-east': RegionalCluster(['us-east-1', 'us-east-2', 'us-east-3']),
            'eu-west': RegionalCluster(['eu-west-1', 'eu-west-2', 'eu-west-3']),
            'asia': RegionalCluster(['asia-1', 'asia-2', 'asia-3'])
        }
        self.global_coordinator = GlobalCoordinator(list(self.regions.keys()))
    
    def write_transaction(self, transaction):
        # Phase 1: Local consensus within each region
        local_results = {}
        for region, cluster in self.regions.items():
            local_results[region] = cluster.prepare_transaction(transaction)
        
        # Phase 2: Global consensus across regions
        if all(local_results.values()):
            global_decision = self.global_coordinator.commit_global(transaction)
            
            # Phase 3: Apply decision locally
            for region, cluster in self.regions.items():
                if global_decision:
                    cluster.commit_transaction(transaction)
                else:
                    cluster.abort_transaction(transaction)
            
            return global_decision
        else:
            # Abort if any region can't prepare
            for region, cluster in self.regions.items():
                cluster.abort_transaction(transaction)
            return False

class RegionalCluster:
    def __init__(self, nodes):
        self.raft_cluster = RaftCluster(nodes)
        self.prepared_transactions = {}
    
    def prepare_transaction(self, transaction):
        # Use Raft for regional consensus
        return self.raft_cluster.propose_prepare(transaction)
    
    def commit_transaction(self, transaction):
        return self.raft_cluster.propose_commit(transaction)
```

**Design decisions:**
- **Regional Raft clusters**: Fast local consensus
- **Global coordinator**: Cross-region coordination
- **Two-phase commit**: Ensure global consistency
- **Conflict resolution**: Handle concurrent updates

**Q8: How would you implement consensus in a blockchain system?**

**Answer:** Blockchain consensus with Proof of Stake:

```python
class ProofOfStakeConsensus:
    def __init__(self, validators):
        self.validators = validators  # {validator_id: stake_amount}
        self.current_epoch = 0
        self.finalized_blocks = []
        self.pending_blocks = {}
    
    def select_proposer(self, slot):
        """Select block proposer based on stake"""
        total_stake = sum(self.validators.values())
        random.seed(slot)  # Deterministic randomness
        
        target = random.randint(0, total_stake - 1)
        current_sum = 0
        
        for validator_id, stake in self.validators.items():
            current_sum += stake
            if current_sum > target:
                return validator_id
        
        return list(self.validators.keys())[0]
    
    def propose_block(self, slot, transactions):
        """Propose new block"""
        proposer = self.select_proposer(slot)
        
        if proposer == self.validator_id:
            block = Block(
                slot=slot,
                parent_hash=self.get_head_block_hash(),
                transactions=transactions,
                proposer=proposer
            )
            
            # Sign and broadcast block
            block.signature = self.sign_block(block)
            self.broadcast_block(block)
            
            return block
    
    def attest_block(self, block):
        """Attest to a block (vote for it)"""
        if self.is_valid_block(block):
            attestation = Attestation(
                slot=block.slot,
                block_hash=block.hash,
                validator_id=self.validator_id
            )
            
            attestation.signature = self.sign_attestation(attestation)
            self.broadcast_attestation(attestation)
            
            return attestation
    
    def finalize_block(self, block):
        """Finalize block with 2/3 stake attestations"""
        attestations = self.get_attestations_for_block(block)
        total_attesting_stake = sum(
            self.validators[att.validator_id] 
            for att in attestations
        )
        
        total_stake = sum(self.validators.values())
        
        if total_attesting_stake >= (2 * total_stake) // 3:
            self.finalized_blocks.append(block)
            return True
        
        return False
```

**Q9: Explain how Multi-Paxos works and its advantages over basic Paxos.**

**Answer:** Multi-Paxos optimizes basic Paxos for multiple consensus instances:

**Basic Paxos issues:**
- Each value requires full Paxos protocol
- High message overhead for multiple decisions
- No clear leader, leading to conflicts

**Multi-Paxos improvements:**
```python
class MultiPaxosNode:
    def __init__(self, node_id, nodes):
        self.node_id = node_id
        self.nodes = nodes
        self.leader = None
        self.instances = {}  # slot -> PaxosInstance
        self.next_slot = 0
    
    def become_leader(self):
        """Phase 1: Become leader for all future instances"""
        self.leader = self.node_id
        
        # Send prepare for all future slots
        for slot in range(self.next_slot, self.next_slot + 100):
            self.send_prepare_for_slot(slot)
    
    def propose_value(self, value):
        """Phase 2: Propose value (leader only)"""
        if self.leader != self.node_id:
            return False
        
        slot = self.next_slot
        self.next_slot += 1
        
        # Skip phase 1 since we're already leader
        return self.send_accept_for_slot(slot, value)
    
    def send_accept_for_slot(self, slot, value):
        """Send accept messages for specific slot"""
        accepts = 0
        for node in self.nodes:
            if node.accept_proposal(slot, value):
                accepts += 1
        
        return accepts > len(self.nodes) // 2
```

**Advantages:**
- **Leader optimization**: One node handles all proposals
- **Reduced messages**: Skip phase 1 for subsequent proposals
- **Better throughput**: Pipeline multiple instances
- **Clearer semantics**: More like traditional leader-based systems

**Q10: How do you test consensus algorithms for correctness and performance?**

**Answer:** Comprehensive testing strategy:

```python
class ConsensusTestSuite:
    def __init__(self, algorithm_class):
        self.algorithm_class = algorithm_class
        self.test_results = {}
    
    def test_basic_consensus(self):
        """Test basic consensus functionality"""
        nodes = [self.algorithm_class(i, 5) for i in range(5)]
        
        # All nodes should agree on same value
        value = "test_value"
        results = []
        
        for node in nodes:
            result = node.propose(value)
            results.append(result)
        
        # Verify all nodes decided same value
        decided_values = [node.get_decided_value() for node in nodes]
        assert all(v == decided_values[0] for v in decided_values)
        
        return True
    
    def test_fault_tolerance(self):
        """Test behavior with node failures"""
        nodes = [self.algorithm_class(i, 5) for i in range(5)]
        
        # Fail f nodes (where f < n/2 for crash failures)
        failed_nodes = nodes[:1]  # Fail 1 out of 5
        for node in failed_nodes:
            node.fail()
        
        # Remaining nodes should still reach consensus
        active_nodes = nodes[1:]
        value = "fault_test_value"
        
        for node in active_nodes:
            node.propose(value)
        
        # Verify consensus among active nodes
        decided_values = [node.get_decided_value() for node in active_nodes]
        assert all(v == decided_values[0] for v in decided_values)
        
        return True
    
    def test_network_partition(self):
        """Test behavior during network partitions"""
        nodes = [self.algorithm_class(i, 5) for i in range(5)]
        
        # Create partition: [0,1,2] | [3,4]
        partition1 = nodes[:3]  # Majority
        partition2 = nodes[3:]  # Minority
        
        # Simulate partition
        self.create_network_partition(partition1, partition2)
        
        # Majority partition should continue operating
        value = "partition_test"
        majority_results = []
        
        for node in partition1:
            result = node.propose(value)
            majority_results.append(result)
        
        # Majority should reach consensus
        assert any(majority_results)  # At least one should succeed
        
        # Minority should not be able to make progress
        minority_results = []
        for node in partition2:
            result = node.propose("minority_value")
            minority_results.append(result)
        
        assert not any(minority_results)  # All should fail
        
        return True
    
    def performance_benchmark(self, num_requests=1000):
        """Benchmark consensus performance"""
        nodes = [self.algorithm_class(i, 5) for i in range(5)]
        leader = nodes[0]  # Assume first node is leader
        
        start_time = time.time()
        successful_requests = 0
        
        for i in range(num_requests):
            if leader.propose(f"request_{i}"):
                successful_requests += 1
        
        end_time = time.time()
        
        throughput = successful_requests / (end_time - start_time)
        latency = (end_time - start_time) / successful_requests
        
        return {
            'throughput_rps': throughput,
            'average_latency_ms': latency * 1000,
            'success_rate': successful_requests / num_requests
        }
    
    def run_all_tests(self):
        """Run complete test suite"""
        tests = [
            ('basic_consensus', self.test_basic_consensus),
            ('fault_tolerance', self.test_fault_tolerance),
            ('network_partition', self.test_network_partition),
            ('performance', self.performance_benchmark)
        ]
        
        for test_name, test_func in tests:
            try:
                result = test_func()
                self.test_results[test_name] = {'status': 'PASS', 'result': result}
            except Exception as e:
                self.test_results[test_name] = {'status': 'FAIL', 'error': str(e)}
        
        return self.test_results

# Usage
test_suite = ConsensusTestSuite(RaftNode)
results = test_suite.run_all_tests()
print(f"Test results: {results}")
```

## Best Practices

1. **Choose the right algorithm**: Match algorithm to your specific requirements
2. **Handle network failures**: Implement proper timeout and retry mechanisms
3. **Persist critical state**: Save state that must survive crashes
4. **Test thoroughly**: Include fault injection and partition testing
5. **Monitor performance**: Track latency, throughput, and availability
6. **Plan for recovery**: Implement proper recovery and catch-up mechanisms
7. **Consider security**: Protect against malicious actors if needed

## Further Reading

- [Raft Paper](https://raft.github.io/raft.pdf)
- [PBFT Paper](http://pmg.csail.mit.edu/papers/osdi99.pdf)
- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Consensus in Distributed Systems](https://www.microsoft.com/en-us/research/publication/consensus-on-transaction-commit/)
- [Blockchain Consensus Algorithms](https://arxiv.org/abs/1707.01873)
