# ATDD Patterns and Test Documentation

Acceptance Test-Driven Development (ATDD) patterns make tests readable, maintainable, and serve as living documentation.

## ATDD Overview

ATDD structures tests using:

- **Given**: Setup / preconditions
- **When**: Action under test
- **Then**: Expected outcome / verification

This creates tests that read like specifications.

## Comment Tags

### [FEATURE] Tag

Defines the feature area for the entire test codeunit:

```al
codeunit 50100 "Customer Rewards Tests"
{
    // [FEATURE] [Customer Rewards]

    Subtype = Test;
}
```

Multiple features:

```al
// [FEATURE] [Customer Rewards] [Loyalty Program]
```

### [Scenario] Tag

Describes the specific test scenario with optional work item ID:

```al
[Test]
procedure TestCustomerGetsGoldLevelAfterFourOrders()
begin
    // [Scenario 12345] Customer achieves Gold level after placing four orders
end;
```

The ScenarioID (12345) links to a work item in Azure DevOps, Jira, etc.

### [Given] Tag

Documents preconditions and setup:

```al
[Test]
procedure TestSalesOrderPosting()
begin
    // [Scenario] Posted sales order creates invoice

    // [Given] A customer exists
    Initialize();
    LibrarySales.CreateCustomer(Customer);

    // [Given] A sales order with one line
    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Order, Customer."No.");
    LibrarySales.CreateSalesLine(SalesLine, SalesHeader, SalesLine.Type::Item, '', 1);
end;
```

Multiple GIVENs for complex setup:

```al
// [Given] Customer Rewards is activated
ActivateCustomerRewards();

// [Given] Bronze level requires 2 points
AddRewardLevel('BRONZE', 2);

// [Given] Silver level requires 5 points
AddRewardLevel('SILVER', 5);
```

### [When] Tag

Documents the action being tested. **Only one WHEN per test**:

```al
// [When] Post the sales order
LibrarySales.PostSalesDocument(SalesHeader, true, true);
```

If you need multiple WHENs, split into separate tests:

```al
// BAD: Multiple actions in one test
[Test]
procedure TestCreateAndPost()
begin
    // [When] Create order
    CreateOrder();
    // [When] Post order
    PostOrder();
end;

// GOOD: Separate tests
[Test]
procedure TestCreateOrder()
begin
    // [When] Create order
end;

[Test]
procedure TestPostOrder()
begin
    // [When] Post existing order
end;
```

### [Then] Tag

Documents expected outcomes. Multiple THENs allowed:

```al
// [Then] Invoice is created
SalesInvoiceHeader.SetRange("Sell-to Customer No.", Customer."No.");
Assert.RecordIsNotEmpty(SalesInvoiceHeader);

// [Then] Order is deleted
Assert.IsFalse(SalesHeader.Get(SalesHeader."Document Type", SalesHeader."No."), 'Order should be deleted');

// [Then] Customer balance is updated
Customer.CalcFields("Balance (LCY)");
Assert.AreEqual(ExpectedBalance, Customer."Balance (LCY)", 'Wrong balance');
```

## Complete Test Structure

```al
[Test]
[HandlerFunctions('ConfirmHandler')]
procedure TestCustomerAchievesBronzeLevelAfterTwoOrders()
var
    Customer: Record Customer;
    SalesHeader: Record "Sales Header";
    CustomerCard: TestPage "Customer Card";
begin
    // [Scenario 45678] Customer gets Bronze level after two orders
    // [FEATURE] [Customer Rewards]

    // [Given] Customer Rewards is activated with Bronze at 2 points
    Initialize();
    ActivateCustomerRewards();
    AddRewardLevel('BRONZE', 2);

    // [Given] A new customer with zero points
    LibrarySales.CreateCustomer(Customer);

    // [When] Customer places two orders
    CreateAndPostSalesOrder(Customer."No.");
    CreateAndPostSalesOrder(Customer."No.");

    // [Then] Customer has 2 reward points
    CustomerCard.OpenView();
    CustomerCard.GoToRecord(Customer);
    Assert.AreEqual(2, CustomerCard.RewardPoints.AsInteger(), 'Should have 2 points');

    // [Then] Customer has Bronze level
    Assert.AreEqual('BRONZE', CustomerCard.RewardLevel.Value, 'Should have Bronze level');
end;
```

## Test Method Naming

Method names should describe the scenario:

```al
// Pattern: Test[Feature][Scenario][ExpectedOutcome]

// Good names - describe what is tested
procedure TestPostingSalesOrderCreatesInvoice()
procedure TestCustomerCannotOrderWhenBlocked()
procedure TestDiscountAppliesWhenQuantityExceedsTen()
procedure TestErrorWhenPostingWithoutLines()

// Bad names - too vague
procedure Test1()
procedure TestSalesOrder()
procedure TestPosting()
```

### Naming for Negative Tests

```al
// Pattern: Test[Scenario]ErrorsWhen[Condition]
procedure TestPostingErrorsWhenCustomerBlocked()
procedure TestValidationFailsWhenAmountNegative()
procedure TestSaveRejectsEmptyDescription()
```

## Helper Method Patterns

### Create Helpers for Setup

