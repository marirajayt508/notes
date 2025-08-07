# Closure Patterns 🔒

## 📖 Definition
Closures are functions that have access to variables from their outer (enclosing) scope even after the outer function has returned. They're fundamental to many JavaScript patterns and provide powerful ways to create private variables and maintain state.

## 🎯 When to Use
- When you need private variables and methods
- When creating function factories
- When implementing callbacks with persistent state
- When building modules with encapsulation
- When creating decorators and higher-order functions

## ✅ Pros
- True privacy and encapsulation
- Persistent state without global variables
- Elegant function composition
- Memory efficient when used properly
- Foundation for many advanced patterns

## ❌ Cons
- Can cause memory leaks if not handled properly
- Can be confusing for beginners
- Debugging can be challenging
- Performance overhead in some cases

## 🚀 JavaScript Implementations

### 1. Basic Closure Pattern
```javascript
function createCounter() {
    let count = 0; // Private variable
    
    return function() {
        count++; // Access to outer scope
        return count;
    };
}

const counter1 = createCounter();
const counter2 = createCounter();

console.log(counter1()); // 1
console.log(counter1()); // 2
console.log(counter2()); // 1 (independent counter)
console.log(counter1()); // 3

// count is not accessible from outside
console.log(counter1.count); // undefined
```

### 2. Closure with Multiple Methods
```javascript
function createBankAccount(initialBalance = 0) {
    let balance = initialBalance;
    let transactionHistory = [];
    
    function addTransaction(type, amount) {
        transactionHistory.push({
            type,
            amount,
            balance: balance,
            timestamp: new Date()
        });
    }
    
    return {
        deposit(amount) {
            if (amount <= 0) {
                throw new Error('Deposit amount must be positive');
            }
            balance += amount;
            addTransaction('deposit', amount);
            return balance;
        },
        
        withdraw(amount) {
            if (amount <= 0) {
                throw new Error('Withdrawal amount must be positive');
            }
            if (amount > balance) {
                throw new Error('Insufficient funds');
            }
            balance -= amount;
            addTransaction('withdrawal', amount);
            return balance;
        },
        
        getBalance() {
            return balance;
        },
        
        getTransactionHistory() {
            return [...transactionHistory]; // Return copy to maintain privacy
        },
        
        getAccountSummary() {
            const totalDeposits = transactionHistory
                .filter(t => t.type === 'deposit')
                .reduce((sum, t) => sum + t.amount, 0);
                
            const totalWithdrawals = transactionHistory
                .filter(t => t.type === 'withdrawal')
                .reduce((sum, t) => sum + t.amount, 0);
                
            return {
                currentBalance: balance,
                totalDeposits,
                totalWithdrawals,
                transactionCount: transactionHistory.length
            };
        }
    };
}

// Usage
const account = createBankAccount(100);

account.deposit(50);
account.withdraw(30);
account.deposit(25);

console.log('Balance:', account.getBalance()); // 145
console.log('Summary:', account.getAccountSummary());
console.log('History:', account.getTransactionHistory());

// Private variables are not accessible
console.log(account.balance); // undefined
console.log(account.transactionHistory); // undefined
```

