# Observer Pattern 👁️

## 📖 Definition
The Observer pattern defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

## 🎯 When to Use
- When changes to one object require updating multiple objects
- When you want to decouple the subject from its observers
- When you need to broadcast information to multiple subscribers
- When implementing event systems or reactive programming

## ✅ Pros
- Loose coupling between subject and observers
- Dynamic relationships (can add/remove observers at runtime)
- Supports broadcast communication
- Follows Open/Closed Principle
- Foundation for reactive programming

## ❌ Cons
- Can cause memory leaks if observers aren't properly removed
- Unexpected updates and cascading effects
- Difficult to debug complex observer chains
- Performance overhead with many observers

## 🚀 JavaScript Implementations

### 1. Basic Observer Pattern
```javascript
class Subject {
    constructor() {
        this.observers = [];
    }
    
    subscribe(observer) {
        if (typeof observer.update !== 'function') {
            throw new Error('Observer must have an update method');
        }
        
        this.observers.push(observer);
        console.log('Observer subscribed');
    }
    
    unsubscribe(observer) {
        const index = this.observers.indexOf(observer);
        if (index > -1) {
            this.observers.splice(index, 1);
            console.log('Observer unsubscribed');
        }
    }
    
    notify(data) {
        console.log(`Notifying ${this.observers.length} observers`);
        this.observers.forEach(observer => {
            try {
                observer.update(data);
            } catch (error) {
                console.error('Error notifying observer:', error);
            }
        });
    }
}

class Observer {
    constructor(name) {
        this.name = name;
    }
    
    update(data) {
        console.log(`${this.name} received update:`, data);
    }
}

// Usage
const subject = new Subject();
const observer1 = new Observer('Observer 1');
const observer2 = new Observer('Observer 2');

subject.subscribe(observer1);
subject.subscribe(observer2);

subject.notify({ message: 'Hello Observers!' });
subject.unsubscribe(observer1);
subject.notify({ message: 'Second notification' });
```

### 2. Event Emitter Pattern
```javascript
class EventEmitter {
    constructor() {
        this.events = new Map();
    }
    
    on(eventName, callback) {
        if (!this.events.has(eventName)) {
            this.events.set(eventName, []);
        }
        
        this.events.get(eventName).push(callback);
        return this; // For method chaining
    }
    
    once(eventName, callback) {
        const onceWrapper = (...args) => {
            callback(...args);
            this.off(eventName, onceWrapper);
        };
        
        this.on(eventName, onceWrapper);
        return this;
    }
    
    off(eventName, callback) {
        if (!this.events.has(eventName)) {
            return this;
        }
        
        const callbacks = this.events.get(eventName);
        const index = callbacks.indexOf(callback);
        
        if (index > -1) {
            callbacks.splice(index, 1);
        }
        
        if (callbacks.length === 0) {
            this.events.delete(eventName);
        }
        
        return this;
    }
    
    emit(eventName, ...args) {
        if (!this.events.has(eventName)) {
            return false;
        }
        
        const callbacks = this.events.get(eventName);
        callbacks.forEach(callback => {
            try {
                callback(...args);
            } catch (error) {
                console.error(`Error in event handler for '${eventName}':`, error);
            }
        });
        
        return true;
    }
    
    removeAllListeners(eventName) {
        if (eventName) {
            this.events.delete(eventName);
        } else {
            this.events.clear();
        }
        return this;
    }
    
    listenerCount(eventName) {
        return this.events.has(eventName) ? this.events.get(eventName).length : 0;
    }
    
    eventNames() {
        return Array.from(this.events.keys());
    }
}

// Usage
const emitter = new EventEmitter();

emitter.on('user:login', (user) => {
    console.log(`User ${user.name} logged in`);
});

emitter.on('user:login', (user) => {
    console.log(`Welcome back, ${user.name}!`);
});

emitter.once('app:start', () => {
    console.log('Application started - this will only run once');
});

emitter.emit('user:login', { name: 'John Doe', id: 123 });
emitter.emit('app:start');
emitter.emit('app:start'); // Won't trigger the once listener again
```

