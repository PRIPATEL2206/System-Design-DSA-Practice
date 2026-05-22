# Programming Paradigms — Think Like a Software Engineer

## Why Interviewers Care

You won't be directly asked "Explain OOP vs FP." But your **code in interviews reveals which paradigms you understand**. Writing clean, modular, well-structured code signals engineering maturity — exactly what Senior+ roles demand.

---

## The Four Major Paradigms

### 1. Imperative Programming
**What:** You tell the computer exactly WHAT to do, step by step.

```python
# Imperative: "Here's how to find the sum"
total = 0
for num in numbers:
    total += num
```

**When used:** Low-level algorithms, performance-critical code, interview coding.

### 2. Object-Oriented Programming (OOP)
**What:** Organize code around "objects" that bundle data + behavior.

**The 4 Pillars (Asked in every OOP interview):**

| Pillar | One-line Definition | Real Example |
|--------|-------------------|--------------|
| Encapsulation | Hide internal details, expose an interface | `BankAccount.withdraw()` hides balance validation |
| Abstraction | Model real-world entities simply | `Animal` class — don't expose DNA, expose `speak()` |
| Inheritance | Share behavior through parent-child hierarchy | `Dog(Animal)` gets `eat()` for free |
| Polymorphism | Same interface, different behavior | `shape.area()` works for Circle AND Rectangle |

```python
class PaymentProcessor:
    def process(self, amount):
        raise NotImplementedError

class StripeProcessor(PaymentProcessor):
    def process(self, amount):
        # Stripe-specific logic
        return stripe.charge(amount)

class RazorpayProcessor(PaymentProcessor):
    def process(self, amount):
        # Razorpay-specific logic
        return razorpay.capture(amount)

# Polymorphism: caller doesn't care which processor
def checkout(processor: PaymentProcessor, amount):
    return processor.process(amount)
```

**When used:** System design, LLD interviews, building maintainable systems.

### 3. Functional Programming (FP)
**What:** Treat computation as evaluating mathematical functions. No side effects, no mutation.

**Core Principles:**
- **Pure functions:** Same input → always same output, no side effects
- **Immutability:** Never modify data, create new copies
- **First-class functions:** Functions are values you can pass around
- **Composition:** Build complex behavior from simple functions

```python
# Functional approach
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# Instead of loops, use transformations
squared = list(map(lambda x: x**2, numbers))
evens = list(filter(lambda x: x % 2 == 0, numbers))
total = reduce(lambda acc, x: acc + x, numbers, 0)

# Composition
def pipeline(*funcs):
    def apply(data):
        result = data
        for f in funcs:
            result = f(result)
        return result
    return apply

process = pipeline(
    lambda x: [i**2 for i in x],
    lambda x: [i for i in x if i > 10],
    sum
)
```

**When used:** Data pipelines (PySpark!), React state management, concurrent programming.

**Why FP matters for your career (Prince):**
- PySpark is fundamentally functional (map, filter, reduce on RDDs/DataFrames)
- AWS Lambda is about pure, stateless functions
- Modern Python embraces FP: list comprehensions, generators, functools

### 4. Declarative Programming
**What:** You describe WHAT you want, not HOW to get it.

```sql
-- Declarative: "Give me active users sorted by name"
SELECT name, email FROM users WHERE active = true ORDER BY name;

-- vs Imperative equivalent:
# active_users = []
# for user in users:
#     if user.active:
#         active_users.append(user)
# active_users.sort(key=lambda u: u.name)
```

**When used:** SQL, HTML/CSS, configuration files, infrastructure-as-code (Terraform).

---

## SOLID Principles — The Engineering Standard

These are the principles that separate junior from senior engineers. Know them cold.

