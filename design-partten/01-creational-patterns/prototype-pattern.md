# Prototype Pattern 🧬

## 📖 Definition
The Prototype pattern creates objects by cloning existing instances rather than creating new ones from scratch. It's useful when object creation is expensive or complex.

## 🎯 When to Use
- When object creation is expensive (database calls, network requests)
- When you need to create objects similar to existing ones
- When you want to avoid subclassing for object creation
- When you need to create objects at runtime based on dynamic configuration

## ✅ Pros
- Reduces object creation cost
- Simplifies object creation process
- Allows runtime object creation
- Reduces need for subclassing
- Can create objects without knowing their exact types

## ❌ Cons
- Cloning complex objects with circular references can be tricky
- Deep vs shallow copy considerations
- May require implementing clone methods for all classes

## 🚀 JavaScript Implementations

### 1. Basic Prototype Pattern
```javascript
class Shape {
    constructor(type, color, x = 0, y = 0) {
        this.type = type;
        this.color = color;
        this.x = x;
        this.y = y;
    }
    
    // Shallow clone
    clone() {
        return Object.create(
            Object.getPrototypeOf(this),
            Object.getOwnPropertyDescriptors(this)
        );
    }
    
    // Alternative shallow clone
    shallowClone() {
        return Object.assign(Object.create(Object.getPrototypeOf(this)), this);
    }
    
    move(x, y) {
        this.x = x;
        this.y = y;
    }
    
    draw() {
        console.log(`Drawing ${this.color} ${this.type} at (${this.x}, ${this.y})`);
    }
}

class Circle extends Shape {
    constructor(color, x, y, radius) {
        super('circle', color, x, y);
        this.radius = radius;
    }
    
    draw() {
        console.log(`Drawing ${this.color} circle with radius ${this.radius} at (${this.x}, ${this.y})`);
    }
}

class Rectangle extends Shape {
    constructor(color, x, y, width, height) {
        super('rectangle', color, x, y);
        this.width = width;
        this.height = height;
    }
    
    draw() {
        console.log(`Drawing ${this.color} rectangle ${this.width}x${this.height} at (${this.x}, ${this.y})`);
    }
}

// Usage
const originalCircle = new Circle('red', 10, 20, 15);
const clonedCircle = originalCircle.clone();

clonedCircle.color = 'blue';
clonedCircle.move(50, 60);

originalCircle.draw(); // Drawing red circle with radius 15 at (10, 20)
clonedCircle.draw();   // Drawing blue circle with radius 15 at (50, 60)
```

### 2. Deep Clone Implementation
```javascript
class ComplexObject {
    constructor(name, config = {}) {
        this.name = name;
        this.config = config;
        this.metadata = {
            created: new Date(),
            version: '1.0.0',
            tags: []
        };
        this.children = [];
    }
    
    addChild(child) {
        this.children.push(child);
    }
    
    addTag(tag) {
        this.metadata.tags.push(tag);
    }
    
    // Deep clone implementation
    deepClone() {
        // Handle primitive types and null
        if (this === null || typeof this !== 'object') {
            return this;
        }
        
        // Handle Date
        if (this instanceof Date) {
            return new Date(this.getTime());
        }
        
        // Handle Array
        if (Array.isArray(this)) {
            return this.map(item => item && typeof item === 'object' ? item.deepClone() : item);
        }
        
        // Handle Object
        const cloned = Object.create(Object.getPrototypeOf(this));
        
        for (const key in this) {
            if (this.hasOwnProperty(key)) {
                const value = this[key];
                if (value && typeof value === 'object') {
                    if (value.deepClone && typeof value.deepClone === 'function') {
                        cloned[key] = value.deepClone();
                    } else if (value instanceof Date) {
                        cloned[key] = new Date(value.getTime());
                    } else if (Array.isArray(value)) {
                        cloned[key] = value.map(item => 
                            item && typeof item === 'object' && item.deepClone 
                                ? item.deepClone() 
                                : item
                        );
                    } else {
                        cloned[key] = JSON.parse(JSON.stringify(value));
                    }
                } else {
                    cloned[key] = value;
                }
            }
        }
        
        return cloned;
    }
    
    // Alternative using JSON (limited but simple)
    jsonClone() {
        return Object.assign(
            Object.create(Object.getPrototypeOf(this)),
            JSON.parse(JSON.stringify(this))
        );
    }
}

// Usage
const original = new ComplexObject('Original', { theme: 'dark', lang: 'en' });
original.addTag('important');
original.addTag('prototype');

const child = new ComplexObject('Child', { nested: true });
original.addChild(child);

const deepCloned = original.deepClone();
deepCloned.name = 'Deep Clone';
deepCloned.config.theme = 'light';
deepCloned.metadata.tags.push('cloned');

console.log('Original:', original);
console.log('Deep Cloned:', deepCloned);
console.log('Tags are separate:', original.metadata.tags !== deepCloned.metadata.tags);
```

