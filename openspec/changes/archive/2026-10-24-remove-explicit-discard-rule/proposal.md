# Proposal

## Why

The requirement "Non-Unit results are discarded explicitly" in `scala-code-style`, and the bullet it puts in `rules/virtuslab-scala-style.md`, are withdrawn. Where a repository's compiler options warn about a discarded value, the zero-warnings requirement already makes the agent discard it explicitly, so the rule adds nothing there. Where they do not warn, the rule sends the agent to "the form the repository already uses", which a repository that never discards explicitly does not have, and the agent ends up adding `val _ =` bindings that the surrounding code never writes.

## What Changes

- The requirement "Non-Unit results are discarded explicitly" is removed from `scala-code-style`, with both of its scenarios: the unused return value that is discarded explicitly, and the result needed further down that is bound to a named `val`.
- `rules/virtuslab-scala-style.md` loses the matching bullet under "Syntax and function shape", the one that begins "When a non-Unit result is intentionally unused". No other line of the file changes.
- Two neighbouring statements stay, on purpose, because they rest on other requirements rather than on the removed one: the chaining bullet's clause that `<|` takes a function returning `Unit`, so a side effect's result is discarded inside the lambda (scenario "A side effect that returns a value" of "Single-use values are chained"), and the zero-warnings scenario in which the build reports a discarded value and the agent removes the cause rather than suppressing it. After this change the agent still discards explicitly wherever the compiler would otherwise warn, and inside `<|`; elsewhere it writes an unused non-Unit result the way the surrounding code does.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `scala-code-style`: the requirement "Non-Unit results are discarded explicitly" is removed. No requirement is added or modified.

## Impact

- `rules/virtuslab-scala-style.md`: one bullet deleted; the file keeps its five sections, its six code fences and its closing attribution, and stays within the 100 to 150 lines the archived designs set for it.
- `openspec/specs/scala-code-style/spec.md`: the requirement and its two scenarios leave the main spec when the change is archived, taking it from fourteen requirements to thirteen; the Purpose paragraph is unaffected.
- `README.md`, `skills/upickle/` and `skills/sbt-launcher/`: unchanged. Neither the README's row for the rules file nor either skill mentions discarding a result.
- Projects that installed the rules file by pasting or copying it pick the removal up when they copy the file again; no dependency or compiler option changes.
- The archived changes that introduced the rule and later restated the chaining scenario keep their text, as history.
