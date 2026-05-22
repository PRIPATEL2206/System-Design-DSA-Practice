# OOP & Design Patterns — Complete Deep Dive

## 1. Four Pillars of OOP

### Encapsulation
```
Bundle data + methods together. Hide internal state, expose only interface.

class BankAccount:
    def __init__(self):
        self.__balance = 0  # private — can't access from outside
    
    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount  # controlled access
    
    def get_balance(self):
        return self.__balance  # read-only access

Why: Change internal implementation without breaking callers.
```

### Abstraction
```
Hide complex implementation, show only what's necessary.

from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount): pass  # WHAT, not HOW
    
class StripeProcessor(PaymentProcessor):
    def process_payment(self, amount):
        # complex Stripe API integration hidden here
        
class RazorpayProcessor(PaymentProcessor):
    def process_payment(self, amount):
        # complex Razorpay API integration hidden here

# Caller doesn't care which processor:
processor.process_payment(500)  # works for any implementation
```

### Inheritance
```
Create new class from existing class. "IS-A" relationship.

class Animal:
    def breathe(self): print("Breathing")

class Dog(Animal):  # Dog IS-A Animal
    def bark(self): print("Woof!")

dog = Dog()
dog.breathe()  # inherited from Animal
dog.bark()     # own method

Types:
- Single: One parent
- Multiple: Multiple parents (Python supports, Java doesn't)
- Multilevel: A → B → C chain
- Hierarchical: One parent, multiple children

⚠️ Prefer composition over inheritance (flexible, less coupling)
```

### Polymorphism
```
Same interface, different behavior.

# Method overriding (runtime polymorphism)
class Shape:
    def area(self): pass

class Circle(Shape):
    def area(self): return 3.14 * self.r ** 2

class Rectangle(Shape):
    def area(self): return self.w * self.h

# Same method call, different behavior:
for shape in [Circle(5), Rectangle(3, 4)]:
    print(shape.area())  # each calculates differently

# Method overloading (compile-time, not native in Python)
# In Java: same method name, different parameters
```

---

## 2. SOLID Principles

```
S — Single Responsibility Principle
    A class should have only ONE reason to change.
    ❌ UserManager: handles login, email, database, logging
    ✅ AuthService, EmailService, UserRepository, Logger (separate)

O — Open/Closed Principle
    Open for extension, closed for modification.
    ❌ Adding new shape requires modifying existing calculate_area()
    ✅ Each shape implements its own area() — add new shape without changing old code

L — Liskov Substitution Principle
    Subtypes must be substitutable for their base types.
    ❌ Square extends Rectangle but breaks setWidth/setHeight contract
    ✅ If code works with Animal, it must work with Dog (no surprises)

I — Interface Segregation Principle
    Don't force classes to implement interfaces they don't use.
    ❌ IWorker: work(), eat(), sleep() — Robot can't eat!
    ✅ IWorkable: work()  |  IFeedable: eat()  — Robot only implements IWorkable

D — Dependency Inversion Principle
    Depend on abstractions, not concretions.
    ❌ OrderService directly creates MySQLDatabase()
    ✅ OrderService depends on DatabaseInterface — inject MySQL/Postgres/Mongo
```

---

## 3. Design Patterns (Top 10 for Interviews)

### Creational Patterns

**1. Singleton** — Only one instance exists
```python
class DatabaseConnection:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Both are the same object:
db1 = DatabaseConnection()
db2 = DatabaseConnection()
assert db1 is db2  # True!

Use: DB connection pool, Logger, Config manager
⚠️ Makes testing hard (global state). Prefer dependency injection.
```

**2. Factory** — Create objects without specifying exact class
```python
class NotificationFactory:
    @staticmethod
    def create(type):
        if type == "email": return EmailNotification()
        if type == "sms": return SMSNotification()
        if type == "push": return PushNotification()
        raise ValueError(f"Unknown type: {type}")

notification = NotificationFactory.create("email")
notification.send("Hello!")

Use: When creation logic is complex or varies by type.
```

**3. Builder** — Construct complex objects step by step
```python
class QueryBuilder:
    def __init__(self):
        self._table = None
        self._conditions = []
        self._limit = None
    
    def from_table(self, table): self._table = table; return self
    def where(self, condition): self._conditions.append(condition); return self
    def limit(self, n): self._limit = n; return self
    def build(self): return f"SELECT * FROM {self._table} ..."

query = QueryBuilder().from_table("users").where("age > 25").limit(10).build()

Use: When object has many optional parameters.
```

### Structural Patterns

