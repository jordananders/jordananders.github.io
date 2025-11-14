---
layout: default
title:  "Authentication vs Authorization: Implementing It Right"
date:   2025-11-14 17:00:00
categories: Security Development Authentication
---

I've seen too many applications confuse authentication and authorization. They implement great login systems (authentication) but then let any logged-in user do anything (broken authorization). Or they check permissions meticulously but have weak authentication that lets anyone in.

You need both. Here's the difference and how to implement them correctly.

## The Core Difference

**Authentication**: Who are you?
**Authorization**: What are you allowed to do?

Think of it like entering a building:
- **Authentication** is showing your ID at the front desk
- **Authorization** is your badge determining which floors you can access

Authentication always happens first. You can't determine what someone can do until you know who they are.

## Authentication: Proving Identity

### The Three Factors

Authentication is based on one or more of these factors:

**Something you know:**
- Password
- PIN
- Security question

**Something you have:**
- Phone (for SMS code)
- Hardware token
- Smart card

**Something you are:**
- Fingerprint
- Face recognition
- Retina scan

**Best practice**: Use at least two factors (Multi-Factor Authentication / MFA).

### Common Authentication Methods

#### 1. Username/Password (Basic Auth)

**How it works:**
```http
GET /api/users
Authorization: Basic base64(username:password)
```

**Pros:**
- Simple to implement
- Works everywhere

**Cons:**
- Credentials sent with every request
- Vulnerable to credential stuffing
- No session management

**When to use:** Internal tools, dev environments only

**Never use for:** Production APIs, user-facing applications

#### 2. Session-Based Authentication

**How it works:**

```javascript
// Login endpoint
app.post('/login', async (req, res) => {
    const { username, password } = req.body;

    const user = await db.findUser(username);
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
        return res.status(401).send('Invalid credentials');
    }

    // Create session
    req.session.userId = user.id;
    res.send({ message: 'Logged in' });
});

// Protected endpoint
app.get('/profile', requireAuth, (req, res) => {
    // req.session.userId available here
    const user = db.getUser(req.session.userId);
    res.json(user);
});
```

**Pros:**
- Server controls sessions (can revoke)
- Mature, well-understood
- Works with cookies (browser handles storage)

**Cons:**
- Requires server-side session storage
- Harder to scale horizontally
- CSRF vulnerabilities (need CSRF tokens)

**When to use:** Traditional web applications

#### 3. Token-Based Authentication (JWT)

**How it works:**

```javascript
// Login endpoint
app.post('/login', async (req, res) => {
    const { username, password } = req.body;

    const user = await db.findUser(username);
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
        return res.status(401).send('Invalid credentials');
    }

    // Create JWT
    const token = jwt.sign(
        { userId: user.id, role: user.role },
        process.env.JWT_SECRET,
        { expiresIn: '1h' }
    );

    res.json({ token });
});

// Client stores token and sends with requests
// Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

// Protected endpoint
app.get('/profile', authenticateJWT, (req, res) => {
    // req.user available from decoded token
    res.json(req.user);
});

// Middleware
function authenticateJWT(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];

    if (!token) {
        return res.status(401).send('No token provided');
    }

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (err) {
        return res.status(403).send('Invalid token');
    }
}
```

**Pros:**
- Stateless (scales horizontally)
- Works across domains
- Mobile-friendly

**Cons:**
- Can't revoke tokens (until they expire)
- Token size (sent with every request)
- Must protect JWT secret

**When to use:** APIs, mobile apps, microservices

#### 4. OAuth 2.0 / OpenID Connect

**How it works:**

