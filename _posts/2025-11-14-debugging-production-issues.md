---
layout: default
title:  "Debugging Production Issues: A Methodical Approach"
date:   2025-11-14 22:00:00
categories: Development Debugging Operations
---

The worst feeling in software development: "Production is down." Your adrenaline spikes. Users are complaining. Your phone won't stop ringing.

I've debugged hundreds of production issues. The difference between panic and resolution is having a systematic approach. Here's mine.

## The Golden Rules

Before anything else, remember:

1. **Don't make it worse** - Every change in production carries risk
2. **Gather data first** - Don't guess, measure
3. **Fix the symptom, then the cause** - Get users back online, then investigate
4. **Document everything** - Future you will thank present you

## Step 1: Assess the Situation

### Is It Actually Down?

```bash
# Check if service responds
curl -I https://yourapp.com

# Check response time
curl -w "@curl-format.txt" -o /dev/null -s https://yourapp.com

# curl-format.txt:
# time_namelookup:  %{time_namelookup}\n
# time_connect:  %{time_connect}\n
# time_starttransfer:  %{time_starttransfer}\n
# time_total:  %{time_total}\n
```

### What's the Scope?

- **All users or specific users?**
- **All features or specific features?**
- **All regions or specific regions?**
- **Consistent or intermittent?**

**Quick check:**
```bash
# Check from multiple locations
curl https://yourapp.com  # From server
curl https://yourapp.com  # From your laptop
# Use online tools: DownDetector, StatusPage

# Check error rate
grep "Error" /var/log/app.log | wc -l
```

### When Did It Start?

```bash
# Check recent deployments
git log --since="2 hours ago" --oneline

# Check recent config changes
ls -lt /etc/app/ | head

# Check system changes
last | head
```

## Step 2: Check the Obvious First

### Is the Server Actually Running?

```bash
# Check process
ps aux | grep node
ps aux | grep java
ps aux | grep python

# Check if listening on port
netstat -tlnp | grep 3000
lsof -i :3000

# Check CPU/Memory
top
htop
free -h

# Check disk space (this kills apps frequently)
df -h
```

### Check Recent Logs

```bash
# Last 100 lines
tail -100 /var/log/app.log

# Follow logs in real-time
tail -f /var/log/app.log

# Filter for errors
grep -i "error\|exception\|fatal" /var/log/app.log | tail -50

# Check system logs
journalctl -u myapp.service -n 100

# Application-specific
docker logs container_name --tail 100
kubectl logs pod-name --tail=100
```

### Check Database Connectivity

```bash
# Can app reach database?
telnet db.example.com 5432
nc -zv db.example.com 5432

# Check database server
# PostgreSQL
psql -h db.example.com -U dbuser -d mydb -c "SELECT 1"

# MySQL
mysql -h db.example.com -u dbuser -p mydb -e "SELECT 1"

# Check connection pool
# Look for "too many connections" errors in logs
```

### Check External Dependencies

```bash
# API dependencies
curl https://api.thirdparty.com/health

# Check DNS
dig api.thirdparty.com
nslookup api.thirdparty.com

# Check if firewall blocking
telnet api.thirdparty.com 443
```

## Step 3: Gather Diagnostic Data

### Application Metrics

```javascript
// If you have APM tools (New Relic, DataDog, AppDynamics)
// Check:
// - Error rate
// - Response time (p50, p95, p99)
// - Throughput (requests per minute)
// - Database query time
// - External API call time

// Look for:
// - Sudden spikes in error rate
// - Increase in response time
// - Changes in traffic pattern
```

### System Metrics

```bash
# CPU usage over time
sar -u 1 10

# Memory usage
free -h
cat /proc/meminfo

# Disk I/O
iostat -x 1 10

# Network traffic
iftop
nethogs

# Open file descriptors (can hit limits)
lsof | wc -l
ulimit -n
```

### Stack Traces

```bash
# Node.js: Get stack trace of running process
kill -USR1 $(pgrep node)
# Check /tmp for stack trace

# Java: Thread dump
jstack <pid> > thread_dump.txt

# Python: Print stack traces
kill -SIGUSR1 <pid>
# Or use py-spy:
py-spy dump --pid <pid>
```

## Step 4: Common Issues and Solutions

### Issue: High CPU Usage

