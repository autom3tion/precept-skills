---
name: precept-gherkin
description: "Write a Gherkin feature file and its Reqnroll step bindings in a Precept project that references Precept.Reqnroll, and configure Reqnroll from precept.json. Use for any .feature file, [Binding] class or Reqnroll setting."
---

# Gherkin with Precept.Reqnroll

Gherkin is optional: the `Precept.Reqnroll` package turns each `.feature` file into one `[TestSuite]` class at build time. Everything Reqnroll supports — data tables, `ScenarioContext`, constructor injection, hooks — works unchanged.

```gherkin
@smoke @web
Feature: Sign in

  Scenario Outline: A user signs in
    Given I open "/login"
    When I sign in as "<user>"
    Then the greeting should be "<greeting>"

    Examples:
      | user | greeting   |
      | ada  | Hello, Ada |
```

```csharp
using Precept;
using Precept.Web;
using Reqnroll;

[Binding]
public sealed class SignInSteps
{
    [Given("I open {string}")]
    public Task GivenIOpen(string path) => Browser.GoToAsync(path);

    [Then("the greeting should be {string}")]
    public async Task ThenTheGreetingShouldBe(string expected)
    {
        var page = await Browser.PageAsync();
        await Assert.That(page.GetByTestId("greeting"), "the greeting").ToHaveTextAsync(expected);  // retries on the live page
    }
}
```

## Rules

1. Steps use Cucumber expressions (`{string}`, `{int}`) and awaited Precept assertions; a step never sleeps.
2. Tags become `[TestCategory]`, so `@smoke` is what `--precept-filter "smoke"` and the `Filter` section select on. A tag listed in `Reqnroll:AddNonParallelizableMarkerForTags` makes its scenarios `[NonParallelizable]`.
3. One feature file is one scheduling unit under the default `ParallelScope: Class`; a suite of few long features gains nothing from a high `MaxParallelism` until `ParallelScope` is `Test`.
4. Reqnroll is configured from the `Reqnroll` section of `precept.json`, not `reqnroll.json`. Runtime keys (`StopAtFirstError`, `MissingOrPendingStepsOutcome`, `TraceTimings`, `BindingCulture`, `BindingAssemblies`) accept overlays and `PRECEPT_REQNROLL__*`. Build-time keys (`FeatureLanguage`, `AllowRowTests`, `AddNonParallelizableMarkerForTags`, `AllowDebugGeneratedFiles`, `DisableFriendlyTestNames`) are read from the project's own `precept.json` while the code-behind compiles and ignore overlays.
5. Step traces land in the test's own output, not the console. To see a passing scenario's steps: `dotnet run --project <dir> -- --output Detailed --show-stdout All`.
6. Keep `.feature` files out of `bin/`; a copy there is generated twice and every scenario is discovered twice. `*.feature.cs` is generated and belongs in `.gitignore`. If generated code looks stale, delete `**/*.feature.cs` and rebuild.
7. A missing step definition reports as pending by default (`MissingOrPendingStepsOutcome`); set it to `Error` on CI.
8. A `{{token}}` in a step argument — `When I create the brand "Autotest_{{unique}}"` — is plain text until the step definition resolves it: `TestData.Resolve(name)`. Nothing expands step arguments, doc strings or table cells on the step's behalf, so typed parameters and tables bound to objects reach Reqnroll's conversion exactly as written.

Done when the scenario is listed by `--list-tests` under its feature name and "Go to test" lands on the `Scenario:` line of the `.feature` file.