### 3. Function Factory Pattern
```javascript
function createValidator(rules) {
    const validationRules = { ...rules };
    
    return function validate(data) {
        const errors = [];
        
        Object.keys(validationRules).forEach(field => {
            const rule = validationRules[field];
            const value = data[field];
            
            // Required check
            if (rule.required && (value === undefined || value === null || value === '')) {
                errors.push(`${field} is required`);
                return;
            }
            
            // Skip other validations if field is empty and not required
            if (!rule.required && (value === undefined || value === null || value === '')) {
                return;
            }
            
            // Type validation
            if (rule.type && typeof value !== rule.type) {
                errors.push(`${field} must be of type ${rule.type}`);
            }
            
            // Min length validation
            if (rule.minLength && value.length < rule.minLength) {
                errors.push(`${field} must be at least ${rule.minLength} characters long`);
            }
            
            // Max length validation
            if (rule.maxLength && value.length > rule.maxLength) {
                errors.push(`${field} must be no more than ${rule.maxLength} characters long`);
            }
            
            // Pattern validation
            if (rule.pattern && !rule.pattern.test(value)) {
                errors.push(`${field} format is invalid`);
            }
            
            // Custom validation
            if (rule.custom && typeof rule.custom === 'function') {
                const customResult = rule.custom(value);
                if (customResult !== true) {
                    errors.push(customResult || `${field} is invalid`);
                }
            }
        });
        
        return {
            isValid: errors.length === 0,
            errors
        };
    };
}

// Create specific validators
const userValidator = createValidator({
    name: {
        required: true,
        type: 'string',
        minLength: 2,
        maxLength: 50
    },
    email: {
        required: true,
        type: 'string',
        pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    },
    age: {
        required: false,
        type: 'number',
        custom: (value) => value >= 0 && value <= 120 ? true : 'Age must be between 0 and 120'
    }
});

const productValidator = createValidator({
    name: {
        required: true,
        type: 'string',
        minLength: 1,
        maxLength: 100
    },
    price: {
        required: true,
        type: 'number',
        custom: (value) => value > 0 ? true : 'Price must be positive'
    },
    category: {
        required: true,
        type: 'string'
    }
});

// Usage
const userData = {
    name: 'John Doe',
    email: 'john@example.com',
    age: 30
};

const userValidation = userValidator(userData);
console.log('User validation:', userValidation);

const productData = {
    name: 'Laptop',
    price: 999.99,
    category: 'Electronics'
};

const productValidation = productValidator(productData);
console.log('Product validation:', productValidation);
```

### 4. Memoization with Closures
```javascript
function createMemoizedFunction(fn) {
    const cache = new Map();
    
    return function(...args) {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            console.log('Cache hit for:', key);
            return cache.get(key);
        }
        
        console.log('Computing result for:', key);
        const result = fn.apply(this, args);
        cache.set(key, result);
        
        return result;
    };
}

// Expensive function to memoize
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

function factorial(n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

// Create memoized versions
const memoizedFibonacci = createMemoizedFunction(fibonacci);
const memoizedFactorial = createMemoizedFunction(factorial);

// Usage
console.log('Fibonacci:');
console.log(memoizedFibonacci(10)); // Computed
console.log(memoizedFibonacci(10)); // Cached
console.log(memoizedFibonacci(11)); // Computed (uses cached 10)

console.log('\nFactorial:');
console.log(memoizedFactorial(5)); // Computed
console.log(memoizedFactorial(5)); // Cached
console.log(memoizedFactorial(6)); // Computed

// Advanced memoization with TTL (Time To Live)
function createMemoizedFunctionWithTTL(fn, ttlMs = 60000) {
    const cache = new Map();
    
    return function(...args) {
        const key = JSON.stringify(args);
        const now = Date.now();
        
        if (cache.has(key)) {
            const { result, timestamp } = cache.get(key);
            
            if (now - timestamp < ttlMs) {
                console.log('Cache hit (within TTL) for:', key);
                return result;
            } else {
                console.log('Cache expired for:', key);
                cache.delete(key);
            }
        }
        
        console.log('Computing result for:', key);
        const result = fn.apply(this, args);
        cache.set(key, { result, timestamp: now });
        
        return result;
    };
}

// API call simulation
function fetchUserData(userId) {
    // Simulate API delay
    const delay = Math.random() * 1000;
    return new Promise(resolve => {
        setTimeout(() => {
            resolve({
                id: userId,
                name: `User ${userId}`,
                email: `user${userId}@example.com`,
                fetchedAt: new Date().toISOString()
            });
        }, delay);
    });
}

const memoizedFetchUserData = createMemoizedFunctionWithTTL(fetchUserData, 5000); // 5 second TTL

// Usage with async function
async function testMemoizedAsync() {
    console.log('First call:');
    console.log(await memoizedFetchUserData(123));
    
    console.log('\nSecond call (should be cached):');
    console.log(await memoizedFetchUserData(123));
    
    // Wait for cache to expire
    setTimeout(async () => {
        console.log('\nThird call (after TTL expiry):');
        console.log(await memoizedFetchUserData(123));
    }, 6000);
}

testMemoizedAsync();
```

