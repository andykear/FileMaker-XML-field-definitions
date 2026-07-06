# Canonical XML Format for FileMaker Field Definitions (XMFD)

**Author:** Andrew Kear, Clockwork Creative Technology
**Version:** 2.0
**Date:** July 2026
**Verified against:** FileMaker Pro 2025 (v22) and 2026 (v26) on macOS, by round-trip (generate XML, paste, re-export, diff). Table-level paste (§15) verified on v26. Everything below is round-trip confirmed unless flagged in §13.
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Scope

The `fmxmlsnippet type="FMObjectList"` clipboard format containing `<Field>` elements,
pasted into Manage Database to create fields in bulk — and, from v2.0, `<BaseTable>`
elements pasted on the Tables tab to create complete tables, fields included (§15).
Requires the MBS Plugin present in FileMaker Pro (no MBS scripting needed). Companion
specs cover script-step and layout XML.

---

## Quick start

1. Write or generate XML conforming to this spec (no comments, see §1).
2. Select all, copy.
3. In FileMaker Pro, open Manage Database. For bare `<Field>` elements, select the
   target table on the Fields tab and paste. For `<BaseTable>` snippets, paste on the
   Tables tab — FileMaker creates the table and every field inside it (§15).

Fields are created in document order. FM assigns real internal IDs, and resolves Summary
source fields and Lookup source fields by name at paste time.

---

## 1. Envelope

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmxmlsnippet type="FMObjectList">
  <!-- one or more <Field> elements -->
</fmxmlsnippet>
```

Identical envelope to script XML. 2-space indent, UTF-8.

**No XML comments.** The field paste handler fails or pastes partially if the snippet
contains `<!-- -->` comments (the script paste handler tolerates them; this one does not).
Keep generated field XML comment-free; document intent with `<Comment>` elements and field
names, which round-trip cleanly.

---

## 2. `<Field>` — top-level attributes

```xml
<Field id="1" dataType="Text" fieldType="Normal" name="field_name">
```

### `id`

Internal numeric ID. For generation, use sequential integers starting at 1 — FM assigns
real IDs on paste. **Use unique sequential IDs within a multi-field paste, not `id="1"`
for all fields.** Duplicate IDs cause silent drops when calc auto-enter fields are
included.

### `dataType`

| Value | FM type |
|---|---|
| `Text` | Text |
| `Number` | Number |
| `Date` | Date |
| `Time` | Time |
| `TimeStamp` | Timestamp |
| `Binary` | Container |

### `fieldType`

| Value | FM type |
|---|---|
| `Normal` | Regular field |
| `Calculated` | Calculation field |
| `Summary` | Summary field |

### `name`

Field name as a string. May contain spaces. Case-sensitive.

---

## 3. Child element order by `fieldType`

### Normal field

```
<Comment/>
<AutoEnter>
  <ConstantData/>
  <!-- optional additional auto-enter children -->
</AutoEnter>
<Validation>
  <!-- validation children -->
</Validation>
<Storage/>
<!-- Furigana only present when Furigana is configured (after Storage) -->
<!-- FileMaker 2026 (v26): Annotation then DisplayNames follow, after Furigana.
     Confirmed order: Storage, Furigana, Annotation, DisplayNames. See sections 10 and 14. -->
```

### Calculation field

```
<Calculation/>    <!-- ALWAYS FIRST -->
<Comment/>
<AutoEnter/>      <!-- simplified form, no sub-elements -->
<Storage/>
<!-- FileMaker 2026 (v26): Annotation then DisplayNames follow, after Storage -->
```

`<Calculation>` is always first. This differs from Normal fields where `<Comment>` leads.

### Summary field

```
<SummaryInfo>
  <SummaryField/>
  <AdditionalField/>    <!-- optional, see §9.3 -->
</SummaryInfo>
<Comment/>
<AutoEnter>
  <ConstantData/>
