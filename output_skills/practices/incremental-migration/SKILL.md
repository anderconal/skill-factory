---
name: incremental-migration
description: Guides incremental replacement of legacy code using four patterns matched to scale — Strangler Fig, Branch by Abstraction, Parallel Change, and Lift-Up Conditional. Use when modernizing legacy without a full rewrite or development freeze.
---

# Incremental Migration

STARTER_CHARACTER = 🌿

When starting, announce: "🌿 Using INCREMENTAL-MIGRATION skill".

## The Core Principle

"For each desired change, make the change easy, then make the easy change." — Kent Beck

Never rewrite everything at once. Old code keeps running while new code grows alongside it.

## Choose the Pattern by Scale

```
Reemplazas un sistema/servicio completo  → Strangler Fig
Reemplazas un componente o librería      → Branch by Abstraction
Cambias una firma de método o API        → Parallel Change
Separas lógica pura de efectos secundarios → Lift-Up Conditional
```

---

## Pattern 1: Strangler Fig (service/system level)

Build new alongside old. Intercept calls at the boundary. Route traffic progressively from old to new. Delete old when migration is complete.

**Three moves:**
1. **Intercept** — place a facade or proxy at the boundary between callers and legacy code
2. **Route** — direct a subset of calls to the new implementation; legacy handles the rest
3. **Retire** — when all calls go through the new path, delete the legacy code

**Where to place the interceptor:**
- HTTP layer: reverse proxy (nginx, YARP in .NET) routes specific paths to the new service
- Interface boundary: introduce `IOrderService`; both implementations satisfy it; a feature flag decides which runs
- Message bus: new consumer handles a subset of messages; legacy handles the rest

```csharp
// C#: introduce the interface (no behavior change yet)
public interface IOrderProcessor { OrderResult Process(Order order); }

// Legacy class implements the interface without internal changes
public class LegacyOrderProcessor : IOrderProcessor { ... }

// New implementation, fully testable
public class OrderProcessor : IOrderProcessor { ... }

// Route via feature flag
services.AddScoped<IOrderProcessor>(sp =>
    featureFlags.UseNewOrderProcessor
        ? sp.GetRequiredService<OrderProcessor>()
        : sp.GetRequiredService<LegacyOrderProcessor>());
```

```typescript
// TypeScript/Vue: composable as interceptor
export function useOrderService(): OrderService {
    return featureFlag('new-order-service')
        ? new OrderService()
        : new LegacyOrderService();
}
```

**Rules:**
- New code never calls legacy code directly — dependency flows one way
- Track which path each call takes; 100% on the new path is your exit criterion
- The interceptor routes — it does not transform business logic

---

## Pattern 2: Branch by Abstraction (component/library level)

Use when replacing an entire internal component or library (ORM, payment provider, auth module) while the rest of the system keeps running.

**Five steps:**
1. **Create an abstraction** — introduce an interface (contract) between your code and the legacy component
2. **Refactor callers** — make all code call the interface, not the legacy component directly; nothing changes behind the interface yet
3. **Build the new component** — write the new implementation that satisfies the interface
4. **Toggle** — use dependency injection or a feature flag to swap implementations; migrate progressively per user, flow, or environment
5. **Clean up** — delete the old implementation once the toggle is at 100%

```csharp
// Step 1: abstraction
public interface IPaymentGateway
{
    PaymentResult Charge(decimal amount, string token);
}

// Step 2: legacy wrapped behind interface (no internal change)
public class LegacyPaymentGateway : IPaymentGateway { ... }

// Step 3: new implementation
public class StripePaymentGateway : IPaymentGateway { ... }

// Step 4: toggle
services.AddScoped<IPaymentGateway>(sp =>
    featureFlags.UseStripe
        ? sp.GetRequiredService<StripePaymentGateway>()
        : sp.GetRequiredService<LegacyPaymentGateway>());
```

---

