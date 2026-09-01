---
name: harmonyos-doc-generator
description: Use when documenting a HarmonyOS application or industry in Huawei's official industry-guide format - an application case, key scenario documents, a scenario index and runnable per-scenario source. Works with or without access to the application's source code. Also triggers on "dokuman uret", "generate docs", "document this app".
---

# HarmonyOS Industry Documentation Generator

Produces the same document set Huawei publishes for its own industry guides:
one application case, one document per key scenario, a scenario index, and a
runnable minimal project for each scenario.

Section names and ordering mirror the official guides exactly, in English.

## The source of scenarios is the industry skill

Not the application. `harmonyos-<industry>-industry` carries the capabilities
Huawei treats as key scenarios for the vertical, and every card holds verified
code lifted from a compiled sample project with the published document's
defects already corrected.

That is what makes this work **without the application's source code**, which
matters on machines where the code cannot leave the internal network.

## Which mode you are in

| | Industry skill | Application source | What changes |
|---|---|---|---|
| **Mode 1** | required | none | scenarios and code come entirely from the skill's cards |
| **Mode 2** | required | available | same selection, but the application's own code is used where it implements the scenario |

Mode 2 aligns with the application; it does not depend on it. A scenario worth
documenting stays in the set even when the application has not built it. Never
drop one only because the application lacks it, never add one only because it
has it.

If `harmonyos-<industry>-industry` is not installed, say so and stop. It is
the only required input.

## What this needs from the user

Ask for anything not already stated, then stop and wait:

- **Industry** - education, tourism, auto, food, and so on
- **Application name** - goes in the case document title
- **Target API level** - appears in every Constraints section
- **Whether the application's source is available** - decides the mode

## Two rules that override everything

**Source rule.** Every code block comes from verified code: a card's
**Verified snippets** in Mode 1, the application's own source in Mode 2. Never
write fresh code for a document. Never guess an API name - find it, or delete
the sentence. Never invent a screenshot; leave a placeholder.

**Independence rule.** Every ArkUI component, kit and system API named in a
document links to its official reference page. A sentence you cannot back with
an official page is probably a product-specific detail that does not belong in
the document. Exact link format is in `references/document-templates.md`.

Output is English throughout, including when the conversation is not.

## Workflow

### Step 0 - select scenarios, then stop

Read `references/scenario-selection.md` and apply it. It sets a hard budget of
10 to 18 scenarios, four gates every candidate must pass, a ranking for the
survivors, and a cap of three per capability cluster.

In Mode 2, read the application first: module structure from
`build-profile.json5` and each `oh-package.json5`, every `.ets` and `.ts`
under the source roots, permissions and abilities from every `module.json5`,
routing from `route_map.json` and `Navigation` usage.

Then report and **wait for approval**:

- the selected scenarios, with cluster and reason
- the candidates rejected, with the gate or ranking that removed them
- in Mode 2, the module list and the kits and permissions in use

Selection decides the quality of the whole set. Never skip this stop, and
never start writing while the count is still above 18.

### Step 1 - write the documents

Follow `references/document-templates.md` for all three document types.
Application case first, then scenarios in index order, then the index.

### Step 2 - build the scenario source

For each scenario, the smallest project under `docs/source_code/<NN>_source_code/`
that runs it alone - `<NN>` is the scenario's own two-digit number, so
`03-course-reading-progress.md` is served by `source_code/03_source_code/`.
Entry module only. No file the scenario does not need. Only the permissions it
actually exercises.

Moving code from a multi-module application into a single entry module is not
a copy. Resource resolution is module-scoped - `$rawfile` resolves against the
calling module's resources, not another HAR's. State the adaptation you made
in the document's Implementation approach section, and build the project
before calling the scenario done. A scenario whose source does not compile is
not finished.

### Step 3 - verify

Walk this list and answer each item:

1. Is the scenario count between 10 and 18?
2. Does every scenario document have all seven sections, in order?
3. Does every code block come from verified code, unedited except for marked
   cuts?
4. Does every link in `key-scenario-examples.md` resolve to a real file?
5. Does each scenario's `source_code/` tree match its Project directory
   section, and does it compile?
6. Is there exactly one `<NN>_source_code` directory per scenario document, with
   the numbers running unbroken from `01`, and does every number match its
   document?
7. Any abbreviated `.../slug` links left?
8. Any sentence not in English?
9. Does every named component and API link to its official reference page?
10. Any internal name - class, module, screen - in a scenario title?
11. In Mode 2, any documented capability with no counterpart in the code or in
    a card?

Close with a summary: how many documents, how many scenarios, which were
rejected and why, and which screenshot and diagram placeholders still need
filling by hand.

## Output structure

```
docs/
├── main-industry-doc.md
├── 01-<scenario-slug>.md
├── 02-<scenario-slug>.md
├── key-scenario-examples.md
└── source_code/
    ├── 01_source_code/
    └── 02_source_code/
```

The application case is always `main-industry-doc.md` and carries no number -
it is not one of the key scenarios and must not be counted as one.

**Scenarios are numbered from `01`**, in the order of
`key-scenario-examples.md`, so the highest number is the scenario count. A set
that starts at `02` makes the last number disagree with the total, which is the
first thing a reader checks.

Each scenario's runnable project is `source_code/<NN>_source_code/`, numbered to
match its document: `07-read-aloud-practice.md` is served by
`source_code/07_source_code/`. The number is the whole mapping - do not put the
slug in the directory name as well.

Scenario document slugs are lowercase, hyphenated, four words at most.

## Files

| File | What it holds |
|---|---|
| [references/scenario-selection.md](references/scenario-selection.md) | the two modes, the count budget, the gates, the ranking and the spread rule |
| [references/document-templates.md](references/document-templates.md) | section-by-section skeletons for all three document types |
| [references/scenario-index.md](references/scenario-index.md) | 406 scenarios across 19 industries - the granularity and naming reference |
