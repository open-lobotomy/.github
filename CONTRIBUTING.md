# Contributing to Open Lobotomy organization defaults

This repository controls shared GitHub behavior. A small edit can affect every repository that
inherits a file or calls a reusable workflow, so verify the callers before changing the contract.

## Repository structure

- `.github/workflows/ci.yml` validates this repository's workflow files with `actionlint`.
- `.github/workflows/dotnet-ci.yml` restores the consuming repository's local tools and runs
  `dotnet ci --check` on Linux.
- `.github/workflows/dotnet-ci-windows.yml` restores, builds, and tests the caller's requested
  solution on Windows.
- `profile/README.md` is the organization landing page.
- `renovate.json` is the shared Renovate preset.

There is no application code in this repository.

## Reusable workflow contracts

A caller references a workflow by repository path and branch:

```yaml
jobs:
  ci:
    uses: open-lobotomy/.github/.github/workflows/dotnet-ci.yml@main
```

Treat each `workflow_call` input, secret, default, output, permission, and job dependency as a public
interface. Search the organization for every caller before renaming or removing one. Additive inputs
need safe defaults so existing callers keep working.

The Linux workflow accepts optional private-reference and local-package inputs. The Windows workflow
also requires `solution-file`. Keep secret access scoped to the step that needs it, and never print a
token or a generated NuGet source containing credentials.

## GitHub inheritance

GitHub inherits supported community files from this repository only when a child repository does not
define its own copy. Inheritance is all or nothing per file. Before adding or changing an organization
default, inspect the repositories that inherit it and the repositories that intentionally override it.

## Validation

Run `actionlint` against every workflow change. The repository CI downloads the current `actionlint`
release and runs it from the repository root.

Also perform the validation that matches the change:

- Search all Open Lobotomy callers when a reusable workflow interface changes.
- Check both Linux and Windows shell syntax when a shared step moves between runners.
- Resolve every relative Markdown link when changing the profile or community documentation.
- Validate `renovate.json` as JSON and compare it with at least one consuming repository before
  changing inherited behavior.

## Pull requests

Branch from `main`, keep organization-wide changes narrow, and explain which repositories inherit or
call the changed surface. A reusable workflow change is not complete until at least one representative
consumer passes with the new contract.
