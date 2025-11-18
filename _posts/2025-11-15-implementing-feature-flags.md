---
layout: default
title:  "Implementing Feature Flags: A Practical Guide"
date:   2025-11-15 02:00:00
categories: Development DevOps Deployment
---

Feature flags changed how I deploy code. Instead of deploying and hoping, I deploy features hidden behind flags, then gradually enable them. Instant rollback. Targeted releases. A/B testing. All without redeploying.

Here's how to implement feature flags effectively.

## What Are Feature Flags?

Feature flags (or feature toggles) are conditional statements that control feature visibility:

```javascript
if (featureFlags.isEnabled('new-checkout')) {
    return <NewCheckout />;
}
return <OldCheckout />;
```

Deploy code to production. Enable it for 1% of users. Watch for errors. Gradually roll out. If problems arise, disable instantly.

## Types of Feature Flags

### Release Flags

Control when features go live:

```javascript
if (flags.isEnabled('new-dashboard')) {
    return <NewDashboard />;
}
```

**Lifecycle:** Short-term. Remove once feature is fully released.

### Experiment Flags (A/B Tests)

Test feature variants:

```javascript
const variant = flags.getVariant('checkout-button-color');
// Returns: 'blue', 'green', or 'red'
return <Button color={variant} />;
```

**Lifecycle:** Medium-term. Remove after experiment concludes.

### Ops Flags

Control operational aspects:

```javascript
if (flags.isEnabled('enable-cache')) {
    return getCachedData();
}
return fetchFreshData();
```

**Lifecycle:** Long-term or permanent.

### Permission Flags

Control access by user attributes:

```javascript
if (flags.isEnabled('premium-features', { plan: user.plan })) {
    return <PremiumDashboard />;
}
```

## Simple Implementation

### Basic In-Memory Flags

```javascript
// flags.js
const flags = {
    'new-checkout': true,
    'dark-mode': false,
    'premium-features': true
};

function isEnabled(flagName) {
    return flags[flagName] || false;
}

module.exports = { isEnabled };
```

### JSON Configuration

```javascript
// flags.json
{
    "new-checkout": {
        "enabled": true,
        "percentage": 100
    },
    "new-dashboard": {
        "enabled": true,
        "percentage": 10,
        "users": ["user123", "user456"]
    }
}
```

```javascript
const flags = require('./flags.json');

function isEnabled(flagName, userId) {
    const flag = flags[flagName];
    if (!flag || !flag.enabled) return false;

    // Check if user is in allowlist
    if (flag.users && flag.users.includes(userId)) {
        return true;
    }

    // Check percentage rollout
    if (flag.percentage) {
        const hash = hashUserId(userId);
        return hash < flag.percentage;
    }

    return true;
}

function hashUserId(userId) {
    // Simple consistent hash (0-100)
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
        hash = ((hash << 5) - hash) + userId.charCodeAt(i);
        hash = hash & hash;
    }
    return Math.abs(hash) % 100;
}
```

### Database-Backed Flags

```javascript
// Store flags in database for dynamic updates
const flagsTable = {
    name: 'feature_flags',
    columns: {
        id: 'PRIMARY KEY',
        name: 'VARCHAR(255) UNIQUE',
        enabled: 'BOOLEAN',
        percentage: 'INT',
        rules: 'JSON'
    }
};

async function isEnabled(flagName, context = {}) {
    const flag = await db.query(
        'SELECT * FROM feature_flags WHERE name = ?',
        [flagName]
    );

    if (!flag || !flag.enabled) return false;

    // Evaluate rules
    if (flag.rules) {
        return evaluateRules(flag.rules, context);
    }

    // Percentage rollout
    if (flag.percentage < 100) {
        const hash = hashString(context.userId || '');
        return hash < flag.percentage;
    }

    return true;
}
```

## React Implementation

