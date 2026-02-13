# Test Types: Unit vs Integration Tests

Understanding the difference between unit tests and integration tests is fundamental to writing effective, fast, and maintainable test suites in Business Central.

## The Core Problem

Most AL tests are **integration tests** disguised as unit tests. This leads to:

- Slow test suites (hours instead of seconds)
- Brittle tests (break when dependencies change)
- Complex setup (many GIVENs for unrelated functionality)
- Low value per test (testing Microsoft code, not your code)

## Unit Tests

**Definition:** Tests that validate a single unit of code (procedure, function, component) in **isolation** from its dependencies.

### Characteristics

| Property         | Unit Tests                               |
| ---------------- | ---------------------------------------- |
| Execution speed  | <100ms per test                          |
| Dependencies     | Mocked/stubbed                           |
| Database access  | None or minimal                          |
| Setup complexity | Simple (1-2 GIVENs)                      |
| Brittleness      | Low (test only YOUR code)                |
| What they test   | Business logic, calculations, algorithms |

### Example: True Unit Test

```al
[Test]
procedure TestBonusCalculation()
var
    MockConverter: Codeunit "Mock Currency Converter";
    BonusCalc: Codeunit "Bonus Calculator";
    Result: Decimal;
begin
    // [Scenario] Bonus is 10% of sales

    // [Given] Mock converter always returns 100
    MockConverter.SetReturnValue(100);

    // [When] Calculating bonus
    Result := BonusCalc.Calculate(1000, MockConverter);

    // [Then] Bonus is 10%
    Assert.AreEqual(100, Result, 'Bonus should be 10% of converted amount');
end;
```

**Notice:**

- No database setup
- No currencies/exchange rates created
- Tests ONLY the bonus calculation logic
- Single, clear assertion

## Integration Tests

**Definition:** Tests that validate multiple components working together, including database access, base app functionality, and cross-module interactions.

### Characteristics

| Property         | Integration Tests                     |
| ---------------- | ------------------------------------- |
| Execution speed  | 500ms - 5s+ per test                  |
| Dependencies     | Real implementations                  |
| Database access  | Heavy                                 |
| Setup complexity | Complex (5+ GIVENs)                   |
| Brittleness      | High (dependencies change)            |
| What they test   | End-to-end flows, module interactions |

### Example: Integration Test (Typical AL Test)

```al
[Test]
procedure TestBonusCalculationWithCurrencyConversion()
var
    Currency: Record Currency;
    Customer: Record Customer;
    CustLedgerEntry: Record "Cust. Ledger Entry";
begin
    // [Scenario] Bonus calculated from foreign currency sales

    // [Given] Currency with exchange rate
    LibraryERM.CreateCurrency(Currency);
    LibraryERM.CreateExchangeRate(Currency.Code, WorkDate(), 10, 10);

    // [Given] Customer with currency
    LibrarySales.CreateCustomer(Customer);
    Customer.Validate("Currency Code", Currency.Code);
    Customer.Modify();

    // [Given] Sales ledger entry
    CreateSalesLedgerEntry(Customer, 1000, Currency.Code);

    // [When] Running bonus calculation batch
    BonusBatch.Run(Customer);

    // [Then] Bonus is correct
    BonusEntry.SetRange("Customer No.", Customer."No.");
    BonusEntry.FindFirst();
    Assert.AreEqual(100, BonusEntry.Amount, 'Bonus incorrect');
end;
```

**Notice:**

- 3 database setups before we even test OUR code
- Testing currency and exchange rate functionality (Microsoft code)
- If exchange rate logic changes, test breaks
- Test name suggests bonus calculation, but actually tests the whole stack

## The Testing Pyramid

```
         /\
        /  \    E2E Tests (few)
       /----\
      /      \  Integration Tests (some)
     /--------\
    /          \ Unit Tests (many)
   /------------\
```

**Ideal distribution:**

- 70% Unit tests (fast, isolated)
- 20% Integration tests (key flows)
- 10% End-to-end tests (critical paths)

**Reality in BC:**

- 99% Integration tests
- 1% True unit tests

## Why BC Tests Are Usually Integration Tests

1. **Tight coupling** - Base app components are interdependent
2. **No abstraction** - Direct table access everywhere
3. **Learning from Microsoft** - Microsoft's test suite is integration tests
4. **"Simple" code fallacy** - Simple code doesn't need decoupling (wrong!)

## Converting to Unit Tests

### Step 1: Identify Dependencies

Your code depends on:

- Base app tables (Currency Exchange Rate, Customer, etc.)
- System functions (UserId(), WorkDate())
- External services (APIs, web services)

### Step 2: Abstract Dependencies

Create interfaces:

```al
interface ICurrencyConverter
{
    procedure Convert(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal): Decimal;
}
```

### Step 3: Inject Dependencies

```al
procedure Calculate(SalesAmount: Decimal; Converter: Interface ICurrencyConverter): Decimal
begin
    exit(Converter.Convert('USD', 'LCY', SalesAmount) * 0.10);
end;
```

