# Proposal

## Why

The repository holds one skill, and that skill sits in `corp-scala/` directly under the repository root. The first change chose that location on purpose and rejected a `skills/` layer because there was no second skill to justify it (archived change `2026-10-19-launcher-and-dependency-rules`, design D1). More skills are now planned, so the premise of that decision is gone: with sibling skill directories at the root nothing marks which directories are skills, there is no index that lists them, and the install instruction "copy `corp-scala/`" does not generalise to a second one.

## What Changes

- The skill moves from `corp-scala/` to `skills/corp-scala/`, as a rename with its contents untouched: `skills/corp-scala/SKILL.md` and `skills/corp-scala/references/sbt-launcher.md`. The frontmatter `name: corp-scala`, the description, the body, and the links between the two files (`references/sbt-launcher.md`, `../SKILL.md`) are all relative to the skill directory and stay exactly as they are.
- `skills/` becomes the one place for skills in this repository: every skill is a directory `skills/<name>/` holding a `SKILL.md`, with `<name>` equal to the frontmatter `name`, and with every file the skill needs inside that directory, so that copying the directory alone installs it. The repository root itself is not a skill.
- A root `README.md` is added as the index: what the repository is, a list of the skills it holds with a one-line summary each, how to install one (copy `skills/<name>/` into the agent's skills directory), and the convention a new skill must follow. The README has to be updated whenever a skill is added, which is the point of having it.
- Design D1 of the first change is reversed: the `skills/` layer that was considered and declined is now the convention.
- **BREAKING** for anyone whose copy, symlink, or documentation points at `corp-scala/` at the repository root; that path stops existing. Installed copies are not affected: the installed directory is still named `corp-scala` and the skill's `name` is unchanged, so agents that already have it keep working.
- Unchanged: the skill's rules and reference text, the three main specs (`sbt-invocation`, `launcher-owned-configuration`, `project-dependencies`), the flat layout of `openspec/specs/`, the archived changes (their mentions of `corp-scala/` stay as the historical record), and the OpenSpec tooling under `.claude/`.

## Capabilities

### New Capabilities

- `skill-collection`: how skills are organised in this repository so that several can coexist and be installed independently: one directory per skill under `skills/`, named after the skill; each skill self-contained within its directory; the root README listing every skill and how to install and add one.

### Modified Capabilities

None. The three existing capabilities specify agent behaviour inside corporate Scala projects and never name the skill's location in this repository.

## Impact

- Files: `corp-scala/SKILL.md` becomes `skills/corp-scala/SKILL.md` and `corp-scala/references/sbt-launcher.md` becomes `skills/corp-scala/references/sbt-launcher.md`, both as pure renames (`git mv`, no content change). One new file, `README.md`, at the repository root. After the change there is no `corp-scala/` at the root and `grep -rn "corp-scala/" --exclude-dir=.git --exclude-dir=openspec .` matches only the README.
- Downstream: the install path changes from `corp-scala/` to `skills/corp-scala/`. Anyone who checks out this repository and symlinks or copies the old path has to repoint once. Agents that already hold an installed copy are unaffected.
- OpenSpec: specs stay flat under `openspec/specs/` for now. A later skill's capabilities get names that make their owning skill clear, or get nested under a per-skill directory when they arrive; moving the three existing specs is not part of this change (design records this).
- The GitHub repository is named `corp-scala-skill`, in the singular. Renaming it is a repository setting outside this change's files; the README describes the repository as a collection of skills regardless of its name.
- Assumptions, recorded so review can overturn them: skills are installed by copying a directory, with no plugin manifest or marketplace file in this repository (the `skills/` name is chosen so that a Claude Code plugin manifest could be added later without another move); the layout names skills in the Agent Skills format already used (`SKILL.md` with `name` and `description` frontmatter, optional `references/`, `scripts/`, `assets/`); the README is short and factual, not a user guide for each skill, since each `SKILL.md` already describes its own skill.
- Verification stays as in the earlier changes: by text checks and review, since nothing here is executable.
