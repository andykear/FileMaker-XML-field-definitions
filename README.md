# FileMaker Field Definition XML Specification

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

---

## How it works

FileMaker's Script Workspace accepts XML paste — the `fmxmlsnippet type="FMObjectList"` clipboard format that developers use to share and generate scripts.

The same envelope works for field definitions. Paste a correctly structured snippet into Manage Database with a table selected and FileMaker silently creates every field exactly as specified.

Claris has never documented the format. The paste handler is strict in ways the XML parser is not — wrong structure produces fields with incorrect settings or silently drops elements with no error message.

This specification was built through systematic round-trip testing against production FileMaker 2025 solutions: generate XML, paste, export DDR, diff, iterate. A 74-field validation suite covering every documented variant was used to confirm the spec before publication.

---

## What this means in practice

Describe the fields you need to an AI model with this spec as context, or write the XML directly. Then:

1. Open the generated XML in a text editor, select all, copy
2. In FileMaker Pro, open Manage Database and select the target table on the Fields tab
3. Paste

---

## Pasting into FileMaker

Layout mode requires the `fmxmlsnippet type="LayoutObjectList"` format on the clipboard in FileMaker's internal clipboard format — not plain text. This skill has been tested with the **MBS Plugin** installed. Plugin-free clipboard conversion options are available in the FileMaker community and should work with this format, but have not been tested by Clockwork.

---

## Companion skill

This skill covers field objects.
There are 3 companion skills covering Scripts, Field and Layout plus an XML inspector app.

[FileMaker Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill) — script steps for the Script Workspace

[FileMaker Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill) — layout objects for Layout mode

[FileMaker Field Definitions XML Skill ](https://github.com/andykear/FileMaker-XML-field-definitions) — field definitions for Manage Database

[FileMaker XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source) - Browser based XML Inspector.


---

## What the spec covers

- All six data types: Text, Number, Date, Time, Timestamp, Container
- All three field types: Normal, Calculation, Summary
- Every auto-enter mechanism — system values, serial, constant data, calculation, lookup — including all valid coexistence combinations
- All validation options: strict data type, range, max length, value list, calculation validation, custom error messages
- Storage variants: all index levels, global fields, repeating fields, external container storage (Open and Secure)
- Summary operations: Total, Average, Count, Maximum, List
- Silent failure modes: duplicate IDs, unresolved value list references, Furigana dependency
- Ready-to-paste templates for UUID primary keys, audit stamps, and common field patterns

---

## Files

```
SKILL.md                              Claude skill definition
README.md                             This file
references/
  filemaker_xmfd_spec.md              Full specification (v0.6)
```

---

## Related

[FileMaker Script XML Skill for Claude](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill) — the companion reverse-engineered specification for Script Workspace clipboard XML. Same envelope, same methodology, same community ethos.

---

## Contributing

Found something that doesn't round-trip? A production export that contradicts the spec? Open an issue or PR. The spec improves through community testing — that's how it was built.

---

## Licence

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — free to use, share, and adapt with attribution.

---

*Clockwork Creative Technology — clockworkct.co.uk*
