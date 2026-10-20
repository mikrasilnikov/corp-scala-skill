# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- The launcher script itself is not in this repository. The skill names it by file name in 70 places across `corp-scala/SKILL.md` (143 lines) and `corp-scala/references/sbt-launcher.md` (391 lines), and the three main specs quote it in 13 requirements. No launcher or sbt project is available here, so verification is by text checks, as in the first change.
- The old name appears in every kind of Markdown position: inside backticks in prose, in fenced `bash` and `text` blocks, in table cells, in the YAML frontmatter `description` string, in one `###` heading, and in a fixed-width layer diagram whose comment column is aligned by spaces. The new name is four characters shorter.
- Three unrelated tokens share the prefix and must survive untouched: `sbt-launch.jar` (the sbt launcher jar that sbt-extras downloads), `~/.sbt/launchers/` (its cache directory), and `references/sbt-launcher.md` (the reference document, linked three times from `SKILL.md`).
- The archived change `2026-10-19-launcher-and-dependency-rules` holds a verification trace keyed by line numbers of the two skill files. It is a frozen record, but a rename that keeps every line in place keeps that trace readable against the new text.

## Goals / Non-Goals

**Goals:**

- Rename the script everywhere the skill and the specs name it, with no change to the meaning of any rule, requirement, or scenario.
- Preserve structure: same headings, same links, same line count in both skill files, same requirement names in the specs.

**Non-Goals:**

- Renaming `references/sbt-launcher.md` or changing its title "sbt launcher reference".
- Any mention of the old name as an alias or transition note.
- Editing the archived change or the main specs directly.
- Reviewing or improving the rules themselves while touching every line that names the launcher.

## Decisions

### D1. Replace the exact token `sbt-launcher.sh`, nothing wider

The implementation substitutes the literal `sbt-launcher.sh` with `sbt-sber.sh`. It does not touch the word "launcher", which remains the skill's generic term (the report area "launcher precondition", the heading "Launcher-owned configuration", the sentence "the launcher is a black box"), nor the prefix `sbt-launch`, which also belongs to `sbt-launch.jar` and `~/.sbt/launchers/`.

Alternative considered: renaming the concept too ("the sber script", "sber-owned configuration"). Rejected: the concept is still a launcher; only its file name is corporate.

### D2. The reference document keeps its file name

`references/sbt-launcher.md` describes the whole stack (wrapper, sbt-extras, sbt), not the script alone. Its title and the three links from `SKILL.md` stay. Renaming it would change nothing an agent types and would be a separate, trivial change if symmetry is wanted later.

### D3. No compatibility alias for the old name

The skill names exactly one launcher file, and the "No launcher, no build" rule depends on that. Writing "`./sbt-sber.sh` (or `./sbt-launcher.sh` in older repositories)" would give an agent a second file to look for and a reason to keep going when the expected one is missing. The request was a rename, not an addition. A repository that still ships `sbt-launcher.sh` is fixed by renaming the file in that repository.

### D4. Fixed-width and anchored places are adjusted by hand after the substitution

- The layer diagram in the reference's section 1 aligns its comment column with spaces after the component name. The line `sbt-launcher.sh            corporate wrapper, ...` gets four more spaces after `sbt-sber.sh` so the column still lines up with `sbt-extras \`sbt\` script` and `sbt` below it. Markdown tables need no re-alignment.
- The heading `### Launcher (\`sbt-launcher.sh\`)` becomes `### Launcher (\`sbt-sber.sh\`)`. Its auto-generated anchor changes. No link points at it: the only intra-document links are the numbered table of contents and `#your-own-invocation-mistakes`, both unaffected.
- The frontmatter `description` is a single YAML string; the token changes inside it and the string stays on one line, so the frontmatter still parses.

### D5. Specs change through this change's delta specs only

Each of the 13 affected requirements is copied whole from `openspec/specs/<capability>/spec.md` into `specs/<capability>/spec.md` under `## MODIFIED Requirements`, with the header text identical and the file name replaced in the body and scenarios, so that sync or archive replaces each requirement in place rather than adding a duplicate. Requirements that do not quote the file name are left out of the deltas. The apply tasks do not edit `openspec/specs/` directly.

### D6. Verification by text checks

- Zero matches for `sbt-launcher.sh` under `corp-scala/` and under the change's `specs/`.
- Per file, the count of `sbt-sber.sh` equals the previous count of `sbt-launcher.sh`: 26 in `SKILL.md`, 44 in the reference, and 11, 4, and 2 in the three deltas.
- `sbt-launch.jar`, `~/.sbt/launchers/`, and `references/sbt-launcher.md` still appear exactly as before (the first two by count, the third by the three links resolving).
- Line counts unchanged: 143 and 391.
- `openspec validate rename-launcher-to-sbt-sber --strict` passes.

## Risks / Trade-offs

- [A repository whose launcher is still `sbt-launcher.sh` is no longer buildable by agents under the skill] → Intended; see D3. The fix is renaming the script in that repository, and the proposal marks this as breaking.
- [The substitution hits an unrelated token] → D1 restricts it to the exact file name; D6 checks `sbt-launch.jar`, `~/.sbt/launchers/`, and the reference links afterwards.
- [The diagram column or a heading anchor drifts] → D4 names both spots; D6 re-reads the diagram block.
- [A delta header differs from the main spec by a character, so sync adds a requirement instead of replacing one] → Blocks are copied by script from the main specs with only the file name changed; the header lines are checked for exact matches before tasks are written.
- [The changed `description` stops the skill from triggering] → Only the file-name token changes; `build.sbt`, `project/plugins.sbt`, sbt, Scala, and the task words stay.

## Migration Plan

Agents pick up the change by copying the updated `corp-scala/` directory into their skills directory; rollback is restoring the previous copy. Repositories must name the launcher `sbt-sber.sh` at the root, which is outside this repository.
