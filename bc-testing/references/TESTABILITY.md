# Testability: Designing Code for Testing

Testability is not about tests—it's about how your **production code** is designed. Code that's hard to test is hard to maintain.

## The Testability Problem

Consider this code:

```al
procedure ConvertCurrency(FromAmount: Decimal; FromCurrency: Code[10]; ToCurrency: Code[10]) Result: Decimal
var
    ExchRate: Record "Currency Exchange Rate";
begin
    exit(ExchRate.ExchangeAmtFCYToFCY(WorkDate(), FromCurrency, ToCurrency, FromAmount));
end;
```

To test this, you must:

1. Create currencies in database
2. Create exchange rates in database
3. Run the test
4. Hope exchange rate math is as you expect

**You're not testing YOUR code—you're testing Microsoft's code.**

## The Solution: Abstraction

Testable code separates **what** from **how**:

```al
// What - the interface
interface ICurrencyConverter
{
    procedure Convert(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal): Decimal;
}

// How - production implementation
codeunit 50100 "BC Currency Converter" implements ICurrencyConverter
{
    procedure Convert(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal): Decimal
    var
        ExchRate: Record "Currency Exchange Rate";
    begin
        exit(ExchRate.ExchangeAmtFCYToFCY(WorkDate(), FromCurrency, ToCurrency, Amount));
    end;
}

// How - test implementation
codeunit 50950 "Test Currency Converter" implements ICurrencyConverter
{
    var
        FixedRate: Decimal;

    procedure Convert(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal): Decimal
    begin
        exit(Amount * FixedRate);
    end;

    procedure SetRate(Rate: Decimal)
    begin
        FixedRate := Rate;
    end;
}
```

## Dependency Injection

Instead of creating dependencies inside code, inject them:

### Before (Tight Coupling)

```al
procedure ProcessOrder(OrderNo: Code[20])
var
    Permission: Record "Order Permission";
    Logger: Codeunit "Order Logger";
    Converter: Record "Currency Exchange Rate";
begin
    // Code directly depends on specific implementations
    if not Permission.UserCanProcess(UserId()) then
        Error('No permission');

    // Process order with converter
    // Log with logger
end;
```

### After (Loose Coupling)

```al
procedure ProcessOrder(
    OrderNo: Code[20];
    PermissionChecker: Interface IPermissionChecker;
    Converter: Interface ICurrencyConverter;
    Logger: Interface ILogger
)
begin
    if not PermissionChecker.CanProcess(UserId()) then
        Error('No permission');

    // Process using injected dependencies
end;

// Keep existing signature for backward compatibility
procedure ProcessOrder(OrderNo: Code[20])
var
    DefaultPermission: Codeunit "BC Permission Checker";
    DefaultConverter: Codeunit "BC Currency Converter";
    DefaultLogger: Codeunit "BC Order Logger";
begin
    ProcessOrder(OrderNo, DefaultPermission, DefaultConverter, DefaultLogger);
end;
```

## Identifying Dependencies

Dependencies are **anything your code relies on** that:

- Accesses the database
- Calls external services
- Uses system functions (UserId, WorkDate, etc.)
- Involves complex setup

### Common Dependencies in BC

| Dependency           | Type     | Testability Issue       |
| -------------------- | -------- | ----------------------- |
| Tables               | Database | Requires setup data     |
| Codeunits (Base App) | Logic    | Brings own requirements |
| External APIs        | Network  | Unavailable in tests    |
| Users/Permissions    | System   | Can't control in tests  |
| WorkDate/Today       | System   | Unpredictable           |
| Number Series        | Database | Requires setup          |

## Creating Interfaces

### Step 1: Define the Contract

```al
interface IPermissionChecker
{
    procedure CanConvert(FromCurrency: Code[10]; ToCurrency: Code[10]; User: Text[50]): Boolean;
}

interface ILogger
{
    procedure Log(Message: Text);
    procedure LogConversion(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal; Result: Decimal);
}

interface IDateProvider
{
    procedure GetWorkDate(): Date;
    procedure GetToday(): Date;
}
```

### Step 2: Production Implementations

