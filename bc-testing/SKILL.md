---
name: bc-testing
description: Guides writing automated tests for Microsoft Dynamics 365 Business Central AL code. Covers unit tests vs integration tests, test doubles (dummies, stubs, spies, mocks), ATDD patterns (Given-When-Then), UI handlers, TestPages, assertions, data isolation, HTTP mocking with HttpClientHandler, dependency injection, interfaces for testability, and Microsoft test libraries (Library Assert, Library Sales, Any). Use when creating test codeunits, test methods, or designing testable AL code.
metadata:
  author: bc-skills
  version: "2.0"
---

# Business Central AL Testing Skill

Comprehensive guide for writing automated tests in AL for Microsoft Dynamics 365 Business Central. This skill helps you create well-structured, maintainable, and isolated tests following Microsoft's official patterns and ATDD methodology.

## When to Apply

Reference these guidelines when:

- Writing new test codeunits or test methods for AL code
- Creating UI handlers (MessageHandler, ConfirmHandler, PageHandler, etc.)
- Testing pages using TestPage objects
- Setting up test data with Library codeunits
- Implementing ATDD patterns with Given-When-Then structure
- Mocking external service calls in tests
- Configuring test isolation and transaction handling
- Reviewing or refactoring existing BC tests

## Rule Categories by Priority

| Priority | Category       | Impact   | Prefix       |
| -------- | -------------- | -------- | ------------ |
| 1        | Test Structure | CRITICAL | `test-`      |
| 2        | UI Handlers    | CRITICAL | `handler-`   |
| 3        | Test Types     | CRITICAL | `types-`     |
| 4        | Assertions     | HIGH     | `assert-`    |
| 5        | Test Pages     | HIGH     | `page-`      |
| 6        | Testability    | HIGH     | `design-`    |
| 7        | Data Setup     | MEDIUM   | `data-`      |
| 8        | Test Isolation | MEDIUM   | `isolation-` |
| 9        | ATDD Patterns  | MEDIUM   | `atdd-`      |
| 10       | HTTP Mocking   | MEDIUM   | `http-`      |
| 11       | Event Mocking  | LOW      | `mock-`      |

## Quick Reference

### 1. Test Structure (CRITICAL)

- `test-codeunit-subtype` - Set `Subtype = Test` on test codeunits
- `test-method-attribute` - Use `[Test]` attribute on test methods
- `test-handler-functions` - Declare all handlers in `[HandlerFunctions]`
- `test-initialize-pattern` - Always include Initialize procedure
- `test-naming` - Name tests as `Test<Scenario>` describing what is tested
- `test-separation` - Keep test code separate from production code

### 2. UI Handlers (CRITICAL)

- `handler-message` - Handle Message dialogs with MessageHandler
- `handler-confirm` - Handle Confirm dialogs with ConfirmHandler
- `handler-page` - Handle non-modal pages with PageHandler
- `handler-modal-page` - Handle modal pages with ModalPageHandler
- `handler-report` - Handle reports with ReportHandler
- `handler-request-page` - Handle report request pages with RequestPageHandler
- `handler-required` - Every handler in HandlerFunctions must be called

### 3. Test Types (CRITICAL)

- `types-unit-vs-integration` - Prefer unit tests (<100ms) over integration tests (500ms+)
- `types-isolation` - Unit tests: test YOUR code only, not Microsoft's
- `types-dependencies` - Identify and abstract dependencies for testability
- `types-doubles` - Use test doubles: dummies, stubs, spies, mocks
- `types-pyramid` - Target 70% unit, 20% integration, 10% E2E tests
- `types-speed` - Tests must be fast; slow tests become useless

### 4. Assertions (HIGH)

- `assert-areequal` - Use Assert.AreEqual for value comparisons
- `assert-istrue` - Use Assert.IsTrue/IsFalse for boolean checks
- `assert-error` - Use ASSERTERROR for negative tests
- `assert-lasterror` - Verify error text with GETLASTERRORTEXT
- `assert-messages` - Always include descriptive error messages

### 5. Test Pages (HIGH)

- `page-open-methods` - Use OpenView/OpenEdit/OpenNew appropriately
- `page-field-access` - Access fields via dot notation
- `page-trap` - Use Trap to intercept page opens
- `page-actions` - Invoke actions with .Invoke method
- `page-navigation` - Use GoToRecord/GoToKey for navigation

### 6. Testability (HIGH)

- `design-interfaces` - Use interfaces to abstract dependencies
- `design-injection` - Inject dependencies as parameters
- `design-separation` - Separate business logic from database access
- `design-single-responsibility` - Each component does ONE thing
- `design-explicit-deps` - Make dependencies explicit, not hidden
- `design-composition` - Prefer composition over inheritance

### 7. Data Setup (MEDIUM)

- `data-random` - Use Library - Random for non-specific values
- `data-seed` - Use SetSeed for reproducible random data
- `data-libraries` - Use Library - Sales, Library - Purchase, etc.
- `data-cleanup` - Clean up test data in Initialize procedure
- `data-permissions` - Test with Library - Lower Permissions

### 8. Test Isolation (MEDIUM)

- `isolation-property` - Set TestIsolation on test runner codeunits
- `isolation-transaction` - Choose TransactionModel carefully
- `isolation-rollback` - Prefer AutoRollback when no COMMIT needed
- `isolation-independent` - Design tests to be independent of each other

### 9. ATDD Patterns (MEDIUM)

