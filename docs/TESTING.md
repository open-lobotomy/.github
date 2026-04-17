<!--
Canonical: https://github.com/open-lobotomy/.github/blob/main/TESTING.md
Version: 1.0.0
Last updated: 2026-04-17
-->

# Testing Standards

This document defines testing conventions for all Open Lobotomy repositories. It is the canonical source — each consumer repo keeps an identical copy at its root. Drift between a consumer copy and this file is a CI failure (enforced by `dotnet ci check-testing-doc`).

When updating this file: bump `Version` in the header, update `Last updated`, and notify downstream repos to run `dotnet ci sync-testing-doc` on their next change.

## Framework stack

All Open Lobotomy test projects use the same stack. Pin these versions in each repo's `Directory.Packages.props`:

| Package | Version |
|---|---|
| xunit.v3 | 3.2.2 |
| xunit.v3.runner.console | 3.2.2 |
| xunit.runner.visualstudio | 3.1.5 |
| xunit.analyzers | 1.27.0 |
| Microsoft.NET.Test.Sdk | 18.3.0 |
| Moq | 4.20.72 |
| AwesomeAssertions | 9.4.0 |
| AutoFixture.AutoMoq | 4.18.1 |
| AutoFixture.Xunit3 | 4.19.0 |
| coverlet.collector | 8.0.0 |
| coverlet.msbuild | 8.0.0 |

When a version bump is needed, update `open-lobotomy-tooling`'s `Directory.Packages.props` first, verify its tests pass, then bump this table (minor version) and propagate.

## Test project naming

Folder, `.csproj`, assembly name, and root namespace all match: `{Project}.Tests` (plural).

Example: the test project for `LobotomyCorporation.Mods.Abstractions` lives at `test/LobotomyCorporation.Mods.Abstractions.Tests/LobotomyCorporation.Mods.Abstractions.Tests.csproj` with root namespace `LobotomyCorporation.Mods.Abstractions.Tests`.

## Test class and method naming

- **Class name:** `{UnitUnderTest}Tests`. One test class per production type.
- **File name:** mirrors the production file. `MyService.cs` → `MyServiceTests.cs`.
- **Method name:** `{Method}_{Scenario}_{Expected}` using PascalCase for each part and underscores as separators.

Examples:

- `Equals_IdenticalInstances_ReturnsTrue`
- `TryParse_WithEmptyInput_ReturnsFalse`
- `Constructor_WithNullDependency_Throws`
- `IsBindCandidate_ForDifferentReceiver_ReturnsFalse`

The three parts answer three questions: *what is being tested, under what conditions, and what outcome is expected.* A reader scanning test names should be able to infer the behavior under test without opening the file.

## Coverage thresholds

80% line, 80% branch, 80% method. Configure per-repo in `coverlet.json` at the repo root:

```json
{
  "lineThreshold": 80,
  "branchThreshold": 80,
  "methodThreshold": 80
}
```

Enforced by `dotnet ci --check` on every push and PR. Locally, `dotnet ci` runs the full check (format + test + coverage).

New modules that cannot yet meet 80% should be added to `ci.json`'s `skipModules` list with a comment explaining why, and removed from the list once coverage catches up. `skipModules` is not a permanent exemption — treat entries as debt.

## Boundary separation (Category 1 / Category 2)

Every production class falls into one of two categories. Decide which when you add the class.

**Category 1 — Fully testable.** Business logic with injectable dependencies: services, view models, parsers, models, extension methods. Must reach the coverage threshold. Test these directly.

**Category 2 — Boundary wrappers.** Thin shims that touch the game runtime, platform APIs, or UI framework and cannot be meaningfully unit-tested. Mark with `[ExcludeFromCodeCoverage]` and keep the logic minimal — delegate to a Category 1 class, do not embed logic.

Examples by project type:

- **Unity mod DLLs (Harmony patches):** the `[HarmonyPostfix]` method with `__instance` / `__result` parameters is Category 2. The extension method it calls (containing the actual logic) is Category 1. This is the "two-method patch pattern" — see `lobotomy-corporation-mods/.claude/CLAUDE.md` for the specific shape Harmony patches take.
- **Avalonia installer:** `Program.cs`, `App.axaml.cs`, every `.axaml.cs` code-behind file.
- **BepInEx preloader patchers:** `Finish()`, `InitializeConfig()`, `OnAssemblyLoadFile()`, Harmony patch callbacks, safe logging helpers.
- **MonoBehaviour lifecycle entries:** `Awake()`, `Update()`, `OnGUI()` dispatchers that delegate immediately to a Category 1 controller.
- **Source-generated code:** generated partial classes and JSON context types (e.g. `ManifestJsonContext`).

If a class needs `[ExcludeFromCodeCoverage]`, keep it as small as possible. The moment Category 2 code grows a branching condition or a transformation, extract that work to a Category 1 class.

## InternalsVisibleTo policy

> *A library's contract is its public API. An application's contract is its entry point. An analyzer's contract is its diagnostics. Match the access modifier policy to whichever of those three your project actually ships.*

