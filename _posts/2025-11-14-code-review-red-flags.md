---
layout: default
title:  "Code Review Red Flags: What I Look For"
date:   2025-11-14 14:00:00
categories: Development Security CodeQuality
---

I've reviewed thousands of pull requests. Most are fine. Some are great. And some make me reach for coffee before I even start writing comments.

After years of code reviews, I've developed a mental checklist of red flags—patterns that consistently lead to bugs, security vulnerabilities, or maintenance nightmares. Here's what makes me stop and dig deeper.

## The Big Red Flags

These are the issues that can cause real damage—security breaches, data loss, system failures. When I see these, everything else waits.

### 1. User Input Goes Directly to Database/Shell/Eval

**Red flag:**
```javascript
// NO NO NO
const userId = req.query.id;
db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

**Why it's bad:** SQL injection. An attacker sends `1 OR 1=1--` and gets all user records.

**Recent context:** A 2025 Veracode study found nearly 45% of AI-generated code contains OWASP Top-10 flaws, with injection vulnerabilities being the most common.

**What I look for:**
- Concatenated SQL queries
- `eval()` or `exec()` with user input
- Shell commands built with string concatenation
- Unsanitized data in templates

**Better approach:**
```javascript
// Parameterized query
const userId = req.query.id;
db.query('SELECT * FROM users WHERE id = ?', [userId]);
```

### 2. Authentication Logic in Weird Places

**Red flag:**
```javascript
// Controller checking permissions directly
if (user.id === document.ownerId || user.role === 'admin') {
    // Allow access
}
```

**Why it's bad:** This logic gets copied, modified slightly, and inevitably someone forgets a check. Or the conditions drift between implementations.

**What I look for:**
- Permission checks scattered across controllers
- Inconsistent authorization logic
- Missing authorization checks
- Role checks that don't match documented access model

**Better approach:**
```javascript
// Centralized permission system
if (permissionService.canAccess(user, document, 'edit')) {
    // Allow access
}
```

### 3. Secrets in Code

**Red flag:**
```python
# Pushed to public repo
API_KEY = "sk_live_1234567890abcdef"
DB_PASSWORD = "P@ssw0rd123"
ENCRYPTION_KEY = "super_secret_key"
```

**Why it's bad:** Once in version control, it's there forever. Even if you delete it in the next commit, it's in the history.

**What I look for:**
- Hardcoded API keys
- Database credentials
- Encryption keys
- JWT secrets
- Third-party service tokens

**Better approach:**
```python
# Environment variables
API_KEY = os.getenv('API_KEY')
DB_PASSWORD = os.getenv('DB_PASSWORD')

# Or secrets management
secrets = vault_client.get_secrets()
```

### 4. Trust ing Data from Client Side

**Red flag:**
```javascript
// Trusting client-sent prices
app.post('/checkout', (req, res) => {
    const { itemId, price, quantity } = req.body;
    const total = price * quantity; // Client controls the price!
    processPayment(total);
});
```

**Why it's bad:** Client can send any price they want. `price: 0.01` for a $1000 item? Sure.

**What I look for:**
- Prices from client
- User roles/permissions from client
- Calculation results from client (taxes, totals, discounts)
- Any business logic result from client

**Better approach:**
```javascript
// Look up authoritative data server-side
app.post('/checkout', (req, res) => {
    const { itemId, quantity } = req.body;
    const item = database.getItem(itemId);
    const total = item.price * quantity;
    processPayment(total);
});
```

### 5. Error Messages That Leak Information

**Red flag:**
```python
except Exception as e:
    return f"Database error: {str(e)}"
    # Outputs: "Table 'admin_users' doesn't exist"
    # Or: "ORA-01017: invalid username/password; logon denied"
```

**Why it's bad:** Attackers learn your database structure, technology stack, file paths, etc.

**What I look for:**
- Raw exception messages sent to users
- Stack traces in production responses
- Detailed database errors
- File paths in error messages

**Better approach:**
```python
except Exception as e:
    logger.error(f"Database error: {str(e)}")  # Log details server-side
    return "An error occurred. Please try again later."  # Generic message to user
```

## Security Red Flags

These might not be immediately exploitable, but they create unnecessary risk.

### 6. Weak Randomness for Security

**Red flag:**
```javascript
// Don't use Math.random() for security
const token = Math.random().toString(36).substring(2);
const sessionId = Date.now() + Math.random();
```

**Why it's bad:** `Math.random()` is predictable. Not suitable for tokens, session IDs, or anything security-related.

**Better approach:**
```javascript
// Use cryptographically secure random
const crypto = require('crypto');
const token = crypto.randomBytes(32).toString('hex');
```

### 7. Password Handling Issues

**Red flags:**
```python
# Storing plaintext passwords
user.password = request.password

# Weak hashing
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()