```javascript
// FeatureFlagProvider.js
import React, { createContext, useContext } from 'react';

const FeatureFlagContext = createContext({});

export function FeatureFlagProvider({ children, flags }) {
    return (
        <FeatureFlagContext.Provider value={flags}>
            {children}
        </FeatureFlagContext.Provider>
    );
}

export function useFeatureFlag(flagName) {
    const flags = useContext(FeatureFlagContext);
    return flags[flagName] || false;
}

// Usage
function App() {
    const flags = {
        'new-checkout': true,
        'dark-mode': false
    };

    return (
        <FeatureFlagProvider flags={flags}>
            <MyApp />
        </FeatureFlagProvider>
    );
}

function CheckoutPage() {
    const showNewCheckout = useFeatureFlag('new-checkout');

    return showNewCheckout ? <NewCheckout /> : <OldCheckout />;
}
```

## Best Practices

### 1. Keep Flags Short-Lived

Technical debt accumulates fast:

```javascript
// 3 months later...
if (flagA) {
    if (flagB) {
        if (!flagC) {
            // Impossible to understand
        }
    }
}
```

**Remove flags once features are fully released.**

### 2. Name Flags Clearly

```javascript
// BAD
'flag1', 'new-feature', 'test'

// GOOD
'checkout-v2', 'dark-mode-2025', 'premium-dashboard'
```

### 3. Log Flag Evaluations

```javascript
function isEnabled(flagName, context) {
    const result = evaluate(flagName, context);

    logger.info('Flag evaluated', {
        flag: flagName,
        result: result,
        userId: context.userId,
        timestamp: new Date()
    });

    return result;
}
```

### 4. Have a Kill Switch

For quick rollback:

```javascript
// Disable a feature instantly
await db.query(
    'UPDATE feature_flags SET enabled = false WHERE name = ?',
    ['problematic-feature']
);
```

### 5. Test Both Paths

```javascript
describe('Checkout', () => {
    it('renders old checkout when flag is off', () => {
        const { getByText } = render(
            <FeatureFlagProvider flags={{ 'new-checkout': false }}>
                <Checkout />
            </FeatureFlagProvider>
        );
        expect(getByText('Old Checkout')).toBeTruthy();
    });

    it('renders new checkout when flag is on', () => {
        const { getByText } = render(
            <FeatureFlagProvider flags={{ 'new-checkout': true }}>
                <Checkout />
            </FeatureFlagProvider>
        );
        expect(getByText('New Checkout')).toBeTruthy();
    });
});
```

## Rollout Strategies

### 1. Percentage Rollout

```javascript
// 10% → 25% → 50% → 100%
{
    "new-feature": {
        "enabled": true,
        "percentage": 10
    }
}
```

### 2. User Allowlist

```javascript
// Internal team first, then beta users
{
    "new-feature": {
        "enabled": true,
        "users": ["employee1", "employee2", "beta-user1"]
    }
}
```

### 3. Attribute-Based

```javascript
{
    "premium-features": {
        "enabled": true,
        "rules": [
            {"attribute": "plan", "operator": "equals", "value": "premium"}
        ]
    }
}
```

## Third-Party Services

For larger teams, consider:

- **LaunchDarkly** - Full-featured, expensive
- **Split.io** - Good A/B testing
- **Flagsmith** - Open source option
- **Unleash** - Self-hosted open source

These provide:
- Dashboard for non-developers
- Audit logging
- Analytics
- Segments/targeting
- A/B testing built-in

## Common Pitfalls

### 1. Flag Explosion

Too many flags = unmaintainable code.

**Solution:** Set expiration dates. Review flags monthly.

### 2. Nested Flags

```javascript
// Nightmare to debug
if (flagA && !flagB && flagC) {
    // What combination enabled this?
}
```

**Solution:** Keep flag logic simple. One flag per feature.

### 3. Forgetting to Remove Flags

Old flags linger forever.

**Solution:** Create tickets to remove flags. Add to definition of done.

## Resources

- [Feature Toggles (Martin Fowler)](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly Docs](https://docs.launchdarkly.com/)
- [Unleash (Open Source)](https://www.getunleash.io/)
- [Flagsmith (Open Source)](https://www.flagsmith.com/)

---

*Questions about feature flags? [Let me know](mailto:jordan@jordananderson.us).*