### Step 4: Create Test Doubles

```al
codeunit 50950 "Mock Currency Converter" implements ICurrencyConverter
{
    var
        ReturnValue: Decimal;

    procedure Convert(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal): Decimal
    begin
        exit(ReturnValue);
    end;

    procedure SetReturnValue(Value: Decimal)
    begin
        ReturnValue := Value;
    end;
}
```

## Types of Test Doubles

### Dummy

Does nothing, just satisfies interface requirements:

```al
codeunit 50951 "Dummy Logger" implements ILogger
{
    procedure Log(Message: Text)
    begin
        // Does nothing
    end;
}
```

### Stub

Returns predetermined values:

```al
codeunit 50952 "Stub Permission Checker" implements IPermissionChecker
{
    var
        Allowed: Boolean;

    procedure CanPerform(Action: Text): Boolean
    begin
        exit(Allowed);
    end;

    procedure SetAllowed(Value: Boolean)
    begin
        Allowed := Value;
    end;
}
```

### Spy

Records what happened:

```al
codeunit 50953 "Spy Logger" implements ILogger
{
    var
        Invoked: Boolean;
        LastMessage: Text;

    procedure Log(Message: Text)
    begin
        Invoked := true;
        LastMessage := Message;
    end;

    procedure WasInvoked(): Boolean
    begin
        exit(Invoked);
    end;
}
```

### Mock

Verifies expected behavior:

```al
codeunit 50954 "Mock Email Sender" implements IEmailSender
{
    var
        ExpectedRecipient: Text;
        Sent: Boolean;

    procedure Send(Recipient: Text; Subject: Text; Body: Text)
    begin
        if Recipient <> ExpectedRecipient then
            Error('Expected %1, got %2', ExpectedRecipient, Recipient);
        Sent := true;
    end;
}
```

## Complete Unit Test Example

```al
codeunit 50100 "Commission Calculator Tests"
{
    Subtype = Test;

    var
        Assert: Codeunit Assert;

    [Test]
    procedure TestStandardCommission()
    var
        DummyConverter: Codeunit "Dummy Currency Converter";
        SpyLogger: Codeunit "Spy Logger";
        MockPermission: Codeunit "Mock Permission Checker";
        Calculator: Codeunit "Commission Calculator";
        Result: Decimal;
    begin
        // [Scenario] Commission is 5% of sales

        // [Given] User has permission
        MockPermission.SetAllowed(true);

        // [When] Calculating commission
        Result := Calculator.Calculate(1000, MockPermission, DummyConverter, SpyLogger);

        // [Then] Commission is 5%
        Assert.AreEqual(50, Result, 'Commission should be 5%');

        // [Then] Operation was logged
        Assert.IsTrue(SpyLogger.WasInvoked(), 'Should log commission calculation');
    end;

    [Test]
    procedure TestNoPermissionFails()
    var
        DummyConverter: Codeunit "Dummy Currency Converter";
        SpyLogger: Codeunit "Spy Logger";
        MockPermission: Codeunit "Mock Permission Checker";
        Calculator: Codeunit "Commission Calculator";
    begin
        // [Scenario] No permission causes error

        // [Given] User has no permission
        MockPermission.SetAllowed(false);

        // [When] Calculating commission
        asserterror Calculator.Calculate(1000, MockPermission, DummyConverter, SpyLogger);

        // [Then] Error raised
        Assert.ExpectedError('Permission denied');

        // [Then] Nothing was logged
        Assert.IsFalse(SpyLogger.WasInvoked(), 'Should not log when permission denied');
    end;
}
```

## When to Write Integration Tests

Integration tests are still valuable:

1. **Critical business flows** - Posting, document lifecycle
2. **Cross-module interactions** - Sales affecting inventory
3. **Permission testing** - Real permission sets
4. **Customer-facing scenarios** - User story validation

## Performance Comparison

| Metric              | Unit Tests   | Integration Tests |
| ------------------- | ------------ | ----------------- |
| 100 tests execution | < 10 seconds | 5-30 minutes      |
| Developer feedback  | Immediate    | Delayed           |
| CI pipeline impact  | Minimal      | Significant       |
| Failure isolation   | Clear        | Often unclear     |

## Best Practices

1. **Write unit tests first** - Test business logic in isolation
2. **Keep integration tests focused** - One clear scenario per test
3. **Don't duplicate** - If unit tested, don't integration test same logic
4. **Decouple for testability** - Design code to be unit-testable
5. **Measure test time** - Track and optimize slow tests
6. **Use test doubles** - Interfaces + mocks = fast tests

## Further Reading

- [TESTABILITY.md](TESTABILITY.md) - Designing code for testability
- [PATTERNS.md](PATTERNS.md) - ATDD patterns for both test types
- [ISOLATION.md](ISOLATION.md) - Transaction handling for integration tests
