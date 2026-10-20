# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- The skill is two files, `corp-scala/SKILL.md` (143 lines) and `corp-scala/references/sbt-launcher.md` (391 lines), with frontmatter `name: corp-scala`. The only links between them are `references/sbt-launcher.md` (three times, from the skill body) and `../SKILL.md` (once, from the reference). Both are relative to the skill directory, so they survive a move of the directory as a whole. Neither file names its own location in the repository.
- The repository root holds, besides the skill, `openspec/` (specs and changes), `.claude/` (the OpenSpec workflow skills and commands used to work on this repository, not deliverables), and `.gitignore`. There is no README.
- The first change's design D1 chose `corp-scala/` at the root and rejected a `skills/` layer only because there was no second occupant. It also rejected `SKILL.md` at the root because the repository would then be the skill, named after its checkout directory, and a second skill could never live beside it. Both of those arguments still hold; the first one now points the other way.
- Outside this repository the `skills/<name>/SKILL.md` layout is the established one for a collection: Anthropic's own skills repository uses it, a Claude Code plugin keeps its skills in `skills/`, and a project's local skills live in `.claude/skills/<name>/`. In every case the directory name is what the harness installs under, and the frontmatter `name` is what the agent sees.
- The two archived changes name `corp-scala/` in their proposals, designs, tasks, and the first change's verification trace. They are frozen records and are not edited.
- Nothing in this repository is executable, so verification is by text checks, as in both earlier changes.

## Goals / Non-Goals

**Goals:**

- Put the skill where a second, third, and later skill can sit beside it without any of them moving again.
- Change nothing inside the skill: not a byte of `SKILL.md` or the reference, so the move is a pure rename that git tracks as one.
- Make the convention explicit once, in the `skill-collection` spec and the README, so adding a skill later is following a rule rather than inventing one.

**Non-Goals:**

- A Claude Code plugin manifest (`.claude-plugin/plugin.json`) or a marketplace file. Not asked for; the `skills/` name keeps the door open.
- Renaming the GitHub repository `corp-scala-skill`. That is a repository setting, not a file in it.
- Restructuring `openspec/specs/` per skill, or editing the three existing main specs.
- Editing the archived changes to update their `corp-scala/` paths.
- A README per skill, or any content in the root README that restates a skill's rules.

## Decisions

### D1. One layer: `skills/<name>/`, with the directory named after the frontmatter `name`

Every skill is `skills/<name>/SKILL.md`, and `<name>` equals the frontmatter `name`, so that the directory a person copies and the name an agent is shown agree. For the existing skill that is `skills/corp-scala/` with `name: corp-scala`, which already matches.

Alternatives considered:

- Sibling directories at the root (`corp-scala/`, `corp-foo/`, ...). Rejected: nothing distinguishes a skill directory from `openspec/` or `.claude/`, each addition widens the root, and "copy `<dir>/`" has to be re-explained per skill.
- Grouping by domain (`skills/scala/corp-scala/`). Rejected as premature: there is one skill and no second domain; a flat `skills/` can be regrouped later if a real grouping appears.
- Making the repository a plugin now. Not asked for; see Non-Goals. Choosing `skills/` means a manifest can be added later without another move.

### D2. The move is a `git mv` and nothing else touches the skill

`git mv corp-scala skills/corp-scala`, with no edit to either file in the same change. Because the skill's links are relative and it never names its own path, no content needs to change, and a pure rename keeps `git log --follow` and `git blame` intact (git detects a rename at 100% similarity). Any edit to the skill's text belongs in its own change, as the two earlier changes did.

A `.DS_Store` or other untracked file inside `corp-scala/` would survive `git mv` and keep the old directory in place; the tasks check for and remove any leftover so that `corp-scala/` really is gone.

### D3. A root `README.md` is the index, and it is short

