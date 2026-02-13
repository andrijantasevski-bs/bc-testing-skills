# HTTP Mocking in AL Tests

Mock HTTP calls to test integrations with external services without network dependencies. This ensures consistent, isolated tests.

## Why Mock HTTP Requests?

When testing code that calls external APIs:

- **Network dependencies** - Tests fail if API is down
- **Unpredictable responses** - External data changes
- **Slow execution** - Network latency adds up
- **Rate limiting** - Too many calls may block API access
- **Data consistency** - Can't control external state

Mocking eliminates these issues.

## HttpClientHandler

The `[HttpClientHandler]` attribute (introduced in AL 15.0) intercepts HTTP requests during tests:

```al
[Test]
[HandlerFunctions('HttpClientHandler')]
procedure TestExternalAPICall()
var
    MyService: Codeunit "My External Service";
begin
    // [Scenario] External API returns customer data

    // [Given] API is configured
    Initialize();

    // [When] Calling external service
    MyService.GetCustomerData('CUST001');

    // [Then] Data is processed correctly
    Assert.AreEqual(ExpectedValue, ActualValue, 'API response not processed');
end;

[HttpClientHandler]
procedure HttpClientHandler(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    // Mock a successful response
    if (Request.RequestType = HttpRequestType::Get) and
       Request.Path.Contains('/customers/')
    then begin
        Response.HttpStatusCode := 200;
        Response.ReasonPhrase := 'OK';
        Response.Content.WriteFrom('{"name": "Test Customer", "id": "CUST001"}');
        exit(false); // Use mocked response
    end;

    exit(true); // Fall through to real endpoint
end;
```

## Handler Method Signature

```al
procedure HandlerName(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
```

**Parameters:**

| Parameter | Type                    | Description                                            |
| --------- | ----------------------- | ------------------------------------------------------ |
| Request   | TestHttpRequestMessage  | Contains request details (method, path, headers, body) |
| Response  | TestHttpResponseMessage | Set status code, headers, and body for mock response   |

**Return Value:**

| Return  | Behavior                                                    |
| ------- | ----------------------------------------------------------- |
| `false` | Use the mocked response (request not sent to real endpoint) |
| `true`  | Allow request through to actual endpoint                    |

## TestHttpRequestMessage Properties

Access request details to determine appropriate mock response:

```al
Request.RequestType      // HttpRequestType enum (Get, Post, Put, Delete, etc.)
Request.Path             // Full URL path
Request.Method           // HTTP method as Text
Request.GetHeaders()     // HttpHeaders collection
Request.Content          // Request body (HttpContent)
```

## TestHttpResponseMessage Properties

Configure mock response:

```al
Response.HttpStatusCode := 200;              // Status code
Response.ReasonPhrase := 'OK';               // Status phrase
Response.Content.WriteFrom('{"data": []}');  // Response body
Response.GetHeaders().Add('x-custom', 'val'); // Headers
```

## TestHttpRequestPolicy Property

Control outbound HTTP behavior at codeunit level:

```al
codeunit 50100 "My API Tests"
{
    Subtype = Test;
    TestHttpRequestPolicy = AllowOutboundFromHandler;

    // Tests here
}
```

### Policy Values

| Policy                     | Behavior                                                   |
| -------------------------- | ---------------------------------------------------------- |
| `BlockOutboundRequests`    | Any uncaught HTTP request throws error                     |
| `AllowOutboundFromHandler` | All requests go through handler; handler can allow through |
| `AllowAllOutboundRequests` | All HTTP requests allowed without restriction              |

**Recommendation:** Use `BlockOutboundRequests` or `AllowOutboundFromHandler` for test isolation.

## Mocking Different Response Scenarios

### Success Response

```al
[HttpClientHandler]
procedure MockSuccess(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    Response.HttpStatusCode := 200;
    Response.Content.WriteFrom('{"status": "success", "data": {"id": 123}}');
    exit(false);
end;
```

### Error Responses

```al
[HttpClientHandler]
procedure MockUnauthorized(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    Response.HttpStatusCode := 401;
    Response.ReasonPhrase := 'Unauthorized';
    Response.Content.WriteFrom('{"error": "Invalid API key"}');
    exit(false);
end;

[HttpClientHandler]
procedure MockServerError(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    Response.HttpStatusCode := 500;
    Response.ReasonPhrase := 'Internal Server Error';
    exit(false);
end;

[HttpClientHandler]
procedure MockTimeout(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    Response.HttpStatusCode := 408;
    Response.ReasonPhrase := 'Request Timeout';
    exit(false);
end;
```

