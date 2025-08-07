# Builder Pattern 🔨

## 📖 Definition
The Builder pattern constructs complex objects step by step. It allows you to produce different types and representations of an object using the same construction code.

## 🎯 When to Use
- When creating objects with many optional parameters
- When object construction is complex and requires multiple steps
- When you want to create different representations of the same object
- When you need to enforce a specific construction sequence

## ✅ Pros
- Allows step-by-step object construction
- Can create different representations of objects
- Isolates complex construction code
- Provides better control over construction process
- Makes code more readable for complex objects

## ❌ Cons
- Increases code complexity for simple objects
- Requires creating multiple new classes
- May be overkill for objects with few properties

## 🚀 JavaScript Implementations

### 1. Basic Builder Pattern
```javascript
class User {
    constructor(builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.email = builder.email;
        this.phone = builder.phone;
        this.address = builder.address;
        this.age = builder.age;
        this.isActive = builder.isActive;
        this.roles = builder.roles;
        this.preferences = builder.preferences;
    }
    
    getFullName() {
        return `${this.firstName} ${this.lastName}`;
    }
    
    toString() {
        return JSON.stringify(this, null, 2);
    }
}

class UserBuilder {
    constructor(firstName, lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.roles = [];
        this.preferences = {};
        this.isActive = true;
    }
    
    setEmail(email) {
        this.email = email;
        return this;
    }
    
    setPhone(phone) {
        this.phone = phone;
        return this;
    }
    
    setAddress(address) {
        this.address = address;
        return this;
    }
    
    setAge(age) {
        this.age = age;
        return this;
    }
    
    setActive(isActive) {
        this.isActive = isActive;
        return this;
    }
    
    addRole(role) {
        this.roles.push(role);
        return this;
    }
    
    setPreference(key, value) {
        this.preferences[key] = value;
        return this;
    }
    
    build() {
        return new User(this);
    }
}

// Usage
const user = new UserBuilder('John', 'Doe')
    .setEmail('john.doe@example.com')
    .setPhone('+1-555-123-4567')
    .setAge(30)
    .addRole('admin')
    .addRole('user')
    .setPreference('theme', 'dark')
    .setPreference('notifications', true)
    .build();

console.log(user.toString());
```

### 2. Fluent Interface Builder
```javascript
class QueryBuilder {
    constructor() {
        this.query = {
            select: [],
            from: '',
            joins: [],
            where: [],
            groupBy: [],
            having: [],
            orderBy: [],
            limit: null,
            offset: null
        };
    }
    
    select(...columns) {
        this.query.select.push(...columns);
        return this;
    }
    
    from(table) {
        this.query.from = table;
        return this;
    }
    
    join(table, condition) {
        this.query.joins.push({ type: 'INNER', table, condition });
        return this;
    }
    
    leftJoin(table, condition) {
        this.query.joins.push({ type: 'LEFT', table, condition });
        return this;
    }
    
    rightJoin(table, condition) {
        this.query.joins.push({ type: 'RIGHT', table, condition });
        return this;
    }
    
    where(condition) {
        this.query.where.push({ type: 'AND', condition });
        return this;
    }
    
    orWhere(condition) {
        this.query.where.push({ type: 'OR', condition });
        return this;
    }
    
    groupBy(...columns) {
        this.query.groupBy.push(...columns);
        return this;
    }
    
    having(condition) {
        this.query.having.push(condition);
        return this;
    }
    
    orderBy(column, direction = 'ASC') {
        this.query.orderBy.push({ column, direction });
        return this;
    }
    
    limit(count) {
        this.query.limit = count;
        return this;
    }
    
    offset(count) {
        this.query.offset = count;
        return this;
    }
    
    build() {
        let sql = '';
        
        // SELECT
        if (this.query.select.length > 0) {
            sql += `SELECT ${this.query.select.join(', ')}`;
        } else {
            sql += 'SELECT *';
        }
        
        // FROM
        if (this.query.from) {
            sql += ` FROM ${this.query.from}`;
        }
        
        // JOINS
        this.query.joins.forEach(join => {
            sql += ` ${join.type} JOIN ${join.table} ON ${join.condition}`;
        });
        
        // WHERE
        if (this.query.where.length > 0) {
            sql += ' WHERE ';
            sql += this.query.where.map((w, index) => {
                if (index === 0) return w.condition;
                return `${w.type} ${w.condition}`;
            }).join(' ');
        }
        
        // GROUP BY
        if (this.query.groupBy.length > 0) {
            sql += ` GROUP BY ${this.query.groupBy.join(', ')}`;
        }
        
        // HAVING
        if (this.query.having.length > 0) {
            sql += ` HAVING ${this.query.having.join(' AND ')}`;
        }
        
        // ORDER BY
        if (this.query.orderBy.length > 0) {
            sql += ' ORDER BY ';
            sql += this.query.orderBy.map(o => `${o.column} ${o.direction}`).join(', ');
        }
        
        // LIMIT
        if (this.query.limit !== null) {
            sql += ` LIMIT ${this.query.limit}`;
        }
        
        // OFFSET
        if (this.query.offset !== null) {
            sql += ` OFFSET ${this.query.offset}`;
        }
        
        return sql;
    }
}

// Usage
const query = new QueryBuilder()
    .select('u.id', 'u.name', 'p.title')
    .from('users u')
    .leftJoin('posts p', 'u.id = p.user_id')
    .where('u.active = 1')
    .where('u.created_at > "2023-01-01"')
    .orderBy('u.name')
    .limit(10)
    .build();

console.log(query);
// SELECT u.id, u.name, p.title FROM users u LEFT JOIN posts p ON u.id = p.user_id WHERE u.active = 1 AND u.created_at > "2023-01-01" ORDER BY u.name ASC LIMIT 10
```

