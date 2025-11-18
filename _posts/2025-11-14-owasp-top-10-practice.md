---
layout: default
title:  "OWASP Top 10 in Practice: Real-World Examples"
date:   2025-11-14 21:00:00
categories: Security WebDevelopment OWASP
---

The OWASP Top 10 was just updated in November 2025. I've fixed vulnerabilities from every category on this list. Here's what these vulnerabilities actually look like in code, and how to prevent them.

## A01:2025 - Broken Access Control

**Still #1.** This has been the top vulnerability since 2021. Now includes SSRF (Server-Side Request Forgery).

### What It Looks Like

```javascript
// BAD: User can access any document by changing the ID
app.get('/documents/:id', requireAuth, async (req, res) => {
    const doc = await db.getDocument(req.params.id);
    res.json(doc);  // No ownership check!
});

// Attacker changes URL from /documents/123 to /documents/999
// Gets access to someone else's document
```

### How to Fix It

```javascript
// GOOD: Check ownership
app.get('/documents/:id', requireAuth, async (req, res) => {
    const doc = await db.getDocument(req.params.id);

    if (!doc) {
        return res.status(404).send('Not found');
    }

    // Check if user owns document or is admin
    if (doc.ownerId !== req.user.id && !req.user.isAdmin) {
        return res.status(403).send('Forbidden');
    }

    res.json(doc);
});
```

### SSRF Example

```javascript
// BAD: Server fetches user-supplied URL
app.post('/fetch-image', async (req, res) => {
    const imageUrl = req.body.url;
    const image = await fetch(imageUrl);  // Attacker can access internal services!
    res.send(image);
});

// Attacker sends: {"url": "http://localhost:8080/admin/secrets"}

// GOOD: Whitelist allowed domains
app.post('/fetch-image', async (req, res) => {
    const imageUrl = req.body.url;
    const allowedDomains = ['cdn.example.com', 'images.example.com'];

    const url = new URL(imageUrl);
    if (!allowedDomains.includes(url.hostname)) {
        return res.status(400).send('Invalid domain');
    }

    const image = await fetch(imageUrl);
    res.send(image);
});
```

## A02:2025 - Security Misconfiguration

**Moved up from #5.** Default configurations, unnecessary features enabled, verbose error messages.

### What It Looks Like

```javascript
// BAD: Detailed errors in production
app.use((err, req, res, next) => {
    res.status(500).json({
        error: err.message,
        stack: err.stack,  // Exposes internal paths and logic!
        query: req.query
    });
});

// Attacker sees:
// {
//   "error": "ENOENT: no such file or directory, open '/var/www/app/config/database.yml'",
//   "stack": "Error: ENOENT...\n    at /var/www/app/controllers/user.js:45:12"
// }
```

### How to Fix It

```javascript
// GOOD: Generic errors in production, log details
app.use((err, req, res, next) => {
    // Log full error server-side
    logger.error('Request error', {
        error: err.message,
        stack: err.stack,
        url: req.url,
        user: req.user?.id
    });

    // Send generic error to client
    if (process.env.NODE_ENV === 'production') {
        res.status(500).json({
            error: 'An error occurred. Please try again later.'
        });
    } else {
        // Detailed errors in development
        res.status(500).json({
            error: err.message,
            stack: err.stack
        });
    }
});
```

### Configuration Checklist

```javascript
// Check your configuration
const securityConfig = {
    // Disable unnecessary features
    enableDebugMode: false,
    enableTestEndpoints: false,

    // Use secure defaults
    cookieSecure: true,
    cookieHttpOnly: true,
    cookieSameSite: 'strict',

    // Remove default credentials
    adminPassword: process.env.ADMIN_PASSWORD, // Not 'admin'

    // Disable directory listing
    serveDirectoryIndexes: false,

    // Use security headers
    useHelmet: true
};
```

## A03:2025 - Software Supply Chain Failures

**NEW in 2025.** Expanded from "Vulnerable and Outdated Components." Now includes supply chain attacks.

### What It Looks Like

