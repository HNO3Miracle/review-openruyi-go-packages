# Review Report Template

Report findings first. Omit empty sections.

```markdown
**Findings**
- High: `SPECS/<package>/<package>.spec:<line>` requires `go(example/path/v2)`, but no configured repository provides it. The existing `<producer>` installs that path but exposes only `<other capability>`. Fix the producer capability and update this consumer before triggering CI.
- Medium: `SPECS/<package>/<patch>:<line>` is a hand-written diff without a mail header and is numbered as an accepted upstream fix although its PR is still open. Regenerate it with `git format-patch` and keep it in the 2000 range.

**Package Status**
| Package | Result | Pre-commit | OBS amd64 | OBS riscv64 | Notes |
|---|---|---|---|---|---|
| `<package-a>` | fixed | pass | succeeded | succeeded | Complete |
| `<package-b>` | blocked | pass | unresolvable | unresolvable | Needs `<PR/package>` |

**Dependency Edges**
`<producer PR>` -> `<consumer PR>` because `<capability>` is required.

**Remaining Risk**
State only untested architectures, pending services, external PRs, or assumptions that still affect mergeability.
```

For a single package, use a short finding list followed by one validation sentence instead of a large table.

Never summarize `broken`, `failed`, and `unresolvable` as the same kind of CI failure. Never say a branch is green when any relevant package has not reached `succeeded`.
