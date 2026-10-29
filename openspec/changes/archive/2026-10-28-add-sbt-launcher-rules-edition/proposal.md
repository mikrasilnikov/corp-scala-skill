# Proposal

## Why

The body of `skills/sbt-launcher/SKILL.md` is a set of obligations that hold for every change in a corporate Scala repository: every build runs through `./sbt-launcher.sh`, every change is formatted with `scalafmtAll` even when nothing is compiled, and no dependency is ever added by an agent. This repository already has a test for such text. The design of the archived change `2026-10-22-add-virtuslab-scala-style-rules` (D1) ruled that rules which must be in context whenever Scala is written belong in a rules file, loaded whole on every session, because a skill loads only when the agent judges a task relevant to its description. For the launcher rules that judgment is a compliance risk rather than a loss of capability: a rename with no build never looks like "format code", a library added in passing while coding never looks like "touch a dependency", and a quick `sbt test` never reaches the skill at all. The skill predates rules files in this repository by three days, and the question was recorded in the other direction, as a skill edition of the style rules, but never asked of the launcher skill.

## What Changes

- Add `rules/sbt-launcher.md`, a rules edition of the skill's body: the same obligations in the same words, packaged as a flat rules file that a project loads whole on every session. It takes the skill's name, so that `skills/sbt-launcher/` and `rules/sbt-launcher.md` are read at once as one set of rules in two packagings.
- The rules edition differs from the body of `skills/sbt-launcher/SKILL.md` only where the packaging shows: it has no frontmatter; a preamble paragraph stating scope and precedence, as every rules file must, stands where the paragraph that sends the reader to `references/sbt-launcher.md` stood, that paragraph being dropped since the reference is explanation and the body is complete without it, as the reference itself says; the two remaining pointers to the flag table are dropped the same way; and "this skill and its reference" becomes "these rules". Every other line, the opening paragraph included, is identical.
- The two editions stay in step: a change that edits a rule in one edits it in the other in the same change, and the permitted differences are enumerated so that a comparison of the two files shows nothing else.
- The README's Rules files section gains a row for the new file, which says that it mirrors the `sbt-launcher` skill; the install step says that the file is installed in the repositories that have the launcher at their root, and that a project may carry the rules file while an engineer has the skill installed, since the agent then meets the same text twice at most; the convention for adding a rules file gains the mirror rule, and its other bullets are brought in line with the `rules-collection` spec: the scope bullet is generalised as the scope requirement is, the first bullet asks for a heading on the first line, and the last asks for a library row and install precondition where a file relies on libraries.
- The `rules-collection` spec, written with one rules file in view, is generalised for a second kind of file: the layout requirement's first-file scenario becomes a two-file scenario; the scope-and-precedence requirement, which asks for the language, the language versions and the kind of code, is reworded around what the repository must provide for the rules to be followed, so that it fits a file about a build as well as a file about code; and a requirement for a rules file that mirrors a skill is added.
- Unchanged: every file under `skills/`, including `skills/sbt-launcher/SKILL.md` and its reference, byte for byte; `rules/virtuslab-scala-style.md`; the four content specs `sbt-invocation`, `launcher-owned-configuration`, `project-dependencies` and `source-formatting`, which describe the agent's behaviour whichever packaging delivers the text; and `skill-collection`.
- Not breaking: nothing moves or is renamed. Installed copies of the skill keep working, and the new file is additive.

## Capabilities

### New Capabilities

None. The behaviour the rules edition asks of an agent is already specified by `sbt-invocation`, `launcher-owned-configuration`, `project-dependencies` and `source-formatting`; this change adds a second packaging of the text that states it, and the layout and installation of rules files belong to `rules-collection`.

### Modified Capabilities

- `rules-collection`: the layout requirement's "first rules file" scenario is updated to name both files; the scope-and-precedence requirement is generalised from libraries to what the repository must provide, so that a rules file about a build states the launcher, plugin and configuration it relies on and what the agent does when one is missing; a requirement is added for a rules file that mirrors a skill: the skill's name, the skill's every rule in the skill's words, enumerated differences only, both editions changed together, and the mirror named in the README.

## Impact

- Files: one new file `rules/sbt-launcher.md` of about 160 lines; `README.md` gains one table row, a few sentences in the install step and one bullet in the convention, and the convention's other three bullets are reworded; the artifacts of this change. Nothing under `skills/` or `openspec/specs/` changes until archive, which updates `rules-collection`.
- Always-on cost: the body is about 20 KB, and a project that installs both rules files holds about 36 KB of rules in every session. The skill alone held only its 843-byte description in context until it triggered.
- Downstream: a team installs the file per repository by the README's two routes. The file belongs in the repositories that carry the launcher, so the natural owner of the rules file is whoever standardises `sbt-launcher.sh` across repositories and can ship the two together; that rollout is outside this repository.
- Maintenance: every later edit to a launcher rule is made twice, once per edition, in the same change; the enumerated differences make the check mechanical.
- Verification: text checks and a read-through, as in the earlier changes; nothing in this repository is executable.