### 3. Prototype Registry Pattern
```javascript
class PrototypeRegistry {
    constructor() {
        this.prototypes = new Map();
    }
    
    register(name, prototype) {
        this.prototypes.set(name, prototype);
    }
    
    unregister(name) {
        this.prototypes.delete(name);
    }
    
    create(name, customizations = {}) {
        const prototype = this.prototypes.get(name);
        if (!prototype) {
            throw new Error(`Prototype '${name}' not found`);
        }
        
        const cloned = prototype.clone();
        
        // Apply customizations
        Object.assign(cloned, customizations);
        
        return cloned;
    }
    
    list() {
        return Array.from(this.prototypes.keys());
    }
}

// Document templates
class Document {
    constructor(title, content, template) {
        this.title = title;
        this.content = content;
        this.template = template;
        this.created = new Date();
        this.metadata = {};
    }
    
    clone() {
        const cloned = Object.create(Object.getPrototypeOf(this));
        cloned.title = this.title;
        cloned.content = this.content;
        cloned.template = this.template;
        cloned.created = new Date(); // New creation time
        cloned.metadata = { ...this.metadata }; // Shallow copy metadata
        return cloned;
    }
    
    setMetadata(key, value) {
        this.metadata[key] = value;
    }
}

// Setup registry with templates
const documentRegistry = new PrototypeRegistry();

// Register document templates
documentRegistry.register('letter', new Document(
    'Business Letter Template',
    'Dear [Recipient],\n\n[Content]\n\nSincerely,\n[Sender]',
    'business-letter'
));

documentRegistry.register('invoice', new Document(
    'Invoice Template',
    'Invoice #[Number]\nDate: [Date]\nBill To: [Client]\n\nItems:\n[Items]\n\nTotal: [Amount]',
    'invoice'
));

documentRegistry.register('report', new Document(
    'Report Template',
    'Executive Summary\n[Summary]\n\nDetails\n[Details]\n\nConclusion\n[Conclusion]',
    'report'
));

// Usage
const businessLetter = documentRegistry.create('letter', {
    title: 'Welcome Letter',
    metadata: { recipient: 'John Doe', sender: 'Jane Smith' }
});

const monthlyReport = documentRegistry.create('report', {
    title: 'Monthly Sales Report',
    metadata: { month: 'January', year: 2024 }
});

console.log('Available templates:', documentRegistry.list());
console.log('Business Letter:', businessLetter);
console.log('Monthly Report:', monthlyReport);
```

## 🌟 Real-World Examples

