+++
title = "Debugging Production Issues: A Systematic Approach"
date = 2024-12-11
description = "How to efficiently diagnose and fix issues in production systems."
[taxonomies]
tags = ["debugging", "observability", "production"]
+++

Production is down. Alerts are firing. Management is asking questions. Here's how to stay calm and fix it fast.

<!-- more -->

## The OODA Loop for Incidents

Military decision-making adapted for debugging:

```
┌─────────────────────────────────────────────────┐
│                                                 │
│    Observe → Orient → Decide → Act → Repeat    │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Step 1: Observe

Gather data before making changes:

```bash
# Quick system health check
top -l 1 | head -10
df -h
free -m

# Recent logs
journalctl -u your-service --since "10 minutes ago"

# Network connections
netstat -an | grep ESTABLISHED | wc -l

# Process state
ps aux | grep your-service
```

**Key metrics to check:**
- CPU/Memory/Disk usage
- Request latency (p50, p95, p99)
- Error rates
- Queue depths
- Connection pools

## Step 2: Orient

Ask the right questions:

1. **What changed?** Recent deployments? Config changes?
2. **When did it start?** Correlate with events
3. **What's the blast radius?** All users? Some users? One region?
4. **Is it getting worse?** Stable? Recovering?

```sql
-- Example: Check if errors correlate with specific users
SELECT user_id, COUNT(*) as error_count
FROM error_logs
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY user_id
ORDER BY error_count DESC
LIMIT 10;
```

## Step 3: Decide

Form a hypothesis and choose an action:

```
Hypothesis: Database connection pool is exhausted

Evidence:
- Errors: "connection timeout"
- DB metrics: pool utilization at 100%
- Recent change: New feature with N+1 queries

Action options:
1. Rollback the deployment (safest)
2. Increase pool size (quick fix)
3. Fix the N+1 query (root cause)
```

## Step 4: Act

Make ONE change at a time:

```bash
# Rollback example
kubectl rollout undo deployment/api-server

# Or scale up
kubectl scale deployment/api-server --replicas=10

# Watch the effect
watch -n 5 'curl -s http://localhost/health | jq .status'
```

## The 5 Whys

Dig to the root cause:

```
Problem: API returning 500 errors

Why? → Database queries timing out
Why? → Connection pool exhausted  
Why? → Queries holding connections too long
Why? → N+1 query pattern in new feature
Why? → Missing eager loading in ORM

Root cause: ORM configuration issue
```

## Common Patterns and Quick Fixes

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Memory climbing | Memory leak | Restart pods |
| Latency spike | GC pressure | Increase heap |
| Connection errors | Pool exhaustion | Increase pool size |
| CPU 100% | Infinite loop | Kill and debug |
| Disk full | Log files | Rotate logs |

## Post-Incident: Write It Down

Every incident should produce:

```markdown
## Incident Report: API Outage 2024-12-11

**Duration**: 14:32 - 15:17 (45 minutes)
**Impact**: 30% of API requests failed
**Root Cause**: N+1 query in new feature

### Timeline
- 14:32: Alerts fire for high error rate
- 14:45: Identified connection pool exhaustion
- 15:00: Rolled back deployment
- 15:17: Error rate returned to normal

### Action Items
- [ ] Add eager loading to ORM queries
- [ ] Add connection pool monitoring
- [ ] Load test new features before deploy
```

## Key Takeaways

1. **Observe first** - Don't change things until you understand
2. **One change at a time** - Otherwise you won't know what worked
3. **Rollback is always an option** - Sometimes the fastest fix
4. **Automate the boring stuff** - Runbooks for common issues
5. **Blameless postmortems** - Focus on systems, not people

The best debugging skill is staying calm under pressure.
