# Canonical XML Format for FileMaker Value List Definitions (XMVL)

**Author:** Andrew Kear, Clockwork Creative Technology
**Version:** 2.1
**Verified against:** FileMaker Pro 2026 (v26) on macOS, by round-trip (generate XML, paste, re-export, diff). Every behaviour carrying a ✓ (round trip) marker is paste-verified; writer-side and open items are marked accordingly.
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Scope

Value list definitions in the `fmxmlsnippet type="FMObjectList"` envelope, copied from and
pasted into **File ▸ Manage ▸ Value Lists** — a separate dialog from Manage Database, where
fields and tables paste. Companion specs: `filemaker_xmfd_spec.md` (field definitions) and
`filemaker_xmtb_spec.md` (table definitions). Setup and paste mechanics are in the README.

Corroboration: writer side from production corpora totalling ~850 value lists across
independent live systems; paste side round-trip verified on v26 across all three source
arms (Custom, Field, External), id and collision behaviour, separator handling, the
fossil rule, and dangling-reference handling (§9). All examples below are synthetic; no
production data or live host appears.

**Confidence markers** carry an explicit scope qualifier, because writer-side and
paste-side are different claims:

| Mark | Meaning |
|---|---|
| ✓ (round trip) | Crafted, pasted, and confirmed in the dialog and on re-copy. |
| ✓ (writer) | Seen in FileMaker's clipboard output; says nothing on its own about the paste handler. |
| ◎ | Corroborated externally, or observed at scale but not deliberately tested. |
| ○ | Unknown. Listed in §9. |

**The one rule that governs the whole spec:** on paste, FileMaker resolves field and
external references **by name**; the `id` you supply is advisory and gets **rewritten to
reality** — healed to the true id when the name matches, zeroed to a Missing marker when
it does not. The value list's own `id` is the sole exception: it is always reassigned.
This is the same name-binding, context-rewriting behaviour the table handler shows (table
spec §4, §7).

---

## 1. Envelope

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmxmlsnippet type="FMObjectList">
  <ValueList id="1" name="status">
    <Source value="Custom"/>
    <CustomValues>
      <Text>Active&#13;Inactive&#13;Pending</Text>
    </CustomValues>
  </ValueList>
</fmxmlsnippet>
```

The classic `FMObjectList` envelope, UTF-8 — **not** the FileMaker 2026 `FMObjectTransfer`
wrapper used by custom menu sets. The two clipboard generations coexist in 26.0.1; a value
list snippet carries **no provenance block** (no author, no file UUID) — that leakage is a
property of the new envelope only. ✓ (writer)

**Multiple lists copy in one envelope.** Selecting several lists in the dialog and copying
yields multiple `<ValueList>` siblings in one `FMObjectList` — the same batch shape as
multi-`<BaseTable>` copy (table spec §1). ✓ (writer). Whether a multi-list snippet pastes
every list in one operation is expected to mirror single-list paste but was not separately
tested. ◎

**Tag collision — do not confuse.** The top-level `<ValueList>` here is a value list
*definition*. The `<ValueList>` child of a field's `<Validation>` in the field spec (§7.5)
is a *reference* to a value list by id. Same tag, different role — one *defines* a value
list, the other *points at* one from a field's validation, and they paste in different
dialogs. A field snippet references value lists; it never defines them.

## 2. Element order and attribute inventory

Within a `<ValueList>`, element order is fixed:

```
Source → [CustomValues] → [PrimaryField] → [SecondaryField] → [ShowRelated | External]
```

`ShowRelated` and `External` are **mutually exclusive** — the Field arm uses `ShowRelated`,
the External arm uses `External`, in the same structural slot.

| Element | Attributes / content | Notes | Scope |
|---|---|---|---|
| `ValueList` | `id`, `name` | Top-level. `id` always reassigned on paste (§8). | ✓ (round trip) |
| `Source` | `value` = `Custom` \| `Field` \| `External` | Selects the active arm. Self-closing. | ✓ (round trip) |
| `CustomValues` | contains one `Text` | Custom arm payload. Empty box → self-closing `<Text/>`. | ✓ (round trip) |
| `Text` | CR-delimited values | See §4. | ✓ (round trip) |
| `PrimaryField` | `show`, `sort`, `resortLanguage` | Contains a `Field`. `show` encodes the display choice with `SecondaryField` (§3.2); `resortLanguage` on the sort key. | ✓ (round trip) |
| `SecondaryField` | `show`, `sort`, `resortLanguage` | Present only when a second field is configured. | ✓ (round trip) |
| `ShowRelated` | `value` | Field arm only. `False` self-closes; `True` contains a `Table`. | ✓ (round trip) |
| `Table` | `id`, `name` | Starting TO. Binds by name; a wrong `id` heals to the real TO id on paste (rt1). TO ids are a distinct namespace from field and value list ids. | ✓ (round trip) |
| `External` | — | External arm only. Contains a `FileReference` then a cross-file `ValueList`. | ✓ (round trip) |
| `FileReference` | `id`, `name` | The external data source. Contains `UniversalPathList`. | ✓ (round trip) |
| `UniversalPathList` | fmnet / file path | e.g. `fmnet:/host.example.com/RemoteFile`. | ✓ (round trip) |
| `ValueList` (in `External`) | `id`, `name` | The list in the *other* file. Resolved by name; id self-heals on paste (§3.3). | ✓ (round trip) |
| `Field` (in Primary/SecondaryField) | `table`, `id`, `name` | The bound field. Resolved by name; id rewritten on paste (§8). | ✓ (round trip) |

## 3. Source arms

**3.1 Custom values** — minimal form, as §1. Pastes and round-trips whole: values and CR
separators preserved on re-copy. ✓ (round trip)

**3.2 Values from a field** — full form, every option engaged:

```xml
<ValueList id="2" name="people">
  <Source value="Field"/>
  <PrimaryField show="False" sort="False">
    <Field table="contacts" id="1" name="contact_id"/>
  </PrimaryField>
  <SecondaryField resortLanguage="English" show="True" sort="True">
    <Field table="contacts" id="2" name="full_name"/>
  </SecondaryField>
  <ShowRelated value="True">
    <Table id="1065094" name="projects"/>
  </ShowRelated>
