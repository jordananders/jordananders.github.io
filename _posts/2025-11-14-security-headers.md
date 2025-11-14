---
layout: default
title:  "Security Headers Every Developer Should Implement"
date:   2025-11-14 13:00:00
categories: Security WebDevelopment
---

I once inherited a web application with zero security headers. Not one. The site was vulnerable to clickjacking, XSS attacks, MIME sniffing, and basically every OWASP vulnerability that could be mitigated with proper HTTP headers.

Adding security headers took about 30 minutes and immediately closed several attack vectors. Here's what every developer needs to know about HTTP security headers in 2025.

## Why Security Headers Matter

Security headers are HTTP response headers that tell browsers how to behave when handling your site's content. They're your first line of defense against many common web vulnerabilities.

**The reality:** Most attacks don't require sophisticated zero-days. They exploit basic misconfigurations that security headers prevent.

With regulations like the UK Cyber Security and Resilience Bill (2025) expanding digital service security requirements, security headers are shifting from "nice to have" to "compliance requirement."

## The Essential Headers

Let me walk through the headers you should implement today, with examples you can copy and paste.

### 1. Content-Security-Policy (CSP)

**What it does:** Prevents XSS attacks by controlling which resources can load on your page.

**The problem it solves:** An attacker injects malicious JavaScript into your page (via XSS). Without CSP, the browser executes it. With CSP, the browser blocks unauthorized scripts.

**Basic implementation:**

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';
```

**What this means:**
- `default-src 'self'` - Only load resources from your own domain
- `script-src 'self'` - Only execute scripts from your domain
- `style-src 'self' 'unsafe-inline'` - Styles from your domain + inline styles (use sparingly)
- `img-src 'self' data:` - Images from your domain + data URIs
- `font-src 'self'` - Fonts from your domain
- `connect-src 'self'` - AJAX requests only to your domain
- `frame-ancestors 'none'` - Prevent your site from being framed (replaces X-Frame-Options)

**For sites using CDNs:**

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com;
```

**Strict CSP (recommended for 2025):**

Use nonces for scripts (generated server-side per request):

```
Content-Security-Policy: script-src 'nonce-{random-value}'; object-src 'none'; base-uri 'none';
```

In your HTML:
```html
<script nonce="{random-value}">
  // Your JavaScript
</script>
```

The nonce changes with each page load, making injected scripts impossible to execute.

**Testing CSP:**

Start with report-only mode to see what would break:

```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-violation-report
```

Browser sends violation reports to your endpoint without blocking anything. Fix issues, then enforce.

### 2. Strict-Transport-Security (HSTS)

**What it does:** Forces browsers to only use HTTPS for your site.

**The problem it solves:** User types `http://yoursite.com`, gets redirected to `https://` — but that first request is vulnerable to man-in-the-middle attacks. HSTS eliminates the HTTP request entirely.

**Implementation:**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**What this means:**
- `max-age=31536000` - Remember for 1 year (31536000 seconds)
- `includeSubDomains` - Apply to all subdomains
- `preload` - Include in browser HSTS preload lists

**Important:** Only enable HSTS after confirming your entire site works correctly over HTTPS, including all subdomains if using `includeSubDomains`.

**Preload list submission:**

