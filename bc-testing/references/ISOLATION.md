# Test Isolation

Test isolation ensures tests are independent and don't affect each other. Proper isolation prevents flaky tests and enables running tests in any order.

## Why Isolation Matters

Without proper isolation:

- Tests may pass/fail depending on execution order
- One failing test can cause cascading failures
- Debugging becomes difficult
- Parallel execution is impossible

## TestIsolation Property

Set on **test runner codeunits** (SubType = TestRunner):

```al
codeunit 50200 "My Test Runner"
{
    Subtype = TestRunner;
    TestIsolation = Function; // Roll back after each test method

    trigger OnRun()
    begin
        Codeunit.Run(Codeunit::"Sales Tests");
        Codeunit.Run(Codeunit::"Purchase Tests");
    end;
}
```

### TestIsolation Values

| Value      | Rollback Behavior                                        |
| ---------- | -------------------------------------------------------- |
| `Disabled` | No rollback (default). Tests must clean up manually      |
| `Codeunit` | Roll back all changes after each test codeunit completes |
| `Function` | Roll back all changes after each test method completes   |

### Choosing TestIsolation

| Scenario                | Recommended               |
| ----------------------- | ------------------------- |
| Fast, independent tests | `Function`                |
| Tests that share setup  | `Codeunit`                |
| Performance testing     | `Disabled`                |
| Legacy tests            | `Disabled` (then migrate) |

## TransactionModel Attribute

Set on **individual test methods** to control transaction behavior:

```al
[Test]
[TransactionModel(TransactionModel::AutoRollback)]
procedure TestWithAutoRollback()
begin
    // All changes rolled back after test
end;

[Test]
[TransactionModel(TransactionModel::AutoCommit)]
procedure TestWithCommit()
begin
    // Changes committed; use when code calls COMMIT
end;

[Test]
[TransactionModel(TransactionModel::None)]
procedure TestWithNoTransaction()
begin
    // Each trigger commits separately; simulates real user
end;
```

### TransactionModel Values

| Value          | Transaction Started | COMMIT Allowed             | Rollback                            |
| -------------- | ------------------- | -------------------------- | ----------------------------------- |
| `AutoRollback` | Yes, by test method | No (error if called)       | Automatic                           |
| `AutoCommit`   | Yes, by test method | Yes                        | Must be manual or use TestIsolation |
| `None`         | No                  | Yes (each trigger commits) | Must be manual or use TestIsolation |

### When to Use Each TransactionModel

**AutoRollback** (default, most common):

- Code doesn't call COMMIT
- Want automatic cleanup
- Standard business logic tests

```al
[Test]
[TransactionModel(TransactionModel::AutoRollback)]
procedure TestStandardOperation()
begin
    // Most tests use this
    CreateSalesOrder();
    // Automatically rolled back
end;
```

**AutoCommit**:

- Code explicitly calls COMMIT
- Testing transaction boundaries
- Testing background jobs

```al
[Test]
[TransactionModel(TransactionModel::AutoCommit)]
procedure TestWithExplicitCommit()
begin
    // Code calls COMMIT internally
    RunBatchProcessWithCommit();
    // Must clean up manually or use TestIsolation
end;
```

**None**:

- Want to simulate real user behavior
- Each page field validation commits separately
- Read-only tests

```al
[Test]
[TransactionModel(TransactionModel::None)]
procedure TestReadOnlyOperation()
begin
    // Good for calculation-only tests
    ValidateFormula();
end;
```

## RequiredTestIsolation Property (Runtime 16+)

Set on **test codeunits** to require a minimum isolation level:

```al
codeunit 50100 "Sales Tests"
{
    Subtype = Test;
    RequiredTestIsolation = Function; // Requires Function or Codeunit isolation
}
```

If the test runner doesn't meet the requirement, tests will fail with an error.

## Combining Properties

TransactionModel and TestIsolation work together:

| TransactionModel | TestIsolation | Result                                               |
| ---------------- | ------------- | ---------------------------------------------------- |
| AutoRollback     | Any           | Test rolls back changes (fastest)                    |
| AutoCommit       | Disabled      | Changes persist; manual cleanup needed               |
| AutoCommit       | Function      | Runner rolls back after each method                  |
| AutoCommit       | Codeunit      | Runner rolls back after each codeunit                |
| None             | Function      | Each trigger commits, runner rolls back after method |

## Initialize with Commit Pattern

For complex setup that should persist across tests within a codeunit:

```al
local procedure Initialize()
begin
    CleanupTestData(); // Always clean up between tests

    if IsInitialized then
        exit;

    // One-time setup
    SetupRequiredConfiguration();

    IsInitialized := true;
    Commit(); // Commit setup so test transaction can roll back cleanly
end;
```

