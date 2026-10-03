---
applyTo: "**"
description: "How to write, configure and run tests on the Precept test framework."
---

# Precept

This suite runs on Precept, a .NET 10 test framework on Microsoft.Testing.Platform. Documentation: https://autom3tion.github.io/precept-docs/

- A test is a `[TestSuite]` class with `[Test]` or `[TestCase(...)]` methods, or a `.feature` file with Reqnroll bindings.
- Every assertion is awaited: `await Assert.That(x).ToBeEqualAsync(y)`. Wait with `Assert.Eventually(() => …)`, never a sleep.
- Reach the system through `Rest`, `Browser`, `Db`, `TestData`, `Rpc`; never construct a client, browser or connection.
- Settings live in `precept.json`, overlaid by `precept.{environment}.json`, then `PRECEPT_*` variables. A module's section is read only when its `AddPrecept…` is called in `IPreceptStartup`.
- Run with `dotnet run --project <dir> -- --precept-filter "smoke and not wip"`; list with `--list-tests`; choose an environment with `--precept-environment`. A filter must not start with `@`.

Load the skill for the task:

| Task | Skill |
| --- | --- |
| Add or change a test, step or setting | `precept-writing-tests` |
| A run fails, finds nothing or ignores a setting | `precept-debugging-runs` |
| Move tests in from another framework | `precept-migrating` |
| Update the Precept packages | `precept-upgrading` |
| Set up or fix an Azure Pipelines run | `precept-pipelines` |
