---
layout: default
title:  "API Design Decisions I Wish I'd Made Differently"
date:   2025-11-14 23:00:00
categories: API Development Architecture
---

I've designed APIs that I'm proud of. And APIs that haunt me years later. The difference usually comes down to a few key decisions made early on.

Here's what I wish I'd known about API design before building production APIs.

## Decision 1: Versioning Strategy

### What I Did Wrong

**No versioning at all:**
```
GET /api/users
```

When I needed to change the response format, I had two bad options:
1. Break existing clients
2. Never change the API

**What I Should Have Done:**

```
GET /api/v1/users
```

Or use header versioning:
```
GET /api/users
Accept: application/vnd.myapp.v1+json
```

**Lesson:** Plan for versioning from day one. You WILL need to change things.

## Decision 2: Using Verbs in URLs

### What I Did Wrong

```
POST /api/getUser
POST /api/updateUserEmail
POST /api/deleteUser
```

This defeats the purpose of REST. HTTP methods already define the action.

**What I Should Have Done:**

```
GET    /api/users/:id          # Get user
PUT    /api/users/:id          # Update user
PATCH  /api/users/:id/email    # Update email only
DELETE /api/users/:id          # Delete user
```

**Lesson:** Use HTTP methods correctly. URLs should identify resources, not actions.

## Decision 3: Inconsistent Response Formats

### What I Did Wrong

```javascript
// Endpoint 1 returns array directly
GET /api/users
[{id: 1, name: "John"}, {id: 2, name: "Jane"}]

// Endpoint 2 wraps in object
GET /api/posts
{data: [{id: 1, title: "Post 1"}]}

// Endpoint 3 has different structure
GET /api/comments
{results: [...], count: 10}
```

Clients had to handle three different patterns.

**What I Should Have Done:**

```javascript
// Consistent format for all endpoints
{
    "data": [...],
    "meta": {
        "count": 100,
        "page": 1,
        "total_pages": 10
    }
}
```

**Lesson:** Pick one response format and stick to it everywhere.

## Decision 4: No Pagination from the Start

### What I Did Wrong

```
GET /api/users
```

Returns all users. Worked fine with 100 users. Broke catastrophically at 100,000 users.

**What I Should Have Done:**

```
GET /api/users?page=1&limit=20
```

Or cursor-based:
```
GET /api/users?cursor=abc123&limit=20
```

**Lesson:** Always paginate collections. Even if you think you'll never have many records.

## Decision 5: Poor Error Responses

### What I Did Wrong

```javascript
// Just HTTP status codes
400 Bad Request

// Or cryptic messages
{error: "Invalid input"}

// Or codes clients can't use
{error_code: "USR_001"}
```

Clients had no idea what went wrong or how to fix it.

**What I Should Have Done:**

```javascript
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Validation failed",
        "details": [
            {
                "field": "email",
                "message": "Email must be valid",
                "value": "notanemail"
            }
        ]
    }
}
```

**Lesson:** Errors should be actionable. Tell clients what's wrong and how to fix it.

## Decision 6: Not Using HTTP Status Codes Correctly

### What I Did Wrong

```javascript
// Everything returns 200, even errors
HTTP 200 OK
{success: false, error: "User not found"}
```

**What I Should Have Done:**

```
200 OK - Success
201 Created - Resource created
204 No Content - Success, no body
400 Bad Request - Client error (validation, etc.)
401 Unauthorized - Not authenticated
403 Forbidden - Authenticated but not authorized
404 Not Found - Resource doesn't exist
409 Conflict - Resource conflict (duplicate, etc.)
422 Unprocessable Entity - Validation error
500 Internal Server Error - Server error
```

**Lesson:** HTTP status codes exist for a reason. Use them.

## Decision 7: Exposing Database IDs

### What I Did Wrong

```
GET /api/users/1
GET /api/users/2
GET /api/users/3
```

Sequential IDs let users:
- Enumerate all resources
- Know how many resources exist
- Potentially guess valid IDs

**What I Should Have Done:**

```
GET /api/users/550e8400-e29b-41d4-a716-446655440000
```