</AutoEnter>
<!-- NO <Validation>, NO <Storage> -->
<!-- FileMaker 2026 (v26): Annotation then DisplayNames follow, after AutoEnter -->
```

---

## 4. `<Comment>`

```xml
<Comment/>
<Comment>text content here</Comment>
```

Always present on every field type. Self-closing when empty.

---

## 5. `<AutoEnter>` — Normal fields

### 5.1 Attributes

| Attribute | Values | Notes |
|---|---|---|
| `allowEditing` | `"True"` / `"False"` | `"False"` = prohibit modification during data entry |
| `overwriteExistingValue` | `"True"` / `"False"` | `"False"` = do not replace existing value |
| `alwaysEvaluate` | `"True"` / `"False"` | Recalculate on every commit |
| `value` | see §5.2 | System value auto-enter type |
| `constant` | `"True"` / `"False"` | Constant data auto-enter active |
| `furigana` | `"True"` / `"False"` | Furigana input active |
| `lookup` | `"True"` / `"False"` | Lookup auto-enter active |
| `calculation` | `"True"` / `"False"` | Calculation auto-enter active |

`overwriteExistingValue` and `alwaysEvaluate` are omitted when not applicable to the
auto-enter type in use.

### 5.2 `value` attribute — system value auto-enter

| XML value | UI label |
|---|---|
| `"CreationDate"` | Creation → Date |
| `"CreationTime"` | Creation → Time |
| `"CreationTimeStamp"` | Creation → Timestamp |
| `"CreationName"` | Creation → Name |
| `"CreationAccountName"` | Creation → Account Name |
| `"ModificationDate"` | Modification → Date |
| `"ModificationTime"` | Modification → Time |
| `"ModificationTimeStamp"` | Modification → Timestamp |
| `"ModificationName"` | Modification → Name |
| `"ModificationAccountName"` | Modification → Account Name |
| `"ConstantData"` | Data (constant) |
| `"PreviousRecord"` | Value from last visited record |

Absent when no system value is active.

`"CreationName"` and `"ModificationName"` are the user-name variants, distinct from the
account-name variants. `"CreationName"` is generation-confirmed (pasted XML produces
Creation → Name ticked in the UI); `"ModificationName"` is export-observed in two
independent production files and sits in the same symmetric value pair. All other values
are fully round-trip confirmed.

### 5.3 Child elements of `<AutoEnter>`

Multiple mechanisms can coexist (Serial + Calculation, Calculation + Lookup, etc).
`<ConstantData>` is always present.

**`<ConstantData>`** — always present:
```xml
<ConstantData/>                        <!-- no constant defined -->
<ConstantData>Active</ConstantData>    <!-- constant text -->
<ConstantData>N</ConstantData>         <!-- single character -->
<ConstantData>0</ConstantData>         <!-- numeric string -->
<ConstantData>00</ConstantData>        <!-- zero-padded string -->
```

FM stores `<ConstantData>` content verbatim regardless of the field's `dataType`. Any
string content is valid.

**`<Serial>`**:
```xml
<Serial increment="1" nextValue="1" generate="OnCreation"/>
<Serial increment="1" nextValue="1" generate="OnCommit"/>
```
`nextValue` may be numeric or alphanumeric: `"SGP-27286"`, `"SC-001"`, `"A1131700"`.
Serial auto-enter works on both Number and Text fields — on Text the generated value
is stored as a string.

**`<Calculation>`** — auto-enter by calculation:
```xml
<Calculation table="table_name"><![CDATA[expression]]></Calculation>
```

**`<Lookup>`**:
```xml
<Lookup>
  <Table id="1065094" name="table_occurrence_name"/>
  <Field table="relationship_name" id="3" name="field_name"/>
  <NoMatchCopyOption value="DoNotCopy"/>
  <CopyEmptyContent value="False"/>
</Lookup>
```

`NoMatchCopyOption` values:

| XML value | UI label |
|---|---|
| `"DoNotCopy"` | do not copy |
| `"CopyNextLower"` | copy next lower value |
| `"CopyNextHigher"` | copy next higher value |
| `"CopyConstant"` | use [constant] |

When `"CopyConstant"`, additional child elements appear inside `<Lookup>`:
```xml
<NoMatchCopyOption value="CopyConstant"/>
<CopyConstantValue>23</CopyConstantValue>
<CopyEmptyContent value="False"/>
```

`<CopyConstantValue>` may be self-closing when the constant is empty:
```xml
<CopyConstantValue/>
```

`CopyEmptyContent` controls what happens when the lookup source field is empty.
`"False"` = do not copy empty content (the existing field value is preserved).
`"True"` = copy even if empty (the field is cleared).

**Broken lookup reference** — when the source field does not resolve, FM stores:
```xml
<Field table="" id="0" name=""/>
```
Various low id values (0, 2, 3, 5) have been seen in production exports. All represent
the same broken state.

All four `NoMatchCopyOption` values, the `<CopyConstantValue>` child (populated for
`CopyConstant`), and both `CopyEmptyContent` states round-trip. Lookup references resolve
against the target file's relationships and field IDs, so a generated `<Lookup>` block
only round-trips when the relationship and source field exist in the target.

### 5.4 Coexisting AutoEnter children

The following combinations are valid and confirmed by round-trip testing:

- `<ConstantData>` text + `<Calculation>` child when `value="ConstantData"` — FM
  preserves both elements regardless of which is active.
- `<Calculation>` child present when `calculation="False"` — the attribute controls
  execution; element presence is independent.
- `<Calculation>` + `<Lookup>` children on the same field.
- `<Serial>` + `<ConstantData>` + `<Calculation>` on the same field.

### 5.5 Standard AutoEnter patterns

**No auto-enter:**
```xml
<AutoEnter allowEditing="True" constant="False" furigana="False" lookup="False" calculation="False">
  <ConstantData/>
</AutoEnter>
```

**Constant data:**
```xml
<AutoEnter allowEditing="True" value="ConstantData" constant="True" furigana="False" lookup="False" calculation="False">
  <ConstantData>Active</ConstantData>
</AutoEnter>
```

**Creation timestamp — prohibit modification:**
```xml
<AutoEnter allowEditing="False" value="CreationTimeStamp" constant="False" furigana="False" lookup="False" calculation="False">
  <ConstantData/>
</AutoEnter>
```

**Serial number:**
```xml
<AutoEnter allowEditing="True" constant="False" furigana="False" lookup="False" calculation="False">
  <Serial increment="1" nextValue="1" generate="OnCreation"/>
  <ConstantData/>
</AutoEnter>
```

**UUID primary key — prohibit modification, overwrite on creation:**
```xml
<AutoEnter allowEditing="False" overwriteExistingValue="True" alwaysEvaluate="False" constant="False" furigana="False" lookup="False" calculation="True">
  <ConstantData/>
  <Calculation table="table_name"><![CDATA[Get( UUID )]]></Calculation>
