# Factory Pattern 🏭

## 📖 Definition
The Factory pattern provides an interface for creating objects without specifying their exact class. It encapsulates object creation logic and returns instances based on given parameters.

## 🎯 When to Use
- When you need to create objects without knowing their exact types
- When object creation is complex and needs to be centralized
- When you want to decouple object creation from usage
- When you need to create different types of objects based on conditions

## ✅ Pros
- Encapsulates object creation logic
- Reduces code duplication
- Makes code more maintainable
- Follows Open/Closed Principle
- Provides flexibility in object creation

## ❌ Cons
- Can add unnecessary complexity for simple cases
- May create too many small classes
- Can make code harder to understand if overused

## 🚀 JavaScript Implementations

### 1. Simple Factory
```javascript
class Car {
    constructor(make, model, year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }
    
    getInfo() {
        return `${this.year} ${this.make} ${this.model}`;
    }
}

class Truck {
    constructor(make, model, year, capacity) {
        this.make = make;
        this.model = model;
        this.year = year;
        this.capacity = capacity;
    }
    
    getInfo() {
        return `${this.year} ${this.make} ${this.model} (${this.capacity} tons)`;
    }
}

class Motorcycle {
    constructor(make, model, year, engineSize) {
        this.make = make;
        this.model = model;
        this.year = year;
        this.engineSize = engineSize;
    }
    
    getInfo() {
        return `${this.year} ${this.make} ${this.model} (${this.engineSize}cc)`;
    }
}

// Simple Factory
class VehicleFactory {
    static createVehicle(type, make, model, year, extra) {
        switch (type.toLowerCase()) {
            case 'car':
                return new Car(make, model, year);
            case 'truck':
                return new Truck(make, model, year, extra);
            case 'motorcycle':
                return new Motorcycle(make, model, year, extra);
            default:
                throw new Error(`Vehicle type "${type}" is not supported`);
        }
    }
}

// Usage
const car = VehicleFactory.createVehicle('car', 'Toyota', 'Camry', 2023);
const truck = VehicleFactory.createVehicle('truck', 'Ford', 'F-150', 2023, 2.5);
const bike = VehicleFactory.createVehicle('motorcycle', 'Honda', 'CBR', 2023, 600);

console.log(car.getInfo()); // 2023 Toyota Camry
console.log(truck.getInfo()); // 2023 Ford F-150 (2.5 tons)
console.log(bike.getInfo()); // 2023 Honda CBR (600cc)
```

### 2. Factory Method Pattern
```javascript
// Abstract Creator
class VehicleCreator {
    createVehicle() {
        throw new Error('createVehicle method must be implemented');
    }
    
    deliverVehicle(customerInfo) {
        const vehicle = this.createVehicle();
        console.log(`Delivering ${vehicle.getInfo()} to ${customerInfo.name}`);
        return vehicle;
    }
}

// Concrete Creators
class CarCreator extends VehicleCreator {
    constructor(make, model, year) {
        super();
        this.make = make;
        this.model = model;
        this.year = year;
    }
    
    createVehicle() {
        return new Car(this.make, this.model, this.year);
    }
}

class TruckCreator extends VehicleCreator {
    constructor(make, model, year, capacity) {
        super();
        this.make = make;
        this.model = model;
        this.year = year;
        this.capacity = capacity;
    }
    
    createVehicle() {
        return new Truck(this.make, this.model, this.year, this.capacity);
    }
}

// Usage
const carCreator = new CarCreator('BMW', 'X5', 2023);
const truckCreator = new TruckCreator('Mercedes', 'Actros', 2023, 40);

const customerInfo = { name: 'John Doe', address: '123 Main St' };
const car = carCreator.deliverVehicle(customerInfo);
const truck = truckCreator.deliverVehicle(customerInfo);
```