### Game Object Prototypes
```javascript
class GameObject {
    constructor(name, sprite, x = 0, y = 0) {
        this.name = name;
        this.sprite = sprite;
        this.x = x;
        this.y = y;
        this.health = 100;
        this.speed = 1;
        this.abilities = [];
        this.inventory = [];
    }
    
    clone() {
        const cloned = Object.create(Object.getPrototypeOf(this));
        cloned.name = this.name;
        cloned.sprite = this.sprite;
        cloned.x = this.x;
        cloned.y = this.y;
        cloned.health = this.health;
        cloned.speed = this.speed;
        cloned.abilities = [...this.abilities]; // Shallow copy arrays
        cloned.inventory = [...this.inventory];
        return cloned;
    }
    
    addAbility(ability) {
        this.abilities.push(ability);
    }
    
    addItem(item) {
        this.inventory.push(item);
    }
    
    move(dx, dy) {
        this.x += dx * this.speed;
        this.y += dy * this.speed;
    }
}

class Enemy extends GameObject {
    constructor(name, sprite, x, y, damage) {
        super(name, sprite, x, y);
        this.damage = damage;
        this.aggressive = true;
    }
    
    attack(target) {
        console.log(`${this.name} attacks ${target.name} for ${this.damage} damage!`);
        target.health -= this.damage;
    }
}

class PowerUp extends GameObject {
    constructor(name, sprite, x, y, effect) {
        super(name, sprite, x, y);
        this.effect = effect;
        this.consumed = false;
    }
    
    apply(target) {
        if (!this.consumed) {
            console.log(`${target.name} uses ${this.name}!`);
            this.effect(target);
            this.consumed = true;
        }
    }
}

// Game Object Factory using Prototypes
class GameObjectFactory {
    constructor() {
        this.prototypes = new Map();
        this.setupPrototypes();
    }
    
    setupPrototypes() {
        // Enemy prototypes
        this.prototypes.set('goblin', new Enemy('Goblin', 'goblin.png', 0, 0, 15));
        this.prototypes.set('orc', new Enemy('Orc', 'orc.png', 0, 0, 25));
        this.prototypes.set('dragon', new Enemy('Dragon', 'dragon.png', 0, 0, 50));
        
        // PowerUp prototypes
        this.prototypes.set('health-potion', new PowerUp(
            'Health Potion', 
            'health-potion.png', 
            0, 0, 
            (target) => { target.health += 50; }
        ));
        
        this.prototypes.set('speed-boost', new PowerUp(
            'Speed Boost', 
            'speed-boost.png', 
            0, 0, 
            (target) => { target.speed *= 2; }
        ));
    }
    
    create(type, x, y, customizations = {}) {
        const prototype = this.prototypes.get(type);
        if (!prototype) {
            throw new Error(`Unknown game object type: ${type}`);
        }
        
        const gameObject = prototype.clone();
        gameObject.x = x;
        gameObject.y = y;
        
        // Apply customizations
        Object.assign(gameObject, customizations);
        
        return gameObject;
    }
    
    registerPrototype(name, prototype) {
        this.prototypes.set(name, prototype);
    }
}

// Usage
const factory = new GameObjectFactory();

// Create enemies at different positions
const goblin1 = factory.create('goblin', 100, 200);
const goblin2 = factory.create('goblin', 150, 250, { name: 'Elite Goblin', damage: 20 });
const dragon = factory.create('dragon', 500, 600);

// Create power-ups
const healthPotion = factory.create('health-potion', 75, 125);
const speedBoost = factory.create('speed-boost', 200, 300);

console.log('Goblin 1:', goblin1);
console.log('Elite Goblin:', goblin2);
console.log('Dragon:', dragon);
```