| Principle | Rule | Violation Example | Fix |
|-----------|------|-------------------|-----|
| **S**ingle Responsibility | A class does ONE thing | `UserManager` that handles auth, email, logging | Split into `AuthService`, `EmailService`, `Logger` |
| **O**pen/Closed | Open for extension, closed for modification | Adding new payment type requires editing `process_payment()` | Use strategy pattern with interface |
| **L**iskov Substitution | Subtypes must be substitutable for parent | `Square(Rectangle)` breaks if you set width independently | Redesign hierarchy |
| **I**nterface Segregation | Don't force implementation of unused methods | `Worker` interface with `eat()` for `Robot` class | Split into `Workable`, `Eatable` |
| **D**ependency Inversion | Depend on abstractions, not concretions | `OrderService` directly creates `MySQLDatabase()` | Inject `Database` interface |

### The Most Important One: Dependency Inversion

```python
# BAD — tightly coupled
class ReportGenerator:
    def __init__(self):
        self.db = PostgresDatabase()  # Hardcoded dependency
    
    def generate(self):
        data = self.db.query("SELECT ...")
        return format_report(data)

# GOOD — dependency injection
class ReportGenerator:
    def __init__(self, db: Database):  # Accept any Database
        self.db = db
    
    def generate(self):
        data = self.db.query("SELECT ...")
        return format_report(data)

# Now you can test with MockDatabase, switch to MySQL, etc.
```

---

## Design Patterns You MUST Know for Interviews

### Creational
| Pattern | When | One-liner |
|---------|------|-----------|
| Singleton | One instance globally (DB connection pool) | `getInstance()` returns same object |
| Factory | Create objects without specifying exact class | `PaymentFactory.create("stripe")` |
| Builder | Construct complex objects step by step | `Query().select("name").where("active").limit(10)` |

### Structural
| Pattern | When | One-liner |
|---------|------|-----------|
| Adapter | Make incompatible interfaces work together | Wrap old API to match new interface |
| Decorator | Add behavior without modifying class | `@cache`, `@retry`, `@log` in Python |
| Facade | Simplify complex subsystem | `OrderService` wraps inventory + payment + shipping |

### Behavioral
| Pattern | When | One-liner |
|---------|------|-----------|
| Strategy | Swap algorithms at runtime | Different sorting/pricing strategies |
| Observer | Notify multiple objects of state change | Event systems, pub/sub |
| Iterator | Traverse collection without exposing internals | Python's `__iter__` / `__next__` |

---

## How This Shows Up in Real Interviews

### LLD Interview (45 min)
"Design a parking lot system"
- They want: Classes, interfaces, SOLID principles, design patterns
- You use: OOP + Strategy pattern + Factory pattern

### Coding Interview
"Your code is clean" (positive signal) means:
- Meaningful variable names (not `i`, `j`, `temp`)
- Functions that do one thing
- No global state mutation
- Clear data flow

### System Design Interview
"How would you make this extensible?"
- Plugin architecture (Strategy pattern)
- Event-driven (Observer pattern)
- Microservices (each service = Single Responsibility)

---

## Paradigm Selection Guide

| Scenario | Best Paradigm | Why |
|----------|--------------|-----|
| Data transformation pipeline | Functional | Pure transformations, easy to parallelize |
| Complex domain modeling | OOP | Real-world entities map to objects |
| Database queries | Declarative (SQL) | Optimizer handles the "how" |
| Performance-critical algorithm | Imperative | Direct control over every operation |
| React/Frontend state | Functional + Declarative | Immutable state, declarative UI |
| AWS infrastructure | Declarative (IaC) | Describe desired state, tools figure out how |
| Interview coding | Imperative + light OOP | Clarity and directness win |

---

## Key Takeaways

1. You don't need to pick one paradigm — great engineers blend them fluently
2. For interviews: write imperative solutions but with clean structure (good names, small functions)
3. For system design: demonstrate SOLID and design patterns knowledge
4. For your PySpark work: think functionally — transformations on immutable DataFrames
5. OOP is not about inheritance (avoid deep hierarchies) — it's about **encapsulation and polymorphism**