```al
local procedure CreateSalesOrderWithAmount(var SalesHeader: Record "Sales Header"; CustomerNo: Code[20]; Amount: Decimal)
var
    SalesLine: Record "Sales Line";
begin
    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Order, CustomerNo);
    LibrarySales.CreateSalesLine(SalesLine, SalesHeader, SalesLine.Type::"G/L Account", '', 1);
    SalesLine.Validate("Unit Price", Amount);
    SalesLine.Modify(true);
end;
```

### Create Helpers for Verification

```al
local procedure VerifyCustomerBalance(CustomerNo: Code[20]; ExpectedBalance: Decimal)
var
    Customer: Record Customer;
begin
    Customer.Get(CustomerNo);
    Customer.CalcFields("Balance (LCY)");
    Assert.AreEqual(
        ExpectedBalance,
        Customer."Balance (LCY)",
        StrSubstNo('Customer %1 should have balance %2', CustomerNo, ExpectedBalance)
    );
end;
```

### Action Helpers

```al
local procedure PostSalesOrderAndGetInvoice(var SalesHeader: Record "Sales Header"; var SalesInvoiceHeader: Record "Sales Invoice Header")
var
    SalesPost: Codeunit "Sales-Post";
begin
    SalesPost.Run(SalesHeader);
    SalesInvoiceHeader.SetRange("Pre-Assigned No.", SalesHeader."No.");
    SalesInvoiceHeader.FindFirst();
end;
```

## Test Codeunit Organization

```al
codeunit 50100 "Sales Order Tests"
{
    // [FEATURE] [Sales] [Orders]

    Subtype = Test;
    TestPermissions = Disabled;

    var
        Assert: Codeunit Assert;
        LibrarySales: Codeunit "Library - Sales";
        IsInitialized: Boolean;

    // ==================== Tests ====================

    [Test]
    procedure TestCreateSalesOrder()
    begin
        // Test implementation
    end;

    [Test]
    procedure TestPostSalesOrder()
    begin
        // Test implementation
    end;

    // ==================== Negative Tests ====================

    [Test]
    procedure TestCannotPostOrderWithoutLines()
    begin
        // Test implementation
    end;

    // ==================== Handlers ====================

    [MessageHandler]
    procedure MessageHandler(Message: Text[1024])
    begin
    end;

    [ConfirmHandler]
    procedure ConfirmHandler(Question: Text[1024]; var Reply: Boolean)
    begin
        Reply := true;
    end;

    // ==================== Helper Methods ====================

    local procedure Initialize()
    begin
        // Setup
    end;

    local procedure CreateSalesOrderWithLine(var SalesHeader: Record "Sales Header")
    begin
        // Helper
    end;

    // ==================== Verification ====================

    local procedure VerifyInvoiceCreated(SalesHeaderNo: Code[20])
    begin
        // Verification helper
    end;
}
```

## User Story Traceability

Link tests to user stories:

```al
// User Story: As a sales manager, I want customers to earn reward points
// so that I can track their loyalty.

codeunit 50100 "Customer Rewards Tests"
{
    // [FEATURE] [Customer Rewards]

    [Test]
    procedure TestCustomerEarnsPointsOnSalesOrder()
    begin
        // [Scenario US-123] Customer earns 1 point per sales order
    end;

    [Test]
    procedure TestCustomerSeesPointsOnCard()
    begin
        // [Scenario US-124] Customer card displays reward points
    end;
}
```

## Documentation Benefits

Well-documented tests serve as:

1. **Specifications**: Define expected behavior
2. **Living documentation**: Always current with code
3. **Examples**: Show how to use features
4. **Regression detection**: Catch behavior changes
5. **Onboarding**: Help new developers understand features

## Anti-Patterns

### Missing Documentation

```al
// BAD: No context
[Test]
procedure Test1()
begin
    Initialize();
    DoSomething();
    Assert.IsTrue(result, '');
end;

// GOOD: Clear documentation
[Test]
procedure TestCustomerBalanceUpdatedAfterPosting()
begin
    // [Scenario] Posting invoice updates customer balance
    // [Given] A customer with zero balance
    // [When] Post sales invoice for $100
    // [Then] Customer balance is $100
end;
```

### Too Many Assertions

```al
// BAD: Testing too much
[Test]
procedure TestEverything()
begin
    // [When] Create and post order
    CreateAndPostOrder();

    // [Then] 50 different verifications...
end;

// GOOD: Focused test
[Test]
procedure TestPostingCreatesInvoice()
begin
    // [When] Post order
    PostOrder();

    // [Then] Invoice created
    VerifyInvoiceCreated();
end;
```

### Implicit Dependencies

```al
// BAD: Depends on previous test
[Test]
procedure Test2()
begin
    // Assumes Test1 created data
    Customer.Get('TEST');
end;

// GOOD: Self-contained
[Test]
procedure Test2()
begin
    Initialize();
    CreateCustomer(Customer);
    // ...
end;
```

## Migration Tips

When adding ATDD to existing tests:

1. Add `// [Scenario]` first - describes the test purpose
2. Add `// [Given]` - identify setup code
3. Add `// [When]` - identify the action (should be ONE)
4. Add `// [Then]` - identify assertions
5. Refactor if multiple actions in one test
