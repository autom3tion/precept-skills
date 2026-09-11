---
name: precept-pipelines
description: "Run a Precept suite on Azure Pipelines: adopt the shipped pipeline and job template, set its parameters, pass settings as PRECEPT_* variables, publish the TRX with attachments, and read exit codes. Use when adding or changing a pipeline, a scheduled run, a variable group, or a CI failure in a Precept project."
---

# A Precept suite on Azure Pipelines

Precept ships the pipeline. Copy both files into the repository that holds the tests rather than writing a `dotnet test` job by hand:

| File | Holds |
| --- | --- |
| https://autom3tion.github.io/precept-docs/azure-pipelines.yml | The entry pipeline: triggers, schedule and the `parameters` block that becomes the **Run pipeline** dialog. |
| https://autom3tion.github.io/precept-docs/templates/precept-test-job.yml | One run as a job: restore, build, browser install, run, publish the TRX and the artifact directory. |

A second entry file over the same template is how a nightly, a release smoke check and an on-demand regression differ. The full page is https://autom3tion.github.io/precept-docs/azure-devops.html.

## Steps

1. **Check the prerequisites in the test repository**: a `global.json` at the repository root with `"test": { "runner": "Microsoft.Testing.Platform" }`, a `precept.{environment}.json` for every environment the dialog will offer, and an `IPreceptStartup` that registers each reporter the pipeline may turn on.
2. **Copy the two files** and set the template's fixed parameters — `testProject`, `testProjectDirectory`, `dotnetVersion`, `vmImage`, `timeoutInMinutes` — in the entry file. Set `installBrowsers` to `false` for a suite that drives no browser.
3. **Put secrets in a variable group**, never in YAML: the ReportPortal API key, the Teams webhook, the Azure DevOps token. The template maps them to `PRECEPT_REPORTING__*` variables.
4. **Add a dialog parameter only in the entry file**, never in the template; parameters are compile-time and a scheduled run cannot set them, so a schedule with different settings is a second entry file.
5. **Verify with a real run**: the summary names the environment, a TRX is published with attachments, and a deliberately failing test turns the job red.

## Rules

- **Two channels, by kind.** What the *run* is goes on the command line: `--precept-environment`, `--precept-filter`. Anything that is a *setting* goes as a `PRECEPT_*` variable, `:` written as `__`: `Reporting:Teams:NotifyOn` is `PRECEPT_REPORTING__TEAMS__NOTIFYON`. Every setting in `precept.json` can be set this way, and it outranks both files.
- **`PRECEPT_` is the framework's namespace.** A pipeline variable of the job's own must not carry it; the template uses `SUITE_*` for those.
- **A filter must not start with `@`**, or the platform reads it as a response file.
- **The run timeout sits inside the job timeout.** The template sets `RunTimeoutMinutes` five minutes short of `timeoutInMinutes`, so a hung suite is ended by Precept with a TRX and a report rather than by the agent with nothing. Keep that gap when changing either.
- **Results need `--report-trx` and `PublishTestResults@2` with `publishRunAttachments: true`**; that is what carries screenshots and traces into the run. Keep `ArtifactDirectory` short on Windows agents, or attachments past 260 characters go missing from the TRX while present on disk.
- **Reporters are CI-only by default** and wake up because the agent sets `TF_BUILD`. A container job without it needs `PRECEPT_ISCONTINUOUSINTEGRATION=true`.
- **Exit codes**: `0` passed, `2` at least one test failed, `8` no test ran. Code 8 is correct behaviour for a filter that matched nothing; add `--minimum-expected-tests <n>` for a suite whose size is known, so a filter that quietly matched a dozen tests fails too.
- **`MaxParallelism` above the number of feature files changes nothing** under the default `ParallelScope: Class`; the dialog's parallel scope is the fix, not a bigger agent.
- **`.runsettings` does nothing** on this platform; there is no VSTest bridge.

## When it goes wrong

| Symptom | Cause |
| --- | --- |
| "Testing with VSTest target is no longer supported" | No `global.json` with the MTP runner at the repository root. |
| The run fails opening a response file named after a tag | The filter starts with `@`. |
| Exit code 8 | The filter matched nothing, or the overlay's `Filter:Exclude` removed everything. |
| "reporting is CI-only and this run is not on a build agent" on an agent | No CI variable; set `PRECEPT_ISCONTINUOUSINTEGRATION=true`. |
| ReportPortal rejects the key | The variable group or variable name does not match, so the literal `$(reportPortalApiKey)` was sent. |
| Azure DevOps creates the run but refuses the results | The build service identity lacks test management rights on the project. |
| Every scenario discovered twice | A `.feature` file under `bin` was globbed. |

Done when the pipeline uses the shipped template, secrets come from a variable group, and one run has produced a TRX with attachments under the intended environment.
