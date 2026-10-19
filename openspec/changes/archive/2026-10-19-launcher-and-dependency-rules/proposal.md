# Proposal

## Why

Agents working in corporate Scala repositories fall back on generic sbt habits: running bare `sbt`, adding resolvers when resolution fails, adding or bumping dependencies to make code compile, and raising heap when a build runs out of memory. In this environment every one of those habits either fails, bypasses corporate authentication and repository controls, or silently changes the project's supply chain. The build only works through `./sbt-launcher.sh`, which configures and starts the sbt-extras runner with corporate repositories, credentials, and memory settings, and dependencies are approved and added by people, not agents.

This is the first change of the `corp-scala` skill, which will grow to describe how agents work on Scala projects with spec-driven development in a corporate environment. It establishes the build-invocation and dependency guardrails that every later requirement will assume.

## What Changes

- Introduce the `corp-scala` skill as the deliverable of this repository: a `SKILL.md` that agents load when working in a corporate Scala project, plus a reference document for the launcher and its argument model.
- Agents invoke sbt only through `./sbt-launcher.sh`, run from the repository root, with `-batch` on every invocation. The command line carries sbt tasks and commands plus settings that only change sbt's output; `--error` is never used because it hides warnings.
- Agents separate infrastructure failures from build failures. A missing or failing launcher, an unresolvable dependency, or an `OutOfMemoryError` means stop and tell the user what happened. A compile or test failure after sbt has loaded is ordinary engineering work. So is a mistake in the agent's own command, such as running from the wrong directory or a malformed option: the agent corrects it and runs again, as long as the corrected command follows every invocation rule.
- Launcher-owned configuration is read-only to agents: authentication, repositories, and memory settings, including the launcher script itself, every repository configuration file at the repository root (one per build environment), `.sbtopts` and `.jvmopts`, the user's `~/.sbt` tree, and the environment variables and runner flags that would override them.
- Agents never add, remove, upgrade, or re-scope project dependencies in any form, including library dependencies, sbt plugins, `scalaVersion`, `sbt.version`, unmanaged jars, and inter-module `dependsOn`, and never pull artifacts through side channels. When new functionality needs a dependency, the agent states what it needs and asks the user to add it.

## Capabilities

### New Capabilities

- `sbt-invocation`: how an agent runs the build. The launcher as the only entry point, the working directory, mandatory `-batch`, which command-line options are permitted, how infrastructure failures are told apart from build failures and handled, and how the agent corrects its own invocation mistakes.
- `launcher-owned-configuration`: the configuration an agent must treat as read-only because the launcher owns it: authentication, repositories, memory, the files and environment variables that carry them, and the launcher as a black box.
- `project-dependencies`: the rule that agents never change the project's dependency set, what counts as a dependency, the side channels that are also closed, how agents use dependencies that already exist, and how they ask for new ones.

### Modified Capabilities

None. The project has no existing specs.

## Impact

- New files: `corp-scala/SKILL.md` and `corp-scala/references/sbt-launcher.md` at the repository root. No existing code is touched; the repository currently holds only OpenSpec scaffolding.
- Downstream: every agent session that loads the skill in a corporate Scala repository. The skill constrains agent behaviour only; it does not change the launcher, sbt-extras, or any project build.
- External dependency: the reference document describes the sbt-extras argument model (runner flags versus pass-through sbt commands, option files, environment variables) at the sbt-extras revision the launcher uses. A project keeps one sbt-extras version for its lifetime, so the description only needs re-checking if the launcher ever changes it.
- Out of scope for this change, to be added later: a warning policy for code quality, the spec-driven development workflow itself, and the remaining corporate requirements such as formatting, testing conventions, and approved libraries. The launcher script and the repository configuration files are inputs to this skill, not deliverables of it.
