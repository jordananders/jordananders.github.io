---
layout: default
title:  "Regular Expressions for Common Scenarios"
date:   2025-11-14 19:00:00
categories: Development Regex Programming
---

I used to Google "regex for email validation" every single time. Not anymore. This is my collection of regex patterns I actually use, with explanations that make sense.

## Email Validation

### Basic Email (Good Enough for Most Cases)

```regex
^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
```

**Matches:**
- user@example.com
- john.doe+tag@company.co.uk
- name_123@sub.domain.org

**Doesn't match:**
- @example.com (no username)
- user@.com (invalid domain)
- user@domain (no TLD)

**JavaScript:**
```javascript
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
emailRegex.test('user@example.com'); // true
```

**Note:** Perfect email validation is impossible with regex. Use this for basic validation, then send a confirmation email.

## Phone Numbers

### US Phone Number

```regex
^\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})$
```

**Matches:**
- (555) 123-4567
- 555-123-4567
- 555.123.4567
- 5551234567

**JavaScript with formatting:**
```javascript
const phone = "(555) 123-4567";
const match = phone.match(/^\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})$/);
if (match) {
    const formatted = `(${match[1]}) ${match[2]}-${match[3]}`;
    console.log(formatted); // (555) 123-4567
}
```

### International Phone (E.164 Format)

```regex
^\+[1-9]\d{1,14}$
```

**Matches:**
- +14155552671
- +442071234567
- +919876543210

## URLs

### Basic URL

```regex
^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)$
```

**Matches:**
- https://example.com
- http://www.example.com/path?query=value
- https://sub.domain.example.com:8080/path

**JavaScript:**
```javascript
const urlRegex = /^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)$/;
urlRegex.test('https://example.com'); // true
```

### Extract Domain from URL

```regex
^https?:\/\/(?:www\.)?([^\/]+)
```

**JavaScript:**
```javascript
const url = "https://www.example.com/path";
const match = url.match(/^https?:\/\/(?:www\.)?([^\/]+)/);
console.log(match[1]); // example.com
```

## Dates and Times

### Date (MM/DD/YYYY or MM-DD-YYYY)

```regex
^(0[1-9]|1[0-2])[\/\-](0[1-9]|[12][0-9]|3[01])[\/\-](19|20)\d{2}$
```

**Matches:**
- 12/31/2025
- 01-15-2024
- 06/30/2023

### Date (YYYY-MM-DD) (ISO 8601)

```regex
^(19|20)\d{2}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$
```

**Matches:**
- 2025-12-31
- 2024-01-15

### Time (24-hour HH:MM)

```regex
^([01][0-9]|2[0-3]):[0-5][0-9]$
```

**Matches:**
- 00:00
- 23:59
- 14:30

### Time (12-hour with AM/PM)

```regex
^(0?[1-9]|1[0-2]):[0-5][0-9]\s?(AM|PM|am|pm)$
```

**Matches:**
- 12:00 PM
- 9:30 AM
- 03:45 pm

## Numbers

### Integer (Positive or Negative)

```regex
^-?\d+$
```

**Matches:**
- 123
- -456
- 0

### Decimal Number

```regex
^-?\d+(\.\d+)?$
```

**Matches:**
- 123
- 123.45
- -67.89
- 0.5

### Positive Integer Only

```regex
^[1-9]\d*$
```

**Matches:**
- 1
- 123
- 9999

**Doesn't match:**
- 0
- -5

### Price/Currency (Two Decimal Places)

```regex
^\$?\d+(\.\d{2})?$
```

**Matches:**
- 19.99
- $19.99
- 100
- $100.00

## Passwords

### Strong Password

```regex
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$
```

**Requirements:**
- At least 8 characters
- At least one lowercase letter
- At least one uppercase letter
- At least one digit
- At least one special character

**JavaScript validation:**
```javascript
const passwordRegex = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;

function validatePassword(password) {
    if (!passwordRegex.test(password)) {
        return "Password must contain: 8+ characters, uppercase, lowercase, number, special character";
    }
    return "Valid";
}
```

## Username/Slug

### Username (Letters, Numbers, Underscore, Hyphen)

