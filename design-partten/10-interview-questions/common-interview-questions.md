# Common Design Pattern Interview Questions 🎯

## 📚 Overview
This document contains the most frequently asked design pattern questions in JavaScript interviews, organized by difficulty level and pattern type.

## 🟢 Beginner Level Questions

### Q1: What is a design pattern?
**Answer:**
A design pattern is a reusable solution to a commonly occurring problem in software design. It's a template or blueprint that can be applied to solve similar problems in different contexts.

**Key Points:**
- Not code, but a description of how to solve a problem
- Proven solutions to recurring design problems
- Improve code reusability, maintainability, and communication
- Common vocabulary for developers

**Example:**
```javascript
// Singleton Pattern - ensures only one instance
class Database {
    static instance = null;
    
    static getInstance() {
        if (!Database.instance) {
            Database.instance = new Database();
        }
        return Database.instance;
    }
}
```

### Q2: What's the difference between Singleton and Factory patterns?
**Answer:**
- **Singleton**: Ensures only one instance of a class exists
- **Factory**: Creates objects without specifying their exact class

**Singleton Example:**
```javascript
class Logger {
    static instance = null;
    
    static getInstance() {
        if (!Logger.instance) {
            Logger.instance = new Logger();
        }
        return Logger.instance;
    }
    
    log(message) {
        console.log(`[${new Date().toISOString()}] ${message}`);
    }
}

const logger1 = Logger.getInstance();
const logger2 = Logger.getInstance();
console.log(logger1 === logger2); // true
```

**Factory Example:**
```javascript
class VehicleFactory {
    static createVehicle(type) {
        switch(type) {
            case 'car': return new Car();
            case 'bike': return new Bike();
            default: throw new Error('Unknown vehicle type');
        }
    }
}

const car = VehicleFactory.createVehicle('car');
const bike = VehicleFactory.createVehicle('bike');
```

### Q3: How do you implement the Observer pattern in JavaScript?
**Answer:**
The Observer pattern allows objects to subscribe to and receive notifications about events.

```javascript
class EventEmitter {
    constructor() {
        this.events = {};
    }
    
    on(event, callback) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(callback);
    }
    
    emit(event, data) {
        if (this.events[event]) {
            this.events[event].forEach(callback => callback(data));
        }
    }
    
    off(event, callback) {
        if (this.events[event]) {
            this.events[event] = this.events[event].filter(cb => cb !== callback);
        }
    }
}

// Usage
const emitter = new EventEmitter();

emitter.on('user:login', (user) => {
    console.log(`User ${user.name} logged in`);
});

emitter.emit('user:login', { name: 'John' });
```

### Q4: What is the Module pattern and why is it useful?
**Answer:**
The Module pattern provides encapsulation and privacy using closures, preventing global namespace pollution.

```javascript
const Calculator = (function() {
    // Private variables
    let result = 0;
    
    // Private method
    function log(operation, value) {
        console.log(`${operation}: ${value}`);
    }
    
    // Public API
    return {
        add(value) {
            result += value;
            log('ADD', value);
            return this;
        },
        
        subtract(value) {
            result -= value;
            log('SUBTRACT', value);
            return this;
        },
        
        getResult() {
            return result;
        }
    };
})();

Calculator.add(5).subtract(2);
console.log(Calculator.getResult()); // 3
```

## 🟡 Intermediate Level Questions

### Q5: Explain the difference between Classical and Prototypal inheritance in JavaScript
**Answer:**
- **Classical Inheritance**: Uses classes and constructors (ES6+)
- **Prototypal Inheritance**: Uses prototypes and object delegation

**Classical (ES6 Classes):**
```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        console.log(`${this.name} makes a sound`);
    }
}

class Dog extends Animal {
    speak() {
        console.log(`${this.name} barks`);
    }
}

const dog = new Dog('Rex');
dog.speak(); // Rex barks
```

