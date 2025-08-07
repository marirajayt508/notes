# Event-Driven Architecture

Event-driven architecture (EDA) is a software design pattern where the flow of the program is determined by events such as user actions, sensor outputs, or messages from other programs. It promotes loose coupling, scalability, and real-time responsiveness in distributed systems.

## Table of Contents
- [What is Event-Driven Architecture?](#what-is-event-driven-architecture)
- [Core Components](#core-components)
- [Event Patterns](#event-patterns)
- [Implementation Strategies](#implementation-strategies)
- [Message Brokers and Event Stores](#message-brokers-and-event-stores)
- [Real-World Applications](#real-world-applications)
- [Best Practices](#best-practices)
- [Interview Questions & Answers](#interview-questions--answers)

## What is Event-Driven Architecture?

### Definition
Event-driven architecture is a design paradigm where components communicate through the production and consumption of events. An event represents a significant change in state or an occurrence that other parts of the system might be interested in.

### Key Characteristics
- **Asynchronous Communication**: Components don't wait for responses
- **Loose Coupling**: Producers and consumers are independent
- **Scalability**: Easy to scale individual components
- **Real-time Processing**: Events can be processed as they occur
- **Fault Tolerance**: System can continue operating if some components fail

### Benefits
- **Responsiveness**: Real-time reaction to events
- **Scalability**: Independent scaling of components
- **Flexibility**: Easy to add new event consumers
- **Resilience**: Decoupled components reduce failure propagation
- **Auditability**: Event logs provide complete system history

## Core Components

### 1. Event Producers

Components that generate and publish events when something significant happens.

```python
import json
import time
from abc import ABC, abstractmethod
from typing import Dict, Any, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Event:
    event_id: str
    event_type: str
    source: str
    timestamp: datetime
    data: Dict[str, Any]
    version: str = "1.0"
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            'event_id': self.event_id,
            'event_type': self.event_type,
            'source': self.source,
            'timestamp': self.timestamp.isoformat(),
            'data': self.data,
            'version': self.version
        }

class EventProducer(ABC):
    def __init__(self, event_bus):
        self.event_bus = event_bus
    
    @abstractmethod
    def produce_event(self, event: Event):
        pass

class UserService(EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.users = {}
    
    def create_user(self, user_data: Dict[str, Any]) -> str:
        user_id = f"user_{len(self.users) + 1}"
        self.users[user_id] = user_data
        
        # Produce user created event
        event = Event(
            event_id=f"user_created_{user_id}_{int(time.time())}",
            event_type="user.created",
            source="user-service",
            timestamp=datetime.now(),
            data={
                'user_id': user_id,
                'email': user_data.get('email'),
                'name': user_data.get('name')
            }
        )
        
        self.produce_event(event)
        return user_id
    
    def produce_event(self, event: Event):
        self.event_bus.publish(event)

class OrderService(EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.orders = {}
    
    def create_order(self, order_data: Dict[str, Any]) -> str:
        order_id = f"order_{len(self.orders) + 1}"
        self.orders[order_id] = {
            **order_data,
            'status': 'pending',
            'created_at': datetime.now()
        }
        
        # Produce order created event
        event = Event(
            event_id=f"order_created_{order_id}_{int(time.time())}",
            event_type="order.created",
            source="order-service",
            timestamp=datetime.now(),
            data={
                'order_id': order_id,
                'user_id': order_data.get('user_id'),
                'items': order_data.get('items'),
                'total_amount': order_data.get('total_amount')
            }
        )
        
        self.produce_event(event)
        return order_id
    
    def update_order_status(self, order_id: str, status: str):
        if order_id in self.orders:
            old_status = self.orders[order_id]['status']
            self.orders[order_id]['status'] = status
            
            # Produce order status changed event
            event = Event(
                event_id=f"order_status_changed_{order_id}_{int(time.time())}",
                event_type="order.status_changed",
                source="order-service",
                timestamp=datetime.now(),
                data={
                    'order_id': order_id,
                    'old_status': old_status,
                    'new_status': status
                }
            )
            
            self.produce_event(event)
    
    def produce_event(self, event: Event):
        self.event_bus.publish(event)
```

### 2. Event Consumers

Components that subscribe to and process events.

```python
from abc import ABC, abstractmethod
from typing import Callable, List

class EventConsumer(ABC):
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.subscriptions = []
    
    @abstractmethod
    def handle_event(self, event: Event):
        pass
    
    def subscribe(self, event_type: str):
        self.event_bus.subscribe(event_type, self.handle_event)
        self.subscriptions.append(event_type)

class EmailService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("user.created")
        self.subscribe("order.created")
        self.subscribe("order.status_changed")
    
    def handle_event(self, event: Event):
        if event.event_type == "user.created":
            self._send_welcome_email(event)
        elif event.event_type == "order.created":
            self._send_order_confirmation(event)
        elif event.event_type == "order.status_changed":
            self._send_status_update(event)
    
    def _send_welcome_email(self, event: Event):
        user_data = event.data
        print(f"Sending welcome email to {user_data['email']}")
        # Simulate email sending
        time.sleep(0.1)
    
    def _send_order_confirmation(self, event: Event):
        order_data = event.data
        print(f"Sending order confirmation for order {order_data['order_id']}")
        # Simulate email sending
        time.sleep(0.1)
    
    def _send_status_update(self, event: Event):
        order_data = event.data
        print(f"Sending status update: Order {order_data['order_id']} is now {order_data['new_status']}")
        # Simulate email sending
        time.sleep(0.1)

class InventoryService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.inventory = {
            'item_1': 100,
            'item_2': 50,
            'item_3': 75
        }
        self.subscribe("order.created")
    
    def handle_event(self, event: Event):
        if event.event_type == "order.created":
            self._reserve_inventory(event)
    
    def _reserve_inventory(self, event: Event):
        order_data = event.data
        items = order_data.get('items', [])
        
        print(f"Reserving inventory for order {order_data['order_id']}")
        
        for item in items:
            item_id = item['item_id']
            quantity = item['quantity']
            
            if item_id in self.inventory and self.inventory[item_id] >= quantity:
                self.inventory[item_id] -= quantity
                print(f"Reserved {quantity} units of {item_id}")
            else:
                print(f"Insufficient inventory for {item_id}")
                # Could produce an inventory shortage event here

class AnalyticsService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.metrics = {
            'users_created': 0,
            'orders_created': 0,
            'orders_completed': 0
        }
        self.subscribe("user.created")
        self.subscribe("order.created")
        self.subscribe("order.status_changed")
    
    def handle_event(self, event: Event):
        if event.event_type == "user.created":
            self.metrics['users_created'] += 1
        elif event.event_type == "order.created":
            self.metrics['orders_created'] += 1
        elif event.event_type == "order.status_changed":
            if event.data.get('new_status') == 'completed':
                self.metrics['orders_completed'] += 1
        
        print(f"Analytics updated: {self.metrics}")
```

### 3. Event Bus/Broker

The central component that routes events from producers to consumers.

```python
import threading
from collections import defaultdict
from typing import Dict, List, Callable
import queue
import time

class InMemoryEventBus:
    def __init__(self):
        self.subscribers = defaultdict(list)  # event_type -> list of handlers
        self.event_queue = queue.Queue()
        self.running = False
        self.worker_thread = None
        self.lock = threading.Lock()
    
    def subscribe(self, event_type: str, handler: Callable[[Event], None]):
        with self.lock:
            self.subscribers[event_type].append(handler)
    
    def unsubscribe(self, event_type: str, handler: Callable[[Event], None]):
        with self.lock:
            if handler in self.subscribers[event_type]:
                self.subscribers[event_type].remove(handler)
    
    def publish(self, event: Event):
        self.event_queue.put(event)
    
    def start(self):
        self.running = True
        self.worker_thread = threading.Thread(target=self._process_events)
        self.worker_thread.start()
    
    def stop(self):
        self.running = False
        if self.worker_thread:
            self.worker_thread.join()
    
    def _process_events(self):
        while self.running:
            try:
                event = self.event_queue.get(timeout=1)
                self._dispatch_event(event)
            except queue.Empty:
                continue
    
    def _dispatch_event(self, event: Event):
        with self.lock:
            handlers = self.subscribers.get(event.event_type, [])
        
        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                print(f"Error handling event {event.event_id}: {e}")

# Advanced event bus with persistence
class PersistentEventBus:
    def __init__(self, storage_backend):
        self.storage = storage_backend
        self.subscribers = defaultdict(list)
        self.event_queue = queue.Queue()
        self.running = False
        self.worker_thread = None
        self.lock = threading.Lock()
    
    def subscribe(self, event_type: str, handler: Callable[[Event], None]):
        with self.lock:
            self.subscribers[event_type].append(handler)
    
    def publish(self, event: Event):
        # Persist event first
        self.storage.store_event(event)
        # Then queue for processing
        self.event_queue.put(event)
    
    def replay_events(self, from_timestamp: datetime = None):
        """Replay events from storage"""
        events = self.storage.get_events(from_timestamp)
        for event in events:
            self._dispatch_event(event)
    
    def start(self):
        self.running = True
        self.worker_thread = threading.Thread(target=self._process_events)
        self.worker_thread.start()
    
    def stop(self):
        self.running = False
        if self.worker_thread:
            self.worker_thread.join()
    
    def _process_events(self):
        while self.running:
            try:
                event = self.event_queue.get(timeout=1)
                self._dispatch_event(event)
            except queue.Empty:
                continue
    
    def _dispatch_event(self, event: Event):
        with self.lock:
            handlers = self.subscribers.get(event.event_type, [])
        
        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                print(f"Error handling event {event.event_id}: {e}")
                # Could implement retry logic here

class EventStorage:
    def __init__(self):
        self.events = []
        self.lock = threading.Lock()
    
    def store_event(self, event: Event):
        with self.lock:
            self.events.append(event)
    
    def get_events(self, from_timestamp: datetime = None) -> List[Event]:
        with self.lock:
            if from_timestamp:
                return [e for e in self.events if e.timestamp >= from_timestamp]
            return self.events.copy()
```

## Event Patterns

### 1. Event Sourcing

Store all changes as a sequence of events rather than just the current state.

```python
from typing import List, Optional
import json

class EventStore:
    def __init__(self):
        self.events = {}  # aggregate_id -> list of events
        self.snapshots = {}  # aggregate_id -> snapshot
    
    def append_events(self, aggregate_id: str, events: List[Event], expected_version: int = -1):
        if aggregate_id not in self.events:
            self.events[aggregate_id] = []
        
        current_version = len(self.events[aggregate_id])
        if expected_version != -1 and current_version != expected_version:
            raise Exception(f"Concurrency conflict: expected version {expected_version}, got {current_version}")
        
        self.events[aggregate_id].extend(events)
    
    def get_events(self, aggregate_id: str, from_version: int = 0) -> List[Event]:
        return self.events.get(aggregate_id, [])[from_version:]
    
    def save_snapshot(self, aggregate_id: str, snapshot: Dict[str, Any], version: int):
        self.snapshots[aggregate_id] = {
            'data': snapshot,
            'version': version
        }
    
    def get_snapshot(self, aggregate_id: str) -> Optional[Dict[str, Any]]:
        return self.snapshots.get(aggregate_id)

class BankAccount:
    def __init__(self, account_id: str):
        self.account_id = account_id
        self.balance = 0
        self.version = 0
        self.uncommitted_events = []
    
    def deposit(self, amount: float):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        
        event = Event(
            event_id=f"deposit_{self.account_id}_{int(time.time())}",
            event_type="account.deposited",
            source="bank-account",
            timestamp=datetime.now(),
            data={
                'account_id': self.account_id,
                'amount': amount,
                'balance_after': self.balance + amount
            }
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
    
    def withdraw(self, amount: float):
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        
        event = Event(
            event_id=f"withdraw_{self.account_id}_{int(time.time())}",
            event_type="account.withdrawn",
            source="bank-account",
            timestamp=datetime.now(),
            data={
                'account_id': self.account_id,
                'amount': amount,
                'balance_after': self.balance - amount
            }
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
    
    def _apply_event(self, event: Event):
        if event.event_type == "account.deposited":
            self.balance += event.data['amount']
        elif event.event_type == "account.withdrawn":
            self.balance -= event.data['amount']
        
        self.version += 1
    
    def get_uncommitted_events(self) -> List[Event]:
        return self.uncommitted_events.copy()
    
    def mark_events_as_committed(self):
        self.uncommitted_events.clear()
    
    @classmethod
    def from_events(cls, account_id: str, events: List[Event]) -> 'BankAccount':
        account = cls(account_id)
        for event in events:
            account._apply_event(event)
        return account

class BankAccountRepository:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
    
    def save(self, account: BankAccount):
        events = account.get_uncommitted_events()
        if events:
            self.event_store.append_events(
                account.account_id, 
                events, 
                account.version - len(events)
            )
            account.mark_events_as_committed()
    
    def get(self, account_id: str) -> Optional[BankAccount]:
        events = self.event_store.get_events(account_id)
        if not events:
            return None
        
        return BankAccount.from_events(account_id, events)
```

### 2. CQRS (Command Query Responsibility Segregation)

Separate read and write models, often used with event sourcing.

```python
from abc import ABC, abstractmethod

# Command side (Write model)
class Command(ABC):
    pass

class CreateAccountCommand(Command):
    def __init__(self, account_id: str, initial_deposit: float):
        self.account_id = account_id
        self.initial_deposit = initial_deposit

class DepositCommand(Command):
    def __init__(self, account_id: str, amount: float):
        self.account_id = account_id
        self.amount = amount

class WithdrawCommand(Command):
    def __init__(self, account_id: str, amount: float):
        self.account_id = account_id
        self.amount = amount

class CommandHandler(ABC):
    @abstractmethod
    def handle(self, command: Command):
        pass

class BankAccountCommandHandler(CommandHandler):
    def __init__(self, repository: BankAccountRepository, event_bus):
        self.repository = repository
        self.event_bus = event_bus
    
    def handle(self, command: Command):
        if isinstance(command, CreateAccountCommand):
            self._handle_create_account(command)
        elif isinstance(command, DepositCommand):
            self._handle_deposit(command)
        elif isinstance(command, WithdrawCommand):
            self._handle_withdraw(command)
    
    def _handle_create_account(self, command: CreateAccountCommand):
        account = BankAccount(command.account_id)
        if command.initial_deposit > 0:
            account.deposit(command.initial_deposit)
        
        self.repository.save(account)
        
        # Publish events
        for event in account.get_uncommitted_events():
            self.event_bus.publish(event)
    
    def _handle_deposit(self, command: DepositCommand):
        account = self.repository.get(command.account_id)
        if not account:
            raise ValueError(f"Account {command.account_id} not found")
        
        account.deposit(command.amount)
        self.repository.save(account)
        
        # Publish events
        for event in account.get_uncommitted_events():
            self.event_bus.publish(event)
    
    def _handle_withdraw(self, command: WithdrawCommand):
        account = self.repository.get(command.account_id)
        if not account:
            raise ValueError(f"Account {command.account_id} not found")
        
        account.withdraw(command.amount)
        self.repository.save(account)
        
        # Publish events
        for event in account.get_uncommitted_events():
            self.event_bus.publish(event)

# Query side (Read model)
class AccountSummary:
    def __init__(self, account_id: str, balance: float, transaction_count: int):
        self.account_id = account_id
        self.balance = balance
        self.transaction_count = transaction_count

class AccountSummaryProjection(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.summaries = {}  # account_id -> AccountSummary
        self.subscribe("account.deposited")
        self.subscribe("account.withdrawn")
    
    def handle_event(self, event: Event):
        account_id = event.data['account_id']
        
        if account_id not in self.summaries:
            self.summaries[account_id] = AccountSummary(account_id, 0, 0)
        
        summary = self.summaries[account_id]
        
        if event.event_type == "account.deposited":
            summary.balance += event.data['amount']
            summary.transaction_count += 1
        elif event.event_type == "account.withdrawn":
            summary.balance -= event.data['amount']
            summary.transaction_count += 1
    
    def get_account_summary(self, account_id: str) -> Optional[AccountSummary]:
        return self.summaries.get(account_id)

class QueryHandler:
    def __init__(self, projection: AccountSummaryProjection):
        self.projection = projection
    
    def get_account_summary(self, account_id: str) -> Optional[AccountSummary]:
        return self.projection.get_account_summary(account_id)
```

### 3. Saga Pattern

Manage distributed transactions using a sequence of events.

```python
from enum import Enum
from typing import Dict, Any, List

class SagaStatus(Enum):
    STARTED = "started"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATING = "compensating"
    COMPENSATED = "compensated"

class SagaStep:
    def __init__(self, name: str, command: Command, compensation_command: Command = None):
        self.name = name
        self.command = command
        self.compensation_command = compensation_command
        self.completed = False
        self.compensated = False

class Saga:
    def __init__(self, saga_id: str, steps: List[SagaStep]):
        self.saga_id = saga_id
        self.steps = steps
        self.current_step = 0
        self.status = SagaStatus.STARTED
        self.completed_steps = []
    
    def get_next_command(self) -> Optional[Command]:
        if self.current_step < len(self.steps):
            return self.steps[self.current_step].command
        return None
    
    def mark_step_completed(self):
        if self.current_step < len(self.steps):
            self.steps[self.current_step].completed = True
            self.completed_steps.append(self.current_step)
            self.current_step += 1
            
            if self.current_step >= len(self.steps):
                self.status = SagaStatus.COMPLETED
    
    def start_compensation(self):
        self.status = SagaStatus.COMPENSATING
        self.current_step = len(self.completed_steps) - 1
    
    def get_compensation_command(self) -> Optional[Command]:
        if self.status == SagaStatus.COMPENSATING and self.current_step >= 0:
            step_index = self.completed_steps[self.current_step]
            step = self.steps[step_index]
            if step.compensation_command and not step.compensated:
                return step.compensation_command
        return None
    
    def mark_compensation_completed(self):
        if self.status == SagaStatus.COMPENSATING and self.current_step >= 0:
            step_index = self.completed_steps[self.current_step]
            self.steps[step_index].compensated = True
            self.current_step -= 1
            
            if self.current_step < 0:
                self.status = SagaStatus.COMPENSATED

class OrderSaga(Saga):
    def __init__(self, saga_id: str, order_data: Dict[str, Any]):
        steps = [
            SagaStep(
                "reserve_inventory",
                ReserveInventoryCommand(order_data['items']),
                ReleaseInventoryCommand(order_data['items'])
            ),
            SagaStep(
                "process_payment",
                ProcessPaymentCommand(order_data['payment_info']),
                RefundPaymentCommand(order_data['payment_info'])
            ),
            SagaStep(
                "create_shipment",
                CreateShipmentCommand(order_data['shipping_info']),
                CancelShipmentCommand(order_data['shipping_info'])
            )
        ]
        super().__init__(saga_id, steps)
        self.order_data = order_data

class SagaManager(EventConsumer):
    def __init__(self, event_bus, command_handler):
        super().__init__(event_bus)
        self.command_handler = command_handler
        self.active_sagas = {}  # saga_id -> Saga
        
        # Subscribe to saga-related events
        self.subscribe("saga.step_completed")
        self.subscribe("saga.step_failed")
    
    def start_saga(self, saga: Saga):
        self.active_sagas[saga.saga_id] = saga
        self._execute_next_step(saga)
    
    def handle_event(self, event: Event):
        if event.event_type == "saga.step_completed":
            self._handle_step_completed(event)
        elif event.event_type == "saga.step_failed":
            self._handle_step_failed(event)
    
    def _handle_step_completed(self, event: Event):
        saga_id = event.data['saga_id']
        saga = self.active_sagas.get(saga_id)
        
        if saga:
            saga.mark_step_completed()
            
            if saga.status == SagaStatus.COMPLETED:
                print(f"Saga {saga_id} completed successfully")
                del self.active_sagas[saga_id]
            else:
                self._execute_next_step(saga)
    
    def _handle_step_failed(self, event: Event):
        saga_id = event.data['saga_id']
        saga = self.active_sagas.get(saga_id)
        
        if saga:
            saga.start_compensation()
            self._execute_compensation(saga)
    
    def _execute_next_step(self, saga: Saga):
        command = saga.get_next_command()
        if command:
            try:
                self.command_handler.handle(command)
            except Exception as e:
                # Publish step failed event
                event = Event(
                    event_id=f"saga_step_failed_{saga.saga_id}_{int(time.time())}",
                    event_type="saga.step_failed",
                    source="saga-manager",
                    timestamp=datetime.now(),
                    data={'saga_id': saga.saga_id, 'error': str(e)}
                )
                self.event_bus.publish(event)
    
    def _execute_compensation(self, saga: Saga):
        command = saga.get_compensation_command()
        if command:
            try:
                self.command_handler.handle(command)
                saga.mark_compensation_completed()
                
                if saga.status == SagaStatus.COMPENSATED:
                    print(f"Saga {saga.saga_id} compensated successfully")
                    del self.active_sagas[saga.saga_id]
                else:
                    self._execute_compensation(saga)
            except Exception as e:
                print(f"Compensation failed for saga {saga.saga_id}: {e}")
```

## Implementation Strategies

### 1. Message Queues

Using message queues for reliable event delivery.

```python
import pika
import json
from typing import Callable

class RabbitMQEventBus:
    def __init__(self, connection_params):
        self.connection = pika.BlockingConnection(connection_params)
        self.channel = self.connection.channel()
        self.exchange_name = 'events'
        
        # Declare exchange
        self.channel.exchange_declare(
            exchange=self.exchange_name,
            exchange_type='topic',
            durable=True
        )
    
    def publish(self, event: Event):
        message = json.dumps(event.to_dict())
        
        self.channel.basic_publish(
            exchange=self.exchange_name,
            routing_key=event.event_type,
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # Make message persistent
                message_id=event.event_id,
                timestamp=int(event.timestamp.timestamp())
            )
        )
    
    def subscribe(self, event_pattern: str, handler: Callable[[Event], None]):
        # Create queue for this consumer
        queue_name = f"queue_{event_pattern}_{id(handler)}"
        
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.queue_bind(
            exchange=self.exchange_name,
            queue=queue_name,
            routing_key=event_pattern
        )
        
        def callback(ch, method, properties, body):
            try:
                event_data = json.loads(body)
                event = Event(
                    event_id=event_data['event_id'],
                    event_type=event_data['event_type'],
                    source=event_data['source'],
                    timestamp=datetime.fromisoformat(event_data['timestamp']),
                    data=event_data['data'],
                    version=event_data.get('version', '1.0')
                )
                
                handler(event)
                ch.basic_ack(delivery_tag=method.delivery_tag)
                
            except Exception as e:
                print(f"Error processing event: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback
        )
    
    def start_consuming(self):
        self.channel.start_consuming()
    
    def stop_consuming(self):
        self.channel.stop_consuming()
    
    def close(self):
        self.connection.close()

# Apache Kafka implementation
from kafka import KafkaProducer, KafkaConsumer
import json

class KafkaEventBus:
    def __init__(self, bootstrap_servers):
        self.bootstrap_servers = bootstrap_servers
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )
        self.consumers = {}
    
    def publish(self, event: Event):
        topic = event.event_type.replace('.', '_')
        
        self.producer.send(
            topic,
            key=event.event_id,
            value=event.to_dict()
        )
        self.producer.flush()
    
    def subscribe(self, event_pattern: str, handler: Callable[[Event], None]):
        topic = event_pattern.replace('.', '_')
        
        consumer = KafkaConsumer(
            topic,
            bootstrap_servers=self.bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            key_deserializer=lambda k: k.decode('utf-8') if k else None,
            group_id=f"consumer_group_{id(handler)}"
        )
        
        self.consumers[topic] = consumer
        
        def consume_messages():
            for message in consumer:
                try:
                    event_data = message.value
                    event = Event(
                        event_id=event_data['event_id'],
                        event_type=event_data['event_type'],
                        source=event_data['source'],
                        timestamp=datetime.fromisoformat(event_data['timestamp']),
                        data=event_data['data'],
                        version=event_data.get('version', '1.0')
                    )
                    
                    handler(event)
                    
                except Exception as e:
                    print(f"Error processing Kafka message: {e}")
        
        # Start consumer in separate thread
        consumer_thread = threading.Thread(target=consume_messages)
        consumer_thread.daemon = True
        consumer_thread.start()
    
    def close(self):
        self.producer.close()
        for consumer in self.consumers.values():
            consumer.close()
```

## Message Brokers and Event Stores

### 1. Event Store Implementation

```python
import sqlite3
import json
from typing import List, Optional, Dict, Any
from datetime import datetime

class SQLiteEventStore:
    def __init__(self, db_path: str = "events.db"):
        self.db_path = db_path
        self._init_database()
    
    def _init_database(self):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                event_id TEXT UNIQUE NOT NULL,
                event_type TEXT NOT NULL,
                source TEXT NOT NULL,
                timestamp TEXT NOT NULL,
                data TEXT NOT NULL,
                version TEXT NOT NULL,
                aggregate_id TEXT,
                aggregate_version INTEGER,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        cursor.execute('''
            CREATE INDEX IF NOT EXISTS idx_event_type ON events(event_type)
        ''')
        
        cursor.execute('''
            CREATE INDEX IF NOT EXISTS idx_aggregate ON events(aggregate_id, aggregate_version)
        ''')
        
        cursor.execute('''
            CREATE INDEX IF NOT EXISTS idx_timestamp ON events(timestamp)
        ''')
        
        conn.commit()
        conn.close()
    
    def append_event(self, event: Event, aggregate_id: str = None, aggregate_version: int = None):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
                INSERT INTO events (event_id, event_type, source, timestamp, data, version, aggregate_id, aggregate_version)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                event.event_id,
                event.event_type,
                event.source,
                event.timestamp.isoformat(),
                json.dumps(event.data),
                event.version,
                aggregate_id,
                aggregate_version
            ))
            
            conn.commit()
            
        except sqlite3.IntegrityError as e:
            raise Exception(f"Event {event.event_id} already exists") from e
        finally:
            conn.close()
    
    def get_events(self, 
                   event_types: List[str] = None,
                   from_timestamp: datetime = None,
                   to_timestamp: datetime = None,
                   aggregate_id: str = None,
                   limit: int = None) -> List[Event]:
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        query = "SELECT event_id, event_type, source, timestamp, data, version FROM events WHERE 1=1"
        params = []
        
        if event_types:
            placeholders = ','.join(['?' for _ in event_types])
            query += f" AND event_type IN ({placeholders})"
            params.extend(event_types)
        
        if from_timestamp:
            query += " AND timestamp >= ?"
            params.append(from_timestamp.isoformat())
        
        if to_timestamp:
            query += " AND timestamp <= ?"
            params.append(to_timestamp.isoformat())
        
        if aggregate_id:
            query += " AND aggregate_id = ?"
            params.append(aggregate_id)
        
        query += " ORDER BY timestamp ASC"
        
        if limit:
            query += " LIMIT ?"
            params.append(limit)
        
        cursor.execute(query, params)
        rows = cursor.fetchall()
        conn.close()
        
        events = []
        for row in rows:
            event = Event(
                event_id=row[0],
                event_type=row[1],
                source=row[2],
                timestamp=datetime.fromisoformat(row[3]),
                data=json.loads(row[4]),
                version=row[5]
            )
            events.append(event)
        
        return events
    
    def get_event_by_id(self, event_id: str) -> Optional[Event]:
        events = self.get_events()
        for event in events:
            if event.event_id == event_id:
                return event
        return None

class EventStoreProjection:
    """Base class for event store projections"""
    
    def __init__(self, event_store: SQLiteEventStore, projection_name: str):
        self.event_store = event_store
        self.projection_name = projection_name
        self.last_processed_timestamp = None
    
    def rebuild_projection(self):
        """Rebuild projection from all events"""
        self.reset_projection()
        events = self.event_store.get_events()
        
        for event in events:
            self.handle_event(event)
        
        self.last_processed_timestamp = datetime.now()
    
    def update_projection(self):
        """Update projection with new events since last update"""
        events = self.event_store.get_events(from_timestamp=self.last_processed_timestamp)
        
        for event in events:
            self.handle_event(event)
        
        self.last_processed_timestamp = datetime.now()
    
    def handle_event(self, event: Event):
        """Override this method to handle specific events"""
        pass
    
    def reset_projection(self):
        """Override this method to reset projection state"""
        pass

class UserStatisticsProjection(EventStoreProjection):
    def __init__(self, event_store: SQLiteEventStore):
        super().__init__(event_store, "user_statistics")
        self.stats = {
            'total_users': 0,
            'total_orders': 0,
            'total_revenue': 0.0
        }
    
    def handle_event(self, event: Event):
        if event.event_type == "user.created":
            self.stats['total_users'] += 1
        elif event.event_type == "order.created":
            self.stats['total_orders'] += 1
            self.stats['total_revenue'] += event.data.get('total_amount', 0)
    
    def reset_projection(self):
        self.stats = {
            'total_users': 0,
            'total_orders': 0,
            'total_revenue': 0.0
        }
    
    def get_statistics(self) -> Dict[str, Any]:
        return self.stats.copy()
```

### 2. Event Streaming with Apache Kafka

```python
from kafka.admin import KafkaAdminClient, NewTopic
from kafka.errors import TopicAlreadyExistsError

class KafkaEventStreaming:
    def __init__(self, bootstrap_servers: List[str]):
        self.bootstrap_servers = bootstrap_servers
        self.admin_client = KafkaAdminClient(
            bootstrap_servers=bootstrap_servers
        )
    
    def create_topic(self, topic_name: str, num_partitions: int = 3, replication_factor: int = 1):
        """Create Kafka topic for event streaming"""
        topic = NewTopic(
            name=topic_name,
            num_partitions=num_partitions,
            replication_factor=replication_factor
        )
        
        try:
            self.admin_client.create_topics([topic])
            print(f"Topic {topic_name} created successfully")
        except TopicAlreadyExistsError:
            print(f"Topic {topic_name} already exists")
    
    def setup_event_topics(self):
        """Setup standard event topics"""
        topics = [
            "user-events",
            "order-events",
            "payment-events",
            "inventory-events",
            "notification-events"
        ]
        
        for topic in topics:
            self.create_topic(topic)

class EventStreamProcessor:
    """Process event streams with windowing and aggregation"""
    
    def __init__(self, kafka_servers: List[str]):
        self.kafka_servers = kafka_servers
        self.processors = {}
    
    def create_windowed_aggregation(self, 
                                   input_topic: str,
                                   output_topic: str,
                                   window_size_seconds: int,
                                   aggregation_func: Callable):
        """Create windowed aggregation processor"""
        
        consumer = KafkaConsumer(
            input_topic,
            bootstrap_servers=self.kafka_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            group_id=f"aggregator_{input_topic}"
        )
        
        producer = KafkaProducer(
            bootstrap_servers=self.kafka_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        
        def process_stream():
            window_data = []
            window_start = time.time()
            
            for message in consumer:
                current_time = time.time()
                
                # Check if window should be closed
                if current_time - window_start >= window_size_seconds:
                    if window_data:
                        # Apply aggregation function
                        result = aggregation_func(window_data)
                        
                        # Send aggregated result
                        producer.send(output_topic, value={
                            'window_start': window_start,
                            'window_end': current_time,
                            'result': result,
                            'count': len(window_data)
                        })
                    
                    # Reset window
                    window_data = []
                    window_start = current_time
                
                # Add message to current window
                window_data.append(message.value)
        
        # Start processor in separate thread
        processor_thread = threading.Thread(target=process_stream)
        processor_thread.daemon = True
        processor_thread.start()
        
        self.processors[f"{input_topic}_to_{output_topic}"] = processor_thread

# Usage example
def calculate_order_metrics(events):
    """Aggregate order events to calculate metrics"""
    total_orders = len(events)
    total_revenue = sum(event.get('total_amount', 0) for event in events)
    avg_order_value = total_revenue / total_orders if total_orders > 0 else 0
    
    return {
        'total_orders': total_orders,
        'total_revenue': total_revenue,
        'avg_order_value': avg_order_value
    }

# Setup stream processing
streaming = KafkaEventStreaming(['localhost:9092'])
streaming.setup_event_topics()

processor = EventStreamProcessor(['localhost:9092'])
processor.create_windowed_aggregation(
    'order-events',
    'order-metrics',
    60,  # 1-minute windows
    calculate_order_metrics
)
```

## Real-World Applications

### 1. E-commerce Order Processing

```python
class ECommerceEventSystem:
    def __init__(self):
        self.event_bus = InMemoryEventBus()
        self.event_store = SQLiteEventStore()
        
        # Services
        self.user_service = UserService(self.event_bus)
        self.order_service = OrderService(self.event_bus)
        self.inventory_service = InventoryService(self.event_bus)
        self.payment_service = PaymentService(self.event_bus)
        self.shipping_service = ShippingService(self.event_bus)
        self.email_service = EmailService(self.event_bus)
        
        # Start event bus
        self.event_bus.start()
    
    def process_order(self, user_id: str, items: List[Dict], payment_info: Dict):
        """Process complete order workflow"""
        
        # Create order
        order_data = {
            'user_id': user_id,
            'items': items,
            'total_amount': sum(item['price'] * item['quantity'] for item in items),
            'payment_info': payment_info
        }
        
        order_id = self.order_service.create_order(order_data)
        
        # The rest happens through events:
        # 1. InventoryService reserves items
        # 2. PaymentService processes payment
        # 3. ShippingService creates shipment
        # 4. EmailService sends confirmations
        
        return order_id

class PaymentService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("order.created")
    
    def handle_event(self, event: Event):
        if event.event_type == "order.created":
            self._process_payment(event)
    
    def _process_payment(self, event: Event):
        order_data = event.data
        print(f"Processing payment for order {order_data['order_id']}")
        
        # Simulate payment processing
        time.sleep(0.2)
        
        # Publish payment processed event
        payment_event = Event(
            event_id=f"payment_processed_{order_data['order_id']}_{int(time.time())}",
            event_type="payment.processed",
            source="payment-service",
            timestamp=datetime.now(),
            data={
                'order_id': order_data['order_id'],
                'amount': order_data['total_amount'],
                'status': 'completed'
            }
        )
        
        self.event_bus.publish(payment_event)

class ShippingService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("payment.processed")
    
    def handle_event(self, event: Event):
        if event.event_type == "payment.processed":
            self._create_shipment(event)
    
    def _create_shipment(self, event: Event):
        order_data = event.data
        print(f"Creating shipment for order {order_data['order_id']}")
        
        # Simulate shipment creation
        time.sleep(0.1)
        
        # Publish shipment created event
        shipment_event = Event(
            event_id=f"shipment_created_{order_data['order_id']}_{int(time.time())}",
            event_type="shipment.created",
            source="shipping-service",
            timestamp=datetime.now(),
            data={
                'order_id': order_data['order_id'],
                'tracking_number': f"TRACK_{order_data['order_id']}",
                'estimated_delivery': '3-5 business days'
            }
        )
        
        self.event_bus.publish(shipment_event)
```

### 2. IoT Sensor Data Processing

```python
import random
from typing import Dict, Any

class IoTEventSystem:
    def __init__(self):
        self.event_bus = KafkaEventBus(['localhost:9092'])
        self.sensor_manager = SensorManager(self.event_bus)
        self.alert_service = AlertService(self.event_bus)
        self.analytics_service = IoTAnalyticsService(self.event_bus)
    
    def simulate_sensor_data(self):
        """Simulate IoT sensor data generation"""
        sensors = ['temp_01', 'humidity_01', 'pressure_01', 'motion_01']
        
        while True:
            for sensor_id in sensors:
                self.sensor_manager.generate_reading(sensor_id)
            time.sleep(1)

class SensorManager(EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.sensors = {
            'temp_01': {'type': 'temperature', 'unit': 'celsius', 'location': 'room_a'},
            'humidity_01': {'type': 'humidity', 'unit': 'percent', 'location': 'room_a'},
            'pressure_01': {'type': 'pressure', 'unit': 'hpa', 'location': 'room_b'},
            'motion_01': {'type': 'motion', 'unit': 'boolean', 'location': 'hallway'}
        }
    
    def generate_reading(self, sensor_id: str):
        """Generate sensor reading"""
        if sensor_id not in self.sensors:
            return
        
        sensor_info = self.sensors[sensor_id]
        
        # Generate realistic sensor data
        if sensor_info['type'] == 'temperature':
            value = random.uniform(18.0, 28.0)
        elif sensor_info['type'] == 'humidity':
            value = random.uniform(30.0, 70.0)
        elif sensor_info['type'] == 'pressure':
            value = random.uniform(1000.0, 1020.0)
        elif sensor_info['type'] == 'motion':
            value = random.choice([True, False])
        else:
            value = random.uniform(0, 100)
        
        event = Event(
            event_id=f"sensor_reading_{sensor_id}_{int(time.time() * 1000)}",
            event_type="sensor.reading",
            source=f"sensor-{sensor_id}",
            timestamp=datetime.now(),
            data={
                'sensor_id': sensor_id,
                'sensor_type': sensor_info['type'],
                'location': sensor_info['location'],
                'value': value,
                'unit': sensor_info['unit']
            }
        )
        
        self.produce_event(event)
    
    def produce_event(self, event: Event):
        self.event_bus.publish(event)

class AlertService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("sensor.reading")
        self.thresholds = {
            'temperature': {'min': 15.0, 'max': 30.0},
            'humidity': {'min': 20.0, 'max': 80.0},
            'pressure': {'min': 995.0, 'max': 1025.0}
        }
    
    def handle_event(self, event: Event):
        if event.event_type == "sensor.reading":
            self._check_thresholds(event)
    
    def _check_thresholds(self, event: Event):
        sensor_data = event.data
        sensor_type = sensor_data['sensor_type']
        value = sensor_data['value']
        
        if sensor_type in self.thresholds:
            threshold = self.thresholds[sensor_type]
            
            if value < threshold['min'] or value > threshold['max']:
                alert_event = Event(
                    event_id=f"alert_{sensor_data['sensor_id']}_{int(time.time())}",
                    event_type="sensor.alert",
                    source="alert-service",
                    timestamp=datetime.now(),
                    data={
                        'sensor_id': sensor_data['sensor_id'],
                        'sensor_type': sensor_type,
                        'location': sensor_data['location'],
                        'current_value': value,
                        'threshold_min': threshold['min'],
                        'threshold_max': threshold['max'],
                        'severity': 'high' if value < threshold['min'] * 0.8 or value > threshold['max'] * 1.2 else 'medium'
                    }
                )
                
                self.event_bus.publish(alert_event)
                print(f"ALERT: {sensor_type} sensor {sensor_data['sensor_id']} value {value} is out of range!")

class IoTAnalyticsService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("sensor.reading")
        self.readings = defaultdict(list)
        self.window_size = 60  # 1 minute
    
    def handle_event(self, event: Event):
        if event.event_type == "sensor.reading":
            self._process_reading(event)
    
    def _process_reading(self, event: Event):
        sensor_data = event.data
        sensor_id = sensor_data['sensor_id']
        
        # Add reading to window
        self.readings[sensor_id].append({
            'timestamp': event.timestamp,
            'value': sensor_data['value']
        })
        
        # Remove old readings outside window
        cutoff_time = event.timestamp - timedelta(seconds=self.window_size)
        self.readings[sensor_id] = [
            r for r in self.readings[sensor_id] 
            if r['timestamp'] > cutoff_time
        ]
        
        # Calculate analytics every 10 readings
        if len(self.readings[sensor_id]) % 10 == 0:
            self._calculate_analytics(sensor_id, sensor_data)
    
    def _calculate_analytics(self, sensor_id: str, sensor_data: Dict):
        readings = self.readings[sensor_id]
        if not readings:
            return
        
        values = [r['value'] for r in readings]
        
        analytics = {
            'sensor_id': sensor_id,
            'sensor_type': sensor_data['sensor_type'],
            'location': sensor_data['location'],
            'window_size_seconds': self.window_size,
            'reading_count': len(values),
            'avg_value': sum(values) / len(values),
            'min_value': min(values),
            'max_value': max(values),
            'std_deviation': statistics.stdev(values) if len(values) > 1 else 0
        }
        
        analytics_event = Event(
            event_id=f"analytics_{sensor_id}_{int(time.time())}",
            event_type="sensor.analytics",
            source="analytics-service",
            timestamp=datetime.now(),
            data=analytics
        )
        
        self.event_bus.publish(analytics_event)
        print(f"Analytics for {sensor_id}: avg={analytics['avg_value']:.2f}, std={analytics['std_deviation']:.2f}")
```

## Best Practices

### 1. Event Design Principles

```python
class EventDesignPrinciples:
    """Best practices for event design"""
    
    @staticmethod
    def create_well_designed_event():
        """Example of well-designed event"""
        return Event(
            # Use descriptive, hierarchical event types
            event_type="order.payment.completed",
            
            # Include unique, traceable ID
            event_id=f"payment_completed_{uuid.uuid4()}",
            
            # Identify the source system
            source="payment-service-v2",
            
            # Use ISO timestamp
            timestamp=datetime.now(),
            
            # Include all necessary data, avoid references
            data={
                'order_id': 'order_12345',
                'payment_id': 'pay_67890',
                'amount': 99.99,
                'currency': 'USD',
                'payment_method': 'credit_card',
                'customer_id': 'cust_abc123',
                'transaction_id': 'txn_xyz789'
            },
            
            # Version for schema evolution
            version="2.1"
        )
    
    @staticmethod
    def event_naming_conventions():
        """Event naming best practices"""
        return {
            'format': 'domain.entity.action',
            'examples': [
                'user.account.created',
                'order.payment.processed',
                'inventory.item.reserved',
                'notification.email.sent'
            ],
            'guidelines': [
                'Use past tense for actions',
                'Be specific and descriptive',
                'Use consistent naming across services',
                'Avoid abbreviations'
            ]
        }

class EventVersioning:
    """Handle event schema evolution"""
    
    def __init__(self):
        self.schema_registry = {}
    
    def register_schema(self, event_type: str, version: str, schema: Dict):
        """Register event schema version"""
        key = f"{event_type}:{version}"
        self.schema_registry[key] = schema
    
    def validate_event(self, event: Event) -> bool:
        """Validate event against registered schema"""
        key = f"{event.event_type}:{event.version}"
        schema = self.schema_registry.get(key)
        
        if not schema:
            print(f"No schema found for {key}")
            return False
        
        # Simple validation (in practice, use jsonschema)
        required_fields = schema.get('required', [])
        for field in required_fields:
            if field not in event.data:
                print(f"Missing required field: {field}")
                return False
        
        return True
    
    def migrate_event(self, event: Event, target_version: str) -> Event:
        """Migrate event to target version"""
        # Implementation would depend on specific migration rules
        # This is a simplified example
        
        if event.version == "1.0" and target_version == "2.0":
            # Example migration: add default currency field
            migrated_data = event.data.copy()
            if 'currency' not in migrated_data:
                migrated_data['currency'] = 'USD'
            
            return Event(
                event_id=event.event_id,
                event_type=event.event_type,
                source=event.source,
                timestamp=event.timestamp,
                data=migrated_data,
                version=target_version
            )
        
        return event

# Setup schema registry
versioning = EventVersioning()

# Register schemas
versioning.register_schema(
    "order.created",
    "1.0",
    {
        'required': ['order_id', 'user_id', 'items', 'total_amount'],
        'properties': {
            'order_id': {'type': 'string'},
            'user_id': {'type': 'string'},
            'items': {'type': 'array'},
            'total_amount': {'type': 'number'}
        }
    }
)

versioning.register_schema(
    "order.created",
    "2.0",
    {
        'required': ['order_id', 'user_id', 'items', 'total_amount', 'currency'],
        'properties': {
            'order_id': {'type': 'string'},
            'user_id': {'type': 'string'},
            'items': {'type': 'array'},
            'total_amount': {'type': 'number'},
            'currency': {'type': 'string'}
        }
    }
)
```

### 2. Error Handling and Resilience

```python
class ResilientEventConsumer(EventConsumer):
    def __init__(self, event_bus, max_retries=3, retry_delay=1.0):
        super().__init__(event_bus)
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.dead_letter_queue = []
    
    def handle_event_with_retry(self, event: Event):
        """Handle event with retry logic"""
        for attempt in range(self.max_retries + 1):
            try:
                self.handle_event(event)
                return  # Success
            except Exception as e:
                if attempt < self.max_retries:
                    print(f"Attempt {attempt + 1} failed for event {event.event_id}: {e}")
                    time.sleep(self.retry_delay * (2 ** attempt))  # Exponential backoff
                else:
                    print(f"All attempts failed for event {event.event_id}: {e}")
                    self._send_to_dead_letter_queue(event, str(e))
    
    def _send_to_dead_letter_queue(self, event: Event, error_message: str):
        """Send failed event to dead letter queue"""
        dead_letter_event = {
            'original_event': event.to_dict(),
            'error_message': error_message,
            'failed_at': datetime.now().isoformat(),
            'retry_count': self.max_retries
        }
        
        self.dead_letter_queue.append(dead_letter_event)
        
        # In production, send to actual dead letter queue (e.g., separate Kafka topic)
        print(f"Event {event.event_id} sent to dead letter queue")
    
    def process_dead_letter_queue(self):
        """Process events from dead letter queue"""
        for dead_letter_event in self.dead_letter_queue:
            # Could implement manual review, automatic retry after delay, etc.
            print(f"Dead letter event: {dead_letter_event}")

class CircuitBreakerEventConsumer(EventConsumer):
    def __init__(self, event_bus, failure_threshold=5, recovery_timeout=60):
        super().__init__(event_bus)
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def handle_event(self, event: Event):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            # Process event
            super().handle_event(event)
            
            if self.state == 'HALF_OPEN':
                self.state = 'CLOSED'
                self.failure_count = 0
                
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = 'OPEN'
                print(f"Circuit breaker opened after {self.failure_count} failures")
            
            raise e
```

## Interview Questions & Answers

### Basic Questions

**Q1: What is Event-Driven Architecture and what are its main benefits?**

**Answer:** Event-Driven Architecture (EDA) is a design pattern where components communicate through the production and consumption of events. An event represents a significant change in state or occurrence.

**Main benefits:**
- **Loose Coupling**: Producers and consumers are independent
- **Scalability**: Components can be scaled independently
- **Real-time Processing**: Events can be processed as they occur
- **Resilience**: System continues operating if some components fail
- **Auditability**: Complete event history provides audit trail

**Example:**
```python
# Producer doesn't know about consumers
class OrderService:
    def create_order(self, order_data):
        order_id = self.save_order(order_data)
        
        # Just publish event - doesn't care who listens
        event = Event("order.created", {"order_id": order_id})
        self.event_bus.publish(event)
        
        return order_id

# Multiple consumers can react independently
class EmailService:
    def handle_order_created(self, event):
        self.send_confirmation_email(event.data["order_id"])

class InventoryService:
    def handle_order_created(self, event):
        self.reserve_items(event.data["order_id"])
```

**Q2: Explain the difference between Event Sourcing and traditional CRUD operations.**

**Answer:**

**Traditional CRUD:**
- Stores current state only
- Updates overwrite previous data
- No history of changes
- Difficult to audit or replay

**Event Sourcing:**
- Stores sequence of events (changes)
- Current state derived from events
- Complete history preserved
- Can replay events to rebuild state

**Comparison:**
```python
# Traditional CRUD
class BankAccount:
    def __init__(self, account_id):
        self.account_id = account_id
        self.balance = 0  # Only current state
    
    def deposit(self, amount):
        self.balance += amount  # Overwrites previous state
        self.save_to_database()

# Event Sourcing
class EventSourcedBankAccount:
    def __init__(self, account_id):
        self.account_id = account_id
        self.events = []  # Store all changes
        self.balance = 0
    
    def deposit(self, amount):
        event = DepositEvent(amount, timestamp=now())
        self.events.append(event)  # Append, don't overwrite
        self.balance += amount  # Derived state
    
    @classmethod
    def from_events(cls, events):
        account = cls()
        for event in events:
            account.apply_event(event)  # Rebuild state
        return account
```

**Q3: What is CQRS and how does it relate to Event-Driven Architecture?**

**Answer:** CQRS (Command Query Responsibility Segregation) separates read and write models, often used with EDA.

**Key concepts:**
- **Commands**: Change state (writes)
- **Queries**: Read state (reads)
- **Separate models**: Different data structures for reads vs writes

**Implementation:**
```python
# Command side (Write model)
class CreateOrderCommand:
    def __init__(self, user_id, items):
        self.user_id = user_id
        self.items = items

class OrderCommandHandler:
    def handle(self, command):
        order = Order(command.user_id, command.items)
        self.repository.save(order)
        
        # Publish event
        event = Event("order.created", order.to_dict())
        self.event_bus.publish(event)

# Query side (Read model)
class OrderSummaryProjection:
    def __init__(self):
        self.summaries = {}  # Optimized for reads
    
    def handle_order_created(self, event):
        # Update read model
        order_data = event.data
        self.summaries[order_data['order_id']] = {
            'order_id': order_data['order_id'],
            'status': 'created',
            'total': order_data['total']
        }

class OrderQueryHandler:
    def get_order_summary(self, order_id):
        return self.projection.summaries.get(order_id)
```

### Intermediate Questions

**Q4: How do you handle event ordering and ensure events are processed in the correct sequence?**

**Answer:** Several strategies for event ordering:

**1. Single Partition/Queue:**
```python
class OrderedEventBus:
    def __init__(self):
        self.queue = queue.Queue()  # Single queue ensures order
    
    def publish(self, event):
        self.queue.put(event)
    
    def process_events(self):
        while True:
            event = self.queue.get()
            self.dispatch_event(event)  # Processed in order
```

**2. Partition by Entity:**
```python
class PartitionedEventBus:
    def __init__(self, num_partitions=10):
        self.partitions = [queue.Queue() for _ in range(num_partitions)]
    
    def publish(self, event):
        # Events for same entity go to same partition
        partition_key = hash(event.aggregate_id) % len(self.partitions)
        self.partitions[partition_key].put(event)
    
    def process_partition(self, partition_id):
        partition = self.partitions[partition_id]
        while True:
            event = partition.get()
            self.dispatch_event(event)  # Order preserved per entity
```

**3. Sequence Numbers:**
```python
class SequencedEvent(Event):
    def __init__(self, event_type, data, sequence_number):
        super().__init__(event_type, data)
        self.sequence_number = sequence_number

class SequenceTracker:
    def __init__(self):
        self.expected_sequence = {}  # entity_id -> next expected sequence
        self.pending_events = {}     # entity_id -> list of out-of-order events
    
    def process_event(self, event):
        entity_id = event.aggregate_id
        expected = self.expected_sequence.get(entity_id, 1)
        
        if event.sequence_number == expected:
            # Process in-order event
            self.handle_event(event)
            self.expected_sequence[entity_id] = expected + 1
            
            # Check for pending events that can now be processed
            self.process_pending_events(entity_id)
        else:
            # Store out-of-order event
            if entity_id not in self.pending_events:
                self.pending_events[entity_id] = []
            self.pending_events[entity_id].append(event)
```

**Q5: How do you implement the Saga pattern for distributed transactions?**

**Answer:** Saga pattern manages distributed transactions through a sequence of compensatable steps:

**Choreography-based Saga:**
```python
class OrderSaga:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.event_bus.subscribe("order.created", self.handle_order_created)
        self.event_bus.subscribe("payment.failed", self.handle_payment_failed)
        self.event_bus.subscribe("inventory.reserved", self.handle_inventory_reserved)
    
    def handle_order_created(self, event):
        # Step 1: Reserve inventory
        reserve_event = Event("inventory.reserve", {
            "order_id": event.data["order_id"],
            "items": event.data["items"]
        })
        self.event_bus.publish(reserve_event)
    
    def handle_inventory_reserved(self, event):
        # Step 2: Process payment
        payment_event = Event("payment.process", {
            "order_id": event.data["order_id"],
            "amount": event.data["amount"]
        })
        self.event_bus.publish(payment_event)
    
    def handle_payment_failed(self, event):
        # Compensate: Release inventory
        compensate_event = Event("inventory.release", {
            "order_id": event.data["order_id"]
        })
        self.event_bus.publish(compensate_event)
```

**Orchestration-based Saga:**
```python
class OrderSagaOrchestrator:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.active_sagas = {}
    
    def start_order_saga(self, order_data):
        saga_id = f"order_saga_{order_data['order_id']}"
        saga = OrderSagaState(saga_id, order_data)
        self.active_sagas[saga_id] = saga
        
        # Execute first step
        self.execute_next_step(saga)
    
    def execute_next_step(self, saga):
        next_step = saga.get_next_step()
        if next_step:
            command = next_step.create_command()
            try:
                self.execute_command(command)
                saga.mark_step_completed()
                self.execute_next_step(saga)  # Continue to next step
            except Exception as e:
                saga.start_compensation()
                self.execute_compensation(saga)
    
    def execute_compensation(self, saga):
        compensation = saga.get_next_compensation()
        if compensation:
            try:
                self.execute_command(compensation)
                saga.mark_compensation_completed()
                self.execute_compensation(saga)  # Continue compensation
            except Exception as e:
                # Handle compensation failure
                self.handle_compensation_failure(saga, e)
```

**Q6: What are the challenges of event versioning and schema evolution?**

**Answer:** Event versioning challenges and solutions:

**Challenges:**
- **Backward compatibility**: Old consumers must handle new event versions
- **Forward compatibility**: New consumers must handle old event versions
- **Schema migration**: Evolving event structure over time
- **Consumer coordination**: Multiple consumers may expect different versions

**Solutions:**

**1. Additive Changes Only:**
```python
# Version 1.0
{
    "event_type": "user.created",
    "version": "1.0",
    "data": {
        "user_id": "123",
        "email": "user@example.com"
    }
}

# Version 2.0 - Add optional fields
{
    "event_type": "user.created", 
    "version": "2.0",
    "data": {
        "user_id": "123",
        "email": "user@example.com",
        "name": "John Doe",        # New optional field
        "preferences": {}          # New optional field
    }
}
```

**2. Event Upcasting:**
```python
class EventUpcast:
    def upcast_event(self, event):
        if event.version == "1.0" and event.event_type == "user.created":
            # Migrate v1.0 to v2.0
            return Event(
                event_type=event.event_type,
                version="2.0",
                data={
                    **event.data,
                    "name": "",  # Default value
                    "preferences": {}
                }
            )
        return event
```

**3. Multiple Event Versions:**
```python
class VersionedEventHandler:
    def handle_event(self, event):
        handler_method = f"handle_{event.event_type.replace('.', '_')}_v{event.version.replace('.', '_')}"
        
        if hasattr(self, handler_method):
            getattr(self, handler_method)(event)
        else:
            # Fallback to generic handler
            self.handle_generic_event(event)
    
    def handle_user_created_v1_0(self, event):
        # Handle version 1.0 specifically
        pass
    
    def handle_user_created_v2_0(self, event):
        # Handle version 2.0 specifically
        pass
```

### Advanced Questions

**Q7: Design an event-driven system for a real-time trading platform.**

**Answer:** Real-time trading platform with strict latency and consistency requirements:

```python
class TradingEventSystem:
    def __init__(self):
        # Ultra-low latency event bus
        self.event_bus = InMemoryEventBus()  # No network overhead
        
        # Services
        self.market_data_service = MarketDataService(self.event_bus)
        self.order_service = OrderService(self.event_bus)
        self.risk_service = RiskService(self.event_bus)
        self.execution_service = ExecutionService(self.event_bus)
        self.position_service = PositionService(self.event_bus)
        
        # Event sourcing for audit trail
        self.event_store = HighPerformanceEventStore()

class MarketDataService(EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.current_prices = {}
    
    def update_price(self, symbol, price, volume):
        old_price = self.current_prices.get(symbol)
        self.current_prices[symbol] = price
        
        # Publish price update event
        event = Event(
            event_type="market.price_updated",
            source="market-data",
            timestamp=datetime.now(),
            data={
                'symbol': symbol,
                'price': price,
                'volume': volume,
                'previous_price': old_price,
                'change': price - old_price if old_price else 0
            }
        )
        
        self.produce_event(event)

class OrderService(EventConsumer, EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.orders = {}
        self.subscribe("risk.approved")
        self.subscribe("execution.filled")
    
    def place_order(self, order_data):
        order_id = f"order_{int(time.time() * 1000000)}"  # Microsecond precision
        
        order = {
            'order_id': order_id,
            'symbol': order_data['symbol'],
            'side': order_data['side'],  # BUY/SELL
            'quantity': order_data['quantity'],
            'price': order_data['price'],
            'status': 'pending_risk_check',
            'timestamp': datetime.now()
        }
        
        self.orders[order_id] = order
        
        # Publish order created event
        event = Event(
            event_type="order.created",
            source="order-service",
            timestamp=datetime.now(),
            data=order
        )
        
        self.produce_event(event)
        return order_id
    
    def handle_event(self, event):
        if event.event_type == "risk.approved":
            self._send_to_execution(event)
        elif event.event_type == "execution.filled":
            self._update_order_status(event)

class RiskService(EventConsumer, EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("order.created")
        self.position_limits = {}
        self.current_positions = {}
    
    def handle_event(self, event):
        if event.event_type == "order.created":
            self._check_risk(event)
    
    def _check_risk(self, event):
        order_data = event.data
        
        # Check position limits
        symbol = order_data['symbol']
        quantity = order_data['quantity']
        side = order_data['side']
        
        current_position = self.current_positions.get(symbol, 0)
        new_position = current_position + (quantity if side == 'BUY' else -quantity)
        
        limit = self.position_limits.get(symbol, 1000000)
        
        if abs(new_position) <= limit:
            # Risk approved
            approval_event = Event(
                event_type="risk.approved",
                source="risk-service",
                timestamp=datetime.now(),
                data={
                    'order_id': order_data['order_id'],
                    'approved': True
                }
            )
        else:
            # Risk rejected
            approval_event = Event(
                event_type="risk.rejected",
                source="risk-service", 
                timestamp=datetime.now(),
                data={
                    'order_id': order_data['order_id'],
                    'reason': 'Position limit exceeded'
                }
            )
        
        self.produce_event(approval_event)

class ExecutionService(EventConsumer, EventProducer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("risk.approved")
        self.subscribe("market.price_updated")
        self.pending_orders = {}
    
    def handle_event(self, event):
        if event.event_type == "risk.approved":
            self._queue_for_execution(event)
        elif event.event_type == "market.price_updated":
            self._check_executable_orders(event)
    
    def _queue_for_execution(self, event):
        order_id = event.data['order_id']
        self.pending_orders[order_id] = event.data
    
    def _check_executable_orders(self, market_event):
        symbol = market_event.data['symbol']
        current_price = market_event.data['price']
        
        # Check if any pending orders can be executed
        for order_id, order_data in list(self.pending_orders.items()):
            if (order_data['symbol'] == symbol and 
                self._can_execute(order_data, current_price)):
                
                self._execute_order(order_id, order_data, current_price)
                del self.pending_orders[order_id]
    
    def _execute_order(self, order_id, order_data, execution_price):
        fill_event = Event(
            event_type="execution.filled",
            source="execution-service",
            timestamp=datetime.now(),
            data={
                'order_id': order_id,
                'symbol': order_data['symbol'],
                'side': order_data['side'],
                'quantity': order_data['quantity'],
                'execution_price': execution_price,
                'execution_time': datetime.now()
            }
        )
        
        self.produce_event(fill_event)
```

**Q8: How do you implement event replay and system recovery?**

**Answer:** Event replay strategies for system recovery:

**1. Full System Replay:**
```python
class EventReplayService:
    def __init__(self, event_store, event_bus):
        self.event_store = event_store
        self.event_bus = event_bus
    
    def replay_all_events(self, from_timestamp=None):
        """Replay all events from a specific point in time"""
        events = self.event_store.get_events(from_timestamp=from_timestamp)
        
        print(f"Replaying {len(events)} events...")
        
        for event in events:
            try:
                self.event_bus.publish(event)
                time.sleep(0.001)  # Small delay to prevent overwhelming
            except Exception as e:
                print(f"Error replaying event {event.event_id}: {e}")
    
    def replay_for_aggregate(self, aggregate_id):
        """Replay events for specific aggregate"""
        events = self.event_store.get_events_for_aggregate(aggregate_id)
        
        for event in events:
            self.event_bus.publish(event)
```

**2. Snapshot + Incremental Replay:**
```python
class SnapshotReplayService:
    def __init__(self, event_store, snapshot_store):
        self.event_store = event_store
        self.snapshot_store = snapshot_store
    
    def rebuild_aggregate(self, aggregate_id):
        """Rebuild aggregate from snapshot + events"""
        
        # Get latest snapshot
        snapshot = self.snapshot_store.get_latest_snapshot(aggregate_id)
        
        if snapshot:
            # Start from snapshot
            aggregate = self.deserialize_aggregate(snapshot)
            from_version = snapshot.version
        else:
            # No snapshot, start from beginning
            aggregate = self.create_empty_aggregate(aggregate_id)
            from_version = 0
        
        # Apply events since snapshot
        events = self.event_store.get_events_for_aggregate(
            aggregate_id, 
            from_version=from_version
        )
        
        for event in events:
            aggregate.apply_event(event)
        
        return aggregate
    
    def create_snapshot(self, aggregate):
        """Create snapshot of current aggregate state"""
        snapshot = AggregateSnapshot(
            aggregate_id=aggregate.id,
            version=aggregate.version,
            data=aggregate.serialize(),
            timestamp=datetime.now()
        )
        
        self.snapshot_store.save_snapshot(snapshot)
```

**3. Projection Rebuilding:**
```python
class ProjectionRebuilder:
    def __init__(self, event_store):
        self.event_store = event_store
        self.projections = {}
    
    def register_projection(self, name, projection):
        self.projections[name] = projection
    
    def rebuild_projection(self, projection_name):
        """Rebuild specific projection from events"""
        projection = self.projections[projection_name]
        
        # Reset projection state
        projection.reset()
        
        # Get all relevant events
        event_types = projection.get_handled_event_types()
        events = self.event_store.get_events(event_types=event_types)
        
        # Apply events to rebuild projection
        for event in events:
            projection.handle_event(event)
        
        print(f"Rebuilt projection {projection_name} with {len(events)} events")
    
    def rebuild_all_projections(self):
        """Rebuild all registered projections"""
        for name in self.projections:
            self.rebuild_projection(name)
```

**Q9: How do you handle event deduplication and idempotency?**

**Answer:** Strategies for ensuring events are processed exactly once:

**1. Event Deduplication:**
```python
class DeduplicatingEventBus:
    def __init__(self, storage):
        self.storage = storage
        self.processed_events = set()
        self.subscribers = defaultdict(list)
    
    def publish(self, event):
        # Check if event already processed
        if event.event_id in self.processed_events:
            print(f"Event {event.event_id} already processed, skipping")
            return
        
        # Store event ID to prevent reprocessing
        self.processed_events.add(event.event_id)
        self.storage.store_processed_event(event.event_id)
        
        # Dispatch to subscribers
        self._dispatch_event(event)
    
    def _dispatch_event(self, event):
        handlers = self.subscribers.get(event.event_type, [])
        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                # Remove from processed set if handler fails
                self.processed_events.discard(event.event_id)
                self.storage.remove_processed_event(event.event_id)
                raise e
```

**2. Idempotent Event Handlers:**
```python
class IdempotentEventHandler:
    def __init__(self):
        self.processed_events = set()
    
    def handle_event(self, event):
        # Check if already processed
        if event.event_id in self.processed_events:
            return  # Skip processing
        
        try:
            # Process event
            self._process_event(event)
            
            # Mark as processed
            self.processed_events.add(event.event_id)
            
        except Exception as e:
            # Don't mark as processed if failed
            print(f"Failed to process event {event.event_id}: {e}")
            raise
    
    def _process_event(self, event):
        # Idempotent business logic
        if event.event_type == "order.created":
            order_id = event.data['order_id']
            
            # Check if order already exists (idempotent check)
            if not self.order_exists(order_id):
                self.create_order(event.data)
```

**3. At-Least-Once with Idempotent Operations:**
```python
class BankAccountEventHandler:
    def __init__(self):
        self.accounts = {}
    
    def handle_deposit_event(self, event):
        account_id = event.data['account_id']
        amount = event.data['amount']
        transaction_id = event.data['transaction_id']  # Unique transaction ID
        
        account = self.accounts.get(account_id, {'balance': 0, 'transactions': set()})
        
        # Idempotent check using transaction ID
        if transaction_id in account['transactions']:
            print(f"Transaction {transaction_id} already processed")
            return
        
        # Process deposit
        account['balance'] += amount
        account['transactions'].add(transaction_id)
        
        self.accounts[account_id] = account
        print(f"Processed deposit: {amount} to account {account_id}")
```

**Q10: How do you monitor and debug event-driven systems?**

**Answer:** Comprehensive monitoring and debugging strategies:

**1. Event Tracing:**
```python
class EventTracer:
    def __init__(self):
        self.traces = {}  # correlation_id -> list of events
    
    def trace_event(self, event, correlation_id=None):
        if not correlation_id:
            correlation_id = event.data.get('correlation_id', event.event_id)
        
        if correlation_id not in self.traces:
            self.traces[correlation_id] = []
        
        self.traces[correlation_id].append({
            'event_id': event.event_id,
            'event_type': event.event_type,
            'timestamp': event.timestamp,
            'source': event.source,
            'data': event.data
        })
    
    def get_trace(self, correlation_id):
        return self.traces.get(correlation_id, [])
    
    def visualize_trace(self, correlation_id):
        trace = self.get_trace(correlation_id)
        
        print(f"Event Trace for {correlation_id}:")
        for i, event in enumerate(trace):
            print(f"{i+1}. {event['timestamp']} - {event['event_type']} from {event['source']}")
```

**2. Event Metrics:**
```python
class EventMetrics:
    def __init__(self):
        self.metrics = {
            'events_published': defaultdict(int),
            'events_processed': defaultdict(int),
            'processing_times': defaultdict(list),
            'errors': defaultdict(int)
        }
    
    def record_event_published(self, event_type):
        self.metrics['events_published'][event_type] += 1
    
    def record_event_processed(self, event_type, processing_time):
        self.metrics['events_processed'][event_type] += 1
        self.metrics['processing_times'][event_type].append(processing_time)
    
    def record_error(self, event_type):
        self.metrics['errors'][event_type] += 1
    
    def get_summary(self):
        summary = {}
        
        for event_type in self.metrics['events_published']:
            published = self.metrics['events_published'][event_type]
            processed = self.metrics['events_processed'][event_type]
            errors = self.metrics['errors'][event_type]
            
            processing_times = self.metrics['processing_times'][event_type]
            avg_time = sum(processing_times) / len(processing_times) if processing_times else 0
            
            summary[event_type] = {
                'published': published,
                'processed': processed,
                'errors': errors,
                'success_rate': (processed / published) * 100 if published > 0 else 0,
                'avg_processing_time_ms': avg_time * 1000
            }
        
        return summary
```

**3. Event Flow Visualization:**
```python
class EventFlowVisualizer:
    def __init__(self):
        self.flows = defaultdict(set)  # event_type -> set of triggered event_types
    
    def record_flow(self, trigger_event_type, resulting_event_type):
        self.flows[trigger_event_type].add(resulting_event_type)
    
    def generate_flow_diagram(self):
        """Generate DOT format for Graphviz"""
        dot = "digraph EventFlow {\n"
        
        for trigger, results in self.flows.items():
            for result in results:
                dot += f'  "{trigger}" -> "{result}";\n'
        
        dot += "}"
        return dot
    
    def detect_cycles(self):
        """Detect potential infinite loops in event flows"""
        visited = set()
        rec_stack = set()
        cycles = []
        
        def dfs(node, path):
            if node in rec_stack:
                cycle_start = path.index(node)
                cycles.append(path[cycle_start:] + [node])
                return
            
            if node in visited:
                return
            
            visited.add(node)
            rec_stack.add(node)
            
            for neighbor in self.flows.get(node, []):
                dfs(neighbor, path + [node])
            
            rec_stack.remove(node)
        
        for event_type in self.flows:
            if event_type not in visited:
                dfs(event_type, [])
        
        return cycles
```

## Best Practices Summary

1. **Event Design**: Use clear naming, include all necessary data, version events
2. **Error Handling**: Implement retry logic, dead letter queues, circuit breakers
3. **Ordering**: Consider ordering requirements, use partitioning when needed
4. **Idempotency**: Design handlers to be idempotent, use unique identifiers
5. **Monitoring**: Implement comprehensive tracing, metrics, and alerting
6. **Testing**: Test event flows, failure scenarios, and recovery procedures
7. **Documentation**: Document event schemas, flows, and dependencies

## Further Reading

- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Event-Driven Architecture](https://aws.amazon.com/event-driven-architecture/)