**4. Adapter** — Make incompatible interfaces work together
```python
class OldPaymentGateway:
    def make_payment(self, amount_in_paise): ...

class NewPaymentAdapter:
    def __init__(self, old_gateway):
        self.gateway = old_gateway
    
    def pay(self, amount_in_rupees):
        self.gateway.make_payment(amount_in_rupees * 100)

Use: Integrating legacy systems, third-party libraries.
```

**5. Decorator** — Add behavior to objects dynamically
```python
class Coffee:
    def cost(self): return 50

class MilkDecorator:
    def __init__(self, coffee):
        self._coffee = coffee
    def cost(self): return self._coffee.cost() + 20

class SugarDecorator:
    def __init__(self, coffee):
        self._coffee = coffee
    def cost(self): return self._coffee.cost() + 10

coffee = SugarDecorator(MilkDecorator(Coffee()))
print(coffee.cost())  # 50 + 20 + 10 = 80

Use: Adding features without modifying base class. Python @decorators.
```

### Behavioral Patterns

**6. Observer** — Notify multiple objects of state changes
```python
class EventBus:
    def __init__(self):
        self._subscribers = {}
    
    def subscribe(self, event, callback):
        self._subscribers.setdefault(event, []).append(callback)
    
    def publish(self, event, data):
        for callback in self._subscribers.get(event, []):
            callback(data)

bus = EventBus()
bus.subscribe("order_placed", send_email)
bus.subscribe("order_placed", update_inventory)
bus.publish("order_placed", order)  # both notified!

Use: Event systems, UI updates, pub/sub.
```

**7. Strategy** — Swap algorithms at runtime
```python
class PricingStrategy:
    def calculate(self, amount): pass

class RegularPricing(PricingStrategy):
    def calculate(self, amount): return amount

class PremiumPricing(PricingStrategy):
    def calculate(self, amount): return amount * 0.8  # 20% off

class Order:
    def __init__(self, strategy: PricingStrategy):
        self.strategy = strategy
    
    def total(self, amount):
        return self.strategy.calculate(amount)

Use: Payment methods, sorting algorithms, compression strategies.
```

**8. Template Method** — Define skeleton, let subclasses fill details
```python
class DataPipeline:
    def run(self):  # template method (fixed order)
        data = self.extract()
        cleaned = self.transform(data)
        self.load(cleaned)
    
    def extract(self): raise NotImplementedError
    def transform(self, data): raise NotImplementedError
    def load(self, data): raise NotImplementedError

class CSVPipeline(DataPipeline):
    def extract(self): return read_csv(...)
    def transform(self, data): return clean(data)
    def load(self, data): write_to_db(data)
```

**9. Command** — Encapsulate request as an object
```python
class Command:
    def execute(self): pass
    def undo(self): pass

class AddTextCommand(Command):
    def __init__(self, editor, text):
        self.editor = editor
        self.text = text
    
    def execute(self): self.editor.add(self.text)
    def undo(self): self.editor.remove(self.text)

# Enables: Undo/redo, queuing, logging all operations
```

**10. Iterator** — Access elements without exposing structure
```python
class TreeIterator:
    """Inorder traversal of BST — caller doesn't know tree structure"""
    def __init__(self, root):
        self.stack = []
        self._push_left(root)
    
    def __next__(self):
        node = self.stack.pop()
        self._push_left(node.right)
        return node.val
    
    def _push_left(self, node):
        while node:
            self.stack.append(node)
            node = node.left
```

---

## 4. Clean Architecture Principles

```
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │    ┌──────────────────────────────────────┐     │    │
│  │    │    ┌───────────────────────────┐     │     │    │
│  │    │    │       Entities            │     │     │    │
│  │    │    │   (business rules)        │     │     │    │
│  │    │    └───────────────────────────┘     │     │    │
│  │    │         Use Cases                     │     │    │
│  │    │    (application logic)                │     │    │
│  │    └──────────────────────────────────────┘     │    │
│  │           Controllers / Gateways                 │    │
│  │      (interface adapters)                        │    │
│  └─────────────────────────────────────────────────┘    │
│              Frameworks & Drivers                         │
│         (DB, Web, External services)                     │
└──────────────────────────────────────────────────────────┘

Dependency Rule: Inner circles know NOTHING about outer circles.
  Entities don't know about DB, HTTP, or frameworks.
  This makes the core testable and framework-independent.
```

---

## 5. Interview Questions

1. Explain the 4 pillars of OOP with real examples.
2. What is SOLID? Explain each principle.
3. Composition vs Inheritance — when to use which?
4. Explain the Singleton pattern. What are its problems?
5. Design a parking lot system (uses Factory, Strategy, Observer).
6. What is dependency injection? Why is it important?
7. Explain the Observer pattern. Where is it used in real systems?
8. Abstract class vs Interface — differences and when to use.
9. What is the Open/Closed principle? Give a code example.
10. Design a notification system using appropriate design patterns.
