# Singleton Pattern 🔒

## 📖 Definition
The Singleton pattern ensures that a class has only one instance and provides a global point of access to that instance.

## 🎯 When to Use
- Database connections
- Logging services
- Configuration settings
- Cache management
- Thread pools

## ✅ Pros
- Controlled access to sole instance
- Reduced memory footprint
- Global access point
- Lazy initialization possible

## ❌ Cons
- Difficult to test
- Violates Single Responsibility Principle
- Can create tight coupling
- Thread safety issues in multi-threaded environments

## 🚀 JavaScript Implementations

### 1. Basic Singleton (ES5)
```javascript
function Singleton() {
    if (Singleton.instance) {
        return Singleton.instance;
    }
    
    this.data = [];
    this.timestamp = Date.now();
    
    Singleton.instance = this;
    return this;
}

// Usage
const instance1 = new Singleton();
const instance2 = new Singleton();
console.log(instance1 === instance2); // true
```

### 2. Module Pattern Singleton
```javascript
const Singleton = (function() {
    let instance;
    
    function createInstance() {
        return {
            data: [],
            timestamp: Date.now(),
            
            addData(item) {
                this.data.push(item);
            },
            
            getData() {
                return this.data;
            }
        };
    }
    
    return {
        getInstance() {
            if (!instance) {
                instance = createInstance();
            }
            return instance;
        }
    };
})();

// Usage
const instance1 = Singleton.getInstance();
const instance2 = Singleton.getInstance();
console.log(instance1 === instance2); // true
```

### 3. ES6 Class Singleton
```javascript
class Singleton {
    constructor() {
        if (Singleton.instance) {
            return Singleton.instance;
        }
        
        this.data = [];
        this.timestamp = Date.now();
        
        Singleton.instance = this;
    }
    
    addData(item) {
        this.data.push(item);
    }
    
    getData() {
        return this.data;
    }
    
    static getInstance() {
        if (!Singleton.instance) {
            Singleton.instance = new Singleton();
        }
        return Singleton.instance;
    }
}

// Usage
const instance1 = new Singleton();
const instance2 = Singleton.getInstance();
console.log(instance1 === instance2); // true
```

### 4. Modern ES6+ with Private Fields
```javascript
class Singleton {
    static #instance;
    #data = [];
    #timestamp = Date.now();
    
    constructor() {
        if (Singleton.#instance) {
            return Singleton.#instance;
        }
        Singleton.#instance = this;
    }
    
    addData(item) {
        this.#data.push(item);
    }
    
    getData() {
        return [...this.#data]; // Return copy to maintain encapsulation
    }
    
    static getInstance() {
        if (!Singleton.#instance) {
            Singleton.#instance = new Singleton();
        }
        return Singleton.#instance;
    }
}
```

## 🌟 Real-World Examples

### Database Connection Manager
```javascript
class DatabaseConnection {
    static #instance;
    #connection = null;
    
    constructor() {
        if (DatabaseConnection.#instance) {
            return DatabaseConnection.#instance;
        }
        DatabaseConnection.#instance = this;
    }
    
    async connect() {
        if (!this.#connection) {
            // Simulate database connection
            this.#connection = {
                host: 'localhost',
                port: 5432,
                connected: true,
                connectionTime: Date.now()
            };
            console.log('Database connected');
        }
        return this.#connection;
    }
    
    async query(sql) {
        if (!this.#connection) {
            await this.connect();
        }
        // Simulate query execution
        return { sql, result: 'Query executed', timestamp: Date.now() };
    }
    
    disconnect() {
        if (this.#connection) {
            this.#connection.connected = false;
            this.#connection = null;
            console.log('Database disconnected');
        }
    }
}

// Usage
const db1 = new DatabaseConnection();
const db2 = new DatabaseConnection();
console.log(db1 === db2); // true

await db1.connect();
const result = await db2.query('SELECT * FROM users');
```

