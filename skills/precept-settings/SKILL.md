---
name: precept-settings
description: "Configure a Precept suite: precept.json, environment overlays, PRECEPT_* variables, module sections, IPreceptStartup and a project's own settings class. Use when touching precept*.json, Startup.cs or a setting's value."
---

# Configuring a Precept suite

`precept.json` beside the project is copied to the output automatically. It is overlaid in order by `precept.{environment}.json`, then `PRECEPT_*` environment variables, each winning over the last. Comments and trailing commas are allowed. Put `"$schema": "https://autom3tion.github.io/precept-docs/precept.schema.json"` at the top of every one of these files for completion.

```json
{
  "$schema": "https://autom3tion.github.io/precept-docs/precept.schema.json",
  "Environment": "local",
  "MaxParallelism": 4,
  "ParallelScope": "Class",
  "RunTimeoutMinutes": 720,
  "DefaultRetries": 0,
  "Web": { "BaseUrl": "https://localhost:5001", "Browser": "chromium", "Headless": true },
  "Api": { "BaseUrl": "https://localhost:5001/api" },
  "ConnectionStrings": { "Default": "Host=localhost;Database=app" },
  "Filter": { "Include": "", "Exclude": "wip" }
}
```

## Rules

1. **A section is read only by the module that registers it.** `Web` needs `services.AddPreceptWeb()`, `Api` needs `AddPreceptApi()`, `TestData` needs `AddPreceptTestData()`, and so on, in `IPreceptStartup.ConfigureServices`. A `Web` block in a project that never calls `AddPreceptWeb()` does nothing.
2. **Override in code through the registration callback**, not by assigning on `PreceptSettings`: `services.AddPreceptWeb(web => web.Headless = false)`.
3. **Environment variables** map `:` to `__` with a `PRECEPT_` prefix: `Web:Headless` → `PRECEPT_WEB__HEADLESS`, `Filter:Exclude` → `PRECEPT_FILTER__EXCLUDE`. This is where secrets go, never the file.
4. **The environment** is resolved once, first source wins: `--precept-environment`, `PRECEPT_ENVIRONMENT`, `DOTNET_ENVIRONMENT`, the `PreceptEnvironment` MSBuild property (make each environment a build configuration for IDE switching), `"Environment"` in `precept.json`, then `local`. An `"Environment"` key inside an overlay is ignored. `.runsettings` is VSTest-only and does nothing; the platform's equivalent is `testconfig.json`.
5. **A suite's own settings** go in the same file under their own section, bound to a class with `services.AddPreceptSettings<AuthSettings>()` and read from `PreceptTestContext.Current.Services`. Unknown sections are valid everywhere; the schema never flags them.
6. **`Filter` removes tests from a run entirely** — not published, not reachable by filter or explorer — where `--precept-filter` only selects among what is left. Both are single expressions so an overlay can replace or clear them.
7. **Startup hooks.** `ConfigureServices` runs on discovery too, so it holds registrations only. `BeforeRunAsync`/`AfterRunAsync` run once around the first and last test and are skipped by `--list-tests`; a throwing `BeforeRunAsync` fails every selected test with its exception. Singletons that are disposable are disposed after `AfterRunAsync`.
8. **Defaults worth knowing.** No per-test time limit (`DefaultTimeoutMilliseconds: 0`); the run has one (`RunTimeoutMinutes: 720`). Retries default to `0` and count reruns. Reporters (`Reporting:*`) default to CI-only and are dropped from a local run.

Done when the run banner names the intended environment and no line says an overlay was expected but not found. For an unexplained no-op, run `precept_diagnose_project` from the `Precept.Mcp` server if it is configured.
