# Open Lobotomy organization defaults

Shared GitHub configuration for the
[Open Lobotomy organization](https://github.com/open-lobotomy). GitHub uses this repository for
the organization profile and for reusable workflows that individual projects call from their own
CI pipelines.

## What lives here

- `.github/workflows/dotnet-ci.yml` runs the standard Linux `dotnet ci --check` workflow.
- `.github/workflows/dotnet-ci-windows.yml` restores, builds, and tests a named solution on Windows.
- `profile/README.md` renders on the organization's public GitHub profile.
- `renovate.json` provides the organization Renovate preset.

Repository-local community files take precedence over organization defaults. GitHub replaces an
inherited file as a whole, so a project-specific `CONTRIBUTING.md` does not merge with a file from
this repository.

Read [CONTRIBUTING.md](CONTRIBUTING.md) before changing a reusable workflow or an organization-wide
default.