The `Commit()` at end of Initialize:

1. Separates setup from test transaction
2. Allows `AutoRollback` tests to roll back only their changes
3. Setup remains for subsequent tests

## Manual Cleanup Pattern

When TestIsolation is Disabled:

```al
local procedure Initialize()
begin
    // Clean up before each test
    CleanupSalesOrders();
    CleanupCustomers();

    if IsInitialized then
        exit;

    SetupConfiguration();
    IsInitialized := true;
end;

local procedure CleanupSalesOrders()
var
    SalesHeader: Record "Sales Header";
begin
    SalesHeader.SetRange("Document Type", SalesHeader."Document Type"::Order);
    SalesHeader.DeleteAll(true);
end;
```

## Event Subscription Isolation

When using event subscribers for mocking:

```al
codeunit 50102 "Mock External Service"
{
    EventSubscriberInstance = Manual;

    [EventSubscriber(ObjectType::Codeunit, Codeunit::"External Service", 'OnCallService', '', false, false)]
    procedure MockServiceCall(var Response: Text)
    begin
        Response := '{"status": "success"}';
    end;
}

codeunit 50100 "My Tests"
{
    Subtype = Test;

    var
        MockService: Codeunit "Mock External Service";

    local procedure Initialize()
    begin
        UnbindSubscription(MockService); // Clean up from previous
        BindSubscription(MockService);   // Bind for this test
    end;

    [Test]
    procedure TestWithMock()
    begin
        Initialize();
        // MockService now handles external calls
    end;
}
```

## Database State Considerations

### Temporary Tables

Use temporary tables when possible to avoid database changes:

```al
procedure TestWithTempTable()
var
    TempItem: Record Item temporary;
begin
    // No database changes
    TempItem.Init();
    TempItem."No." := 'TEST';
    TempItem.Insert();

    // Test with temp record
    ProcessItem(TempItem);
end;
```

### CRONUS Demo Data

Tests may depend on CRONUS demo data:

```al
procedure TestWithDemoData()
var
    Customer: Record Customer;
begin
    // Assumes CRONUS demo data exists
    Customer.Get('10000');
    Assert.AreEqual('The Cannon Group PLC', Customer.Name, 'Expected CRONUS customer');
end;
```

Better approach - create test data:

```al
procedure TestWithCreatedData()
var
    Customer: Record Customer;
begin
    // Creates own data, independent of demo company
    LibrarySales.CreateCustomer(Customer);
    // ... test with Customer
end;
```

## Best Practices

1. **Default to AutoRollback**: Most tests don't need COMMIT
2. **Use TestIsolation = Function**: Highest isolation, catch more issues
3. **Always Initialize**: Reset state at start of each test
4. **Clean up test-specific data**: Don't rely solely on rollback
5. **Avoid global state**: Tests shouldn't modify setup tables permanently
6. **Unbind subscriptions**: In Initialize before binding new ones
7. **Use temporary tables**: When real database changes aren't needed
8. **Test in isolation first**: Then test integration

## Common Issues

### Error: "COMMIT not allowed"

```al
// Code calls COMMIT but test uses AutoRollback
[Test]
[TransactionModel(TransactionModel::AutoRollback)]
procedure TestBatchJob()
begin
    RunBatchJobThatCommits(); // ERROR!
end;

// Fix: Use AutoCommit
[Test]
[TransactionModel(TransactionModel::AutoCommit)]
procedure TestBatchJob()
begin
    RunBatchJobThatCommits(); // OK
end;
```

### Tests pass individually, fail together

```al
// Bad: Test leaves data behind
[Test]
procedure Test1()
begin
    CreateCustomerWithNo('TEST'); // Creates customer 'TEST'
end;

[Test]
procedure Test2()
begin
    CreateCustomerWithNo('TEST'); // Fails - 'TEST' already exists!
end;

// Fix: Clean up in Initialize
local procedure Initialize()
begin
    DeleteTestCustomers();
end;
```

### Flaky tests due to timing

```al
// Risky: Depends on clock
[Test]
procedure TestWithTimestamp()
begin
    Record."Created At" := CurrentDateTime;
    // ... later
    Assert.AreEqual(CurrentDateTime, Record."Created At", 'Wrong time');
end;

// Better: Control the time
[Test]
procedure TestWithControlledTime()
var
    ExpectedTime: DateTime;
begin
    ExpectedTime := CreateDateTime(Today, 120000T);
    Record."Created At" := ExpectedTime;
    Assert.AreEqual(ExpectedTime, Record."Created At", 'Wrong time');
end;
```