# Logging passwords
logger.info(f"User {username} logged in with password {password}")
```

**What I look for:**
- Passwords in logs
- Weak hashing (MD5, SHA1 without salt)
- Passwords transmitted in URLs
- Password requirements that are too weak

**Better approach:**
```python
import bcrypt

# Strong hashing with automatic salt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt())

# Verify password
if bcrypt.checkpw(password.encode(), stored_hash):
    # Password correct
```

### 8. Broken Access Control

**Red flag:**
```javascript
// Only checking if user is logged in, not if they own the resource
app.delete('/documents/:id', requireAuth, (req, res) => {
    Document.delete(req.params.id);  // Any logged-in user can delete any document!
});
```

**What I look for:**
- Missing ownership checks
- Role checks without resource ownership
- Direct object references without validation
- Admin-only functions accessible to regular users

**Better approach:**
```javascript
app.delete('/documents/:id', requireAuth, async (req, res) => {
    const doc = await Document.findById(req.params.id);

    if (!doc) return res.status(404).send('Not found');
    if (doc.ownerId !== req.user.id && !req.user.isAdmin) {
        return res.status(403).send('Forbidden');
    }

    await doc.delete();
    res.status(200).send('Deleted');
});
```

## Code Quality Red Flags

These might not break anything immediately, but they make maintenance harder and bugs more likely.

### 9. The God Object

**Red flag:**
```python
class UserManager:
    def create_user(self): pass
    def authenticate(self): pass
    def send_email(self): pass
    def generate_report(self): pass
    def process_payment(self): pass
    def update_inventory(self): pass
    def calculate_shipping(self): pass
    def validate_address(self): pass
    # ... 50 more methods
```

**Why it's bad:** Violates single responsibility principle. Impossible to test. Every change risks breaking something.

**What I look for:**
- Classes with 20+ methods
- Classes with vague names like `Manager`, `Helper`, `Utility`
- Classes that do multiple unrelated things

### 10. Copy-Paste Code

**Red flag:**
```javascript
function calculatePriceForProduct1(quantity) {
    let base = 10;
    let discount = quantity > 100 ? 0.1 : 0;
    return base * quantity * (1 - discount);
}

function calculatePriceForProduct2(quantity) {
    let base = 20;
    let discount = quantity > 100 ? 0.1 : 0;
    return base * quantity * (1 - discount);
}

// ... 10 more nearly identical functions
```

**Why it's bad:** Fix a bug? Now you have to fix it in 12 places. Guaranteed someone will miss one.

**What I look for:**
- Identical or near-identical code blocks
- Same logic with minor variations
- Comments like "same as above but for X"

**Better approach:**
```javascript
function calculatePrice(basePrice, quantity) {
    const discount = quantity > 100 ? 0.1 : 0;
    return basePrice * quantity * (1 - discount);
}
```

### 11. Magic Numbers

**Red flag:**
```java
if (status == 3 && amount > 1000 && type != 7) {
    // What do these numbers mean?
}
```

**What I look for:**
- Unexplained numbers in conditionals
- Hardcoded configuration values
- Status codes without constants

**Better approach:**
```java
final int STATUS_APPROVED = 3;
final int AMOUNT_THRESHOLD = 1000;
final int TYPE_STANDARD = 7;

if (status == STATUS_APPROVED &&
    amount > AMOUNT_THRESHOLD &&
    type != TYPE_STANDARD) {
    // Clear what this checks
}
```

### 12. Missing Error Handling

**Red flag:**
```javascript
// Just hoping it works
const data = JSON.parse(userInput);
fs.writeFileSync('/important/file.txt', data);
const result = apiClient.call(data);
```

**What I look for:**
- Network calls without timeout or error handling
- File operations without error checks
- JSON parsing without try/catch
- Null dereferencing potential

**Better approach:**
```javascript
try {
    const data = JSON.parse(userInput);

    try {
        fs.writeFileSync('/important/file.txt', data);
    } catch (err) {
        logger.error('File write failed:', err);
        return res.status(500).send('Could not save data');
    }

    const result = await apiClient.call(data, { timeout: 5000 });
    return res.status(200).json(result);

} catch (err) {
    if (err instanceof SyntaxError) {
        return res.status(400).send('Invalid JSON');
    }
    logger.error('Unexpected error:', err);
    return res.status(500).send('Internal error');
}
```

## Performance Red Flags

### 13. N+1 Queries

**Red flag:**
```python
# Get all users
users = User.all()

# For each user, make another database call
for user in users:
    orders = Order.where(user_id=user.id)  # N additional queries!
    print(f"{user.name}: {len(orders)} orders")
```

**Why it's bad:** 1000 users = 1001 database queries. This kills performance.

**What I look for:**
- Queries inside loops
- Lazy loading in iterations
- Multiple round-trips to database/API

**Better approach:**
```python
# Single query with join
users_with_orders = User.join(Order).group_by(User.id).all()