Submit your domain to [hstspreload.org](https://hstspreload.org/) to be hardcoded into browsers. This protects users even on their first visit to your site.

### 3. X-Content-Type-Options

**What it does:** Prevents MIME type sniffing.

**The problem it solves:** Browser detects a file with extension `.txt` actually contains JavaScript, so it executes it. Attackers exploit this to bypass security controls.

**Implementation:**

```
X-Content-Type-Options: nosniff
```

That's it. One value: `nosniff`. Always use it.

### 4. X-Frame-Options (Legacy, but still relevant)

**What it does:** Prevents your site from being embedded in iframes (clickjacking protection).

**The problem it solves:** Attacker embeds your site in an invisible iframe, overlays fake UI, tricks users into clicking buttons they can't see.

**Implementation:**

```
X-Frame-Options: DENY
```

Or if you need to allow framing on same origin:

```
X-Frame-Options: SAMEORIGIN
```

**Modern alternative:** Use CSP's `frame-ancestors` directive instead:

```
Content-Security-Policy: frame-ancestors 'none';
```

**Why both?** CSP `frame-ancestors` is more flexible, but older browsers still use X-Frame-Options. Include both for maximum compatibility.

### 5. Referrer-Policy

**What it does:** Controls how much referrer information is sent with requests.

**The problem it solves:** Sensitive information in URLs (tokens, IDs) leaks to third-party sites via the Referer header.

**Implementation:**

```
Referrer-Policy: strict-origin-when-cross-origin
```

**Policy options:**
- `no-referrer` - Never send referrer (breaks analytics)
- `no-referrer-when-downgrade` - Send full URL unless HTTPS→HTTP (default)
- `same-origin` - Only send referrer for same-origin requests
- `strict-origin` - Only send origin, not full URL
- `strict-origin-when-cross-origin` - Full URL for same-origin, origin-only for cross-origin (recommended)

### 6. Permissions-Policy (formerly Feature-Policy)

**What it does:** Controls which browser features can be used.

**The problem it solves:** Third-party scripts accessing camera, microphone, geolocation without permission.

**Implementation:**

```
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(), usb=()
```

**To allow specific features:**

```
Permissions-Policy: geolocation=(self "https://maps.example.com"), camera=(self)
```

**Common features to disable:**
- `geolocation` - GPS access
- `microphone` - Mic access
- `camera` - Webcam access
- `payment` - Payment API
- `usb` - USB device access
- `autoplay` - Video/audio autoplay

### 7. X-Permitted-Cross-Domain-Policies

**What it does:** Restricts Adobe Flash and PDF cross-domain requests.

**Implementation:**

```
X-Permitted-Cross-Domain-Policies: none
```

Even though Flash is dead, PDF readers still respect this header.

## Implementing Security Headers

### Method 1: Web Server Configuration

**Nginx:**

```nginx
# /etc/nginx/conf.d/security-headers.conf
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

**Apache:**

```apache
# .htaccess or httpd.conf
Header set Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';"
Header set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
Header set X-Content-Type-Options "nosniff"
Header set X-Frame-Options "DENY"
Header set Referrer-Policy "strict-origin-when-cross-origin"
Header set Permissions-Policy "geolocation=(), microphone=(), camera=()"
```

**IIS:**

```xml
<!-- web.config -->
<system.webServer>
  <httpProtocol>
    <customHeaders>
      <add name="Content-Security-Policy" value="default-src 'self';" />
      <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains; preload" />
      <add name="X-Content-Type-Options" value="nosniff" />
      <add name="X-Frame-Options" value="DENY" />
      <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
      <add name="Permissions-Policy" value="geolocation=(), microphone=(), camera=()" />
    </customHeaders>
  </httpProtocol>
</system.webServer>
```

### Method 2: Application Code

**Node.js/Express:**

```javascript
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  strictTransportSecurity: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
}));
```

**ASP.NET Core:**

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy", "default-src 'self'");
    context.Response.Headers.Add("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload");
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

    await next();
});
```

**Python/Django:**

```python
# settings.py
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'",)
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
```

**PHP:**

```php
<?php
header("Content-Security-Policy: default-src 'self'; script-src 'self';");
header("Strict-Transport-Security: max-age=31536000; includeSubDomains; preload");
header("X-Content-Type-Options: nosniff");
header("X-Frame-Options: DENY");
header("Referrer-Policy: strict-origin-when-cross-origin");
?>
```

## Testing Your Headers

### Manual Testing

```bash
curl -I https://yoursite.com
```

Look for security headers in the response.

### Automated Tools

**1. SecurityHeaders.com**

Visit https://securityheaders.com/, enter your URL, get a grade (A+ is the goal).

**2. Mozilla Observatory**

https://observatory.mozilla.org/ — comprehensive security scan

**3. Chrome DevTools**

F12 → Network tab → Click any request → Headers tab

Look for security headers under "Response Headers."

### Validating CSP

**Browser console errors:**

Open DevTools → Console. CSP violations appear as errors:

```
Refused to load script from 'https://evil.com/script.js' because it violates the following Content Security Policy directive: "script-src 'self'"
```

**Report URI:**

Set up a report endpoint:

```
Content-Security-Policy: default-src 'self'; report-uri /csp-reports
```

Violations are POST'd to your endpoint as JSON:

```json
{
  "csp-report": {
    "blocked-uri": "https://evil.com/script.js",
    "violated-directive": "script-src",
    "original-policy": "default-src 'self';"
  }
}
```

## Common Mistakes

### Mistake 1: Breaking Your Site with CSP

**Problem:** Deploying strict CSP without testing

**Solution:** Use `Content-Security-Policy-Report-Only` first, monitor violations, fix issues, then enforce

