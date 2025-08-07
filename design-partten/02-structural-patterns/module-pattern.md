# Module Pattern 📦

## 📖 Definition
The Module pattern provides a way to encapsulate private and public methods and variables, preventing them from leaking into the global scope and accidentally colliding with another developer's interface.

## 🎯 When to Use
- When you need to create private scope and encapsulation
- When you want to avoid global namespace pollution
- When you need to organize related functionality together
- When you want to create reusable components with clean APIs

## ✅ Pros
- Encapsulation and privacy
- Clean namespace management
- Organized code structure
- Prevents global scope pollution
- Supports both private and public methods

## ❌ Cons
- Can be harder to extend
- Testing private methods is difficult
- Memory usage (each instance creates new copies)
- Can be overkill for simple functionality

## 🚀 JavaScript Implementations

### 1. Basic Module Pattern (IIFE)
```javascript
const Calculator = (function() {
    // Private variables and methods
    let result = 0;
    const history = [];
    
    function logOperation(operation, value) {
        history.push(`${operation}: ${value} = ${result}`);
        console.log(`Operation logged: ${operation}`);
    }
    
    function validateNumber(num) {
        if (typeof num !== 'number' || isNaN(num)) {
            throw new Error('Invalid number provided');
        }
    }
    
    // Public API
    return {
        add(num) {
            validateNumber(num);
            result += num;
            logOperation('ADD', num);
            return this;
        },
        
        subtract(num) {
            validateNumber(num);
            result -= num;
            logOperation('SUBTRACT', num);
            return this;
        },
        
        multiply(num) {
            validateNumber(num);
            result *= num;
            logOperation('MULTIPLY', num);
            return this;
        },
        
        divide(num) {
            validateNumber(num);
            if (num === 0) {
                throw new Error('Division by zero');
            }
            result /= num;
            logOperation('DIVIDE', num);
            return this;
        },
        
        getResult() {
            return result;
        },
        
        reset() {
            result = 0;
            history.length = 0;
            return this;
        },
        
        getHistory() {
            return [...history]; // Return copy to maintain encapsulation
        }
    };
})();

// Usage
Calculator
    .add(10)
    .multiply(2)
    .subtract(5)
    .divide(3);

console.log(Calculator.getResult()); // 5
console.log(Calculator.getHistory());
// Cannot access private variables
console.log(Calculator.result); // undefined
```

### 2. Revealing Module Pattern
```javascript
const UserManager = (function() {
    // Private variables
    const users = [];
    let currentUser = null;
    const settings = {
        maxUsers: 100,
        requireEmail: true
    };
    
    // Private methods
    function validateUser(user) {
        if (!user.name || user.name.trim() === '') {
            throw new Error('User name is required');
        }
        
        if (settings.requireEmail && (!user.email || !isValidEmail(user.email))) {
            throw new Error('Valid email is required');
        }
        
        if (users.length >= settings.maxUsers) {
            throw new Error('Maximum user limit reached');
        }
    }
    
    function isValidEmail(email) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    }
    
    function generateId() {
        return Date.now().toString(36) + Math.random().toString(36).substr(2);
    }
    
    function findUserById(id) {
        return users.find(user => user.id === id);
    }
    
    function findUserByEmail(email) {
        return users.find(user => user.email === email);
    }
    
    // Public methods
    function addUser(userData) {
        validateUser(userData);
        
        if (findUserByEmail(userData.email)) {
            throw new Error('User with this email already exists');
        }
        
        const user = {
            id: generateId(),
            name: userData.name,
            email: userData.email,
            createdAt: new Date(),
            isActive: true
        };
        
        users.push(user);
        return { ...user }; // Return copy
    }
    
    function removeUser(id) {
        const index = users.findIndex(user => user.id === id);
        if (index === -1) {
            throw new Error('User not found');
        }
        
        const removedUser = users.splice(index, 1)[0];
        
        if (currentUser && currentUser.id === id) {
            currentUser = null;
        }
        
        return { ...removedUser };
    }
    
    function updateUser(id, updates) {
        const user = findUserById(id);
        if (!user) {
            throw new Error('User not found');
        }
        
        // Validate updates
        if (updates.name !== undefined) {
            if (!updates.name || updates.name.trim() === '') {
                throw new Error('User name cannot be empty');
            }
            user.name = updates.name;
        }
        
        if (updates.email !== undefined) {
            if (!isValidEmail(updates.email)) {
                throw new Error('Invalid email format');
            }
            
            const existingUser = findUserByEmail(updates.email);
            if (existingUser && existingUser.id !== id) {
                throw new Error('Email already in use');
            }
            
            user.email = updates.email;
        }
        
        if (updates.isActive !== undefined) {
            user.isActive = Boolean(updates.isActive);
        }
        
        user.updatedAt = new Date();
        return { ...user };
    }
    
    function getUser(id) {
        const user = findUserById(id);
        return user ? { ...user } : null;
    }
    
    function getAllUsers() {
        return users.map(user => ({ ...user }));
    }
    
    function setCurrentUser(id) {
        const user = findUserById(id);
        if (!user) {
            throw new Error('User not found');
        }
        
        currentUser = user;
        return { ...user };
    }
    
    function getCurrentUser() {
        return currentUser ? { ...currentUser } : null;
    }
    
    function getUserCount() {
        return users.length;
    }
    
    function updateSettings(newSettings) {
        Object.assign(settings, newSettings);
        return { ...settings };
    }
    
    function getSettings() {
        return { ...settings };
    }
    
    // Reveal public methods
    return {
        addUser,
        removeUser,
        updateUser,
        getUser,
        getAllUsers,
        setCurrentUser,
        getCurrentUser,
        getUserCount,
        updateSettings,
        getSettings
    };
})();

// Usage
const user1 = UserManager.addUser({
    name: 'John Doe',
    email: 'john@example.com'
});

const user2 = UserManager.addUser({
    name: 'Jane Smith',
    email: 'jane@example.com'
});

UserManager.setCurrentUser(user1.id);
console.log('Current user:', UserManager.getCurrentUser());
console.log('Total users:', UserManager.getUserCount());

// Private methods and variables are not accessible
console.log(UserManager.users); // undefined
console.log(UserManager.validateUser); // undefined
```