```json
// package.json with vulnerable dependencies
{
  "dependencies": {
    "express": "4.16.0",  // Vulnerable version from 2018
    "lodash": "4.17.15",  // Known prototype pollution
    "axios": "0.19.0"     // Outdated, has vulnerabilities
  }
}
```

### How to Fix It

```bash
# Check for vulnerabilities
npm audit

# Update dependencies
npm update

# Fix vulnerabilities automatically
npm audit fix

# Use tools like Snyk or Dependabot
# They create PRs automatically when vulnerabilities are found
```

### Supply Chain Best Practices

```javascript
// Verify package integrity
// package-lock.json contains integrity hashes
{
  "packages": {
    "node_modules/express": {
      "version": "4.18.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-5/PsL6iGPdfQ/lKM1UuielYgv3BUoJfz1aUwU9vHZ+J7gyvwdQXFEBIEIaxeGf0GIcreATNyBExtalisDbuMqQ=="
    }
  }
}

// Always commit package-lock.json / yarn.lock
// Don't blindly install packages - review them first
// Use private registry for internal packages
```

## A04:2025 - Cryptographic Failures

**Dropped from #2 to #4.** Exposing sensitive data, weak crypto, insecure transmission.

### What It Looks Like

```javascript
// BAD: Storing passwords in plain text
const user = {
    username: 'john',
    password: 'password123'  // NEVER DO THIS
};

// BAD: Weak hashing
const crypto = require('crypto');
const hash = crypto.createHash('md5').update(password).digest('hex');

// BAD: Transmitting sensitive data over HTTP
fetch('http://api.example.com/user/ssn?ssn=123-45-6789');
```

### How to Fix It

```javascript
// GOOD: Hash passwords properly
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(password, 10);

// GOOD: Verify password
const match = await bcrypt.compare(password, hash);

// GOOD: Always use HTTPS
fetch('https://api.example.com/user/profile');

// GOOD: Encrypt sensitive data at rest
const crypto = require('crypto');
const algorithm = 'aes-256-gcm';
const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
const iv = crypto.randomBytes(16);

const cipher = crypto.createCipheriv(algorithm, key, iv);
let encrypted = cipher.update(sensitiveData, 'utf8', 'hex');
encrypted += cipher.final('hex');
```

## A05:2025 - Injection

**Dropped from #3 to #5.** SQL injection, NoSQL injection, XSS, command injection.

### SQL Injection

```javascript
// BAD: Direct string concatenation
app.get('/users', (req, res) => {
    const userId = req.query.id;
    const query = `SELECT * FROM users WHERE id = ${userId}`;
    db.query(query);  // Attacker sends: ?id=1 OR 1=1
});

// GOOD: Parameterized queries
app.get('/users', (req, res) => {
    const userId = req.query.id;
    const query = 'SELECT * FROM users WHERE id = ?';
    db.query(query, [userId]);
});
```

### XSS (Cross-Site Scripting)

```javascript
// BAD: Unsanitized user input
app.get('/search', (req, res) => {
    const searchTerm = req.query.q;
    res.send(`Results for: ${searchTerm}`);
    // Attacker sends: ?q=<script>alert('XSS')</script>
});

// GOOD: Escape user input
const escape = require('escape-html');
app.get('/search', (req, res) => {
    const searchTerm = escape(req.query.q);
    res.send(`Results for: ${searchTerm}`);
});

// BETTER: Use templating engine that auto-escapes
res.render('search', { searchTerm: req.query.q });
```

### Command Injection

```javascript
// BAD: User input in shell command
const exec = require('child_process').exec;
app.post('/convert', (req, res) => {
    const filename = req.body.filename;
    exec(`convert ${filename} output.pdf`);  // DANGEROUS!
    // Attacker sends: {"filename": "file.txt; rm -rf /"}
});

// GOOD: Use libraries instead of shell commands
const converter = require('pdf-converter');
app.post('/convert', (req, res) => {
    const filename = req.body.filename;
    converter.convert(filename, 'output.pdf');
});
```