### Mistake 2: HSTS Without Full HTTPS

**Problem:** Enabling HSTS when parts of your site don't work over HTTPS

**Result:** Those pages become inaccessible

**Solution:** Ensure 100% HTTPS coverage before enabling HSTS

### Mistake 3: Forgetting Subdomains

**Problem:** `includeSubDomains` flag breaks subdomains that aren't HTTPS-ready

**Solution:** Audit all subdomains first, or omit `includeSubDomains` initially

### Mistake 4: Too Permissive CSP

**Problem:** `script-src * 'unsafe-inline'` — this defeats the purpose

**Solution:** Whitelist specific domains only, avoid `unsafe-inline` if possible

### Mistake 5: Not Testing in All Browsers

**Problem:** CSP works in Chrome, breaks in Safari

**Solution:** Test in Chrome, Firefox, Safari, Edge

## My Security Headers Checklist

When deploying a new application, I use this checklist:

**Planning:**
- [ ] Inventory all external resources (CDNs, fonts, analytics)
- [ ] Determine if iframing is needed
- [ ] Identify features that need browser permissions

**Implementation:**
- [ ] Add Content-Security-Policy (start with report-only)
- [ ] Add Strict-Transport-Security (after verifying full HTTPS)
- [ ] Add X-Content-Type-Options
- [ ] Add X-Frame-Options
- [ ] Add Referrer-Policy
- [ ] Add Permissions-Policy

**Testing:**
- [ ] Test in Chrome DevTools
- [ ] Test in Firefox
- [ ] Test in Safari
- [ ] Check SecurityHeaders.com score
- [ ] Monitor CSP violation reports
- [ ] Verify mobile functionality

**Deployment:**
- [ ] Deploy to staging
- [ ] Monitor for 48 hours
- [ ] Switch CSP from report-only to enforce
- [ ] Deploy to production
- [ ] Monitor for 1 week

**Maintenance:**
- [ ] Review CSP violations monthly
- [ ] Update headers as new features are added
- [ ] Re-scan with security tools quarterly

## Real-World Example

Here's my standard configuration for a typical web application:

```nginx
# Modern web app with CDN, Google Fonts, and analytics

add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' https://cdn.jsdelivr.net https://www.google-analytics.com;
  style-src 'self' https://fonts.googleapis.com 'unsafe-inline';
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https://www.google-analytics.com;
  connect-src 'self' https://api.yourservice.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
" always;

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), payment=(), usb=()" always;
add_header X-Permitted-Cross-Domain-Policies "none" always;
```

This configuration:
- Prevents XSS attacks
- Forces HTTPS
- Prevents MIME sniffing
- Prevents clickjacking
- Minimizes information leakage
- Restricts browser features

## Measuring Impact

After implementing security headers on one of my projects:

**Before:**
- SecurityHeaders.com grade: F
- Mozilla Observatory: 25/100
- Vulnerable to: XSS, clickjacking, MIME sniffing

**After:**
- SecurityHeaders.com grade: A+
- Mozilla Observatory: 95/100
- Protected against common attack vectors

Implementation time: 30 minutes
Ongoing maintenance: ~5 minutes per deployment

## Staying Current

Security headers evolve. Stay informed:

- **OWASP Secure Headers Project**: https://owasp.org/www-project-secure-headers/
- **MDN Web Docs**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers
- **SecurityHeaders.com blog**: Updates on header best practices
- **Mozilla Observatory**: Tracks header adoption and recommendations

## Final Thoughts

Security headers are low-effort, high-impact security improvements. They won't stop a determined attacker, but they'll prevent opportunistic attacks and automated scanning tools from finding easy vulnerabilities.

In 2025, security headers aren't optional—they're expected. Regulators, auditors, and security-conscious customers all look for them.

Start with the basics (CSP, HSTS, X-Content-Type-Options), test thoroughly, and refine over time.

## Sources and Further Reading

- [OWASP HTTP Security Response Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Security Headers Quick Reference - web.dev](https://web.dev/articles/security-headers)
- [HTTP Security Headers - Invicti](https://www.invicti.com/white-papers/whitepaper-http-security-headers)
- [Content Security Policy (CSP) - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- [X-Frame-Options header - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Frame-Options)
- [Security Headers UK 2025 Compliance Guide - GuardianScan](https://guardianscan.ai/blog/website-security-headers-uk-2025)

---

*Questions about implementing security headers? [Let me know](mailto:jordan@jordananderson.us).*
