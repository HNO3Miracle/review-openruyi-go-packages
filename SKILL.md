---
name: review-openruyi-go-packages
description: Review or repair openRuyi Go RPM packages one package at a time. Use when examining Go SPEC files, package additions or updates, Prometheus dependency branches, pull requests, patch series, module boundaries, Provides/Requires, test exclusions, pre-commit failures, GitHub CI, or OBS build results. Enforce current openRuyi packaging policy, repository-wide provider and consumer consistency, one-package-per-commit history, and isolated OBS validation.
---

# Review openRuyi Go Packages

Review each package as a packaging decision, not as a formatting-only file.

## Establish the Review Scope

1. Read the repository instructions, current openRuyi packaging documentation, PR conversation, and CI logs before changing files.
2. Resolve the real comparison base. For a GitHub PR, distinguish the PR head branch from its base branch; normally compare the head against `upstream/main` with a three-dot diff.
3. List changed `SPECS/<package>/` directories, then create a review ledger with one row per package.
4. Use bulk commands only to inventory files, providers, consumers, commits, or build states. Never use a bulk rewrite or regex-only judgment as a substitute for reading each package.
5. Preserve unrelated worktree changes. Use a clean worktree when branch history must be rewritten.

## Review One Package

Finish these steps for one package before moving to the next:

1. Read the complete spec with line numbers and inspect every file in its package directory.
2. Read every patch in full, including its mail header, rationale, affected code, and test changes.
3. Inspect the exact upstream source selected by `Source`, including all `go.mod`, `go.work`, nested modules, licenses, generated code, test fixtures, and binaries.
4. Search `upstream/main`, the current PR set, and relevant pending branches for duplicate packages, equivalent unversioned package names, existing `go(...)` providers, and all consumers.
5. Check the package against [references/go-package-checklist.md](references/go-package-checklist.md). Consult the live documentation links in that file whenever repository policy may have changed.
6. Record a package result as `pass`, `fixed`, or `blocked`, with concrete evidence. Do not carry an unresolved assumption into the next package.

## Make Repairs

- Keep the package boundary at the VCS repository when the complete repository can be packaged coherently. Install every nested module at its real import path.
- Prefer declarative `BuildSystem` behavior. Add `%prep`, `%build`, `%install`, or `%check` hooks only for work the default macros cannot express.
- Add missing build and test dependencies before excluding tests. Do not delete indirect dependencies, test dependencies, nested modules, or fixtures merely to make a build pass.
- Update consumers whenever a package name, import-path capability, module boundary, or `Provides` changes.
- Prefer an upstream release or accepted upstream fix over a downstream patch when this does not regress version ordering or API compatibility.
- Apply manual edits with `apply_patch`. Keep changes scoped to the package under review and its proven consumers.

## Preserve History

- Keep one logical package per commit and add `Signed-off-by` to every commit.
- Do not leave `fixup!` or `squash!` commits in the submitted branch.
- When repairing an existing multi-package PR that requires one-package-per-commit history, fold the repair into that package's original commit using a controlled rebase or rebuilt branch, then use `--force-with-lease` only after checking the remote head.
- Use commit subjects such as `SPECS: <package>: Add package.` or `SPECS: <package>: Fix <issue>.` Do not use a generic `Add ...` metadata subject.

## Validate in Order

1. Run focused syntax and source checks for the package.
2. Run repository pre-commit hooks on the changed files.
3. Validate every `#!RemoteAsset` checksum and ensure every remote `Source` has one.
4. Push the package to an isolated OBS subproject whose repositories depend only on openRuyi.
5. Manually rerun OBS source services after changing the Git branch or sources. Distinguish `broken` service failures from `failed` builds and `unresolvable` dependency failures.
6. Require amd64 success before pushing a repair to the GitHub PR unless the user explicitly sets a different gate. Do not use local `pbuild` as a substitute for this OBS check.
7. Trigger or inspect GitHub PR CI only after its prerequisite packages and branch changes are available.

Never report success while a relevant OBS job is `building`, `scheduled`, `broken`, `failed`, or `unresolvable`.

## Report Results

Lead with actionable findings ordered by severity. Include package and line references, the violated rule, behavioral impact, and exact repair. End with a flat package matrix and validation state using [references/report-template.md](references/report-template.md).

If asked only to review, do not edit. If asked to fix, complete the repair, OBS validation, history cleanup, and push unless blocked by external state.
