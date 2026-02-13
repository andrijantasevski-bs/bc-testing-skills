# Test Pages

TestPage objects simulate user interaction with pages without displaying UI. They enable testing page behavior, field validation, and actions programmatically.

## TestPage Types

| Type              | Purpose                                    |
| ----------------- | ------------------------------------------ |
| `TestPage`        | Regular pages (card, list, document, etc.) |
| `TestRequestPage` | Report request pages                       |

## Opening Test Pages

### OpenView

Opens page in view mode (read-only):

```al
var
    CustomerList: TestPage "Customer List";
begin
    CustomerList.OpenView();
    // Page is open for viewing
    CustomerList.Close();
end;
```

### OpenEdit

Opens page in edit mode:

```al
var
    CustomerCard: TestPage "Customer Card";
begin
    CustomerCard.OpenEdit();
    CustomerCard."No.".SetValue('10000');
    CustomerCard.Name.SetValue('New Customer');
    CustomerCard.Close();
end;
```

### OpenNew

Opens page to create a new record:

```al
var
    CustomerCard: TestPage "Customer Card";
begin
    CustomerCard.OpenNew();
    CustomerCard.Name.SetValue('Brand New Customer');
    CustomerCard.OK().Invoke();
end;
```

## Accessing Fields

Use dot notation to access page fields:

```al
var
    CustomerCard: TestPage "Customer Card";
    CustomerNo: Code[20];
    CustomerName: Text;
begin
    CustomerCard.OpenView();
    CustomerCard.First();

    // Read field values
    CustomerNo := CustomerCard."No.".Value;
    CustomerName := CustomerCard.Name.Value;

    // Set field values
    CustomerCard.Name.SetValue('Updated Name');
    CustomerCard.Address.SetValue('123 Main St');

    // Get as specific type
    Assert.AreEqual(10000, CustomerCard."Credit Limit (LCY)".AsDecimal(), 'Wrong credit limit');
    Assert.IsTrue(CustomerCard.Blocked.AsBoolean(), 'Should be blocked');

    CustomerCard.Close();
end;
```

### Field Methods

| Method                | Description       |
| --------------------- | ----------------- |
| `.Value`              | Get/set as text   |
| `.AsInteger()`        | Get as integer    |
| `.AsDecimal()`        | Get as decimal    |
| `.AsBoolean()`        | Get as boolean    |
| `.AsDate()`           | Get as date       |
| `.AsTime()`           | Get as time       |
| `.AsDateTime()`       | Get as datetime   |
| `.SetValue(NewValue)` | Set field value   |
| `.Activate()`         | Focus on field    |
| `.Visible`            | Check if visible  |
| `.Editable`           | Check if editable |
| `.Enabled`            | Check if enabled  |

## Navigation Methods

### Basic Navigation

```al
var
    CustomerList: TestPage "Customer List";
begin
    CustomerList.OpenView();

    // Move to records
    CustomerList.First();     // Go to first record
    CustomerList.Next();      // Move to next record
    CustomerList.Previous();  // Move to previous record
    CustomerList.Last();      // Go to last record

    CustomerList.Close();
end;
```

### GoToRecord

Navigate to a specific record:

```al
var
    Customer: Record Customer;
    CustomerCard: TestPage "Customer Card";
begin
    Customer.Get('10000');

    CustomerCard.OpenView();
    CustomerCard.GoToRecord(Customer);

    Assert.AreEqual('10000', CustomerCard."No.".Value, 'Wrong customer');
    CustomerCard.Close();
end;
```

### GoToKey

Navigate using primary key values:

```al
var
    CustomerCard: TestPage "Customer Card";
begin
    CustomerCard.OpenView();
    CustomerCard.GoToKey('10000'); // Primary key value
    CustomerCard.Close();
end;
```

### FindFirstField / FindNextField

Search for records by field value:

```al
var
    CustomerList: TestPage "Customer List";
begin
    CustomerList.OpenView();

    // Find first customer with specific city
    if CustomerList.FindFirstField(City, 'Seattle') then begin
        // Found
        Assert.AreEqual('Seattle', CustomerList.City.Value, 'Wrong city');
    end;

    // Find next with same city
    if CustomerList.FindNextField(City, 'Seattle') then begin
        // Found another
    end;

    CustomerList.Close();
end;
```

## Filtering

Use the `.Filter` property to filter page data:

```al
var
    CustomerList: TestPage "Customer List";
begin
    CustomerList.OpenView();

    // Set filters
    CustomerList.Filter.SetFilter("No.", '10000..20000');
    CustomerList.Filter.SetFilter(City, 'Seattle');
    CustomerList.Filter.SetFilter("Balance (LCY)", '>1000');

    // Iterate filtered results
    if CustomerList.First() then
        repeat
            // Process filtered customers
        until not CustomerList.Next();

    CustomerList.Close();
end;
```

## Invoking Actions

```al
var
    CustomerCard: TestPage "Customer Card";
begin
    CustomerCard.OpenEdit();
    CustomerCard.GoToKey('10000');

    // Invoke page actions
    CustomerCard."Update Credit Limit".Invoke();

    // Built-in actions
    CustomerCard.OK().Invoke();      // Click OK
    CustomerCard.Cancel().Invoke();  // Click Cancel
    CustomerCard.Yes().Invoke();     // Click Yes
    CustomerCard.No().Invoke();      // Click No

    CustomerCard.Close();
end;
```

### Checking Action Visibility

```al
var
    CustomerCard: TestPage "Customer Card";
begin
    CustomerCard.OpenView();

    Assert.IsTrue(
        CustomerCard."Sales Orders".Visible,
        'Sales Orders action should be visible'
    );

    Assert.IsFalse(
        CustomerCard."Delete".Enabled,
        'Delete should be disabled in view mode'
    );

    CustomerCard.Close();
end;
```

