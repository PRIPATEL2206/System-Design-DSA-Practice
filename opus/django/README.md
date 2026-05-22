# Django Senior Developer & DevOps Guide

A comprehensive deep-dive into Django for senior developers and DevOps engineers — covering architecture, performance, security, deployment, and production patterns.

---

## Structure

| # | File | Focus Area |
|---|------|-----------|
| 1 | [01_orm_models_querysets.md](01_orm_models_querysets.md) | ORM internals, custom managers, QuerySet optimization, raw SQL |
| 2 | [02_rest_api_design.md](02_rest_api_design.md) | DRF, serializers, viewsets, pagination, throttling, versioning |
| 3 | [03_security_hardening.md](03_security_hardening.md) | CSRF, XSS, SQL injection, auth, permissions, OWASP top 10 |
| 4 | [04_performance_caching.md](04_performance_caching.md) | Query optimization, Redis caching, database indexing, profiling |
| 5 | [05_middleware_signals_async.md](05_middleware_signals_async.md) | Custom middleware, signals, async views, Celery, channels |
| 6 | [06_deployment_devops.md](06_deployment_devops.md) | Docker, Gunicorn, Nginx, CI/CD, AWS deployment, scaling |
| 7 | [07_testing_strategies.md](07_testing_strategies.md) | Unit/integration/E2E, factories, mocking, coverage, TDD |
| 8 | [08_database_migrations_patterns.md](08_database_migrations_patterns.md) | Zero-downtime migrations, multi-DB, read replicas, partitioning |
| 9 | [09_admin_customization.md](09_admin_customization.md) | Custom admin, inline models, actions, dashboard, security |
| 10 | [10_project_structure.md](10_project_structure.md) | Production layout, service layer, selectors, settings, Makefile |
| 11 | [11_aws_integration.md](11_aws_integration.md) | S3, SES, SQS, Lambda, CloudWatch, Secrets Manager, ElastiCache |
| 12 | [12_interview_coding_rounds.md](12_interview_coding_rounds.md) | Rate limiter, URL shortener, task queue, leaderboard, booking system |

---

## Who This Is For

- **Senior Developers**: Architecture decisions, performance tuning, code patterns
- **Senior DevOps Engineers**: Deployment, scaling, monitoring, infrastructure
- **Interview Prep**: Every topic maps to real interview questions at product companies

---

## Quick Reference

### Django Request Lifecycle
```
Client Request
    │
    ▼
Nginx (reverse proxy, static files, SSL)
    │
    ▼
Gunicorn (WSGI server, multiple workers)
    │
    ▼
Django Middleware Stack (top → bottom)
    │ SecurityMiddleware
    │ SessionMiddleware
    │ CommonMiddleware
    │ CsrfViewMiddleware
    │ AuthenticationMiddleware
    │ MessageMiddleware
    │
    ▼
URL Router (urls.py)
    │
    ▼
View (function or class-based)
    │
    ▼
ORM → Database
    │
    ▼
Template / Serializer (response)
    │
    ▼
Middleware Stack (bottom → top, response phase)
    │
    ▼
Gunicorn → Nginx → Client
```

### Production Stack
```
                    ┌─────────────────┐
                    │   CloudFront    │ (CDN for static/media)
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │     Nginx       │ (reverse proxy, SSL termination)
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Gunicorn      │ (WSGI, 2*CPU+1 workers)
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼──┐   ┌──────▼─────┐  ┌────▼──────┐
     │  Django   │   │   Redis    │  │ Celery    │
     │  App      │   │  (cache +  │  │ (async    │
     │           │   │   broker)  │  │  tasks)   │
     └────┬──────┘   └────────────┘  └───────────┘
          │
     ┌────▼──────┐
     │ PostgreSQL │
     │ (primary + │
     │  replica)  │
     └────────────┘
```