```javascript
// Redirect to OAuth provider
app.get('/login', (req, res) => {
    const authUrl = `https://oauth-provider.com/authorize?
        client_id=${CLIENT_ID}&
        redirect_uri=${REDIRECT_URI}&
        response_type=code&
        scope=openid email profile`;

    res.redirect(authUrl);
});

// OAuth callback
app.get('/callback', async (req, res) => {
    const { code } = req.query;

    // Exchange code for access token
    const tokenResponse = await fetch('https://oauth-provider.com/token', {
        method: 'POST',
        body: JSON.stringify({
            code,
            client_id: CLIENT_ID,
            client_secret: CLIENT_SECRET,
            redirect_uri: REDIRECT_URI,
            grant_type: 'authorization_code'
        })
    });

    const { access_token, id_token } = await tokenResponse.json();

    // Decode id_token to get user info
    const user = jwt.decode(id_token);

    req.session.user = user;
    res.redirect('/dashboard');
});
```

**Pros:**
- Don't store passwords
- Users can use existing accounts (Google, GitHub, etc.)
- Centralized authentication

**Cons:**
- Complex to implement correctly
- Dependency on third-party service
- User privacy concerns

**When to use:** User-facing apps, SaaS products

### Multi-Factor Authentication (MFA)

**2025 Best Practice:** MFA should be default, not optional.

**Implementation example (TOTP):**

```javascript
const speakeasy = require('speakeasy');

// Generate secret for user
const secret = speakeasy.generateSecret({ name: 'MyApp' });

// Store secret.base32 in database for user
user.mfaSecret = secret.base32;

// User scans QR code with authenticator app
const qrCode = await QRCode.toDataURL(secret.otpauth_url);

// Verify code during login
const verified = speakeasy.totp.verify({
    secret: user.mfaSecret,
    encoding: 'base32',
    token: userEnteredCode,
    window: 2  // Allow 2 time steps of variance
});

if (verified) {
    // Complete login
}
```

## Authorization: Controlling Access

Once you know *who* someone is, you need to determine *what* they can do.

### Authorization Models

#### 1. Role-Based Access Control (RBAC)

**Concept:** Users have roles, roles have permissions.

```javascript
// Define roles and permissions
const roles = {
    admin: ['users:read', 'users:write', 'users:delete', 'posts:*'],
    editor: ['posts:read', 'posts:write', 'posts:delete'],
    viewer: ['posts:read']
};

// Check permission
function hasPermission(user, permission) {
    const userPermissions = roles[user.role] || [];

    return userPermissions.some(p => {
        if (p === permission) return true;
        if (p.endsWith(':*')) {
            const resource = p.split(':')[0];
            return permission.startsWith(resource + ':');
        }
        return false;
    });
}

// Use in middleware
function requirePermission(permission) {
    return (req, res, next) => {
        if (!req.user) {
            return res.status(401).send('Not authenticated');
        }

        if (!hasPermission(req.user, permission)) {
            return res.status(403).send('Forbidden');
        }

        next();
    };
}

// Apply to routes
app.delete('/users/:id', requirePermission('users:delete'), (req, res) => {
    // Only admins can access this
});
```

**Pros:**
- Simple to understand
- Easy to audit
- Works for most applications

**Cons:**
- Can become complex with many roles
- Role explosion (need specific role for every combination)

**When to use:** Most applications

#### 2. Attribute-Based Access Control (ABAC)

**Concept:** Access based on attributes (user attributes, resource attributes, environment).

```javascript
function canAccess(user, resource, action) {
    const rules = [
        // Users can edit their own posts
        {
            resource: 'post',
            action: 'edit',
            condition: (user, resource) => user.id === resource.authorId
        },
        // Admins can edit any post
        {
            resource: 'post',
            action: 'edit',
            condition: (user) => user.role === 'admin'
        },
        // Users can view published posts
        {
            resource: 'post',
            action: 'view',
            condition: (user, resource) => resource.status === 'published'
        },
        // Users can view their own drafts
        {
            resource: 'post',
            action: 'view',
            condition: (user, resource) =>
                resource.status === 'draft' && user.id === resource.authorId
        }
    ];

    return rules.some(rule =>
        rule.resource === resource.type &&
        rule.action === action &&
        rule.condition(user, resource)
    );
}

// Usage
app.get('/posts/:id', authenticateJWT, async (req, res) => {
    const post = await db.getPost(req.params.id);

    if (!canAccess(req.user, post, 'view')) {
        return res.status(403).send('Forbidden');
    }

    res.json(post);
});
```