</AutoEnter>
```

**Calculation — overwrite always:**
```xml
<AutoEnter allowEditing="True" overwriteExistingValue="True" alwaysEvaluate="True" constant="False" furigana="False" lookup="False" calculation="True">
  <ConstantData/>
  <Calculation table="table_name"><![CDATA[expression]]></Calculation>
</AutoEnter>
```

**Calculation — do not overwrite existing:**
```xml
<AutoEnter allowEditing="True" overwriteExistingValue="False" alwaysEvaluate="False" constant="False" furigana="False" lookup="False" calculation="True">
  <ConstantData/>
  <Calculation table="table_name"><![CDATA[expression]]></Calculation>
</AutoEnter>
```

**Value from previous record:**
```xml
<AutoEnter allowEditing="True" value="PreviousRecord" constant="False" furigana="False" lookup="False" calculation="False">
  <ConstantData/>
</AutoEnter>
```

---

## 6. `<AutoEnter>` — Calculation fields

Simplified form with no sub-elements:

```xml
<AutoEnter alwaysEvaluate="False"/>
<AutoEnter alwaysEvaluate="True"/>
```

### 6.1 Calculation context: the `table` attribute

The `table` attribute on every `<Calculation>` element (calculation fields, auto-enter
calcs, validation calcs, container Location calcs) records the **evaluation context** —
the "evaluate this calculation from the context of" table occurrence in Manage Database —
not necessarily the base table name. Production exports show calcs inside `BaseTable
name="jobs"` carrying `table="employees jobs tagged"` where a non-default context TO was
selected. Legacy exports with broken context carry `table=""`.

Generation rules:

- Emit the base table name unless the user specifies a context TO.
- Never emit `table=""` — it is a broken-reference fossil, not a valid target.
- On table paste, FM rewrites the context to the created table whenever the supplied
  value does not resolve — both on collision rename and when the named TO simply does
  not exist (§15.4). The local-context value is self-correcting; emit the base table
  name.

---

## 7. `<Validation>`

Present on Normal fields. Absent from Summary fields. Presence on Calculation fields
is inconsistent and the rule has not been confirmed.

### 7.1 Attributes

| Attribute | Values | Notes |
|---|---|---|
| `messageCalc` | `"True"` / `"False"` | Error message is a calculation |
| `message` | `"True"` / `"False"` | Custom error message is set |
| `maxLength` | `"True"` / `"False"` | Max character length active |
| `valuelist` | `"True"` / `"False"` | Value list validation active |
| `calculation` | `"True"` / `"False"` | Calculation validation active |
| `alwaysValidateCalculation` | `"True"` / `"False"` | See note below |
| `type` | `"OnlyDuringDataEntry"` / `"Always"` | When validation fires |

`alwaysValidateCalculation` is the inverse of the "Validate only if field has been
modified" checkbox in the Specify Calculation dialog for calculation validation.
`"True"` = always validate (checkbox off); `"False"` = validate only when the field has
been modified (checkbox on). Confirmed by round-trip on v26.

`message` and `messageCalc` distinguish the two forms of custom failure message.
`message="True"` with an `<ErrorMessage>` child is a static message. `messageCalc="True"`
with a `<MessageCalculation>` child is a calculated message, a capability added in
FileMaker 21.1.1 (earlier clients can only set or display a static message). Both
attributes are independent of `StrictValidation`.

### 7.2 Child element order

```
<StrictDataType/>        <!-- if strict data type active -->
<NotEmpty/>
<Unique/>
<Existing/>
<ValueList/>             <!-- if value list active AND reference resolves -->
<Range/>                 <!-- if range active -->
<Calculation/>           <!-- if calculation validation active -->
<MaxDataLength/>         <!-- if max length active -->
<StrictValidation/>
<ErrorMessage/>          <!-- if present; see note below -->
<MessageCalculation/>    <!-- if message calculation active -->
```

### 7.3 Child elements in detail

```xml
<StrictDataType value="Numeric"/>
<StrictDataType value="FourDigitYear"/>
<StrictDataType value="TimeOfDay"/>

<NotEmpty value="True"/>
<Unique value="True"/>
<Existing value="True"/>

<ValueList id="1" name="list_name"/>

<Range from="1" to="100"/>
<!-- FM emits attributes as to then from: <Range to="100" from="1"/>.
     Attribute order is not significant on paste; match native order when mirroring output. -->

<Calculation table="table_name"><![CDATA[validation expression]]></Calculation>

<MaxDataLength value="255"/>
<!-- On Text fields this is a character limit. On Container (Binary) fields the SAME
     element is the kilobyte limit (Maximum number of kilobytes). The field's dataType
     determines the unit. Gated by maxLength="True" in both cases. -->

<StrictValidation value="True"/>
<!-- "True"  = override not allowed -->
<!-- "False" = allow user to override during data entry -->

<ErrorMessage>This field is required</ErrorMessage>

<MessageCalculation>
  <Calculation table="table_name"><![CDATA[expression]]></Calculation>
