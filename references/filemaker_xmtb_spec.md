# Canonical XML Format for FileMaker Table Definitions (XMTB)

**Author:** Andrew Kear, Clockwork Creative Technology
**Version:** 2.1
**Verified against:** FileMaker Pro 2026 (v26) on macOS, by round-trip (generate XML, paste, re-export, diff). Every behaviour here is round-trip confirmed unless flagged in §8.
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Scope

The `fmxmlsnippet type="FMObjectList"` envelope with `<BaseTable>` elements, pasted on the
**Tables tab** of Manage Database to create complete tables — fields included — in one
operation. The field format *inside* a `<BaseTable>` is identical to the bare-field format
in the field spec (`filemaker_xmfd_spec.md`); this spec covers the table wrapper and the
Tables-tab handler's behaviour. Value list definitions have their own spec
(`filemaker_xmvl_spec.md`). Setup and paste mechanics are in the README.

Corroboration: four independent production exports on the copy side (tables from three
different solutions, including a 300+ field production table), one generated 12-field
table round-tripped byte-identical on v26 (151 elements, zero drift), and eight
repeat-paste collision runs. Items not yet paste-tested are listed in §8.

---

## 1. Envelope

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmxmlsnippet type="FMObjectList">
  <BaseTable comment="" name="contacts">
    <Field id="1" ...>...</Field>
    <Field id="2" ...>...</Field>
  </BaseTable>