### 3. Abstract Factory Pattern
```javascript
// Abstract Products
class Engine {
    start() {
        throw new Error('start method must be implemented');
    }
}

class Transmission {
    shift() {
        throw new Error('shift method must be implemented');
    }
}

// Concrete Products for Luxury Cars
class LuxuryEngine extends Engine {
    start() {
        return 'Luxury V8 engine started smoothly';
    }
}

class LuxuryTransmission extends Transmission {
    shift() {
        return 'Smooth automatic transmission shift';
    }
}

// Concrete Products for Economy Cars
class EconomyEngine extends Engine {
    start() {
        return 'Economy 4-cylinder engine started';
    }
}

class EconomyTransmission extends Transmission {
    shift() {
        return 'Manual transmission engaged';
    }
}

// Abstract Factory
class CarPartsFactory {
    createEngine() {
        throw new Error('createEngine method must be implemented');
    }
    
    createTransmission() {
        throw new Error('createTransmission method must be implemented');
    }
}

// Concrete Factories
class LuxuryCarPartsFactory extends CarPartsFactory {
    createEngine() {
        return new LuxuryEngine();
    }
    
    createTransmission() {
        return new LuxuryTransmission();
    }
}

class EconomyCarPartsFactory extends CarPartsFactory {
    createEngine() {
        return new EconomyEngine();
    }
    
    createTransmission() {
        return new EconomyTransmission();
    }
}

// Car Assembly
class CarAssembly {
    constructor(partsFactory) {
        this.engine = partsFactory.createEngine();
        this.transmission = partsFactory.createTransmission();
    }
    
    assemble() {
        return {
            engineStatus: this.engine.start(),
            transmissionStatus: this.transmission.shift()
        };
    }
}

// Usage
const luxuryFactory = new LuxuryCarPartsFactory();
const economyFactory = new EconomyCarPartsFactory();

const luxuryCar = new CarAssembly(luxuryFactory);
const economyCar = new CarAssembly(economyFactory);

console.log('Luxury Car:', luxuryCar.assemble());
console.log('Economy Car:', economyCar.assemble());
```

## 🌟 Real-World Examples

### HTTP Client Factory
```javascript
class HttpClient {
    constructor(baseURL, timeout = 5000) {
        this.baseURL = baseURL;
        this.timeout = timeout;
    }
    
    async request(method, endpoint, data = null) {
        const url = `${this.baseURL}${endpoint}`;
        const options = {
            method,
            headers: { 'Content-Type': 'application/json' },
            timeout: this.timeout
        };
        
        if (data) {
            options.body = JSON.stringify(data);
        }
        
        try {
            const response = await fetch(url, options);
            return await response.json();
        } catch (error) {
            throw new Error(`HTTP ${method} request failed: ${error.message}`);
        }
    }
}

class HttpClientFactory {
    static clients = new Map();
    
    static createClient(type, config = {}) {
        // Return cached client if exists
        const cacheKey = `${type}_${JSON.stringify(config)}`;
        if (this.clients.has(cacheKey)) {
            return this.clients.get(cacheKey);
        }
        
        let client;
        
        switch (type.toLowerCase()) {
            case 'api':
                client = new HttpClient(
                    config.baseURL || 'https://api.example.com',
                    config.timeout || 10000
                );
                break;
                
            case 'auth':
                client = new HttpClient(
                    config.baseURL || 'https://auth.example.com',
                    config.timeout || 5000
                );
                break;
                
            case 'cdn':
                client = new HttpClient(
                    config.baseURL || 'https://cdn.example.com',
                    config.timeout || 15000
                );
                break;
                
            default:
                throw new Error(`HTTP client type "${type}" is not supported`);
        }
        
        // Cache the client
        this.clients.set(cacheKey, client);
        return client;
    }
    
    static clearCache() {
        this.clients.clear();
    }
}

// Usage
const apiClient = HttpClientFactory.createClient('api', {
    baseURL: 'https://jsonplaceholder.typicode.com',
    timeout: 8000
});

const authClient = HttpClientFactory.createClient('auth', {
    baseURL: 'https://auth.myapp.com'
});

// Same configuration returns cached instance
const apiClient2 = HttpClientFactory.createClient('api', {
    baseURL: 'https://jsonplaceholder.typicode.com',
    timeout: 8000
});

console.log(apiClient === apiClient2); // true (cached)
```