### Three-way classification

| Project type | Contract shape | IVT policy |
|---|---|---|
| **Library** | Public types consumed by external code via `using` and `new` | **Ban IVT.** Test through the public surface. If testing an internal directly matters, refactor it to public or restructure. |
| **Application** | Entry point: `OutputType=Exe`, a Harmony-patched mod DLL, a BepInEx preloader, a MonoBehaviour entry, a CLI tool | **Allow IVT.** Access modifiers are organizational; the entry point is what ships. Use `<InternalsVisibleTo Include="{Project}.Tests" />` and `<InternalsVisibleTo Include="DynamicProxyGenAssembly2" />` (for Moq). |
| **Analyzer or source generator** | Behavioral — diagnostics emitted, attributes recognized, code generated | **Allow IVT.** Roslyn loads the assembly by `[DiagnosticAnalyzer]` / `[Generator]` attribute discovery; consumers never reference the assembly's types directly. Access modifiers don't define the contract. |

### Caveat — sibling analyzer utilities

If an analyzer assembly exposes shared utilities (e.g., a `SyntaxHelpers` class) that *sibling* analyzer assemblies in the same org consume via `using`, those utilities *are* a library contract for the siblings. Extract them into a conventional library package rather than letting the hybrid leak across assembly boundaries. The analyzer project remains application-shaped for its own internals; the shared utilities live in a library that follows library rules.

### Classification examples

- `LobotomyCorporation.Mods.Abstractions`, `LobotomyCorporation.Mods.Testing`, `LobotomyCorporation.Mods.Common` → library → no IVT. The Extensions-and-Internals pattern (public extension method + `internal` implementation + public interface for mocking) enables testability without breaking the ban.
- `OpenLobotomy.Tooling` (CLI tool) → application → IVT allowed.
- Every Harmony-patched mod in `lobotomy-corporation-mods` → application → IVT allowed.
- The `ConfigurationManager` runtime DLL → application → IVT allowed.
- `Harmony2ForLmm` (Avalonia installer) → application → IVT allowed.
- `RetargetHarmony` (BepInEx preloader) → application → IVT allowed.
- `LobotomyCorporation.Mods.Analyzers` → analyzer → IVT allowed.
- `LobotomyCorporation.Mods.ConfigurationManager.Integration` → source generator → IVT allowed.
- `OpenLobotomy.Standards`, `LobotomyCorporation.Mods.Analyzers` (when acting as a packaging-only MSBuild carrier) → no types to test directly; IVT question is moot.

## I/O abstraction

New production code must route file, process, and environment I/O through the abstractions in `OpenLobotomy.Abstractions`:

- `IFileSystem` — `fileSystem.File.Exists()`, `fileSystem.Directory.Create()`, `fileSystem.Path.Combine()`.
- `IProcessRunner` — wraps `System.Diagnostics.Process`.
- `IEnvironmentProvider` — environment variables and special folders.

Do not use `System.IO.File`, `System.IO.Directory`, or `System.IO.Path` static methods directly in production code. When integrating with third-party libraries that accept file paths, prefer stream overloads with `IFileSystem.File.OpenRead` / `IFileSystem.File.Create`.

Tests use the in-memory mocks from `LobotomyCorporation.Mods.Testing`:

- `MockFileSystem` — implements `IFileSystem` against an in-memory dictionary.
- `MockProcessRunner` — scripted process-invocation responses.
- `MockEnvironmentProvider` — controlled environment state.

Integration tests that exercise real I/O use real temp directories (`fileSystem.Path.GetTempPath()`) and clean up with `IDisposable` or collection fixtures.

Retrofit existing code opportunistically — don't block new work on migrating callsites that don't need to change.

## AutoFixture

`AutoFixture.AutoMoq` and `AutoFixture.Xunit3` are available in every repo's `Directory.Packages.props`. Use where they meaningfully reduce boilerplate:

- **Good fit:** DTO-heavy constructor tests, property matrix assertions, tests where "some valid value" is more truthful than a specific one.
- **Skip it:** when a direct `Mock<T>` is clearer, or when the test's readability depends on seeing explicit input values.

Custom auto-data attributes (`debug-panel`'s `LobotomyAutoDataAttribute`, `LobotomyInlineAutoDataAttribute`, `GlobalTimeoutAttribute`) are an acceptable per-repo pattern but not required. If a new repo introduces them, follow the same file layout: `test/{Project}.Tests/Attributes/{Name}Attribute.cs`.

## Repo-specific extensions

Each repo's `.claude/CLAUDE.md` contains a short "Repo-specific testing notes" section covering anything genuinely unique to that repo — analyzer-test infrastructure, collection fixtures for static state, snapshot-testing conventions, net35 target-framework constraints, etc.

Those sections *extend* TESTING.md, they do not override it. If a repo-specific note conflicts with this document, update the canonical here rather than diverging locally.

---

*Version 1.0.0. Changes are proposed via PR to `open-lobotomy/.github`; approved changes bump the version in the header and trigger downstream sync via `dotnet ci sync-testing-doc`.*