### 5. Event Handler with Closure State
```javascript
function createEventManager() {
    const listeners = new Map();
    let eventId = 0;
    
    function addEventListener(element, eventType, handler, options = {}) {
        const id = ++eventId;
        let callCount = 0;
        const maxCalls = options.maxCalls || Infinity;
        const debounceMs = options.debounce || 0;
        let debounceTimer;
        
        const wrappedHandler = function(event) {
            // Debouncing
            if (debounceMs > 0) {
                clearTimeout(debounceTimer);
                debounceTimer = setTimeout(() => {
                    executeHandler(event);
                }, debounceMs);
            } else {
                executeHandler(event);
            }
            
            function executeHandler(event) {
                if (callCount >= maxCalls) {
                    console.log(`Handler ${id} reached max calls (${maxCalls})`);
                    return;
                }
                
                callCount++;
                console.log(`Handler ${id} called ${callCount} times`);
                
                try {
                    handler.call(this, event, { callCount, id });
                } catch (error) {
                    console.error(`Error in handler ${id}:`, error);
                }
                
                // Auto-remove after max calls
                if (callCount >= maxCalls) {
                    removeEventListener(id);
                }
            }
        };
        
        element.addEventListener(eventType, wrappedHandler);
        
        listeners.set(id, {
            element,
            eventType,
            handler: wrappedHandler,
            originalHandler: handler,
            options
        });
        
        return id;
    }
    
    function removeEventListener(id) {
        const listener = listeners.get(id);
        if (listener) {
            listener.element.removeEventListener(listener.eventType, listener.handler);
            listeners.delete(id);
            console.log(`Removed event listener ${id}`);
            return true;
        }
        return false;
    }
    
    function getListenerInfo(id) {
        const listener = listeners.get(id);
        if (listener) {
            return {
                id,
                eventType: listener.eventType,
                options: listener.options,
                element: listener.element.tagName || 'Unknown'
            };
        }
        return null;
    }
    
    function getAllListeners() {
        return Array.from(listeners.keys()).map(id => getListenerInfo(id));
    }
    
    function removeAllListeners() {
        listeners.forEach((listener, id) => {
            removeEventListener(id);
        });
    }
    
    return {
        addEventListener,
        removeEventListener,
        getListenerInfo,
        getAllListeners,
        removeAllListeners
    };
}

// Usage (in browser environment)
if (typeof document !== 'undefined') {
    const eventManager = createEventManager();
    
    // Create a button for testing
    const button = document.createElement('button');
    button.textContent = 'Click me!';
    button.id = 'test-button';
    document.body.appendChild(button);
    
    // Add event listener with options
    const listenerId1 = eventManager.addEventListener(
        button,
        'click',
        (event, meta) => {
            console.log(`Button clicked! Call #${meta.callCount}`);
        },
        {
            maxCalls: 3,
            debounce: 500
        }
    );
    
    // Add another listener
    const listenerId2 = eventManager.addEventListener(
        button,
        'mouseover',
        (event, meta) => {
            console.log(`Mouse over! Handler ID: ${meta.id}`);
        },
        {
            debounce: 200
        }
    );
    
    console.log('All listeners:', eventManager.getAllListeners());
    
    // Remove listener after 10 seconds
    setTimeout(() => {
        eventManager.removeEventListener(listenerId2);
        console.log('Remaining listeners:', eventManager.getAllListeners());
    }, 10000);
}
```

## 🌟 Real-World Examples

### Configuration Manager with Closures
```javascript
function createConfigManager(defaultConfig = {}) {
    let config = { ...defaultConfig };
    const subscribers = [];
    const history = [];
    
    function notifySubscribers(key, oldValue, newValue) {
        subscribers.forEach(callback => {
            try {
                callback(key, oldValue, newValue, { ...config });
            } catch (error) {
                console.error('Error in config subscriber:', error);
            }
        });
    }
    
    function addToHistory(action, key, oldValue, newValue) {
        history.push({
            action,
            key,
            oldValue,
            newValue,
            timestamp: new Date()
        });
        
        // Keep only last 100 entries
        if (history.length > 100) {
            history.shift();
        }
    }
    
    return {
        get(key) {
            return key ? config[key] : { ...config };
        },
        
        set(key, value) {
            if (typeof key === 'object') {
                // Bulk update
                Object.keys(key).forEach(k => {
                    const oldValue = config[k];
                    config[k] = key[k];
                    addToHistory('set', k, oldValue, key[k]);
                    notifySubscribers(k, oldValue, key[k]);
                });
            } else {
                // Single update
                const oldValue = config[key];
                config[key] = value;
                addToHistory('set', key, oldValue, value);
                notifySubscribers(key, oldValue, value);
            }
        },
        
        delete(key) {
            if (key in config) {
                const oldValue = config[key];
                delete config[key];
                addToHistory('delete', key, oldValue, undefined);
                notifySubscribers(key, oldValue, undefined);
                return true;
            }
            return false;
        },
        
        has(key) {
            return key in config;
        },
        
        reset(newConfig = defaultConfig) {
            const oldConfig = { ...config };
            config = { ...newConfig };
            addToHistory('reset', null, oldConfig, config);
            
            // Notify about all changes
            const allKeys = new Set([...Object.keys(oldConfig), ...Object.keys(config)]);
            allKeys.forEach(key => {
                notifySubscribers(key, oldConfig[key], config[key]);
            });
        },
        
        subscribe(callback) {
            if (typeof callback !== 'function') {
                throw new Error('Callback must be a function');
            }
            
            subscribers.push(callback);
            
            // Return unsubscribe function
            return function unsubscribe() {
                const index = subscribers.indexOf(callback);
                if (index > -1) {
                    subscribers.splice(index, 1);
                    return true;
                }
                return false;
            };
        },
        
        getHistory() {
            return [...history];
        },
        
        clearHistory() {
            history.length = 0;
        },
        
        getSubscriberCount() {
            return subscribers.length;
        }
    };
}