**Pros:**
- Fine-grained control
- Flexible
- Handles complex scenarios

**Cons:**
- More complex to implement
- Harder to audit
- Performance overhead

**When to use:** Complex authorization requirements, document management, healthcare

#### 3. Access Control Lists (ACL)

**Concept:** Each resource has a list of who can access it and what they can do.

```javascript
// Document with ACL
const document = {
    id: 123,
    title: "Project Plan",
    acl: [
        { userId: 'user1', permissions: ['read', 'write'] },
        { userId: 'user2', permissions: ['read'] },
        { groupId: 'team-leads', permissions: ['read', 'write', 'delete'] }
    ]
};

function hasAccess(user, document, permission) {
    // Check user-specific permissions
    const userAcl = document.acl.find(entry => entry.userId === user.id);
    if (userAcl && userAcl.permissions.includes(permission)) {
        return true;
    }

    // Check group permissions
    for (const group of user.groups) {
        const groupAcl = document.acl.find(entry => entry.groupId === group);
        if (groupAcl && groupAcl.permissions.includes(permission)) {
            return true;
        }
    }

    return false;
}
```

**Pros:**
- Resource-level control
- User-specific permissions
- Good for collaboration tools

**Cons:**
- Can become unwieldy at scale
- Harder to manage globally

**When to use:** File systems, document sharing, Google Docs-style applications

### Common Authorization Patterns

#### Resource Ownership

```javascript
app.delete('/posts/:id', authenticateJWT, async (req, res) => {
    const post = await db.getPost(req.params.id);

    // Check ownership
    if (post.authorId !== req.user.id && req.user.role !== 'admin') {
        return res.status(403).send('You can only delete your own posts');
    }

    await db.deletePost(req.params.id);
    res.send('Deleted');
});
```

#### Hierarchical Permissions

```javascript
const hierarchy = {
    owner: 4,
    admin: 3,
    editor: 2,
    viewer: 1
};

function hasMinimumRole(userRole, requiredRole) {
    return hierarchy[userRole] >= hierarchy[requiredRole];
}

app.post('/posts', authenticateJWT, (req, res) => {
    if (!hasMinimumRole(req.user.role, 'editor')) {
        return res.status(403).send('Requires editor role or higher');
    }

    // Create post
});
```

## Best Practices (2025)

### Authentication Best Practices

**1. Always use HTTPS**
```nginx
# Force HTTPS redirect
server {
    listen 80;
    return 301 https://$server_name$request_uri;
}
```

**2. Never store passwords in plain text**
```javascript
const bcrypt = require('bcrypt');

// Hash password
const hash = await bcrypt.hash(password, 10);

// Verify password
const match = await bcrypt.compare(password, hash);
```

**3. Implement rate limiting on login endpoints**
```javascript
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 attempts
    message: 'Too many login attempts, try again later'
});

app.post('/login', loginLimiter, async (req, res) => {
    // Login logic
});
```

**4. Use secure session configuration**
```javascript
app.use(session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: true,      // HTTPS only
        httpOnly: true,    // Not accessible via JavaScript
        sameSite: 'strict', // CSRF protection
        maxAge: 3600000    // 1 hour
    }
}));
```

**5. Implement MFA for sensitive operations**

Even if not requiring MFA for login, require it for:
- Password changes
- Email changes
- Financial transactions
- Admin actions

**6. Use established libraries**
```javascript
// DON'T: Roll your own crypto
function myHash(password) {
    return password.split('').reverse().join(''); // TERRIBLE
}

// DO: Use proven libraries
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(password, 10);
```

### Authorization Best Practices

**1. Principle of Least Privilege**

Grant minimum permissions needed:

```javascript
// BAD: Give everyone admin role
user.role = 'admin';

// GOOD: Grant specific permissions needed
user.permissions = ['posts:read', 'posts:write'];
```

**2. Check authorization on server, not client**