### 3. Reactive Observable Pattern
```javascript
class Observable {
    constructor(subscriberFunction) {
        this._subscribe = subscriberFunction;
    }
    
    subscribe(observer) {
        const unsubscribe = this._subscribe(observer);
        return { unsubscribe };
    }
    
    static create(subscriberFunction) {
        return new Observable(subscriberFunction);
    }
    
    static fromArray(array) {
        return new Observable(observer => {
            array.forEach(item => observer.next(item));
            observer.complete();
            
            return () => {}; // No cleanup needed
        });
    }
    
    static fromEvent(element, eventName) {
        return new Observable(observer => {
            const handler = (event) => observer.next(event);
            element.addEventListener(eventName, handler);
            
            return () => {
                element.removeEventListener(eventName, handler);
            };
        });
    }
    
    map(transformFn) {
        return new Observable(observer => {
            return this.subscribe({
                next: value => observer.next(transformFn(value)),
                error: err => observer.error(err),
                complete: () => observer.complete()
            });
        });
    }
    
    filter(predicateFn) {
        return new Observable(observer => {
            return this.subscribe({
                next: value => {
                    if (predicateFn(value)) {
                        observer.next(value);
                    }
                },
                error: err => observer.error(err),
                complete: () => observer.complete()
            });
        });
    }
    
    debounce(delay) {
        return new Observable(observer => {
            let timeoutId;
            
            return this.subscribe({
                next: value => {
                    clearTimeout(timeoutId);
                    timeoutId = setTimeout(() => {
                        observer.next(value);
                    }, delay);
                },
                error: err => observer.error(err),
                complete: () => {
                    clearTimeout(timeoutId);
                    observer.complete();
                }
            });
        });
    }
}

// Usage
const numbers$ = Observable.fromArray([1, 2, 3, 4, 5]);

const subscription = numbers$
    .filter(x => x % 2 === 0)
    .map(x => x * 2)
    .subscribe({
        next: value => console.log('Next:', value),
        error: err => console.error('Error:', err),
        complete: () => console.log('Complete!')
    });

// DOM event example
if (typeof document !== 'undefined') {
    const button = document.getElementById('myButton');
    const clicks$ = Observable.fromEvent(button, 'click');
    
    const debouncedClicks$ = clicks$.debounce(300);
    
    const clickSubscription = debouncedClicks$.subscribe({
        next: event => console.log('Button clicked!', event),
        error: err => console.error('Click error:', err)
    });
    
    // Cleanup
    setTimeout(() => {
        clickSubscription.unsubscribe();
    }, 10000);
}
```

### 4. Model-View Observer Pattern
```javascript
class Model extends EventEmitter {
    constructor(initialData = {}) {
        super();
        this.data = { ...initialData };
    }
    
    get(key) {
        return this.data[key];
    }
    
    set(key, value) {
        const oldValue = this.data[key];
        this.data[key] = value;
        
        this.emit('change', { key, value, oldValue });
        this.emit(`change:${key}`, { value, oldValue });
    }
    
    update(updates) {
        const changes = [];
        
        Object.keys(updates).forEach(key => {
            const oldValue = this.data[key];
            const newValue = updates[key];
            
            if (oldValue !== newValue) {
                this.data[key] = newValue;
                changes.push({ key, value: newValue, oldValue });
            }
        });
        
        if (changes.length > 0) {
            this.emit('change', changes);
            changes.forEach(change => {
                this.emit(`change:${change.key}`, {
                    value: change.value,
                    oldValue: change.oldValue
                });
            });
        }
    }
    
    toJSON() {
        return { ...this.data };
    }
}

class View {
    constructor(model, element) {
        this.model = model;
        this.element = element;
        this.setupEventListeners();
        this.render();
    }
    
    setupEventListeners() {
        this.model.on('change', () => {
            this.render();
        });
        
        this.model.on('change:name', (change) => {
            console.log(`Name changed from "${change.oldValue}" to "${change.value}"`);
        });
    }
    
    render() {
        if (this.element) {
            this.element.innerHTML = `
                <h2>${this.model.get('name') || 'Unknown'}</h2>
                <p>Email: ${this.model.get('email') || 'Not provided'}</p>
                <p>Age: ${this.model.get('age') || 'Unknown'}</p>
                <p>Last updated: ${new Date().toLocaleTimeString()}</p>
            `;
        }
    }
}

// Usage
const userModel = new Model({
    name: 'John Doe',
    email: 'john@example.com',
    age: 30
});

// Create view (in browser environment)
if (typeof document !== 'undefined') {
    const viewElement = document.getElementById('user-view');
    const userView = new View(userModel, viewElement);
    
    // Model changes will automatically update the view
    setTimeout(() => {
        userModel.set('name', 'Jane Smith');
    }, 2000);
    
    setTimeout(() => {
        userModel.update({
            email: 'jane@example.com',
            age: 28
        });
    }, 4000);
}
```

