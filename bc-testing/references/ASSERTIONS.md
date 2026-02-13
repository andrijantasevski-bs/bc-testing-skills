# Assertions

Assertions validate expected outcomes in tests. Business Central provides the `Assert` codeunit and `ASSERTERROR` keyword for verification.

## Assert Codeunit (Codeunit 130000)

The `Assert` codeunit from the Test Framework provides assertion methods.

### Setup

```al
codeunit 50100 "My Tests"
{
    Subtype = Test;

    var
        Assert: Codeunit Assert; // From Library Assert
}
```

## Equality Assertions

### Assert.AreEqual

Verifies two values are equal:

```al
procedure TestValuesAreEqual()
var
    Expected: Integer;
    Actual: Integer;
begin
    Expected := 100;
    Actual := CalculateValue();

    Assert.AreEqual(Expected, Actual, 'Values should be equal');
end;
```

Supports all primitive types:

```al
// Integer
Assert.AreEqual(5, Customer.Count, 'Wrong customer count');

// Decimal
Assert.AreEqual(100.50, SalesLine.Amount, 'Wrong amount');

// Text
Assert.AreEqual('John', Customer.Name, 'Wrong name');

// Date
Assert.AreEqual(Today, SalesHeader."Order Date", 'Wrong order date');

// Boolean
Assert.AreEqual(true, Item.Blocked, 'Item should be blocked');

// Code
Assert.AreEqual('ITEM001', SalesLine."No.", 'Wrong item no.');

// Enum
Assert.AreEqual(SalesHeader."Document Type"::Order, ActualType, 'Wrong document type');
```

### Assert.AreNotEqual

Verifies two values are different:

```al
procedure TestValuesAreDifferent()
begin
    Assert.AreNotEqual('', Customer."No.", 'Customer No. should be assigned');
    Assert.AreNotEqual(0, SalesLine.Amount, 'Amount should not be zero');
end;
```

### Assert.AreNearlyEqual

Verifies decimal values within a tolerance:

```al
procedure TestDecimalPrecision()
var
    Expected: Decimal;
    Actual: Decimal;
    Tolerance: Decimal;
begin
    Expected := 100.00;
    Actual := 100.004;
    Tolerance := 0.01;

    Assert.AreNearlyEqual(Expected, Actual, Tolerance, 'Values should be nearly equal');
end;
```

## Boolean Assertions

### Assert.IsTrue / Assert.IsFalse

```al
procedure TestBooleanConditions()
begin
    // IsTrue - condition must be true
    Assert.IsTrue(Customer.Get('10000'), 'Customer should exist');
    Assert.IsTrue(SalesHeader.Find('='), 'Record should be found');
    Assert.IsTrue(StrLen(Name) > 0, 'Name should not be empty');

    // IsFalse - condition must be false
    Assert.IsFalse(Item.Blocked, 'Item should not be blocked');
    Assert.IsFalse(Customer.IsEmpty(), 'Customers should exist');
end;
```

## Record Assertions

### Assert.RecordIsEmpty

Verifies a record has no rows:

```al
procedure TestRecordDeleted()
var
    TempItem: Record Item temporary;
begin
    // Delete all items
    TempItem.DeleteAll();

    Assert.RecordIsEmpty(TempItem);
end;
```

### Assert.RecordIsNotEmpty

Verifies a record has rows:

```al
procedure TestRecordsExist()
var
    Customer: Record Customer;
begin
    Customer.SetFilter("Balance (LCY)", '>0');

    Assert.RecordIsNotEmpty(Customer);
end;
```

### Assert.RecordCount

Verifies exact record count:

```al
procedure TestRecordCount()
var
    SalesLine: Record "Sales Line";
begin
    SalesLine.SetRange("Document No.", OrderNo);

    Assert.RecordCount(SalesLine, 3); // Exactly 3 lines expected
end;
```

### Assert.TableIsEmpty / Assert.TableIsNotEmpty

Verifies entire table state:

```al
procedure TestTableState()
var
    MyTable: Record "My Table";
begin
    Assert.TableIsEmpty(Database::"My Table"); // No records in table

    // Or
    Assert.TableIsNotEmpty(Database::Customer); // Customers exist
end;
```

## Error Assertions

### Assert.ExpectedError

Use when error text is known:

```al
procedure TestErrorMessage()
begin
    asserterror InvalidOperation();

    Assert.ExpectedError('Quantity cannot be negative');
end;
```

### Assert.ExpectedErrorCode

Use when error code is known:

```al
procedure TestErrorCode()
begin
    asserterror RecordNotFoundOperation();

    Assert.ExpectedErrorCode('Dialog');
end;
```

## ASSERTERROR Keyword

`ASSERTERROR` expects the following statement to raise an error. If no error occurs, the test fails.

### Basic Usage

