# Instructions

This file provides guidance when working with code in this repository.

## Repository Purpose

This is the **organization-level `.github` repository** for [Open Lobotomy](https://github.com/open-lobotomy). It provides shared defaults inherited by all org repositories — community health files, reusable CI workflows, the org profile, and configuration templates. There is no application code here.

## Structure

- `.github/workflows/` — Reusable CI workflows (`dotnet-ci.yml` for Linux, `dotnet-ci-windows.yml` for Windows) called by individual repo pipelines
- `.github/ISSUE_TEMPLATE/` — Bug report and feature request templates
- `.github/PULL_REQUEST_TEMPLATE.md` — PR template
- `profile/README.md` — Org landing page (renders at github.com/open-lobotomy)
- `templates/dependabot.yml` — Copy-paste reference for new repos
- Root community files: `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`, `LICENSE`

## Reusable Workflows

Individual repos call the workflows like this:

```yaml
jobs:
  ci:
    uses: open-lobotomy/.github/.github/workflows/dotnet-ci.yml@main
    with:
      solution-file: MyProject.slnx
```

The Linux workflow runs first; Windows runs after it succeeds (`needs: [ci]`). Key inputs: `solution-file` (required), `dotnet-version` (default `10.x`), `collect-coverage`, `coverage-files`, `upload-codecov`/`upload-codacy`, `private-references-repo`, `local-pack-projects`. Secrets: `CODECOV_TOKEN`, `CODACY_PROJECT_TOKEN`, `PRIVATE_REFERENCES_TOKEN`.

## Org-Wide Code Quality Standards

These are enforced in all org repos via CI:

- **Warnings as errors** — all compiler and analyzer warnings are build errors
- **All .NET analyzers enabled** — Microsoft, Roslynator, Unity, xUnit at error severity
- **SPDX file header** — every `.cs` file must start with `// SPDX-License-Identifier: MIT`
- **CSharpier formatting** — auto-fix with `dotnet csharpier format .`

## Development Commands (for Org Repos)

```bash
./scripts/setup-dev.sh          # Install local CI tool + pre-commit hook
dotnet ci                       # Auto-fix formatting, run tests + coverage
dotnet ci --check               # Check-only mode (no auto-fix)
```

## GitHub Inheritance Model

GitHub uses **all-or-nothing inheritance per file** from this repo. If an org repo creates its own `CONTRIBUTING.md`, the entire org default is ignored — there is no merging.