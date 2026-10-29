# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- The skill is two files. The body of `skills/sbt-launcher/SKILL.md`, the text after its frontmatter, is 160 lines and about 20 KB: an H1, an opening paragraph, a paragraph pointing at the reference, then four sections, `Running the build`, `Formatting`, `Launcher-owned configuration` and `Dependencies`, with two fenced templates (the stop-and-report shape and the dependency request) and two fenced bash blocks, one of them indented inside a bullet. `references/sbt-launcher.md` is 438 lines and its opening paragraph says "The skill file is what you obey. This file explains", so nothing normative lives only in the reference.
- Exactly four places in the body show the skill packaging: the pointer paragraph after the opening (body line 6); the parenthetical "with reasons in the [flag table](...)" in the allowlist bullet (line 30); "shown elsewhere in this skill and its reference" in the first formatting rule (line 95); and the sentence "The [flag table](...) explains each one." under "No overrides through runner flags" (line 125). No other line names the skill, the reference or a link.
- `rules-collection` requires a rules file to begin with a heading, carry no frontmatter and no links, depend on no other file, and state scope and precedence before its first rule. Its scope requirement was worded for a style file ("the language, the language versions, and the kind of code", required libraries), and its first-file scenario says `rules/` holds one entry.
- The README's Rules files section has one row, an install paragraph that opens with the libraries `virtuslab-scala-style` needs, the two OpenCode routes, and a three-bullet convention. The spec says a file without library rules gets no library text in its row or install step.
- `rules/virtuslab-scala-style.md` is 111 lines and about 16 KB and contains fenced code blocks, so fenced blocks are already accepted in a rules file. Its design budgeted 100 to 150 lines for a file held in context on every session.
- The design of `2026-10-22-add-virtuslab-scala-style-rules` recorded a skill edition of the style rules as an open question; this change is that question's mirror image.
- Nothing in the repository is executable, and the earlier changes verify by text checks.

## Goals / Non-Goals

**Goals:**

- Put the launcher obligations in a packaging that is in context on every session of a governed repository, without the skill having to trigger.
- Keep one wording: the rules edition and the skill body are the same text apart from an enumerated, checkable set of differences.
- Leave the skill and its installed copies untouched.
- Make `rules-collection` fit a rules file about a build as well as one about code.

**Non-Goals:**

- Moving the body out of the skill or shrinking the skill to its reference, the split-by-nature option. It breaks installed copies and leaves the skill unable to enforce anything on its own; it stays available as a later change once the rules edition is adopted.
- Generating one edition from the other with a script.
- A digest or shorter rules file: a third text to keep in step.
- Changing the reference, the four content specs, `skill-collection` or `rules/virtuslab-scala-style.md`.
- A Claude Code install route in the README: the existing sentence that the text works in any agent that reads a Markdown instructions file covers `CLAUDE.md`.

## Decisions

### D1. A second edition beside the skill, not a move

Three shapes were weighed. Keeping the skill alone leaves the trigger risk on every change. Moving the body to `rules/` and shrinking the skill to a diagnosis entry point matches each artifact to its loading mechanism, but it breaks every installed copy, leaves an engineer who has only the skill with no guardrails, and the rules file could not point at the reference, since a rules file links nowhere. A second edition is additive, breaks nothing, closes the gap in every repository that installs it, and keeps the skill whole for user-level installs. Its price is duplication, which the collection specs already accept as a principle ("each keeps its own copy") and which D3 makes checkable. The move stays possible later.

### D2. The rules file takes the skill's name

`rules/sbt-launcher.md`, matching `skills/sbt-launcher/`. The same name in the two collections says "the same rules, packaged twice", which is what the README then states in one clause. Alternative considered: `sbt-launcher-rules.md`. Rejected: the `rules/` directory already says it, and a different name would suggest a different text.

### D3. The body is the text; the rules edition is the body with four enumerated differences

The rules file is the body of `SKILL.md` copied verbatim, then changed in exactly four places:

1. The opening: the blank line after the frontmatter is dropped so the file begins with its heading; the H1 `# sbt launcher` and the opening paragraph stay as they are; the pointer paragraph that follows them is replaced by the preamble paragraph of D4, one line like every paragraph in the body, so that no line below it shifts.
2. Line 30 of the body: ", with reasons in the [flag table](references/sbt-launcher.md#2-runner-flags)" is removed, so the sentence ends "and the rest), and the `set ...` and `++ ...` commands (Scala version, compiler options, resolvers, dependencies)."
3. Line 95: "shown elsewhere in this skill and its reference" becomes "shown elsewhere in these rules".
4. Line 125: the sentence "The [flag table](references/sbt-launcher.md#2-runner-flags) explains each one." is removed.

Everything else, including the two fenced templates and the two bash blocks, is identical. The check is mechanical: `diff <(awk 'BEGIN{fm=0} /^---$/{fm++; next} fm>=2 && (p || NF){p=1; print}' skills/sbt-launcher/SKILL.md) rules/sbt-launcher.md`, which compares the rules file with the body stripped of the blank line after the frontmatter, prints exactly four one-line hunks, today `5c5`, `29c29`, `94c94` and `124c124`, and nothing else; both sides are 159 lines. A later change that edits a rule makes the same edit in both files and runs the same diff, whose hunk numbers move with the text but whose count stays four.