### 3. Module Pattern with Parameters
```javascript
const createApiClient = function(baseUrl, apiKey) {
    // Private variables
    const config = {
        baseUrl: baseUrl,
        apiKey: apiKey,
        timeout: 5000,
        retries: 3
    };
    
    const cache = new Map();
    let requestCount = 0;
    
    // Private methods
    function buildUrl(endpoint) {
        return `${config.baseUrl}${endpoint}`;
    }
    
    function getHeaders() {
        return {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${config.apiKey}`,
            'X-Request-ID': generateRequestId()
        };
    }
    
    function generateRequestId() {
        return `req_${Date.now()}_${++requestCount}`;
    }
    
    function getCacheKey(method, url, data) {
        return `${method}:${url}:${JSON.stringify(data || {})}`;
    }
    
    async function makeRequest(method, endpoint, data = null, options = {}) {
        const url = buildUrl(endpoint);
        const cacheKey = getCacheKey(method, url, data);
        
        // Check cache for GET requests
        if (method === 'GET' && cache.has(cacheKey)) {
            const cached = cache.get(cacheKey);
            if (Date.now() - cached.timestamp < 300000) { // 5 minutes
                console.log('Returning cached response');
                return cached.data;
            }
        }
        
        const requestOptions = {
            method,
            headers: getHeaders(),
            timeout: options.timeout || config.timeout,
            ...options
        };
        
        if (data) {
            requestOptions.body = JSON.stringify(data);
        }
        
        let attempt = 0;
        while (attempt <= config.retries) {
            try {
                const response = await fetch(url, requestOptions);
                
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }
                
                const result = await response.json();
                
                // Cache GET requests
                if (method === 'GET') {
                    cache.set(cacheKey, {
                        data: result,
                        timestamp: Date.now()
                    });
                }
                
                return result;
            } catch (error) {
                attempt++;
                if (attempt > config.retries) {
                    throw new Error(`Request failed after ${config.retries} retries: ${error.message}`);
                }
                
                // Wait before retry
                await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
            }
        }
    }
    
    // Public API
    return {
        get(endpoint, options) {
            return makeRequest('GET', endpoint, null, options);
        },
        
        post(endpoint, data, options) {
            return makeRequest('POST', endpoint, data, options);
        },
        
        put(endpoint, data, options) {
            return makeRequest('PUT', endpoint, data, options);
        },
        
        delete(endpoint, options) {
            return makeRequest('DELETE', endpoint, null, options);
        },
        
        patch(endpoint, data, options) {
            return makeRequest('PATCH', endpoint, data, options);
        },
        
        setConfig(newConfig) {
            Object.assign(config, newConfig);
        },
        
        getConfig() {
            return { ...config };
        },
        
        clearCache() {
            cache.clear();
        },
        
        getCacheSize() {
            return cache.size;
        },
        
        getRequestCount() {
            return requestCount;
        }
    };
};

