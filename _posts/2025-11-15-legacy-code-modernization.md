---
layout: default
title:  "Legacy Code Modernization Strategies"
date:   2025-11-15 03:00:00
categories: Development Architecture Modernization
---

I've modernized systems written in COBOL, Classic ASP, PHP 4, and Angular.js 1.x. It's never fun, but it's often necessary. Here's what I've learned about doing it without burning everything down.

## The Golden Rule

**Don't rewrite from scratch.**

Every developer who sees legacy code thinks "I could rewrite this in a weekend." They're wrong. The legacy system contains years of business logic, edge cases, and bug fixes that aren't documented anywhere except in the code.

**Strangler Fig pattern > Big Bang rewrite**

## Step 1: Understand the System

### Map Dependencies

```bash
# Find all imports/requires
grep -r "require\|import" src/

# Find all API endpoints
grep -r "app.get\|app.post\|router" src/

# Find all database queries
grep -r "SELECT\|INSERT\|UPDATE\|DELETE" src/
```

### Document the Architecture

Even basic documentation helps:

```markdown
## System Overview

**Frontend:** Classic ASP pages
**Backend:** VBScript + COM objects
**Database:** SQL Server 2008
**Authentication:** Windows Auth

**Key Flows:**
1. User login → check AD → create session
2. Order creation → validate → write to DB → send email
3. Report generation → query DB → generate PDF
```

### Identify Pain Points

- What breaks most often?
- What takes longest to change?
- What has the most bugs?
- What is blocking new features?

Start modernizing there.

## Step 2: Add Tests

Before changing anything, add tests. This is non-negotiable.

### Start with Integration Tests

```javascript
describe('Order API', () => {
    it('creates an order', async () => {
        const response = await request(app)
            .post('/api/orders')
            .send({
                customerId: 123,
                items: [{ productId: 1, quantity: 2 }]
            });

        expect(response.status).toBe(201);
        expect(response.body.orderId).toBeDefined();
    });

    it('validates required fields', async () => {
        const response = await request(app)
            .post('/api/orders')
            .send({});

        expect(response.status).toBe(400);
    });
});
```

### Approval Tests for Complex Logic

When you can't easily write assertions:

```javascript
const approvalTests = require('approvals');

it('generates correct report', () => {
    const report = generateReport(testData);
    approvalTests.verify(report);
    // First run: saves output as "approved"
    // Future runs: compares to approved version
});
```

## Step 3: Strangler Fig Pattern

### The Concept

Wrap the old system. Route requests through the wrapper. Gradually replace pieces.

```
Before:
Request → Old System

During:
Request → Router → Old System
                 → New Module 1
                 → New Module 2

After:
Request → New System
```

### Implementation

```javascript
// router.js - Routes requests to old or new system
app.post('/api/orders', async (req, res) => {
    if (featureFlags.isEnabled('new-order-service')) {
        // New implementation
        return newOrderService.create(req, res);
    }
    // Forward to legacy system
    return proxy.forward(req, res, 'http://legacy-system/api/orders');
});
```

### Migrate One Endpoint at a Time

```javascript
// Week 1: Migrate /api/users
// Week 2: Migrate /api/orders
// Week 3: Migrate /api/products
// ...
```

## Step 4: Decouple the Database

### The Problem

Legacy systems often have:
- Stored procedures with business logic
- Triggers modifying data
- Direct table access from multiple places
- No clear data model

### The Solution

#### Step 4a: Read from New, Write to Both

```javascript
async function getUser(id) {
    // Read from new database
    return newDb.query('SELECT * FROM users WHERE id = ?', [id]);
}

async function createUser(data) {
    // Write to both
    await newDb.query('INSERT INTO users ...', [data]);
    await legacyDb.query('INSERT INTO users ...', [data]);  // Keep in sync
}
```

#### Step 4b: Shadow Reads

Compare old and new results:

```javascript
async function getUser(id) {
    const [oldResult, newResult] = await Promise.all([
        legacyDb.query('...'),
        newDb.query('...')
    ]);

    // Compare and log differences
    if (!deepEqual(oldResult, newResult)) {
        logger.warn('Data mismatch', { old: oldResult, new: newResult });
    }

    return oldResult;  // Still use old system
}
```

#### Step 4c: Cut Over

Once confident, switch to new database.

## Step 5: Modernize Incrementally

### UI First (If Appropriate)

Build new UI that calls existing APIs:

```
Old UI → Old API → Old Database

New UI → Old API → Old Database
```

Later replace the API:

```
New UI → New API → New Database
```

### API First

Build new API that calls existing database:

```
Old UI → New API → Old Database
```

Then modernize database:

```
New UI → New API → New Database
```

## Common Patterns

### Anti-Corruption Layer

Translate between old and new systems:

```javascript
// Translate legacy response to modern format
function transformLegacyUser(legacyUser) {
    return {
        id: legacyUser.USER_ID,
        email: legacyUser.EMAIL_ADDR,
        name: `${legacyUser.FIRST_NM} ${legacyUser.LAST_NM}`,
        createdAt: new Date(legacyUser.CREATE_DT)
    };
}
```

### Branch by Abstraction

Hide implementation behind interface:

```javascript
// Interface
class UserRepository {
    async find(id) { throw new Error('Not implemented'); }
    async save(user) { throw new Error('Not implemented'); }
}

// Legacy implementation
class LegacyUserRepository extends UserRepository {
    async find(id) {
        return legacyDb.query('SELECT * FROM USERS WHERE USER_ID = ?', [id]);
    }
}

// New implementation
class NewUserRepository extends UserRepository {
    async find(id) {
        return newDb.query('SELECT * FROM users WHERE id = ?', [id]);
    }
}

// Switch implementations via config
const userRepository = config.useNewDb
    ? new NewUserRepository()
    : new LegacyUserRepository();
```

## Common Mistakes

### 1. Underestimating Complexity

**Reality:** That simple-looking form has 50 edge cases you don't know about.

**Solution:** Talk to users. Read the code carefully. Test extensively.

### 2. Breaking Existing Functionality

**Reality:** "It didn't work before" doesn't matter. Users depend on current behavior.

**Solution:** Test against production behavior, not spec.

### 3. Modernizing Everything at Once

**Reality:** Big bang rewrites fail.

**Solution:** Small, incremental changes with tests.

### 4. Ignoring Operations

**Reality:** New system needs monitoring, deployment, logging.

**Solution:** Build operational capability alongside features.

## My Modernization Checklist

### Before Starting:
- [ ] Business case justified
- [ ] Scope defined and limited
- [ ] Stakeholder buy-in
- [ ] Timeline realistic (usually 2-3x initial estimate)

### During:
- [ ] Tests for existing behavior
- [ ] Feature flags for rollout
- [ ] Monitoring for new system
- [ ] Rollback plan
- [ ] Running old and new in parallel

### After Each Phase:
- [ ] Performance comparison
- [ ] Bug rate comparison
- [ ] User feedback collected
- [ ] Documentation updated

## Resources

- [Working Effectively with Legacy Code (Book)](https://www.oreilly.com/library/view/working-effectively-with/0131177052/)
- [Strangler Fig Pattern - Martin Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Branch by Abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html)
- [Approval Tests](https://approvaltests.com/)

---

*Questions about legacy modernization? [Let me know](mailto:jordan@jordananderson.us).*