```al
codeunit 50101 "DB Permission Checker" implements IPermissionChecker
{
    procedure CanConvert(FromCurrency: Code[10]; ToCurrency: Code[10]; User: Text[50]): Boolean
    var
        Permission: Record "Currency Exchange Permission";
    begin
        Permission.SetRange("User ID", User);
        Permission.SetFilter("From Currency", '%1|%2', '', FromCurrency);
        Permission.SetFilter("To Currency", '%1|%2', '', ToCurrency);
        exit(not Permission.IsEmpty());
    end;
}

codeunit 50102 "DB Logger" implements ILogger
{
    procedure Log(Message: Text)
    var
        LogEntry: Record "Application Log";
    begin
        LogEntry."Entry No." := 0;
        LogEntry.Message := CopyStr(Message, 1, MaxStrLen(LogEntry.Message));
        LogEntry."Created At" := CurrentDateTime();
        LogEntry.Insert(true);
    end;

    procedure LogConversion(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal; Result: Decimal)
    begin
        Log(StrSubstNo('Converted %1 %2 to %3 %4', Amount, FromCurrency, Result, ToCurrency));
    end;
}
```

### Step 3: Test Implementations

```al
codeunit 50951 "Mock Permission Checker" implements IPermissionChecker
{
    var
        IsAllowed: Boolean;

    procedure CanConvert(FromCurrency: Code[10]; ToCurrency: Code[10]; User: Text[50]): Boolean
    begin
        exit(IsAllowed);
    end;

    procedure SetAllowed(Allowed: Boolean)
    begin
        IsAllowed := Allowed;
    end;
}

codeunit 50952 "Spy Logger" implements ILogger
{
    var
        WasLogCalled: Boolean;
        LogMessages: List of [Text];

    procedure Log(Message: Text)
    begin
        WasLogCalled := true;
        LogMessages.Add(Message);
    end;

    procedure LogConversion(FromCurrency: Code[10]; ToCurrency: Code[10]; Amount: Decimal; Result: Decimal)
    begin
        Log(StrSubstNo('Converted %1 %2 to %3 %4', Amount, FromCurrency, Result, ToCurrency));
    end;

    procedure WasInvoked(): Boolean
    begin
        exit(WasLogCalled);
    end;

    procedure GetMessageCount(): Integer
    begin
        exit(LogMessages.Count());
    end;
}
```

## The Business Logic Layer

Separate business logic from data access:

### Before (Mixed Concerns)

```al
procedure CalculateCommission(CustomerNo: Code[20]): Decimal
var
    CustLedger: Record "Cust. Ledger Entry";
    CommissionSetup: Record "Commission Setup";
    Total: Decimal;
begin
    // Data access mixed with business logic
    CustLedger.SetRange("Customer No.", CustomerNo);
    CustLedger.SetRange("Document Type", CustLedger."Document Type"::Invoice);
    CustLedger.CalcSums(Amount);
    Total := CustLedger.Amount;

    CommissionSetup.Get();

    if Total > CommissionSetup."High Threshold" then
        exit(Total * CommissionSetup."High Rate" / 100)
    else
        exit(Total * CommissionSetup."Low Rate" / 100);
end;
```

### After (Separated Concerns)

```al
// Data access layer
codeunit 50110 "Customer Sales Data"
{
    procedure GetTotalInvoicedAmount(CustomerNo: Code[20]): Decimal
    var
        CustLedger: Record "Cust. Ledger Entry";
    begin
        CustLedger.SetRange("Customer No.", CustomerNo);
        CustLedger.SetRange("Document Type", CustLedger."Document Type"::Invoice);
        CustLedger.CalcSums(Amount);
        exit(CustLedger.Amount);
    end;
}

// Business logic layer (testable without database)
codeunit 50111 "Commission Calculator"
{
    procedure Calculate(TotalSales: Decimal; HighThreshold: Decimal; HighRate: Decimal; LowRate: Decimal): Decimal
    begin
        if TotalSales > HighThreshold then
            exit(TotalSales * HighRate / 100)
        else
            exit(TotalSales * LowRate / 100);
    end;
}

// Unit test - no database needed
[Test]
procedure TestHighCommissionRate()
var
    Calculator: Codeunit "Commission Calculator";
    Result: Decimal;
begin
    // [Scenario] High sales get high rate

    // [When] Sales above threshold
    Result := Calculator.Calculate(10000, 5000, 10, 5);

    // [Then] High rate applied
    Assert.AreEqual(1000, Result, 'Should be 10% of 10000');
end;
```

## Event-Based Decoupling

When interfaces aren't possible, use events:

### In Production Code