**Diagnosis:**
```bash
# Find which process
top

# For Node.js, profile CPU
node --prof app.js
node --prof-process isolate-*.log > processed.txt
```

**Common causes:**
- Infinite loop
- Inefficient algorithm
- Regex catastrophic backtracking

**Quick fix:**
```bash
# Restart service (buys time)
systemctl restart myapp

# Scale horizontally if possible
# Add more instances
```

### Issue: High Memory Usage / Memory Leak

**Diagnosis:**
```bash
# Check memory usage
ps aux --sort=-%mem | head

# Node.js heap snapshot
node --inspect app.js
# Connect Chrome DevTools, take heap snapshot

# Check for swap usage (means memory exhausted)
free -h
```

**Common causes:**
- Event listeners not removed
- Global variables accumulating
- Cached data never cleared

**Quick fix:**
```bash
# Restart service
# But set up monitoring for recurrence
```

### Issue: Database Connection Pool Exhausted

**Symptoms in logs:**
```
Error: too many connections
Error: Connection timeout
Error: Pool is full
```

**Diagnosis:**
```sql
-- PostgreSQL: Check active connections
SELECT count(*) FROM pg_stat_activity;

-- MySQL: Check connections
SHOW PROCESSLIST;

-- Check for long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

**Fix:**
```javascript
// Ensure connections are released
pool.query('SELECT * FROM users', (err, result) => {
    // Do something
    client.release();  // MUST call this!
});

