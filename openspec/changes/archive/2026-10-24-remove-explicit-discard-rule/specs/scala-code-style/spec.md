# Spec Delta

## REMOVED Requirements

### Requirement: Non-Unit results are discarded explicitly
**Reason**: Where the repository's compiler options warn about a discarded value, the requirement "Code compiles with zero warnings" already makes the agent discard it explicitly, and where they do not, this requirement sent the agent to a form "the repository already uses" that a repository without explicit discards does not have, so the agent added `val _ =` bindings the surrounding code never writes.
**Migration**: An unused non-Unit result that the compiler would warn about is still discarded explicitly, under "Code compiles with zero warnings", which fixes the cause in code rather than suppressing the warning; a side effect's result inside `<|` is still discarded inside the lambda, under the scenario "A side effect that returns a value" of "Single-use values are chained". Any other unused non-Unit result is written as the surrounding code writes one, and a result that is needed further down is bound to a `val` as any needed value is, which no rule has to say.