// Usage
const apiClient = createApiClient('https://api.example.com', 'your-api-key');

// Configure client
apiClient.setConfig({ timeout: 10000, retries: 5 });

// Make requests
apiClient.get('/users')
    .then(users => console.log('Users:', users))
    .catch(error => console.error('Error:', error));

apiClient.post('/users', { name: 'John Doe', email: 'john@example.com' })
    .then(user => console.log('Created user:', user))
    .catch(error => console.error('Error:', error));
```

### 4. ES6 Module Pattern
```javascript
// userService.js
class UserService {
    #users = [];
    #currentUser = null;
    #settings = {
        maxUsers: 100,
        requireEmail: true
    };
    
    // Private methods
    #validateUser(user) {
        if (!user.name || user.name.trim() === '') {
            throw new Error('User name is required');
        }
        
        if (this.#settings.requireEmail && (!user.email || !this.#isValidEmail(user.email))) {
            throw new Error('Valid email is required');
        }
        
        if (this.#users.length >= this.#settings.maxUsers) {
            throw new Error('Maximum user limit reached');
        }
    }
    
    #isValidEmail(email) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    }
    
    #generateId() {
        return Date.now().toString(36) + Math.random().toString(36).substr(2);
    }
    
    #findUserById(id) {
        return this.#users.find(user => user.id === id);
    }
    
    #findUserByEmail(email) {
        return this.#users.find(user => user.email === email);
    }
    
    // Public methods
    addUser(userData) {
        this.#validateUser(userData);
        
        if (this.#findUserByEmail(userData.email)) {
            throw new Error('User with this email already exists');
        }
        
        const user = {
            id: this.#generateId(),
            name: userData.name,
            email: userData.email,
            createdAt: new Date(),
            isActive: true
        };
        
        this.#users.push(user);
        return { ...user };
    }
    
    removeUser(id) {
        const index = this.#users.findIndex(user => user.id === id);
        if (index === -1) {
            throw new Error('User not found');
        }
        
        const removedUser = this.#users.splice(index, 1)[0];
        
        if (this.#currentUser && this.#currentUser.id === id) {
            this.#currentUser = null;
        }
        
        return { ...removedUser };
    }
    
    getUser(id) {
        const user = this.#findUserById(id);
        return user ? { ...user } : null;
    }
    
    getAllUsers() {
        return this.#users.map(user => ({ ...user }));
    }
    
    getUserCount() {
        return this.#users.length;
    }
    
    setCurrentUser(id) {
        const user = this.#findUserById(id);
        if (!user) {
            throw new Error('User not found');
        }
        
        this.#currentUser = user;
        return { ...user };
    }
    
    getCurrentUser() {
        return this.#currentUser ? { ...this.#currentUser } : null;
    }
}

// Export singleton instance
export default new UserService();

