# openRuyi Go Package Checklist

Use the current published documentation as the authority. Repository precedent is evidence, not policy, because existing specs may be stale or incorrect.

## Live References

- General specification: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines
- Go policy: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/languages/Golang
- Go build systems: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/BuildSystems/golang
- Versioning: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/Versioning
- Patch policy: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/Patch
- Source policy: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/SourceURL
- Naming: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/Naming
- Split packages: https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/SplitPackage

## 1. Package Identity and Boundary

- Decide whether the result is a library, program, or program plus library. A program-only package uses the upstream program name without a `go-` prefix. A source library package uses the normalized import or VCS repository path.
- Normalize library names from the canonical repository root. Add `-vN` only when the import path carries that semantic-import major, except for documented parallel-version cases.
- Search the repository before adding a package. Check both exact names and older names without a version suffix. An absent `Provides` does not prove the source is absent.
- Prefer one source package per VCS repository. Include all coherent nested modules from the selected snapshot, even if they have separate `go.mod` files or tags.
- Do not package a nested module separately when a repository-level package already installs the same files. Prevent duplicate ownership and install conflicts.
- Install each module under `%{go_sys_gopath}` at its actual import path. A repository-level archive does not justify paths such as `/v2/v2` or repeated subdirectory components.
- Do not restrict a large package to the modules currently needed by one consumer. Package the complete repository unless a concrete conflict or unsupported component requires a documented exception.

Evidence to inspect:

```bash
find <source-root> -name go.mod -o -name go.work
rg -n '^module |^go |^toolchain |^require |^replace ' <source-root>/**/go.mod
rg -n -F 'go(<import-path>)' SPECS
rg -n -F '<physical-install-path>' SPECS
```

## 2. Version and Source

- Prefer the latest stable upstream release compatible with the dependency graph. Never silently downgrade an existing package.
- Follow the import path's major-version semantics, not an arbitrary Git tag name.
- For a project that has never released, use `0+gitYYYYMMDD.<short-hash>`. The date is the packaging date, not the commit date.
- For a snapshot after a release, start with the latest release and append `+gitYYYYMMDD.<short-hash>`.
- Use `~alpha`, `~beta`, or `~rc` for prereleases. Avoid alpha versions when a suitable stable release exists.
- Keep the full commit in `commit_id` when pinning a snapshot and use a short hash only in `Version`.
- Ensure `Source0` resolves to the exact source used by `Version` and `%prep` expects the archive's real top-level directory.
- Reuse `%{version}`, `%{commit_id}`, and other defined values in `Source` and setup options. Do not repeat a literal version or commit where the corresponding macro is valid.
- Put a valid `#!RemoteAsset:  sha256:<64 hex>` immediately above every remote `Source` line. Use predictable archive names after `#/` when required.
- Inspect archive contents rather than guessing GitHub archive layout, submodules, or nested module paths.

## 3. Spec Layout and Formatting

Use this order unless a documented package-specific reason requires more fields:

```text
SPDX header
%define/%global values and documented test macros
Name, Version, Release, Summary, License, URL, optional VCS
RemoteAsset + Source entries
BuildArch
BuildSystem
Patch fields
BuildOption fields
BuildRequires
Provides
Requires/Conflicts/Obsoletes
%description
custom build sections when needed
%files
%changelog / %autochangelog
```

- Place `BuildArch` after the final `Source` and immediately before `BuildSystem`.
- Place patches after `BuildSystem` and before the first `BuildOption`; without a `BuildOption`, place them before `BuildRequires`.
- Order `BuildOption` fields by build phase. Keep exactly two spaces after `BuildOption(...):` and use the repository's aligned field columns elsewhere.
- Use one dependency per line. Keep related blocks separated by a single blank line.
- Use `%autorelease` and `%autochangelog` exactly.
- Keep `Summary` concise, in American English, and without a trailing period. Do not repeat the summary or write a feature list into it.
- Keep `%description` concise and package-specific. Avoid marketing copy, implementation trivia, and descriptions of unrelated dependencies.
- Put `%doc` before `%license` in `%files`, then installed paths. Use globs such as `README*` or `LICENSE*` only when the archive contents justify them.
- Do not add `SPDX-FileContributor` or sign ordinary comments for small mechanical changes.
- Do not add `_service` or `_constraints` to the GitHub package directory unless repository policy explicitly requires it; keep OBS-only metadata in OBS.

