# FileMaker Field Definition XML Specification

[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-XML-field-definitions?style=social)](https://github.com/andykear/FileMaker-XML-field-definitions)
[![Last commit](https://img.shields.io/github/last-commit/andykear/FileMaker-XML-field-definitions)](https://github.com/andykear/FileMaker-XML-field-definitions)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

Reverse-engineered specification of FileMaker's undocumented field definition XML format. Covers field types, auto-enter, validation, calculation and summary fields, verified against FileMaker's own exports rather than inferred.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

---

## How it works

FileMaker's Script Workspace accepts XML paste — the `fmxmlsnippet type="FMObjectList"` clipboard format that developers use to share and generate scripts.

The same envelope works for field definitions. Paste a correctly structured snippet into Manage Database with a table selected and FileMaker silently creates every field exactly as specified.

Claris has never documented the format. The paste handler is strict in ways the XML parser is not — wrong structure produces fields with incorrect settings or silently drops elements with no error or warning.

This specification was built through systematic round-trip testing: generate XML, paste, re-export, diff, iterate. Originally validated against FileMaker 2025 (v22) with a 74-field suite, then extended to FileMaker 2026 (v26) covering the new annotation and display-name elements and re-confirming the existing format.

---

## FileMaker 2026 (v26) status

FileMaker 2026 (internal version 26, released 10 June 2026) adds two field-level elements: `<Annotation>`, a plain-text AI/DDL description read by `FieldAnnotation()`, and `<DisplayNames>`, custom field display names read by `FieldDisplayNames()`. Both are round-trip confirmed on v26 and fully documented in §14. The highlights below cover what's surprising about them.

Not in this spec: the 2026 calculation-controlled field entry (editable / non-editable / read-only) is a **layout object** property, not a field definition, and is covered by the Layout XML spec.

---

## Using it

Describe the fields you need to an AI model with this spec as context, or write the XML directly. Then select all, copy, and in FileMaker Pro open Manage Database, select the target table on the Fields tab, and paste. FileMaker creates the fields and assigns real internal IDs.

The snippet must be on the clipboard in FileMaker's internal `fmxmlsnippet type="FMObjectList"` format, not plain text, which is what the MBS Plugin provides (see Requirements).

---

## Specification highlights

A few things round-trip testing turned up that no documentation tells you:

- **Display names are a calculation, not a string.** FileMaker 2026's custom field display names are stored as a calculation returning a JSON object, keyed by context (`fm_common`, `fm_export`, `fm_sort`, `fm_table_view`, plus your own keys). They can be dynamic. Every secondary write-up called them a static label.
- **Field annotations silently change DDL scope.** Annotate one field in a table and the AI-facing DDL now excludes every *un*annotated field. A one-field edit reshapes what the whole table exposes to an LLM.
- **Summary sub-options are distinct operation strings, not flags.** "By population", "running", "weighted", "subtotalled" each get their own `operation` value (`StdDeviationByPopulation`, `RunningCount`, `WeightedAverage`, `FractionalSubtotal`), not a checkbox attribute.
- **Comments break the field paste handler** — even though the *script* paste handler tolerates them. Same envelope, different parser.
- **Unique validation forces an index.** Paste a unique field with `index="None"` and FileMaker quietly upgrades it to `Minimal`.
- **`<MaxDataLength>` is overloaded** — characters on a text field, kilobytes on a container. The `dataType` decides the unit.

Plus the unglamorous-but-essential silent failure modes: duplicate IDs, unresolved value-list references, the Furigana/value-list dependency.

## What's covered

All six data types, all three field types (Normal, Calculation, Summary), every auto-enter mechanism (system values, serial, constant, calculation, lookup) with valid coexistence combinations, every validation option (including calculated custom messages), all storage variants (index levels, global, repeating, external container Open/Secure), all 13 summary operations, the FileMaker 2026 annotation and display-name elements, and ready-to-paste templates for UUID keys and audit stamps.

---

## Requirements

Claude (or any capable model with the spec in context) and the **MBS Plugin** present in FileMaker Pro. No MBS scripting needed, the plugin just has to be installed. Tested with the MBS Plugin in FileMaker 2024, 2025, and 2026.

---

## Files

```
SKILL.md                              Claude skill definition
README.md                             This file
references/
  filemaker_xmfd_spec.md              Full specification (v1.0)
```

---

## Version history

| Version | Notes |
|---|---|
| 1.0 | FileMaker 2026 (v26) verified. Annotation and display-name elements (display names confirmed as a JSON-returning calculation), full 13-operation summary set, calculated validation messages, container external storage, lookup, and Furigana all round-trip confirmed on v26. Comment-handler and unique-index behaviours documented. |
| 0.6 | FileMaker 2025 (v22) baseline. All field types, auto-enter, validation, storage, and a 74-field validation suite. |

---

## Companion repos

Five open-source resources for the FileMaker/Claris community:

[FileMaker Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill) — script steps for the Script Workspace

[FileMaker Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill) — layout objects for Layout mode

[FileMaker Field Definitions XML Skill](https://github.com/andykear/FileMaker-XML-field-definitions) — field definitions for Manage Database

[FileMaker XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source) — browser-based Save as XML analyser

[FileMaker XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber) — redacts credentials before sharing with AI tools

---

## Contributing

Found something that doesn't round-trip? A production export that contradicts the spec? Open an issue or PR. The spec improves through community testing, that's how it was built.

---

## Licence

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — free to use, share, and adapt with attribution.

---

*Clockwork Creative Technology — clockworkct.co.uk. Bespoke FileMaker development, automated artwork systems, and hosted solutions. Working on something and need a hand? Get in touch.*
