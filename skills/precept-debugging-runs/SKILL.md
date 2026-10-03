---
name: precept-debugging-runs
description: "Use when a Precept run misbehaves: a test fails or is flaky, no tests are found, tests are listed twice, a setting or filter has no effect, the wrong environment is used, parallelism does not change, or a run times out."
---

# Debugging a Precept run

## Steps

1. **Get the evidence.** Rerun the one test with its full output:
   `dotnet run --project <dir> -- --precept-filter "name:*<name>*" --output Detailed --show-stdout All --report-trx`.
   Logs and Reqnroll steps belong to the test's output, not the console. Screenshots and traces are in the test's folder under `ArtifactDirectory`; a rerun writes to `attempt-n` beneath it.
2. **Ask before investigating by hand.** `precept_explain_failure` with the error text; `precept_diagnose_project` when there is no error at all. Without the MCP server, use the table.
3. **Classify the failure**: product, environment, or test. Say which, with the evidence.
4. **Fix the cause**, rerun the test, then run its whole class.

## Symptoms

| Symptom | Cause |
| --- | --- |
| "Testing with VSTest target is no longer supported", or `--project` is an unknown switch | No `"test": { "runner": "Microsoft.Testing.Platform" }` in the `global.json` at the repository root. A copy beside the project is not read. |
| The run tries to open a response file | The filter value starts with `@`. Write `"smoke and not @slow"`. |
| Build is green, no tests found, exit code 8 | The filter matched nothing (`precept_validate_filter`), or `Filter:Exclude` in `precept.json` removed them. For Gherkin: delete `**/*.feature.cs` and rebuild. |
| Every scenario is listed twice | A `.feature` file under `bin/` or `obj/` is being compiled. |
| A setting has no effect | The module's `AddPrecept…` is not called in `IPreceptStartup`; or another overlay won (`precept_explain_environment`); or the key is misspelt — unknown keys are never an error. `.runsettings` is not read at all. |
| A `Reqnroll` setting works in `precept.json` but not in an overlay | Build-time keys (`FeatureLanguage`, `AllowRowTests`, …) are read once, at compile time. |
| Raising `MaxParallelism` changes nothing | Parallelism is per class, and one feature file is one class (`precept_explain_parallelism`). `ParallelScope: Test` schedules tests individually. |
| The whole session fails before any test runs | Discovery is strict: a hook with the wrong signature, or a test with parameters and no `[TestCase]`. |
| Every test fails with the same exception | `BeforeRunAsync` threw. |
| A test passed but its log shows a failure | It was retried; the log has one `[Attempt n of m]` block per attempt. |
| Tests fail with "the run exceeded its … minute limit" | `RunTimeoutMinutes` was reached. Find the test that hung; do not raise the limit first. |
| A scenario is pending, not failed | A step has no binding. `Reqnroll:MissingOrPendingStepsOutcome` decides how that is reported. |
| A reporter does nothing locally | Reporters are CI-only by default. `PRECEPT_ISCONTINUOUSINTEGRATION=true` turns them on. |
| Fails only in parallel | Shared state between classes. Fix it, or mark the class `[NonParallelizable]`. |

## Rules

1. **Never make a test green with a sleep, a retry or a wider timeout.** A timing failure is fixed with `Assert.Eventually` or an assertion on a locator.
2. **Never lower `MaxParallelism` for the whole suite** to hide one class that shares state.
3. **Do not change the product's expected value to match what it returned** unless the requirement changed. A product bug is reported, not asserted around.

Done when the cause is named with its evidence, the fix is run, and anything you could not reproduce is reported as such.
