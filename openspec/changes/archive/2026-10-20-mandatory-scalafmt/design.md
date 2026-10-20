# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- `skills/sbt-launcher/SKILL.md` (143 lines) has three top-level sections, "Running the build", "Launcher-owned configuration", and "Dependencies", one per existing capability. "Running the build" holds the invocation pattern, the command-line allowlist (a table whose last row is "sbt tasks and commands"), a `bash` example block, and the subsection "When a build fails", which classifies failures by sbt's load marker: own invocation mistakes (three cases, one of them "sbt rejected a command you mistyped ... `Not a valid key`"), before-the-marker failures, after-the-marker work, "Two failures after the marker are still stop-and-report", and the report template whose first line lists the report areas. The frontmatter `description` is one YAML line that names the trigger tokens.
- `skills/sbt-launcher/references/sbt-launcher.md` (391 lines) explains failures by layer in section 4, with abbreviated example blocks it already marks as approximate ("Exact wording varies by version"), an "After the marker: stop-and-report exceptions" subsection, an "After the marker: your work" subsection, and a "Your own invocation mistakes" table whose last row maps `Not a valid key` to "Fix the spelling". Section 5 is two tables, worked examples and common mistakes. The table of contents lists six sections.
- The sbt-scalafmt plugin, per the scalafmt documentation: `scalafmtAll` runs `scalafmt` for every configuration in which it is enabled (since 2.0.0-RC5); `scalafmtOnly <file>...` formats listed files; `scalafmtCheck` and `scalafmtCheckAll` check without formatting; `scalafmtSbt` formats `*.sbt` and `project/**/*.scala`; `scalafmtConfig` defaults to `.scalafmt.conf` at the project root; the scalafmt version is pinned in that file and downloaded dynamically on first use; `scalafmtFilter` limits the files considered (`diff-dirty`, `diff-ref=<spec>`, `none`, or git-tracked files); `scalafmtOnCompile` formats on compile and the documentation discourages it; `scalafmtSbtOnLoad` can run `scalafmtSbt` on every load (the key exists from sbt-scalafmt 2.6.2; the 2.5.6, 2.6.0, and 2.6.1 jars do not have it). Every one of these is a build setting or a task, so the repository owner controls scope through them, and the agent cannot, because `set` is outside the allowlist and the build definition is not the agent's to edit for this.
- `scalafmtAll` covers the configurations of the current project only (`ScopeFilter(configurations = inAnyConfiguration)`), so an unscoped run from the root reaches another module only through sbt aggregation. A build with no root project gets a default root that aggregates every module; an explicitly defined root (`project in file(".")`) aggregates the modules its `.aggregate(...)` lists and, transitively, whatever those aggregate, and a build can turn aggregation off for the task with `scalafmtAll / aggregate := false`. A module whose build turns the plugin off (`.disablePlugins(ScalafmtPlugin)`) has no `scalafmtAll`: an aggregated run skips it, and `<module>/scalafmtAll` fails with `No such setting/task`.
- A missing plugin surfaces after the load marker, as `[error] Not a valid key: scalafmtAll` (with `Not a valid command` beside it), the same message the skill today reads as the agent's own typo. A missing `.scalafmt.conf` surfaces as a plugin error after the marker as well, `scalafmt: File does not exist: <root>/.scalafmt.conf`. A scalafmt version that cannot be fetched surfaces as `scalafmt: [v<version>] failed to download [<root>/.scalafmt.conf]`, which matches none of the messages the skill lists for an unresolved artifact and names `.scalafmt.conf`, so it needs naming explicitly.
- Decisions taken during exploration and recorded in the proposal: every corporate repository has the plugin and a `.scalafmt.conf`; `scalafmtAll` is the only required task and the repository decides its scope; `.scalafmt.conf` is read-only; a missing plugin or configuration stops the task.
- No sbt project, launcher, or scalafmt is available in this repository, so verification is by text checks, as in every earlier change, plus a live run in throwaway projects outside the repository (sbt 1.13.0 and 2.0.11 through a local `sbt-launch.jar`, sbt-scalafmt 2.6.2, scalafmt 3.11.5), which is where the messages above and the aggregation behaviour were observed.

## Goals / Non-Goals

**Goals:**

- One rule an agent can follow without bookkeeping: put `scalafmtAll` first in the invocation it was going to run anyway.
- Fit the existing invocation pattern (one sbt start per round, tasks in order) and the existing failure classification (extend the after-the-marker list, do not restructure it).
- Keep scope decisions with the repository owner and keep the agent's hands off every scalafmt setting.
- Keep the skill readable as one document: a section per capability, failure handling in one place.

**Non-Goals:**