Alternatives considered: a shorter digest, rejected as a third text to maintain and because the body already holds only obligations, so there is little to cut; a generated edition, rejected because nothing in the repository is executable and the transformation is four hand edits; dropping the bash blocks and templates, rejected because the templates are rules about the shape of the agent's output and the blocks show the exact command the rules demand.

### D4. The preamble

The preamble is one paragraph on one line, standing where the body's pointer paragraph stood, after the opening paragraph that both editions share. The two paragraphs together meet the generalised scope requirement before the first `## `; the preamble says:

- Scope: the rules apply throughout the repository the agent is working in ("this repository"), provided it builds its Scala code with sbt, to every sbt command the agent runs, every Scala source change it makes and every dependency it considers for that code or its sbt build, on any Scala and sbt version. The scope test is the build tool alone: the launcher is a provision, not a marker, so a repository that lacks it stays governed and "No launcher, no build" fires as soon as a rule needs the launcher. The shared opening paragraph already says that the build works only through the launcher, which starts the sbt-extras runner with the corporate credentials, repositories and memory settings, and that dependencies are approved and added by people, not agents.
- Precedence: the rules are not conventions and do not yield to the repository's habits or to any other tool that is installed; where a rule hands a decision to the repository, such as which files scalafmt formats or which modules carry the plugin, the rule itself says so.
- Provisions: the repository must have the launcher at its root, the sbt-scalafmt plugin in its build and a `.scalafmt.conf`. A missing one leaves the rules in force: the agent stops and reports it when a rule below finds it missing, as that rule describes, such as "No launcher, no build" or formatting not available, and never supplies it. No library is named, since none is needed.

The first wording scoped the rules to "every Scala repository built with sbt that has `./sbt-launcher.sh` at its root". It made the launcher both the scope test and a provision, so a repository whose launcher was missing fell out of scope, and an agent could skip "No launcher, no build" and run another sbt. Three rewordings were rejected. "In the repository they are installed in" refers to nothing when the text reaches the agent through a user-level instructions file. "Wherever its Scala code is built with sbt" exempts code that sbt does not build, such as a scala-cli or Ammonite script, which is exactly what "No side channels" and "Formatting runs only through the launcher" name. "This Scala repository built with sbt" assumes the build tool instead of testing it. The provisions sentence first said "when one of them is missing, stop and report it as the rules below describe", which read as an order to stop on the absence alone, even in a task that reaches no rule needing the item; it now says the rules still apply and ties the stop to the rule that finds the item missing. The dependency clause is limited to the repository's Scala code and its sbt build, so that an npm or Python part of the same repository, its own build tooling included, is not pulled under the dependency rules; every other clause already names Scala or sbt.

The preamble names neither the skill nor the reference, so the file depends on nothing (D5).

### D5. The pointers are dropped, not redirected

The three pointers to `references/sbt-launcher.md` are removed rather than reworded to name the `sbt-launcher` skill as further reading. A rules file is read where the skill may not be installed; naming the skill would make the file depend on something outside it, which `rules-collection` forbids; and the reference's opening paragraph says the skill file is what you obey, so the body loses nothing normative without it. An agent that needs a flag outside the command-line table already has its rule: ask the user; `--debug` and `-trace <n>`, which the table permits, need no reference.

### D6. README changes

Six edits to the Rules files section and nothing elsewhere:

- A row for `sbt-launcher` whose summary condenses the skill's description, "Building, testing, formatting and changing dependencies in corporate Scala projects built with sbt through `./sbt-launcher.sh`", and ends by saying that it mirrors the `sbt-launcher` skill, packaged to be loaded on every session.
- Two sentences in the install paragraph: `sbt-launcher` is installed in the repositories that have the launcher at their root, by the same two routes; a project may carry the rules file while an engineer has the skill installed, since the agent then meets the same text twice at most. Nothing about libraries, as the spec requires for a file that needs none.
- One bullet in the convention: a file that mirrors a skill's body takes the skill's name, keeps every line identical to the body except where the packaging differs, says so in its row, and is changed together with the skill. The bullet lists the differences the ADDED requirement allows (no frontmatter or the blank line after it, a preamble after the opening paragraph that states everything the scope bullet asks for, no pointer to the skill or its supporting files), since a contributor reads the README without this change's artifacts.
- The convention bullet that restates the scope requirement is generalised as the MODIFIED requirement is (D7): it names the scope for rules about code (the language, its versions and the kind of code) and for rules about a build (the build tool), the provisions for each (libraries by organisation and artifact, with any compiler option, or the scripts, plugins and configuration files the repository must have), and the behaviour the requirement's scenarios fix for each: for a library or compiler option the agent asks for it, by organisation and artifact or by name, and never adds it or writes another form; for a script, plugin or configuration file it reports it and never supplies it; and a rules file about a build says that the rules still apply when one is missing. Left as it was, the bullet would teach the next author the requirement's old, library-only form.
- The first and last bullets of the convention restate two main-spec requirements the README did not carry, found when the convention was read as a contributor would read it: the first asks for a Markdown heading on the file's first line ("A rules file is self-contained and has no frontmatter"), and the last asks that a file whose rules rely on libraries name them in its row and in an install precondition, and that a file relying on none say nothing about libraries in either ("The README names a rules file's required libraries as an install precondition").

