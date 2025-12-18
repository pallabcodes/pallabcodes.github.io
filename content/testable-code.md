+++
title = "The Art of Writing Testable Code"
date = 2024-12-16
description = "Practical patterns for writing code that's easy to test without over-engineering."
[taxonomies]
tags = ["testing", "software-design", "clean-code"]
+++

Writing testable code isn't about testing—it's about good design. Code that's easy to test is usually easy to understand, modify, and maintain.

<!-- more -->

## The Core Principle: Dependency Injection

The single most important technique for testable code is dependency injection. Instead of creating dependencies inside your class, pass them in.

### Before: Untestable

```python
class OrderService:
    def __init__(self):
        self.db = PostgresDatabase()  # Hard-coded dependency
        self.email = SMTPEmailClient()  # Another hard-coded dependency
    
    def place_order(self, order: Order) -> bool:
        self.db.save(order)
        self.email.send(order.customer_email, "Order confirmed!")
        return True
```

### After: Testable

```python
class OrderService:
    def __init__(self, db: Database, email: EmailClient):
        self.db = db
        self.email = email
    
    def place_order(self, order: Order) -> bool:
        self.db.save(order)
        self.email.send(order.customer_email, "Order confirmed!")
        return True

# In tests
def test_place_order():
    mock_db = MockDatabase()
    mock_email = MockEmailClient()
    service = OrderService(mock_db, mock_email)
    
    order = Order(customer_email="test@example.com")
    assert service.place_order(order) == True
    assert mock_db.saved_count == 1
    assert mock_email.sent_count == 1
```

## Pure Functions Are Your Friend

Pure functions (same input → same output, no side effects) are trivially testable:

```go
// Pure function - easy to test
func CalculateDiscount(price float64, quantity int) float64 {
    if quantity >= 10 {
        return price * 0.9
    }
    return price
}

// Test
func TestCalculateDiscount(t *testing.T) {
    tests := []struct {
        price    float64
        quantity int
        expected float64
    }{
        {100.0, 5, 100.0},
        {100.0, 10, 90.0},
        {100.0, 15, 90.0},
    }
    
    for _, tt := range tests {
        result := CalculateDiscount(tt.price, tt.quantity)
        if result != tt.expected {
            t.Errorf("got %f, want %f", result, tt.expected)
        }
    }
}
```

## Separate I/O from Logic

Push I/O operations to the edges of your system:

```
┌────────────────────────────────────────────────┐
│                    Handler                      │
│  (I/O: HTTP, Database, Files)                  │
│                     │                           │
│                     ▼                           │
│              ┌─────────────┐                    │
│              │ Pure Logic  │  ← Test this!     │
│              └─────────────┘                    │
│                     │                           │
│                     ▼                           │
│             (I/O: Response)                     │
└────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Inject dependencies** - Never `new` up collaborators inside constructors
2. **Prefer pure functions** - They're trivially testable
3. **Isolate I/O** - Keep side effects at the boundaries
4. **Small interfaces** - Depend on abstractions, not implementations

Testable code is a byproduct of good design, not an afterthought.