## 4. Build System and Sections

- A Go library normally uses `BuildArch: noarch` and `BuildSystem: golangmodules`.
- Every Go library requires both `BuildRequires: go` and `BuildRequires: go-rpm-macros`.
- Let `golangmodules` provide default preparation, installation, and testing. Do not hand-write those phases merely to imitate `go2spec` output.
- Use `BuildOption(prep)` for a different archive directory only when needed. Prefer a small `%prep -a` or `%install -a` hook for true additions over replacing the whole generated phase.
- `%section -p` runs before the generated phase; `%section -a` runs after it. Verify the chosen direction against the expanded build log.
- For a module in a repository subdirectory, use explicit `pushd <literal-subdirectory>`/`popd` around only the phases that require it. Do not invent a one-use `go_source_subdir` macro.
- For a program plus library, build binaries explicitly but retain the complete source installation. For a pure library, do not compile unrelated binaries.
- Add a `BuildRequires` for tools invoked explicitly by custom sections or upstream tests, such as `git`, `unzip`, or a code generator, unless the base build contract demonstrably guarantees them. Group test-only tools below a `# For tests` comment when that improves clarity.
- If generated build scripts or examples install shell files in a noarch package, filter their automatic interpreter dependency with the appropriate RPM dependency-generator mechanism; do not add a runtime shell dependency without checking whether the file is executable and shipped.

## 5. Dependencies and Capabilities

- Keep build dependencies complete for every packaged module and its runnable test suite. Do not delete indirect or test dependencies just because one consumer does not use them.
- Derive runtime `Requires` from imports in installed non-test source and from the package's role as buildable source. Do not blindly copy every test-only dependency into runtime `Requires`.
- Express Go dependencies as `go(<actual import path>)`, not RPM package names.
- Search actual repository providers before adding a new package. Existing packages may have an unversioned RPM name while providing a versioned import path.
- Verify every required capability against the producer's installed files and explicit `Provides`. Automatic Go Provides generation is not assumed to be deployed.
- For repository-level large packages, provide every real module root and every installed import capability required by current consumers. Never invent a child capability for a directory that is excluded or not installed.
- Multiple `Provides` are justified by real nested modules, compatibility paths, or migration aliases. They are not justified merely to silence an unsolvable log.
- If an existing producer installs a required path but lacks the needed capability, fix that producer and then update consumers consistently. Do not create a duplicate source package as a shortcut.
- When replacing a split package with a large package, update every consumer and remove all split-package dependencies and overlapping files as one coordinated change.
- Audit dependency cycles at package and PR level. Move packages between PRs so dependencies form a DAG whose leaf PRs can build against openRuyi alone.
- Keep dependency-coherent PRs near 30 package commits where practical. Do not break a valid DAG merely to satisfy an arbitrary count.

Useful searches:

```bash
rg -n '^Provides:[[:space:]]+go\(' SPECS
rg -n '^BuildRequires:|^Requires:' SPECS/<package>/<package>.spec
rg -n -F 'go(<capability>)' SPECS
git branch -r --contains <commit>
gh pr list --state open --search '<package-or-import-path>'
```

## 6. Tests

