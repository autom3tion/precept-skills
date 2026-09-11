---
name: precept-migrating
description: "Migrate, port or convert an existing .NET test suite (NUnit, xUnit, MSTest, SpecFlow, FluentAssertions, Selenium, RestSharp) to Precept, with every old test accounted for. Use when moving tests into a Precept project or when a task says migrate, port or convert."
---

# Migrating a suite to Precept

## Steps

1. **Inventory.** List every test in the old project — `dotnet test --list-tests` on it, or count `[Test]`/`[Fact]`/`[TestMethod]` attributes and `Scenario:` lines — and keep the list. It is the completion criterion.
2. **Stand the target up first.** A `global.json` at the repository root with `"test": { "runner": "Microsoft.Testing.Platform" }`, the packages, a `precept.json`, and an `IPreceptStartup` calling each `AddPrecept…` the suite will need (`precept-settings`). Confirm `dotnet run --project <dir> -- --list-tests` works on an empty project before moving a single test.
3. **Map constructs** with the table below, one class or feature at a time. Keep the old test's name and category so the inventory can be matched back.
4. **Replace waits with assertions.** Every `Thread.Sleep`, `WebDriverWait`, polling loop and `Task.Delay` becomes `Assert.Eventually(() => …)` or a web-first assertion on a locator (`precept-tests`, `precept-modules`).
5. **Decide parallelism deliberately.** Precept runs classes in parallel by default. A fixture that relied on ordering, shared static state or a single database row gets `[NonParallelizable]` or a rewrite; do not lower `MaxParallelism` for the whole suite to hide it.
6. **Move configuration.** `.runsettings` is not read by the platform at all; `app.config` and `appsettings.json` keys go into `precept.json` sections — Precept's for the modules, a section of your own bound with `AddPreceptSettings<T>()` for the rest — and secrets into `PRECEPT_*` variables.
7. **Account for every entry in the inventory**: migrated, merged into another test, or dropped with a written reason. `--list-tests` on the new project must show the migrated count, and a run must be green or every failure classified as product, environment or migration.

## Mapping

| From | To |
| --- | --- |
| NUnit `[TestFixture]`, `[Test]`, `[TestCase]` | `[TestSuite]`, `[Test]`, `[TestCase]` |
| NUnit `[SetUp]`/`[TearDown]`, `[OneTimeSetUp]`/`[OneTimeTearDown]` | `[BeforeTest]`/`[AfterTest]`, static `[BeforeSuite]`/`[AfterSuite]` |
| NUnit `[Category]`, `[Ignore]`, `[NonParallelizable]`, `[Timeout]` | same names in Precept |
| NUnit `[Retry(3)]` (three tries in all) | `[Retry(2)]` — Precept counts reruns |
| NUnit `Assert.Ignore()`, `Assert.Inconclusive()` | throw `PreceptIgnoreException`, `PreceptInconclusiveException` |
| `TestContext.WriteLine`, `TestContext.AddTestAttachment` | `PreceptTestContext.Current.Log(...)`, `.AttachAsync(path)` |
| xUnit `[Fact]`, `[Theory]` + `[InlineData]` | `[Test]`, `[TestCase]` |
| xUnit constructor/`IDisposable`, `IClassFixture<T>` | `[BeforeTest]`/`[AfterTest]`; a static `[BeforeSuite]` or a singleton registered in `ConfigureServices` |
| xUnit `[Trait("Category", "x")]`, `ITestOutputHelper` | `[TestCategory("x")]`, `PreceptTestContext.Current.Log` |
| MSTest `[TestClass]`, `[TestMethod]`, `[DataRow]` | `[TestSuite]`, `[Test]`, `[TestCase]` |
| MSTest `[TestInitialize]`/`[ClassInitialize]` and cleanups | `[BeforeTest]`/`[BeforeSuite]` and their after-hooks |
| SpecFlow `TechTalk.SpecFlow`, `SpecFlow.NUnit`/`.xUnit` packages, `specflow.json` | `Reqnroll` namespace, the `Precept.Reqnroll` package, the `Reqnroll` section of `precept.json` (`precept-gherkin`); `[Binding]` classes and hooks unchanged |
| FluentAssertions `x.Should().Be(y)`, `.NotBeNull()`, `.Contain(i)`, `.HaveCount(n)`, `.BeTrue()` | `await Assert.That(x).ToBeEqualAsync(y)`, `.Not.ToBeNullAsync()`, `.ToContainAsync(i)`, `.ToHaveCountAsync(n)`, `.ToBeTrueAsync()` |
| FluentAssertions `act.Should().Throw<T>()`, `using new AssertionScope()`, `.Because` | `await Assert.That(act).ToThrowAsync<T>()`, `Assert.MultipleAsync(...)`, `.Because("…")` in front of the assertion |
| Selenium `IWebDriver`, `FindElement(By…)`, `WebDriverWait`, `driver.Quit()` | `IPageContext.Page` in page objects (`precept-page-objects`), `GetByRole`/`GetByLabel`/`Locator`, a web-first assertion, nothing — the session is disposed with the test |
| RestSharp/`HttpClient` request and `JsonConvert.DeserializeObject<T>` | `Rest.Get("/path").SendAsync()`, `Assert.That(response).Json<T>()` / `JsonAt("field")` |
| `.runsettings`, `appsettings.json`, `Environment.GetEnvironmentVariable` | `precept.json` and overlays, a bound settings class, `PRECEPT_*` and `{{env:NAME}}` in test data |

Done when every inventory line has an outcome, `--list-tests` on the new project matches the migrated count, the suite runs without a sleep in it, and no `.runsettings` or `specflow.json` is left behind to suggest it is still read.