### Configuration Templates
```javascript
class Configuration {
    constructor(name) {
        this.name = name;
        this.database = {};
        this.server = {};
        this.logging = {};
        this.features = {};
        this.security = {};
    }
    
    clone() {
        const cloned = new Configuration(this.name);
        cloned.database = { ...this.database };
        cloned.server = { ...this.server };
        cloned.logging = { ...this.logging };
        cloned.features = { ...this.features };
        cloned.security = { ...this.security };
        return cloned;
    }
    
    setDatabase(config) {
        Object.assign(this.database, config);
        return this;
    }
    
    setServer(config) {
        Object.assign(this.server, config);
        return this;
    }
    
    setLogging(config) {
        Object.assign(this.logging, config);
        return this;
    }
    
    setFeatures(config) {
        Object.assign(this.features, config);
        return this;
    }
    
    setSecurity(config) {
        Object.assign(this.security, config);
        return this;
    }
}

class ConfigurationManager {
    constructor() {
        this.templates = new Map();
        this.setupDefaultTemplates();
    }
    
    setupDefaultTemplates() {
        // Development template
        const devTemplate = new Configuration('development')
            .setDatabase({
                host: 'localhost',
                port: 5432,
                name: 'app_dev',
                ssl: false
            })
            .setServer({
                port: 3000,
                host: '0.0.0.0',
                cors: true
            })
            .setLogging({
                level: 'debug',
                console: true,
                file: false
            })
            .setFeatures({
                debugMode: true,
                hotReload: true,
                mockData: true
            })
            .setSecurity({
                jwt: { secret: 'dev-secret', expiresIn: '24h' },
                rateLimit: false
            });
        
        // Production template
        const prodTemplate = new Configuration('production')
            .setDatabase({
                host: process.env.DB_HOST,
                port: 5432,
                name: 'app_prod',
                ssl: true
            })
            .setServer({
                port: process.env.PORT || 8080,
                host: '0.0.0.0',
                cors: false
            })
            .setLogging({
                level: 'error',
                console: false,
                file: true
            })
            .setFeatures({
                debugMode: false,
                hotReload: false,
                mockData: false
            })
            .setSecurity({
                jwt: { secret: process.env.JWT_SECRET, expiresIn: '1h' },
                rateLimit: true
            });
        
        // Testing template
        const testTemplate = new Configuration('testing')
            .setDatabase({
                host: 'localhost',
                port: 5432,
                name: 'app_test',
                ssl: false
            })
            .setServer({
                port: 3001,
                host: 'localhost',
                cors: true
            })
            .setLogging({
                level: 'warn',
                console: true,
                file: false
            })
            .setFeatures({
                debugMode: false,
                hotReload: false,
                mockData: true
            })
            .setSecurity({
                jwt: { secret: 'test-secret', expiresIn: '1h' },
                rateLimit: false
            });
        
        this.templates.set('development', devTemplate);
        this.templates.set('production', prodTemplate);
        this.templates.set('testing', testTemplate);
    }
    
    createConfiguration(templateName, overrides = {}) {
        const template = this.templates.get(templateName);
        if (!template) {
            throw new Error(`Configuration template '${templateName}' not found`);
        }
        
        const config = template.clone();
        
        // Apply overrides
        if (overrides.database) config.setDatabase(overrides.database);
        if (overrides.server) config.setServer(overrides.server);
        if (overrides.logging) config.setLogging(overrides.logging);
        if (overrides.features) config.setFeatures(overrides.features);
        if (overrides.security) config.setSecurity(overrides.security);
        
        return config;
    }
    
    registerTemplate(name, template) {
        this.templates.set(name, template);
    }
}

// Usage
const configManager = new ConfigurationManager();

// Create configurations for different environments
const devConfig = configManager.createConfiguration('development');
const prodConfig = configManager.createConfiguration('production');

// Create custom configuration based on development template
const customConfig = configManager.createConfiguration('development', {
    server: { port: 4000 },
    database: { name: 'custom_app_dev' },
    features: { experimentalFeatures: true }
});

console.log('Development Config:', devConfig);
console.log('Production Config:', prodConfig);
console.log('Custom Config:', customConfig);
```

## 🎯 Interview Questions

### Q1: What's the difference between shallow and deep cloning?
**Answer:**
- **Shallow Clone**: Copies only the top-level properties. Nested objects are still referenced
- **Deep Clone**: Recursively copies all nested objects and arrays
```javascript
const original = { a: 1, b: { c: 2 } };
const shallow = { ...original };
const deep = JSON.parse(JSON.stringify(original));

shallow.b.c = 3; // This affects original.b.c
deep.b.c = 4;    // This doesn't affect original
```

### Q2: When would you use Prototype pattern over Factory pattern?
**Answer:**
- **Prototype**: When you have expensive object creation and want to clone existing instances
- **Factory**: When you need to create different types of objects based on parameters
- **Prototype** is better for customizing existing objects, **Factory** for creating new types

### Q3: How do you handle circular references in cloning?
**Answer:**
```javascript
function deepCloneWithCircular(obj, visited = new WeakMap()) {
    if (obj === null || typeof obj !== 'object') return obj;
    if (visited.has(obj)) return visited.get(obj);
    
    const cloned = Array.isArray(obj) ? [] : {};
    visited.set(obj, cloned);
    
    for (const key in obj) {
        if (obj.hasOwnProperty(key)) {
            cloned[key] = deepCloneWithCircular(obj[key], visited);
        }
    }
    
    return cloned;
}
```

### Q4: What are the limitations of JSON.parse(JSON.stringify()) for cloning?
**Answer:**
- Doesn't handle functions, undefined, symbols
- Converts dates to strings
- Loses prototype chain
- Can't handle circular references
- Doesn't preserve Map, Set, RegExp objects

## 💡 Best Practices

1. **Choose appropriate cloning depth** - Shallow vs deep based on needs
2. **Handle circular references** - Use WeakMap to track visited objects
3. **Consider performance** - Cloning can be expensive for large objects
4. **Preserve prototype chain** - Use Object.create() when needed
5. **Implement custom clone methods** - For complex objects with special requirements
6. **Use registry pattern** - For managing multiple prototypes

## 🔗 Related Patterns
- **Factory Pattern** - Alternative approach to object creation
- **Builder Pattern** - For step-by-step object construction
- **Flyweight Pattern** - For sharing common object parts
- **Command Pattern** - Commands can be prototyped for undo/redo
