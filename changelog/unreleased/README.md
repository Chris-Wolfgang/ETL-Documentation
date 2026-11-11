# Changelog fragments

`CHANGELOG.md` is not edited by hand. A pull request that makes a reader-visible change to the site adds
**one fragment file** to this directory; when a batch of changes is published the fragments are assembled
into a new section of `CHANGELOG.md` and deleted. This avoids merge conflicts on `CHANGELOG.md` and keeps
each entry next to the change that earned it.

This is the fleet-wide convention from
[repo-template](https://github.com/Chris-Wolfgang/repo-template/blob/main/changelog/unreleased/README.md),
applied to a documentation repository.

## Writing a fragment

Create `changelog/unreleased/<short-change-name>.md` (kebab-case, named for the change, not the PR number):

```
type: docs

Add a cookbook recipe for loading a gzip-compressed JSON Lines file into SQL Server.
```

- **Line 1** is `type: <kind>` where `<kind>` is one of:

  | Kind | Use for | Assembles under |
  |------|---------|-----------------|
  | `breaking` | Removed or moved pages that break existing links | Breaking changes |
  | `feature` | A new page, guide or recipe | Added |
  | `fix` | A correction (wrong signature, stale version, broken link) | Fixed |
  | `docs` | Other reader-visible content changes | Documentation |
  | `internal` | Build, CI, tooling or dependency changes with no visible effect | Internal |

- **The rest** is a one-sentence, reader-facing description.

## When a fragment is needed

The CI check (`scripts/changelog.ps1 check`, run by `.github/workflows/changelog.yaml`) is the
repo-template script unchanged. It requires a fragment only when a PR changes files under `src/`, and this
repository has no `src/`, so here a fragment is **encouraged, not enforced**. The check still validates
every fragment in this directory on every PR: a recognised `type:` on line 1 and a non-empty description.

```powershell
pwsh ./scripts/changelog.ps1 check                     # against origin/main
```

## Assembling

```powershell
pwsh ./scripts/changelog.ps1 assemble -Version 2026.11.0   # or let it derive the next version
```

`assemble` inserts a `## [<version>] - <today>` section under `## [Unreleased]` in `CHANGELOG.md`, grouped
by kind, then deletes the fragments.