- `atdd-scenario` - Document with `// [Scenario]` comments
- `atdd-given` - Document preconditions with `// [Given]`
- `atdd-when` - Document action with `// [When]` (only one per test)
- `atdd-then` - Document verification with `// [Then]`
- `atdd-feature` - Use `// [FEATURE]` tag at codeunit level

### 10. HTTP Mocking (MEDIUM)

- `http-handler` - Use `[HttpClientHandler]` attribute for HTTP tests
- `http-policy` - Set TestHttpRequestPolicy to control outbound calls
- `http-response` - Create TestHttpResponseMessage with proper content
- `http-status` - Set appropriate HTTP status codes in mock responses
- `http-block` - Use BlockOutboundRequests to prevent real API calls
- `http-onprem` - Note: HttpClientHandler only works on-premises (AL 15.0+)

### 11. Event Mocking (LOW)

- `mock-events` - Use EventSubscriberInstance = Manual for controlled mocking
- `mock-external` - Mock all external service calls
- `mock-binding` - Use BindSubscription/UnbindSubscription in tests

## Core Test Method Pattern

Every test should follow this structure:

```al
[Test]
[HandlerFunctions('MessageHandler,ConfirmHandler')]
procedure TestCustomerCreationUpdatesRewardPoints()
var
    Customer: Record Customer;
    CustomerCard: TestPage "Customer Card";
begin
    // [Scenario] New customer gets initial reward points

    // [Given] Customer Rewards is activated
    Initialize();
    ActivateCustomerRewards();

    // [When] Create new customer
    LibrarySales.CreateCustomer(Customer);

    // [Then] Customer has zero reward points
    CustomerCard.OpenView();
    CustomerCard.GoToRecord(Customer);
    Assert.AreEqual(0, CustomerCard.RewardPoints.AsInteger(), 'New customer should have 0 points');
end;
```

## Test Codeunit Structure

```al
codeunit 50100 "My Feature Tests"
{
    // [FEATURE] [My Feature]

    Subtype = Test;
    TestPermissions = Disabled;

    var
        Assert: Codeunit Assert;
        LibrarySales: Codeunit "Library - Sales";
        LibraryRandom: Codeunit "Library - Random";
        IsInitialized: Boolean;

    local procedure Initialize()
    begin
        if IsInitialized then
            exit;

        // Setup code here
        LibraryRandom.SetSeed(1);

        IsInitialized := true;
    end;

    [Test]
    procedure TestMyScenario()
    begin
        // Test implementation
    end;
}
```

## Handler Method Signatures

| Handler Type              | Signature                                                                              |
| ------------------------- | -------------------------------------------------------------------------------------- |
| MessageHandler            | `procedure Handler(Message: Text[1024])`                                               |
| ConfirmHandler            | `procedure Handler(Question: Text[1024]; var Reply: Boolean)`                          |
| StrMenuHandler            | `procedure Handler(Options: Text[1024]; var Choice: Integer; Instruction: Text[1024])` |
| PageHandler               | `procedure Handler(var ThePage: TestPage "Page Name")`                                 |
| ModalPageHandler          | `procedure Handler(var ThePage: TestPage "Page Name")`                                 |
| ReportHandler             | `procedure Handler(var TheReport: Report "Report Name")`                               |
| RequestPageHandler        | `procedure Handler(var RequestPage: TestRequestPage "Report Name")`                    |
| HyperlinkHandler          | `procedure Handler(Hyperlink: Text[1024])`                                             |
| SendNotificationHandler   | `procedure Handler(var Notification: Notification): Boolean`                           |
| RecallNotificationHandler | `procedure Handler(var Notification: Notification): Boolean`                           |

## Best Practices Summary

1. **Prefer unit tests over integration tests** - Unit tests run in <100ms; integration tests take 500ms+
2. **Test YOUR code, not Microsoft's** - Abstract dependencies with interfaces
3. **Test both success and failure paths** - Positive tests verify correct behavior; negative tests verify error handling
4. **Use random data** - Only hardcode values when the specific value matters
5. **Keep tests independent** - Each test should work regardless of execution order
6. **Leave database clean** - Use Initialize to reset state; use TestIsolation for rollback
7. **No user interaction** - All UI must be handled by handler methods
8. **Fast execution** - All tests should run under 10 seconds total (unit tests)
9. **Limit test count** - No more than 100 test methods per codeunit; ~25 codeunits per suite
10. **Test permissions** - Verify functionality works without SUPER permissions
11. **Mock external calls** - Never make real external service requests in tests
12. **Document with ATDD** - Use Given-When-Then comments for readability
13. **Design for testability** - Use interfaces and dependency injection

## Detailed Documentation

Read individual reference files for detailed explanations and code examples:

### Core Testing

- [Test Codeunits and Methods](references/TEST-CODEUNITS.md) - Codeunit setup, attributes, runners
- [UI Handlers](references/HANDLERS.md) - All handler types with examples
- [Test Pages](references/TEST-PAGES.md) - TestPage usage and patterns
- [Assertions](references/ASSERTIONS.md) - Assert methods and ASSERTERROR
- [Data Setup](references/DATA-SETUP.md) - Library codeunits, random data
- [Test Isolation](references/ISOLATION.md) - Transaction models, rollback
- [ATDD Patterns](references/PATTERNS.md) - Given-When-Then documentation

### Advanced Topics

- [Test Types](references/TEST-TYPES.md) - Unit vs integration tests, test doubles, testing pyramid
- [Testability](references/TESTABILITY.md) - Dependency injection, interfaces, designing testable code
- [HTTP Mocking](references/HTTP-MOCKING.md) - HttpClientHandler, mocking external API calls