## A06:2025 - Insecure Design

**Dropped from #4 to #6.** Flaws in architecture and design, not just implementation bugs.

### What It Looks Like

```javascript
// BAD: No rate limiting on password reset
app.post('/reset-password', async (req, res) => {
    const email = req.body.email;
    await sendPasswordResetEmail(email);
    res.send('Reset email sent');
});
// Attacker can enumerate valid emails by spamming this endpoint

// BAD: Unlimited password attempts
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const user = await db.findUser(username);
    if (await bcrypt.compare(password, user.passwordHash)) {
        res.send('Logged in');
    } else {
        res.status(401).send('Invalid credentials');
    }
});
// Attacker can brute force passwords
```

### How to Fix It

```javascript
// GOOD: Rate limiting
const rateLimit = require('express-rate-limit');

const resetLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 3,  // 3 requests per window
    message: 'Too many reset attempts, try again later'
});

app.post('/reset-password', resetLimiter, async (req, res) => {
    const email = req.body.email;
    await sendPasswordResetEmail(email);
    // Always return same message (don't reveal if email exists)
    res.send('If that email exists, a reset link was sent');
});

// GOOD: Account lockout after failed attempts
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const user = await db.findUser(username);

    if (user.failedAttempts >= 5) {
        const lockoutExpiry = user.lastFailedAttempt + (15 * 60 * 1000);
        if (Date.now() < lockoutExpiry) {
            return res.status(429).send('Account locked. Try again in 15 minutes');
        }
    }

    if (await bcrypt.compare(password, user.passwordHash)) {
        await db.updateUser(user.id, { failedAttempts: 0 });
        res.send('Logged in');
    } else {
        await db.updateUser(user.id, {
            failedAttempts: user.failedAttempts + 1,
            lastFailedAttempt: Date.now()
        });
        res.status(401).send('Invalid credentials');
    }
});
```

## A07:2025 - Authentication Failures

**Dropped from #7.** Weak passwords, credential stuffing, session hijacking.

### What It Looks Like

```javascript
// BAD: Weak password requirements
function validatePassword(password) {
    return password.length >= 6;  // Too weak!
}

// BAD: Session without timeout
req.session.userId = user.id;
// Session never expires

// BAD: Predictable session IDs
const sessionId = `${user.id}_${Date.now()}`;  // Predictable!
```

### How to Fix It

```javascript
// GOOD: Strong password requirements
function validatePassword(password) {
    const minLength = 8;
    const hasUpper = /[A-Z]/.test(password);
    const hasLower = /[a-z]/.test(password);
    const hasNumber = /\d/.test(password);
    const hasSpecial = /[!@#$%^&*]/.test(password);

    return password.length >= minLength &&
           hasUpper && hasLower && hasNumber && hasSpecial;
}

// GOOD: Session with timeout and security flags
app.use(session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: true,        // HTTPS only
        httpOnly: true,      // No JavaScript access
        sameSite: 'strict',  // CSRF protection
        maxAge: 3600000      // 1 hour timeout
    }
}));

// GOOD: Implement MFA
const speakeasy = require('speakeasy');
const verified = speakeasy.totp.verify({
    secret: user.mfaSecret,
    token: req.body.mfaCode
});
```

## A09:2025 - Security Logging and Monitoring Failures

**Critical for detecting breaches.** Can't defend against what you can't see.

### What to Log

```javascript
const winston = require('winston');
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.json(),
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' })
    ]
});

// Log authentication events
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const user = await db.findUser(username);

    if (await bcrypt.compare(password, user.passwordHash)) {
        logger.info('Successful login', {
            userId: user.id,
            ip: req.ip,
            userAgent: req.get('user-agent'),
            timestamp: new Date()
        });
        res.send('Logged in');
    } else {
        logger.warn('Failed login attempt', {
            username: username,
            ip: req.ip,
            userAgent: req.get('user-agent'),
            timestamp: new Date()
        });
        res.status(401).send('Invalid credentials');
    }
});

// Log authorization failures
function checkPermission(user, resource, action) {
    const allowed = hasPermission(user, resource, action);

    if (!allowed) {
        logger.warn('Authorization failure', {
            userId: user.id,
            resource: resource,
            action: action,
            ip: req.ip,
            timestamp: new Date()
        });
    }

    return allowed;
}

// Log security-relevant events
logger.info('Password changed', { userId: user.id });
logger.warn('Suspicious activity detected', { userId: user.id, pattern: 'rapid_requests' });
logger.error('Security exception', { error: err.message, stack: err.stack });
```

