# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- The repository holds only OpenSpec scaffolding. The skill does not exist yet, so this is greenfield.
- A skill is a directory with a `SKILL.md` (YAML frontmatter with `name` and `description`, then Markdown instructions) and optional `references/`, `scripts/`, and `assets/`. Loading is layered: the frontmatter is always in context, the body loads when the skill triggers, references load only when the body points the agent at them. The body should stay well under 500 lines.
- The launcher is a thin wrapper over the sbt-extras `sbt` script, and a project keeps one sbt-extras version for its lifetime. The launcher sets authentication, repository, and memory configuration and forwards every other argument. sbt-extras consumes the runner flags it recognises and passes the remainder to sbt as commands. Three details matter for the rules: `.sbtopts`, `.sbtxopts`, `SBT_OPTS`, and `SBTX_OPTS` go through the same flag parser as the command line, while `.jvmopts` and `JVM_OPTS` go to the JVM verbatim; `-S<opt>` is rewritten into `set scalacOptions in ThisBuild += ...`; `-trace <n>` is rewritten into `set traceLevel in ThisBuild := n` (`set every traceLevel := n` in revisions before August 2014). `-mem` has not been an sbt-extras flag since 2013, though the official sbt runner script still has it. With no command at all the runner defaults to the interactive `shell`.
- No launcher or sbt project is available in this repository, so the skill is written against the launcher contract the user described, and verification is by review and dry-run prompts rather than by running a build.
- Later changes will add further corporate requirements to the same skill, so the structure must leave room for them.

## Goals / Non-Goals

**Goals:**

- Encode the three capabilities so an agent can check its own compliance mechanically: an explicit allowlist for the command line, a concrete read-only file list, a content rule for recognising repository configuration files, and an observable marker separating infrastructure failures from build failures.
- Keep `SKILL.md` short enough to be read every time; move the sbt-extras detail into one reference file.
- Make the structure extensible: one top-level section per capability, so later changes add sections and reference files rather than restructuring.

**Non-Goals:**

- Writing, wrapping, or modifying the launcher. No `scripts/` directory in this change: a wrapper around the wrapper would be another thing to keep in sync and could be mistaken for the launcher itself.
- An automated eval harness. This change verifies by tracing scenarios to skill text and by a handful of dry-run prompts.
- A warning policy for code quality, the dependency approval process on the human side, and the spec-driven workflow itself.

## Decisions

### D1. Skill name and location: `corp-scala/` at the repository root

`corp-scala/SKILL.md` and `corp-scala/references/sbt-launcher.md`. The skill is installed by copying the `corp-scala/` directory into an agent's skills directory.

Alternatives considered: `SKILL.md` at the repository root would make the repository itself the skill, so its name would come from the checkout directory (`corp-scala-skill`) and a second skill could never live beside it. A `skills/corp-scala/` layer adds a level for which there is no second occupant yet.

### D2. Two layers of disclosure

`SKILL.md` carries everything an agent must obey on every build: launcher only, from the root, `-batch` always, the command-line allowlist, never `--error`, the stop-and-report list with the report format, the build-failure path, the read-only configuration list, the environment-variable list, the black-box rule, the dependency rule with the request template. Target under 150 lines.

`references/sbt-launcher.md` carries the explanation: a diagram of the layers (agent, launcher, sbt-extras, sbt), the full sbt-extras flag table classified as allowed or forbidden with what each flag does, the option files and environment variables sbt-extras parses, the failure-layer markers with example output, worked command examples including quoting and module scoping, and the sbt-extras revision the table was checked against. `SKILL.md` sends the agent there when it needs any flag beyond `-batch` and `-no-colors`, or when a launcher run fails.

Alternative considered: a single `SKILL.md`. Rejected because the flag table alone is long and only needed occasionally, and later changes will add material to the body.

### D3. Allowlist, not denylist, on the command line

The permitted set is named: sbt tasks and commands, `-batch`, `-no-colors`, `--debug`, `-trace <n>`. Everything else is forbidden. The denylist in the reference exists to explain *why* each forbidden flag is forbidden, not to define the boundary. Agents follow "only these" more reliably than "not those", and the allowlist is the operational form of the agreed principle that only output-affecting settings are permitted.

`--warn` is output-only and therefore permitted by the spec, but the skill does not recommend it because it also hides the test pass summary. `--error` is forbidden outright because it hides warnings.

### D4. Failure classification by sbt's load marker

The agent decides which path it is on by whether sbt printed `welcome to sbt` or `loading settings for project` before the failure. Before the marker: launcher or infrastructure, stop and report. After the marker: build failure, fix the code, with two named exceptions that are still stop-and-report: unresolved artifacts (sbt or coursier `not found`, `download failed`, `Error downloading`) and `OutOfMemoryError` or GC overhead errors.

