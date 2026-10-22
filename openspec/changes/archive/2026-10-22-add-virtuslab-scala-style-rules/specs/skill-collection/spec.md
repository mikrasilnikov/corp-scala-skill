# Spec Delta

## MODIFIED Requirements

### Requirement: The root README lists every skill
The repository root SHALL contain a `README.md` that states that the repository is a collection of skills and rules files, lists every directory under `skills/` by name with a one-line summary, tells the reader how to install a skill (copy `skills/<name>/` into the agent's skills directory), and states the convention a new skill must follow. The rules files are listed as `rules-collection` specifies.

#### Scenario: Finding the skills
- **WHEN** a reader opens `README.md`
- **THEN** they see every skill under `skills/` listed by name with a summary, and no skill present in `skills/` is missing from the list

#### Scenario: A skill is added
- **WHEN** a new directory is added under `skills/`
- **THEN** `README.md` gains an entry for it in the same change

#### Scenario: Installing from the README
- **WHEN** a reader follows the README's install instruction for `sbt-launcher`
- **THEN** they copy `skills/sbt-launcher/` into their agent's skills directory and the installed directory is named `sbt-launcher`

#### Scenario: Skills and rules files are told apart
- **WHEN** a reader opens `README.md`
- **THEN** its opening says that skills are installed as directories and loaded on demand, and rules files are loaded whole into the agent's context, and the two are listed in separate sections
