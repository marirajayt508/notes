## 📋 Real Questions from ThoughtWorks Code Pairing Interviews

### __Round 1: Understanding the Codebase (First 30 mins)__

#### Questions Candidates Reported:

1. __"Can you walk me through what happens when a smart meter sends readings to the API?"__

   - Trace the complete flow from endpoint to response

2. __"What do you think about the current code structure?"__

   - Don't be overly critical
   - Mention both positives and areas for improvement

3. __"What bugs or issues do you see in the existing code?"__

   - Missing meter mappings (only 3 of 5)
   - No error handling
   - Division by zero risk in usage calculation
   - No input validation

4. __"Explain the difference between kW and kWh in this context"__

   - kW = instantaneous power (like speed)
   - kWh = energy over time (like distance)

5. __"How does the cost calculation work?"__

   - They want to see if you understand the usage.js logic

6. __"What would happen if we send invalid data?"__

   - Show where the code would break

### __Round 2: Live Coding Tasks (45-60 mins)__

#### Most Common Tasks Given:

__1. "Add validation to the store readings endpoint"__

```javascript
// They want to see:
- Validate meter ID exists
- Check readings array is not empty
- Validate timestamp and reading values
- Return proper HTTP status codes
```

__2. "Implement an endpoint to get usage for the last N hours"__

```javascript
// GET /usage/last/:hours/:smartMeterId
// They look for:
- Proper date filtering
- Edge case handling
- Clear variable names
```

__3. "Fix the bug where meters 3 and 4 don't have price plans"__

```javascript
// Simple fix but they want to see:
- You identify the issue quickly
- You add a test for it
- You think about why it happened
```

__4. "Add an alert system when usage exceeds a threshold"__

```javascript
// Common follow-up: "How would you store alert preferences?"
// They want to see your design thinking
```

__5. "Implement CSV export for readings"__

```javascript
// They check if you:
- Set proper headers
- Format dates correctly
- Handle empty data
```

### __Round 3: Extension Questions During Coding__

#### Questions They Ask While You Code:

1. __"What if two requests come at the same time?"__

   - Shows if you think about concurrency

2. __"How would you test this?"__

   - They want to see TDD approach

3. __"What would you do differently in production?"__

   - Database instead of in-memory
   - Add logging and monitoring
   - Authentication/authorization

4. __"Can you make this more efficient?"__

   - Shows optimization thinking

5. __"What if we have millions of readings?"__

   - Pagination, caching, database indexing

### __Specific Scenarios Candidates Faced:__

#### __Scenario 1: The Interviewer Breaks Your Code__

```javascript
// You implement validation, then they ask:
"What if I send readings = null instead of undefined?"
// They want to see how you handle edge cases
```

#### __Scenario 2: Requirements Change Mid-Task__

```javascript
// Initial: "Export last 24 hours of data"
// Midway: "Actually, make it configurable - last N hours"
// They test adaptability
```

#### __Scenario 3: Performance Questions__

```javascript
"This endpoint is slow with 10,000 readings. How would you fix it?"
// Expected answers:
- Caching
- Database with indexes
- Pagination
- Background processing
```

### __Behavioral Questions During Coding:__

1. __"Tell me about a time you had to refactor legacy code"__
2. __"How do you ensure code quality in your team?"__
3. __"What's your approach to code reviews?"__
4. __"How do you handle technical debt?"__

### __Common Traps to Avoid:__

1. __They show you this code:__

```javascript
const average = readings.reduce((sum, r) => sum + r.reading, 0) / readings.length;
```

__They ask:__ "What's wrong here?" __Answer:__ Division by zero if readings is empty

2. __They ask:__ "Should we use MongoDB or PostgreSQL for this?" __Trap:__ There's no right answer - they want to hear your reasoning

3. __They say:__ "The existing code is pretty bad, right?" __Correct response:__ "It works but could be improved by adding..."

### __What Impresses Interviewers:__

✅ __Writing tests FIRST (TDD)__

```javascript
// Write test
it('should return 400 for invalid meter ID', () => {...})
// Then implement
```

✅ __Clear communication__

```javascript
"I'm going to validate the input first because..."
"Let me check if this edge case is handled..."
```

✅ __Using git properly__

```bash
git add .
git commit -m "Add validation for meter readings endpoint"
# Not just one big commit at the end
```

✅ __Asking good questions__

- "Should we validate that timestamps are not in the future?"
- "What's the expected behavior for duplicate readings?"
- "Is there a max limit on readings array size?"

### __Final Round Questions:__

1. __"How would you deploy this to production?"__

   - CI/CD pipeline
   - Environment variables
   - Health checks
   - Monitoring

2. __"Design a dashboard for this API"__

   - They want to see system design thinking

3. __"How would you scale this to 1 million users?"__

   - Load balancing
   - Caching strategy
   - Database sharding