for user in users_with_orders:
    print(f"{user.name}: {len(user.orders)} orders")
```

### 14. Loading Everything When You Need Little

**Red flag:**
```sql
-- Just need the count
SELECT * FROM logs WHERE date > '2025-01-01';
-- Returns 10 million rows, each with 50 columns
```

**What I look for:**
- `SELECT *` when specific columns needed
- Loading entire collections into memory
- Fetching data that's never used

**Better approach:**
```sql
-- Just get what you need
SELECT COUNT(*) FROM logs WHERE date > '2025-01-01';
```

## Testing Red Flags

### 15. Tests That Don't Actually Test

**Red flag:**
```python
def test_calculate_total():
    result = calculate_total([10, 20, 30])
    assert result is not None  # Passes even if result is wrong!
```

**What I look for:**
- Assertions that only check "not null"
- Tests with no assertions
- Tests that mock everything (testing the mocks, not the code)
- Tests that always pass

**Better approach:**
```python
def test_calculate_total():
    result = calculate_total([10, 20, 30])
    assert result == 60
    assert isinstance(result, (int, float))
```

## My Code Review Checklist

When reviewing a PR, I systematically check:

**Security:**
- [ ] All user input validated and sanitized
- [ ] No SQL injection vulnerabilities
- [ ] No hardcoded secrets
- [ ] Authentication/authorization checks in place
- [ ] Error messages don't leak info
- [ ] Crypto uses secure random
- [ ] Passwords handled correctly

**Logic:**
- [ ] Edge cases handled
- [ ] Null/undefined checks where needed
- [ ] Error handling in place
- [ ] Resource cleanup (connections, files closed)

**Quality:**
- [ ] Code is readable
- [ ] Functions do one thing
- [ ] No code duplication
- [ ] Names are clear
- [ ] Comments explain "why," not "what"

**Performance:**
- [ ] No N+1 queries
- [ ] Efficient algorithms
- [ ] Appropriate indexing
- [ ] No unnecessary data loading

**Tests:**
- [ ] Tests cover new functionality
- [ ] Tests actually verify behavior
- [ ] Edge cases tested
- [ ] Tests are clear and maintainable

## Tools I Use

While manual review catches business logic issues that tools can't, automation helps with the obvious stuff:

**SAST (Static Analysis):**
- SonarQube
- Checkmarx
- Semgrep

**Dependency Scanning:**
- Dependabot
- Snyk
- OWASP Dependency Check

**Linters:**
- ESLint (JavaScript)
- Pylint (Python)
- RuboCop (Ruby)

**Code Formatting:**
- Prettier (JavaScript)
- Black (Python)
- Consistent formatting means I focus on logic, not style

## The AI Code Review Challenge (2025 Update)

With AI-generated code becoming more common, I've added new red flags:

**AI-Generated Code Issues:**
- Overly generic variable names (item1, data2, result3)
- Copy-pasted patterns that don't fit the context
- Missing business logic understanding
- Security vulnerabilities (45% of AI code has OWASP Top-10 flaws)

**My approach:**
- Treat AI-generated code like junior developer code
- Review it extra carefully for security issues
- Verify it actually solves the problem correctly
- Check that it follows project patterns

## When to Push Back

Not all issues deserve pushback, but these do:

**Always:**
- Security vulnerabilities
- Data loss risks
- Breaking changes without migration path
- Hardcoded secrets

**Usually:**
- Missing tests for critical functionality
- Copy-paste code that should be refactored
- Performance issues that will impact users

**Sometimes:**
- Code style issues (if they impact readability)
- Missing documentation (for complex logic)
- Overly clever code (future maintainers will suffer)

## The Most Important Thing

The point of code review isn't to catch every tiny issue. It's to:
1. **Prevent serious bugs and security issues**
2. **Share knowledge** (reviewer learns, author learns)
3. **Maintain code quality** over time

I focus on high-impact issues first. If there's a SQL injection vulnerability, I don't waste time on variable naming.

## Resources

- [OWASP Code Review Guide](https://owasp.org/www-project-code-review-guide/)
- [Application Security Code Review Guide 2025 (OWASP Checklist)](https://getfailsafe.com/application-security-code-review-the-ultimate-2025-guide/)
- [Secure Code Review Best Practices - DevCom](https://devcom.com/tech-blog/secure-code-review-best-practices-to-protect-your-applications/)
- [Code Review Best Practices - Wiz](https://www.wiz.io/academy/code-review-best-practices)
- [Security Code Review Checklist - Redwerk](https://redwerk.com/blog/security-code-review-checklist/)

---

*Questions about code reviews? [Reach out](mailto:jordan@jordananderson.us).*