## Page Parts and Subpages

Access subpages and FactBoxes using dot notation:

```al
var
    SalesOrder: TestPage "Sales Order";
begin
    SalesOrder.OpenEdit();
    SalesOrder.GoToKey(SalesHeader."Document Type"::Order, SalesHeader."No.");

    // Access subpage (Sales Lines)
    SalesOrder.SalesLines.First();
    SalesOrder.SalesLines.Quantity.SetValue(5);
    SalesOrder.SalesLines.Next();

    // Access FactBox
    Assert.AreEqual(
        CustomerNo,
        SalesOrder."Sell-to Customer Sales History"."No.".Value,
        'Wrong customer in FactBox'
    );

    SalesOrder.Close();
end;
```

### Adding Lines to Subpage

```al
var
    SalesOrder: TestPage "Sales Order";
begin
    SalesOrder.OpenNew();
    SalesOrder."Sell-to Customer No.".SetValue('10000');

    // Add first line
    SalesOrder.SalesLines.New();
    SalesOrder.SalesLines.Type.SetValue('Item');
    SalesOrder.SalesLines."No.".SetValue('1000');
    SalesOrder.SalesLines.Quantity.SetValue(1);

    // Add second line
    SalesOrder.SalesLines.New();
    SalesOrder.SalesLines.Type.SetValue('Item');
    SalesOrder.SalesLines."No.".SetValue('1001');
    SalesOrder.SalesLines.Quantity.SetValue(2);

    SalesOrder.OK().Invoke();
end;
```

## Trap Method

Use `Trap` to intercept pages opened by code:

```al
var
    CustomerCard: TestPage "Customer Card";
    SalesOrderList: TestPage "Sales Order List";
begin
    CustomerCard.OpenEdit();
    CustomerCard.GoToKey('10000');

    // Trap the next page that opens
    SalesOrderList.Trap();

    // This action opens Sales Order List page
    CustomerCard."Sales Orders".Invoke();

    // SalesOrderList now references the opened page
    Assert.IsTrue(SalesOrderList.First(), 'Expected sales orders');
    SalesOrderList.Close();

    CustomerCard.Close();
end;
```

## TestRequestPage

For testing report request pages:

```al
[Test]
[HandlerFunctions('CustomerListRequestPageHandler')]
procedure TestCustomerReport()
begin
    Report.Run(Report::"Customer - List");
end;

[RequestPageHandler]
procedure CustomerListRequestPageHandler(var CustomerList: TestRequestPage "Customer - List")
begin
    // Access request page fields
    CustomerList."Customer".SetFilter("No.", '10000..20000');

    // Set options if available
    // CustomerList.IncludeInactive.SetValue(true);

    // Run the report
    CustomerList.OK().Invoke();

    // Or cancel
    // CustomerList.Cancel().Invoke();
end;
```

## Validation Events

Setting values triggers validation:

```al
var
    SalesOrder: TestPage "Sales Order";
begin
    SalesOrder.OpenNew();

    // This triggers OnValidate for "Sell-to Customer No."
    SalesOrder."Sell-to Customer No.".SetValue('10000');

    // Customer name should be populated by validation
    Assert.AreNotEqual('', SalesOrder."Sell-to Customer Name".Value, 'Name should be set');

    SalesOrder.Close();
end;
```

## Complete Example

```al
[Test]
[HandlerFunctions('ConfirmShipmentHandler')]
procedure TestSalesOrderShipment()
var
    Customer: Record Customer;
    Item: Record Item;
    SalesHeader: Record "Sales Header";
    SalesOrder: TestPage "Sales Order";
begin
    // [Given] A customer and item exist
    Initialize();
    LibrarySales.CreateCustomer(Customer);
    LibraryInventory.CreateItem(Item);

    // [When] Create and ship a sales order
    SalesOrder.OpenNew();
    SalesOrder."Sell-to Customer No.".SetValue(Customer."No.");

    SalesOrder.SalesLines.New();
    SalesOrder.SalesLines.Type.SetValue('Item');
    SalesOrder.SalesLines."No.".SetValue(Item."No.");
    SalesOrder.SalesLines.Quantity.SetValue(10);

    SalesOrder.Post.Invoke(); // Triggers ConfirmHandler

    // [Then] Shipment was created
    SalesHeader.SetRange("Document Type", SalesHeader."Document Type"::Order);
    SalesHeader.SetRange("Sell-to Customer No.", Customer."No.");
    Assert.IsTrue(SalesHeader.FindFirst(), 'Order should exist');
    Assert.IsTrue(SalesHeader."Shipping No." <> '', 'Should have shipment');
end;

[ConfirmHandler]
procedure ConfirmShipmentHandler(Question: Text[1024]; var Reply: Boolean)
begin
    Reply := true; // Confirm shipment
end;
```

## Common Mistakes

### Opening page multiple times

```al
// BAD: Page already open
CustomerCard.OpenView();
CustomerCard.OpenView(); // Error!

// GOOD: Close first or reuse
CustomerCard.OpenView();
// ...
CustomerCard.Close();
CustomerCard.OpenView();
```

### Not handling lookups

```al
// BAD: Validation opens lookup with no handler
[Test]
procedure TestWithLookup()
begin
    SalesLine."No.".SetValue(''); // Opens item lookup - fails without handler!
end;

// GOOD: Add handler
[Test]
[HandlerFunctions('ItemLookupHandler')]
procedure TestWithLookup()
begin
    SalesLine."No.".SetValue('');
end;
```
