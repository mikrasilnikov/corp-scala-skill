# Design

## Context

See proposal.md for motivation. The facts that shape the approach:

- The launcher script itself is not in this repository. The skill names it by file name in 70 places across `skills/corp-scala/SKILL.md` (143 lines, 26 occurrences) and `skills/corp-scala/references/sbt-launcher.md` (391 lines, 44 occurrences), and three main specs quote it in 13 requirements (11, 4, and 2 occurrences). The skill's own name appears in two lines of `SKILL.md` (frontmatter `name`, title), four lines of `README.md`, and three scenarios of the `skill-collection` spec. No launcher or sbt project is available here, so verification is by text checks, as in every earlier change.
- This is the exact reverse of the archived change `2026-10-20-rename-launcher-to-sbt-sber`, whose design and tasks record where the token sits: inside backticks in prose, in fenced `bash` and `text` blocks, in table cells, in the YAML frontmatter `description` string, in one `###` heading, and in a fixed-width layer diagram whose comment column is aligned by spaces. That change shortened the name by four characters and added four spaces of padding on the diagram line; this one lengthens it by four and takes them out.
- The new launcher name shares its prefix with the reference document `references/sbt-launcher.md`, linked three times from `SKILL.md`, and the old name's neighbours `sbt-launch.jar` and `~/.sbt/launchers/` are still present. Checks must tell `sbt-launcher.sh` and `sbt-launcher.md` apart.
- The layout change `2026-10-20-multi-skill-layout` moved the skill with a pure `git mv` and no content edit, so that git recorded a rename. This change cannot be byte-identical, because the frontmatter `name` and the title line must change, but it can keep the content edit small enough that git still detects the rename.
- The `skill-collection` spec requires the directory name to equal the frontmatter `name`, every link to resolve inside the skill directory, and the README to list every directory under `skills/`.

## Goals / Non-Goals

**Goals:**

- Rename the launcher everywhere the skill and the specs name it, with no change to the meaning of any rule, requirement, or scenario.
- Rename the skill so that its directory, its frontmatter `name`, and the file an agent sees at the repository root agree.
- Preserve structure: same headings, same links, same line count in both skill files, same requirement names in the specs.
- Leave a reader who installed the old skill one sentence that tells them what to do.

**Non-Goals:**

- Renaming `references/sbt-launcher.md` or changing its title.
- Any mention of either old name as an alias or transition note inside the skill.
- Editing the archived changes or the main specs directly.
- Reviewing or improving the rules while touching every line that names the launcher.
- Renaming the GitHub repository; that is a repository setting, not a file in it.

## Decisions

### D1. Replace the exact token `sbt-sber.sh`, nothing wider

The implementation substitutes the literal `sbt-sber.sh` with `sbt-launcher.sh`. It does not touch the word "launcher", which stays the skill's generic term and now happens to agree with the file name, nor `sbt-launch.jar` and `~/.sbt/launchers/`, which the substitution cannot reach because they do not contain the old token.

Alternative considered: reworking sentences where "the launcher" and "`sbt-launcher.sh`" now sit side by side. Rejected: the sentences read correctly, and rewording would turn a rename into an edit of the rules.

### D2. The skill is renamed by `git mv` plus the smallest possible content edit

`git mv skills/corp-scala skills/sbt-launcher` first, then the edits inside the moved files: the launcher token everywhere, `name: corp-scala` to `name: sbt-launcher`, and `# Corporate Scala` to `# sbt launcher`. The reference changes only by the launcher token. Both files keep their line counts, so git detects the rename (well above its default similarity threshold) and `git log --follow` and `git blame` stay intact.

Alternatives considered: a pure move in one change and the edits in another, as the layout change did. Rejected: the move exists only because the name changes, so the two are one decision and one review. A new directory plus deletion of the old one: rejected, it loses the rename in history for no gain.

### D3. The skill's name is the launcher's base name

`sbt-launcher` is what an agent sees at the repository root (`sbt-launcher.sh`), what the reference is already called (`references/sbt-launcher.md`), and what the first change's trigger tokens assumed. The title line becomes `# sbt launcher`, matching the reference's "sbt launcher reference". The frontmatter description keeps the word "corporate", because the environment is still corporate; only the file-name token in it changes.

Alternatives considered: `sbt-sber`, which names the organisation the launcher is dropping from its own name; `sber-scala` or `corp-scala-build`, which are as generic as the name being replaced.

### D4. No compatibility alias for either old name; one removal sentence in the README

The skill names exactly one launcher file, and the "No launcher, no build" rule depends on that; the earlier rename's design made the same choice for the same reason. Every corporate repository ships `sbt-launcher.sh`, so nothing in the skill mentions `sbt-sber.sh`.

