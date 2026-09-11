# Precept skills

Guidance for coding agents working in a test project built on
[Precept](https://autom3tion.github.io/precept-docs/), a .NET 10 test automation framework on
Microsoft.Testing.Platform. Describes **Precept 0.10.0**.

Nothing is authored here. These files are published from the Precept repository on every change
and every release, so the API they describe is the API that version ships.

## What is here

| Path | For |
| --- | --- |
| `skills/precept-tests` | A plain C# test class: attributes, hooks, retries, outcomes and awaited assertions. |
| `skills/precept-gherkin` | A `.feature` file and its Reqnroll bindings, and the `Reqnroll` settings section. |
| `skills/precept-settings` | `precept.json`, environment overlays, `PRECEPT_*` variables, startup and a suite's own settings. |
| `skills/precept-running` | Running, listing and filtering tests, environments, artifacts and CI. |
| `skills/precept-modules` | `Rest`, `Browser`, `Db`, `TestData`, `Rpc` and screenplay, one example each. |
| `skills/precept-page-objects` | Designing page objects and components on `IPageContext`, and when screenplay is the better shape. |
| `skills/precept-migrating` | Moving a suite from NUnit, xUnit, MSTest, SpecFlow, FluentAssertions, Selenium or RestSharp, with every old test accounted for. |
| `skills/precept-pipelines` | Running the suite on Azure Pipelines with the shipped pipeline and job template, settings as `PRECEPT_*` variables, TRX and attachments, exit codes. |
| `.github/agents/precept.agent.md` | A GitHub Copilot agent that writes and runs Precept tests and brings the Precept MCP server with it. |
| `.github/instructions/precept.instructions.md` | A one-page always-on summary for Copilot. |

The skills are in the [Agent Skills](https://agentskills.io) format — a `SKILL.md` with `name`
and `description` frontmatter — which GitHub Copilot, Claude Code, Cursor and the other agents that
read that format all understand.

## Installing

```bash
npx skills add autom3tion/precept-skills
```

That puts the skills where each agent in your repository looks for them. For GitHub Copilot, also
copy `.github/agents` and `.github/instructions` into your repository's `.github/` folder; the agent
declares the [Precept MCP server](https://autom3tion.github.io/precept-docs/Precept.Mcp.html) itself,
launched with `dnx Precept.Mcp`, so nothing else has to be configured for it.

Match the version to the Precept your project references. The MCP server reports the version it
answers out of, and the skills say theirs above; an agent reading the docs for one version and
writing code against another is the drift these exist to prevent.

Full documentation: [Working with a coding agent](https://autom3tion.github.io/precept-docs/coding-agents.html).