**Prototypal:**
```javascript
const Animal = {
    init(name) {
        this.name = name;
        return this;
    },
    
    speak() {
        console.log(`${this.name} makes a sound`);
    }
};

const Dog = Object.create(Animal);
Dog.speak = function() {
    console.log(`${this.name} barks`);
};

const dog = Object.create(Dog).init('Rex');
dog.speak(); // Rex barks
```

### Q6: How would you implement a Decorator pattern in JavaScript?
**Answer:**
The Decorator pattern adds new functionality to objects without altering their structure.

```javascript
// Base component
class Coffee {
    cost() {
        return 5;
    }
    
    description() {
        return 'Simple coffee';
    }
}

// Decorator base class
class CoffeeDecorator {
    constructor(coffee) {
        this.coffee = coffee;
    }
    
    cost() {
        return this.coffee.cost();
    }
    
    description() {
        return this.coffee.description();
    }
}

// Concrete decorators
class MilkDecorator extends CoffeeDecorator {
    cost() {
        return this.coffee.cost() + 2;
    }
    
    description() {
        return this.coffee.description() + ', milk';
    }
}

class SugarDecorator extends CoffeeDecorator {
    cost() {
        return this.coffee.cost() + 1;
    }
    
    description() {
        return this.coffee.description() + ', sugar';
    }
}

// Usage
let coffee = new Coffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);

console.log(coffee.description()); // Simple coffee, milk, sugar
console.log(coffee.cost()); // 8
```

### Q7: What is the Strategy pattern and when would you use it?
**Answer:**
The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```javascript
// Strategy interface
class PaymentStrategy {
    pay(amount) {
        throw new Error('pay method must be implemented');
    }
}

// Concrete strategies
class CreditCardPayment extends PaymentStrategy {
    constructor(cardNumber) {
        super();
        this.cardNumber = cardNumber;
    }
    
    pay(amount) {
        console.log(`Paid $${amount} using Credit Card ending in ${this.cardNumber.slice(-4)}`);
    }
}

class PayPalPayment extends PaymentStrategy {
    constructor(email) {
        super();
        this.email = email;
    }
    
    pay(amount) {
        console.log(`Paid $${amount} using PayPal account ${this.email}`);
    }
}

class BankTransferPayment extends PaymentStrategy {
    constructor(accountNumber) {
        super();
        this.accountNumber = accountNumber;
    }
    
    pay(amount) {
        console.log(`Paid $${amount} using Bank Transfer from account ${this.accountNumber}`);
    }
}

// Context
class ShoppingCart {
    constructor() {
        this.items = [];
        this.paymentStrategy = null;
    }
    
    addItem(item) {
        this.items.push(item);
    }
    
    setPaymentStrategy(strategy) {
        this.paymentStrategy = strategy;
    }
    
    checkout() {
        const total = this.items.reduce((sum, item) => sum + item.price, 0);
        this.paymentStrategy.pay(total);
    }
}

// Usage
const cart = new ShoppingCart();
cart.addItem({ name: 'Book', price: 20 });
cart.addItem({ name: 'Pen', price: 5 });

cart.setPaymentStrategy(new CreditCardPayment('1234567890123456'));
cart.checkout(); // Paid $25 using Credit Card ending in 3456
```

### Q8: How do you prevent memory leaks with event listeners?
**Answer:**
Memory leaks occur when event listeners aren't properly removed. Here are solutions:

```javascript
class ComponentWithListeners {
    constructor() {
        this.listeners = [];
        this.handleClick = this.handleClick.bind(this);
        this.handleResize = this.handleResize.bind(this);
    }
    
    init() {
        // Store references for cleanup
        this.addListener(document, 'click', this.handleClick);
        this.addListener(window, 'resize', this.handleResize);
    }
    
    addListener(element, event, handler) {
        element.addEventListener(event, handler);
        this.listeners.push({ element, event, handler });
    }
    
    handleClick(event) {
        console.log('Document clicked');
    }
    
    handleResize(event) {
        console.log('Window resized');
    }
    
    destroy() {
        // Clean up all listeners
        this.listeners.forEach(({ element, event, handler }) => {
            element.removeEventListener(event, handler);
        });
        this.listeners = [];
    }
}

// Usage
const component = new ComponentWithListeners();
component.init();

// Later, when component is no longer needed
component.destroy(); // Prevents memory leaks
```

