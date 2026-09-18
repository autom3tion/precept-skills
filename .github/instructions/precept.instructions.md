---
applyTo: "**"
description: "How to write, configure and run tests on the Precept test framework."
---

# Precept

Precept is a .NET 10 test automation framework on Microsoft.Testing.Platform: its own runner (discovery, scheduling, retries, lifecycle, reporting) plus the modules an end-to-end suite needs — HTTP (`Precept.Api`), browsers (`Precept.Web`), databases (`Precept.Data`), gRPC, containers, test data, per-environment configuration and reporting integrations. Gherkin is one optional module (`Precept.Reqnroll`), not the headline. Full documentation: https://autom3tion.github.io/precept-docs/

## Writing a test

- A test is a class with `[TestSuite]` and methods with `[Test]` (or `[TestCase(...)]`). Add `using Precept;` and the module namespace (`Precept.Web`, `Precept.Api`).
- Every assertion is awaited: `await Assert.That(x).ToBeEqualAsync(y)`. There is no `.Should()` and no synchronous overload. Wait for a value with `Assert.Eventually(() => …)`, never with a sleep.
- Module entry points are static: `Rest`, `Browser`, `Db`, `TestData`, `Rpc`. `PreceptTestContext.Current` is the running test — `Log(...)`, `AttachAsync(...)`, `RegisterForDisposal(...)`, `CancellationToken`.
- Settings live in `precept.json` beside the project, overlaid by `precept.{environment}.json`, then `PRECEPT_*` variables (`Web:Headless` → `PRECEPT_WEB__HEADLESS`). A module's section is only read when its `AddPrecept…` is called in `IPreceptStartup.ConfigureServices`.
- Run with `dotnet test --project <csproj>` (needs the MTP runner in the root `global.json`) or `dotnet run --project <dir> -- --precept-filter "smoke and not wip"`. A filter value must not start with `@`.

Use the skills under `.github/skills/` for the detail of each of these — including `precept-page-objects` for a page object and `precept-migrating` for tests moved in from another framework, `precept-upgrading` for a move to a newer Precept, `precept-pipelines` for Azure Pipelines — and the `Precept` agent in `.github/agents/` for a change that spans them.