// Usage
const appConfig = createConfigManager({
    theme: 'light',
    language: 'en',
    notifications: true,
    autoSave: false
});

// Subscribe to changes
const unsubscribe1 = appConfig.subscribe((key, oldValue, newValue, fullConfig) => {
    console.log(`Config changed: ${key} from ${oldValue} to ${newValue}`);
});

const unsubscribe2 = appConfig.subscribe((key, oldValue, newValue) => {
    if (key === 'theme') {
        console.log(`Theme changed to: ${newValue}`);
        // Update UI theme
    }
});

// Make changes
appConfig.set('theme', 'dark');
appConfig.set({
    language: 'es',
    notifications: false
});

console.log('Current config:', appConfig.get());
console.log('History:', appConfig.getHistory());

// Unsubscribe
unsubscribe1();
appConfig.set('autoSave', true); // Only second subscriber will be notified
```

### Rate Limiter with Closures
```javascript
function createRateLimiter(maxRequests, windowMs) {
    const requests = [];
    
    return function rateLimitedFunction(fn) {
        return function(...args) {
            const now = Date.now();
            
            // Remove old requests outside the window
            while (requests.length > 0 && requests[0] <= now - windowMs) {
                requests.shift();
            }
            
            // Check if we've exceeded the limit
            if (requests.length >= maxRequests) {
                const oldestRequest = requests[0];
                const waitTime = windowMs - (now - oldestRequest);
                
                throw new Error(
                    `Rate limit exceeded. Try again in ${Math.ceil(waitTime / 1000)} seconds.`
                );
            }
            
            // Add current request
            requests.push(now);
            
            // Execute the function
            return fn.apply(this, args);
        };
    };
}

// Create different rate limiters
const apiRateLimiter = createRateLimiter(5, 60000); // 5 requests per minute
const searchRateLimiter = createRateLimiter(10, 10000); // 10 requests per 10 seconds

// Wrap functions with rate limiting
const rateLimitedApiCall = apiRateLimiter(function(endpoint, data) {
    console.log(`API call to ${endpoint} with data:`, data);
    return { success: true, timestamp: new Date() };
});

const rateLimitedSearch = searchRateLimiter(function(query) {
    console.log(`Searching for: ${query}`);
    return { results: [`Result for ${query}`], count: 1 };
});

// Usage
try {
    // These should work
    for (let i = 0; i < 5; i++) {
        rateLimitedApiCall('/users', { page: i });
    }
    
    // This should throw an error
    rateLimitedApiCall('/users', { page: 6 });
} catch (error) {
    console.error('Rate limit error:', error.message);
}