## 🔴 Advanced Level Questions

### Q9: Implement a Pub/Sub system with namespaces and wildcards
**Answer:**
```javascript
class PubSub {
    constructor() {
        this.events = {};
    }
    
    subscribe(event, callback) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(callback);
        
        // Return unsubscribe function
        return () => {
            this.events[event] = this.events[event].filter(cb => cb !== callback);
            if (this.events[event].length === 0) {
                delete this.events[event];
            }
        };
    }
    
    publish(event, data) {
        // Direct event match
        if (this.events[event]) {
            this.events[event].forEach(callback => callback(data, event));
        }
        
        // Wildcard matching
        Object.keys(this.events).forEach(pattern => {
            if (this.matchesPattern(pattern, event)) {
                this.events[pattern].forEach(callback => callback(data, event));
            }
        });
    }
    
    matchesPattern(pattern, event) {
        if (pattern === event) return false; // Already handled above
        
        // Convert pattern to regex
        const regexPattern = pattern
            .replace(/\*/g, '[^.]*')  // * matches anything except dots
            .replace(/\*\*/g, '.*');  // ** matches anything including dots
            
        const regex = new RegExp(`^${regexPattern}$`);
        return regex.test(event);
    }
}

// Usage
const pubsub = new PubSub();

// Subscribe to specific events
const unsubscribe1 = pubsub.subscribe('user.login', (data) => {
    console.log('User logged in:', data);
});

// Subscribe with wildcards
const unsubscribe2 = pubsub.subscribe('user.*', (data, event) => {
    console.log(`User event ${event}:`, data);
});

const unsubscribe3 = pubsub.subscribe('**', (data, event) => {
    console.log(`Global listener for ${event}:`, data);
});

// Publish events
pubsub.publish('user.login', { id: 1, name: 'John' });
pubsub.publish('user.logout', { id: 1 });
pubsub.publish('system.startup', { version: '1.0' });

// Cleanup
unsubscribe1();
unsubscribe2();
unsubscribe3();
```

### Q10: Create a Command pattern with undo/redo functionality
**Answer:**
```javascript
// Command interface
class Command {
    execute() {
        throw new Error('execute method must be implemented');
    }
    
    undo() {
        throw new Error('undo method must be implemented');
    }
}

// Concrete commands
class AddTextCommand extends Command {
    constructor(editor, text) {
        super();
        this.editor = editor;
        this.text = text;
    }
    
    execute() {
        this.editor.addText(this.text);
    }
    
    undo() {
        this.editor.removeText(this.text.length);
    }
}

class DeleteTextCommand extends Command {
    constructor(editor, length) {
        super();
        this.editor = editor;
        this.length = length;
        this.deletedText = '';
    }
    
    execute() {
        this.deletedText = this.editor.getText().slice(-this.length);
        this.editor.removeText(this.length);
    }
    
    undo() {
        this.editor.addText(this.deletedText);
    }
}

// Receiver
class TextEditor {
    constructor() {
        this.content = '';
    }
    
    addText(text) {
        this.content += text;
    }
    
    removeText(length) {
        this.content = this.content.slice(0, -length);
    }
    
    getText() {
        return this.content;
    }
}

// Invoker with undo/redo
class EditorInvoker {
    constructor() {
        this.history = [];
        this.currentPosition = -1;
    }
    
    execute(command) {
        // Remove any commands after current position (for new branch)
        this.history = this.history.slice(0, this.currentPosition + 1);
        
        // Execute and add to history
        command.execute();
        this.history.push(command);
        this.currentPosition++;
    }
    
    undo() {
        if (this.currentPosition >= 0) {
            const command = this.history[this.currentPosition];
            command.undo();
            this.currentPosition--;
            return true;
        }
        return false;
    }
    
    redo() {
        if (this.currentPosition < this.history.length - 1) {
            this.currentPosition++;
            const command = this.history[this.currentPosition];
            command.execute();
            return true;
        }
        return false;
    }
    
    canUndo() {
        return this.currentPosition >= 0;
    }
    
    canRedo() {
        return this.currentPosition < this.history.length - 1;
    }
}

// Usage
const editor = new TextEditor();
const invoker = new EditorInvoker();

// Execute commands
invoker.execute(new AddTextCommand(editor, 'Hello '));
invoker.execute(new AddTextCommand(editor, 'World!'));
console.log(editor.getText()); // "Hello World!"

invoker.execute(new DeleteTextCommand(editor, 6));
console.log(editor.getText()); // "Hello "

// Undo operations
invoker.undo();
console.log(editor.getText()); // "Hello World!"

invoker.undo();
console.log(editor.getText()); // "Hello "

// Redo operations
invoker.redo();
console.log(editor.getText()); // "Hello World!"
```