</ValueList>
```

Binds on paste; the field reference resolves **by name**, and FileMaker rewrites the
`Field id` to the true field's id (§8). The `ShowRelated` `Table` reference resolves the
same way: a deliberately wrong `Table id` heals to the real TO id when the name matches
(rt1 sent `999999`, FileMaker returned `1065089`), and `value="True"` plus "related values
only" round-trip. ✓ (round trip)

`SecondaryField` binds and round-trips (both fields resolve by name, ids heal), and `sort`
on either field plus `resortLanguage` on the sort key round-trip exactly — rt1 `zz_sortsecond`
kept `SecondaryField sort="True"`, `zz_resortlang` returned `resortLanguage="English"`
unchanged. ✓ (round trip)

**The `show` attributes encode the display choice; they are not per-field visibility
toggles.** `SecondaryField show="True"` is the *"show values only from second field"*
checkbox — the second field is displayed by the **presence** of the `SecondaryField`
element, and `show` on it controls whether the first field is suppressed. The three display
states, all native-verified and round-tripping:

| Display | `PrimaryField show` | `SecondaryField` |
|---|---|---|
| First field only | `True` | absent |
| Both fields | `True` | present, `show="False"` |
| Second field only | `False` | present, `show="True"` |

Sending `PrimaryField show="True"` **with** `SecondaryField show="True"` is contradictory —
show the first, but show only the second — and does not round-trip: FileMaker honours the
second-only flag and rewrites primary to `show="False"` (rt1 `zz_secondary`, `zz_sortsecond`).
That is the only non-round-tripping combination; the three valid states above are stable.
When generating "display both", emit `PrimaryField show="True"` with `SecondaryField
show="False"`, never both `show="True"`. ✓ (round trip)

**3.3 Value list from another file (External arm)** — a dedicated `<External>` block, no
field references required:

```xml
<ValueList id="3" name="suppliers">
  <Source value="External"/>
  <External>
    <FileReference id="1" name="RemoteFile">
      <UniversalPathList>fmnet:/host.example.com/RemoteFile</UniversalPathList>
    </FileReference>
    <ValueList id="441" name="RemoteList"/>
  </External>
</ValueList>
```

The `External` block holds a `FileReference` (the external data source, with its
`UniversalPathList` path) and a cross-file `ValueList` descriptor naming the list *in the
other file*. It occupies the slot the Field arm uses for `ShowRelated`; the two are
mutually exclusive. Pasting **reuses** an existing matching data source rather than
duplicating it (the `FileReference id` came back unchanged, not incremented), and the
cross-file list resolves **by name** — a deliberately wrong cross-file `id` self-heals to
the true id on paste, so external references are portable across files despite the foreign
id space. ✓ (round trip)

An External arm needs only `Source value="External"` and the `External` block. A
`PrimaryField`/`SecondaryField` alongside an active External source is a **fossil** from a
former Field configuration (§6), not part of the External arm.

**External data source resolution is eager, not lazy.** When the named data source does
not exist in the target, FileMaker attempts to *open the referenced file* during paste and
throws a blocking modal ("The file '…' could not be opened. Either the host is not
available, or the file is not available on that host."). The `.fmp12` extension is inferred
from the bare `FileReference` name. After the dialog is cleared the value list **is**
created, as a live dangle: source "From Another File", presenting **two** missing tokens —
`<File Missing>` for the `FileReference` and `<Value List Missing>` for the cross-file
`ValueList`. FileMaker does not auto-create the source and does not fail silently. This is
the noisiest failure in the format: a bad External arm interrupts paste with a connection
error before installing dead, so never emit an External arm whose data source is not
guaranteed present in the target. ✓ (round trip)

## 4. Separator and divider semantics

- Values are separated by carriage return, serialised `&#13;`. Custom arm round-trips
  whole. ✓ (round trip)
