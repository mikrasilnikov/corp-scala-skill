# Agent skills

This repository is a collection of agent skills in the `SKILL.md` format. Each skill is a directory `skills/<name>/` that holds its `SKILL.md` and every file it needs, so it can be installed on its own. The `.claude/` directory holds the OpenSpec tooling for working on this repository; it contains no skills to install.

## Skills

| Skill | Summary |
| --- | --- |
| `sbt-launcher` | Building, testing, formatting, and changing dependencies in corporate Scala projects built with sbt. |

## Installing a skill

Copy the skill's directory, `skills/<name>/`, into your agent's skills directory. The installed directory keeps the skill's name: `skills/sbt-launcher/` installs as `sbt-launcher/`. For Claude Code, for example:

```bash
mkdir -p ~/.claude/skills
cp -R skills/sbt-launcher ~/.claude/skills/
```

This creates `~/.claude/skills/sbt-launcher/SKILL.md`. To install a skill for one project only, copy it into that project's `.claude/skills/` instead.

If you installed this skill under its former name, delete that `corp-scala/` directory from your skills directory: left beside `sbt-launcher/`, it names a launcher that no longer exists.

## Adding a skill

- Create the directory `skills/<name>/` with a `SKILL.md` whose frontmatter `name` is `<name>`. Skills go nowhere else: not at the repository root, and not under `.claude/`, which is tooling.
- Keep every file the skill needs (`references/`, `scripts/`, `assets/`) inside that directory, and make every link relative to it. Do not link into another skill; if two skills need the same text, each keeps its own copy.
- Add a row for the skill to the table above, with a one-line summary of its `description`, in the same change.