```javascript
// BAD: Client-side check only
if (currentUser.isAdmin) {
    <DeleteButton />  // Attacker can show button and call API
}

// GOOD: Server enforces
app.delete('/users/:id', requireRole('admin'), (req, res) => {
    // Server validates, regardless of client
});
```

**3. Validate ownership and permissions**

```javascript
// Check BOTH authentication AND authorization
app.put('/documents/:id', authenticateJWT, async (req, res) => {
    const doc = await db.getDocument(req.params.id);

    // Authenticated? (401)
    if (!req.user) {
        return res.status(401).send('Not authenticated');
    }

    // Authorized? (403)
    if (doc.ownerId !== req.user.id && req.user.role !== 'admin') {
        return res.status(403).send('Forbidden');
    }

    // Both checks passed
    await db.updateDocument(req.params.id, req.body);
    res.send('Updated');
});
```

**4. Log authorization failures**

```javascript
function checkPermission(user, resource, action) {
    const allowed = hasPermission(user, resource, action);

    if (!allowed) {
        logger.warn('Authorization failed', {
            userId: user.id,
            resource,
            action,
            timestamp: new Date()
        });
    }

    return allowed;
}
```

**5. Regular access reviews**

Audit permissions quarterly:
- Remove unused roles
- Revoke access for former employees
- Review admin access
- Check for permission creep

## Common Mistakes

### Mistake 1: Confusing 401 and 403

```javascript
// WRONG
if (!req.user) {
    return res.status(403).send('Forbidden');  // Should be 401
}

// CORRECT
if (!req.user) {
    return res.status(401).send('Unauthorized');  // Not authenticated
}

if (!hasPermission(req.user, 'admin')) {
    return res.status(403).send('Forbidden');  // Authenticated but not authorized
}
```

**401 Unauthorized**: Not authenticated (who are you?)
**403 Forbidden**: Authenticated but not authorized (I know who you are, but you can't do this)

### Mistake 2: Only Checking Authentication

```javascript
// BAD: Any logged-in user can delete any user
app.delete('/users/:id', requireAuth, async (req, res) => {
    await db.deleteUser(req.params.id);
});

// GOOD: Check both authentication and authorization
app.delete('/users/:id', requireAuth, requireRole('admin'), async (req, res) => {
    await db.deleteUser(req.params.id);
});
```

### Mistake 3: Client-Side Authorization

```javascript
// BAD: Trusting client
app.delete('/admin/users/:id', (req, res) => {
    // Client says they're admin, trust them!
    if (req.body.isAdmin) {
        await db.deleteUser(req.params.id);
    }
});

// GOOD: Server validates
app.delete('/admin/users/:id', requireRole('admin'), async (req, res) => {
    // Server checks auth token/session
    await db.deleteUser(req.params.id);
});
```

### Mistake 4: Storing Passwords Insecurely

```javascript
// NEVER
user.password = req.body.password;  // Plain text

// NEVER
user.password = md5(req.body.password);  // Weak hash

// GOOD
user.passwordHash = await bcrypt.hash(req.body.password, 10);
```

## Resources

- [Authentication and Authorization Best Practices - GitGuardian](https://blog.gitguardian.com/authentication-and-authorization/)
- [Authentication vs. Authorization - Security Boulevard (2025)](https://securityboulevard.com/2025/04/authentication-vs-authorization-understanding-the-pillars-of-identity-security/)
- [Authentication vs Authorization - Frontegg](https://frontegg.com/blog/authentication-vs-authorization)
- [Authentication and Authorization Best Practices in ASP.NET Core](https://antondevtips.com/blog/authentication-and-authorization-best-practices-in-aspnetcore)
- [Authentication vs Authorization: Best Practices to Build Secure APIs](https://www.getambassador.io/blog/authentication-vs-authorization-key-practices)
- [Authentication vs authorization - SailPoint](https://www.sailpoint.com/identity-library/difference-between-authentication-and-authorization)

---

*Questions about auth implementation? [Reach out](mailto:jordan@jordananderson.us).*