- A per-file or changed-files rule in the skill; the plugin's `scalafmtFilter` does that where the repository wants it.
- Requiring `scalafmtSbt`, `scalafmtCheckAll`, or any check-only gate.
- Changing any launcher-owned or dependency rule, or the `launcher-owned-configuration` and `project-dependencies` specs.
- Tracking the plugin's exact message wording across versions; the reference quotes one verified combination and says other versions differ.
- Behaviour for repositories that do not use scalafmt; policy says there are none, and the skill treats one as not set up.

## Decisions

### D1. `scalafmtAll`, unscoped, first in the invocation

The required task is `scalafmtAll`, with no project scope, placed before `compile` and `test` in the same launcher invocation: `./sbt-launcher.sh -batch scalafmtAll compile test`. Formatting first means sbt stops before compiling a file scalafmt cannot parse, and one start covers the whole round, as the existing "several tasks go in one invocation" rule requires. Unscoped means the repository's own scope settings decide what is touched across every module the root aggregates; a `core/scalafmtAll` in its place would be the agent narrowing that policy. Modules the root does not aggregate are D11.

Alternatives considered: `scalafmtOnly` with a list of changed files computed from git, rejected because the plugin already offers the same semantics to the owner through `scalafmtFilter = diff-dirty`, and a rule the agent computes would duplicate it with extra steps and a second source of truth; `set scalafmtFilter ...` on the command line, rejected because `set` is outside the allowlist; `scalafmtCheckAll` as a gate, rejected because it fails on files the agent did not touch where the repository has not narrowed scope, and because formatting is the goal, not a report; module-scoped `scalafmtAll` to match `core/compile`, rejected as a replacement for the unscoped task, and kept only as an addition for a module the root does not aggregate (D11).

### D2. The rule triggers on building, not on editing

The skill says: every invocation that compiles or tests begins with `scalafmtAll`, and a change that ends without a build gets `./sbt-launcher.sh -batch scalafmtAll` on its own. It does not say "after each edit", which would mean one sbt start per edit against the one-start rule, and it does not say "when you changed a source file", which would make the agent keep a list; `scalafmtAll` costs little when nothing has changed. Read-only queries such as `show` and `projects` are not builds and need no formatting first.

### D3. A `## Formatting` section of its own; failure handling stays in "When a build fails"