```regex
^[a-zA-Z0-9_-]{3,16}$
```

**Matches:**
- john_doe
- user123
- test-user

**Requirements:**
- 3-16 characters
- Letters, numbers, underscore, hyphen only

### URL Slug

```regex
^[a-z0-9]+(?:-[a-z0-9]+)*$
```

**Matches:**
- my-blog-post
- product-name-123
- hello-world

## IP Addresses

### IPv4 Address

```regex
^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$
```

**Matches:**
- 192.168.1.1
- 10.0.0.0
- 255.255.255.255

### IPv6 Address (Simplified)

```regex
^(([0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}|::1)$
```

**Matches:**
- 2001:0db8:85a3:0000:0000:8a2e:0370:7334
- ::1

## HTML and Code

### HTML Tags

```regex
<([a-z]+)([^<]+)*(?:>(.*)<\/\1>|\s+\/>)
```

**Matches:**
- `<div>content</div>`
- `<span class="test">text</span>`
- `<br />`

### Hex Color Code

```regex
^#?([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$
```

**Matches:**
- #FFF
- #FFFFFF
- #1a2b3c
- FFF

### Remove HTML Tags

```javascript
const html = "<p>Hello <strong>World</strong></p>";
const text = html.replace(/<[^>]*>/g, '');
console.log(text); // Hello World
```

## File Paths and Extensions

### File Extension

```regex
\.(jpg|jpeg|png|gif|pdf|doc|docx)$
```

**JavaScript:**
```javascript
const filename = "document.pdf";
const hasValidExtension = /\.(jpg|jpeg|png|gif|pdf|doc|docx)$/i.test(filename);
```

### Windows File Path

```regex
^[a-zA-Z]:\\(?:[^\\/:*?"<>|\r\n]+\\)*[^\\/:*?"<>|\r\n]*$
```

**Matches:**
- C:\Users\John\Documents\file.txt
- D:\Projects\app\

### Unix File Path

```regex
^\/(?:[^\/\0]+\/)*[^\/\0]+$
```

**Matches:**
- /home/user/file.txt
- /var/log/app.log

## Text Extraction

### Extract Hashtags

```regex
#\w+
```

**JavaScript:**
```javascript
const text = "Check out #coding and #javascript";
const hashtags = text.match(/#\w+/g);
console.log(hashtags); // ['#coding', '#javascript']
```

### Extract Mentions

```regex
@\w+
```

### Extract Numbers from Text

```regex
\d+\.?\d*
```

**JavaScript:**
```javascript
const text = "The price is $19.99 for 3 items";
const numbers = text.match(/\d+\.?\d*/g);
console.log(numbers); // ['19.99', '3']
```

## Validation Patterns

### Credit Card (Simple)

```regex
^\d{4}-?\d{4}-?\d{4}-?\d{4}$
```

**Matches:**
- 1234-5678-9012-3456
- 1234567890123456

### SSN (US Social Security Number)

```regex
^\d{3}-\d{2}-\d{4}$
```

**Matches:**
- 123-45-6789

### Zip Code (US)

```regex
^\d{5}(-\d{4})?$
```

**Matches:**
- 12345
- 12345-6789

## Common Replace Operations

### Remove Extra Whitespace

```javascript
const text = "Hello    World";
const cleaned = text.replace(/\s+/g, ' ');
console.log(cleaned); // "Hello World"
```

### Trim Leading/Trailing Whitespace

```javascript
const text = "  hello  ";
const trimmed = text.replace(/^\s+|\s+$/g, '');
console.log(trimmed); // "hello"
```

### Remove Non-Alphanumeric Characters

```javascript
const text = "Hello, World! @2025";
const clean = text.replace(/[^a-zA-Z0-9\s]/g, '');
console.log(clean); // "Hello World 2025"
```

### Convert CamelCase to Kebab-Case

```javascript
const camel = "myVariableName";
const kebab = camel.replace(/([a-z])([A-Z])/g, '$1-$2').toLowerCase();
console.log(kebab); // "my-variable-name"
```

### Convert Snake_Case to CamelCase

