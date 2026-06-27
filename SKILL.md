---
name: filemaker-field-xml
description: Use this skill whenever the user wants to work with FileMaker field definition XML. This includes generating paste-ready field definitions (fmxmlsnippet type FMObjectList containing Field elements) for Manage Database, reviewing field XML for silent paste-handler failures, or analysing field definitions from DDR or Save as XML exports. Trigger any time the user mentions FileMaker fields, field definitions, Manage Database paste, auto-enter, field validation, calculation fields, summary fields, or schema scaffolding. Do not attempt FileMaker field XML from memory alone.
---

# FileMaker Field Definition XML Skill

This skill gives Claude a deterministic, empirically verified foundation for generating FileMaker field definition XML — the `fmxmlsnippet type="FMObjectList"` clipboard format containing `<Field>` elements, accepted by FileMaker's Manage Database paste handler.

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## What this skill does

When this skill is active, Claude will:

- Generate paste-ready field definition XML from plain descriptions ("create a UUID primary key, audit stamps, and a status field with a value list")
- Review existing field XML for silent-failure risks before you paste it
- Analyse field definitions from DDR or Save-as-XML exports
- Generate correct data types, auto-enter mechanisms, validation options, and storage settings — without guessing

## Critical rules (enforced before generating)

1. **Unique field IDs.** Assign unique sequential `id` values across all fields in a single paste. Duplicate IDs cause silent drops when calculation auto-enter fields reference each other.
2. **Value list references.** The `<ValueList>` child element is only preserved when the referenced value list ID exists in the target file at paste time. When the target file is unknown, omit the value list and tell the user to assign it manually after pasting.
3. **Furigana dependency.** FileMaker drops the `<Furigana>` element on paste when the field's `<ValueList>` reference does not resolve. Ensure the value list ID is real if Furigana is required.
4. **No XML comments in field paste.** The Manage Database field paste handler fails or pastes only partially when the snippet contains `<!-- -->` comments (unlike the script paste handler, which tolerates them). Generated field XML must be comment-free. Document intent with descriptive field names and `<Comment>` elements instead.
5. **FileMaker 2026 elements (verified).** FileMaker 2026 (v26) adds `<Annotation>` (a `<Text>` child holding plain text, read by `FieldAnnotation()`) and `<DisplayNames enable="...">` (when enabled, a `<Calculation>` returning JSON keyed by `fm_common`/`fm_export`/`fm_sort`/`fm_table_view` plus optional custom keys, read by `FieldDisplayNames()`). Both are round-trip confirmed on Normal, Calculation, and Summary fields (see §14). Omit both for 2025 or mixed targets; emit both for 2026. Note that annotating any field narrows that table's generated DDL to annotated fields only. The 2026 calculation-controlled field entry (read-only via calculation) is a layout object property, not a field definition, and is out of scope here.

## Specification reference

The full specification is in `references/filemaker_xmfd_spec.md` (v1.0), covering:

- All six data types: Text, Number, Date, Time, Timestamp, Container
- All three field types: Normal, Calculation, Summary
- Every auto-enter mechanism — system values, serial, constant data, calculation, lookup — including valid coexistence combinations
- All validation options: strict data type, range, max length, value list, calculation validation, custom error messages
- Storage variants: index levels, global fields, repeating fields, external container storage (Open and Secure)
- Summary operations: Total, Average, Count, Maximum, List
- Ready-to-paste templates for UUID primary keys, audit stamps, and common field patterns

Claude reads this automatically when handling field definition XML tasks. You do not need to reference it in your prompts.

## Usage

**Generate field definitions:**
> "Generate XML for a Contacts table: UUID primary key, first name, last name, email with validation, and creation/modification audit stamps"

**Review existing XML:**
> Paste your fmxmlsnippet and ask: "Check this field XML for paste-handler errors"

**With a DDR:**
> Attach your DDR export and Claude will use real value list IDs and existing field names from your solution.

## Pasting into FileMaker

1. Open the generated XML in a text editor, select all, copy
2. In FileMaker Pro, open Manage Database and select the target table on the Fields tab
3. Paste — FileMaker creates the fields and assigns real internal IDs

**Requires MBS Plugin to be installed in FileMaker Pro.** No MBS scripting is needed — the plugin simply needs to be present.

## What this skill does not cover

- Script step XML — see the companion FileMaker Script XML Skill
- Layout object XML — see the companion FileMaker Layout XML Skill
- Table, relationship, or value list creation — the paste handler accepts field definitions only
