# FileMaker Field and Table Definition XML Specification

[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-XML-field-definitions?style=social)](https://github.com/andykear/FileMaker-XML-field-definitions)
[![Last commit](https://img.shields.io/github/last-commit/andykear/FileMaker-XML-field-definitions)](https://github.com/andykear/FileMaker-XML-field-definitions)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

Reverse engineered specification of FileMaker's undocumented field and table definition XML format. Covers whole table paste, all field types, auto enter, validation, calculation and summary fields. Verified against FileMaker's own behaviour rather than inferred.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

---

## How it works

FileMaker's `fmxmlsnippet type="FMObjectList"` clipboard format is best known for script paste. The same envelope works in Manage Database.

Paste a field snippet with a table selected on the Fields tab and FileMaker creates every field exactly as specified. Paste a `<BaseTable>` snippet on the Tables tab and FileMaker creates the complete table, fields included, in one operation. A snippet containing several tables creates all of them in a single paste, each with its own table occurrence already on the relationships graph. An entire schema scaffold is one paste.

Claris has never documented the format. The paste handlers are strict in ways the XML parser is not. Wrong structure produces fields with incorrect settings, silently drops elements, or quietly disables a calculation, all with no error and no warning.

This specification was built through systematic round trip testing: generate the XML, paste it, export it back out, diff the two, repeat until they match. The v22 baseline used a 74 field validation suite. The v26 work added the annotation and display name elements. The v2.0 table work closed the loop with a generated 12 field table that came back byte identical, 151 elements, zero drift, then pushed every failure mode until the handler's behaviour was pinned down.

---

## Using it

Describe the fields or tables you need to an AI model with this spec as context, or write the XML directly. Copy it, open Manage Database in FileMaker Pro, and paste: on the Fields tab for field snippets with the target table selected, or on the Tables tab for whole tables. FileMaker creates everything and assigns real internal IDs.

The snippet must be on the clipboard in FileMaker's internal format, not plain text, which is what the MBS Plugin provides (see Requirements).

---

## Specification highlights

Things round trip testing turned up that no documentation tells you:

- **Whole tables paste at full fidelity.** A `<BaseTable>` wrapper around the standard field format creates a complete table from the Tables tab. Every auto enter mechanism, validation option, and storage setting survives, and each pasted table gets its own occurrence on the graph.
- **Name collisions rename and rebind.** Paste a table whose name already exists and FileMaker silently creates "contacts 2", then rewrites the calculation context inside it to the new name. Same on the Fields tab: colliding fields get the suffix, and calculations and summaries within the pasted set are rewritten to follow the renamed copies rather than binding back to the originals.
- **Invalid calculations are neutralised, not rejected.** A calculation the handler cannot parse pastes anyway, with its entire body wrapped in a calculation comment. The field looks normal in the field list and its calculation does nothing. This is the sneakiest silent failure in the format.
- **Three handlers, three comment policies.** The script paste handler tolerates XML comments, the field handler fails or pastes partially, and the table handler strips them and carries on. Same envelope, three parsers.
- **Forward references resolve.** A calculation can reference a field defined later in the snippet. Names resolve after all fields exist, so generated XML needs no dependency ordering.
- **Serial next values travel.** The `nextValue` in pasted XML becomes the live counter. A migration feature and a generation footgun in one attribute.
- **Display names are a calculation, not a string.** FileMaker 2026's custom field display names are stored as a calculation returning a JSON object keyed by context. They can be dynamic. Every secondary write up called them a static label.
- **Field annotations silently change DDL scope.** Annotate one field in a table and the AI facing DDL now excludes every field without an annotation.
- **Summary sub options are distinct operation strings, not flags.** "By population", "running", "weighted", and "subtotalled" each get their own `operation` value.
- **Unique validation forces an index.** Paste a unique field with `index="None"` and FileMaker quietly upgrades it to `Minimal`.
- **`<MaxDataLength>` is overloaded.** Characters on a text field, kilobytes on a container. The `dataType` decides the unit.

Plus the unglamorous but essential silent failure modes: duplicate IDs, unresolved value list references, the Furigana and value list dependency, and the `<Field Missing>` tokens FileMaker writes into exports where a referenced field was deleted.

---

## What's covered

All six data types, all three field types (Normal, Calculation, Summary), every auto enter mechanism (system values, serial, constant, calculation, lookup) with valid coexistence combinations, every validation option including calculated custom messages, all storage variants (index levels, global, repeating, external container Open and Secure), all 13 summary operations, the FileMaker 2026 annotation and display name elements, table level paste via the `<BaseTable>` envelope, and templates for UUID keys and audit stamps ready to paste.

Analysis is covered as well as generation: the spec documents the fossils that production exports preserve, from dangling value list references to disabled lookups with their child elements intact, so exported schemas can be read accurately, not just written.

---

## Requirements

Claude (or any capable model with the spec in context) and the **MBS Plugin** present in FileMaker Pro. No MBS scripting needed, the plugin just has to be installed. Tested with the MBS Plugin in FileMaker 2024, 2025, and 2026.

---

## Files

```
SKILL.md                              Claude skill definition
README.md                             This file
references/
  filemaker_xmfd_spec.md              Full specification (v2.0)
```

---

## Version history

| Version | Notes |
|---|---|
| 2.0 | Table level paste. The `<BaseTable>` envelope round trip verified on v26, byte identical: whole tables and multi table schemas paste from the Tables tab at full fidelity, each with a graph occurrence. Forward references resolve, serial nextValue honoured, per table ID scope, collision rename with context and reference rebinding, invalid calculation neutralisation documented. §5.2 corrected with the CreationName and ModificationName values. Calculation table attribute documented as the evaluation context occurrence. Copy side fossil catalogue for analysing production exports. |
| 1.0 | FileMaker 2026 (v26) verified. Annotation and display name elements (display names confirmed as a JSON returning calculation), full 13 operation summary set, calculated validation messages, container external storage, lookup, and Furigana all round trip confirmed on v26. Comment handler and unique index behaviours documented. |
| 0.6 | FileMaker 2025 (v22) baseline. All field types, auto enter, validation, storage, and a 74 field validation suite. |

---

## Companion repos

Five open source resources for the FileMaker/Claris community:

[FileMaker Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill) covers script steps for the Script Workspace

[FileMaker Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill) covers layout objects for Layout mode

[FileMaker Field and Table Definitions XML Skill](https://github.com/andykear/FileMaker-XML-field-definitions) covers schema for Manage Database

[FileMaker XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source) is a browser based Save as XML analyser

[FileMaker XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber) redacts credentials before sharing with AI tools

---

## Contributing

Found something that doesn't round trip? A production export that contradicts the spec? Open an issue or PR. The spec improves through community testing, that's how it was built. The v2.0 table findings came from exactly that: production exports tested against generated XML until every behaviour was accounted for.

---

## Licence

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), free to use, share, and adapt with attribution.

---

*Clockwork Creative Technology, clockworkct.co.uk. Bespoke FileMaker development, automated artwork systems, and hosted solutions. Working on something and need a hand? Get in touch.*