</MessageCalculation>
```

`<ErrorMessage>` is preserved by FM regardless of whether `StrictValidation` is `"True"`.
It may be present even when `StrictValidation` is `"False"`. It may contain XML character
entities:
```xml
<ErrorMessage>Line one&#13;Line two</ErrorMessage>
```

**Static vs calculated custom message (confirmed by round-trip on v26):**

- A **static** custom message emits `message="True"`, `messageCalc="False"`, and only
  `<ErrorMessage>`.
- A **calculated** custom message emits **both** `message="True"` **and**
  `messageCalc="True"`, and FM stores **both** elements: `<MessageCalculation>` holding
  the calc, and `<ErrorMessage>` holding the calc's evaluated static fallback (for clients
  earlier than 21.1.1 that cannot read the calculated form). Element order is
  `<ErrorMessage>` then `<MessageCalculation>`, both after `<StrictValidation>`.

Note the fallback in `<ErrorMessage>` is the evaluated result of the calc. If the calc is
a bare quoted string literal such as `"hi"`, the stored fallback includes the quotes,
because that is what the expression evaluates to as text.

### 7.4 Minimal validation block

```xml
<Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
  <NotEmpty value="False"/>
  <Unique value="False"/>
  <Existing value="False"/>
  <StrictValidation value="False"/>
</Validation>
```

### 7.5 `<ValueList>` reference resolution

The `<ValueList>` child element is only preserved by FM when the referenced value list
ID exists in the file at paste time. If the ID does not resolve, FM silently drops the
`<ValueList>` child but preserves `valuelist="True"` on the `<Validation>` element.
This mirrors the behaviour of broken Lookup field references.

When generating XML for a known target file, use the real value list ID. When generating
for general use, omit `<ValueList>` and set `valuelist="False"` — the field will paste
cleanly and the value list can be assigned manually afterwards.

---

## 8. `<Storage>`

### 8.1 Normal fields

```xml
<Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
```

| Attribute | Values | Notes |
|---|---|---|
| `autoIndex` | `"True"` / `"False"` | Automatically create indexes as needed. Omitted when `index="All"` |
| `index` | `"None"` / `"Minimal"` / `"All"` | Index level |
| `indexLanguage` | see §8.4 | Default language for indexing |
| `global` | `"True"` / `"False"` | Global storage |
| `maxRepetition` | integer string | `"1"` for non-repeating |

**`autoIndex` omission rule:** when `index="All"`, `autoIndex` is omitted entirely.
When `index="None"` or `index="Minimal"`, `autoIndex` is present.

**Unique/Existing validation forces a minimum index.** A field with `<Unique value="True"/>`
or `<Existing value="True"/>` cannot be unindexed. If such a field is pasted with
`index="None"`, FileMaker stores it as `index="Minimal"`. Generate `index="Minimal"`
(or higher) for any field carrying unique or existing validation, as the UUID primary
key template does, to match what FM stores.

**Global field** — `autoIndex` and `index` absent:
```xml
<Storage indexLanguage="English" global="True" maxRepetition="1"/>
```

**Container field (Binary)** — `indexLanguage` and `autoIndex` absent:
```xml
<Storage global="False" maxRepetition="1"/>
```

**Container global:**
```xml
<Storage global="True" maxRepetition="1"/>
```

**Container with external Open storage:**
```xml
<Storage global="False" maxRepetition="1">
  <Remote type="Open" relativeToPath="files/" relativeTo="0">
    <Location>
      <Calculation table="table_name"><![CDATA["path/expression/"]]></Calculation>
    </Location>
  </Remote>
</Storage>
```

**Container with external Secure storage:**
```xml
<Storage global="False" maxRepetition="1">
  <Remote type="Secure" withFewerFolders="False" relativeToPath="files/" relativeTo="0">
    <Secure/>
  </Remote>
</Storage>
```

`relativeTo` values:
- `"0"` — path relative to `relativeToPath`
- `"1"` — relative to database file location

`withFewerFolders` is only present on `type="Secure"`. Confirmed values: `"True"` / `"False"`.

`type` has exactly two values, `"Open"` and `"Secure"`; these are the only external
storage modes FileMaker offers (Claris Help: Setting up container fields to store data
externally). "With fewer folders" is a sub-option of Secure, not a separate type.

A custom open path is a different `<Location>` calculation; `<Secure/>` is an empty marker.

### 8.2 Calculation fields

Unstored:
```xml
<Storage storeCalculationResults="False" indexLanguage="English" global="False" maxRepetition="1"/>
```

Stored — `index="None"` or `"Minimal"`:
```xml
<Storage storeCalculationResults="True" autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
```

Stored — `index="All"` (`autoIndex` omitted):
```xml
<Storage storeCalculationResults="True" index="All" indexLanguage="Unicode_Raw" global="False" maxRepetition="1"/>
```

Global calc — `storeCalculationResults` absent entirely:
```xml
<Storage indexLanguage="English" global="True" maxRepetition="1"/>
```

Container calc — no `indexLanguage`:
```xml
<Storage storeCalculationResults="False" global="False" maxRepetition="1"/>
```

### 8.3 Repeating fields

`maxRepetition` set to repetition count. Applies to Normal, Calculated, and global fields:
```xml
<Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="20"/>
<Storage indexLanguage="English" global="True" maxRepetition="5"/>
```

### 8.4 `indexLanguage` values

**Confirmed from production exports:**

| UI label | XML value |
|---|---|
| Default | `"Unicode_Raw"` |
| Unicode | `"Unicode_Standard"` |
| English | `"English"` |
| Lithuanian | `"Lithuanian"` |
| Spanish / Spanish (Modern) | `"Spanish"` |
| Finnish (v<>w) | `"Finnish_Custom"` |
| German (ä=a) | `"German_Dictionary"` |
| Swedish (v<>w) | `"Swedish_Custom"` |
| Chinese (Pinyin) | `"Chinese"` |
| Chinese (Stroke) | `"Chinese_Stroke"` |
| Serbian (Latin) | `"Serbian"` |
| Greek (Mixed) | `"Greek_Mixed"` |

Other language names are expected to map verbatim to their UI labels (e.g. `"French"`,
`"German"`, `"Japanese"`) but have not been individually verified.

---

## 9. `<SummaryInfo>`

```xml
<SummaryInfo restartForEachSortedGroup="False" summarizeRepetition="Together" operation="Total">
  <SummaryField>
    <Field id="2" name="source_field"/>
  </SummaryField>