### Q11: Implement a Flyweight pattern for a text editor
**Answer:**
```javascript
// Flyweight - stores intrinsic state
class CharacterFlyweight {
    constructor(character, font, size) {
        this.character = character; // intrinsic
        this.font = font;          // intrinsic
        this.size = size;          // intrinsic
    }
    
    // Operation that uses both intrinsic and extrinsic state
    render(context, x, y, color) {
        context.fillStyle = color; // extrinsic state
        context.font = `${this.size}px ${this.font}`;
        context.fillText(this.character, x, y);
    }
}

// Flyweight Factory
class CharacterFlyweightFactory {
    constructor() {
        this.flyweights = new Map();
    }
    
    getFlyweight(character, font, size) {
        const key = `${character}-${font}-${size}`;
        
        if (!this.flyweights.has(key)) {
            this.flyweights.set(key, new CharacterFlyweight(character, font, size));
        }
        
        return this.flyweights.get(key);
    }
    
    getCreatedFlyweightsCount() {
        return this.flyweights.size;
    }
}

// Context - stores extrinsic state
class Character {
    constructor(flyweight, x, y, color) {
        this.flyweight = flyweight; // reference to flyweight
        this.x = x;                 // extrinsic state
        this.y = y;                 // extrinsic state
        this.color = color;         // extrinsic state
    }
    
    render(context) {
        this.flyweight.render(context, this.x, this.y, this.color);
    }
}

// Client - Text Editor
class TextEditor {
    constructor() {
        this.characters = [];
        this.factory = new CharacterFlyweightFactory();
    }
    
    addCharacter(char, x, y, font = 'Arial', size = 12, color = 'black') {
        const flyweight = this.factory.getFlyweight(char, font, size);
        const character = new Character(flyweight, x, y, color);
        this.characters.push(character);
    }
    
    render(context) {
        this.characters.forEach(char => char.render(context));
    }
    
    getStats() {
        return {
            totalCharacters: this.characters.length,
            uniqueFlyweights: this.factory.getCreatedFlyweightsCount(),
            memoryEfficiency: `${this.factory.getCreatedFlyweightsCount()}/${this.characters.length}`
        };
    }
}

// Usage
const editor = new TextEditor();

// Add many characters (simulating a document)
const text = "Hello World! This is a test document with repeated characters.";
let x = 0, y = 20;

for (let i = 0; i < text.length; i++) {
    const char = text[i];
    editor.addCharacter(char, x, y, 'Arial', 12, 'black');
    x += 10;
    
    if (char === ' ') {
        x += 5; // Extra space for readability
    }
    
    if (x > 500) { // Line wrap
        x = 0;
        y += 20;
    }
}

// Add more text with different formatting
const moreText = "BOLD TEXT";
for (let i = 0; i < moreText.length; i++) {
    editor.addCharacter(moreText[i], x, y, 'Arial', 16, 'red');
    x += 12;
}

console.log('Editor Stats:', editor.getStats());
// Output might be: { totalCharacters: 73, uniqueFlyweights: 25, memoryEfficiency: "25/73" }

// In a browser environment, you could render to canvas:
// const canvas = document.getElementById('canvas');
// const context = canvas.getContext('2d');
// editor.render(context);
```