### Logger Service
```javascript
class Logger {
    static #instance;
    #logs = [];
    #logLevel = 'INFO';
    
    constructor() {
        if (Logger.#instance) {
            return Logger.#instance;
        }
        Logger.#instance = this;
    }
    
    setLogLevel(level) {
        this.#logLevel = level;
    }
    
    log(message, level = 'INFO') {
        const logEntry = {
            timestamp: new Date().toISOString(),
            level,
            message
        };
        
        this.#logs.push(logEntry);
        
        if (this.#shouldLog(level)) {
            console.log(`[${logEntry.timestamp}] ${level}: ${message}`);
        }
    }
    
    #shouldLog(level) {
        const levels = { ERROR: 0, WARN: 1, INFO: 2, DEBUG: 3 };
        return levels[level] <= levels[this.#logLevel];
    }
    
    getLogs() {
        return [...this.#logs];
    }
    
    error(message) { this.log(message, 'ERROR'); }
    warn(message) { this.log(message, 'WARN'); }
    info(message) { this.log(message, 'INFO'); }
    debug(message) { this.log(message, 'DEBUG'); }
}

// Usage across different modules
const logger1 = new Logger();
const logger2 = new Logger();

logger1.info('Application started');
logger2.error('Something went wrong');

console.log(logger1 === logger2); // true
console.log(logger1.getLogs()); // Contains both log entries
```

## 🧪 Testing Considerations

### Problem with Testing
```javascript
// Difficult to test because of shared state
describe('Singleton Tests', () => {
    it('should maintain state between tests', () => {
        const instance1 = Singleton.getInstance();
        instance1.addData('test1');
        
        const instance2 = Singleton.getInstance();
        expect(instance2.getData()).toContain('test1'); // This might fail if previous tests modified state
    });
});
```

### Solution: Reset Method
```javascript
class TestableSingleton {
    static #instance;
    #data = [];
    
    constructor() {
        if (TestableSingleton.#instance) {
            return TestableSingleton.#instance;
        }
        TestableSingleton.#instance = this;
    }
    
    // Add reset method for testing
    static reset() {
        TestableSingleton.#instance = null;
    }
    
    addData(item) {
        this.#data.push(item);
    }
    
    getData() {
        return [...this.#data];
    }
}

// In tests
afterEach(() => {
    TestableSingleton.reset();
});
```

## 🎯 Interview Questions

### Q1: What are the drawbacks of Singleton pattern?
**Answer:** 
- Makes unit testing difficult due to shared state
- Violates Single Responsibility Principle
- Creates tight coupling between classes
- Can cause memory leaks if not properly managed
- Difficult to extend or modify

### Q2: How do you implement thread-safe Singleton in JavaScript?
**Answer:** JavaScript is single-threaded, but with Web Workers and async operations, you might need to consider:
```javascript
class ThreadSafeSingleton {
    static #instance;
    static #creating = false;
    
    constructor() {
        if (ThreadSafeSingleton.#creating) {
            throw new Error('Use getInstance() method');
        }
        if (ThreadSafeSingleton.#instance) {
            return ThreadSafeSingleton.#instance;
        }
        ThreadSafeSingleton.#instance = this;
    }
    
    static async getInstance() {
        if (!ThreadSafeSingleton.#instance && !ThreadSafeSingleton.#creating) {
            ThreadSafeSingleton.#creating = true;
            ThreadSafeSingleton.#instance = new ThreadSafeSingleton();
            ThreadSafeSingleton.#creating = false;
        }
        return ThreadSafeSingleton.#instance;
    }
}
```

### Q3: What's the difference between Singleton and Module pattern?
**Answer:**
- **Singleton**: Ensures only one instance of a class
- **Module**: Provides encapsulation and namespace, can have multiple instances
- **Singleton** focuses on instance control, **Module** focuses on organization

### Q4: When should you avoid using Singleton?
**Answer:**
- When you need multiple instances with different configurations
- When testing is a priority (hard to mock/stub)
- When the class has multiple responsibilities
- When you need inheritance or polymorphism

## 💡 Best Practices

1. **Use sparingly** - Only when you truly need one instance
2. **Consider alternatives** - Dependency injection, factory patterns
3. **Make it testable** - Provide reset methods for testing
4. **Use private fields** - Ensure proper encapsulation
5. **Document the reason** - Why singleton is necessary
6. **Consider lazy initialization** - Create instance only when needed

## 🔗 Related Patterns
- **Factory Pattern** - Can be used to create singleton instances
- **Builder Pattern** - Can implement singleton builder
- **Dependency Injection** - Alternative to singleton for managing dependencies