</SummaryInfo>
```

Summary field source references are resolved by **name** during paste, not by ID.
Use the correct field name with any placeholder ID — FM will update it on paste.

### 9.1 Attributes

| Attribute | Values |
|---|---|
| `restartForEachSortedGroup` | `"True"` / `"False"` |
| `summarizeRepetition` | `"Together"` / `"Individually"` |
| `operation` | see §9.2 |

### 9.2 `operation` values

Sub-options (population standard deviation, subtotalled fraction, weighted average,
running total/count) are distinct `operation` strings, not separate attributes. The UI
presents them as checkboxes under a base operation; each maps to its own value.

| XML value | UI selection | `<AdditionalField>` |
|---|---|---|
| `Total` | Total of | No |
| `RunningTotal` | Total of, Running total | Sort field, when restart is on |
| `Average` | Average of | No |
| `WeightedAverage` | Average of, Weighted average | Weight field (always) |
| `Count` | Count of | No |
| `RunningCount` | Count of, Running count | Sort field, when restart is on |
| `Minimum` | Minimum | No |
| `Maximum` | Maximum | No |
| `StdDeviation` | Standard Deviation of | No |
| `StdDeviationByPopulation` | Standard Deviation of, by population | No |
| `Fractional` | Fraction of Total of | No |
| `FractionalSubtotal` | Fraction of Total of, Subtotalled | Group field (always) |
| `List` | List of | No |

### 9.3 `<AdditionalField>`

Present for the operations marked in the table above. It holds the secondary field the
operation needs: the weight field for `WeightedAverage`, the group field for
`FractionalSubtotal`, and the sort field for `RunningTotal` / `RunningCount` when
`restartForEachSortedGroup="True"`.

```xml
<SummaryInfo restartForEachSortedGroup="True" summarizeRepetition="Together" operation="RunningCount">
  <SummaryField>
    <Field id="2" name="value_field"/>
  </SummaryField>
  <AdditionalField>
    <Field id="1" name="sort_field"/>
  </AdditionalField>
</SummaryInfo>
```

Weighted average (the weight field is required regardless of restart):

```xml
<SummaryInfo restartForEachSortedGroup="False" summarizeRepetition="Together" operation="WeightedAverage">
  <SummaryField>
    <Field id="2" name="value_field"/>
  </SummaryField>
  <AdditionalField>
    <Field id="1" name="weight_field"/>
  </AdditionalField>
</SummaryInfo>
```

Like `<SummaryField>`, the `<AdditionalField>` reference is resolved by name on paste.
For `WeightedAverage` and `FractionalSubtotal`, `restartForEachSortedGroup` may be
`"False"` and the `<AdditionalField>` is still present; for the running operations the
`<AdditionalField>` appears only when restart is `"True"`. `summarizeRepetition="Individually"`
is confirmed across all operations.

---

## 10. `<Furigana>`

Japanese-locale feature: auto-populates one field with the phonetic reading of another.
Emitted only when configured. Position: after `<Storage>`, before `<Annotation>` on v26
(order: Storage, Furigana, Annotation, DisplayNames); last child on v22.

**Inactive state**, present in exports but with empty `inputMode`:
```xml
<Furigana inputMode="">
  <Field id="0" baseTable="table_name" name=""/>
</Furigana>
```

**Active state:**
```xml
<Furigana inputMode="Hiragana">
  <Field id="5" baseTable="table_name" name="target_field_name"/>
