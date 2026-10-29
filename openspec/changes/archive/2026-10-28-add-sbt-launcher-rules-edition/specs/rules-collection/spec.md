# Spec Delta

## MODIFIED Requirements

### Requirement: Every rules file is one Markdown file directly under `rules/`
Each rules file in this repository SHALL be a single Markdown file `rules/<name>.md` with a kebab-case `<name>`. No rules file SHALL exist anywhere else, and `rules/` SHALL contain nothing but rules files: no subdirectories and no supporting files.

#### Scenario: The first rules file
- **WHEN** the repository is checked out after this change
- **THEN** `rules/` contains `virtuslab-scala-style.md`, the first rules file, and `sbt-launcher.md`, and any other entry in it is a rules file added under this requirement

#### Scenario: A second rules file is added
- **WHEN** a new rules file named `<name>` is added
- **THEN** it is created as `rules/<name>.md`, and no existing rules file or skill changes because of the addition

#### Scenario: A rules file is not a skill
- **WHEN** an agent or a person inspects `rules/`
- **THEN** they find no `SKILL.md` and no directory there, so nothing under `rules/` is loadable as a skill, and `skills/` is unchanged by the presence of rules files

### Requirement: A rules file states its scope and precedence first
Before its first rule, a rules file SHALL state what it applies to, what wins when one of its rules conflicts with the repository it is loaded in, and what the repository must provide for its rules to be followed, with what the agent does when a provision is missing. For rules about code, the scope is the language, its versions and the kind of code, and the provisions are the libraries every governed module must declare, by organisation and artifact, with any compiler option one needs.

#### Scenario: Opening a rules file
- **WHEN** an agent reads the file from the top
- **THEN** it learns the scope and the precedence before it meets any rule

#### Scenario: Opening a rules file about a build
- **WHEN** an agent reads a rules file about a build from the top
- **THEN** before the first rule it learns the build tool, every script, plugin or configuration file the repository must have, such as a launcher script at its root, and that the rules still apply when any of them is missing

#### Scenario: A rule conflicts with the repository
- **WHEN** a rule contradicts a convention the repository's code already follows
- **THEN** the agent knows from the opening lines which one to follow, without reading further

#### Scenario: A rule relies on a library the module lacks
- **WHEN** the agent reaches a rule that needs a library the current module does not declare
- **THEN** it already knows from the opening lines that it asks for the library, by organisation and artifact, and does not substitute another form

#### Scenario: A rule relies on something the repository must have
- **WHEN** the agent reaches a rule in a rules file about a build that needs a script, plugin or configuration file the repository lacks
- **THEN** it already knows from the opening lines that it stops and reports the missing item rather than supplying it

## ADDED Requirements

### Requirement: A rules file that mirrors a skill carries the skill's rules
A rules file that is an edition of a skill's `SKILL.md` body SHALL bear the skill's name, SHALL state every rule the body states, in its words, and SHALL differ from the body only where the packaging requires: no frontmatter, a scope-and-precedence preamble after its opening paragraph, and no pointer to the skill's supporting files or to the skill itself. A change that edits a rule in either edition SHALL make the same edit in the other, and the README SHALL name the skill the file mirrors.

#### Scenario: Comparing the launcher editions
- **WHEN** `rules/sbt-launcher.md` is compared line by line with the body of `skills/sbt-launcher/SKILL.md`, the text after its frontmatter and the blank line that follows it
- **THEN** they differ only in the preamble paragraph that stands where the body pointed at `references/sbt-launcher.md`, in the two sentences that pointed at its flag table, and in the words that named the skill, and every other line is identical

#### Scenario: A rule changes in one edition
- **WHEN** a change edits, adds or removes a rule in `skills/sbt-launcher/SKILL.md` or in `rules/sbt-launcher.md`
- **THEN** the same change makes the same edit in the other file, and the comparison above still holds afterwards

#### Scenario: The mirror is named in the README
- **WHEN** a reader finds `sbt-launcher` in the README's Rules files table
- **THEN** its row says that the file mirrors the `sbt-launcher` skill, and the install step says that a project may carry the rules file while an engineer also has the skill installed

#### Scenario: Both editions in one session
- **WHEN** a project has `rules/sbt-launcher.md` installed and the agent has the `sbt-launcher` skill installed and loads it
- **THEN** the agent meets the same rules twice, with no rule in one edition contradicting the other
