# Test Codeunits and Test Methods

This document covers creating and configuring test codeunits, test methods, and test runner codeunits in AL for Business Central.

## Test Codeunit Setup

A test codeunit must have `Subtype = Test`:

```al
codeunit 50100 "Sales Order Tests"
{
    Subtype = Test;
    TestPermissions = Disabled; // or NonRestrictive, Restrictive

    // Test methods go here
}
```

### Properties

| Property                | Values                                                               | Description                               |
| ----------------------- | -------------------------------------------------------------------- | ----------------------------------------- |
| `Subtype`               | `Test`                                                               | Required. Marks codeunit as test codeunit |
| `TestPermissions`       | `Disabled`, `NonRestrictive`, `Restrictive`, `InheritFromTestRunner` | Controls permission testing               |
| `RequiredTestIsolation` | `Disabled`, `Codeunit`, `Function`                                   | Runtime 16+. Required isolation level     |

## Test Method Attributes

### [Test] Attribute

Marks a method as a test method:

```al
[Test]
procedure TestSalesOrderPosting()
begin
    // Test code
end;
```

### [HandlerFunctions] Attribute

Declares UI handlers used by the test. **Every declared handler must be called.**

```al
[Test]
[HandlerFunctions('MessageHandler,ConfirmHandler')]
procedure TestWithDialogs()
begin
    // Code that triggers Message and Confirm dialogs
end;

[MessageHandler]
procedure MessageHandler(Message: Text[1024])
begin
    // Handle message
end;

[ConfirmHandler]
procedure ConfirmHandler(Question: Text[1024]; var Reply: Boolean)
begin
    Reply := true;
end;
```

### [TransactionModel] Attribute

Controls transaction behavior for individual test methods:

```al
[Test]
[TransactionModel(TransactionModel::AutoRollback)]
procedure TestWithAutoRollback()
begin
    // Changes rolled back after test
end;
```

| Value          | Behavior                                                        |
| -------------- | --------------------------------------------------------------- |
| `AutoRollback` | Roll back all changes after test. Cannot use COMMIT             |
| `AutoCommit`   | Commit changes. Use when code calls COMMIT                      |
| `None`         | No transaction started by test. Each trigger commits separately |

### [Scope] Attribute

```al
[Test]
[Scope('OnPrem')]
procedure TestOnPremOnly()
begin
    // Only runs on-premises
end;
```

## Test Method Types

A test codeunit can contain three types of methods:

| Type    | Purpose               | Attribute                                    |
| ------- | --------------------- | -------------------------------------------- |
| Test    | Contains test logic   | `[Test]`                                     |
| Handler | Handles UI interrupts | `[MessageHandler]`, `[ConfirmHandler]`, etc. |
| Normal  | Helper/setup methods  | `[Normal]` (default, can be omitted)         |

## Test Method Naming

Use descriptive names that explain the scenario:

```al
// Good - describes scenario
[Test]
procedure TestCustomerHasGoldLevelAfterFourOrders()

// Good - describes expected behavior
[Test]
procedure TestPostingSalesOrderUpdatesInventory()

// Bad - too vague
[Test]
procedure Test1()

// Bad - doesn't describe outcome
[Test]
procedure TestSalesOrder()
```

## Test Execution Behavior

When a test codeunit runs:

1. `OnRun` trigger executes first
2. Each `[Test]` method runs in sequence
3. Each test runs in a separate transaction (by default)
4. If one test fails, remaining tests still execute
5. Results are SUCCESS or FAILURE per method

```al
codeunit 50100 "My Tests"
{
    Subtype = Test;

    trigger OnRun()
    begin
        // Runs before any tests (not recommended for setup)
    end;

    [Test]
    procedure Test1() // Runs second
    begin
    end;

    [Test]
    procedure Test2() // Runs third, even if Test1 fails
    begin
    end;
}
```

## Test Runner Codeunits

