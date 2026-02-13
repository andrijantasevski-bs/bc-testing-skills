# UI Handler Methods

Handler methods intercept UI dialogs during test execution, enabling fully automated tests without user interaction.

## Handler Types Overview

| Handler                   | Intercepts                | Attribute                     |
| ------------------------- | ------------------------- | ----------------------------- |
| MessageHandler            | `Message()` calls         | `[MessageHandler]`            |
| ConfirmHandler            | `Confirm()` calls         | `[ConfirmHandler]`            |
| StrMenuHandler            | `StrMenu()` calls         | `[StrMenuHandler]`            |
| PageHandler               | Non-modal pages           | `[PageHandler]`               |
| ModalPageHandler          | Modal pages               | `[ModalPageHandler]`          |
| ReportHandler             | Report execution          | `[ReportHandler]`             |
| RequestPageHandler        | Report request pages      | `[RequestPageHandler]`        |
| HyperlinkHandler          | `Hyperlink()` calls       | `[HyperlinkHandler]`          |
| SendNotificationHandler   | `Notification.Send()`     | `[SendNotificationHandler]`   |
| RecallNotificationHandler | `Notification.Recall()`   | `[RecallNotificationHandler]` |
| SessionSettingsHandler    | `RequestSessionUpdate()`  | `[SessionSettingsHandler]`    |
| FilterPageHandler         | FilterPageBuilder dialogs | `[FilterPageHandler]`         |

## MessageHandler

Handles `Message()` dialog calls:

```al
[Test]
[HandlerFunctions('MessageHandler')]
procedure TestShowsSuccessMessage()
var
    SalesOrder: Record "Sales Header";
begin
    Initialize();

    // Action that triggers Message('Order posted successfully')
    PostSalesOrder(SalesOrder);
end;

[MessageHandler]
procedure MessageHandler(Message: Text[1024])
begin
    // Verify the message content
    Assert.IsTrue(
        StrPos(Message, 'successfully') > 0,
        'Expected success message, got: ' + Message
    );
end;
```

## ConfirmHandler

Handles `Confirm()` dialog calls and provides the response:

```al
[Test]
[HandlerFunctions('ConfirmHandlerYes')]
procedure TestConfirmDeletion()
var
    Customer: Record Customer;
begin
    Initialize();
    CreateCustomer(Customer);

    // Action triggers: Confirm('Delete customer?')
    Customer.Delete(true);

    Assert.IsFalse(Customer.Get(Customer."No."), 'Customer should be deleted');
end;

[ConfirmHandler]
procedure ConfirmHandlerYes(Question: Text[1024]; var Reply: Boolean)
begin
    // Verify question and set reply
    Assert.IsTrue(
        StrPos(Question, 'Delete') > 0,
        'Unexpected confirm: ' + Question
    );
    Reply := true; // Click Yes
end;

[ConfirmHandler]
procedure ConfirmHandlerNo(Question: Text[1024]; var Reply: Boolean)
begin
    Reply := false; // Click No
end;
```

### Dynamic Confirm Handling

Use library variable storage for flexible responses:

```al
var
    LibraryVariableStorage: Codeunit "Library - Variable Storage";

[Test]
[HandlerFunctions('DynamicConfirmHandler')]
procedure TestMultipleConfirms()
begin
    Initialize();

    // Queue expected replies
    LibraryVariableStorage.Enqueue(true);  // First confirm: Yes
    LibraryVariableStorage.Enqueue(false); // Second confirm: No

    // Action that triggers multiple confirms
    ProcessWithConfirmations();
end;

[ConfirmHandler]
procedure DynamicConfirmHandler(Question: Text[1024]; var Reply: Boolean)
begin
    Reply := LibraryVariableStorage.DequeueBoolean();
end;
```

## StrMenuHandler

Handles `StrMenu()` option dialogs:

```al
[Test]
[HandlerFunctions('StrMenuHandler')]
procedure TestOptionSelection()
begin
    Initialize();

    // Action triggers: StrMenu('Option A,Option B,Option C', 1, 'Choose:')
    SelectOption();
end;

[StrMenuHandler]
procedure StrMenuHandler(Options: Text[1024]; var Choice: Integer; Instruction: Text[1024])
begin
    // Verify options
    Assert.AreEqual('Option A,Option B,Option C', Options, 'Wrong options');

    // Select option 2 (Option B)
    Choice := 2;
end;
```

## PageHandler

Handles non-modal pages (pages opened with `Page.Run`):

```al
[Test]
[HandlerFunctions('CustomerListPageHandler')]
procedure TestOpenCustomerList()
begin
    Initialize();

    // Action that opens Customer List non-modally
    OpenCustomerList();
end;

[PageHandler]
procedure CustomerListPageHandler(var CustomerList: TestPage "Customer List")
begin
    // Verify page opened and data is correct
    Assert.IsTrue(CustomerList.First(), 'Expected customers in list');
    CustomerList.Close();
end;
```

## ModalPageHandler

Handles modal pages (pages opened with `Page.RunModal`):

