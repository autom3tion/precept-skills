---
name: precept-modules
description: "Use a Precept module in a test: Rest (HTTP), Browser and page objects (Playwright), Db (ADO.NET), TestData (data files and factories), and Screenplay actors. Use when a step or test needs to call an API, drive a page, query a database or read test data."
---

# Precept modules

Each module is one package, one registration in `IPreceptStartup.ConfigureServices`, one static entry point, and one section of `precept.json`. Every module hands its resources to the test's scope, so nothing is disposed by hand.

| Module | Register | Entry point | Section |
| --- | --- | --- | --- |
| `Precept.Api` | `services.AddPreceptApi()` | `Rest` | `Api` |
| `Precept.Web` | `services.AddPreceptWeb()` | `Browser` | `Web` |
| `Precept.Data` | `services.AddPreceptData(cs => new NpgsqlConnection(cs))` | `Db` | `ConnectionStrings` |
| `Precept.TestData` | `services.AddPreceptTestData()` | `TestData` | `TestData` |
| `Precept.Grpc` | `services.AddPreceptGrpc()` | `Rpc` | `Grpc` |
| `Precept.Screenplay` | none (brings `Precept.Web`) | `Actor` | — |

## HTTP

```csharp
var response = await Rest.Post("/orders").WithJsonBody(new { sku = "ABC" }).SendAsync();
await Assert.That(response).ToBeSuccessfulAsync();
await Assert.That(response).JsonAt("customer.email").ToEndWithAsync("@example.com");
await Assert.That(response).Json<Order>().ToSatisfyAsync(o => o.Lines.Count == 2, "have two lines");
await Assert.Eventually(() => Rest.Get($"/jobs/{id}").SendAsync()).Within(TimeSpan.FromMinutes(2)).ToHaveJsonValueAsync("state", "Completed");
```

XML goes through `WithXmlBody(dataContract)`, `Xml<T>()` and `XmlAt(xpath)`; XPath prefixes are declared in `Api:XmlNamespaces`. Every failure carries the request line and body.

## Browser

```csharp
var page = await Browser.PageAsync();                       // the test's own session, opened on first use
await Browser.GoToAsync("/login");                          // relative to Web:BaseUrl
await Assert.That(page.GetByRole(AriaRole.Alert)).Not.ToBeVisibleAsync();   // Playwright's retrying expect, no wait needed
await Assert.That(page.GetByTestId("total"), "the basket total").ToHaveTextAsync("£42.00");

var bob = await Browser.StartAsync("bob");                  // a second isolated identity in the same test
```

Page objects inject `IPageContext` and read `.Page` on every use. Anything Playwright offers that Precept does not mirror goes through `ToSatisfyAsync("be …", expect => expect.ToXAsync(...))`. Failure screenshots, traces and console errors are captured by the module; `Web:FailOnConsoleErrors` turns a logged error into a failure after the last line.

## Database

```csharp
await Db.ExecuteAsync("insert into clients (id, name) values (@id, @name)", ("id", id), ("name", "Acme"));
var rows = await Db.QueryAsync("select id, name from clients where active = 1");
await Assert.Eventually(() => Db.ScalarAsync<long>("select count(*) from outbox where sent = 0")).ToBeEqualAsync(0L);
```

Every method takes a `database` name defaulting to `"Default"`. `Db.InRolledBackTransactionAsync` leaves nothing behind.

## Test data

`testdata.json` (plus `testdata.{environment}.json` and `PRECEPT_TESTDATA__*`) holds named records; the set is the plural of the type name (`TestUser` → `users`). Values may carry `{{unique}}`, `{{guid}}`, `{{now:yyyy-MM-dd}}`, `{{env:VAR|fallback}}`, `{{setting:Api:BaseUrl}}`, `{{ref:users:admin:username}}`. `TestData.Get<T>("name")` is stable within a scenario; `New<T>` gives a fresh one; `CreateAsync<T>` runs the registered factory and deletes on teardown. Data files need `<None Update="testdata*.json" CopyToOutputDirectory="PreserveNewest" />`.

## Screenplay

```csharp
using Precept.Screenplay; using Precept.Screenplay.Web;
var alice = await Actor.Named("alice").WhoCanBrowseTheWebAsync();
await alice.AttemptsToAsync(Navigate.To("/login"), Enter.The("ada@example.com").Into(SignInPage.Email), Click.On(SignInPage.Submit));
await Assert.That(alice.Sees(Dashboard.Greeting)).ToHaveTextAsync("Hello, Ada");
```

Targets are `Target.The("the email box", page => page.GetByLabel("Email"))`, declared `static readonly`. An interaction is an immutable class with its factory on itself; a task composes interactions. Actors belong to the test attempt, so a retry starts with an empty cast.

Done when the module is registered in `Startup`, its section is present in `precept.json`, and the test reads through the entry point rather than constructing a client, browser or connection itself.