For the skill's own old name the situation differs in one way: an installed `corp-scala/` copy lives outside this repository, nothing here can update it, and left in place it would fire on the same repositories and name a launcher that no longer exists. The README therefore gains one sentence in its install section telling anyone who installed `corp-scala` to delete that directory. It is the only place either old name survives outside the archive, and it is a transition aid for people, not a rule for agents.

### D5. Fixed-width and anchored places are adjusted by hand after the substitution

- The layer diagram in the reference's section 1 aligns its comment column with spaces after the component name. On line 20 the token grows by four characters, so four of the sixteen spaces after it are removed and `corporate wrapper, a black box to the agent` still starts in column 28, aligned with `the runner (bash)` on line 24 and `sbt-launch.jar boots` on line 31.
- The heading `### Launcher (\`sbt-sber.sh\`)` becomes `### Launcher (\`sbt-launcher.sh\`)`. Its auto-generated anchor changes; no link points at it. The reference's intra-document links are the numbered table of contents and `#your-own-invocation-mistakes`, both unaffected, and the two anchored links from `SKILL.md` point at `#2-runner-flags`, also unaffected.
- The frontmatter `description` is a single YAML string; the token changes inside it and the string stays on one line, so the frontmatter still parses.

### D6. Specs change through this change's delta specs only, copied from the main specs

Each affected requirement is copied whole from `openspec/specs/<capability>/spec.md` into `specs/<capability>/spec.md` under `## MODIFIED Requirements`, header text identical, with only the names replaced in the body and scenarios, so that sync or archive replaces each requirement in place rather than adding a duplicate. Requirements that name neither the launcher nor the skill are left out. The blocks are extracted by script rather than retyped, so a header cannot differ from the main spec by a character.

The `skill-collection` deltas are a token substitution of `corp-scala` with `sbt-launcher`, with one addition: the scenario "The existing skill", which today checks that the skill sits under `skills/` and not at the root, also states that `skills/corp-scala/` no longer exists, so that the spec describes the repository after this change rather than after the layout change alone.

### D7. Verification by text checks

- Zero matches for `sbt-sber` and for `corp-scala` under `skills/`; zero matches for `sbt-sber` under `README.md` and the change's `specs/`; exactly one line of `README.md` matches `corp-scala`, the removal sentence.
- Per file, the count of `sbt-launcher.sh` equals the previous count of `sbt-sber.sh`: 26 in `SKILL.md`, 44 in the reference, and 11, 4, and 2 in the three launcher deltas.
- `sbt-launch.jar` (8) and `~/.sbt/launchers/` (3) still appear exactly as before in the reference; the three `references/sbt-launcher.md` links in `SKILL.md` and the one `../SKILL.md` link in the reference resolve from `skills/sbt-launcher/`.
- Line counts unchanged: 143 and 391. `git diff --cached -M --stat` shows both files as renames.
- `sed -n '2p' skills/sbt-launcher/SKILL.md` prints `name: sbt-launcher`, equal to the directory name; `test -e skills/corp-scala` fails.
- Every directory under `skills/` appears in the README's table, and the README's install example copies `skills/sbt-launcher`.
- The diagram check from the earlier rename still holds: lines 20, 24, and 31 of the reference have a space in column 27 and a letter in column 28.
- `openspec validate rename-to-sbt-launcher --strict` passes.

## Risks / Trade-offs

- [A repository whose launcher is still `sbt-sber.sh` is no longer buildable by agents under the skill] → Intended; the proposal marks it breaking, and every corporate repository ships `sbt-launcher.sh`.
- [An installed `corp-scala/` copy stays beside the new skill and names the old launcher] → D4: the README's install section says to delete it.
- [A grep for the new name also matches the reference document] → D7 counts `sbt-launcher.sh` and `references/sbt-launcher.md` separately.
- [The content edit is large enough that git records a delete and an add instead of a rename] → D2 keeps the edit to the token and two lines; D7 checks `git diff -M --stat`.
- [The diagram column or a heading anchor drifts] → D5 names both spots; D7 re-checks the three aligned lines.
- [A delta header differs from the main spec by a character, so sync adds a requirement instead of replacing one] → D6 extracts blocks by script from the main specs.
- [The changed `description` stops the skill from triggering] → Only the file-name token changes; `build.sbt`, `project/plugins.sbt`, sbt, Scala, and the task words stay.
- [The formatting change that follows is drafted against the old names] → The proposal sequences it after this change is archived.

## Migration Plan

One commit holding the rename, the content edits, the README, and the delta specs. Agents pick up the change when their skills directory gets a copy of `skills/sbt-launcher/` and loses `corp-scala/`. Repositories ship the launcher as `sbt-launcher.sh`, which is outside this repository. Rollback is reverting the commit and reinstalling the previous copy.

## Open Questions

- Whether the GitHub repository should be renamed from `corp-scala-skill`, now that nothing in it is called corp-scala. Deferrable: it is a repository setting, and the README describes the collection whatever the repository is called.