// Or export the class for multiple instances
export { UserService };
```

## 🌟 Real-World Examples

### Shopping Cart Module
```javascript
const ShoppingCart = (function() {
    // Private state
    let items = [];
    let discounts = [];
    const tax = 0.08; // 8% tax
    const shippingRates = {
        standard: 5.99,
        express: 12.99,
        overnight: 24.99
    };
    
    // Private methods
    function findItemIndex(productId) {
        return items.findIndex(item => item.productId === productId);
    }
    
    function calculateSubtotal() {
        return items.reduce((total, item) => total + (item.price * item.quantity), 0);
    }
    
    function calculateDiscounts() {
        const subtotal = calculateSubtotal();
        return discounts.reduce((total, discount) => {
            if (discount.type === 'percentage') {
                return total + (subtotal * discount.value / 100);
            } else if (discount.type === 'fixed') {
                return total + discount.value;
            }
            return total;
        }, 0);
    }
    
    function calculateTax(amount) {
        return amount * tax;
    }
    
    function validateProduct(product) {
        if (!product.id || !product.name || !product.price) {
            throw new Error('Invalid product: id, name, and price are required');
        }
        
        if (product.price < 0) {
            throw new Error('Product price cannot be negative');
        }
    }
    
    function validateQuantity(quantity) {
        if (!Number.isInteger(quantity) || quantity < 1) {
            throw new Error('Quantity must be a positive integer');
        }
    }
    
    // Public API
    return {
        addItem(product, quantity = 1) {
            validateProduct(product);
            validateQuantity(quantity);
            
            const existingIndex = findItemIndex(product.id);
            
            if (existingIndex !== -1) {
                items[existingIndex].quantity += quantity;
            } else {
                items.push({
                    productId: product.id,
                    name: product.name,
                    price: product.price,
                    quantity: quantity,
                    addedAt: new Date()
                });
            }
            
            return this;
        },
        
        removeItem(productId) {
            const index = findItemIndex(productId);
            if (index !== -1) {
                items.splice(index, 1);
            }
            return this;
        },
        
        updateQuantity(productId, quantity) {
            validateQuantity(quantity);
            
            const index = findItemIndex(productId);
            if (index !== -1) {
                items[index].quantity = quantity;
            }
            
            return this;
        },
        
        getItems() {
            return items.map(item => ({ ...item }));
        },
        
        getItemCount() {
            return items.reduce((total, item) => total + item.quantity, 0);
        },
        
        addDiscount(discount) {
            if (!discount.code || !discount.type || discount.value === undefined) {
                throw new Error('Invalid discount: code, type, and value are required');
            }
            
            if (!['percentage', 'fixed'].includes(discount.type)) {
                throw new Error('Discount type must be "percentage" or "fixed"');
            }
            
            // Remove existing discount with same code
            discounts = discounts.filter(d => d.code !== discount.code);
            discounts.push(discount);
            
            return this;
        },
        
        removeDiscount(code) {
            discounts = discounts.filter(d => d.code !== code);
            return this;
        },
        
        getDiscounts() {
            return discounts.map(discount => ({ ...discount }));
        },
        
        calculateTotal(shippingType = 'standard') {
            const subtotal = calculateSubtotal();
            const discountAmount = calculateDiscounts();
            const discountedAmount = Math.max(0, subtotal - discountAmount);
            const taxAmount = calculateTax(discountedAmount);
            const shippingCost = shippingRates[shippingType] || shippingRates.standard;
            
            return {
                subtotal: Number(subtotal.toFixed(2)),
                discounts: Number(discountAmount.toFixed(2)),
                tax: Number(taxAmount.toFixed(2)),
                shipping: Number(shippingCost.toFixed(2)),
                total: Number((discountedAmount + taxAmount + shippingCost).toFixed(2))
            };
        },
        
        clear() {
            items = [];
            discounts = [];
            return this;
        },
        
        isEmpty() {
            return items.length === 0;
        }
    };
})();

// Usage
ShoppingCart
    .addItem({ id: 1, name: 'Laptop', price: 999.99 }, 1)
    .addItem({ id: 2, name: 'Mouse', price: 29.99 }, 2)
    .addDiscount({ code: 'SAVE10', type: 'percentage', value: 10 });

