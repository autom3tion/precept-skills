---
name: precept-running
description: "Run, list and filter Precept tests from the command line or CI, pick an environment, read output and artifacts. Use when running tests, writing a filter expression, or wiring a pipeline."
---

# Running Precept tests

```bash
dotnet test --project Checkout.Tests/Checkout.Tests.csproj                   # SDK runner; needs MTP in the root global.json
dotnet run --project Checkout.Tests -- --list-tests                          # the platform's own CLI
dotnet run --project Checkout.Tests -- --precept-environment staging
dotnet run --project Checkout.Tests -- --precept-filter "smoke and not wip"
dotnet run --project Checkout.Tests -- --output Detailed --show-stdout All   # step trace and logs of passing tests
dotnet run --project Checkout.Tests -- --report-trx                          # TRX with categories, logs and attachments
```

`global.json` at the repository root must contain `"test": { "runner": "Microsoft.Testing.Platform" }`; the SDK reads it from the working directory upward, so a copy beside the project does nothing. `dotnet test` takes `--project <csproj>`, not a positional path, and never lists passing tests.

## Filter expressions

| Form | Matches |
| --- | --- |
| `smoke`, `@smoke`, `tag:smoke` | a category |
| `name:*checkout*` | the display name, glob |
| `class:*.LegacyTests` | the declaring class |

Combine with `and`, `or`, `not`, parentheses; matching is case-insensitive; quote anything with spaces. **The value must not start with `@`** — the platform reads a leading `@` as a response file. Write `"smoke and not @slow"` or wrap it in parentheses.

## Rules

1. Parallelism is per class by default, and a Gherkin feature is one class; if a high `MaxParallelism` changes nothing, that is why. `ParallelScope: Test` schedules scenarios individually.
2. Exit code is non-zero on any failed test. A run that hits `RunTimeoutMinutes` still reports a verdict for every selected test, so prefer it over an agent-level kill.
3. Artifacts go to a directory per test under `ArtifactDirectory`; reruns write to `attempt-n` beneath it. With `--report-trx` they are listed as `<ResultFile>` entries, which Azure DevOps' `PublishTestResults` uploads.
4. No limit is enforced while a debugger is attached.
5. On CI set `PRECEPT_ENVIRONMENT` or pass `--precept-environment`, and let reporters stay `CiOnly` — `IsContinuousIntegration` is detected from the agent's variables.

Done when `--list-tests` shows the expected tests and the run banner names the environment and the effective parallel width.
