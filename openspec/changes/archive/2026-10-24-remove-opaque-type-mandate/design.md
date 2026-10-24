# Design

## Context

See proposal.md for motivation. The facts that shape the approach, taken from the rules file, the three Scala specs and the archived designs:

- `rules/virtuslab-scala-style.md` is 113 lines in five sections with three fenced examples. The archived designs keep it between 100 and 150 lines with at most three examples and one closing attribution line. Its preamble says the modelling rules apply regardless of the repository's habits and that only the syntax rules yield to them.
- Opaque types are named in seven lines of the rules file: the Scala 2.13 equivalence bullet, the companion bullet, the bullet the request quotes, the `opaque type Quantity = Int` line of the smart-constructor example, the phantom-typed identifier bullet, the proof-type bullet and the boundary bullet. In the main specs they are named in `scala-domain-modelling` (four requirements), in the companion requirement of `scala-code-organisation` and in the Scala 2.13 requirement of `scala-code-style`.
- The request names one bullet and "the requirement to always model data with opaque types". Which other rules that names is the one ambiguity; D1 settles it.
- Nothing in this repository is executable; verification is by text checks and review, as in every earlier change.

## Goals / Non-Goals

**Goals:**

- No remaining requirement or bullet makes a value an opaque type for being an identifier, a quantity or a setting.
- Every remaining mention of an opaque type, in the rules file and in the specs, applies once a rule that still stands has called for one, so that the file reads consistently.
- The rules file keeps its shape: preamble, five sections, three examples, line count within the archived bounds, attribution line.

**Non-Goals:**

- Removing opaque types from the file altogether: they remain the Scala 3 form of a proof type and one valid form of a validated value.
- Any edit to `scala-code-organisation`, `scala-code-style`, the README or the skills.
- Telling consumers to unwrap the types their code already declares.

## Decisions

### D1. The unconditional mandates go; the conditional uses stay

The request's second sentence is read as the general form of its first: withdraw every rule that makes data an opaque type for what it is, an identifier, a quantity or a setting, not every rule that names one. The test applied to each of the seven mentions: a rule that applies to all values of a kind goes; a rule that applies on a condition that justifies a type of its own, or that governs an opaque type once another rule has called for one, stays.

| Mention | Verdict | Ground |
| --- | --- | --- |
| "NEVER pass identifiers, quantities or configuration values ..." | removed | the mandate itself |
| "Give identifiers of different entities distinct types ..." | removed | the mandate for identifiers (D2) |
| "... data transfer objects carry the domain's opaque types instead of primitives" and "raw strings and numbers stop at the adapter" | rewritten in the boundary bullet | the mandate at the boundary (D3) |
| `opaque type Quantity = Int` in the smart-constructor example | stays | an invariant justifies the type (D4) |
| "... give the result its own type, an opaque subtype on Scala 3 ..." | stays | conditional on a transformation to record |
| "Keep ... its opaque type's accessors ... in the type's companion" | stays | governs an opaque type once declared |
| "... a value class or a tagged type for an opaque type ..." | stays | an equivalence, still needed by the rules that remain |

Alternatives: remove only the quoted bullet (rejected: the identifier bullet would keep the identifier case in force and the boundary bullet the rest, so the removal would change little); remove every rule that names an opaque type (rejected: the proof-type rule rests on a transformation and the example on an invariant, neither of which the request names, and the companion and equivalence rules say nothing about when to declare one). The proof-type rule is the closest call and is kept; withdrawing it is a one-bullet, one-requirement follow-up if the user wants it.

### D2. "Identifiers are phantom-typed" is removed rather than made conditional

A conditional form, "when identifiers get types of their own, make them distinct", has no content: two declared types are already distinct, and the choice between `Id[Order]` and `OrderId` belongs to the repository. The requirement and its bullet are removed whole, with the migration note that declared identifier types stay in use.

### D3. The boundary requirement is modified, not removed

What survives: conversion at the adapter, and domain code that never imports a generated package or names another system's type. What goes: "raw strings and numbers stop at the adapter", which is the mandate applied at the boundary, since a `String` id in a generated class now legitimately becomes a `String` id in the domain. The generated-class scenario is reworded so that the adapter gives each field the type the domain declares for it. The data-transfer-object scenario, upstream's "keep opaque types in DTOs", keeps its name and is rewritten rather than dropped: `openspec validate` refuses a MODIFIED block that omits a scenario the main spec has, since archive replaces the whole block. Its new outcome is the one the Chimney requirement already implies: the mapping at the adapter gives an identifier or a quantity the type the domain declares for it, whether the DTO carried that type or the primitive its wire format uses. So a DTO may carry the wire primitive, and the bullet says so. The delta carries the whole requirement under MODIFIED, as archive needs.

### D4. The `Quantity` example stays

The example illustrates the smart-constructor rule, and the positivity invariant justifies the type, not the `Int` it wraps. An opaque type remains the lightest Scala 3 form of a validated value, and the rule it illustrates names no form. Replacing it with a case class would change a rule the request does not touch and could be read as a new mandate.

### D5. "Typed values" in the configuration rule and the Chimney scenario stay as written

"A case class of typed values" means that each setting has the type the module reads it as, a `Duration`, an `Int`, a masked secret; with the "configuration value" scenario gone, nothing requires a `Port`, and the text needs no edit. The Chimney scenario's condition, fields that are opaque types with smart constructors, still describes a case that occurs and is left unedited, to keep the change to its subject.

### D6. Edit form

Two bullets deleted and one rewritten, each in place; nothing reordered. The rewritten bullet keeps its position in "Domain modelling", after the proof-type bullet and before the Chimney bullet. The file ends at 111 lines, with five sections, six fence lines and the attribution line.

## Risks / Trade-offs

- [An agent still reaches for opaque types by habit, from the `Quantity` example and the proof-type bullet] → both remaining bullets name their condition, an invariant and a transformation, and no bullet instructs wrapping without one.
- [In a repository written under the mandate, an agent adds a plain `String` identifier beside an `OrderId`] → the migration notes say declared types stay in use; if this is observed, a later change can add a rule that matches the repository's identifier types.
- [The proof-type rule is read as the mandate in disguise] → D1 records the test applied and names the follow-up that withdraws it.