console.log('Cart items:', ShoppingCart.getItems());
console.log('Total calculation:', ShoppingCart.calculateTotal('express'));
```

### Event Manager Module
```javascript
const EventManager = (function() {
    // Private state
    const events = new Map();
    const eventHistory = [];
    let isLogging = true;
    
    // Private methods
    function logEvent(eventName, action, data) {
        if (isLogging) {
            eventHistory.push({
                eventName,
                action,
                data,
                timestamp: new Date()
            });
        }
    }
    
    function validateEventName(eventName) {
        if (typeof eventName !== 'string' || eventName.trim() === '') {
            throw new Error('Event name must be a non-empty string');
        }
    }
    
    function validateCallback(callback) {
        if (typeof callback !== 'function') {
            throw new Error('Callback must be a function');
        }
    }
    
    // Public API
    return {
        on(eventName, callback, options = {}) {
            validateEventName(eventName);
            validateCallback(callback);
            
            if (!events.has(eventName)) {
                events.set(eventName, []);
            }
            
            const listener = {
                callback,
                once: options.once || false,
                priority: options.priority || 0,
                id: Date.now() + Math.random()
            };
            
            events.get(eventName).push(listener);
            
            // Sort by priority (higher priority first)
            events.get(eventName).sort((a, b) => b.priority - a.priority);
            
            logEvent(eventName, 'LISTENER_ADDED', { listenerId: listener.id });
            
            return listener.id;
        },
        
        once(eventName, callback, options = {}) {
            return this.on(eventName, callback, { ...options, once: true });
        },
        
        off(eventName, listenerId) {
            validateEventName(eventName);
            
            if (!events.has(eventName)) {
                return false;
            }
            
            const listeners = events.get(eventName);
            const index = listeners.findIndex(listener => listener.id === listenerId);
            
            if (index !== -1) {
                listeners.splice(index, 1);
                logEvent(eventName, 'LISTENER_REMOVED', { listenerId });
                
                if (listeners.length === 0) {
                    events.delete(eventName);
                }
                
                return true;
            }
            
            return false;
        },
        
        emit(eventName, ...args) {
            validateEventName(eventName);
            
            if (!events.has(eventName)) {
                logEvent(eventName, 'EMIT_NO_LISTENERS', { args });
                return 0;
            }
            
            const listeners = events.get(eventName);
            const listenersToRemove = [];
            let callCount = 0;
            
            listeners.forEach(listener => {
                try {
                    listener.callback(...args);
                    callCount++;
                    
                    if (listener.once) {
                        listenersToRemove.push(listener.id);
                    }
                } catch (error) {
                    console.error(`Error in event listener for '${eventName}':`, error);
                    logEvent(eventName, 'LISTENER_ERROR', { 
                        listenerId: listener.id, 
                        error: error.message 
                    });
                }
            });
            
            // Remove 'once' listeners
            listenersToRemove.forEach(id => this.off(eventName, id));
            
            logEvent(eventName, 'EMIT', { args, callCount });
            
            return callCount;
        },
        
        removeAllListeners(eventName) {
            if (eventName) {
                validateEventName(eventName);
                const removed = events.has(eventName);
                events.delete(eventName);
                logEvent(eventName, 'ALL_LISTENERS_REMOVED', {});
                return removed;
            } else {
                const eventCount = events.size;
                events.clear();
                logEvent('*', 'ALL_EVENTS_CLEARED', { eventCount });
                return eventCount;
            }
        },
        
        getListenerCount(eventName) {
            if (eventName) {
                validateEventName(eventName);
                return events.has(eventName) ? events.get(eventName).length : 0;
            } else {
                return Array.from(events.values()).reduce((total, listeners) => total + listeners.length, 0);
            }
        },
        
        getEventNames() {
            return Array.from(events.keys());
        },
        
        getHistory() {
            return eventHistory.map(entry => ({ ...entry }));
        },
        
        clearHistory() {
            eventHistory.length = 0;
        },
        
        setLogging(enabled) {
            isLogging = Boolean(enabled);
        },
        
        isLogging() {
            return isLogging;
        }
    };
})();

// Usage
const listenerId = EventManager.on('user:login', (user) => {
    console.log(`User ${user.name} logged in`);
}, { priority: 1 });

EventManager.once('app:ready', () => {
    console.log('Application is ready!');
});

EventManager.emit('user:login', { name: 'John Doe', id: 123 });
EventManager.emit('app:ready');

console.log('Event names:', EventManager.getEventNames());
console.log('History:', EventManager.getHistory());
```

## 🎯 Interview Questions

### Q1: What's the difference between Module pattern and Namespace pattern?
**Answer:**
- **Module Pattern**: Creates private scope with closures, encapsulates private variables/methods
- **Namespace Pattern**: Just organizes code into objects, no real privacy
- **Module** provides true encapsulation, **Namespace** is just organization

### Q2: How does the Module pattern prevent global namespace pollution?
**Answer:**
- Uses IIFE to create private scope
- Only exposes necessary methods through return object
- Private variables and methods are not accessible from outside
- Reduces risk of naming conflicts

### Q3: What are the advantages of Revealing Module pattern over basic Module pattern?
**Answer:**
- All methods defined in private scope
- Clear distinction between private and public methods
- Easier to see what's exposed at the bottom
- More consistent syntax for all methods

### Q4: How do ES6 modules compare to the traditional Module pattern?
**Answer:**
- **ES6 Modules**: Built-in language feature, static imports/exports
- **Module Pattern**: Runtime pattern using closures
- **ES6** has better tooling support, tree-shaking, static analysis
- **Module Pattern** works in older browsers, more flexible at runtime

## 💡 Best Practices

1. **Use IIFE for immediate execution** - Creates private scope immediately
2. **Return object for public API** - Clear interface definition
3. **Keep private methods truly private** - Don't expose internal implementation
4. **Use revealing pattern for clarity** - Makes public interface obvious
5. **Consider memory implications** - Each instance creates new closures
6. **Document your public API** - Make it clear what's available to consumers

## 🔗 Related Patterns
- **Singleton Pattern** - Modules are often singletons
- **Factory Pattern** - Can be implemented as modules
- **Observer Pattern** - Event systems often use module pattern
- **Namespace Pattern** - Simpler alternative without privacy
