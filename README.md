# FileMaker Schema Definition XML Specification

[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-XML-field-definitions?style=social)](https://github.com/andykear/FileMaker-XML-field-definitions)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

Reverse-engineered specification of FileMaker's undocumented schema definition XML, across all three definition catalogs: fields, tables, and value lists. Every element ordering rule, collision behaviour, reference-resolution rule, and silent failure mode, established through empirical round-trip testing against production FileMaker solutions.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## Why this exists

FileMaker's Manage Database and Manage Value Lists dialogs accept pasted schema in specific XML formats. Claris has never published a specification for any of them. Get the XML wrong and FileMaker accepts it silently, creating fields with the wrong settings, dropping elements, or disabling a calculation, with no error message.

This specification documents what the paste handlers actually accept across three catalogs: field definitions on the Fields tab, table definitions on the Tables tab, and value list definitions in Manage Value Lists. Every ordering constraint, the collision and rename behaviour, and the reference-resolution rules, established through round-trip testing: generate the XML, paste it, export it back out, diff the two, repeat until they match.

## What it catches

Classes of silent failure that are invisible after paste, each documented with the rule that avoids it:

- **Invalid calculations are neutralised, not rejected** — the field looks normal and does nothing.
- **Fossils install invisibly** — a disabled value list arm, lookup, or validation option rides its stale data into the target.
- **Unresolvable references install as dead bindings** — a missing name goes dead (silently for fields; a missing external source is the one loud exception).
- **Name collisions rename and rebind** — a clashing table or field is silently suffixed and its references rewritten to follow.
- **Element and dependency traps** — duplicate field IDs drop calculations, Furigana drops on an unresolved value list, a pasted serial `nextValue` becomes the live counter.

These are the default failure mode when any tool, human or AI, generates FileMaker schema XML without knowing the paste handler rules.

## Packaged as a Claude Skill

The specification is packaged as a Claude skill with one reference file per catalog. The model reads the core rules plus only the catalog spec each task needs, field, table, or value list, keeping token usage proportional to the task.

Once installed, Claude uses the specification automatically when generating or reviewing FileMaker schema XML. The structural rules are handled deterministically. The model handles the logic.

Tested with Claude, and model-agnostic by design: the reference files are plain markdown, readable by any LLM or human. If you're building tools that generate FileMaker schema XML, the spec is yours to integrate.

## Installation

Download the release zip and load it into Claude's skills. The zip contains the skill folder with SKILL.md and the references/ folder at its root, keep that structure intact. Claude reads SKILL.md and the skill becomes available, toggled on or off like any other. It needs code execution and file creation enabled so it can read the reference files.

Exact steps vary by where you use Claude (desktop, web, or Claude Code) and change over time, so follow Anthropic's current instructions for adding a skill.

## What's in the box

```
SKILL.md                            — Claude skill definition + routing across catalogs
references/
  filemaker_xmfd_spec.md            — Field definitions (XMFD): six data types, three
                                      field types, auto-enter, validation, storage,
                                      summaries, FM 26 annotation and display-name elements
  filemaker_xmtb_spec.md            — Table definitions (XMTB): the BaseTable envelope,
                                      collision rename and rebind, forward references,
                                      serial nextValue, per-table ID scope, graph occurrence
  filemaker_xmvl_spec.md            — Value list definitions (XMVL): custom, field, and
                                      external source arms, separators, show/sort, the
                                      fossil rule, dangling handling, by-name resolution
```

## Usage

Once installed, Claude applies the specification automatically when you ask for FileMaker fields, tables, or value lists. No special prompt needed.

Generate a table:

"Make me a Contacts table with a UUID primary key, audit stamps, and a status field"

Generate a value list:

"Make me a Status value list with Active, Inactive, Pending for Manage Value Lists"

Review existing XML:

Paste your fmxmlsnippet and ask Claude to review it for paste handler errors

With schema context:

Attach your Save as XML (SaXML) export and Claude will use real field, table, and value list names from your solution. You can also attach one exported from Clockwork Inspector.

## Pasting into FileMaker

Manage Database and Manage Value Lists accept fmxmlsnippet type="FMObjectList" via clipboard paste in FileMaker's internal clipboard format, not plain text. Paste field snippets on the Fields tab with the target table selected, whole tables on the Tables tab, and value lists in Manage Value Lists. FileMaker creates everything and assigns real internal IDs.

Getting the XML onto the clipboard in that format is the MBS Plugin's job: it writes the snippet under FileMaker's private type code — XMFD for field definitions, XMTB for tables, and XMVL for value lists. Tested with the MBS Plugin installed; no MBS scripting is needed, it just needs to be present. Many alternatives are available in the community, such as FmClipTools.

## The FileMaker XML suite

One of a set that reverse-engineers FileMaker's clipboard format family end to end — the private type codes FileMaker uses to carry schema and objects through the clipboard. The three generation specs cover all seven codes between them; two tools support the workflow.

**[Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill)** (XMSS, XMSC, XMFN)
The full script step ID dictionary, plus the hidden paste-handler rules that decide whether your XML survives the trip into FileMaker.

**[Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)** (XML2)
All 18 layout object types mapped, every flag decoded, element order confirmed against native output. Verified across 45+ layouts in 10 production files.

**[Field, Table & Value List Definitions](https://github.com/andykear/FileMaker-XML-field-definitions)** (XMFD, XMTB, XMVL) — this repo
Field, table and value list definition XML — auto-enter, validation, storage, calculation options, and the three value list source arms — verified down to the individual option level.

**Analysis — read, audit and clean existing XML**

**[XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source)** (SaXML)
Full-catalog dependency analysis of a Save as XML export, entirely in the browser. Finds unreferenced fields, silent-failure risks, broken references, and diffs two versions of a solution against each other.

**[XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber)** (SaXML + others)
Strips API keys, passwords and internal hostnames out of FileMaker XML before you hand it to an AI tool.

## Licence

CC BY 4.0 — free to use, share, and adapt with attribution.

## Contributing

Found something that doesn't round-trip? A production export that contradicts the spec? Open an issue or PR.

## Version history

| Version | Notes |
|---|---|
| 2.1 | Value list definitions added as a third catalog (filemaker_xmvl_spec.md), and the reference split into one spec per catalog. Value list paste round-trip verified on v26 across all three source arms: handler exists, id reassigned, collision rename, LF separator functional, references resolve by name with id self-heal, fossil install, dangling install as dead binding. Clipboard type codes documented as transport. |
| 2.0 | Table-level paste. The BaseTable envelope round-trip verified on v26, byte identical: whole tables and multi-table schemas paste from the Tables tab at full fidelity, each with a graph occurrence. Forward references resolve, serial nextValue honoured, per-table ID scope, collision rename with context and reference rebinding, invalid calculation neutralisation documented. Copy-side fossil catalogue for analysing production exports. |
| 1.0 | FileMaker 2026 (v26) verified. Annotation and display name elements (display names confirmed as a JSON-returning calculation), full 13-operation summary set, calculated validation messages, container external storage, lookup, and Furigana all round-trip confirmed on v26. |
| 0.6 | FileMaker 2025 (v22) baseline. All field types, auto-enter, validation, storage, and a 74-field validation suite. |

---

*Clockwork Creative Technology — clockworkct.co.uk · github.com/andykear. Bespoke FileMaker development, automated artwork systems, and hosted solutions. Working on something and need a hand? Get in touch.*
