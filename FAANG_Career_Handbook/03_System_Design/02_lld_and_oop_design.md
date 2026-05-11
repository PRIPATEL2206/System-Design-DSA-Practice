# Low-Level Design (LLD) & Object-Oriented Design

## What LLD Interviews Test

Unlike HLD (which tests architecture), LLD tests:
- Can you model a real-world problem with classes?
- Do you follow SOLID principles?
- Can you write extensible, maintainable code?
- Do you understand design patterns?

---

## The LLD Interview Framework

### Step 1: Clarify Requirements (2-3 min)
- What are the core entities?
- What operations are needed?
- What constraints exist?
- What's the scale? (Usually single-machine for LLD)

### Step 2: Identify Core Objects/Entities (3-5 min)
- Nouns in requirements → Classes
- Verbs → Methods
- Adjectives → Attributes

### Step 3: Define Relationships (3-5 min)
- IS-A → Inheritance
- HAS-A → Composition (prefer this!)
- USES → Dependency

### Step 4: Apply Design Patterns (5-10 min)
- Strategy, Observer, Factory, Singleton as needed

### Step 5: Write Key Classes (15-20 min)
- Focus on interfaces and public methods
- Skip getters/setters unless they have logic

---

## LLD Case Study 1: Parking Lot

### Requirements
- Multi-floor parking lot
- Different vehicle sizes (motorcycle, car, bus)
- Different spot sizes (small, medium, large)
- Track which spots are free

### Design

```python
from enum import Enum
from abc import ABC, abstractmethod

class VehicleSize(Enum):
    MOTORCYCLE = 1
    CAR = 2
    BUS = 3

class SpotSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3

class Vehicle(ABC):
    def __init__(self, license_plate: str, size: VehicleSize):
        self.license_plate = license_plate
        self.size = size

class Car(Vehicle):
    def __init__(self, license_plate):
        super().__init__(license_plate, VehicleSize.CAR)

class ParkingSpot:
    def __init__(self, spot_id: str, size: SpotSize, floor: int):
        self.spot_id = spot_id
        self.size = size
        self.floor = floor
        self.vehicle = None
    
    def is_available(self):
        return self.vehicle is None
    
    def can_fit(self, vehicle: Vehicle):
        return self.is_available() and self.size.value >= vehicle.size.value
    
    def park(self, vehicle: Vehicle):
        if not self.can_fit(vehicle):
            raise ValueError("Vehicle cannot fit in this spot")
        self.vehicle = vehicle
    
    def remove_vehicle(self):
        self.vehicle = None

class ParkingFloor:
    def __init__(self, floor_number: int, spots: list):
        self.floor_number = floor_number
        self.spots = spots
    
    def find_available_spot(self, vehicle: Vehicle):
        for spot in self.spots:
            if spot.can_fit(vehicle):
                return spot
        return None

class ParkingLot:
    _instance = None
    
    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
    
    def __init__(self):
        self.floors = []
        self.active_tickets = {}
    
    def park_vehicle(self, vehicle: Vehicle):
        for floor in self.floors:
            spot = floor.find_available_spot(vehicle)
            if spot:
                spot.park(vehicle)
                ticket = ParkingTicket(vehicle, spot)
                self.active_tickets[ticket.ticket_id] = ticket
                return ticket
        raise Exception("Parking lot is full")
    
    def unpark_vehicle(self, ticket_id: str):
        ticket = self.active_tickets.pop(ticket_id)
        ticket.spot.remove_vehicle()
        return ticket.calculate_fee()
```

---

## LLD Case Study 2: Elevator System

### Key Design Decisions
- **Strategy Pattern** for scheduling (FCFS, Shortest Seek, SCAN)
- **Observer Pattern** for button presses notifying controller
- **State Pattern** for elevator states (IDLE, MOVING_UP, MOVING_DOWN)