The agent's own invocation mistakes are a separate category. The command ran from somewhere other than the repository root, passed a permitted option in a malformed form, or mistyped, misquoted, or misscoped an sbt command. The agent corrects the command and runs it again instead of stopping. The correction may only bring the command into line with the invocation rules, never add a forbidden flag, setting, variable, or tool. A command that follows every rule and still fails before the marker is infrastructure, so stop and report. An earlier draft made these stop-and-report too, which halted tasks on typos the agent could fix itself.

Alternatives considered: exit codes, rejected because both layers exit non-zero; matching launcher-specific messages, rejected because the launcher is a black box and its messages are not known to the skill.

### D5. Recognising repository configuration files by content

A file at the repository root that contains a `[repositories]` section is launcher-owned. That is the format sbt itself requires for `sbt.repository.config`, so it holds regardless of file names, which vary by environment and are not known here. The skill adds a conservative default: a root-level file the agent cannot identify as source, build definition, or documentation is left alone. The agent never reads the launcher to learn which files it references, because that would breach the black-box rule.

### D6. Report format for stop-and-report

The skill specifies what the report contains so that reports are actionable: the exact command run, the working directory, the error output verbatim, which launcher-owned area is implicated (launcher precondition, launcher jar download, repositories, memory, Java), and a statement that the agent has not changed anything. The same shape is used for the dependency request, with coordinates, module, configuration, reason, and the declared alternatives considered.

### D7. Discovering declared dependencies without editing

The skill tells the agent to read the build definition files, including `project/*.scala`, or to run a read-only query such as `./sbt-launcher.sh -batch -no-colors "show core/libraryDependencies"`. Both are permitted: reading build files is not modifying them, and `show` is an sbt command that prints a setting.

### D8. Frontmatter description tuned for triggering

The description names the concrete tokens an agent sees in a corporate Scala repository: Scala, sbt, `build.sbt`, `project/plugins.sbt`, `sbt-launcher.sh`, compile, test, and the corporate context. It must trigger on any build, test, or dependency question in such a repository and must not trigger in a non-Scala repository.

### D9. Verification without a launcher

Two checks, both recorded in the change:

1. A scenario trace: every `#### Scenario` across the three specs is mapped to the line or lines in `SKILL.md` or the reference that cover it. A scenario with no line is a gap to fix before the change is done.
2. Dry-run prompts: a small set of adversarial situations (unresolved dependency, out of memory, launcher missing, agent in a module directory, code needs a new library, running one test suite, and the agent's own invocation mistakes with their limits) given to an agent with the skill loaded, with the responses checked against the spec, plus trigger checks on prompts that should and should not activate the skill. The `skill-creator` skill's eval flow is available for a fuller benchmark later.

## Risks / Trade-offs

- [sbt-extras flag surface drifts from the reference table] → Unlikely, since a project keeps one sbt-extras version for its lifetime. The reference still records the revision and date it was checked against so a future launcher change can be reconciled.
- [Agent over-applies stop-and-report and refuses to fix ordinary compile errors] → The load marker and the two named exceptions are stated in `SKILL.md` with examples of both paths.
- [Agent under-applies it and treats an unresolved artifact as a code problem] → Resolution failures are listed as an explicit exception with the typical message fragments.
- [Agent labels an infrastructure failure as its own mistake to keep going] → A correction may only bring the command into line with the invocation rules, never add a flag, setting, variable, or tool, and a command that follows every rule but still fails before the marker is stop-and-report.
- [Allowlist too tight for a legitimate future need, for example an sbt server flag] → The skill tells the agent to ask the user; widening the allowlist is a spec change.
- [`[repositories]` content rule misses a differently formatted file] → The conservative default leaves unfamiliar root files alone.
- [Launcher may already pass `-batch` or `-no-colors`] → Passing them twice is harmless; the skill passes them regardless.
- [Inter-module `dependsOn` prohibition proves too tight in practice] → Accepted knowingly; relaxing it is a one-line spec change later.
- [Skill fails to trigger] → Description names concrete tokens; trigger prompts are part of verification.

## Migration Plan

Not applicable. The skill is new and has no users yet. Rollout is copying `corp-scala/` into agent skill directories; rollback is removing it.

## Open Questions

- The exact file names of the per-environment repository configuration files. Deferrable: the content rule in D5 covers any name.
- Whether the launcher already adds `-batch` or `-no-colors` itself. Deferrable: duplicates are harmless.