</Furigana>
```

When active, `furigana="True"` on `<AutoEnter>` and the inner `<Field>` names the target
(kana) field. The inner `<Field>` uses `baseTable` (not `table`), unique to this context.

`inputMode` values. The six "Translate into" UI options are the complete set. `Hiragana`
is v26 round-trip confirmed; the other five XML values are from v22 and unchanged.

| UI label | XML value |
|---|---|
| (inactive) | `""` |
| As is | `"AsEntered"` |
| Hiragana | `"Hiragana"` (v26 round-trip confirmed) |
| Full-Width Katakana | `"2ByteKatakana"` |
| Full-Width Roman | `"2ByteRoman"` |
| Half-Width Katakana | `"1ByteKatakana"` |
| Half-Width Roman | `"1ByteRoman"` |

**Note:** FM drops the `<Furigana>` element on paste when the field's `<ValueList>`
reference does not resolve. If you need Furigana on a field, ensure the value list ID
is valid in the target file.

---

## 11. Paste handler rules

1. **Sequential unique IDs required.** Use unique sequential integers (1, 2, 3...) across
   all fields in a single paste. Duplicate IDs cause silent drops when calc auto-enter
   fields are included.

2. **Summary source fields resolved by name.** FM updates IDs at paste time — the name
   must match an existing field in the table.

3. **`<ValueList>` child dropped when ID does not resolve.** The `valuelist="True"`
   attribute is preserved on `<Validation>`, but the `<ValueList>` child element is
   removed. This mirrors broken Lookup field references. Use real value list IDs from
   the target file, or omit `<ValueList>` entirely and assign the list manually
   post-paste.

4. **`<Furigana>` dropped when `<ValueList>` does not resolve.** See §10.

5. **`autoIndex` omitted when `index="All"`.** FM does not emit `autoIndex` at this
   index level; generated XML should follow the same rule.

6. **Requires MBS Plugin to be installed.** Copy the XML, select the target
   table in Manage Database on the Fields tab, paste.

7. **No XML comments.** The field paste handler fails or pastes only partially when the
   snippet contains `<!-- -->` comments. Generated field XML must be comment-free. See §1.

8. **Unique/Existing validation forces minimum index.** A field with unique or existing
   validation pasted with `index="None"` is stored as `index="Minimal"`. See §8.1.

9. **ID uniqueness is scoped per table.** Rule 1 applies within one table. In multi-table
   snippets, sibling `<BaseTable>` elements each restart at `id="1"` — this is FM's own
   copy format and is correct, not an error. See §15.3.

10. **Colliding fields are renamed and re-IDed coherently.** Pasting fields into a table
    that already contains those names/IDs: names get the standard space-number suffix
    ("serial_no" → "serial_no 2"), IDs are reassigned to the next available internal
    values, and — critically — calculation and summary references *within the pasted
    set* are rewritten to follow the renamed copies ("quantity * unit_price" becomes
    "quantity 2 * unit_price 2"; a Summary source rebinds to the duplicate's new ID and
    name). The pasted group stays internally coherent rather than binding back to the
    originals. Serial `nextValue` is preserved through the rename.

Rules for the Tables tab handler (whole-table paste) are in §15.

---

## 12. Standard field templates

These templates are ready to paste. Replace `table_name` and `field_name` as required,
and renumber `id` attributes to continue from your highest existing field ID.

### UUID primary key
```xml
<Field id="1" dataType="Text" fieldType="Normal" name="PrimaryKey">
  <Comment>Unique identifier for each record</Comment>
  <AutoEnter allowEditing="False" overwriteExistingValue="True" alwaysEvaluate="False" constant="False" furigana="False" lookup="False" calculation="True">
    <ConstantData/>
    <Calculation table="table_name"><![CDATA[Get( UUID )]]></Calculation>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <NotEmpty value="True"/>
    <Unique value="True"/>
    <Existing value="False"/>
    <StrictValidation value="True"/>
  </Validation>
  <Storage autoIndex="True" index="Minimal" indexLanguage="Unicode_Raw" global="False" maxRepetition="1"/>
</Field>
```

### Creation timestamp
```xml
<Field id="2" dataType="TimeStamp" fieldType="Normal" name="CreationTimestamp">
  <Comment>Date and time each record was created</Comment>
  <AutoEnter allowEditing="False" value="CreationTimeStamp" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <StrictDataType value="FourDigitYear"/>
    <NotEmpty value="True"/>
    <Unique value="False"/>
    <Existing value="False"/>
    <StrictValidation value="True"/>
  </Validation>
  <Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Created by
```xml
<Field id="3" dataType="Text" fieldType="Normal" name="CreatedBy">
  <Comment>Account name of the user who created each record</Comment>
  <AutoEnter allowEditing="False" value="CreationAccountName" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <NotEmpty value="True"/>
    <Unique value="False"/>
    <Existing value="False"/>
    <StrictValidation value="True"/>
  </Validation>
  <Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Modification timestamp
```xml
<Field id="4" dataType="TimeStamp" fieldType="Normal" name="ModificationTimestamp">
  <Comment>Date and time each record was last modified</Comment>
  <AutoEnter allowEditing="True" value="ModificationTimeStamp" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <StrictDataType value="FourDigitYear"/>
    <NotEmpty value="True"/>
    <Unique value="False"/>
    <Existing value="False"/>
    <StrictValidation value="True"/>
  </Validation>
  <Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Modified by
```xml
<Field id="5" dataType="Text" fieldType="Normal" name="ModifiedBy">
  <Comment>Account name of the user who last modified each record</Comment>
  <AutoEnter allowEditing="True" value="ModificationAccountName" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <NotEmpty value="True"/>
    <Unique value="False"/>
    <Existing value="False"/>
    <StrictValidation value="True"/>
  </Validation>
  <Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Plain text field
```xml
<Field id="6" dataType="Text" fieldType="Normal" name="field_name">
  <Comment/>
  <AutoEnter allowEditing="True" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
  <Validation messageCalc="False" message="False" maxLength="False" valuelist="False" calculation="False" alwaysValidateCalculation="False" type="OnlyDuringDataEntry">
    <NotEmpty value="False"/>
    <Unique value="False"/>
    <Existing value="False"/>
    <StrictValidation value="False"/>
  </Validation>
  <Storage autoIndex="True" index="None" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Unstored calculation