## Pattern 3: Parallel Change / Expand-Contract (method/API level)

Use when changing a method signature, DB schema, or public API with zero downtime. Avoids the Big Bang (changing everything at once).

**Three phases:**

**Expand** — add the new signature alongside the old one, without touching the old:
```csharp
// OLD method stays intact
public decimal CalculatePrice(decimal price) { ... }

// NEW method added alongside
public decimal CalculatePrice(decimal price, decimal discount)
{
    return price * (1 - discount);
}
// Old method delegates to new with a safe default
public decimal CalculatePrice(decimal price) => CalculatePrice(price, 0m);
```

**Migrate** — update callers one by one to use the new signature. Deploy after each batch. Both signatures coexist so CI never breaks:
```csharp
// Today: migrate OrderController
var price = CalculatePrice(item.Price, order.Discount);

// Tomorrow: migrate InvoiceService
var price = CalculatePrice(item.Price, invoice.Discount);
```

**Contract** — once no caller uses the old signature, delete it:
```csharp
// Old overload gone — only the new signature remains
public decimal CalculatePrice(decimal price, decimal discount) { ... }
```

Apply the same pattern to DB schema changes: add the new column (Expand), migrate reads/writes to use it (Migrate), drop the old column (Contract).

---

## Pattern 4: Lift-Up Conditional (logic level)

Use when business logic is tangled with side effects (DB calls, email sends, file writes) inside the same if/else block. This is the path toward Hexagonal Architecture.

**The problem:**
```csharp
// Logic and side effects mixed — impossible to unit test
public void ProcessOrder(Order order)
{
    if (order.Total > 1000)
    {
        _db.Save(order);            // side effect
        _email.SendConfirmation();  // side effect
    }
    else
    {
        _db.SaveDraft(order);       // side effect
    }
}
```

**The technique — lift the decision out, push effects outward:**
```csharp
// Pure function — trivially testable
internal OrderDecision Decide(Order order)
{
    return order.Total > 1000
        ? OrderDecision.Confirm
        : OrderDecision.Draft;
}

// Effects isolated in the outer layer (controller/handler)
public void ProcessOrder(Order order)
{
    var decision = Decide(order); // pure, testable
    switch (decision)
    {
        case OrderDecision.Confirm:
            _db.Save(order);
            _email.SendConfirmation();
            break;
        case OrderDecision.Draft:
            _db.SaveDraft(order);
            break;
    }
}
```

```typescript
// TypeScript equivalent
function decide(order: Order): 'confirm' | 'draft' {
    return order.total > 1000 ? 'confirm' : 'draft';
}

async function processOrder(order: Order): Promise<void> {
    const decision = decide(order); // pure, testable
    if (decision === 'confirm') {
        await db.save(order);
        await email.sendConfirmation();
    } else {
        await db.saveDraft(order);
    }
}
```

The pure function (`decide`) has no dependencies and can be unit tested without mocks. The side effects are pushed to the outermost layer where they can be isolated or mocked at the boundary.

Use `hexagonal-architecture` skill when ready to formalize ports and adapters.

---

## Gradual Typing During Migration

When the legacy codebase is untyped (JS → TS), add types at the system boundaries first and work inward:

1. API boundary — type request/response shapes (controllers, route handlers)
2. Data boundary — type DB query results and DTOs
3. Domain boundary — type service method signatures
4. Internal implementation — type individual variables and functions last

In Vue 2 → Vue 3 migrations: type `props` and `emits` contracts before typing internal component logic.

## Credits

Strangler Fig: Martin Fowler, *StranglerFigApplication* — martinfowler.com
Branch by Abstraction: Martin Fowler, *BranchByAbstraction* — martinfowler.com
Parallel Change / Expand-Contract: Kent Beck, *TDD by Example*; popularized by Fowler
Lift-Up Conditional: Rafa Gómez & Javier Ferrer (Codely)
