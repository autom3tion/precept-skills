---
name: precept-tests
description: "Write a plain C# Precept test: [TestSuite]/[Test] classes, hooks, outcomes, retries and awaited assertions. Use when adding or editing a test class or an assertion in a Precept project."
---

# Writing a Precept test

```csharp
using System.Net;
using Precept;
using Precept.Api;

[TestSuite("Checkout")]
[TestCategory("smoke")]
public sealed class CheckoutTests
{
    [BeforeSuite] public static Task Warm() => Task.CompletedTask;   // static, once per class
    [BeforeTest]  public Task Reset() => Task.CompletedTask;         // per test; [AfterTest] runs even on failure

    [Test("A customer can check out")]
    [Retry(2, DelayMilliseconds = 200)]                              // reruns, not runs: 0 means no rerun
    public async Task Checkout()
    {
        var response = await Rest.Post("/orders").WithJsonBody(new { sku = "ABC", quantity = 2 }).SendAsync();

        await Assert.That(response).ToHaveStatusAsync(HttpStatusCode.Created);
        await Assert.That(response).JsonAt("status").ToBeEqualAsync("created");
    }

    [TestCase(1, 2, 3)]
    [TestCase(-5, 5, 0)]                                             // implies [Test]; one case per row
    public async Task Adds(int left, int right, int total) =>
        await Assert.That(left + right).ToBeEqualAsync(total);
}
```

## Rules

1. Every assertion is `await Assert.That(subject).ToBe…Async(...)`. Modifiers go in front: `.Not`, `.Because("…")`, `.Within(TimeSpan)`. Name a subject when the expression is not the point: `Assert.That(total, "the basket total")`.
2. Something that becomes true later is `await Assert.Eventually(() => getValue()).Within(…).ToBe…Async(...)`. Never `Task.Delay` before an assertion.
3. Several checks on one object go in `await Assert.MultipleAsync(async () => { … })` so every failure is reported.
4. A one-off condition is `ToSatisfyAsync(predicate, "be …")`; a reusable one is an extension method on `Subject<T>` (namespace `Precept.Assertions`) that calls `subject.ExpectAsync(Expectation.For<T>("be …", check, describeActual))`. `.Not`, `Eventually` and `MultipleAsync` then work on it with no further code.
5. Runtime outcomes are exceptions: `PreceptIgnoreException` (skipped), `PreceptPendingException`, `PreceptInconclusiveException`. `[Ignore]` skips without running.
6. `[NonParallelizable]` runs the class alone after everything else. `[Timeout(ms)]` is opt-in; there is no per-test limit by default, only `RunTimeoutMinutes` on the run.
7. `PreceptTestContext.Current` is the running test: `Log(...)` lines reach the TRX and reporters, `AttachAsync(path)` records a file, `RegisterForDisposal(x)` disposes after the after-hook, `RegisterVerification(() => …)` fails an otherwise passing test after its last line. Observe `CancellationToken` in waits and HTTP calls.
8. Discovery is strict: a hook with the wrong signature or a parameterised test without `[TestCase]` fails the whole session rather than being skipped.

Done when the test builds, is listed by `dotnet run --project <dir> -- --list-tests`, and its assertions read as the sentence their failure would print.
