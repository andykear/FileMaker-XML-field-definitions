---
name: filemaker-field-xml
description: Use this skill whenever the user wants to work with FileMaker schema definition XML — field definitions, table definitions, or value list definitions. This includes generating paste-ready field definitions (fmxmlsnippet type FMObjectList containing Field elements) for the Fields tab, complete tables (BaseTable elements) for the Tables tab, or value list definitions (ValueList elements) for Manage Value Lists; reviewing any of these for silent paste-handler failures; or analysing them from Save as XML (SaXML) exports. Trigger any time the user mentions FileMaker fields, tables, value lists, Manage Database paste, BaseTable, ValueList, auto-enter, field validation, calculation fields, summary fields, or schema scaffolding. Do not attempt FileMaker schema XML from memory alone.
---

# FileMaker Schema Definition XML Skill

A deterministic, empirically verified foundation for generating and analysing FileMaker's three schema-definition XML catalogs, all carried in the `fmxmlsnippet type="FMObjectList"` clipboard format:

- **Field definitions** — `<Field>` elements, pasted on the **Fields tab** of Manage Database.
- **Table definitions** — `<BaseTable>` elements wrapping complete tables, pasted on the **Tables tab**.
- **Value list definitions** — `<ValueList>` elements, pasted into **Manage Value Lists**.

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community. Licence: CC BY 4.0.

## Critical rules (enforced before generating)

1. **Unique field IDs, scoped per table.** Assign unique sequential `id` values (from 1) within each table. Duplicate IDs within a table cause silent drops when calculation auto-enter fields reference each other. In multi-table snippets, sibling `<BaseTable>` elements each restart at `id="1"` — that is FileMaker's own format, not an error.
2. **Field→value-list references.** The `<ValueList>` child element inside a field's validation is only preserved when the referenced value list ID exists in the target file at paste time. When the target file is unknown, omit it and tell the user to assign the value list manually after pasting. (This is the field-side *reference*; the value list *definition* format is a separate catalog — see the value list spec.)
3. **Furigana dependency.** FileMaker drops the `<Furigana>` element on paste when the field's `<ValueList>` reference does not resolve. Ensure the value list ID is real if Furigana is required.
4. **No XML comments in field paste.** The Fields tab handler fails or pastes only partially when the snippet contains `<!-- -->` comments (unlike the script and Tables tab handlers, which tolerate them). Generate comment-free regardless. Document intent with descriptive field names and `<Comment>` elements.
5. **FileMaker 2026 elements.** v26 adds `<Annotation>` and `<DisplayNames>`. Emit both only for v26 targets; omit for 2025 or mixed. Annotating any field narrows that table's generated DDL to annotated fields only. Structure and JSON keys: field spec §14.
6. **Table envelope.** Wrap fields in `<BaseTable comment="" name="...">` to paste a whole table on the Tables tab; the field format inside is identical to the bare-field format. Forward references resolve, so no dependency ordering. Collisions are safe: silent rename ("contacts 2") with calculation context rebound. Full rules in the table spec.
7. **Serial `nextValue` is honoured on paste.** Whatever value the XML carries becomes the live next serial value. Default it to 1 unless the user specifies otherwise — never copy a nextValue from an example.
8. **Never emit `<Field Missing>` or `<Table Missing>`.** These literal tokens appear in production exports where referenced fields or table occurrences were deleted. They are broken-reference fossils, not valid syntax. When analysing exports, flag them; when generating, they must never appear.
9. **Value list references resolve by name; the id is advisory.** For a Field-source value list, the field **name** must resolve in the target — FileMaker rewrites the supplied `id` on paste, and an unresolvable field name installs a *silent* dead `<Field Missing>` binding. An **External** value list behaves differently: if its data source is not present in the target, paste throws a **blocking file-open error dialog**, then installs a live dangle — so never emit an External arm whose data source isn't guaranteed present. The value list's own `id` is **always reassigned** on paste, so nothing may rely on a value list id surviving.
10. **Emit only the active value list arm.** A `<ValueList>` serialises the union of every source it has been configured as (Custom / Field / External); the inactive arms are invisible fossils. Generate only the arm named by `<Source value>` — paste *installs* fossils, it does not ignore them.
11. **Value list "display both fields".** To display the first and second field together, emit `PrimaryField show="True"` with `SecondaryField show="False"`. `SecondaryField show="True"` means *show only the second field*; sending both `show="True"` is contradictory and FileMaker silently rewrites it to second-only. See value list spec §3.2.

## Specification reference

One spec per catalog in `references/`, read automatically per task:

- `filemaker_xmfd_spec.md` — field definitions
- `filemaker_xmtb_spec.md` — table definitions
- `filemaker_xmvl_spec.md` — value list definitions

## What this skill does not cover

- Script step XML — see the companion FileMaker Script XML Skill
- Layout object XML — see the companion FileMaker Layout XML Skill
- The relationships graph — a pasted table arrives with a standalone occurrence, but relationship wiring is manual