</fmxmlsnippet>
```

`<BaseTable>` carries exactly two attributes — `name` and `comment`, both from the
Tables tab. No `id`, no UUID, no occurrence or record-count metadata. The field elements
inside are **identical** to the bare-field format in the field spec, §2–§10 and §14: the
table envelope is a thin wrapper, not a different format. Every mechanism tested — calc
auto-enter with context table, system values, serial, StrictDataType,
NotEmpty/Unique/StrictValidation, all index levels including the `autoIndex` omission at
`index="All"`, Unicode_Raw, stored and unstored calcs, Total summary — survives the
envelope at full fidelity in both directions.

Multiple `<BaseTable>` siblings in one snippet paste in a **single operation** — one
paste on the Tables tab creates every table in the snippet, each with its fields intact
and per-table IDs preserved. Round-trip confirmed with a two-table snippet, matching
FM's own multi-table copy format. Whole-schema scaffolds are a single paste.

**XML comments are tolerated by the Tables tab handler.** A snippet containing an
`<!-- -->` comment inside the envelope pasted successfully, comment stripped, table and
fields intact — unlike the Fields tab handler, which fails or pastes partially (field
spec §11 rule 6). Same envelope, third parser behaviour, after the script handler's
tolerance and the field handler's rejection. Generate comment-free regardless, so
snippets stay valid for every handler.

## 2. Forward references resolve

A calculation field may reference a field defined **later** in document order. A stored
calc `quantity * unit_price` with `unit_price` defined after it pasted and re-exported
intact. The handler resolves field names after all fields in the table exist, so
generated XML needs **no dependency ordering**. List fields in whatever order reads best.

## 3. Field IDs

- Into a fresh table, supplied IDs are preserved as written — sequential 1..n lands
  as 1..n, and internal references (Summary `<SummaryField><Field id=.../>`) land intact.
- ID uniqueness is scoped **per table**. Sibling `<BaseTable>` elements each start at
  `id="1"`; this is FM's own multi-table copy format.
- Generate sequential integers from 1 within each table.

## 4. Name collisions: silent rename, context rebound

Pasting a `<BaseTable>` whose name already exists produces no error and no prompt. FM
silently creates the table under the standard suffix pattern: `paste_test` →
`paste_test 2`, `paste_test 3`, ... (space plus number).

Critically, every `<Calculation table="...">` inside the renamed table is **rewritten to
the new table name** — even though a table matching the pasted value still exists in the
file. The handler rebinds local calculation context to the newly created table, and
local field references bind to the new table's own fields. Confirmed across seven
consecutive collision pastes.

Consequences: generated schema scaffolds cannot clobber or corrupt an existing table
(worst case is a suffixed duplicate), and the local-context `table` value is
self-correcting under rename.

The same rebinding applies to a **nonexistent context TO**: a calc pasted with
`table="no_such_occurrence_xyz"` re-exports with `table` rewritten to the created
table's name, calculation intact and references resolved — no error, no neutralisation.
Taken together, the `table` attribute is effectively advisory on table paste: the
handler rebinds context to the created table whenever the supplied value does not
resolve. Emit the base table name and move on.

## 5. Serial `nextValue` is honoured

`<Serial ... nextValue="1001"/>` pastes as next value 1001, exactly as written. This
makes table paste a serial-continuity-preserving migration path — and a generation
footgun. Never copy `nextValue` from an example: default it to `1` unless the user
specifies otherwise. Text-field serials with alphanumeric `nextValue` (e.g. `"J114071"`)
are export-observed; the numeric tail increments.

## 6. Table paste creates a graph occurrence

Every pasted table gets exactly one table occurrence on the relationships graph, named
identically to the table, standalone with no relationships. Observed across all three
paste variants: single table, collision rename (the collision pastes produced
occurrences "paste_test 2" through "paste_test 8"), and multi table single paste —
the same behaviour as creating a table manually. Generated schema scaffolds therefore
arrive graph-ready: occurrences exist for scripting and layout context immediately, and
relationship wiring is the only manual step.

## 7. Invalid calculations are neutralised, not rejected

A pasted calculation containing a `<Field Missing>` token (the literal marker FM writes
into exports where a referenced field was deleted) does **not** fail the paste. FM
creates the table and the field, preserves the calculation text, and wraps the entire
body in a calculation comment:

```
pasted:      <Field Missing> + 1
re-exported: /*<Field Missing> + 1*/
```

The field exists and looks normal in the field list, but its calculation evaluates to
nothing. This is the sneakiest silent-failure mode in the format: no error, no drop,
just a dead calc. The same neutralisation presumably applies to any syntactically
invalid calculation, though only the Missing token is round-trip confirmed.

Generation: never emit Missing tokens (they arrive comment-wrapped and dead). Analysis:
flag both bare Missing tokens in exports and calculations whose entire body is
comment-wrapped — the latter is the post-paste signature of a neutralised calc.

## 8. Verification status

Every behaviour in this spec is paste-verified on v26. The only export-observed item not
yet independently generation-tested is the `"ModificationName"` auto-enter value (field
spec §5.2), which is symmetric with the confirmed `"CreationName"`.

## 9. Copy-side fossils (analysis, not generation)

Production table exports faithfully preserve broken state rather than cleaning it.
Expect, and never generate:

- `valuelist="True"` with no `<ValueList>` child — a dangling reference in the source
  file (consistent with field spec §7.5, which describes the paste-side drop). When the
  reference resolves, table copy **does** emit `<ValueList>`.
- `lookup="False"` (or `message="False"`, `maxLength="False"`, `valuelist="False"`) with
  the corresponding child element still present — the option was configured then disabled,
  and FM keeps the child. **The paste handler retains the child and honours the attribute**:
  the option stays off, but the child rides along in the stored definition (rt5 confirmed
  `<MaxDataLength>` under `maxLength="False"` and `<ErrorMessage>` under `message="False"`
  both survive the Tables-tab paste, matching the disabled-Lookup fossil on the Fields tab,
  field spec §5.3). Never generate a disabled option carrying its child.
- Inactive `<Furigana>` with a null reference (`<Field id="0" ... name=""/>`) — see field
  spec §10.
- Internal base table IDs leaking through `<Lookup><Table id="1065xxx" .../>` — the only
  place FM's internal table ID space appears in this format. On paste the `<Table>` is
  **rewritten to the defining field's own base TO**, so a leaked id is inert: it cannot
  mis-target because context overrides whatever is sent (rt5 sent `id="1065999"
  name="GhostTable"`, FM returned the new table's own TO `id="1065091"`). The
  `<Lookup><Field>` resolves by name through the relationship and dangles to
  `<Field table="" id="0" name=""/>` when the relationship or source field is absent, as in
  a freshly pasted standalone table.
- `table=""` on `<Calculation>` — broken evaluation context (field spec §6.1).

---

*Clockwork Creative Technology — clockworkct.co.uk*