// Advanced rate limiter with different strategies
function createAdvancedRateLimiter(options) {
    const {
        maxRequests,
        windowMs,
        strategy = 'sliding', // 'sliding' or 'fixed'
        skipSuccessfulRequests = false,
        skipFailedRequests = false
    } = options;
    
    let requests = [];
    let windowStart = Date.now();
    
    return function(fn) {
        return async function(...args) {
            const now = Date.now();
            
            if (strategy === 'fixed') {
                // Fixed window strategy
                if (now - windowStart >= windowMs) {
                    requests = [];
                    windowStart = now;
                }
            } else {
                // Sliding window strategy
                requests = requests.filter(timestamp => now - timestamp < windowMs);
            }
            
            if (requests.length >= maxRequests) {
                const error = new Error('Rate limit exceeded');
                error.retryAfter = strategy === 'fixed' 
                    ? windowMs - (now - windowStart)
                    : windowMs - (now - requests[0]);
                throw error;
            }
            
            let result;
            let success = true;
            
            try {
                result = await fn.apply(this, args);
            } catch (error) {
                success = false;
                throw error;
            } finally {
                // Add to requests based on strategy
                if (
                    (!skipSuccessfulRequests || !success) &&
                    (!skipFailedRequests || success)
                ) {
                    requests.push(now);
                }
            }
            
            return result;
        };
    };
}

// Usage of advanced rate limiter
const advancedLimiter = createAdvancedRateLimiter({
    maxRequests: 3,
    windowMs: 5000,
    strategy: 'sliding',
    skipFailedRequests: true
});

const rateLimitedFunction = advancedLimiter(async function(data) {
    if (Math.random() < 0.3) {
        throw new Error('Random failure');
    }
    return { success: true, data };
});

// Test the advanced rate limiter
async function testAdvancedRateLimiter() {
    for (let i = 0; i < 10; i++) {
        try {
            const result = await rateLimitedFunction({ attempt: i });
            console.log(`Attempt ${i} succeeded:`, result);
        } catch (error) {
            console.log(`Attempt ${i} failed:`, error.message);
            
            if (error.retryAfter) {
                console.log(`Retry after: ${error.retryAfter}ms`);
            }
        }
        
        // Wait a bit between attempts
        await new Promise(resolve => setTimeout(resolve, 1000));
    }
}

testAdvancedRateLimiter();
```

## 🎯 Interview Questions

### Q1: What is a closure and how does it work?
**Answer:**
A closure is a function that has access to variables from its outer scope even after the outer function has returned. It "closes over" these variables, keeping them alive in memory.

```javascript
function outer(x) {
    return function inner(y) {
        return x + y; // inner has access to x
    };
}

const addFive = outer(5);
console.log(addFive(3)); // 8 - x is still accessible
```

### Q2: How can closures cause memory leaks?
**Answer:**
Closures can cause memory leaks when they hold references to large objects or DOM elements that are no longer needed:

```javascript
function problematic() {
    const largeData = new Array(1000000).fill('data');
    const element = document.getElementById('myElement');
    
    return function() {
        // This closure keeps largeData and element in memory
        console.log('Function called');
    };
}

// Solution: explicitly null references
function better() {
    let largeData = new Array(1000000).fill('data');
    let element = document.getElementById('myElement');
    
    return function() {
        console.log('Function called');
        // Clean up when no longer needed
        largeData = null;
        element = null;
    };
}
```

### Q3: What's the difference between closures and regular functions?
**Answer:**
- **Regular functions**: Only have access to their own parameters and global variables
- **Closures**: Have access to variables from their outer scope, creating a persistent lexical environment
- **Memory**: Closures keep outer variables alive, regular functions don't
- **State**: Closures can maintain private state, regular functions cannot

### Q4: How do you create private variables in JavaScript using closures?
**Answer:**
```javascript
function createPrivateCounter() {
    let count = 0; // Private variable
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
        // count is not directly accessible
    };
}

const counter = createPrivateCounter();
console.log(counter.getCount()); // 0
counter.increment();
console.log(counter.getCount()); // 1
console.log(counter.count); // undefined - private!
```

## 💡 Best Practices

1. **Be mindful of memory usage** - Closures keep outer variables alive
2. **Clean up references** - Set variables to null when no longer needed
3. **Use closures for privacy** - Create truly private variables and methods
4. **Avoid unnecessary closures** - Don't create closures when simple functions suffice
5. **Document closure behavior** - Make it clear what variables are captured
6. **Consider performance** - Closures have slight overhead compared to regular functions

## 🔗 Related Patterns
- **Module Pattern** - Uses closures for encapsulation
- **Factory Pattern** - Often implemented with closures
- **Decorator Pattern** - Closures can wrap and enhance functions
- **Observer Pattern** - Event handlers often use closures for state