- Test all packaged modules by default. Do not use an include list merely to test only the modules needed by Prometheus or another current consumer.
- Add missing test dependencies and external fixtures before considering exclusions.
- Preserve upstream test fixtures. If a Git submodule is absent from a release archive, add it as a checksummed `SourceN`, place it at the exact expected path during `%prep`, and avoid shipping it unless consumers need it.
- Use `%define go_test_exclude` for exact package paths and `%define go_test_exclude_glob` for patterns. Keep a multi-entry exclusion semantically consistent; do not mix exact and glob intent arbitrarily.
- Put test macros near `_name` and `go_import_path`, with a concise comment explaining the concrete failure.
- To skip an entire Go package, exclude its import path through the macro. Do not use `BuildOption(check): -skip` or shell-level `go test $(go list ...)` filtering.
- Use `BuildOption(check)` for genuine Go test arguments such as `-short`, build tags, or a justified `-vet=off`, not as a global test bypass.
- Use `-vet=off` only for a vet diagnostic that cannot reasonably be fixed or excluded at package granularity. It must not hide compilation or test failures.
- Do not use `go_test_ignore_failure` unless policy explicitly approves the package-specific exception.
- Treat map-order, locale, time-zone, network, terminal, architecture, and Go-version failures according to their actual cause. Patch deterministic expectations or environment setup; do not weaken unrelated assertions.

## 7. Patches

- First check the latest stable release, upstream default branch, open issues, and open pull requests. Drop a downstream patch if a suitable release already contains the fix.
- Generate local source patches with `git format-patch`. Require `From`, author, date, subject, rationale, diffstat, `diff --git`, and `a/`/`b/` paths that apply with `-p1`.
- Use four-digit filename prefixes:
  - `0001-0999`: fix from the same upstream version.
  - `1000-1999`: CVE fix or backport from another upstream version.
  - `2000-2999`: openRuyi-only or not-yet-accepted upstream change.
- Keep a pending upstream submission in the `2000` range. Once upstream merges it, do not rename it solely because its status changed.
- Put a concise purpose comment or direct upstream PR URL immediately above each `PatchN` field. Follow current repository/reviewer convention for `PatchN` indices, keep them monotonic, and verify every field maps to the intended filename; do not confuse `Patch0`/`Patch1` tag indices with the filename's policy range.
- Use `%patchlist` above `%description` when there are more than three patches.
- Verify the patch applies to the exact `Source0`. Check that it changes only necessary code and retains meaningful test coverage.
- For a generally useful fix, follow upstream's contribution guide and PR template. Keep downstream packaging rationale out of the upstream commit unless relevant to upstream users.

## 8. Files, Licensing, and Conflicts

- Verify `License` against files actually installed or transformed into the binary RPM, using SPDX identifiers and expressions.
- Include every applicable upstream license text with `%license`.
- Ensure `%files` owns the intended complete import trees without duplicate entries, missing nested modules, or paths owned by another RPM.
- Inspect `%install` output for duplicated directory components and overlaps with existing packages.
- Use `Conflicts`, `Obsoletes`, and compatibility symlinks only for a proven parallel-version or migration need.

## 9. Repository and OBS Validation

- Run `pre-commit` against changed files and inspect every automatic change before committing.
- Run `scripts/remoteassetify.py` or the repository's current equivalent to verify checksums.
- Create an isolated OBS subproject per branch or PR. Its build repositories must depend only on openRuyi, except when explicitly testing a documented DAG edge through another temporary project.
- Update each OBS package service to the exact Git branch and manually run `osc service remoterun <project> <package>` after every source-affecting push.
- Interpret states accurately:
  - `broken`: source service or package metadata failed before normal build.
  - `failed`: RPM build or tests ran and failed; read the build log.
  - `unresolvable`: required capabilities are unavailable in configured repositories.
  - `blocked`: another scheduled/building dependency currently prevents dispatch.
- Check exact missing capabilities from solver output. Confirm whether they belong to another DAG PR, an existing differently named package, or a missing producer capability.
- Require amd64 success as the normal repair gate. Record riscv64 separately; do not claim it probably passes.

## 10. Final Per-Package Gate

Mark the package complete only when all applicable items are evidenced:

- Identity, name, version, source, and package boundary are correct.
- Spec order, spacing, descriptions, and file list follow current policy.
- All modules are installed at real paths without conflicts.
- BuildRequires, Requires, and Provides resolve to actual producers and consumers.
- Tests cover the complete packaged repository with only justified exclusions.
- Patches follow numbering, placement, format, and upstream-status policy.
- Pre-commit and RemoteAsset checks pass.
- Isolated OBS amd64 succeeds.
- The commit is package-scoped, signed off, and has no fixup history.