```xml
<Field id="7" dataType="Text" fieldType="Calculated" name="field_name">
  <Calculation table="table_name"><![CDATA[expression]]></Calculation>
  <Comment/>
  <AutoEnter alwaysEvaluate="False"/>
  <Storage storeCalculationResults="False" indexLanguage="English" global="False" maxRepetition="1"/>
</Field>
```

### Stored calculation — indexed All
```xml
<Field id="8" dataType="Text" fieldType="Calculated" name="field_name">
  <Calculation table="table_name"><![CDATA[expression]]></Calculation>
  <Comment/>
  <AutoEnter alwaysEvaluate="False"/>
  <Storage storeCalculationResults="True" index="All" indexLanguage="Unicode_Raw" global="False" maxRepetition="1"/>
</Field>
```

### Summary — Total
```xml
<Field id="9" dataType="Number" fieldType="Summary" name="field_name">
  <SummaryInfo restartForEachSortedGroup="False" summarizeRepetition="Together" operation="Total">
    <SummaryField>
      <Field id="1" name="source_field_name"/>
    </SummaryField>
  </SummaryInfo>
  <Comment/>
  <AutoEnter allowEditing="True" constant="False" furigana="False" lookup="False" calculation="False">
    <ConstantData/>
  </AutoEnter>
</Field>
```

---

## 13. Known gaps

The only points not fully pinned down. None blocks generation.

- **`StrictDataType` values**: `"Numeric"`, `"FourDigitYear"`, `"TimeOfDay"` confirmed.
  Other Date/Time values may exist but are unobserved.
- **Deliberate element reordering**: child order is confirmed for all observed cases; FM's
  tolerance of intentionally reordered elements is untested.

Calculation fields carry no `<Validation>` block: validation options are not user-settable
on calcs (Claris Help) and none is emitted.

---

## 14. FileMaker 2026 (v26): field-level changes

FileMaker 2026 (internal version 26, released 10 June 2026) adds two field-level elements
to the format: `<Annotation>` and `<DisplayNames>`. Both round-trip on Normal, Calculation,
and Summary fields in empty and populated states. Emit them when targeting 2026.

Position by field type:
- Normal fields: after `<Storage>` (and after `<Furigana>` when present).
- Calculation fields: after `<Storage>`.
- Summary fields: after `<AutoEnter>` (summary fields have no `<Storage>`).

`<Annotation>` precedes `<DisplayNames>` in all cases.

### 14.1 Field annotation (AI / DDL description)

A per-field description aimed at AI models, configured in the Advanced Options for Field
dialog under "Add annotation in Data Definition Language (DDL)". It is separate from
`<Comment>` and does not replace it. At runtime it is exposed by the `FieldAnnotation()`
function (originated v26).

`<Annotation>` wraps a single `<Text>` child holding plain text (not CDATA, not JSON).
Standard XML escaping applies and round-trips intact, confirmed with `&`, `<`, `>`,
quotes, non-ASCII characters, and a `&#13;` line break.

Empty:
```xml
<Annotation>
  <Text/>
</Annotation>
```

Populated:
```xml
<Annotation>
  <Text>Customer reference number used by the finance team</Text>
</Annotation>
```

**Two DDL behaviours (not paste-format, but relevant when writing annotations):**

1. **Annotations narrow DDL scope table-wide.** Once any field in a table has an
   annotation, only annotated fields appear in generated DDL; the rest are excluded.
2. **`[LLM]` comment bridge.** If annotation is blank and `<Comment>` begins with `[LLM]`,
   FM uses the comment text (minus the tag) as the annotation in DDL.

### 14.2 Custom field display name

A display name distinct from the schema name, surfaced in native operations. Configured
via "Customize field display names" in the Advanced Options for Field dialog. Exposed at
runtime by `FieldDisplayNames()` (originated v26).

`<DisplayNames>` carries an `enable` attribute. When enabled it holds a `<Calculation>`
child (same shape as an auto-enter calculation: a `table` attribute and a CDATA body) that
returns a JSON object of display-name keys. It is a calculation, not a stored string, so
display names can be dynamic. Any expression returning the JSON object is valid;
`JSONSetElement` is the canonical form the Specify dialog produces.

Inactive:
```xml
<DisplayNames enable="False"/>
```

Active:
```xml
<DisplayNames enable="True">
  <Calculation table="table_occurrence_name"><![CDATA[JSONSetElement ( "{}" ;
  [ "fm_common" ; "Customer Ref" ; JSONString ] ;
  [ "fm_export" ; "Customer Reference (Export)" ; JSONString ] ;
  [ "fm_sort" ; "Cust Ref" ; JSONString ] ;
  [ "fm_table_view" ; "Ref" ; JSONString ]
)]]></Calculation>
</DisplayNames>
```

The four built-in context keys (Claris Help, Defining advanced field options):

| Key | Context |
|---|---|
| `fm_common` | Fallback for all features below, unless a more specific key is set |
| `fm_export` | Specify Field Order for Export dialog and exported file headers |
| `fm_sort` | Sort Records dialog |
| `fm_table_view` | Table View column header |

Specific key beats `fm_common`; if neither is set for a feature, the field name is used.
Custom keys are allowed (read back via `FieldDisplayNames()`); the `fm_` prefix is reserved.

The `<Calculation>` is stored verbatim, including empty-string keys, and round-trips
unchanged. The sparse return documented for `FieldDisplayNames()` (only keys with values)
is runtime output, not what is stored.

