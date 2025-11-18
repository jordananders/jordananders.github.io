---
layout: default
title:  "Microservices: When They Make Sense (And When They Don't)"
date:   2025-11-15 01:00:00
categories: Architecture Microservices Development
---

I've built both monoliths and microservices. The honest truth: most teams should start with a monolith.

Microservices are powerful, but they come with significant operational overhead. Here's when each approach makes sense.

## The Real Trade-offs

### Monolith Advantages

- **Simpler to develop** - One codebase, one deployment
- **Easier to debug** - No distributed tracing needed
- **Faster** - Zero network latency between modules
- **Cheaper** - Less infrastructure to manage
- **Better for small teams** - Benefits only appear with >10 developers

### Microservices Advantages

- **Independent scaling** - Scale only what needs scaling
- **Technology diversity** - Use the best tool for each job
- **Independent deployment** - Deploy services separately
- **Team autonomy** - Teams own their services
- **Fault isolation** - One service failure doesn't take down everything

## When to Use a Monolith

### 1. Early-Stage Startups

You're figuring out your product. You need to move fast. Microservices slow you down when you're still learning what to build.

**Rule:** Use a monolith until you know your domain well enough to define service boundaries.

### 2. Small Teams (< 10 Developers)

Microservices require:
- DevOps expertise
- CI/CD pipelines for each service
- Distributed tracing
- Service mesh
- Container orchestration

If your team is small, this overhead kills productivity.

### 3. Simple Domains

If your application is CRUD-heavy with straightforward business logic, microservices add complexity without benefit.

### 4. Tight Budgets

Microservices cost more:
- More infrastructure
- More monitoring tools
- More DevOps time
- More inter-service communication

## When to Use Microservices

### 1. Large Teams

With 50+ developers, a monolith becomes a bottleneck. Teams step on each other. Deployments become scary. Microservices give teams autonomy.

### 2. Clear Domain Boundaries

If you can clearly identify bounded contexts:
- User Management
- Order Processing
- Payment Service
- Notification Service

Each can be its own service.

### 3. Different Scaling Requirements

One module needs 100 instances, another needs 1. Microservices let you scale independently.

### 4. Different Technology Needs

ML service in Python, real-time service in Go, main app in Node.js. Microservices allow polyglot architecture.

### 5. High Availability Requirements

One service can fail without taking down the whole system. Critical for 99.99% uptime requirements.

## The Middle Ground: Modular Monolith

In 2025, the **modular monolith** is a serious contender. You get:
- Single deployment (simple)
- Clear module boundaries (organized)
- Ability to extract services later (flexible)

```
my-app/
├── modules/
│   ├── users/
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   ├── orders/
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   └── payments/
│       ├── controllers/
│       ├── services/
│       └── models/
├── shared/
└── main.js
```

**Rule:** Modules can only communicate through defined interfaces. No reaching into another module's database.

## My Decision Framework

### Start with Monolith If:

- [ ] Team < 10 developers
- [ ] Still figuring out the product
- [ ] Simple business logic
- [ ] Tight budget
- [ ] Need to ship fast

### Consider Microservices If:

- [ ] Team > 10 developers (some say >50)
- [ ] Clear bounded contexts
- [ ] Different scaling needs per module
- [ ] Different tech requirements per module
- [ ] High availability requirements
- [ ] Strong DevOps practices already in place

## Migration Path

If you need to migrate from monolith to microservices:

### 1. Strangler Fig Pattern

Gradually replace parts of the monolith:

```
Request → Router → Monolith (old)
                 → New Service 1
                 → New Service 2
```

### 2. Start with Least Coupled Modules

Extract modules that have:
- Clear boundaries
- Minimal dependencies
- Independent data stores

### 3. Don't Big Bang

Extract one service at a time. Keep the monolith running until all services are stable.

## 2025 Reality Check

**Most teams don't need microservices.**

Studies show:
- 90% of microservices teams still batch deploy (negating key benefit)
- Microservices benefits appear only with >10 developers
- Modular monoliths provide similar benefits with less overhead

**Start with a well-structured monolith. Extract to microservices only when you feel the pain of the monolith.**

## Resources

- [Monolith vs Microservices in 2025 - Foojay](https://foojay.io/today/monolith-vs-microservices-2025/)
- [Monolithic vs Microservices: Differences, Pros, & Cons in 2025](https://www.superblocks.com/blog/monolithic-vs-microservices)
- [Microservices vs. Monolith: Choosing the Right Architecture](https://rubyroidlabs.com/blog/2025/04/microservices-vs-monolith/)
- [Monolithic vs Microservices Architecture for 2025 - Scalo](https://www.scalosoft.com/blog/monolithic-vs-microservices-architecture-pros-and-cons-for-2025/)

---

*Questions about architecture decisions? [Let me know](mailto:jordan@jordananderson.us).*
