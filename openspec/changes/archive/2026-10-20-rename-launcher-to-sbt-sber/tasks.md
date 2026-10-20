# Tasks

## 1. Skill body: `corp-scala/SKILL.md`

- [x] 1.1 Replace every `sbt-launcher.sh` with `sbt-sber.sh` in `corp-scala/SKILL.md` (design D1: the exact token only, so the word "launcher", the three `references/sbt-launcher.md` links, and `sbt-launch.jar` stay as they are), including the two occurrences inside the frontmatter `description` and the shell error example `no such file or directory: ./sbt-launcher.sh`. Verify: `grep -o "sbt-sber.sh" corp-scala/SKILL.md | wc -l` prints 26, `grep -c "sbt-launcher.sh" corp-scala/SKILL.md` prints 0, `wc -l corp-scala/SKILL.md` still prints 143, and `git diff --numstat corp-scala/SKILL.md` shows 22 insertions and 22 deletions.
- [x] 1.2 Check the frontmatter and links after the edit: the YAML block still opens and closes with `---`, has `name: corp-scala`, and has a single-line `description:` that now names `./sbt-sber.sh`; the three links to `references/sbt-launcher.md` (one plain, two with `#2-runner-flags`) are unchanged. Verify: `grep -c "references/sbt-launcher.md" corp-scala/SKILL.md` prints 3, `sed -n '1,4p' corp-scala/SKILL.md` shows the intact frontmatter, and `ls corp-scala/references/sbt-launcher.md` succeeds.

## 2. Reference: `corp-scala/references/sbt-launcher.md`

- [x] 2.1 Replace every `sbt-launcher.sh` with `sbt-sber.sh` in the reference (design D1), including the heading on line 178 (`### Launcher (\`sbt-launcher.sh\`)` becomes `### Launcher (\`sbt-sber.sh\`)`), the shell error examples under "Before the launcher runs" and "Launcher", the "Your own invocation mistakes" table, and the worked-examples and common-mistakes tables in section 5. Verify: `grep -o "sbt-sber.sh" corp-scala/references/sbt-launcher.md | wc -l` prints 44, `grep -c "sbt-launcher.sh"` on it prints 0, `wc -l` still prints 391, and `git diff --numstat` shows 37 insertions and 37 deletions.
- [x] 2.2 Re-align the layer diagram in section 1 (design D4): on line 20 the component name is now four characters shorter, so add four spaces after `sbt-sber.sh` so that `corporate wrapper, a black box to the agent` starts in column 28, the same column as `the runner (bash)` on line 24 and `sbt-launch.jar boots` on line 31. Verify: `sed -n '20p;24p;31p' corp-scala/references/sbt-launcher.md | grep -c '^.\{26\} [a-z]'` (a space in column 27, a letter in column 28) prints 3 (it prints 3 before the edit as well; a substitution without the padding drops it to 2).
- [x] 2.3 Confirm the unrelated tokens survived (design D1, D6): `grep -c "sbt-launch\.jar" corp-scala/references/sbt-launcher.md` still prints 8, `grep -c "~/\.sbt/launchers" corp-scala/references/sbt-launcher.md` still prints 3, the title line is still `# sbt launcher reference`, and `diff <(git show HEAD:corp-scala/references/sbt-launcher.md | grep -n '^#') <(grep -n '^#' corp-scala/references/sbt-launcher.md)` reports exactly one differing line, the heading from task 2.1.

## 3. Whole-skill checks

- [x] 3.1 Run the whole-skill text checks from design D6: `grep -rn "sbt-launcher.sh" corp-scala/` prints nothing; `git status --porcelain` lists exactly `corp-scala/SKILL.md` and `corp-scala/references/sbt-launcher.md` as modified and nothing else (the archived change and `openspec/specs/` are untouched); `grep -rn "sbt-launcher.sh" openspec/changes/rename-launcher-to-sbt-sber/specs/` prints nothing.
- [x] 3.2 Read the renamed rules once end to end as an agent would: the "No launcher, no build" paragraph, the report template (`Command:   ./sbt-sber.sh -batch <exact arguments>`), the common-mistakes table (`cd core && ../sbt-sber.sh -batch compile`), and the frontmatter description read naturally with the new name and say nothing that still implies the old one. Verify: `openspec validate rename-launcher-to-sbt-sber --strict` passes and the read-through finds no sentence to reword; if it does, reword without changing meaning and re-run the counts in 1.1 and 2.1.

## Workflow follow-up

- Archive the change after review. Archiving applies the 13 modified requirements to the three main specs; afterwards `grep -rn "sbt-launcher.sh" openspec/specs/` prints nothing and `openspec validate --specs` passes.
- Rename the launcher to `sbt-sber.sh` at the root of each corporate repository that still ships `sbt-launcher.sh`. That is outside this repository (proposal: Impact).
- Optional later change, if symmetry is wanted: rename `references/sbt-launcher.md` and its three links (design D2 keeps it for now).
