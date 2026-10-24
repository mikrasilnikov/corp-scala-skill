# Proposal

## Why

`rules/virtuslab-scala-style.md` makes an agent wrap every identifier, quantity and configuration value in an opaque type (a value class or tagged type on Scala 2.13) whatever the value's meaning, and two further rules carry the same mandate under other names: one gives every entity's identifier its own distinct opaque type, and one keeps opaque types in data transfer objects and stops raw strings and numbers at the adapter. The result is a wrapper, an accessor and a conversion for values that carry no invariant, and a disagreement with any repository whose code passes such values as plain primitives, which the preamble's precedence clause does not resolve, since it covers syntax rules only and says the modelling rules apply regardless. The mandate is withdrawn. A type of its own stays reserved for a value that has an invariant, a closed set of values, a transformation to record, or a Boolean whose meaning a call site cannot read.

## What Changes

- The requirement "Domain concepts are not raw primitives" is removed from `scala-domain-modelling`, with its three scenarios: the identifier parameter, the configuration value, and two identifiers swapped. The agent no longer wraps an identifier, a quantity or a setting for being a `String`, `Int`, `Long` or `Boolean`.
- The requirement "Identifiers are phantom-typed" is removed from `scala-domain-modelling`, with its two scenarios. It mandated one opaque type per entity identifier, or one parameterised `Id[Entity]`, which is the identifier case of the removed requirement; left in place, it would keep that case in force.
- The requirement "Generated and external types stop at the boundary" is modified: it keeps the conversion at the adapter and the ban on domain code importing a generated package, loses the clause that raw strings and numbers stop at the adapter, and restates its data-transfer-object scenario without the mandate: the mapping at the adapter gives an identifier or a quantity the type the domain declares for it, whether the DTO carried that type or the primitive its wire format uses.
- `rules/virtuslab-scala-style.md` loses two bullets under "Domain modelling", the one that begins "NEVER pass identifiers, quantities or configuration values" and the one that begins "Give identifiers of different entities distinct types", and its boundary bullet, the one that begins "Convert types generated from schemas", is rewritten to the conversion at the adapter, the import ban, and a data transfer object that may carry a value as the primitive its wire format uses. No other line of the file changes.
- These stay, on purpose, because each applies on a condition that justifies a type of its own rather than to all data of a kind: the validated constructor for a value with an invariant and its `Quantity` example, which the positivity invariant justifies, not the `Int` it wraps; the two-case enum for a Boolean a call site cannot read; the enum for a closed set; the proof type for a value that must have been transformed before use; the companion that holds an opaque type's accessors; the Scala 2.13 equivalent of an opaque type; and the Chimney scenario for fields that are opaque types with smart constructors. After this change the agent declares an opaque type only where one of those rules asks for it, and otherwise gives a value the type the surrounding code uses for it, a plain `String`, `Int`, `Long` or `Boolean` included.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `scala-domain-modelling`: the requirements "Domain concepts are not raw primitives" and "Identifiers are phantom-typed" are removed; the requirement "Generated and external types stop at the boundary" is modified to drop the clause that raw strings and numbers stop at the adapter and to restate its data-transfer-object scenario without the mandate. No requirement is added.

## Impact

- `rules/virtuslab-scala-style.md`: two bullets deleted and one rewritten, taking the file from 113 to 111 lines; it keeps its preamble, its five sections, its three code examples and its closing attribution, and stays within the 100 to 150 lines the archived designs set for it.
- `openspec/specs/scala-domain-modelling/spec.md`: when the change is archived, two requirements leave the main spec and one is replaced, taking it from fifteen requirements to thirteen; the Purpose paragraph still holds, since enums, `Option` and validated constructors still make invalid data unrepresentable.
- `openspec/specs/scala-code-organisation/spec.md` and `openspec/specs/scala-code-style/spec.md`: unchanged. Their mentions of opaque types, the companion that holds an opaque type's accessors and the Scala 2.13 equivalent of an opaque type, apply once one is declared, and the proof-type rule and the `Quantity` example still declare one.
- `README.md`, `skills/upickle/` and `skills/sbt-launcher/`: unchanged. Neither the README's row for the rules file nor either skill mentions opaque types or the removed rules.
- Projects that installed the rules file pick the change up when they copy the file again; no dependency or compiler option changes. Code already written under the mandate keeps its wrapper types: the rules do not ask for them to be unwrapped, and the agent keeps using the types the surrounding code declares.
- The archived changes that introduced and revised these rules keep their text, as history.
