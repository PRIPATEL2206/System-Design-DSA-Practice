# Programming & Engineering Practices — Write Code Like a Senior Engineer

## Clean Code Principles

### The Standard: Your Code Should Read Like Prose

**Naming:**
```python
# BAD
def calc(d, r):
    return d * r * 0.01

# GOOD
def calculate_interest(deposit_amount, annual_rate):
    return deposit_amount * annual_rate * 0.01
```

**Functions:**
- Do ONE thing (Single Responsibility)
- Keep under 20 lines
- Max 3 parameters (use objects for more)
- Name describes what it DOES, not HOW

```python
# BAD — function does too many things
def process_user(user):
    validate_email(user.email)
    hash_password(user.password)
    save_to_db(user)
    send_welcome_email(user)
    update_analytics(user)

# GOOD — each function has single responsibility
def register_user(user):
    validated_user = validate_user_data(user)
    persisted_user = save_user(validated_user)
    trigger_post_registration(persisted_user)
```

**DRY (Don't Repeat Yourself):**
If you copy-paste code, extract it. But don't abstract too early — wait until you see the pattern 3 times.

**YAGNI (You Ain't Gonna Need It):**
Don't build features you MIGHT need later. Build what's needed NOW.

---

## API Design

### REST API Best Practices
```
Endpoints (nouns, not verbs):
GET    /api/v1/users          → List users
GET    /api/v1/users/123      → Get user 123
POST   /api/v1/users          → Create user
PUT    /api/v1/users/123      → Replace user 123
PATCH  /api/v1/users/123      → Update user 123 partially
DELETE /api/v1/users/123      → Delete user 123

Nested resources:
GET    /api/v1/users/123/orders        → User's orders
POST   /api/v1/users/123/orders        → Create order for user

Filtering, Sorting, Pagination:
GET /api/v1/users?status=active&sort=-created_at&page=2&limit=20
```

### API Response Format
```json
{
  "data": {
    "id": "123",
    "type": "user",
    "attributes": {
      "name": "Prince Patel",
      "email": "prince@example.com"
    }
  },
  "meta": {
    "total": 150,
    "page": 2,
    "per_page": 20
  }
}
```

### Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      {"field": "email", "message": "must not be empty"}
    ]
  }
}
```

### API Versioning Strategies
| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | /api/v1/users | Clear, easy to test | URL pollution |
| Header | Accept: application/vnd.api.v1+json | Clean URLs | Hidden, harder to test |
| Query param | /users?version=1 | Simple | Confusing |

**Best practice:** URL path versioning (/v1/) — most common at FAANG.

### Rate Limiting
```
Headers to return:
X-RateLimit-Limit: 100        (max requests per window)
X-RateLimit-Remaining: 45     (requests left)
X-RateLimit-Reset: 1625097600 (when window resets)

Return 429 Too Many Requests when exceeded
```

### Authentication
| Method | Use Case | How |
|--------|----------|-----|
| API Key | Server-to-server, simple | Key in header: `X-API-Key: abc123` |
| JWT | Stateless auth | Token in `Authorization: Bearer <token>` |
| OAuth 2.0 | Third-party access | Authorization code flow |
| Session cookie | Web apps | Server stores session, client sends cookie |

---

## Git — Beyond the Basics

### The Workflow That FAANG Uses
```
main (production)
  └── develop (integration)
       ├── feature/user-auth
       ├── feature/payment-flow
       └── bugfix/login-crash

Feature Branch Workflow:
1. Create branch from main: git checkout -b feature/my-feature
2. Make commits (small, focused)
3. Push and create PR
4. Code review + CI passes
5. Squash merge to main
```

### Essential Git Commands
```bash
# Daily workflow
git status
git add -p                    # Stage interactively (patch mode)
git commit -m "feat: add user authentication"
git push origin feature/auth

# Rebase (keep history clean)
git fetch origin
git rebase origin/main        # Replay your commits on top of latest main

# Undo mistakes
git stash                     # Temporarily save uncommitted changes
git stash pop                 # Restore stashed changes
git reset --soft HEAD~1       # Undo last commit (keep changes staged)
git revert <commit>           # Create new commit that undoes a previous one