## 🎯 Scenario-Based Questions

### Q12: Design a caching system with multiple eviction policies
**Answer:**
```javascript
// Strategy pattern for eviction policies
class EvictionPolicy {
    evict(cache) {
        throw new Error('evict method must be implemented');
    }
    
    onAccess(key) {
        // Override if needed
    }
    
    onSet(key) {
        // Override if needed
    }
}

class LRUEvictionPolicy extends EvictionPolicy {
    constructor() {
        super();
        this.accessOrder = [];
    }
    
    evict(cache) {
        if (this.accessOrder.length === 0) return null;
        
        const lruKey = this.accessOrder.shift();
        return lruKey;
    }
    
    onAccess(key) {
        // Move to end (most recently used)
        const index = this.accessOrder.indexOf(key);
        if (index > -1) {
            this.accessOrder.splice(index, 1);
        }
        this.accessOrder.push(key);
    }
    
    onSet(key) {
        this.onAccess(key);
    }
    
    onDelete(key) {
        const index = this.accessOrder.indexOf(key);
        if (index > -1) {
            this.accessOrder.splice(index, 1);
        }
    }
}

class LFUEvictionPolicy extends EvictionPolicy {
    constructor() {
        super();
        this.frequencies = new Map();
    }
    
    evict(cache) {
        if (this.frequencies.size === 0) return null;
        
        let minFreq = Infinity;
        let lfuKey = null;
        
        for (const [key, freq] of this.frequencies) {
            if (freq < minFreq) {
                minFreq = freq;
                lfuKey = key;
            }
        }
        
        this.frequencies.delete(lfuKey);
        return lfuKey;
    }
    
    onAccess(key) {
        this.frequencies.set(key, (this.frequencies.get(key) || 0) + 1);
    }
    
    onSet(key) {
        this.frequencies.set(key, 1);
    }
    
    onDelete(key) {
        this.frequencies.delete(key);
    }
}

// Cache implementation
class Cache {
    constructor(maxSize, evictionPolicy) {
        this.maxSize = maxSize;
        this.data = new Map();
        this.evictionPolicy = evictionPolicy;
    }
    
    get(key) {
        if (this.data.has(key)) {
            this.evictionPolicy.onAccess(key);
            return this.data.get(key);
        }
        return null;
    }
    
    set(key, value) {
        if (this.data.has(key)) {
            this.data.set(key, value);
            this.evictionPolicy.onAccess(key);
            return;
        }
        
        // Check if we need to evict
        if (this.data.size >= this.maxSize) {
            const evictedKey = this.evictionPolicy.evict(this.data);
            if (evictedKey) {
                this.data.delete(evictedKey);
            }
        }
        
        this.data.set(key, value);
        this.evictionPolicy.onSet(key);
    }
    
    delete(key) {
        if (this.data.has(key)) {
            this.data.delete(key);
            this.evictionPolicy.onDelete(key);
            return true;
        }
        return false;
    }
    
    size() {
        return this.data.size;
    }
    
    clear() {
        this.data.clear();
        this.evictionPolicy = new this.evictionPolicy.constructor();
    }
}

// Usage
const lruCache = new Cache(3, new LRUEvictionPolicy());

lruCache.set('a', 1);
lruCache.set('b', 2);
lruCache.set('c', 3);

console.log(lruCache.get('a')); // 1 (a becomes most recently used)

lruCache.set('d', 4); // Should evict 'b' (least recently used)

console.log(lruCache.get('b')); // null (evicted)
console.log(lruCache.get('a')); // 1 (still there)
console.log(lruCache.get('c')); // 3 (still there)
console.log(lruCache.get('d')); // 4 (still there)
```