## 🌟 Real-World Examples

### Stock Price Monitor
```javascript
class StockPrice extends EventEmitter {
    constructor(symbol, initialPrice) {
        super();
        this.symbol = symbol;
        this.price = initialPrice;
        this.history = [{ price: initialPrice, timestamp: new Date() }];
        this.isActive = false;
    }
    
    updatePrice(newPrice) {
        const oldPrice = this.price;
        this.price = newPrice;
        
        this.history.push({
            price: newPrice,
            timestamp: new Date()
        });
        
        // Keep only last 100 entries
        if (this.history.length > 100) {
            this.history.shift();
        }
        
        const change = newPrice - oldPrice;
        const changePercent = (change / oldPrice) * 100;
        
        this.emit('priceUpdate', {
            symbol: this.symbol,
            price: newPrice,
            oldPrice,
            change,
            changePercent: Number(changePercent.toFixed(2))
        });
        
        // Emit specific events based on price movement
        if (change > 0) {
            this.emit('priceIncrease', { symbol: this.symbol, price: newPrice, change });
        } else if (change < 0) {
            this.emit('priceDecrease', { symbol: this.symbol, price: newPrice, change });
        }
        
        // Alert for significant changes
        if (Math.abs(changePercent) >= 5) {
            this.emit('significantChange', {
                symbol: this.symbol,
                price: newPrice,
                changePercent
            });
        }
    }
    
    startSimulation() {
        if (this.isActive) return;
        
        this.isActive = true;
        this.simulationInterval = setInterval(() => {
            // Simulate price changes
            const change = (Math.random() - 0.5) * 10; // Random change between -5 and +5
            const newPrice = Math.max(0.01, this.price + change);
            this.updatePrice(Number(newPrice.toFixed(2)));
        }, 1000);
        
        this.emit('simulationStarted', { symbol: this.symbol });
    }
    
    stopSimulation() {
        if (!this.isActive) return;
        
        this.isActive = false;
        clearInterval(this.simulationInterval);
        this.emit('simulationStopped', { symbol: this.symbol });
    }
    
    getHistory() {
        return [...this.history];
    }
}

class Portfolio {
    constructor() {
        this.stocks = new Map();
        this.totalValue = 0;
        this.alerts = [];
    }
    
    addStock(symbol, quantity, stock) {
        this.stocks.set(symbol, { quantity, stock });
        
        // Subscribe to stock price updates
        stock.on('priceUpdate', (data) => {
            this.updatePortfolioValue();
            console.log(`Portfolio: ${data.symbol} updated to $${data.price}`);
        });
        
        stock.on('significantChange', (data) => {
            this.alerts.push({
                type: 'significant_change',
                message: `${data.symbol} changed by ${data.changePercent}%`,
                timestamp: new Date()
            });
            console.warn(`ALERT: ${data.symbol} significant change: ${data.changePercent}%`);
        });
    }
    
    updatePortfolioValue() {
        this.totalValue = 0;
        this.stocks.forEach(({ quantity, stock }) => {
            this.totalValue += quantity * stock.price;
        });
        console.log(`Portfolio total value: $${this.totalValue.toFixed(2)}`);
    }
    
    getAlerts() {
        return [...this.alerts];
    }
}

class TradingBot {
    constructor(name) {
        this.name = name;
        this.trades = [];
    }
    
    watchStock(stock) {
        stock.on('priceDecrease', (data) => {
            if (Math.abs(data.change) > 2) {
                this.buySignal(data);
            }
        });
        
        stock.on('priceIncrease', (data) => {
            if (data.change > 3) {
                this.sellSignal(data);
            }
        });
    }
    
    buySignal(data) {
        const trade = {
            type: 'BUY',
            symbol: data.symbol,
            price: data.price,
            timestamp: new Date()
        };
        
        this.trades.push(trade);
        console.log(`${this.name} - BUY signal for ${data.symbol} at $${data.price}`);
    }
    
    sellSignal(data) {
        const trade = {
            type: 'SELL',
            symbol: data.symbol,
            price: data.price,
            timestamp: new Date()
        };
        
        this.trades.push(trade);
        console.log(`${this.name} - SELL signal for ${data.symbol} at $${data.price}`);
    }
    
    getTrades() {
        return [...this.trades];
    }
}

// Usage
const appleStock = new StockPrice('AAPL', 150.00);
const googleStock = new StockPrice('GOOGL', 2800.00);

const portfolio = new Portfolio();
portfolio.addStock('AAPL', 10, appleStock);
portfolio.addStock('GOOGL', 5, googleStock);

const tradingBot = new TradingBot('AlgoBot 3000');
tradingBot.watchStock(appleStock);
tradingBot.watchStock(googleStock);

// Start price simulations
appleStock.startSimulation();
googleStock.startSimulation();

// Stop after 10 seconds
setTimeout(() => {
    appleStock.stopSimulation();
    googleStock.stopSimulation();
    
    console.log('Final portfolio alerts:', portfolio.getAlerts());
    console.log('Bot trades:', tradingBot.getTrades());
}, 10000);
```

