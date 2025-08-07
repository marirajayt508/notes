# ThoughtWorks JOI Energy Interview Preparation Notes

## Table of Contents
1. [Fix Missing Meter-to-Plan Mappings](#1-fix-missing-meter-to-plan-mappings)
2. [Add Validation and Error Handling](#2-add-validation-and-error-handling)
3. [Get Consumption for Last 24 Hours](#3-get-consumption-for-last-24-hours)
4. [Implement Usage Alerts](#4-implement-usage-alerts)
5. [Switch Price Plans](#5-switch-price-plans)
6. [Export Usage Data as CSV](#6-export-usage-data-as-csv)
7. [Add Authentication](#7-add-authentication)
8. [Historical Data Endpoint](#8-historical-data-endpoint)
9. [Add Pagination and Filtering](#9-add-pagination-and-filtering)

---

## 1. Fix Missing Meter-to-Plan Mappings

### Problem
Only 3 out of 5 meters are mapped to price plans.

### Implementation

```javascript
// src/meters/meters.js
const meterPricePlanMap = {
    [meters.METER0]: pricePlans[pricePlanNames.PRICEPLAN0], // Dr Evil's
    [meters.METER1]: pricePlans[pricePlanNames.PRICEPLAN1], // Power for Everyone
    [meters.METER2]: pricePlans[pricePlanNames.PRICEPLAN0], // Dr Evil's
    [meters.METER3]: pricePlans[pricePlanNames.PRICEPLAN1], // Power for Everyone  
    [meters.METER4]: pricePlans[pricePlanNames.PRICEPLAN2], // The Green Eco
};
```

### Test

```javascript
// src/meters/meters.test.js
describe('meterPricePlanMap', () => {
    it('should have all meters mapped to price plans', () => {
        const { meters, meterPricePlanMap } = require('./meters');
        
        Object.values(meters).forEach(meter => {
            expect(meterPricePlanMap[meter]).toBeDefined();
            expect(meterPricePlanMap[meter]).toHaveProperty('supplier');
            expect(meterPricePlanMap[meter]).toHaveProperty('rate');
        });
    });
});
```

---

## 2. Add Validation and Error Handling

### Implementation

```javascript
// src/middleware/validation.js
const validateSmartMeterId = (req, res, next) => {
    const { smartMeterId } = req.params;
    
    if (!smartMeterId) {
        return res.status(400).json({ 
            error: 'Smart meter ID is required' 
        });
    }
    
    // Check if meter exists in our system
    const { meters } = require('../meters/meters');
    const validMeterIds = Object.values(meters);
    
    if (!validMeterIds.includes(smartMeterId)) {
        return res.status(404).json({ 
            error: `Smart meter ${smartMeterId} not found` 
        });
    }
    
    next();
};

const validateReadings = (req, res, next) => {
    const { smartMeterId, electricityReadings } = req.body;
    
    if (!smartMeterId || !electricityReadings) {
        return res.status(400).json({ 
            error: 'smartMeterId and electricityReadings are required' 
        });
    }
    
    if (!Array.isArray(electricityReadings) || electricityReadings.length === 0) {
        return res.status(400).json({ 
            error: 'electricityReadings must be a non-empty array' 
        });
    }
    
    // Validate each reading
    const invalidReading = electricityReadings.find(reading => 
        !reading.time || 
        typeof reading.time !== 'number' ||
        reading.reading === undefined ||
        typeof reading.reading !== 'number' ||
        reading.reading < 0
    );
    
    if (invalidReading) {
        return res.status(400).json({ 
            error: 'Each reading must have valid time and reading values' 
        });
    }
    
    next();
};

module.exports = { validateSmartMeterId, validateReadings };

// Update app.js to use middleware
const { validateSmartMeterId, validateReadings } = require('./middleware/validation');

app.get("/readings/read/:smartMeterId", validateSmartMeterId, (req, res) => {
    res.send(read(getReadings, req));
});

app.post("/readings/store", validateReadings, (req, res) => {
    res.send(store(setReadings, req));
});
```

### Test

```javascript
// src/middleware/validation.test.js
describe('Validation Middleware', () => {
    describe('validateSmartMeterId', () => {
        it('should return 400 if smartMeterId is missing', () => {
            const req = { params: {} };
            const res = {
                status: jest.fn().mockReturnThis(),
                json: jest.fn()
            };
            const next = jest.fn();
            
            validateSmartMeterId(req, res, next);
            
            expect(res.status).toHaveBeenCalledWith(400);
            expect(res.json).toHaveBeenCalledWith({ 
                error: 'Smart meter ID is required' 
            });
            expect(next).not.toHaveBeenCalled();
        });
        
        it('should call next if smartMeterId is valid', () => {
            const req = { params: { smartMeterId: 'smart-meter-0' } };
            const res = {};
            const next = jest.fn();
            
            validateSmartMeterId(req, res, next);
            
            expect(next).toHaveBeenCalled();
        });
    });
});
```

---

## 3. Get Consumption for Last 24 Hours

### Implementation

```javascript
// src/usage/usage-controller.js
const getLast24HoursUsage = (getReadings, req) => {
    const { smartMeterId } = req.params;
    const readings = getReadings(smartMeterId);
    
    if (!readings || readings.length === 0) {
        return { 
            error: 'No readings found for this meter',
            status: 404 
        };
    }
    
    const now = Date.now() / 1000; // Current time in seconds
    const twentyFourHoursAgo = now - (24 * 60 * 60);
    
    const last24HoursReadings = readings.filter(
        reading => reading.time >= twentyFourHoursAgo
    );
    
    if (last24HoursReadings.length === 0) {
        return {
            smartMeterId,
            period: '24hours',
            consumption: 0,
            message: 'No readings in the last 24 hours'
        };
    }
    
    const consumption = usage(last24HoursReadings);
    const totalKwh = consumption * 24; // Convert to kWh for 24 hour period
    
    return {
        smartMeterId,
        period: '24hours',
        consumption: totalKwh,
        numberOfReadings: last24HoursReadings.length,
        startTime: new Date(twentyFourHoursAgo * 1000).toISOString(),
        endTime: new Date(now * 1000).toISOString()
    };
};

// Add to app.js
app.get("/usage/last-24-hours/:smartMeterId", validateSmartMeterId, (req, res) => {
    const result = getLast24HoursUsage(getReadings, req);
    
    if (result.error) {
        return res.status(result.status || 500).json({ error: result.error });
    }
    
    res.json(result);
});
```

### Test

```javascript
// src/usage/usage-controller.test.js
describe('getLast24HoursUsage', () => {
    it('should return consumption for last 24 hours', () => {
        const now = Date.now() / 1000;
        const mockReadings = [
            { time: now - 3600, reading: 0.5 },      // 1 hour ago
            { time: now - 7200, reading: 0.6 },      // 2 hours ago
            { time: now - 86400, reading: 0.4 },     // 24 hours ago
            { time: now - 172800, reading: 0.3 }     // 48 hours ago (should be excluded)
        ];
        
        const getReadings = jest.fn().mockReturnValue(mockReadings);
        const req = { params: { smartMeterId: 'smart-meter-0' } };
        
        const result = getLast24HoursUsage(getReadings, req);
        
        expect(result).toHaveProperty('consumption');
        expect(result.numberOfReadings).toBe(3); // Only 3 readings in last 24 hours
    });
});
```

---

## 4. Implement Usage Alerts

### Implementation

```javascript
// src/alerts/alerts.js
const alertsConfig = {};
const alertsHistory = {};

const configureAlert = (meterId, config) => {
    alertsConfig[meterId] = {
        threshold: config.threshold,
        email: config.email,
        enabled: config.enabled !== false
    };
    
    if (!alertsHistory[meterId]) {
        alertsHistory[meterId] = [];
    }
    
    return alertsConfig[meterId];
};

const checkUsageAlert = (meterId, currentUsage) => {
    const config = alertsConfig[meterId];
    
    if (!config || !config.enabled) {
        return null;
    }
    
    if (currentUsage > config.threshold) {
        const alert = {
            type: 'threshold_exceeded',
            meterId,
            threshold: config.threshold,
            actualUsage: currentUsage,
            timestamp: new Date().toISOString(),
            message: `Usage ${currentUsage.toFixed(2)} kWh exceeds threshold ${config.threshold} kWh`
        };
        
        // Add to history
        alertsHistory[meterId].push(alert);
        
        // In real implementation, send email here
        console.log(`[ALERT] Sending email to ${config.email}: ${alert.message}`);
        
        return alert;
    }
    
    return null;
};

// src/alerts/alerts-controller.js
const configureAlertEndpoint = (req, res) => {
    const { smartMeterId, threshold, email } = req.body;
    
    if (!smartMeterId || !threshold || !email) {
        return res.status(400).json({ 
            error: 'smartMeterId, threshold, and email are required' 
        });
    }
    
    if (threshold <= 0) {
        return res.status(400).json({ 
            error: 'Threshold must be positive' 
        });
    }
    
    const config = configureAlert(smartMeterId, { threshold, email });
    res.json({ message: 'Alert configured successfully', config });
};

// Add to app.js
app.post("/alerts/configure", (req, res) => {
    configureAlertEndpoint(req, res);
});

// Modify the store readings to check alerts
app.post("/readings/store", validateReadings, (req, res) => {
    const result = store(setReadings, req);
    
    // Check if we should trigger an alert
    const dailyUsage = getLast24HoursUsage(getReadings, req);
    if (dailyUsage.consumption) {
        const alert = checkUsageAlert(req.body.smartMeterId, dailyUsage.consumption);
        if (alert) {
            result.alert = alert;
        }
    }
    
    res.send(result);
});
```

### Test

```javascript
// src/alerts/alerts.test.js
describe('Usage Alerts', () => {
    beforeEach(() => {
        // Clear alerts between tests
        Object.keys(alertsConfig).forEach(key => delete alertsConfig[key]);
    });
    
    describe('configureAlert', () => {
        it('should configure alert for a meter', () => {
            const config = configureAlert('smart-meter-0', {
                threshold: 50,
                email: 'test@example.com'
            });
            
            expect(config).toEqual({
                threshold: 50,
                email: 'test@example.com',
                enabled: true
            });
        });
    });
    
    describe('checkUsageAlert', () => {
        it('should trigger alert when usage exceeds threshold', () => {
            configureAlert('smart-meter-0', {
                threshold: 50,
                email: 'test@example.com'
            });
            
            const alert = checkUsageAlert('smart-meter-0', 75);
            
            expect(alert).toBeTruthy();
            expect(alert.type).toBe('threshold_exceeded');
            expect(alert.actualUsage).toBe(75);
        });
        
        it('should not trigger alert when usage is below threshold', () => {
            configureAlert('smart-meter-0', {
                threshold: 50,
                email: 'test@example.com'
            });
            
            const alert = checkUsageAlert('smart-meter-0', 30);
            
            expect(alert).toBeNull();
        });
    });
});
```

---

## 5. Switch Price Plans

### Implementation

```javascript
// src/price-plans/price-plans-controller.js
const switchPricePlan = (req, res) => {
    const { smartMeterId } = req.params;
    const { pricePlanId } = req.body;
    
    if (!pricePlanId) {
        return res.status(400).json({ 
            error: 'pricePlanId is required' 
        });
    }
    
    // Validate price plan exists
    if (!pricePlans[pricePlanId]) {
        return res.status(404).json({ 
            error: `Price plan ${pricePlanId} not found` 
        });
    }
    
    // Update meter mapping
    meterPricePlanMap[smartMeterId] = pricePlans[pricePlanId];
    
    res.json({
        message: 'Price plan switched successfully',
        smartMeterId,
        newPricePlan: {
            id: pricePlanId,
            ...pricePlans[pricePlanId]
        }
    });
};

// Add to app.js
app.put("/price-plans/switch/:smartMeterId", validateSmartMeterId, (req, res) => {
    switchPricePlan(req, res);
});
```

### Test

```javascript
// src/price-plans/price-plans-controller.test.js
describe('switchPricePlan', () => {
    it('should switch meter to new price plan', () => {
        const req = {
            params: { smartMeterId: 'smart-meter-0' },
            body: { pricePlanId: 'price-plan-2' }
        };
        const res = {
            json: jest.fn(),
            status: jest.fn().mockReturnThis()
        };
        
        switchPricePlan(req, res);
        
        expect(res.json).toHaveBeenCalledWith({
            message: 'Price plan switched successfully',
            smartMeterId: 'smart-meter-0',
            newPricePlan: expect.objectContaining({
                id: 'price-plan-2',
                supplier: 'The Green Eco',
                rate: 1
            })
        });
    });
});
```

---

## 6. Export Usage Data as CSV

### Implementation

```javascript
// src/export/export-controller.js
const exportToCSV = (getReadings, req, res) => {
    const { smartMeterId } = req.params;
    const readings = getReadings(smartMeterId);
    
    if (!readings || readings.length === 0) {
        return res.status(404).json({ 
            error: 'No readings found for this meter' 
        });
    }
    
    // Create CSV header
    let csv = 'Timestamp,Time,Reading (kW)\n';
    
    // Add data rows
    readings.forEach(reading => {
        const date = new Date(reading.time * 1000);
        csv += `${reading.time},${date.toISOString()},${reading.reading}\n`;
    });
    
    // Set headers for file download
    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', `attachment; filename="${smartMeterId}-usage.csv"`);
    
    res.send(csv);
};

// Add usage summary export
const exportUsageSummary = (getReadings, req, res) => {
    const { smartMeterId } = req.params;
    const readings = getReadings(smartMeterId);
    
    if (!readings || readings.length === 0) {
        return res.status(404).json({ 
            error: 'No readings found for this meter' 
        });
    }
    
    // Group by day and calculate daily usage
    const dailyUsage = {};
    
    readings.forEach(reading => {
        const date = new Date(reading.time * 1000).toISOString().split('T')[0];
        if (!dailyUsage[date]) {
            dailyUsage[date] = [];
        }
        dailyUsage[date].push(reading);
    });
    
    // Create CSV
    let csv = 'Date,Average Reading (kW),Number of Readings,Estimated Daily Usage (kWh)\n';
    
    Object.entries(dailyUsage).forEach(([date, dayReadings]) => {
        const avgReading = average(dayReadings);
        const estimatedUsage = avgReading * 24; // Rough estimate
        csv += `${date},${avgReading.toFixed(3)},${dayReadings.length},${estimatedUsage.toFixed(2)}\n`;
    });
    
    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', `attachment; filename="${smartMeterId}-summary.csv"`);
    
    res.send(csv);
};

// Add to app.js
app.get("/export/readings/:smartMeterId", validateSmartMeterId, (req, res) => {
    exportToCSV(getReadings, req, res);
});

app.get("/export/summary/:smartMeterId", validateSmartMeterId, (req, res) => {
    exportUsageSummary(getReadings, req, res);
});
```

### Test

```javascript
// src/export/export-controller.test.js
describe('Export Controller', () => {
    describe('exportToCSV', () => {
        it('should export readings as CSV', () => {
            const mockReadings = [
                { time: 1607686125, reading: 0.5 },
                { time: 1607682525, reading: 0.6 }
            ];
            
            const getReadings = jest.fn().mockReturnValue(mockReadings);
            const req = { params: { smartMeterId: 'smart-meter-0' } };
            const res = {
                setHeader: jest.fn(),
                send: jest.fn(),
                status: jest.fn().mockReturnThis(),
                json: jest.fn()
            };
            
            exportToCSV(getReadings, req, res);
            
            expect(res.setHeader).toHaveBeenCalledWith('Content-Type', 'text/csv');
            expect(res.send).toHaveBeenCalled();
            
            const csvContent = res.send.mock.calls[0][0];
            expect(csvContent).toContain('Timestamp,Time,Reading (kW)');
            expect(csvContent).toContain('1607686125');
            expect(csvContent).toContain('0.5');
        });
    });
});
```

---

## 7. Add Authentication

### Implementation

```javascript
// src/auth/auth.js
const API_KEYS = {
    'api-key-sarah': { user: 'sarah', meterId: 'smart-meter-0' },
    'api-key-peter': { user: 'peter', meterId: 'smart-meter-1' },
    'api-key-charlie': { user: 'charlie', meterId: 'smart-meter-2' },
    'api-key-andrea': { user: 'andrea', meterId: 'smart-meter-3' },
    'api-key-alex': { user: 'alex', meterId: 'smart-meter-4' }
};

const authenticateApiKey = (req, res, next) => {
    const apiKey = req.headers['x-api-key'];
    
    if (!apiKey) {
        return res.status(401).json({ 
            error: 'API key required. Please provide X-API-Key header' 
        });
    }
    
    const user = API_KEYS[apiKey];
    
    if (!user) {
        return res.status(401).json({ 
            error: 'Invalid API key' 
        });
    }
    
    // Attach user info to request
    req.user = user;
    next();
};

// Middleware to ensure users can only access their own data
const authorizeUserAccess = (req, res, next) => {
    const requestedMeterId = req.params.smartMeterId;
    const userMeterId = req.user.meterId;
    
    // Admin users could access all meters (future enhancement)
    if (requestedMeterId && requestedMeterId !== userMeterId) {
        return res.status(403).json({ 
            error: 'Access denied. You can only access your own meter data' 
        });
    }
    
    next();
};

// Update app.js to use authentication
// Apply to all routes except health check
app.get('/health', (req, res) => {
    res.json({ status: 'OK' });
});

// Protected routes
app.use(authenticateApiKey); // Apply to all routes below

app.get("/readings/read/:smartMeterId", 
    validateSmartMeterId, 
    authorizeUserAccess, 
    (req, res) => {
        res.send(read(getReadings, req));
    }
);

// For store, automatically use the user's meter
app.post("/readings/store", validateReadings, (req, res) => {
    // Override smartMeterId with authenticated user's meter
    req.body.smartMeterId = req.user.meterId;
    res.send(store(setReadings, req));
});
```

### Test

```javascript
// src/auth/auth.test.js
describe('Authentication', () => {
    describe('authenticateApiKey', () => {
        it('should reject request without API key', () => {
            const req = { headers: {} };
            const res = {
                status: jest.fn().mockReturnThis(),
                json: jest.fn()
            };
            const next = jest.fn();
            
            authenticateApiKey(req, res, next);
            
            expect(res.status).toHaveBeenCalledWith(401);
            expect(next).not.toHaveBeenCalled();
        });
        
        it('should authenticate valid API key', () => {
            const req = { headers: { 'x-api-key': 'api-key-sarah' } };
            const res = {};
            const next = jest.fn();
            
            authenticateApiKey(req, res, next);
            
            expect(req.user).toEqual({
                user: 'sarah',
                meterId: 'smart-meter-0'
            });
            expect(next).toHaveBeenCalled();
        });
    });
});
```

---

## 8. Historical Data Endpoint

### Implementation

```javascript
// src/readings/readings-controller.js
const getHistoricalData = (getReadings, req, res) => {
    const { smartMeterId } = req.params;
    const { startDate, endDate, granularity = 'hourly' } = req.query;
    
    const readings = getReadings(smartMeterId);
    
    if (!readings || readings.length === 0) {
        return res.status(404).json({ 
            error: 'No readings found for this meter' 
        });
    }
    
    let filteredReadings = readings;
    
    // Apply date filters if provided
    if (startDate || endDate) {
        const start = startDate ? new Date(startDate).getTime() / 1000 : 0;
        const end = endDate ? new Date(endDate).getTime() / 1000 : Date.now() / 1000;
        
        filteredReadings = readings.filter(r => r.time >= start && r.time <= end);
    }
    
    // Group by granularity
    const groupedData = groupByGranularity(filteredReadings, granularity);
    
    res.json({
        smartMeterId,
        startDate: startDate || 'all-time',
        endDate: endDate || 'now',
        granularity,
        dataPoints: groupedData.length,
        data: groupedData
    });
};

const groupByGranularity = (readings, granularity) => {
    const grouped = {};
    
    readings.forEach(reading => {
        const date = new Date(reading.time * 1000);
        let key;
        
        switch (granularity) {
            case 'hourly':
                key = `${date.toISOString().slice(0, 13)}:00:00`;
                break;
            case 'daily':
                key = date.toISOString().slice(0, 10);
                break;
            case 'weekly':
                const week = getWeekNumber(date);
                key = `${date.getFullYear()}-W${week}`;
                break;
            default:
                key = date.toISOString();
        }
        
        if (!grouped[key]) {
            grouped[key] = {
                period: key,
                readings: [],
                averageReading: 0,
                totalReadings: 0
            };
        }
        
        grouped[key].readings.push(reading.reading);
        grouped[key].totalReadings++;
    });
    
    // Calculate averages
    return Object.values(grouped).map(group => {
        group.averageReading = 
            group.readings.reduce((sum, r) => sum + r, 0) / group.readings.length;
        delete group.readings; // Remove raw data to keep response clean
        return group;
    });
};

// Helper function to get week number
const getWeekNumber = (date) => {
    const d = new Date(Date.UTC(date.getFullYear(), date.getMonth(), date.getDate()));
    const dayNum = d.getUTCDay() || 7;
    d.setUTCDate(d.getUTCDate() + 4 - dayNum);
    const yearStart = new Date(Date.UTC(d.getUTCFullYear(),0,1));
    return Math.ceil((((d - yearStart) / 86400000) + 1)/7);
};

// Add to app.js
app.get("/readings/historical/:smartMeterId", 
    validateSmartMeterId,
    authorizeUserAccess,
    (req, res) => {
        getHistoricalData(getReadings, req, res);
    }
);
```

### Test

```javascript
// src/readings/readings-controller.test.js
describe('getHistoricalData', () => {
    it('should return grouped historical data', () => {
        const now = Date.now() / 1000;
        const mockReadings = [
            { time: now, reading: 0.5 },
            { time: now - 3600, reading: 0.6 }, // 1 hour ago
            { time: now - 7200, reading: 0.7 }, // 2 hours ago
        ];
        
        const getReadings = jest.fn().mockReturnValue(mockReadings);
        const req = {
            params: { smartMeterId: 'smart-meter-0' },
            query: { granularity: 'hourly' }
        };
        const res = {
            json: jest.fn(),
            status: jest.fn().mockReturnThis()
        };
        
        getHistoricalData(getReadings, req, res);
        
        expect(res.json).toHaveBeenCalled();
        const response = res.json.mock.calls[0][0];
        expect(response).toHaveProperty('data');
        expect(response.granularity).toBe('hourly');
    });
});
```

---

## 9. Add Pagination and Filtering

### Implementation

```javascript
// src/utils/pagination.js
const paginate = (data, page = 1, limit = 20) => {
    const startIndex = (page - 1) * limit;
    const endIndex = startIndex + limit;
    
    const paginatedData = data.slice(startIndex, endIndex);
    
    return {
        data: paginatedData,
        pagination: {
            page: parseInt(page),
            limit: parseInt(limit),
            total: data.length,
            totalPages: Math.ceil(data.length / limit),
            hasNext: endIndex < data.length,
            hasPrev: page > 1
        }
    };
};

// Update readings controller
const readWithPagination = (getData, req) => {
    const { smartMeterId } = req.params;
    const { page = 1, limit = 20, sortBy = 'time', order = 'desc' } = req.query;
    
    let readings = getData(smartMeterId);
    
    if (!readings || readings.length === 0) {
        return {
            data: [],
            pagination: {
                page: 1,
                limit: parseInt(limit),
                total: 0,
                totalPages: 0,
                hasNext: false,
                hasPrev: false
            }
        };
    }
    
    // Sort readings
    readings = [...readings].sort((a, b) => {
        if (order === 'asc') {
            return a[sortBy] - b[sortBy];
        }
        return b[sortBy] - a[sortBy];
    });
    
    return paginate(readings, page, limit);
};

// Add filtering
const readWithFilters = (getData, req) => {
    const { smartMeterId } = req.params;
    const {
        minReading,
        maxReading,
        startTime,
        endTime,
        page = 1,
        limit = 20
    } = req.query;
    
    let readings = getData(smartMeterId) || [];
    
    // Apply filters
    if (minReading !== undefined) {
        readings = readings.filter(r => r.reading >= parseFloat(minReading));
    }
    
    if (maxReading !== undefined) {
        readings = readings.filter(r => r.reading <= parseFloat(maxReading));
    }