The Skills table row for `sbt-launcher` is left as it is; the mirror is stated once, on the rules side.

### D7. Specs: a `rules-collection` delta and nothing else

The four content specs state what the agent does, whichever text tells it so (design D10 of the style-rules change); they are not touched. `skill-collection` is not touched: nothing under `skills/` changes and the README's skill sections are unchanged. The `rules-collection` delta: the layout requirement is MODIFIED so that its first-file scenario names both files; the scope requirement is MODIFIED to generalise libraries to "what the repository must provide", with scenarios for a rules file about a build; a requirement for a file that mirrors a skill is ADDED. Both MODIFIED descriptions and the ADDED one are kept under 500 characters so that `openspec validate --specs --strict` passes on the main spec after archive.

### D8. Verification by text checks

- `head -1 rules/sbt-launcher.md` prints `# sbt launcher`; `grep -cE '\]\(' rules/sbt-launcher.md` prints 0; `grep -ci 'skill' rules/sbt-launcher.md` prints 0; `grep -c 'references/' rules/sbt-launcher.md` prints 0.
- The diff of D3 prints exactly the four one-line hunks named there and nothing else, and `wc -l rules/sbt-launcher.md` prints 159, the body's line count without the blank line after the frontmatter.
- The text before the first `## ` names the launcher, the sbt-scalafmt plugin, `.scalafmt.conf`, the precedence and the stop-and-report behaviour, and says that the rules still apply when any of the three is missing.
- `diff <(ls rules | sed 's/\.md$//') <(awk '/^## Rules files/,0' README.md | grep -oE '^\| `[a-z0-9-]+`' | tr -d '|` ')` prints nothing; the README's row for `sbt-launcher` contains "mirrors"; `git status --porcelain` lists nothing under `skills/`.
- Paste the file into a scratch `AGENTS.md` outside the repository and read it once as an agent asked for an edit with no build, checking that the formatting rule is reached without any trigger and that every sentence still reads correctly in place; then once each as an agent in a Scala repository built with sbt that lacks `./sbt-launcher.sh` (the preamble keeps it in scope and "No launcher, no build" is reached), as an agent changing a scala-cli script there ("No side channels" and "Formatting runs only through the launcher" still govern it), and as an agent asked only to explain code where `.scalafmt.conf` is missing (nothing in the preamble alone makes it stop).
- `openspec validate add-sbt-launcher-rules-edition --strict` passes.

## Risks / Trade-offs

- [The two editions drift] → the mirror requirement makes the same-change edit a rule, and the four-hunk diff of D3 is the check any later change runs.
- [Always-on cost] → about 20 KB for this file, about 36 KB with the style file. The body already holds only obligations, so this file, at 159 lines, takes no line budget like the style file's 100 to 150; if a later change cuts it, the cut is made in both editions.
- [The file is installed in a repository without the launcher] → the preamble keeps the repository in scope, so as soon as a rule needs the launcher, "No launcher, no build" makes the agent stop and say the repository is not set up for agent builds; intended.
- [An agent with both editions treats them as two rule sets] → identical wording, and the README says they are one text.
- [The preamble's scope sentence lets an agent decide the rules do not apply] → the scope is the whole repository the agent is working in, tested only by whether it builds its Scala code with sbt; the launcher, the plugin and `.scalafmt.conf` are provisions whose absence the preamble says leaves the rules in force, and no Scala code in the repository is exempt because a tool other than sbt builds or runs it, such as a scala-cli or Ammonite script.
- [The MODIFIED scope requirement reads as a rewrite rather than a generalisation] → the existing clauses are kept; the libraries clause becomes one case of "what the repository must provide", and the build case is carried by new scenarios.

## Migration Plan

One commit holding the rules file, the README edit and this change's artifacts. A team adopts the file by pasting it into a project's `AGENTS.md` or listing it under `instructions`, and drops it the same way. Nothing existing moves, so rollback is reverting the commit; installed skills are unaffected either way.

## Open Questions

- Whether to make the split-by-nature move later, body to `rules/` and skill shrunk to a diagnosis entry point over the reference, once repositories carry the rules file. Deferrable: this change is additive; the move would be a breaking change of its own.
- Whether the one scenario in `source-formatting` that says "in a way the skill permits" should say "in a way the rules permit". Deferrable: wording in a scenario, not behaviour; a one-line delta in a later change.
- Whether OpenCode shows a skill's description without loading the skill, which decides how much the skill alone mitigates the trigger gap. Deferrable: it does not change this design.
- Who installs the rules file across corporate repositories, the launcher's owners or each team. Outside this repository.