// Or use async/await with try/finally
const client = await pool.connect();
try {
    const result = await client.query('SELECT * FROM users');
    return result;
} finally {
    client.release();  // Always release
}
```

### Issue: Disk Space Full

**Symptoms:**
```
Error: ENOSPC: no space left on device
Cannot write to file
```

**Diagnosis:**
```bash
df -h  # Check disk usage
du -sh /* | sort -h  # Find large directories
du -sh /var/log  # Logs often culprit
```

**Fix:**
```bash
# Clear old logs
find /var/log -name "*.log" -mtime +7 -delete

# Compress old logs
find /var/log -name "*.log" -mtime +1 -exec gzip {} \;

# Set up log rotation
# Edit /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
}
```

### Issue: Slow Database Queries

**Diagnosis:**
```bash
# Check slow query log
# MySQL:
tail /var/log/mysql/mysql-slow.log

# PostgreSQL:
grep "duration:" /var/log/postgresql/postgresql.log | sort -t: -k3 -n | tail
```

**Quick fix:**
```sql
-- Kill slow query (PostgreSQL)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '5 minutes';

-- Add missing index (if identified)
CREATE INDEX idx_users_email ON users(email);
```

### Issue: External API Timeout

**Symptoms:**
```
Error: connect ETIMEDOUT
Error: socket hang up
```

**Diagnosis:**
```bash
# Check if API is reachable
curl -v https://api.thirdparty.com

# Check response time
time curl https://api.thirdparty.com/endpoint
```

**Quick fix:**
```javascript
// Implement circuit breaker
const CircuitBreaker = require('opossum');

const options = {
    timeout: 3000,  // 3 second timeout
    errorThresholdPercentage: 50,
    resetTimeout: 30000  // Try again after 30 seconds
};

const breaker = new CircuitBreaker(fetchFromAPI, options);

breaker.fallback(() => {
    // Return cached data or default value
    return getCachedData();
});
```

## Step 5: Roll Back or Roll Forward?

### When to Roll Back

- Clear cause: recent deployment introduced bug
- Quick to revert (< 5 minutes)
- No data migration issues

```bash
# Git rollback
git revert HEAD
git push origin main

# Or redeploy previous version
git checkout previous-tag
./deploy.sh

# Docker: Use previous image
docker pull myapp:previous-tag
docker stop myapp-container
docker run myapp:previous-tag
```

### When to Roll Forward

- Fix is simple and quick
- Rolling back would cause data issues
- Problem exists in multiple versions

```bash
# Make fix
git add fix.js
git commit -m "Hotfix: Fix null pointer in user profile"
git push origin main

# Deploy immediately
./deploy.sh
```

## Step 6: Communicate

### Update Status Page

```markdown
**Investigating** - We're aware of elevated error rates and investigating
↓ (15 min later)
**Identified** - Database connection pool exhaustion caused by deployment
↓ (10 min later)
**Monitoring** - Fix deployed, monitoring for stability
↓ (30 min later)
**Resolved** - Issue resolved. Root cause: unclosed database connections
```

### Notify Team

```
Slack/Teams message:
🔴 Production issue: 50% error rate
Status: Investigating
Impact: User login failing
ETA: Investigating, updates in 10 min
Lead: @yourname

(Update every 10-15 minutes)
```

## Step 7: Post-Mortem

**Write it while it's fresh.** Template:

```markdown
# Incident Post-Mortem: [Date]

## Summary
Brief description of what happened

## Timeline
- 14:00 - Deployment to production
- 14:15 - Error rate increased to 50%
- 14:20 - Issue identified: database connection pool exhausted
- 14:25 - Fix deployed
- 14:30 - Error rate returned to normal

## Root Cause
Detailed explanation of what caused the issue

## Impact
- Duration: 30 minutes
- Users affected: ~5,000
- Features impacted: Login, Profile pages

## Resolution
What fixed the issue

## Action Items
- [ ] Add connection pool monitoring
- [ ] Set up alerts for high error rates
- [ ] Code review: ensure all database connections released
- [ ] Add integration tests for this scenario

## Lessons Learned
What we learned and how to prevent this in the future
```

## Prevention: Build Better Observability

### Logging

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

// Log with context
logger.info('User logged in', {
    userId: user.id,
    ip: req.ip,
    timestamp: new Date()
});

// Log errors with stack traces
logger.error('Payment processing failed', {
    error: err.message,
    stack: err.stack,
    userId: user.id,
    amount: payment.amount
});
```

### Metrics

```javascript
const prometheus = require('prom-client');

// Request duration
const httpRequestDuration = new prometheus.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status_code']
});

// Error counter
const errorCounter = new prometheus.Counter({
    name: 'errors_total',
    help: 'Total number of errors',
    labelNames: ['type']
});

// Active connections
const activeConnections = new prometheus.Gauge({
    name: 'database_connections_active',
    help: 'Number of active database connections'
});
```

### Health Checks

```javascript
app.get('/health', async (req, res) => {
    const checks = {
        database: await checkDatabase(),
        redis: await checkRedis(),
        externalAPI: await checkExternalAPI()
    };

    const healthy = Object.values(checks).every(check => check.healthy);

    res.status(healthy ? 200 : 503).json({
        status: healthy ? 'healthy' : 'unhealthy',
        checks
    });
});

async function checkDatabase() {
    try {
        await db.query('SELECT 1');
        return { healthy: true };
    } catch (err) {
        return { healthy: false, error: err.message };
    }
}
```

### Alerts

```yaml
# Alert rules (Prometheus example)
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: rate(errors_total[5m]) > 10
        annotations:
          summary: "High error rate detected"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, http_request_duration_seconds) > 5
        annotations:
          summary: "95th percentile response time > 5s"

      - alert: DatabaseConnectionPoolHigh
        expr: database_connections_active > 90
        annotations:
          summary: "Database connection pool nearly exhausted"
```

## My Debug Toolkit

Essential tools I always have ready:

```bash
# System monitoring
htop
iostat
sar
netstat/ss
lsof

# Log analysis
grep
awk
jq  # For JSON logs

# Network debugging
curl
telnet
nc (netcat)
dig/nslookup
traceroute

# Process debugging
strace  # System call tracing
gdb     # Debugger
lsof    # Open files

# Application-specific
node --inspect  # Node.js debugger
jstack          # Java thread dump
py-spy          # Python profiler
```

## Final Thoughts

Production debugging is stressful, but having a systematic approach helps:

1. Stay calm
2. Gather data before acting
3. Fix the immediate issue first
4. Document everything
5. Learn and improve

The best production issue is the one that never happens. Invest in observability, monitoring, and alerting.

## Resources

- [Site Reliability Engineering Book (Google)](https://sre.google/books/)
- [Debugging: The 9 Indispensable Rules](https://debuggingrules.com/)
- [Production-Ready Microservices](https://www.oreilly.com/library/view/production-ready-microservices/9781491965962/)

---

*Questions about debugging production issues? [Reach out](mailto:jordan@jordananderson.us).*