```al
codeunit 50120 "Order Processor"
{
    var
        OnBeforeProcessOrder: Codeunit "Order Processor Events";

    procedure Process(OrderNo: Code[20])
    var
        IsHandled: Boolean;
    begin
        OnBeforeProcessOrder.OnBeforeProcess(OrderNo, IsHandled);
        if IsHandled then
            exit;

        // Standard processing
    end;
}

codeunit 50121 "Order Processor Events"
{
    [IntegrationEvent(false, false)]
    procedure OnBeforeProcess(OrderNo: Code[20]; var IsHandled: Boolean)
    begin
    end;
}
```

### In Test Code

```al
codeunit 50960 "Order Test Subscriber"
{
    EventSubscriberInstance = Manual;

    var
        ShouldHandle: Boolean;

    [EventSubscriber(ObjectType::Codeunit, Codeunit::"Order Processor Events", 'OnBeforeProcess', '', false, false)]
    local procedure HandleOnBeforeProcess(OrderNo: Code[20]; var IsHandled: Boolean)
    begin
        IsHandled := ShouldHandle;
    end;

    procedure SetShouldHandle(Value: Boolean)
    begin
        ShouldHandle := Value;
    end;
}

// Test
[Test]
procedure TestOrderSkippedWhenHandled()
var
    Subscriber: Codeunit "Order Test Subscriber";
    Processor: Codeunit "Order Processor";
begin
    // [Given] Event handler will handle
    Subscriber.SetShouldHandle(true);
    BindSubscription(Subscriber);

    // [When] Processing order
    Processor.Process('ORD001');

    // [Then] Standard processing skipped
    // ... assertions

    UnbindSubscription(Subscriber);
end;
```

## Design Principles for Testability

### 1. Single Responsibility

Each component does ONE thing:

```al
// Bad - does too much
codeunit 50130 "Sales Everything"
{
    procedure CreateSalesOrderAndPost(...)
    // Creates order, validates, posts, sends email, updates CRM
}

// Good - single purpose
codeunit 50131 "Sales Order Creator"
codeunit 50132 "Sales Order Validator"
codeunit 50133 "Sales Order Poster"
codeunit 50134 "Sales Notification Sender"
```

### 2. Don't Mix Database Access with Logic

```al
// Bad
procedure CalculateDiscount(): Decimal
var
    Customer: Record Customer;
begin
    Customer.Get(); // Database access
    if Customer."Total Sales" > 10000 then // Logic
        exit(0.10)
    else
        exit(0.05);
end;

// Good - separate
procedure GetBaseDiscount(TotalSales: Decimal): Decimal
begin
    if TotalSales > 10000 then
        exit(0.10)
    else
        exit(0.05);
end;
```

### 3. Prefer Composition Over Inheritance

```al
// Less testable - inheritance
codeunit 50140 "Special Order Processor" extends "Order Processor"

// More testable - composition
codeunit 50141 "Order Processor"
{
    procedure Process(Order: Record "Sales Order"; Validator: Interface IOrderValidator)
    begin
        Validator.Validate(Order);
        // Continue processing
    end;
}
```

### 4. Make Dependencies Explicit

```al
// Hidden dependency - hard to test
procedure Calculate()
var
    Setup: Record "My Setup";
begin
    Setup.Get(); // Where does this come from?
end;

// Explicit dependency - easy to test
procedure Calculate(SetupRate: Decimal)
begin
    // Clear what it needs
end;
```

## Benefits of Testable Code

| Benefit          | Impact                                       |
| ---------------- | -------------------------------------------- |
| Fast tests       | Developer feedback in seconds                |
| Reliable tests   | No flaky failures                            |
| Easy maintenance | Change implementation without changing tests |
| Clear design     | Code structure reflects business logic       |
| Documentation    | Tests explain what code does                 |
| Confidence       | Deploy without fear                          |

## Migration Strategy

1. **Start with new code** - Apply principles to new features
2. **Refactor on touch** - When modifying, improve testability
3. **Identify hotspots** - Focus on frequently failing code
4. **Create seams** - Add interfaces at integration points
5. **Extract logic** - Move calculations to testable units

## Further Reading

- [TEST-TYPES.md](TEST-TYPES.md) - Unit vs integration tests
- [PATTERNS.md](PATTERNS.md) - ATDD documentation patterns
- [ISOLATION.md](ISOLATION.md) - Test transaction handling