### Route-Based Mocking

```al
[HttpClientHandler]
procedure MultiRouteMockHandler(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
begin
    case true of
        Request.Path.Contains('/api/customers'):
            begin
                Response.HttpStatusCode := 200;
                Response.Content.WriteFrom('[{"id": "C001"}, {"id": "C002"}]');
                exit(false);
            end;
        Request.Path.Contains('/api/orders'):
            begin
                Response.HttpStatusCode := 200;
                Response.Content.WriteFrom('{"orderId": "ORD001"}');
                exit(false);
            end;
        Request.Path.Contains('/api/error'):
            begin
                Response.HttpStatusCode := 400;
                exit(false);
            end;
    end;

    // Unexpected request - fail the test
    exit(false);
end;
```

## Using LibraryVariableStorage for Dynamic Mocking

Pass expected values to handler:

```al
[Test]
[HandlerFunctions('DynamicHttpHandler')]
procedure TestDynamicResponse()
var
    LibraryVariableStorage: Codeunit "Library - Variable Storage";
begin
    // [Given] Expected response
    LibraryVariableStorage.Enqueue(200); // Status code
    LibraryVariableStorage.Enqueue('{"name": "Dynamic"}'); // Body

    // [When] Make API call
    MyService.CallAPI();

    // [Then] Response processed
    // ... assertions
end;

[HttpClientHandler]
procedure DynamicHttpHandler(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
var
    LibraryVariableStorage: Codeunit "Library - Variable Storage";
begin
    Response.HttpStatusCode := LibraryVariableStorage.DequeueInteger();
    Response.Content.WriteFrom(LibraryVariableStorage.DequeueText());
    exit(false);
end;
```

## Complete Test Example

```al
codeunit 50100 "External API Tests"
{
    Subtype = Test;
    TestHttpRequestPolicy = BlockOutboundRequests;

    var
        Assert: Codeunit Assert;
        LibraryVariableStorage: Codeunit "Library - Variable Storage";
        IsInitialized: Boolean;

    local procedure Initialize()
    begin
        LibraryVariableStorage.Clear();
        if IsInitialized then
            exit;
        IsInitialized := true;
    end;

    [Test]
    [HandlerFunctions('SuccessHttpHandler')]
    procedure TestAPICallSuccess()
    var
        ExchangeService: Codeunit "Currency Exchange Service";
        Result: Decimal;
    begin
        // [Scenario] Currency exchange API returns rate

        // [Given] API configured
        Initialize();

        // [When] Getting exchange rate
        Result := ExchangeService.GetRate('USD', 'EUR');

        // [Then] Rate is returned
        Assert.AreEqual(0.85, Result, 'Exchange rate not correct');
    end;

    [Test]
    [HandlerFunctions('ErrorHttpHandler')]
    procedure TestAPICallHandlesError()
    var
        ExchangeService: Codeunit "Currency Exchange Service";
    begin
        // [Scenario] API error is handled gracefully

        // [Given] API will return error
        Initialize();

        // [When] Getting exchange rate
        asserterror ExchangeService.GetRate('INVALID', 'EUR');

        // [Then] Proper error raised
        Assert.ExpectedError('Invalid currency code');
    end;

    [HttpClientHandler]
    procedure SuccessHttpHandler(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
    begin
        Response.HttpStatusCode := 200;
        Response.Content.WriteFrom('{"rate": 0.85}');
        exit(false);
    end;

    [HttpClientHandler]
    procedure ErrorHttpHandler(Request: TestHttpRequestMessage; var Response: TestHttpResponseMessage): Boolean
    begin
        Response.HttpStatusCode := 400;
        Response.Content.WriteFrom('{"error": "Invalid currency code"}');
        exit(false);
    end;
}
```

## Best Practices

1. **Always mock in tests** - Never call real APIs in automated tests
2. **Use BlockOutboundRequests** - Ensures no accidental real requests
3. **Test all response codes** - Success, client errors (4xx), server errors (5xx)
4. **Test timeout scenarios** - Verify graceful handling of timeouts
5. **Match request specifics** - Check method, path, and headers in handler
6. **Fail on unexpected requests** - Helps catch missing mocks
7. **Use descriptive handler names** - Match test purpose (e.g., `MockPaymentSuccess`)

## Availability Note

`HttpClientHandler` is currently available for **OnPrem** instances only. For SaaS testing of external integrations, consider:

- Interface-based dependency injection
- Event-based mocking
- Test doubles (stubs/mocks)

See [TESTABILITY.md](TESTABILITY.md) for decoupling patterns that work on both SaaS and OnPrem.