The new rules go in a top-level `## Formatting` section between "Running the build" and "Launcher-owned configuration", so the skill keeps one section per capability. The pieces that belong to failure classification (the third stop-and-report case, the own-mistake carve-out, the parse error as the agent's work, the `formatting` report area) are edited into "When a build fails", so an agent diagnosing a failure still finds the whole classification in one place. The Formatting section points at it rather than repeating it.

Alternative considered: a `### Formatting` subsection under "Running the build". Rejected: the section would then hold the read-only rule for `.scalafmt.conf`, which is not about running the build, and the skill's section-per-capability shape would be lost.

### D4. Missing plugin or configuration is the third stop-and-report case, not a dependency request

"Two failures after the marker are still stop-and-report" becomes three. The third, **Formatting not available**, is sbt rejecting a correctly spelled `scalafmtAll` or the plugin reporting a missing `.scalafmt.conf`; the report area is `formatting`, and the template's first line lists it. The own-mistake case "sbt rejected a command you mistyped ... `Not a valid key`" gains the clause that a `Not a valid key` for a correctly spelled scalafmt task is not a typo: the agent checks the spelling against the literal `scalafmtAll` once and, if it matches, treats the failure as formatting not available. A real typo is also recognisable from sbt's own hint, which lists `scalafmtAll` among the similar keys.

The report template's last line, "I have not changed anything.", becomes two lines: `Changed before the stop: <files you edited and files scalafmtAll reformatted, or none>` and "I have not changed anything to work around this failure." A formatting stop comes after the agent has edited code, and any stop after `scalafmtAll` succeeded in the same invocation comes after it reformatted files (D6), so the old line would be false there and had no room to name those files.

This is a stop, not a dependency request, because the plugin is required of every corporate repository rather than needed by the agent's task; the repository is non-compliant in the same way as one without a launcher, and the next step belongs to whoever sets up repositories. The `project-dependencies` scenario "Task needs an sbt plugin" still holds (the agent does not add the plugin and tells the user), so that spec is not modified; the new capability states which report shape applies.

### D5. Read-only scalafmt configuration lives in the Formatting section

`.scalafmt.conf` and the scalafmt settings in the build definition (`scalafmtFilter`, `scalafmtOnCompile`, `scalafmtConfig`, `scalafmtSbtOnLoad`, and the rest), together with which projects have the plugin (`enablePlugins`, `disablePlugins`), are declared read-only in the Formatting section, not added to the "Read-only files" list under "Launcher-owned configuration", because their owner is the repository, not the launcher, and that list is defined by launcher ownership. The section also says in so many words that `set` for any of them is outside the allowlist, although the allowlist already excludes every `set`.

### D6. Reformatted files outside the agent's edit stay formatted, and the agent says so

Where the repository has not narrowed scope, the agent's first `scalafmtAll` may reformat files it never opened. The skill tells the agent to leave them formatted and to name them when it reports, never to revert them. Reverting would undo the repository's own policy; staying silent would surprise reviewers reading the diff. The repository owner can narrow the scope with `scalafmtFilter`, and the proposal records that as the owner's lever.

### D7. Description changes: add tokens, remove none

The frontmatter `description` gains "formatting" in its first sentence, `.scalafmt.conf` among the root files that identify such a repository, "format code" among the moments to read the skill, and one clause in its "Covers" sentence about formatting every change with `scalafmtAll` through the launcher. Every existing token stays, and the description stays one YAML line.

### D8. Reference additions keep the section structure and quote verified output

Section 4 gains, under "After the marker: stop-and-report exceptions", one block with the `Not a valid key: scalafmtAll` / `Not a valid command: scalafmtAll` lines and a sentence on the shape of a missing-configuration error, report area `formatting`; under "After the marker: your work", one block showing a scalafmt parse error pointing at a source file and line; and, in the "Your own invocation mistakes" table, a row after the `Not a valid key` row stating that a correctly spelled scalafmt task rejected this way is not a mistake and is reported as formatting not available. Section 5 gains one worked-example row, "Format, compile, and test in one start", and two common-mistake rows: a narrowed or substituted formatting task, and formatting through the scalafmt CLI or an IDE. No new section and no change to the table of contents. The missing-plugin, missing-configuration, and parse-error blocks are abbreviated real output from the live run (sbt 1.13.0, sbt-scalafmt 2.6.2, scalafmt 3.11.5), with a note that other versions word them differently; the unresolved-artifact passage gains the plugin's `failed to download` message for the pinned scalafmt version, and the worked examples and common mistakes gain the non-aggregated module case, and the own-mistakes table a second row for `No such setting/task` on an added module task (D11).

### D9. One new capability, one modified requirement

`source-formatting` is a new capability with its own requirements, each stating one behaviour with scenarios: the formatting-first invocation, which also covers compiling and testing invocations that other capabilities' scenarios give without `scalafmtAll` (such as `sbt-invocation`'s re-run of `./sbt-launcher.sh -batch compile`), so those capabilities need no modification for it; the scope left to the repository; modules the root does not aggregate (D11); read-only configuration, launcher-only formatting, missing setup stopping the task, and scalafmt source errors as the agent's work. `sbt-invocation` is modified in one requirement only, "The agent corrects its own invocation mistakes", which gains a scenario excluding a correctly spelled scalafmt task from the correction rule; its SHALL text already limits corrections to the agent's own mistakes, so only the scenario is added. The two other launcher capabilities are untouched for the reasons in D4 and D5. The capability keeps the flat layout and no skill prefix, like its three siblings.

### D10. Verification by text checks and a live run

- `grep -c "scalafmtAll" skills/sbt-launcher/SKILL.md` is at least 6 (the description, the Formatting section, the allowlist row, the `bash` example, the third stop case, the own-mistake clause); `grep -c "scalafmtOnly" skills/sbt-launcher/SKILL.md` is at least 1 and every occurrence is in a forbidden list, none in a command the agent is told to run.
- `grep -c "^## " skills/sbt-launcher/SKILL.md` prints 4 and the headings are, in order, "Running the build", "Formatting", "Launcher-owned configuration", "Dependencies".
- The report template's first line contains `| formatting>`; the phrase "Three failures after the marker are still stop-and-report" is present and "Two failures after the marker" is not.
- `sed -n '3p' skills/sbt-launcher/SKILL.md` is the whole description, contains `.scalafmt.conf`, "formatting", `scalafmtAll`, and every token it contained before, and lines 1 and 4 are still `---`.
- The reference still has seven `## ` headings (Contents and the six numbered sections) and the same table of contents; `grep -c "scalafmtAll" skills/sbt-launcher/references/sbt-launcher.md` is at least 3; the three `references/sbt-launcher.md` links in `SKILL.md` and the `../SKILL.md` link in the reference resolve.
- `README.md`'s `sbt-launcher` row mentions formatting; the table still lists every directory under `skills/`.
- `openspec validate mandatory-scalafmt --strict` passes; the `sbt-invocation` delta's requirement header matches the main spec's exactly.
- Outside the repository, in throwaway projects with their own sbt and coursier homes: the missing-plugin, typo, missing-configuration, parse-error, unfetchable-version, aggregated multi-module, non-aggregated root, `scalafmtAll / aggregate := false`, and disabled-plugin cases print what the skill and reference say; `scalafmtAll core/scalafmtAll core/compile` formats `core` under a root that does not aggregate it, and fails with `No such setting/task` where `core` disables the plugin.

### D11. A changed module the root does not aggregate gets `<module>/scalafmtAll` as well

