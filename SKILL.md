---
name: filemaker-field-xml
description: Use this skill whenever the user wants to work with FileMaker field definition or table definition XML. This includes generating paste-ready field definitions (fmxmlsnippet type FMObjectList containing Field elements) for Manage Database, generating complete tables with fields (BaseTable elements) for the Tables tab, reviewing field or table XML for silent paste-handler failures, or analysing field definitions from DDR or Save as XML exports. Trigger any time the user mentions FileMaker fields, field definitions, tables, Manage Database paste, BaseTable, auto-enter, field validation, calculation fields, summary fields, or schema scaffolding. Do not attempt FileMaker field or table XML from memory alone.
---

# FileMaker Field and Table Definition XML Skill

This skill gives Claude a deterministic, empirically verified foundation for generating FileMaker field and table definition XML — the `fmxmlsnippet type="FMObjectList"` clipboard format containing `<Field>` elements (Fields tab) or `<BaseTable>` elements wrapping complete tables (Tables tab), accepted by FileMaker's Manage Database paste handlers.

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## What this skill does

When this skill is active, Claude will:

- Generate paste-ready field definition XML from plain descriptions ("create a UUID primary key, audit stamps, and a status field with a value list")
- Generate complete tables — fields included — as `<BaseTable>` snippets that paste on the Tables tab in one operation, including multi-table schema scaffolds
- Review existing field XML for silent-failure risks before you paste it
- Analyse field definitions from DDR or Save-as-XML exports
- Generate correct data types, auto-enter mechanisms, validation options, and storage settings — without guessing

## Critical rules (enforced before generating)

1. **Unique field IDs, scoped per table.** Assign unique sequential `id` values (from 1) within each table. Duplicate IDs within a table cause silent drops when calculation auto-enter fields reference each other. In multi-table snippets, sibling `<BaseTable>` elements each restart at `id="1"` — that is FileMaker's own format, not an error.
2. **Value list references.** The `<ValueList>` child element is only preserved when the referenced value list ID exists in the target file at paste time. When the target file is unknown, omit the value list and tell the user to assign it manually after pasting.
3. **Furigana dependency.** FileMaker drops the `<Furigana>` element on paste when the field's `<ValueList>` reference does not resolve. Ensure the value list ID is real if Furigana is required.
4. **No XML comments in field paste.** The Manage Database field paste handler fails or pastes only partially when the snippet contains `<!-- -->` comments (unlike the script paste handler, which tolerates them). Generated field XML must be comment-free. Document intent with descriptive field names and `<Comment>` elements instead.
5. **FileMaker 2026 elements (verified).** FileMaker 2026 (v26) adds `<Annotation>` (a `<Text>` child holding plain text, read by `FieldAnnotation()`) and `<DisplayNames enable="...">` (when enabled, a `<Calculation>` returning JSON keyed by `fm_common`/`fm_export`/`fm_sort`/`fm_table_view` plus optional custom keys, read by `FieldDisplayNames()`). Both are round-trip confirmed on Normal, Calculation, and Summary fields (see §14). Omit both for 2025 or mixed targets; emit both for 2026. Note that annotating any field narrows that table's generated DDL to annotated fields only. The 2026 calculation-controlled field entry (read-only via calculation) is a layout object property, not a field definition, and is out of scope here.
6. **Table envelope (v2.0).** A `<BaseTable comment="" name="...">` element wrapping the fields pastes on the **Tables tab** and creates the whole table in one operation. The field format inside is identical to the bare-field format — round-trip verified byte-identical on v26. Forward references resolve (a calc may reference a field defined later in the snippet), so no dependency ordering is needed. Name collisions are safe: FileMaker silently renames ("contacts 2") and rewrites calculation context to the new table.
7. **Serial `nextValue` is honoured on paste.** Whatever value the XML carries becomes the live next serial value. Default it to 1 unless the user specifies otherwise — never copy a nextValue from an example.
8. **Never emit `<Field Missing>` or `<Table Missing>`.** These literal tokens appear in production exports where referenced fields or table occurrences were deleted. They are broken-reference fossils, not valid syntax. When analysing exports, flag them; when generating, they must never appear.

## Specification reference

The full specification is in `references/filemaker_xmfd_spec.md` (v2.0), covering:

- All six data types: Text, Number, Date, Time, Timestamp, Container
- All three field types: Normal, Calculation, Summary
- Every auto-enter mechanism — system values, serial, constant data, calculation, lookup — including valid coexistence combinations
- All validation options: strict data type, range, max length, value list, calculation validation, custom error messages
- Storage variants: index levels, global fields, repeating fields, external container storage (Open and Secure)
- Summary operations: Total, Average, Count, Maximum, List
- Ready-to-paste templates for UUID primary keys, audit stamps, and common field patterns
- Table-level paste (§15): the `<BaseTable>` envelope, collision rename behaviour, forward-reference resolution, serial nextValue handling, per-table ID scope, and the copy-side fossil catalogue for analysing production exports

Claude reads this automatically when handling field definition XML tasks. You do not need to reference it in your prompts.

## Usage

**Generate field definitions:**
> "Generate XML for a Contacts table: UUID primary key, first name, last name, email with validation, and creation/modification audit stamps"

**Generate a complete table:**
> "Make me a new invoices table with line items — as a table paste I can drop on the Tables tab"

**Review existing XML:**
> Paste your fmxmlsnippet and ask: "Check this field XML for paste-handler errors"

**With a DDR:**
> Attach your DDR export and Claude will use real value list IDs and existing field names from your solution.

## Pasting into FileMaker

1. Open the generated XML in a text editor, select all, copy
2. In FileMaker Pro, open Manage Database. For field snippets, select the target table on the Fields tab. For `<BaseTable>` snippets, go to the Tables tab
3. Paste — FileMaker creates the fields (or the whole table) and assigns real internal IDs

**Requires MBS Plugin to be installed in FileMaker Pro.** No MBS scripting is needed — the plugin simply needs to be present.

## What this skill does not cover

- Script step XML — see the companion FileMaker Script XML Skill
- Layout object XML — see the companion FileMaker Layout XML Skill
- Relationship or value list creation — tables and fields paste; the graph and value lists do not