### 3. Director Pattern with Builder
```javascript
// Product
class House {
    constructor() {
        this.foundation = null;
        this.walls = [];
        this.roof = null;
        this.windows = [];
        this.doors = [];
        this.garage = null;
        this.garden = null;
    }
    
    describe() {
        return {
            foundation: this.foundation,
            walls: this.walls.length,
            roof: this.roof,
            windows: this.windows.length,
            doors: this.doors.length,
            hasGarage: !!this.garage,
            hasGarden: !!this.garden
        };
    }
}

// Abstract Builder
class HouseBuilder {
    constructor() {
        this.house = new House();
    }
    
    buildFoundation() {
        throw new Error('buildFoundation must be implemented');
    }
    
    buildWalls() {
        throw new Error('buildWalls must be implemented');
    }
    
    buildRoof() {
        throw new Error('buildRoof must be implemented');
    }
    
    buildWindows() {
        throw new Error('buildWindows must be implemented');
    }
    
    buildDoors() {
        throw new Error('buildDoors must be implemented');
    }
    
    buildGarage() {
        // Optional - default implementation
        return this;
    }
    
    buildGarden() {
        // Optional - default implementation
        return this;
    }
    
    getHouse() {
        return this.house;
    }
}

// Concrete Builders
class ModernHouseBuilder extends HouseBuilder {
    buildFoundation() {
        this.house.foundation = 'Concrete slab foundation';
        return this;
    }
    
    buildWalls() {
        this.house.walls = [
            'Glass wall - North',
            'Concrete wall - South',
            'Glass wall - East',
            'Concrete wall - West'
        ];
        return this;
    }
    
    buildRoof() {
        this.house.roof = 'Flat roof with solar panels';
        return this;
    }
    
    buildWindows() {
        this.house.windows = [
            'Floor-to-ceiling window - Living room',
            'Skylight - Kitchen',
            'Large window - Master bedroom'
        ];
        return this;
    }
    
    buildDoors() {
        this.house.doors = [
            'Sliding glass door - Main entrance',
            'Wooden door - Bedroom'
        ];
        return this;
    }
    
    buildGarage() {
        this.house.garage = 'Underground garage for 2 cars';
        return this;
    }
    
    buildGarden() {
        this.house.garden = 'Minimalist zen garden';
        return this;
    }
}

class TraditionalHouseBuilder extends HouseBuilder {
    buildFoundation() {
        this.house.foundation = 'Stone foundation';
        return this;
    }
    
    buildWalls() {
        this.house.walls = [
            'Brick wall - North',
            'Brick wall - South',
            'Brick wall - East',
            'Brick wall - West'
        ];
        return this;
    }
    
    buildRoof() {
        this.house.roof = 'Gabled roof with clay tiles';
        return this;
    }
    
    buildWindows() {
        this.house.windows = [
            'Bay window - Living room',
            'Double-hung window - Kitchen',
            'Casement window - Bedroom 1',
            'Casement window - Bedroom 2'
        ];
        return this;
    }
    
    buildDoors() {
        this.house.doors = [
            'Wooden front door with glass panels',
            'Wooden door - Bedroom 1',
            'Wooden door - Bedroom 2',
            'French doors - Patio'
        ];
        return this;
    }
    
    buildGarage() {
        this.house.garage = 'Detached 2-car garage';
        return this;
    }
    
    buildGarden() {
        this.house.garden = 'English cottage garden';
        return this;
    }
}

// Director
class HouseDirector {
    constructor(builder) {
        this.builder = builder;
    }
    
    buildMinimalHouse() {
        return this.builder
            .buildFoundation()
            .buildWalls()
            .buildRoof()
            .buildWindows()
            .buildDoors()
            .getHouse();
    }
    
    buildFullHouse() {
        return this.builder
            .buildFoundation()
            .buildWalls()
            .buildRoof()
            .buildWindows()
            .buildDoors()
            .buildGarage()
            .buildGarden()
            .getHouse();
    }
}

// Usage
const modernBuilder = new ModernHouseBuilder();
const traditionalBuilder = new TraditionalHouseBuilder();

const modernDirector = new HouseDirector(modernBuilder);
const traditionalDirector = new HouseDirector(traditionalBuilder);

const modernHouse = modernDirector.buildFullHouse();
const traditionalHouse = traditionalDirector.buildMinimalHouse();

console.log('Modern House:', modernHouse.describe());
console.log('Traditional House:', traditionalHouse.describe());
```

