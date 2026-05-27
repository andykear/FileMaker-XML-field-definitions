# Canonical XML Format for FileMaker Field Definitions (XMFD)

**Author:** Andrew Kear, Clockwork Creative Technology
**Version:** 0.6
**Date:** May 2026
**Verified against:** FileMaker Pro 2025 on macOS
**Licence:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)

---

## Introduction

FileMaker (FM) exposes an undocumented clipboard format for field definitions that allows
fields to be created in bulk by pasting XML directly into Manage Database. The format uses
the same `fmxmlsnippet type="FMObjectList"` envelope as the Script Workspace clipboard
format, but contains `<Field>` elements rather than script steps.

This capability is genuinely useful — it enables automated schema assembly, solution
scaffolding, and schema migration. It requires MBS Plugin to be installed in FileMaker
Pro, but no MBS scripting is needed. Despite this, the format has never been documented
by Claris.

This spec is reverse-engineered from production solution exports and validated by
round-trip testing. It covers every field type, every auto-enter mechanism, all validation
options, and container external storage. It is published as a community resource for
developers who need to generate or manipulate field definitions programmatically.

**Related:** The companion [FileMaker Script XML Spec](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill)
documents the equivalent format for Script Workspace clipboard XML.

---

## Quick start

Requires MBS Plugin to be installed in FileMaker Pro.

1. Write or generate XML conforming to this spec.
2. Open the file in a text editor, select all, copy.
3. In FileMaker Pro, open Manage Database and select the target table on the Fields tab.
4. Paste. FM creates the fields and assigns real internal IDs.

Fields are added to the table in the order they appear in the XML. FM resolves Summary
field source references and Lookup source fields by name at paste time.

---

## 1. Envelope

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmxmlsnippet type="FMObjectList">
  <!-- one or more <Field> elements -->
</fmxmlsnippet>
```

Identical envelope to script XML. 2-space indent throughout. UTF-8 encoding.
XML comments are permitted — the paste handler ignores them.

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
<!-- <Furigana/> only present when Furigana is configured -->
```

### Calculation field

```
<Calculation/>    <!-- ALWAYS FIRST -->
<Comment/>
<AutoEnter/>      <!-- simplified form, no sub-elements -->
<Storage/>
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
| `"CreationAccountName"` | Creation → Name |
| `"ModificationDate"` | Modification → Date |
| `"ModificationTime"` | Modification → Time |
| `"ModificationTimeStamp"` | Modification → Timestamp |
| `"ModificationAccountName"` | Modification → Name |
| `"ConstantData"` | Data (constant) |
| `"PreviousRecord"` | Value from last visited record |

Absent when no system value is active.

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
| `alwaysValidateCalculation` | `"True"` / `"False"` | Only `"False"` seen in exports |
| `type` | `"OnlyDuringDataEntry"` / `"Always"` | When validation fires |

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

<Calculation table="table_name"><![CDATA[validation expression]]></Calculation>

<MaxDataLength value="255"/>

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

| XML value | UI label | Round-trip confirmed |
|---|---|---|
| `Total` | Total of | Yes |
| `Average` | Average of | Yes |
| `Count` | Count of | Yes |
| `Minimum` | Minimum | No |
| `Maximum` | Maximum | Yes |
| `StdDeviation` | Standard Deviation of | No |
| `Fractional` | Fraction of Total of | No |
| `RunningTotal` | Total of + Running total | No |
| `WeightedAverage` | Average of (weighted) | No |
| `List` | List of | Yes |

### 9.3 `<AdditionalField>`

Required for `WeightedAverage` (weight field) and for `RunningTotal` when
`restartForEachSortedGroup="True"` (sort field):

```xml
<SummaryInfo restartForEachSortedGroup="True" summarizeRepetition="Together" operation="RunningTotal">
  <SummaryField>
    <Field id="2" name="value_field"/>
  </SummaryField>
  <AdditionalField>
    <Field id="1" name="sort_field"/>
  </AdditionalField>
</SummaryInfo>
```

---

## 10. `<Furigana>`

Furigana is a Japanese-locale feature that auto-populates one field with the phonetic
reading of another as the user types. It is only relevant in Japanese-locale FileMaker
installations. Most developers can ignore this section entirely.

The element is only emitted when Furigana is configured on the field. Position: last
child of `<Field>`, after `<Storage>`.

**Inactive state** — present in exports but with empty `inputMode`:
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

When active, `furigana="True"` on `<AutoEnter>` and the inner `<Field>` child is
populated. The inner `<Field>` uses a `baseTable` attribute not seen in other contexts.

`inputMode` confirmed values:

| UI label | XML value |
|---|---|
| (inactive) | `""` |
| As is | `"AsEntered"` |
| Hiragana | `"Hiragana"` |
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

- **`StrictDataType` values** — `"Numeric"`, `"FourDigitYear"`, and `"TimeOfDay"`
  confirmed. Other values likely exist for Date and Time fields but have not been
  observed in exports.
- **`alwaysValidateCalculation="True"`** — never seen in exports.
- **Calculation field `<Validation>` presence** — absent in most Calculated field
  exports, but the rule governing when it appears has not been established.
- **`Remote type` values** — `"Open"` and `"Secure"` confirmed; other values may exist.
- **`summarizeRepetition="Individually"`** — valid in the FM UI but not yet round-trip
  confirmed.
- **Furigana active state on paste** — inactive Furigana (`inputMode=""`) confirmed;
  paste behaviour when Furigana is actively configured has not been tested.
- **Child element ordering** — ordering is derived from export analysis and assumed
  correct, but has not been tested by deliberately reordering elements.

---

## Contributing

Errors, omissions, and additional confirmed variants are welcome via GitHub Issues at
the repository where this spec is published. If you have access to FileMaker exports
that cover unverified items in §13, please open an issue or pull request.

---

*Clockwork Creative Technology — clockworkct.co.uk*