```javascript
const snake = "my_variable_name";
const camel = snake.replace(/_([a-z])/g, (match, letter) => letter.toUpperCase());
console.log(camel); // "myVariableName"
```

## Lookahead and Lookbehind

### Positive Lookahead

```regex
\d+(?=px)
```

**Matches numbers followed by "px":**
- "100" in "100px"
- "50" in "50px"

**JavaScript:**
```javascript
const text = "width: 100px, height: 50px";
const numbers = text.match(/\d+(?=px)/g);
console.log(numbers); // ['100', '50']
```

### Negative Lookahead

```regex
\d+(?!px)
```

**Matches numbers NOT followed by "px":**
- "100" in "100em"
- "50" in "50 items"

### Positive Lookbehind

```regex
(?<=\$)\d+
```

**Matches numbers preceded by "$":**
- "100" in "$100"

### Negative Lookbehind

```regex
(?<!\$)\d+
```

**Matches numbers NOT preceded by "$":**
- "100" in "100 items"

## Common Flags

### JavaScript Regex Flags

```javascript
// i - case insensitive
/hello/i.test("HELLO"); // true

// g - global (find all matches)
"hello hello".match(/hello/g); // ['hello', 'hello']

// m - multiline
/^hello/m.test("world\nhello"); // true

// s - dotall (. matches newline)
/hello.world/s.test("hello\nworld"); // true

// Combined flags
const regex = /pattern/gim;
```

## Useful Regex Patterns

### Remove Leading Zeros

```javascript
const number = "00123";
const clean = number.replace(/^0+/, '');
console.log(clean); // "123"
```

### Validate GitHub Username

```regex
^[a-z\d](?:[a-z\d]|-(?=[a-z\d])){0,38}$
```

### Extract YouTube Video ID

```regex
(?:youtube\.com\/(?:[^\/\n\s]+\/\S+\/|(?:v|e(?:mbed)?)\/|\S*?[?&]v=)|youtu\.be\/)([a-zA-Z0-9_-]{11})
```

### Match Balanced Parentheses (Simple)

```regex
\(([^()]*)\)
```

### Extract JSON Value

```javascript
const json = '{"name": "John", "age": 30}';
const nameMatch = json.match(/"name":\s*"([^"]*)"/);
console.log(nameMatch[1]); // "John"
```

## Testing Your Regex

### Online Tools

- **regex101.com** - My favorite, shows explanation and matches
- **regexr.com** - Visual tool with reference
- **regexpal.com** - Simple tester

### JavaScript Console Testing

```javascript
// Test if pattern matches
/pattern/.test("string"); // true or false

// Extract matches
"string".match(/pattern/); // array of matches

// Replace
"string".replace(/pattern/, "replacement");

// Check what matched
const match = "test123".match(/(\w+)(\d+)/);
console.log(match[0]); // Full match: "test123"
console.log(match[1]); // First group: "test"
console.log(match[2]); // Second group: "123"
```

## Common Gotchas

### Escape Special Characters

```javascript
// BAD: . matches any character
const regex = /example.com/;
regex.test("exampleXcom"); // true (oops!)

// GOOD: Escape the dot
const regex = /example\.com/;
regex.test("exampleXcom"); // false
```

**Characters that need escaping:** `. * + ? ^ $ { } ( ) | [ ] \ /`

### Greedy vs Non-Greedy

```javascript
const html = "<div>Hello</div><div>World</div>";

// Greedy (default): matches as much as possible
html.match(/<div>.*<\/div>/)[0]; // "<div>Hello</div><div>World</div>"

// Non-greedy: add ? to match as little as possible
html.match(/<div>.*?<\/div>/g); // ["<div>Hello</div>", "<div>World</div>"]
```

### Anchors Matter

```javascript
// Without anchors: matches anywhere in string
/\d{3}/.test("abc123def"); // true

// With anchors: must match entire string
/^\d{3}$/.test("abc123def"); // false
/^\d{3}$/.test("123"); // true
```

## Resources

- [Regex101](https://regex101.com/) - Best regex tester
- [RegExr](https://regexr.com/) - Visual regex tool
- [MDN Regex Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions)

---

*Got a regex pattern to add? [Let me know](mailto:jordan@jordananderson.us).*