### What NOT to Log

```javascript
// NEVER log sensitive data
logger.info('Login attempt', {
    username: username,
    password: password,  // NEVER LOG PASSWORDS
    ssn: user.ssn,       // NEVER LOG PII
    creditCard: user.cc  // NEVER LOG PAYMENT DATA
});
```

## A10:2025 - Mishandling of Exceptional Conditions

**NEW in 2025.** Error handling that leaks information or causes security issues.

### What It Looks Like

```javascript
// BAD: Errors reveal system details
app.get('/file', (req, res) => {
    try {
        const data = fs.readFileSync(req.query.path);
        res.send(data);
    } catch (err) {
        res.status(500).send(err.message);
        // Reveals: "ENOENT: no such file or directory, open '/etc/passwd'"
    }
});

// BAD: Different error messages reveal information
app.post('/login', async (req, res) => {
    const user = await db.findUser(req.body.username);
    if (!user) {
        return res.status(401).send('User not found');  // Reveals user doesn't exist
    }
    if (!await bcrypt.compare(req.body.password, user.passwordHash)) {
        return res.status(401).send('Incorrect password');  // Reveals user exists
    }
    res.send('Logged in');
});
```

### How to Fix It

```javascript
// GOOD: Generic error messages
app.get('/file', (req, res) => {
    try {
        // Validate path first
        if (!isValidPath(req.query.path)) {
            return res.status(400).send('Invalid file path');
        }

        const data = fs.readFileSync(req.query.path);
        res.send(data);
    } catch (err) {
        logger.error('File read error', { path: req.query.path, error: err.message });
        res.status(500).send('Unable to read file');
    }
});

// GOOD: Same error message regardless of reason
app.post('/login', async (req, res) => {
    const user = await db.findUser(req.body.username);
    const valid = user && await bcrypt.compare(req.body.password, user.passwordHash);

    if (!valid) {
        logger.warn('Failed login', { username: req.body.username });
        // Same message whether user doesn't exist or password is wrong
        return res.status(401).send('Invalid username or password');
    }

    res.send('Logged in');
});
```

## My Security Checklist

When reviewing code for security, I check:

- [ ] **Access Control**: Every endpoint checks user authorization
- [ ] **Input Validation**: All user input is validated and sanitized
- [ ] **Dependencies**: `npm audit` shows no vulnerabilities
- [ ] **Crypto**: Passwords hashed with bcrypt, sensitive data encrypted
- [ ] **Injection**: Parameterized queries, no `eval()` or `exec()` with user input
- [ ] **HTTPS**: All traffic over HTTPS, secure cookies
- [ ] **Rate Limiting**: Auth endpoints have rate limits
- [ ] **Logging**: Security events logged (but not sensitive data)
- [ ] **Error Handling**: Generic errors to users, detailed logs server-side
- [ ] **Configuration**: No default credentials, debug mode off in production

## Resources

- [OWASP Top 10 2025](https://owasp.org/www-project-top-ten/)
- [OWASP 2025 Release Announcement](https://www.theregister.com/2025/11/11/new_owasp_top_ten_broken/)
- [Two New Categories Added - SecurityWeek](https://www.securityweek.com/two-new-web-application-risk-categories-added-to-owasp-top-10/)
- [OWASP Top 10 2025 Analysis - GBHackers](https://gbhackers.com/owasp-top-10-2025-released/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

---

*Questions about web application security? [Reach out](mailto:jordan@jordananderson.us).*