### Shopping Cart with Observers
```javascript
class ShoppingCart extends EventEmitter {
    constructor() {
        super();
        this.items = [];
        this.discounts = [];
    }
    
    addItem(product, quantity = 1) {
        const existingItem = this.items.find(item => item.product.id === product.id);
        
        if (existingItem) {
            existingItem.quantity += quantity;
        } else {
            this.items.push({ product, quantity });
        }
        
        this.emit('itemAdded', { product, quantity, cart: this.getCartSummary() });
        this.emit('cartChanged', this.getCartSummary());
    }
    
    removeItem(productId) {
        const index = this.items.findIndex(item => item.product.id === productId);
        
        if (index !== -1) {
            const removedItem = this.items.splice(index, 1)[0];
            this.emit('itemRemoved', { 
                product: removedItem.product, 
                quantity: removedItem.quantity,
                cart: this.getCartSummary() 
            });
            this.emit('cartChanged', this.getCartSummary());
        }
    }
    
    updateQuantity(productId, quantity) {
        const item = this.items.find(item => item.product.id === productId);
        
        if (item) {
            const oldQuantity = item.quantity;
            item.quantity = quantity;
            
            this.emit('quantityUpdated', {
                product: item.product,
                oldQuantity,
                newQuantity: quantity,
                cart: this.getCartSummary()
            });
            this.emit('cartChanged', this.getCartSummary());
        }
    }
    
    applyDiscount(discount) {
        this.discounts.push(discount);
        this.emit('discountApplied', { discount, cart: this.getCartSummary() });
        this.emit('cartChanged', this.getCartSummary());
    }
    
    getCartSummary() {
        const subtotal = this.items.reduce((total, item) => {
            return total + (item.product.price * item.quantity);
        }, 0);
        
        const discountAmount = this.discounts.reduce((total, discount) => {
            return total + (discount.type === 'percentage' 
                ? subtotal * discount.value / 100 
                : discount.value);
        }, 0);
        
        return {
            itemCount: this.items.reduce((total, item) => total + item.quantity, 0),
            subtotal: Number(subtotal.toFixed(2)),
            discountAmount: Number(discountAmount.toFixed(2)),
            total: Number((subtotal - discountAmount).toFixed(2)),
            items: [...this.items]
        };
    }
}

class InventoryManager {
    constructor() {
        this.inventory = new Map();
    }
    
    setStock(productId, quantity) {
        this.inventory.set(productId, quantity);
    }
    
    watchCart(cart) {
        cart.on('itemAdded', ({ product, quantity }) => {
            this.updateStock(product.id, -quantity);
        });
        
        cart.on('itemRemoved', ({ product, quantity }) => {
            this.updateStock(product.id, quantity);
        });
        
        cart.on('quantityUpdated', ({ product, oldQuantity, newQuantity }) => {
            const difference = newQuantity - oldQuantity;
            this.updateStock(product.id, -difference);
        });
    }
    
    updateStock(productId, change) {
        const currentStock = this.inventory.get(productId) || 0;
        const newStock = currentStock + change;
        this.inventory.set(productId, Math.max(0, newStock));
        
        console.log(`Inventory: Product ${productId} stock updated to ${newStock}`);
        
        if (newStock <= 5) {
            console.warn(`LOW STOCK ALERT: Product ${productId} has only ${newStock} items left`);
        }
    }
    
    getStock(productId) {
        return this.inventory.get(productId) || 0;
    }
}

class PriceCalculator {
    watchCart(cart) {
        cart.on('cartChanged', (summary) => {
            console.log('Price Calculator - Cart Summary:');
            console.log(`Items: ${summary.itemCount}`);
            console.log(`Subtotal: $${summary.subtotal}`);
            console.log(`Discounts: -$${summary.discountAmount}`);
            console.log(`Total: $${summary.total}`);
            console.log('---');
        });
    }
}

class RecommendationEngine {
    constructor() {
        this.recommendations = new Map();
    }
    
    watchCart(cart) {
        cart.on('itemAdded', ({ product }) => {
            this.generateRecommendations(product);
        });
    }
    
    generateRecommendations(product) {
        // Simple recommendation logic
        const recommendations = [
            { id: 999, name: `Accessory for ${product.name}`, price: 19.99 },
            { id: 998, name: `Premium ${product.name}`, price: product.price * 1.5 }
        ];
        
        console.log(`Recommendations based on ${product.name}:`, recommendations);
    }
}

// Usage
const cart = new ShoppingCart();
const inventory = new InventoryManager();
const priceCalculator = new PriceCalculator();
const recommendationEngine = new RecommendationEngine();

// Set up observers
inventory.watchCart(cart);
priceCalculator.watchCart(cart);
recommendationEngine.watchCart(cart);

// Set initial inventory
inventory.setStock(1, 50);
inventory.setStock(2, 3); // Low stock item

// Add items to cart
cart.addItem({ id: 1, name: 'Laptop', price: 999.99 }, 2);
cart.addItem({ id: 2, name: 'Mouse', price: 29.99 }, 1);

// Apply discount
cart.applyDiscount({ type: 'percentage', value: 10 });

// Update quantity
cart.updateQuantity(1, 3);
```

