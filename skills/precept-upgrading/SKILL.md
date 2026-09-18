---
name: precept-upgrading
description: "Move a suite to a newer Precept, then find and fix everything the new version changed — renamed APIs, settings keys that silently stopped working, and defaults that moved. Use when updating or upgrading Precept packages, when a new Precept release is out, or after a version bump."
---

# Upgrading Precept

Start from the release notes to learn which areas moved, then prove it against the project itself: the compiler names the APIs that moved, the published schema names the settings keys that moved, and the settings reference names the defaults that moved. The notes narrow the search; they never end it. Work through all three — a build that succeeds proves only the first.

## Steps

1. **Establish both versions.** `dotnet list package` for what the project has; `dotnet package search Precept.TestPlatform --source https://api.nuget.org/v3/index.json --exact-match` for what is out. Every `Precept.*` package in the solution moves to the same version together — mixing them is unsupported and fails in ways that read like bugs.
2. **Read the release notes between the two versions.** <https://autom3tion.github.io/precept-docs/release-notes.html>, newest first, one section per version. Note the modules, settings and behaviours the entries name: those are where the audit below starts, and a `Breaking changes` section names what is expected to need work. An entry names a change, not every line it touched, so it is a map and never a substitute for steps 5 to 7.
3. **Bump in one place.** `Directory.Packages.props` if the repository uses central package management, every `.csproj` otherwise. Do not bump one package to try it.
4. **Update the agent's own copies.** `npx skills add autom3tion/precept-skills` for these skills, and the MCP server — `dnx Precept.Mcp@<version>`, or `dotnet tool update --global Precept.Mcp`. The server bundles its own `Precept.Core` and answers out of *that* version; `precept_diagnose_project` raises `PMCP008` when it could disagree with the project. Skills describing one version while the project references another is the whole failure mode this step exists to prevent.
5. **Build, and read every error as a rename before treating it as a removal.** A missing type or member usually moved or was renamed; find the replacement in the docs page that owns the area before rewriting anything around it. Never pin a package back to make the build pass, and never suppress the error.
6. **Audit every settings file against the new schema.** This is the half a build cannot catch. Fetch `https://autom3tion.github.io/precept-docs/precept.schema.json`, then list every key in `precept.json`, each `precept.{environment}.json` overlay and any `testconfig.json` that the schema does not contain. A key Precept no longer knows is not an error and is not underlined — the schema deliberately allows a project's own sections — so it simply stops doing anything, quietly, and the suite goes on passing with the setting ignored.
7. **Check the defaults, not only the keys.** A setting that still binds can still mean something new. Compare the values the project *relies on* against `https://autom3tion.github.io/precept-docs/settings-reference.html`, which lists every key at its default for that version. State a default explicitly in `precept.json` when the suite depends on it.
8. **Run `precept_diagnose_project`**, then list and run the suite. Classify every failure as product, environment or upgrade, and fix the upgrade ones before reporting anything.

## Where a change hides

| Symptom | Where to look |
| --- | --- |
| Build error on a Precept type, member or namespace | Step 5. Renamed far more often than removed. |
| Builds, runs, but a setting has no effect | Step 6. The key was renamed and now binds to nothing. |
| Builds and passes, but behaves differently | Step 7. A default moved, or the setting's values changed shape — for example a boolean that became an enum. |
| The agent's advice contradicts the project | Step 4. The MCP server or the skills are a different version from the packages. |
| A `.feature` file produces no tests after the bump | Delete `**/*.feature.cs` and rebuild. Reqnroll caches code-behind against the feature file, so a new generator regenerates nothing on its own. `PMCP001` reports it. |

## Rules

1. **Report what you changed and what you could not.** An upgrade that silently drops a capability the suite used is worse than one that stops and says so.
2. **Do not widen the upgrade into a refactor.** Fix what the new version broke; leave everything else for its own change.
3. **While the major version is 0, a minor bump may rename things.** One minor release to the next is not guaranteed to be source-compatible, so never treat a minor bump as a no-op and skip the audit.
4. **One version for the whole solution**, including a shared step-definition library that references `Precept.Core` directly.

Done when every `Precept.*` package is on one version, the build is clean without a suppression or a pinned-back package, no settings file holds a key the new schema does not know, `precept_diagnose_project` is quiet, and the suite is green or every failure is classified.