```al
[Test]
[HandlerFunctions('ItemLookupModalHandler')]
procedure TestItemLookup()
var
    SalesLine: Record "Sales Line";
begin
    Initialize();

    // Opening item lookup triggers modal page
    SalesLine.Validate("No.", ''); // Triggers lookup
end;

[ModalPageHandler]
procedure ItemLookupModalHandler(var ItemList: TestPage "Item List")
begin
    // Select an item
    ItemList.First();
    ItemList.OK().Invoke();
end;
```

### Passing Data to Modal Handlers

```al
var
    LibraryVariableStorage: Codeunit "Library - Variable Storage";

[Test]
[HandlerFunctions('SelectCustomerModalHandler')]
procedure TestSelectSpecificCustomer()
var
    Customer: Record Customer;
begin
    Initialize();
    LibrarySales.CreateCustomer(Customer);

    // Pass customer no to handler
    LibraryVariableStorage.Enqueue(Customer."No.");

    SelectCustomerFromLookup();
end;

[ModalPageHandler]
procedure SelectCustomerModalHandler(var CustomerList: TestPage "Customer List")
var
    CustomerNo: Code[20];
begin
    CustomerNo := LibraryVariableStorage.DequeueText();
    CustomerList.GoToKey(CustomerNo);
    CustomerList.OK().Invoke();
end;
```

## ReportHandler

Handles report execution (replaces entire report including request page):

```al
[Test]
[HandlerFunctions('SalesInvoiceReportHandler')]
procedure TestPrintSalesInvoice()
var
    SalesInvoiceHeader: Record "Sales Invoice Header";
begin
    Initialize();
    CreatePostedSalesInvoice(SalesInvoiceHeader);

    // Print invoice
    Report.Run(Report::"Sales - Invoice", true, false, SalesInvoiceHeader);
end;

[ReportHandler]
procedure SalesInvoiceReportHandler(var SalesInvoice: Report "Sales - Invoice")
begin
    // Report runs with default settings
    // Can access report data here
end;
```

> **Note**: When using ReportHandler, the RequestPageHandler is NOT called. Use ReportHandler when you want to bypass the request page entirely.

## RequestPageHandler

Handles report request pages (use when you need to set report options):

```al
[Test]
[HandlerFunctions('CustomerListReportRequestPageHandler')]
procedure TestCustomerReport()
begin
    Initialize();

    Report.Run(Report::"Customer - List");
end;

[RequestPageHandler]
procedure CustomerListReportRequestPageHandler(var CustomerList: TestRequestPage "Customer - List")
begin
    // Set report options
    CustomerList."Customer".SetFilter("No.", '10000..20000');
    CustomerList.OK().Invoke();
end;
```

## HyperlinkHandler

Handles `Hyperlink()` calls:

```al
[Test]
[HandlerFunctions('HyperlinkHandler')]
procedure TestOpenWebsite()
begin
    Initialize();

    // Action that calls Hyperlink('https://example.com')
    OpenCompanyWebsite();
end;

[HyperlinkHandler]
procedure HyperlinkHandler(Hyperlink: Text[1024])
begin
    Assert.AreEqual('https://example.com', Hyperlink, 'Wrong URL');
end;
```

## SendNotificationHandler

Handles notification display:

```al
[Test]
[HandlerFunctions('NotificationHandler')]
procedure TestShowsNotification()
var
    MyNotification: Notification;
begin
    Initialize();

    MyNotification.Message('Item is out of stock');
    MyNotification.Send();
end;

[SendNotificationHandler]
procedure NotificationHandler(var TheNotification: Notification): Boolean
begin
    Assert.AreEqual('Item is out of stock', TheNotification.Message, 'Wrong notification');
    exit(true); // Return true to indicate handled
end;
```

## Multiple Handlers

Declare multiple handlers separated by commas:

```al
[Test]
[HandlerFunctions('MessageHandler,ConfirmHandler,ModalPageHandler')]
procedure TestComplexWorkflow()
begin
    Initialize();

    // Workflow that triggers all three UI elements
    RunComplexProcess();
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

[ModalPageHandler]
procedure ModalPageHandler(var ThePage: TestPage "Some Page")
begin
    ThePage.OK().Invoke();
end;
```

## Handler Rules

1. **Every handler must be called** - If you declare a handler in `[HandlerFunctions]`, the test must trigger it
2. **Handlers are consumed in order** - If same dialog appears twice, handler runs twice
3. **Unhandled UI = test failure** - Any UI without a handler fails the test
4. **Handlers are method-specific** - Each test method declares its own handlers

## Common Patterns

### Verifying Error Messages

```al
[Test]
[HandlerFunctions('ErrorMessageHandler')]
procedure TestShowsValidationError()
begin
    Initialize();

    asserterror InvalidOperation(); // Expected to fail

    // Error is handled by ErrorMessageHandler
end;
```

### Handling Multiple Pages in Sequence

```al
[Test]
[HandlerFunctions('FirstPageHandler,SecondPageHandler')]
procedure TestWizardFlow()
begin
    RunWizard(); // Opens FirstPage, then SecondPage
end;
```

### Closing Without Action

```al
[ModalPageHandler]
procedure CancelHandler(var ThePage: TestPage "Some Page")
begin
    ThePage.Cancel().Invoke(); // Close without saving
end;
```