- A hyphen alone on a line is **divider syntax, not data** (per the dialog's own help
  text). A value list cannot hold a literal lone-hyphen value. ◎
- Consecutive CRs (`&#13;&#13;`, an empty line) are preserved verbatim — no
  normalisation. ✓ (writer)
- A trailing `&#13;` tracks whether the dialog content ended with a return; preserved
  verbatim. ✓ (writer)
- An empty custom-values box serialises as self-closing `<Text/>`. ✓ (writer)
- **LF (`&#10;`) is functionally accepted, not a silent failure.** A list seeded with LF
  separators pastes as distinct values (no weld), displays correctly in the dialog, and
  drives a layout control correctly. On re-copy FileMaker emits the separator as a **bare
  newline** rather than `&#13;` — a serialisation difference with no functional
  consequence, because the stored list works. ✓ (round trip)

**Why FileMaker escapes CR but not LF** — and why generators must still emit `&#13;`. XML
1.0 end-of-line handling (§2.11) forces a parser to normalise any *literal* CR to LF on
input, before parsing. A bare `\r` value separator would therefore be silently rewritten
to `\n` on the next read — so FileMaker writes CR as a **character reference** (`&#13;`),
which is resolved *after* normalisation and survives. LF is already the normalisation
target, so a literal LF is XML-safe and FileMaker leaves it unescaped. **Generation rule:
emit `&#13;`** — not because LF breaks anything (it doesn't), but because it is FileMaker's
canonical form and there is no upside to diverging. Never emit a bare `\r`, the one form
that would silently rot.

## 5. Show and sort semantics

| Dialog state | XML encoding |
|---|---|
| Display first field only | `PrimaryField show="True"`, no `SecondaryField` |
| Also display second field | `SecondaryField` present, `show="True"` |
| Show values only from second field | `PrimaryField show="False"`, `SecondaryField show="True"` |
| Sort by first field | `PrimaryField sort="True"` |
| Sort by second field | `PrimaryField sort="False"`, `SecondaryField sort="True"` |
| Re-sort by language | `resortLanguage` on the sorting field |

`resortLanguage` attaches to whichever field is the active sort key, not a fixed field.
All rows ✓ (writer); the Field arm's binding and round-trip are ✓ (round trip) per §3.2.

## 6. The hybrid fossil rule

**Novel finding, now round-trip confirmed.** A value list switched between sources retains
the *inactive* arm's data in its serialisation: the writer emits the **union** of
everything the list has ever been configured as, and — critically — **paste installs the
fossil into the target's stored definition**; it is not stripped.

```xml
<!-- synthetic: Field source carrying a Custom fossil -->
<ValueList id="4" name="assignee">
  <Source value="Field"/>
  <CustomValues>
    <Text>Alice&#13;Bob&#13;Carol</Text>
  </CustomValues>
  <PrimaryField show="True" sort="True">
    <Field table="employees" id="12" name="assignee"/>
  </PrimaryField>
  <ShowRelated value="False"/>
</ValueList>
```

The active arm is whichever `Source value` names; the rest is fossil. The fossil spans
`CustomValues`, `PrimaryField`, and `SecondaryField` but **not** `ShowRelated`. The Edit
Value List dialog shows only the active arm; the fossil is invisible from the UI but
present in storage — confirmed by pasting a fossil-bearing list and re-copying it with the
fossil intact. ✓ (round trip)

**Generation rule: emit only the active arm.** Because paste *installs* the inactive arm
rather than ignoring it, a generator that emits both arms ships a phantom hidden arm into
the target — not a harmless no-op.

**Data-protection consequence (demonstrated, not just reasoned).** A value list copied to
share carries invisible stale data — old field bindings, former custom values — into
wherever it is pasted. This is a Scrubber case: **strip inactive arms before a value list
leaves the building.**

## 7. Dangling and degenerate forms

The writer preserves broken bindings verbatim, and **paste installs them as live-looking
but dead** rather than rejecting them (same family as the table spec §7, invalid-calc
neutralisation).

A field reference whose **name does not resolve** installs with the source intact and the
field shown as `<Field Missing>` — no paste error. On paste the `id` is **zeroed to `0`**:

```xml
<!-- pasted -->
<PrimaryField show="True" sort="True">
  <Field table="orders" id="999" name=""/>
</PrimaryField>

<!-- re-exported: unresolvable name → id zeroed, source kept, field dead -->
<PrimaryField show="True" sort="True">
  <Field table="orders" id="0" name=""/>
</PrimaryField>
```

**Provenance signature (for analysis).** The rewritten id distinguishes two dangle
origins:

- `id="0" name=""` — a **paste-side** dangle, or a reference whose name never resolved.
  FileMaker zeroed it on paste.
- *nonzero* `id` with `name=""` — a **deletion-side** fossil: the field once existed at
  that id and was later deleted, and the id survived in the copy. This is the wild form
  seen in production exports.

Degenerate forms also exist: a `Field` source with **no** `PrimaryField`, and an empty
`<Text/>` on a Custom source. ✓ (writer, exists in wild).

**Generation:** never emit an unresolvable reference. The
`PrimaryField`/`SecondaryField`/cross-file `ValueList` **name** must resolve in the target,
or you ship a silent `<Field Missing>` (or `<unknown>` external list) — a value list that
looks present in the list view with a dead binding.

## 8. ID allocation, name collision, and reference resolution

**The value list's own id is always reassigned on paste** — the sole reference in this
spec that is *not* name-resolved. A supplied `ValueList id` is discarded and the list
takes the next value from the file's per-file sequential counter, whether or not the
supplied id collides. ✓ (round trip)

This is the **opposite** of fields and tables, which honour supplied ids on paste (table
spec §3, §5). A generator cannot set a stable value list id, and nothing downstream should
rely on one surviving. It does not matter in practice, because —

**All other references resolve by name; the id is advisory and rewritten to reality:**

- **Field references** (Field arm): resolve by name. A wrong id with a resolvable name
  self-heals to the true field id (999 → 5); an unresolvable name zeroes to `0`
  (§7). ✓ (round trip)
- **Cross-file value list** (External arm): resolves by name. A wrong cross-file id
  self-heals to the true id in the external file (999 → 441). ✓ (round trip)

**Collision and duplication:**

- Value list ids allocate from a per-file sequential counter in creation order — gaps
  present, no observed reuse of deleted ids. ✓ (round trip, confirmed on paste)
- Two name conventions, keyed to the *operation*:
  - **Name collision on paste or create** → space + number: `test` → `test 2`,
    `test 3`. Confirmed on paste. ✓ (round trip)
  - **Duplicate button** → `name Copy`, `name Copy 2`. ✓ (round trip)
- A **unique** name pastes unchanged (no rename). ✓ (round trip)
- A `Copy`-suffixed list in the wild frequently diverges completely from its original — a
  `Copy` suffix is duplicate-then-edit residue, **not** evidence of redundancy. Relevant
  to audit tooling. ✓ (writer)

## 9. Verification status

Every behaviour in this spec carrying a ✓ (round trip) marker is paste-verified on v26.
The original reader-test set is fully resolved:

| Test | Result |
|---|---|
| Paste handler exists | Yes — not a copy-only format. |
| id honour vs reassign | Reassigned, always. |
| LF separator | Functional; returned as a bare newline; no weld. |
| Basic paste of each arm | Custom, Field, External all paste and bind. |
| Collision convention on paste | Space + number; unique names unchanged. |
| Field / external-list binding | By name; id self-heals or zeroes. |
| External data source (FileReference) | Eager: FM opens the file on paste. Missing source → error modal, then live dangle (`<File Missing>` + `<Value List Missing>`). |
| Fossil install vs strip | Installs (not stripped). |
| Dangling install vs neutralise | Installs as a dead `<Field Missing>` binding, no error. |

**Remaining open (○):**

- **The layout `DDRInfo` value-list reference** (`<ValueList id="N"/>`, in the layout
  skill) is a *different* reference in a *different* handler. This spec's name-binding does
  not prove name-binding there — that binding still warrants its own test before the layout
  spec's "id must be real" line is revised.

Not in scope for this spec: value list *lineage* fingerprinting (id correlation across
sibling files) is a forensic/analysis technique, not a format rule — it belongs with the
Inspector, not here.

---

*Clockwork Creative Technology — clockworkct.co.uk*