Where the root does not aggregate a module the agent changed, the unscoped `scalafmtAll` succeeds without touching that module, and `scalafmtAll core/compile core/test` passes with `core` unformatted and no error; the live run showed exactly that under a root defined with `dependsOn(core)` and no `.aggregate`. The agent then appends `<module>/scalafmtAll` right after the unscoped task, `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll core/compile core/test`, and `./sbt-launcher.sh -batch scalafmtAll core/scalafmtAll` for a change with no build. The unscoped task still runs first, so this adds a module rather than narrowing the repository's scope, and `core/scalafmtAll` reads the same `.scalafmt.conf` and `scalafmtFilter`.

The agent tells the case from the build definition: with no root project defined, sbt's default root aggregates every module; an explicit root aggregates the modules its `.aggregate(...)` lists and, transitively, whatever those aggregate; `scalafmtAll / aggregate := false` turns aggregation off for the task. When it cannot tell, it adds the module task anyway; where the root does aggregate the module, scalafmt finds those files already formatted and nothing changes.

A module whose build turns the plugin off (`.disablePlugins(ScalafmtPlugin)`) is outside the repository's formatting scope. There `<module>/scalafmtAll` fails with `No such setting/task` after the unscoped task succeeded, which the live run showed; the agent drops the module task, runs the same invocation again with `scalafmtAll` still first, and names the module as excluded in its report. This is a correction of the agent's own command, classified with the own-mistake cases in "When a build fails", not formatting not available, since the plugin is present in the build; turning the plugin on or off is read-only (D5).

Alternatives considered: stop and report with area `formatting` when a changed module is not aggregated, rejected because such builds are legitimate and the plugin and configuration are present, so blocking the task would treat a working repository as non-compliant; leaving it as an unstated assumption that corporate roots aggregate everything, rejected because the failure is silent and the agent would report an unformatted change as formatted.

## Risks / Trade-offs

- [A repository without the plugin, or with a plugin older than 2.0.0-RC5 that has no `scalafmtAll`, stops agents] → Intended and marked breaking in the proposal; the fix is in that repository (add or upgrade the plugin), and the report names the task sbt rejected.
- [A repository that has not set `scalafmtFilter` gets a large reformatting diff on the agent's first build] → D6: left formatted and named in the report; the owner narrows scope with `scalafmtFilter`. The proposal records this consequence.
- [`Not a valid key` is read as a typo and the agent retries, or as a missing plugin when it was a typo] → D4: one spelling check against the literal `scalafmtAll` decides; the reference's table row spells it out.
- [The pinned scalafmt version cannot be fetched through the corporate repositories] → The unresolved-artifact case: report area repositories, no version change in `.scalafmt.conf`. Its message list gains the plugin's `failed to download`, which would otherwise be misread as a configuration problem because it names `.scalafmt.conf`.
- [The plugin's messages in another version differ from the reference's examples] → D8 quotes one verified version, says others differ, and places them by position after the marker, as section 4 does for other layers.
- [A changed module the root does not aggregate stays unformatted without an error] → D11: the agent adds `<module>/scalafmtAll` after the unscoped task, and adds it whenever it cannot tell.
- [The added module task fails because the build disables the plugin for that module] → D11: `No such setting/task` after a successful unscoped run means excluded by the owner; the agent drops the module task, runs again, and names the module, without turning the plugin on.
- [Other capabilities' scenarios show compiling or testing invocations without `scalafmtAll` after archive] → D9: the formatting-first requirement states that it applies to them, and a scenario shows the conversion.
- [`SKILL.md` outgrows the first design's 150-line target] → Accepted; a rule that applies to every build belongs in the skill, and the examples live in the reference.
- [The longer description stops triggering, or triggers on non-sbt Scala projects] → D7 adds tokens and removes none; the closing "Not for projects that do not build with sbt" stays.
- [`scalafmtOnCompile` set by a repository makes `scalafmtAll` redundant] → Harmless: the task runs twice over already-formatted files; the rule stays simple.
- [`scalafmtAll` fails on a file the agent never touched, so the build cannot proceed] → The spec has the agent name the file and the error to the user and forbids dropping `scalafmtAll` to get past it; fixing or excluding that file is the user's call, since the agent had no reason to edit it.

## Migration Plan

One commit holding the skill edits, the reference edits, the README cell, and the delta specs. Agents pick up the change when their skills directory gets the updated `skills/sbt-launcher/`. Repositories need the plugin and `.scalafmt.conf`, which policy already requires. Rollback is reverting the commit and reinstalling the previous copy.

## Open Questions

- Whether the corporate build template should set `scalafmtFilter := "diff-dirty"` so that `scalafmtAll` touches only changed files everywhere. Deferrable: it is a decision for the template's owners outside this repository, and the skill's rule is the same either way.