Use UUIDs or other non-sequential identifiers.

**Lesson:** Don't expose internal database IDs in APIs.

## Decision 8: No Rate Limiting

### What I Did Wrong

No rate limits. First time someone hit the API in a loop, they brought down the entire service.

**What I Should Have Done:**

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,  // 100 requests per window
    standardHeaders: true,
    legacyHeaders: false,
    message: 'Too many requests, please try again later'
});

app.use('/api/', limiter);
```

**Response headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1609459200
```

**Lesson:** Always rate limit APIs. Protect your infrastructure and other users.

## Decision 9: Accepting Any Field in Updates

### What I Did Wrong

```javascript
PATCH /api/users/:id
{
    "name": "John",
    "role": "admin",  // User shouldn't be able to change this!
    "balance": 999999  // Definitely shouldn't change this!
}
```

Accepted any field client sent. Led to privilege escalation.

**What I Should Have Done:**

```javascript
// Whitelist allowed fields
const allowedFields = ['name', 'email', 'bio'];

app.patch('/api/users/:id', (req, res) => {
    const updates = {};
    for (const field of allowedFields) {
        if (req.body[field] !== undefined) {
            updates[field] = req.body[field];
        }
    }

    // Only update whitelisted fields
    await db.updateUser(req.params.id, updates);
});
```

**Lesson:** Whitelist accepted fields. Never trust client input.

## Decision 10: No API Documentation

### What I Did Wrong

No documentation. Told people to "read the code" or "check the examples."

**What I Should Have Done:**

Use OpenAPI/Swagger:

```yaml
openapi: 3.0.0
info:
  title: My API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        email:
          type: string
          format: email
```

**Lesson:** Document your API. Future developers (including you) will thank you.

## My Current API Design Checklist

When designing a new API, I check:

### URL Structure
- [ ] RESTful resource-based URLs
- [ ] No verbs in URLs
- [ ] Consistent naming (plural for collections)
- [ ] Version in URL or header

### Request/Response
- [ ] Consistent response format
- [ ] Proper HTTP status codes
- [ ] Pagination for collections
- [ ] Field filtering/selection
- [ ] Sorted results where appropriate

### Security
- [ ] Authentication required
- [ ] Authorization checks
- [ ] Rate limiting
- [ ] Input validation
- [ ] Whitelist accepted fields
- [ ] Use UUIDs not sequential IDs
- [ ] HTTPS only

### Error Handling
- [ ] Consistent error format
- [ ] Actionable error messages
- [ ] Proper HTTP status codes
- [ ] Validation errors list all issues

### Documentation
- [ ] OpenAPI/Swagger spec
- [ ] Example requests/responses
- [ ] Authentication guide
- [ ] Error code reference

## Good API Design Patterns

### Filtering

```
GET /api/users?role=admin&status=active
```

### Sorting

```
GET /api/users?sort=created_at:desc
```

### Field Selection (Sparse Fieldsets)

```
GET /api/users?fields=id,name,email
```

### Including Related Resources

```
GET /api/posts?include=author,comments
```

### Pagination

```
GET /api/users?page=2&per_page=20

Response:
{
    "data": [...],
    "pagination": {
        "page": 2,
        "per_page": 20,
        "total": 1000,
        "total_pages": 50
    }
}
```

### Bulk Operations

```
POST /api/users/batch

{
    "operations": [
        {"method": "POST", "path": "/users", "body": {...}},
        {"method": "PATCH", "path": "/users/123", "body": {...}}
    ]
}
```

## Final Thoughts

API design is hard to change once clients depend on it. Invest time upfront to:

1. Version from day one
2. Use consistent patterns
3. Document everything
4. Validate all input
5. Handle errors gracefully

The cost of fixing a bad API design later is 10x the cost of getting it right initially.

## Resources

- [REST API Tutorial](https://restfulapi.net/)
- [Microsoft API Design Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [OpenAPI Specification](https://swagger.io/specification/)

---

*Questions about API design? [Let me know](mailto:jordan@jordananderson.us).*
