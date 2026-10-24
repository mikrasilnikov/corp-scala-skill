# Tasks

## 1. Rules file

- [x] 1.1 Delete from `rules/virtuslab-scala-style.md`, under "Syntax and function shape", the whole bullet that begins "When a non-Unit result is intentionally unused" (line 18 today), leaving the chaining bullet above it and the unbounded-data bullet below it untouched; verify `grep -n 'intentionally unused' rules/virtuslab-scala-style.md` prints nothing, `grep -c 'deleteIfExists' rules/virtuslab-scala-style.md` prints 0, and `grep -n 'discard' rules/virtuslab-scala-style.md` prints only the chaining bullet, which still says that `<|` takes a function returning `Unit`
- [x] 1.2 Confirm the deletion is the file's only change: verify `git diff --shortstat rules/virtuslab-scala-style.md` prints `1 file changed, 1 deletion(-)`, `wc -l rules/virtuslab-scala-style.md` prints 113, `grep -c '^## ' rules/virtuslab-scala-style.md` prints 5, `` grep -c '^```' rules/virtuslab-scala-style.md `` prints 6, and `tail -1 rules/virtuslab-scala-style.md` still names VirtusLab and `Apache-2.0`

## 2. Coherence checks

- [x] 2.1 Read "Syntax and function shape" in `rules/virtuslab-scala-style.md` against `openspec/specs/scala-code-style/spec.md` with this change's delta applied: verify every bullet still maps to a remaining requirement and every remaining requirement to a bullet, no bullet is left for the removed requirement, and the scenario "A side effect that returns a value" and the zero-warnings scenario on a discarded value are still backed by the chaining bullet and the zero-warnings bullet respectively
- [x] 2.2 Confirm no other deliverable names the rule and nothing else changed: verify `grep -rn -i 'intentionally unused' README.md rules skills` prints nothing and `git status --porcelain` lists only `rules/virtuslab-scala-style.md` and `openspec/changes/remove-explicit-discard-rule/`
- [x] 2.3 Verify `openspec validate remove-explicit-discard-rule --strict` passes

## Workflow follow-up

- Archive the change after review; archiving removes the requirement and its two scenarios from `openspec/specs/scala-code-style/spec.md`, after which `grep -c '^### Requirement:' openspec/specs/scala-code-style/spec.md` prints 13 and `grep -n 'discard' openspec/specs/scala-code-style/spec.md` prints only the `<|` scenario and the zero-warnings scenario.
- Verify the archived result.