The README carries four things and no more: one paragraph saying the repository is a collection of skills in the `SKILL.md` format; a list with one row per directory under `skills/`, giving the name and a one-line summary condensed from that skill's own frontmatter description; the install instruction (copy `skills/<name>/` into the agent's skills directory); and the convention for adding a skill (the three requirements of the `skill-collection` spec in plain words, plus "add a row here"). One sentence notes that `.claude/` holds tooling for working on this repository, not skills to install, so nobody looks for deliverables there.

Alternatives considered: no README, relying on `ls skills/`. Rejected: a listing gives names without summaries or install steps, and the proposal's purpose is to make the next addition obvious. A README per skill: rejected, `SKILL.md` already describes its skill and a second description would drift from it.

### D4. `openspec/specs/` stays flat; per-skill nesting is deferred to the first real collision

The three existing capabilities keep their paths. A future skill's capabilities are named so their owning skill is unambiguous, and if two skills ever need the same capability name, that is the moment to nest (`openspec/specs/<skill>/<capability>/`), moving the affected specs in a change made for that purpose.

Alternative considered: nest the three existing specs under `openspec/specs/corp-scala/` now, while there is only one skill. Rejected: OpenSpec has no rename operation for a capability, so it would be a manual move of three main specs plus a mismatch with the archived changes' delta paths, in exchange for a benefit that appears only when a second skill has specs. Doing it later costs the same.

### D5. Archived changes are left as written

The archived proposals, designs, tasks, and verification trace say `corp-scala/`. They describe the repository as it was when they were applied and are not rewritten. Anyone reading them after this change finds the new location through the README and this change.

### D6. Verification by text checks

- `test -f skills/corp-scala/SKILL.md && test -f skills/corp-scala/references/sbt-launcher.md` succeed; `test -e corp-scala` fails.
- `git diff --cached --stat -M` (or `git status --porcelain`) shows the two files as renames and the README as the only addition; `git diff --cached -M --numstat` shows 0 insertions and 0 deletions for both renamed files.
- `cmp` of each moved file against `git show HEAD:corp-scala/<path>` reports no difference.
- The three `references/sbt-launcher.md` links and the one `../SKILL.md` link resolve from their new locations.
- `sed -n '2p' skills/corp-scala/SKILL.md` prints `name: corp-scala`, matching the directory name.
- Every directory under `skills/` appears in the README's list, and the README names `skills/<name>/` as the install path.
- `grep -rn "corp-scala/" --exclude-dir=.git --exclude-dir=openspec .` matches only `README.md`.
- `openspec validate multi-skill-layout --strict` passes.

## Risks / Trade-offs

- [A checkout, symlink, or external document still points at `corp-scala/` at the root] → Intended and marked breaking in the proposal; the README states the new path. Installed copies keep working because the installed directory name and the skill `name` do not change.
- [The rename is combined with a content edit and git stops seeing it as a rename] → D2 forbids edits to the skill in this change; D6 checks 0/0 numstat on both files.
- [An untracked file keeps `corp-scala/` alive after the move] → D2 and the tasks remove leftovers and check `test -e corp-scala` fails.
- [The README's skill list drifts from `skills/` as skills are added] → The `skill-collection` spec requires the README entry in the same change as the skill; the tasks include a check that compares `ls skills/` with the list.
- [A future skill's capability name collides with an existing one] → D4 names the trigger and the remedy; nothing to do now.
- [Someone takes `.claude/skills/` for the deliverables] → The README says in one sentence what `.claude/` is.

## Migration Plan

One commit holding the rename and the new README. Anyone who installs from a checkout repoints to `skills/corp-scala/`; installed copies need no action. Rollback is reverting that commit, which restores `corp-scala/` at the root and removes the README.

## Open Questions

- Whether the GitHub repository should be renamed from `corp-scala-skill` to something that says "skills". Deferrable: it is a repository setting, and the README describes the collection whatever the repository is called.
- Whether to add a Claude Code plugin manifest once there are several skills. Deferrable: D1 chose `skills/` so that this needs no further move.