### UI Component Factory
```javascript
// Base Component
class UIComponent {
    constructor(props = {}) {
        this.props = props;
        this.element = null;
    }
    
    render() {
        throw new Error('render method must be implemented');
    }
    
    mount(parent) {
        if (!this.element) {
            this.element = this.render();
        }
        parent.appendChild(this.element);
    }
}

// Concrete Components
class Button extends UIComponent {
    render() {
        const button = document.createElement('button');
        button.textContent = this.props.text || 'Button';
        button.className = this.props.className || 'btn';
        
        if (this.props.onClick) {
            button.addEventListener('click', this.props.onClick);
        }
        
        return button;
    }
}

class Input extends UIComponent {
    render() {
        const input = document.createElement('input');
        input.type = this.props.type || 'text';
        input.placeholder = this.props.placeholder || '';
        input.className = this.props.className || 'input';
        
        if (this.props.onChange) {
            input.addEventListener('input', this.props.onChange);
        }
        
        return input;
    }
}

class Modal extends UIComponent {
    render() {
        const modal = document.createElement('div');
        modal.className = 'modal';
        modal.innerHTML = `
            <div class="modal-content">
                <span class="close">&times;</span>
                <h2>${this.props.title || 'Modal'}</h2>
                <p>${this.props.content || 'Modal content'}</p>
            </div>
        `;
        
        const closeBtn = modal.querySelector('.close');
        closeBtn.addEventListener('click', () => {
            modal.style.display = 'none';
        });
        
        return modal;
    }
}

// UI Component Factory
class UIComponentFactory {
    static componentTypes = {
        button: Button,
        input: Input,
        modal: Modal
    };
    
    static createComponent(type, props = {}) {
        const ComponentClass = this.componentTypes[type.toLowerCase()];
        
        if (!ComponentClass) {
            throw new Error(`Component type "${type}" is not supported`);
        }
        
        return new ComponentClass(props);
    }
    
    static registerComponent(type, ComponentClass) {
        this.componentTypes[type.toLowerCase()] = ComponentClass;
    }
}

// Usage
const submitButton = UIComponentFactory.createComponent('button', {
    text: 'Submit',
    className: 'btn btn-primary',
    onClick: () => console.log('Form submitted!')
});

const emailInput = UIComponentFactory.createComponent('input', {
    type: 'email',
    placeholder: 'Enter your email',
    onChange: (e) => console.log('Email:', e.target.value)
});

const confirmModal = UIComponentFactory.createComponent('modal', {
    title: 'Confirm Action',
    content: 'Are you sure you want to proceed?'
});

// Mount components
document.body.appendChild(submitButton.render());
document.body.appendChild(emailInput.render());
document.body.appendChild(confirmModal.render());
```

## 🎯 Interview Questions

### Q1: What's the difference between Simple Factory and Factory Method?
**Answer:**
- **Simple Factory**: Uses a single factory class with conditional logic to create objects
- **Factory Method**: Uses inheritance, each subclass implements its own factory method
- **Simple Factory** violates Open/Closed Principle, **Factory Method** follows it

### Q2: When would you use Abstract Factory over Factory Method?
**Answer:**
- When you need to create families of related objects
- When you want to ensure objects from the same family are used together
- When you need to support multiple product lines
- Example: Creating UI components for different themes (Dark/Light)

### Q3: How does Factory pattern help with testing?
**Answer:**
```javascript
// Without Factory - Hard to test
class UserService {
    constructor() {
        this.httpClient = new HttpClient('https://api.example.com');
    }
}

// With Factory - Easy to test
class UserService {
    constructor(httpClientFactory) {
        this.httpClient = httpClientFactory.createClient('api');
    }
}

// In tests, inject mock factory
const mockFactory = {
    createClient: () => mockHttpClient
};
const userService = new UserService(mockFactory);
```

### Q4: What are the alternatives to Factory pattern?
**Answer:**
- **Dependency Injection**: Inject dependencies instead of creating them
- **Builder Pattern**: For complex object construction
- **Prototype Pattern**: For cloning existing objects
- **Service Locator**: For service discovery

## 💡 Best Practices

1. **Use when creation logic is complex** - Don't over-engineer simple cases
2. **Follow naming conventions** - Use clear, descriptive names
3. **Consider caching** - Cache expensive-to-create objects
4. **Make it extensible** - Allow registration of new types
5. **Handle errors gracefully** - Provide meaningful error messages
6. **Document supported types** - Make it clear what can be created

## 🔗 Related Patterns
- **Builder Pattern** - For step-by-step object construction
- **Prototype Pattern** - For cloning objects instead of creating new ones
- **Singleton Pattern** - Factory itself can be a singleton
- **Strategy Pattern** - Different creation strategies