```al
procedure TestNegativeQuantityNotAllowed()
var
    SalesLine: Record "Sales Line";
begin
    // [Given] A sales line
    CreateSalesLine(SalesLine);

    // [When] Set negative quantity
    // [Then] Error is raised
    asserterror SalesLine.Validate(Quantity, -1);
end;
```

### Verifying Error Text

Use `GETLASTERRORTEXT` to verify the exact error message:

```al
procedure TestSpecificError()
var
    Customer: Record Customer;
begin
    asserterror Customer.Get('NONEXISTENT');

    Assert.AreEqual(
        'The Customer does not exist.',
        GetLastErrorText(),
        'Wrong error message'
    );
end;
```

### Partial Error Text Match

```al
procedure TestErrorContains()
var
    Item: Record Item;
begin
    asserterror Item.Validate("Base Unit of Measure", '');

    Assert.IsTrue(
        StrPos(GetLastErrorText(), 'Base Unit of Measure') > 0,
        'Error should mention field name'
    );
end;
```

### ASSERTERROR Rules

1. Must be followed by a statement that raises an error
2. If no error is raised, `ASSERTERROR` itself fails
3. Only use in test code, never in production code
4. Error is suppressed but captured for verification

```al
// Statement raises error - test passes
asserterror Error('Expected error');

// Statement doesn't raise error - ASSERTERROR fails
asserterror Customer.Insert(); // Fails if insert succeeds
```

## Failure Assertions

### Assert.Fail

Unconditionally fails the test:

```al
procedure TestNotImplemented()
begin
    Assert.Fail('Test not yet implemented');
end;

procedure TestUnexpectedPath()
begin
    case DocumentType of
        DocumentType::Order:
            ProcessOrder();
        DocumentType::Invoice:
            ProcessInvoice();
        else
            Assert.Fail('Unexpected document type: ' + Format(DocumentType));
    end;
end;
```

## Descriptive Error Messages

Always include meaningful error messages:

```al
// BAD: No context
Assert.AreEqual(5, Count, '');

// BAD: Generic message
Assert.AreEqual(5, Count, 'Values not equal');

// GOOD: Describes what was expected
Assert.AreEqual(5, Count, 'Expected 5 sales lines after posting');

// GOOD: Includes actual value context
Assert.AreEqual(
    ExpectedAmount,
    ActualAmount,
    StrSubstNo('Invoice %1 amount incorrect', InvoiceNo)
);
```

## Complete Verification Example

```al
[Test]
procedure TestSalesOrderPosting()
var
    SalesHeader: Record "Sales Header";
    SalesLine: Record "Sales Line";
    SalesInvoiceHeader: Record "Sales Invoice Header";
    PostedAmount: Decimal;
begin
    // [Given] A sales order with one line
    Initialize();
    CreateSalesOrderWithLine(SalesHeader, SalesLine);

    // [When] Post the order
    LibrarySales.PostSalesDocument(SalesHeader, true, true);

    // [Then] Invoice is created
    SalesInvoiceHeader.SetRange("Sell-to Customer No.", SalesHeader."Sell-to Customer No.");
    Assert.RecordIsNotEmpty(SalesInvoiceHeader);

    // [Then] Correct amount is posted
    SalesInvoiceHeader.FindFirst();
    SalesInvoiceHeader.CalcFields(Amount);
    Assert.AreEqual(
        SalesLine.Amount,
        SalesInvoiceHeader.Amount,
        'Posted amount should match order amount'
    );

    // [Then] Order is deleted
    Assert.IsFalse(
        SalesHeader.Get(SalesHeader."Document Type", SalesHeader."No."),
        'Order should be deleted after posting'
    );
end;
```

## Negative Test Example

```al
[Test]
procedure TestCannotPostWithoutLines()
var
    SalesHeader: Record "Sales Header";
begin
    // [Given] A sales order without lines
    Initialize();
    LibrarySales.CreateSalesHeader(SalesHeader, SalesHeader."Document Type"::Order, '');

    // [When] Attempt to post
    asserterror LibrarySales.PostSalesDocument(SalesHeader, true, true);

    // [Then] Error is raised
    Assert.IsTrue(
        StrPos(GetLastErrorText(), 'nothing to post') > 0,
        'Should error about no lines to post'
    );
end;
```

## Custom Assert Procedures

Create reusable assertions for common scenarios:

```al
local procedure AssertCustomerBalance(CustomerNo: Code[20]; ExpectedBalance: Decimal)
var
    Customer: Record Customer;
begin
    Customer.Get(CustomerNo);
    Customer.CalcFields("Balance (LCY)");
    Assert.AreEqual(
        ExpectedBalance,
        Customer."Balance (LCY)",
        StrSubstNo('Customer %1 balance incorrect', CustomerNo)
    );
end;

local procedure AssertRecordCount(var RecRef: RecordRef; ExpectedCount: Integer; Context: Text)
begin
    Assert.AreEqual(
        ExpectedCount,
        RecRef.Count(),
        StrSubstNo('Wrong record count for %1', Context)
    );
end;
```