```python
from enum import Enum

class Direction(Enum):
    UP = 1
    DOWN = -1
    IDLE = 0

class ElevatorState(Enum):
    IDLE = 0
    MOVING = 1
    DOOR_OPEN = 2

class Request:
    def __init__(self, floor: int, direction: Direction):
        self.floor = floor
        self.direction = direction

class Elevator:
    def __init__(self, elevator_id: int, capacity: int):
        self.id = elevator_id
        self.current_floor = 0
        self.direction = Direction.IDLE
        self.state = ElevatorState.IDLE
        self.capacity = capacity
        self.requests = []
    
    def add_request(self, floor: int):
        if floor not in self.requests:
            self.requests.append(floor)
            self.requests.sort()
    
    def move(self):
        if not self.requests:
            self.state = ElevatorState.IDLE
            self.direction = Direction.IDLE
            return
        
        next_floor = self.requests[0] if self.direction != Direction.DOWN else self.requests[-1]
        if next_floor > self.current_floor:
            self.direction = Direction.UP
            self.current_floor += 1
        elif next_floor < self.current_floor:
            self.direction = Direction.DOWN
            self.current_floor -= 1
        
        if self.current_floor in self.requests:
            self.requests.remove(self.current_floor)
            self.state = ElevatorState.DOOR_OPEN

class SchedulingStrategy(ABC):
    @abstractmethod
    def select_elevator(self, elevators: list, request: Request) -> Elevator:
        pass

class NearestElevatorStrategy(SchedulingStrategy):
    def select_elevator(self, elevators, request):
        return min(elevators, key=lambda e: abs(e.current_floor - request.floor))

class ElevatorController:
    def __init__(self, elevators: list, strategy: SchedulingStrategy):
        self.elevators = elevators
        self.strategy = strategy
    
    def handle_request(self, request: Request):
        elevator = self.strategy.select_elevator(self.elevators, request)
        elevator.add_request(request.floor)
```

---

## LLD Case Study 3: Design a Rate Limiter

```python
import time
from collections import defaultdict

class RateLimiter(ABC):
    @abstractmethod
    def allow_request(self, client_id: str) -> bool:
        pass

class TokenBucketLimiter(RateLimiter):
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.buckets = {}
    
    def allow_request(self, client_id: str) -> bool:
        now = time.time()
        if client_id not in self.buckets:
            self.buckets[client_id] = {'tokens': self.capacity, 'last_refill': now}
        
        bucket = self.buckets[client_id]
        elapsed = now - bucket['last_refill']
        bucket['tokens'] = min(self.capacity, bucket['tokens'] + elapsed * self.refill_rate)
        bucket['last_refill'] = now
        
        if bucket['tokens'] >= 1:
            bucket['tokens'] -= 1
            return True
        return False

class SlidingWindowLimiter(RateLimiter):
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = defaultdict(list)
    
    def allow_request(self, client_id: str) -> bool:
        now = time.time()
        window_start = now - self.window
        self.requests[client_id] = [t for t in self.requests[client_id] if t > window_start]
        
        if len(self.requests[client_id]) < self.max_requests:
            self.requests[client_id].append(now)
            return True
        return False
```

---

## Common LLD Interview Problems

| Problem | Key Patterns | Focus Areas |
|---------|-------------|-------------|
| Parking Lot | Strategy, Singleton | Spot allocation, pricing |
| Elevator System | Strategy, Observer, State | Scheduling algorithm |
| Library Management | Observer | Book/member CRUD, fines |
| Hotel Booking | Strategy | Room allocation, concurrency |
| Chess Game | State, Command | Move validation, turns |
| File System | Composite | Tree structure, permissions |
| Vending Machine | State | State transitions, inventory |
| ATM | State, Chain of Responsibility | Transaction flow |
| Movie Ticket Booking | Strategy, Observer | Seat selection, concurrency |
| Splitwise (Expense Sharing) | Observer | Debt simplification |

---

## Key Principles for LLD Interviews

1. **Prefer Composition over Inheritance** — "Has-a" is more flexible than "Is-a"
2. **Program to Interfaces** — Accept abstract types, not concrete ones
3. **Single Responsibility** — Each class should have ONE reason to change
4. **Open for Extension, Closed for Modification** — Add new behavior without changing existing code
5. **Keep It Simple** — Don't over-engineer. Start simple, add complexity when needed.

## What Makes a Great LLD Answer

- Clear class hierarchy with well-defined responsibilities
- Extensible design (new vehicle types, new spot types = no code change)
- Concurrency considerations mentioned
- Trade-offs discussed (in-memory vs persistent, simple vs optimized)
- Enums for finite states, not strings
- No business logic in constructors