## 🎯 Interview Questions

### Q1: What's the difference between Observer and Pub/Sub patterns?
**Answer:**
- **Observer**: Direct relationship between subject and observers
- **Pub/Sub**: Uses a message broker/event channel, publishers and subscribers don't know each other
- **Observer** is tightly coupled, **Pub/Sub** is loosely coupled
- **Pub/Sub** can handle complex routing and filtering

### Q2: How do you prevent memory leaks in Observer pattern?
**Answer:**
- Always provide unsubscribe/removeListener methods
- Use WeakMap for storing observers when possible
- Implement cleanup in component lifecycle methods
- Use once() for one-time listeners
- Clear all listeners when objects are destroyed

### Q3: How does Observer pattern relate to MVC architecture?
**Answer:**
- **Model** is the subject that notifies observers
- **View** is the observer that updates when model changes
- **Controller** mediates between model and view
- Enables automatic UI updates when data changes

### Q4: What are the performance considerations with Observer pattern?
**Answer:**
- Too many observers can slow down notifications
- Synchronous notifications can block the main thread
- Consider batching updates for multiple rapid changes
- Use debouncing/throttling for high-frequency events
- Implement priority-based notification ordering

## 💡 Best Practices

1. **Always provide unsubscribe mechanism** - Prevent memory leaks
2. **Handle errors in observers** - Don't let one observer break others
3. **Use meaningful event names** - Make code self-documenting
4. **Consider async notifications** - For heavy operations
5. **Implement once() for one-time events** - Automatic cleanup
6. **Document your events** - What data they provide, when they fire

## 🔗 Related Patterns
- **Mediator Pattern** - Alternative for complex observer relationships
- **Command Pattern** - Commands can be observed for undo/redo
- **State Pattern** - State changes can notify observers
- **MVC/MVP/MVVM** - Observer is fundamental to these architectures