# Investigate
git log --oneline --graph     # Visual branch history
git blame <file>              # Who changed each line
git bisect                    # Binary search for bug-introducing commit
```

### Commit Message Convention (Conventional Commits)
```
type(scope): description

Types: feat, fix, docs, style, refactor, test, chore
Examples:
- feat(auth): add OAuth2 Google login
- fix(payment): handle null card number gracefully
- refactor(api): extract validation middleware
```

---

## CI/CD Fundamentals

### What CI/CD Means
```
Continuous Integration (CI):
- Every push triggers: lint → test → build
- Catch bugs early, before merging

Continuous Delivery (CD):
- After CI passes: deploy to staging automatically
- Deploy to production with one click

Continuous Deployment:
- After CI passes: deploy to production automatically
- Requires high confidence in tests
```

### Typical CI/CD Pipeline
```
Push code → Trigger pipeline → 
  1. Lint (code style check)
  2. Unit tests
  3. Build (compile/bundle)
  4. Integration tests
  5. Security scan
  6. Deploy to staging
  7. Smoke tests on staging
  8. Deploy to production (manual approval or auto)
  9. Health check
  10. Rollback if health check fails
```

### Tools
| Category | Tools |
|----------|-------|
| CI/CD | GitHub Actions, Jenkins, GitLab CI, CircleCI |
| Containerization | Docker |
| Orchestration | Kubernetes |
| IaC | Terraform, CloudFormation |
| Monitoring | Prometheus + Grafana, Datadog, CloudWatch |

---

## Debugging Mindset

### The Scientific Method for Debugging
```
1. REPRODUCE: Can you reliably trigger the bug?
2. ISOLATE: What's the smallest input that causes it?
3. HYPOTHESIZE: What could cause this behavior?
4. TEST: Validate/invalidate your hypothesis
5. FIX: Make the minimal change
6. VERIFY: Does the fix work? Any regressions?
```

### Debugging Strategies
| Strategy | When to Use |
|----------|-------------|
| Print/Log debugging | Quick investigation, production issues |
| Debugger (breakpoints) | Complex control flow, state inspection |
| Binary search (git bisect) | "It worked last week" — find the breaking commit |
| Rubber duck | Explain the problem out loud (often reveals the answer) |
| Read the error message | Seriously — read it carefully. Most errors tell you exactly what's wrong. |
| Check the docs | Before assuming a bug, verify you're using the API correctly |

### Production Debugging
```
1. Check logs (structured logging with request IDs)
2. Check metrics (did error rate spike? CPU/memory?)
3. Check recent deployments (correlation != causation, but often is)
4. Check external dependencies (is the DB slow? Is S3 having issues?)
5. Reproduce locally if possible
```

---

## Code Review — What Senior Engineers Look For

### The Review Checklist
1. **Correctness:** Does it actually solve the problem?
2. **Edge cases:** Empty input, null, overflow, concurrent access?
3. **Performance:** Any N+1 queries? Unnecessary O(n²)?
4. **Security:** SQL injection? XSS? Hardcoded secrets?
5. **Readability:** Can I understand this in 30 seconds?
6. **Testing:** Are critical paths tested?
7. **Architecture:** Does this belong here? Is it in the right layer?

### Writing Good PRs
```
Title: feat(payments): add Stripe webhook handler

## What
Handle Stripe payment webhooks for subscription events.

## Why
Users weren't getting updated subscription status after payment.

## How
- Added webhook endpoint at /api/webhooks/stripe
- Verify signature before processing
- Handle: payment_succeeded, payment_failed, subscription_cancelled
- Idempotent processing (store event_id)

## Testing
- Unit tests for each event type
- Integration test with Stripe test mode
- Manually tested with Stripe CLI webhook forwarding

## Screenshots (if UI change)
```

---

## Key Takeaways

1. Clean code is not about being clever — it's about being clear
2. APIs are contracts — design them carefully, version them, document them
3. Git is a first-class skill — messy git history signals messy thinking
4. CI/CD is table stakes — if your code can't be deployed confidently, it's not production-ready
5. Debugging is hypothesis testing — be systematic, not random
