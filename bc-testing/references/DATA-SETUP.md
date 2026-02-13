# Test Data Setup

Proper test data setup ensures tests are repeatable, isolated, and meaningful. Business Central provides library codeunits for creating test data.

## Library Codeunits

The BCApps Test Framework includes library codeunits for common data scenarios:

| Codeunit                      | Purpose                              |
| ----------------------------- | ------------------------------------ |
| `Library - Sales`             | Create customers, sales documents    |
| `Library - Purchase`          | Create vendors, purchase documents   |
| `Library - Inventory`         | Create items, inventory transactions |
| `Library - Warehouse`         | Warehouse locations, bins            |
| `Library - Manufacturing`     | Production orders, BOMs              |
| `Library - Job`               | Projects and job entries             |
| `Library - Service`           | Service documents and items          |
| `Library - Fixed Asset`       | Fixed assets                         |
| `Library - HR`                | Employees                            |
| `Library - Random`            | Generate random values               |
| `Library - Utility`           | Utility functions                    |
| `Library - Variable Storage`  | Pass data to handlers                |
| `Library - Lower Permissions` | Test with reduced permissions        |
| `Library - Assert`            | Assertion methods                    |
| `Any`                         | Pseudo-random value generation       |

## Using Library - Sales

```al
var
    LibrarySales: Codeunit "Library - Sales";
    Customer: Record Customer;
    SalesHeader: Record "Sales Header";
    SalesLine: Record "Sales Line";
begin
    // Create a customer
    LibrarySales.CreateCustomer(Customer);

    // Create a sales header
    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Order, Customer."No.");

    // Create a sales line
    LibrarySales.CreateSalesLine(SalesLine, SalesHeader, SalesLine.Type::Item, '', 1);

    // Or create complete order with line
    LibrarySales.CreateSalesOrder(SalesHeader);

    // Post sales document
    LibrarySales.PostSalesDocument(SalesHeader, true, true); // Ship and Invoice
end;
```

## Using Library - Purchase

```al
var
    LibraryPurchase: Codeunit "Library - Purchase";
    Vendor: Record Vendor;
    PurchaseHeader: Record "Purchase Header";
    PurchaseLine: Record "Purchase Line";
begin
    // Create a vendor
    LibraryPurchase.CreateVendor(Vendor);

    // Create purchase order
    LibraryPurchase.CreatePurchHeader(PurchaseHeader, PurchaseHeader."Document Type"::Order, Vendor."No.");
    LibraryPurchase.CreatePurchaseLine(PurchaseLine, PurchaseHeader, PurchaseLine.Type::Item, '', 1);

    // Post purchase document
    LibraryPurchase.PostPurchaseDocument(PurchaseHeader, true, true); // Receive and Invoice
end;
```

## Using Library - Inventory

```al
var
    LibraryInventory: Codeunit "Library - Inventory";
    Item: Record Item;
    Location: Record Location;
    ItemJournalLine: Record "Item Journal Line";
begin
    // Create an item
    LibraryInventory.CreateItem(Item);

    // Create item with specific unit cost
    LibraryInventory.CreateItemWithUnitPriceAndUnitCost(Item, 100, 50);

    // Create location
    LibraryInventory.CreateLocation(Location);

    // Post inventory adjustment
    LibraryInventory.CreateItemJournalLineInItemTemplate(ItemJournalLine, Item."No.", '', '', 100);
    LibraryInventory.PostItemJournalLine(ItemJournalLine."Journal Template Name", ItemJournalLine."Journal Batch Name");
end;
```

## Random Data Generation

### Library - Random

Use random values when the specific value doesn't matter:

```al
var
    LibraryRandom: Codeunit "Library - Random";
    RandomInt: Integer;
    RandomDec: Decimal;
    RandomCode: Code[10];
begin
    // Random integer
    RandomInt := LibraryRandom.RandInt(100); // 1 to 100

    // Random integer in range
    RandomInt := LibraryRandom.RandIntInRange(50, 100); // 50 to 100

    // Random decimal
    RandomDec := LibraryRandom.RandDec(1000, 2); // Up to 1000, 2 decimal places

    // Random decimal in range
    RandomDec := LibraryRandom.RandDecInRange(100, 500, 2);

    // Non-zero random
    RandomDec := LibraryRandom.RandDecInDecimalRange(0.01, 100, 2);

    // Random code
    RandomCode := LibraryRandom.RandText(10);
end;
```

### Any Library (BCApps)

The `Any` library provides pseudo-random generation with reproducible seeds:

```al
var
    Any: Codeunit Any;
begin
    // Set seed for reproducibility
    Any.SetSeed(1);

    // Generate values
    IntValue := Any.IntegerInRange(1, 1000);
    DecValue := Any.DecimalInRange(1, 100, 2);
    TextValue := Any.AlphanumericText(20);
    DateValue := Any.DateInRange(Today, Today + 365);
    GuidValue := Any.GuidValue();
end;
```

## Seeding for Reproducibility

Set a seed to ensure the same random values each test run:

```al
local procedure Initialize()
begin
    if IsInitialized then
        exit;

    LibraryRandom.SetSeed(1); // Always generates same sequence

    IsInitialized := true;
end;
```

This ensures:

- Debugging failed tests uses same data
- CI/CD runs are reproducible
- Random values are consistent across runs

## Initialize Pattern

Use an Initialize procedure for all test setup:

```al
codeunit 50100 "Sales Tests"
{
    Subtype = Test;

    var
        LibrarySales: Codeunit "Library - Sales";
        LibraryRandom: Codeunit "Library - Random";
        Assert: Codeunit Assert;
        IsInitialized: Boolean;

    local procedure Initialize()
    var
        SalesSetup: Record "Sales & Receivables Setup";
    begin
        // Always clean up between tests
        CleanupTestData();

        if IsInitialized then
            exit;

        // One-time setup
        LibraryRandom.SetSeed(1);

        // Configure required settings
        SalesSetup.Get();
        SalesSetup."Stockout Warning" := false;
        SalesSetup.Modify();

        IsInitialized := true;
        Commit(); // Commit so test transactions can roll back cleanly
    end;

    local procedure CleanupTestData()
    var
        SalesHeader: Record "Sales Header";
    begin
        SalesHeader.SetRange("Document Type", SalesHeader."Document Type"::Order);
        SalesHeader.DeleteAll(true);
    end;

    [Test]
    procedure TestExample()
    begin
        Initialize();
        // Test code...
    end;
}
```

## Library - Variable Storage

Pass data to handler methods:

```al
var
    LibraryVariableStorage: Codeunit "Library - Variable Storage";

[Test]
[HandlerFunctions('ConfirmHandler')]
procedure TestWithDynamicConfirm()
begin
    Initialize();

    // Queue data for handler
    LibraryVariableStorage.Enqueue(true); // Boolean
    LibraryVariableStorage.Enqueue('Expected Text'); // Text
    LibraryVariableStorage.Enqueue(100); // Integer

    // Code that triggers confirm
    RunProcess();

    // Clear at end (or in Initialize)
    LibraryVariableStorage.AssertEmpty();
end;

[ConfirmHandler]
procedure ConfirmHandler(Question: Text[1024]; var Reply: Boolean)
begin
    Reply := LibraryVariableStorage.DequeueBoolean();
end;
```

### Variable Storage Methods

| Method             | Description                     |
| ------------------ | ------------------------------- |
| `Enqueue(Value)`   | Add value to queue              |
| `DequeueBoolean()` | Get next boolean                |
| `DequeueText()`    | Get next text                   |
| `DequeueInteger()` | Get next integer                |
| `DequeueDecimal()` | Get next decimal                |
| `DequeueDate()`    | Get next date                   |
| `Clear()`          | Clear all queued values         |
| `AssertEmpty()`    | Verify all values were consumed |

## Permission Testing

Test that functionality works without SUPER permissions:

```al
var
    LibraryLowerPermissions: Codeunit "Library - Lower Permissions";

[Test]
procedure TestWithRestrictedPermissions()
var
    Customer: Record Customer;
begin
    Initialize();

    // Set restricted permissions
    LibraryLowerPermissions.SetO365BusFull();

    // Verify functionality works
    LibrarySales.CreateCustomer(Customer);
    Assert.AreNotEqual('', Customer."No.", 'Should create customer');
end;
```

### Common Permission Sets

```al
// Office 365 permissions
LibraryLowerPermissions.SetO365BusFull();
LibraryLowerPermissions.SetO365Basic();

// Functional permissions
LibraryLowerPermissions.SetSalesDocsCreate();
LibraryLowerPermissions.SetPurchDocsCreate();

// Restore
LibraryLowerPermissions.SetOutsideO365Scope();
```

## Creating Test Data Patterns

### Simple Record Creation

```al
local procedure CreateCustomerWithBalance(var Customer: Record Customer; Balance: Decimal)
begin
    LibrarySales.CreateCustomer(Customer);
    // Post invoices to create balance
    CreateAndPostSalesInvoice(Customer."No.", Balance);
end;
```

### Complete Document Creation

```al
local procedure CreateSalesOrderWithLines(var SalesHeader: Record "Sales Header"; LineCount: Integer)
var
    SalesLine: Record "Sales Line";
    Item: Record Item;
    i: Integer;
begin
    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Order, '');

    for i := 1 to LineCount do begin
        LibraryInventory.CreateItem(Item);
        LibrarySales.CreateSalesLine(
            SalesLine,
            SalesHeader,
            SalesLine.Type::Item,
            Item."No.",
            LibraryRandom.RandIntInRange(1, 10)
        );
        SalesLine.Validate("Unit Price", LibraryRandom.RandDecInRange(10, 100, 2));
        SalesLine.Modify(true);
    end;
end;
```

### Setup with Known Values

```al
local procedure CreateKnownSalesScenario(var SalesHeader: Record "Sales Header"; var ExpectedAmount: Decimal)
var
    SalesLine: Record "Sales Line";
    Item: Record Item;
begin
    // Known values for predictable assertions
    LibraryInventory.CreateItemWithUnitPriceAndUnitCost(Item, 25.00, 10.00);

    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Invoice, '');
    LibrarySales.CreateSalesLine(SalesLine, SalesHeader, SalesLine.Type::Item, Item."No.", 4);

    ExpectedAmount := 100.00; // 4 x 25.00
end;
```

## When to Use Random vs. Fixed Data

| Use Random Data When                | Use Fixed Data When           |
| ----------------------------------- | ----------------------------- |
| Value doesn't affect test outcome   | Testing specific calculations |
| Need variety across tests           | Testing boundary conditions   |
| Testing typical scenarios           | Testing error conditions      |
| Performance doesn't depend on value | Value appears in assertions   |

```al
// Random: quantity doesn't matter
SalesLine.Validate(Quantity, LibraryRandom.RandIntInRange(1, 100));

// Fixed: testing specific discount calculation
SalesLine.Validate(Quantity, 10);
SalesLine.Validate("Unit Price", 100.00);
SalesLine.Validate("Line Discount %", 25);
Assert.AreEqual(750.00, SalesLine.Amount, 'Discount calculation wrong');
```
