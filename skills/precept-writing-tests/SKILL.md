---
name: precept-writing-tests
description: "Use when adding or changing a test in a Precept project: a [TestSuite] class, a .feature file or [Binding] step, an assertion, or the precept.json setting or Startup registration the test needs."
---

# Writing a Precept test

## Steps

1. **Read `precept.json` and the `IPreceptStartup` class.** They say which modules the suite has. A module works only when its `AddPrecept…` is called there, and only then is its settings section read.
2. **Read one existing test in the same area and match it**: plain C# or Gherkin, page objects or screenplay, the tags already in use. Do not introduce a second style.
3. **Look the module's API up; do not guess it.** `precept_explain_module` (`precept_explain_testdata` for data files and `{{tokens}}`) answers from the installed version. Without the MCP server, read https://autom3tion.github.io/precept-docs/.
4. **Write the test.**
5. **Prove it.** Build, check `dotnet run --project <dir> -- --list-tests` lists it once, run it with `--precept-filter "name:*<part of the name>*"`, and report the real outcome.

## Shape

```csharp
using Precept;
using Precept.Api;

[TestSuite("Checkout")]
[TestCategory("smoke")]
public sealed class CheckoutTests
{
    [BeforeTest] public Task Reset() => Task.CompletedTask;    // [BeforeSuite]/[AfterSuite] are static

    [Test("A customer can check out")]
    public async Task Checkout()
    {
        var response = await Rest.Post("/orders").WithJsonBody(new { sku = "ABC" }).SendAsync();

        await Assert.That(response).ToBeSuccessfulAsync();
        await Assert.That(response).JsonAt("status").ToBeEqualAsync("created");
    }

    [TestCase(1, 2, 3)]                                        // implies [Test]; one test per row
    public async Task Adds(int left, int right, int total) =>
        await Assert.That(left + right).ToBeEqualAsync(total);
}
```

## Rules

1. **Every assertion is awaited**: `await Assert.That(x).ToBe…Async(y)`. Modifiers go in front: `.Not`, `.Because("…")`, `.Within(TimeSpan)`. There is no `.Should()` and no synchronous form.
2. **Never sleep.** A value that arrives later is `await Assert.Eventually(() => getValue()).ToBe…Async(y)`. An assertion on a Playwright locator already retries.
3. **Several checks on one object** go in `await Assert.MultipleAsync(async () => { … })`, so every failure is reported.
4. **Reach the system through the entry points** — `Rest`, `Browser`, `Db`, `TestData`, `Rpc` — never a hand-built `HttpClient`, browser or connection. They are disposed with the test.
5. **`PreceptTestContext.Current` is the running test**: `Log(...)`, `AttachAsync(path)`, `RegisterForDisposal(x)`, `CancellationToken`.
6. **`[Retry(n)]` counts reruns**, so `[Retry(2)]` is three attempts. There is no per-test timeout unless `[Timeout(ms)]` is set.
7. **Settings go in `precept.json`** or a `precept.{environment}.json` overlay; secrets go in `PRECEPT_*` variables (`Web:Headless` is `PRECEPT_WEB__HEADLESS`). A suite's own section is bound with `services.AddPreceptSettings<T>()`.
8. **`ConfigureServices` holds registrations only** — it runs on discovery too. One-off setup goes in `BeforeRunAsync`.

## Gherkin

Needs the `Precept.Reqnroll` package. Bindings are ordinary Reqnroll `[Binding]` classes using the rules above.

- A tag becomes a category: `@smoke` is what `--precept-filter "smoke"` selects.
- Reqnroll is configured in the `Reqnroll` section of `precept.json`, not a `reqnroll.json`.
- A `{{token}}` in a step argument is plain text until the step calls `TestData.Resolve(text)`.
- `*.feature.cs` is generated: never edit or commit it.

Done when the test is listed once, has been run, and the outcome is reported as it happened — a failure against an environment you cannot reach is not a pass.