### 14.3 Generation guidance

- **2025 or mixed targets:** omit both elements. v26 normalises the empty state in on
  paste; v22 has no knowledge of them.
- **2026 targets:** emit both. Empty state is `<Annotation><Text/></Annotation>` and
  `<DisplayNames enable="False"/>`; populated forms as in §14.1 and §14.2.
- Annotating any field narrows the table's DDL to annotated fields only (§14.1).

### 14.4 Out of scope

2026 calculation-controlled field entry (editable / non-editable / read-only via
calculation) is a **layout object** property (Layout Inspector, Data pane), not a field
definition. See the companion Layout XML spec.

---

## 15. Table-level paste: the `<BaseTable>` envelope

New in v2.0. The same `FMObjectList` envelope, pasted on the **Tables tab** of Manage
Database, creates complete tables — fields included — in one operation.

Corroboration: four independent production exports on the copy side (tables from three
different solutions, including a 300+ field production table), one generated 12-field
table round-tripped byte-identical on v26 (151 elements, zero drift), and eight
repeat-paste collision runs. Items not yet paste-tested are listed in §15.6.

### 15.1 Envelope

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
inside are **identical** to the bare-field format in §2–§10 and §14: the table envelope
is a thin wrapper, not a different format. Every mechanism tested — calc auto-enter with
context table, system values, serial, StrictDataType, NotEmpty/Unique/StrictValidation,
all index levels including the `autoIndex` omission at `index="All"`, Unicode_Raw,
stored and unstored calcs, Total summary — survives the envelope at full fidelity in
both directions.

Multiple `<BaseTable>` siblings in one snippet paste in a **single operation** — one
paste on the Tables tab creates every table in the snippet, each with its fields intact
and per-table IDs preserved. Round-trip confirmed with a two-table snippet, matching
FM's own multi-table copy format. Whole-schema scaffolds are a single paste.

**XML comments are tolerated by the Tables tab handler.** A snippet containing an
`<!-- -->` comment inside the envelope pasted successfully, comment stripped, table and
fields intact — unlike the Fields tab handler, which fails or pastes partially (§11
rule 7). Same envelope, third parser behaviour, after the script handler's tolerance
and the field handler's rejection. Generate comment-free regardless, so snippets stay
valid for every handler.

### 15.2 Forward references resolve

A calculation field may reference a field defined **later** in document order. A stored
calc `quantity * unit_price` with `unit_price` defined after it pasted and re-exported
intact. The handler resolves field names after all fields in the table exist, so
generated XML needs **no dependency ordering**. List fields in whatever order reads best.

### 15.3 Field IDs

- Into a fresh table, supplied IDs are preserved as written — sequential 1..n lands
  as 1..n, and internal references (Summary `<SummaryField><Field id=.../>`) land intact.
- ID uniqueness is scoped **per table**. Sibling `<BaseTable>` elements each start at
  `id="1"`; this is FM's own multi-table copy format.
- Generate sequential integers from 1 within each table.

### 15.4 Name collisions: silent rename, context rebound

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

### 15.5 Serial `nextValue` is honoured

`<Serial ... nextValue="1001"/>` pastes as next value 1001, exactly as written. This
makes table paste a serial-continuity-preserving migration path — and a generation
footgun. Never copy `nextValue` from an example: default it to `1` unless the user
specifies otherwise. Text-field serials with alphanumeric `nextValue` (e.g. `"J114071"`)
are export-observed; the numeric tail increments.

### 15.4a Table paste creates a graph occurrence

Every pasted table gets exactly one table occurrence on the relationships graph, named
identically to the table, standalone with no relationships. Observed across all three
paste variants: single table, collision rename (the collision pastes produced
occurrences "paste_test 2" through "paste_test 8"), and multi table single paste —
the same behaviour as creating a table manually. Generated schema scaffolds therefore
arrive graph-ready: occurrences exist for scripting and layout context immediately, and
relationship wiring is the only manual step.

### 15.5a Invalid calculations are neutralised, not rejected

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

### 15.6 Verification status

Every behaviour in §15 is paste-verified on v26. The only export-observed item not yet
independently generation-tested anywhere in this spec is the `"ModificationName"`
auto-enter value (§5.2), which is symmetric with the confirmed `"CreationName"`.

### 15.7 Copy-side fossils (analysis, not generation)

Production table exports faithfully preserve broken state rather than cleaning it.
Expect, and never generate:

- `valuelist="True"` with no `<ValueList>` child — a dangling reference in the source
  file (consistent with §7.5, which describes the paste-side drop). When the reference
  resolves, table copy **does** emit `<ValueList>`.
- `lookup="False"` (or `message="False"`, `maxLength="False"`, `valuelist="False"`) with
  the corresponding child element still present — the option was configured then
  disabled, and FM keeps the child. Which side the paste handler believes is untested.
- Inactive `<Furigana>` with a null reference (`<Field id="0" ... name=""/>`) — see §10.
- Internal base table IDs leaking through `<Lookup><Table id="1065xxx" .../>` — the only
  place FM's internal table ID space appears in this format. Whether lookups rebind by
  id or by name on paste is untested.
- `table=""` on `<Calculation>` — broken evaluation context (§6.1).

---

*Clockwork Creative Technology — clockworkct.co.uk*