## 🌟 Real-World Examples

### HTTP Request Builder
```javascript
class HttpRequest {
    constructor(builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = builder.headers;
        this.body = builder.body;
        this.timeout = builder.timeout;
        this.retries = builder.retries;
        this.cache = builder.cache;
    }
    
    async execute() {
        const options = {
            method: this.method,
            headers: this.headers,
            timeout: this.timeout
        };
        
        if (this.body) {
            options.body = JSON.stringify(this.body);
        }
        
        let attempt = 0;
        while (attempt <= this.retries) {
            try {
                const response = await fetch(this.url, options);
                
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }
                
                return await response.json();
            } catch (error) {
                attempt++;
                if (attempt > this.retries) {
                    throw error;
                }
                
                // Wait before retry
                await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
            }
        }
    }
}

class HttpRequestBuilder {
    constructor(url) {
        this.url = url;
        this.method = 'GET';
        this.headers = {
            'Content-Type': 'application/json'
        };
        this.timeout = 5000;
        this.retries = 0;
        this.cache = false;
    }
    
    setMethod(method) {
        this.method = method.toUpperCase();
        return this;
    }
    
    setHeader(key, value) {
        this.headers[key] = value;
        return this;
    }
    
    setHeaders(headers) {
        this.headers = { ...this.headers, ...headers };
        return this;
    }
    
    setBody(body) {
        this.body = body;
        return this;
    }
    
    setTimeout(timeout) {
        this.timeout = timeout;
        return this;
    }
    
    setRetries(retries) {
        this.retries = retries;
        return this;
    }
    
    enableCache() {
        this.cache = true;
        return this;
    }
    
    setAuth(token) {
        this.headers['Authorization'] = `Bearer ${token}`;
        return this;
    }
    
    build() {
        return new HttpRequest(this);
    }
}

// Usage
const request = new HttpRequestBuilder('https://api.example.com/users')
    .setMethod('POST')
    .setAuth('your-jwt-token')
    .setBody({
        name: 'John Doe',
        email: 'john@example.com'
    })
    .setTimeout(10000)
    .setRetries(3)
    .enableCache()
    .build();

// Execute the request
request.execute()
    .then(response => console.log('Success:', response))
    .catch(error => console.error('Error:', error));
```