### Q13: How would you implement a middleware pattern for an HTTP server?
**Answer:**
```javascript
class MiddlewareManager {
    constructor() {
        this.middlewares = [];
    }
    
    use(middleware) {
        if (typeof middleware !== 'function') {
            throw new Error('Middleware must be a function');
        }
        this.middlewares.push(middleware);
    }
    
    async execute(context) {
        let index = 0;
        
        const next = async () => {
            if (index >= this.middlewares.length) {
                return;
            }
            
            const middleware = this.middlewares[index++];
            await middleware(context, next);
        };
        
        await next();
    }
}

// Example middlewares
const loggingMiddleware = async (context, next) => {
    console.log(`${new Date().toISOString()} - ${context.method} ${context.url}`);
    await next();
    console.log(`Response: ${context.statusCode}`);
};

const authMiddleware = async (context, next) => {
    const token = context.headers['authorization'];
    
    if (!token) {
        context.statusCode = 401;
        context.body = { error: 'Unauthorized' };
        return; // Don't call next() - stops the chain
    }
    
    // Simulate token validation
    context.user = { id: 1, name: 'John Doe' };
    await next();
};

const corsMiddleware = async (context, next) => {
    context.headers['Access-Control-Allow-Origin'] = '*';
    context.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE';
    await next();
};

const errorHandlingMiddleware = async (context, next) => {
    try {
        await next();
    } catch (error) {
        console.error('Error:', error);
        context.statusCode = 500;
        context.body = { error: 'Internal Server Error' };
    }
};

// Route handler (final middleware)
const routeHandler = async (context, next) => {
    if (context.url === '/users' && context.method === 'GET') {
        context.statusCode = 200;
        context.body = { users: [context.user] };
    } else {
        context.statusCode = 404;
        context.body = { error: 'Not Found' };
    }
};

// Usage
const app = new MiddlewareManager();

app.use(errorHandlingMiddleware);
app.use(loggingMiddleware);
app.use(corsMiddleware);
app.use(authMiddleware);
app.use(routeHandler);

// Simulate request
const context = {
    method: 'GET',
    url: '/users',
    headers: { authorization: 'Bearer token123' },
    statusCode: 200,
    body: null
};

app.execute(context).then(() => {
    console.log('Final context:', context);
});
```

## 💡 Tips for Interviews

### 1. **Structure Your Answers**
- Start with definition
- Explain when to use it
- Provide a simple example
- Discuss pros and cons
- Mention alternatives

### 2. **Common Follow-up Questions**
- "How would you test this pattern?"
- "What are the performance implications?"
- "How does this scale?"
- "What are the alternatives?"

### 3. **Red Flags to Avoid**
- Don't over-engineer simple problems
- Don't use patterns just because you know them
- Don't ignore performance implications
- Don't forget about testing and maintainability

### 4. **Show Real-World Understanding**
- Mention where you've seen the pattern used
- Discuss framework implementations (React, Vue, etc.)
- Talk about trade-offs and alternatives
- Show awareness of modern JavaScript features

### 5. **Practice Coding**
- Be able to implement patterns from scratch
- Understand the underlying principles
- Practice explaining while coding
- Be ready to modify implementations based on requirements

## 🔗 Additional Resources

- **Books**: "Design Patterns" by Gang of Four, "JavaScript Patterns" by Stoyan Stefanov
- **Practice**: LeetCode, HackerRank design pattern problems
- **Real Examples**: Study popular libraries and frameworks
- **Mock Interviews**: Practice with peers or online platforms