Test runners manage test execution and integrate with test frameworks:

```al
codeunit 50200 "My Test Runner"
{
    Subtype = TestRunner;
    TestIsolation = Codeunit; // or Function, Disabled

    trigger OnRun()
    begin
        // Run specific test codeunits
        Codeunit.Run(Codeunit::"Sales Order Tests");
        Codeunit.Run(Codeunit::"Purchase Order Tests");
    end;

    trigger OnBeforeTestRun(CodeunitId: Integer; CodeunitName: Text; FunctionName: Text; FunctionTestPermissions: TestPermissions): Boolean
    begin
        // Called before each test
        // Return false to skip the test
        exit(true);
    end;

    trigger OnAfterTestRun(CodeunitId: Integer; CodeunitName: Text; FunctionName: Text; FunctionTestPermissions: TestPermissions; IsSuccess: Boolean)
    begin
        // Called after each test
        // Log results here
        if not IsSuccess then
            LogTestFailure(CodeunitName, FunctionName);
    end;
}
```

### TestIsolation Property

Set on test runner codeunits to control rollback:

| Value      | Behavior                           |
| ---------- | ---------------------------------- |
| `Disabled` | No automatic rollback (default)    |
| `Codeunit` | Roll back after each test codeunit |
| `Function` | Roll back after each test method   |

## Initialize Pattern

Always use an Initialize procedure for test setup:

```al
codeunit 50100 "Sales Tests"
{
    Subtype = Test;

    var
        IsInitialized: Boolean;
        LibrarySales: Codeunit "Library - Sales";
        LibraryRandom: Codeunit "Library - Random";

    local procedure Initialize()
    var
        SalesHeader: Record "Sales Header";
    begin
        // Clean up from previous tests
        SalesHeader.DeleteAll();

        if IsInitialized then
            exit;

        // One-time setup
        LibraryRandom.SetSeed(1);

        IsInitialized := true;
        Commit(); // Commit setup so test can roll back only test changes
    end;

    [Test]
    procedure TestSalesOrderCreation()
    begin
        // [Given] System is initialized
        Initialize();

        // [When] Create sales order
        // ...
    end;
}
```

## Running Tests

### From VS Code

Press `Ctrl+F5` to publish and open Business Central, then:

1. Open Test Tool page (130401)
2. Select "Get Test Codeunits" > "Select Test Codeunits"
3. Choose your test codeunits
4. Click "Run" or "Run Selected"

### From Test Tool Page

```al
// Navigate to page 130401 "Test Tool"
// Or run in AL:
Page.Run(Page::"Test Tool");
```

## Dependencies

Test projects need dependencies on test libraries in `app.json`:

```json
{
  "dependencies": [
    {
      "id": "dd0be2ea-f733-4d65-bb34-a28f4624fb14",
      "name": "Library Assert",
      "publisher": "Microsoft",
      "version": "19.0.0.0"
    },
    {
      "id": "5d86850b-0d76-4eca-bd7b-951ad998e997",
      "name": "Tests-TestLibraries",
      "publisher": "Microsoft",
      "version": "19.0.0.0"
    }
  ]
}
```

## Common Mistakes

### Mistake: Handler not called

```al
// ERROR: ConfirmHandler declared but not invoked
[Test]
[HandlerFunctions('ConfirmHandler')]
procedure TestWithoutConfirm()
begin
    // Code that doesn't trigger Confirm
end;
```

### Mistake: Missing handler

```al
// ERROR: Code triggers Message but no MessageHandler declared
[Test]
procedure TestThatShowsMessage()
begin
    Message('Hello'); // Fails - unhandled UI
end;
```

### Mistake: Using OnRun for setup

```al
// BAD: OnRun runs once, not before each test
trigger OnRun()
begin
    SetupTestData(); // Won't reset between tests
end;

// GOOD: Use Initialize pattern
local procedure Initialize()
begin
    SetupTestData();
end;
```