### Form Builder
```javascript
class FormField {
    constructor(type, name, options = {}) {
        this.type = type;
        this.name = name;
        this.label = options.label || name;
        this.placeholder = options.placeholder || '';
        this.required = options.required || false;
        this.validation = options.validation || [];
        this.options = options.options || []; // For select, radio, checkbox
        this.value = options.value || '';
    }
    
    render() {
        let html = `<div class="form-field">`;
        html += `<label for="${this.name}">${this.label}${this.required ? ' *' : ''}</label>`;
        
        switch (this.type) {
            case 'text':
            case 'email':
            case 'password':
            case 'number':
                html += `<input type="${this.type}" id="${this.name}" name="${this.name}" 
                        placeholder="${this.placeholder}" value="${this.value}"
                        ${this.required ? 'required' : ''}>`;
                break;
                
            case 'textarea':
                html += `<textarea id="${this.name}" name="${this.name}" 
                        placeholder="${this.placeholder}" ${this.required ? 'required' : ''}>${this.value}</textarea>`;
                break;
                
            case 'select':
                html += `<select id="${this.name}" name="${this.name}" ${this.required ? 'required' : ''}>`;
                this.options.forEach(option => {
                    const selected = option.value === this.value ? 'selected' : '';
                    html += `<option value="${option.value}" ${selected}>${option.label}</option>`;
                });
                html += `</select>`;
                break;
                
            case 'radio':
                this.options.forEach(option => {
                    const checked = option.value === this.value ? 'checked' : '';
                    html += `<label><input type="radio" name="${this.name}" value="${option.value}" ${checked}> ${option.label}</label>`;
                });
                break;
        }
        
        html += `</div>`;
        return html;
    }
}

class Form {
    constructor(builder) {
        this.action = builder.action;
        this.method = builder.method;
        this.fields = builder.fields;
        this.submitText = builder.submitText;
        this.cssClass = builder.cssClass;
    }
    
    render() {
        let html = `<form action="${this.action}" method="${this.method}" class="${this.cssClass}">`;
        
        this.fields.forEach(field => {
            html += field.render();
        });
        
        html += `<button type="submit">${this.submitText}</button>`;
        html += `</form>`;
        
        return html;
    }
}

class FormBuilder {
    constructor() {
        this.action = '';
        this.method = 'POST';
        this.fields = [];
        this.submitText = 'Submit';
        this.cssClass = 'form';
    }
    
    setAction(action) {
        this.action = action;
        return this;
    }
    
    setMethod(method) {
        this.method = method;
        return this;
    }
    
    setCssClass(cssClass) {
        this.cssClass = cssClass;
        return this;
    }
    
    setSubmitText(text) {
        this.submitText = text;
        return this;
    }
    
    addTextField(name, options = {}) {
        this.fields.push(new FormField('text', name, options));
        return this;
    }
    
    addEmailField(name, options = {}) {
        this.fields.push(new FormField('email', name, options));
        return this;
    }
    
    addPasswordField(name, options = {}) {
        this.fields.push(new FormField('password', name, options));
        return this;
    }
    
    addTextArea(name, options = {}) {
        this.fields.push(new FormField('textarea', name, options));
        return this;
    }
    
    addSelectField(name, options, fieldOptions = {}) {
        fieldOptions.options = options;
        this.fields.push(new FormField('select', name, fieldOptions));
        return this;
    }
    
    addRadioGroup(name, options, fieldOptions = {}) {
        fieldOptions.options = options;
        this.fields.push(new FormField('radio', name, fieldOptions));
        return this;
    }
    
    build() {
        return new Form(this);
    }
}

// Usage
const contactForm = new FormBuilder()
    .setAction('/contact')
    .setMethod('POST')
    .setCssClass('contact-form')
    .setSubmitText('Send Message')
    .addTextField('name', {
        label: 'Full Name',
        placeholder: 'Enter your full name',
        required: true
    })
    .addEmailField('email', {
        label: 'Email Address',
        placeholder: 'Enter your email',
        required: true
    })
    .addSelectField('subject', [
        { value: 'general', label: 'General Inquiry' },
        { value: 'support', label: 'Support' },
        { value: 'sales', label: 'Sales' }
    ], {
        label: 'Subject',
        required: true
    })
    .addTextArea('message', {
        label: 'Message',
        placeholder: 'Enter your message',
        required: true
    })
    .build();

console.log(contactForm.render());
```

## 🎯 Interview Questions

### Q1: When would you use Builder pattern over Constructor?
**Answer:**
- When object has many optional parameters (telescoping constructor problem)
- When you need to enforce construction steps
- When creating different representations of the same object
- When construction process is complex

### Q2: What's the difference between Builder and Factory patterns?
**Answer:**
- **Builder**: Focuses on step-by-step construction of complex objects
- **Factory**: Focuses on creating objects without specifying exact class
- **Builder** is about "how to build", **Factory** is about "what to build"

### Q3: How does Builder pattern solve the telescoping constructor problem?
**Answer:**
```javascript
// Telescoping Constructor Problem
class User {
    constructor(name, email, phone, address, age, isActive, roles, preferences) {
        // Too many parameters, hard to remember order
    }
}

// Builder Solution
const user = new UserBuilder('John', 'Doe')
    .setEmail('john@example.com')
    .setPhone('123-456-7890')
    .setAge(30)
    .build();
```

### Q4: Can you implement a Builder without a separate Builder class?
**Answer:**
```javascript
class User {
    constructor() {
        this.firstName = '';
        this.lastName = '';
        this.email = '';
    }
    
    setFirstName(firstName) {
        this.firstName = firstName;
        return this;
    }
    
    setLastName(lastName) {
        this.lastName = lastName;
        return this;
    }
    
    setEmail(email) {
        this.email = email;
        return this;
    }
}

// Usage
const user = new User()
    .setFirstName('John')
    .setLastName('Doe')
    .setEmail('john@example.com');
```

## 💡 Best Practices

1. **Use for complex objects** - Don't over-engineer simple objects
2. **Make it fluent** - Return `this` for method chaining
3. **Validate at build time** - Check required fields in build() method
4. **Consider immutability** - Make built objects immutable
5. **Provide sensible defaults** - Set reasonable default values
6. **Use Director when needed** - For common construction sequences

## 🔗 Related Patterns
- **Factory Pattern** - Can use builders to create objects
- **Composite Pattern** - Builders often create composite structures
- **Strategy Pattern** - Different building strategies
- **Template Method** - Define construction algorithm skeleton
