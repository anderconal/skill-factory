---
name: legacy-safety-net
description: Builds a test safety net for untested legacy code using characterization tests (Golden Master), seam identification, and Sprout/Wrap methods. Use when adding tests to code that has none, before refactoring.
---

# Legacy Safety Net

STARTER_CHARACTER = 🕸️

When starting, announce: "🕸️ Using LEGACY-SAFETY-NET skill".

## The Core Principle

"Legacy code is code without tests." — Michael Feathers

You cannot refactor what you cannot test. You cannot test what you cannot understand. Characterization tests let you capture behavior before you understand it — and protect it during refactoring.

## Step 1: Characterization Tests (Golden Master)

The goal is not to verify correctness. The goal is to capture current behavior — bugs included — so any change breaks a test.

**The algorithm:**
1. Call the function/class with realistic inputs
2. Capture all outputs (return values, side effects, console output, DB state)
3. Save the output as the approved "golden" baseline
4. Write a test that compares actual output against the baseline
5. If refactoring changes the output → test fails → you notice

```csharp
// C# with ApprovalTests
[Fact]
public void CharacterizeOrderProcessor_StandardOrder()
{
    var sut = new OrderProcessor(new RealDependency());
    var result = sut.Process(TestData.StandardOrder);
    Approvals.Verify(result.ToString());
}
```

```typescript
// TypeScript with Jest snapshots
test('characterize invoice generator', () => {
    const result = generateInvoice(testOrder);
    expect(result).toMatchSnapshot(); // Creates .snap file on first run
});
```

Scrub timestamps, IDs, and random values before comparison — they create false failures.

Use the `approval-tests` skill for full implementation details per language.

## Step 2: Find Seams

A **seam** is a place where you can alter behavior without editing the source code at that point.

Common seam types:
- **Interface/DI seam**: The class accepts a dependency via constructor or property. Replace it in tests with a test double.
- **Method override seam**: Subclass the class under test and override the method you need to control.
- **Module seam**: Redirect a module import in tests (`jest.mock()` in TS/JS).

In C#: prefer constructor injection. In TS/JS: prefer `jest.mock()` or explicit dependency injection.

**If there are no seams** (global state, `new` inside methods, static calls everywhere): do not refactor to add DI yet. First use a subclass-and-override seam to get any test in place, then introduce proper injection incrementally.

## Step 3: Sprout Method

Use when you must add new behavior to a monster method you cannot safely modify.

Add the new logic as a separate, testable method. Call it from inside the legacy method.

```csharp
// BEFORE: 600-line method, untouchable
public void ProcessOrder(Order order)
{
    // ... 600 lines ...
}

// AFTER: new logic lives in a sprout — fully testable in isolation
public void ProcessOrder(Order order)
{
    // ... 600 lines unchanged ...
    order.Tax = CalculateTax(order); // ← sprout called here
}

internal decimal CalculateTax(Order order) // ← new, small, testable
{
    return order.Total * TaxRate;
}
```

## Step 4: Wrap Method

Use when you need to execute new logic before or after existing behavior.

Rename the original method. Create a new method with the original name that calls the old one.

```csharp
// BEFORE
public void SaveOrder(Order order) { /* untestable legacy */ }

// AFTER
private void SaveOrder_Legacy(Order order) { /* original, unchanged */ }

public void SaveOrder(Order order) // ← new method, original name
{
    SaveOrder_Legacy(order);
    _eventBus.Publish(new OrderSaved(order.Id)); // ← new testable logic
}
```

```typescript
// TypeScript equivalent
class OrderService {
    private saveOrder_legacy(order: Order): void { /* original */ }

    saveOrder(order: Order): void {
        this.saveOrder_legacy(order);
        this.eventBus.publish(new OrderSaved(order.id));
    }
}
```

## Sequence

1. **Characterize** — capture what the code does now (protect behavior)
2. **Find a seam** — identify where to inject control
3. **Sprout or Wrap** — add new behavior without touching the untested core
4. **Refactor the core** — only after characterization tests pass

## Credits

Michael Feathers, *Working Effectively with Legacy Code* (Prentice Hall, 2004).
