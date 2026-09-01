# Document templates

Three document types. Section names and ordering mirror Huawei's official
industry guides; the mapping is at the end of this file.

---

## 1. Application case

`docs/main-industry-doc.md`

The name is fixed and carries no number. The application case is not a key
scenario, so it never takes a scenario number - the scenarios start at `01`.

```
---
title: Education Hub Application Case
industry: education
min_api: 20
---
```

### Introduction

What the application is and who it serves. Then three bullets: the
development model (Stage plus declarative UI), the device form factor and HAP
plan, and the package type chosen for business modules with the reason for
that choice.

### Application layout

The bottom navigation tabs and what each one does. Leave a placeholder
instead of a screenshot:

```
<!-- screenshot: home tab -->
```

### Application architecture design

**Module division** - a table, on the principle of high cohesion and low
coupling:

| Module | Responsibilities |
|---|---|

**Software view** - three layers, one paragraph each:

- Product customisation layer - the HAP, page framework, navigation
- Basic feature layer - business modules, HAR
- Common capability layer - account, network, DFX, HAR

```
<!-- diagram: software view -->
```

### Logical view design

How modules depend on each other and on system services.

```
<!-- diagram: logical view -->
```

### Industry key technical solutions

The techniques that distinguish this industry. Three sub-headings each:

- **Feature design** - what it does, what the user sees
- **Solution design** - which component and API, and why that choice
- **Code reference** - a link to the scenario document

### Application framework code

Include this note verbatim:

> This document does not contain the full application code. Only the
> framework code is included.

If parts of the application are deliberately incomplete - authentication that
is UI only, stub utility classes - say so plainly here.

### Code running environment

DevEco Studio version, SDK version, API level, devices tested.

---

## 2. Key scenario

`docs/<NN>-<scenario-slug>.md`, numbered from `01` in index order.

```
---
title: Channel Selection
industry: education
min_api: 20
source_code: source_code/01_source_code
---
```

`source_code` names the directory for **this** scenario's number, so the
frontmatter of `01-channel-selection.md` points at `source_code/01_source_code`.

Seven sections, exactly these, in this order.

### Scenario introduction

One or two sentences: what is built and with which component. Link the main
component to its official reference page inline, so the reader does not have
to scroll to Reference documents.

### Effect preview

```
<!-- screenshot: channel edit sheet -->
```

### Implementation approach

Numbered steps. Each step is one technical move, with a `ts` code block
beneath it when it needs one.

Code blocks are copied from verified code, never rewritten: the industry
card's **Verified snippets** in Mode 1, the application's own source in
Mode 2. Abbreviate freely, but mark every cut with `// ...`. Add short
English comments on the lines that carry the meaning.

When a card's snippet is marked as corrected, use the corrected form and say
so in the step - that correction is the reason the scenario is worth
documenting.

Three to six steps. More than that means the scenario should be split.

### Constraints and limitations

Bullets: API level, SDK version, DevEco version, device and region limits.
Write the ones that actually apply; do not fill in a template.

### Project directory

A `ts` block holding the tree, with an aligned comment on every file:

```ts
├──entry/src/main/ets
│  ├──common
│  │  └──CommonConstants.ets     // shared constants
│  ├──components
│  │  └──ChannelSelection.ets    // channel picker component
│  └──pages
│     └──HomePage.ets            // application home page
└──entry/src/main/resources      // application resources
```

The tree must match what is actually under `source_code/<NN>_source_code/`.

### Reference documents

This is the section that makes the document independent of your codebase.
Every ArkUI component, kit and system API named anywhere in the document gets
its official reference page here, as a full address:

```
https://developer.huawei.com/consumer/en/doc/harmonyos-references/<page>
https://developer.huawei.com/consumer/en/doc/harmonyos-guides/<page>
```

Rules:

- No bare component names. Writing "using the Grid component" obliges you to
  link `Grid`.
- Never guess an address. If the industry skill is installed, look it up in
  its `references/api-map.md`; otherwise verify the component name against
  the official documentation. If you cannot find it, rewrite the sentence -
  do not publish an invented URL.
- No abbreviated links. A link written as `.../slug` goes nowhere.

### Code download

A relative link to `source_code/<NN>_source_code/`, with the same `<NN>` as the
document.

---

## 3. Scenario index

`docs/key-scenario-examples.md`

Links only. No prose, no code, no descriptions.

```
---
title: Key Scenario Examples
industry: education
---

# Key Scenario Examples

- **[Channel Selection](01-channel-selection.md)**
- **[Font Size Adjustment](02-font-size-adjustment.md)**
- **[Reading Progress Card](03-reading-progress-card.md)**
```

Ordering is covered in [scenario-selection.md](scenario-selection.md).

---

## Mapping to the official guides

| Official section | English heading used here |
|---|---|
| 简介 | Introduction |
| 应用布局 | Application layout |
| 应用架构设计 / 模块划分 | Application architecture design / Module division |
| 软件视图设计 | Software view |
| 逻辑视图设计 | Logical view design |
| 行业关键技术方案 | Industry key technical solutions |
| 应用框架代码 | Application framework code |
| 代码运行环境 | Code running environment |
| 场景介绍 | Scenario introduction |
| 效果预览 | Effect preview |
| 实现思路 | Implementation approach |
| 约束与限制 | Constraints and limitations |
| 工程目录 | Project directory |
| 参考文档 | Reference documents |
| 代码下载 | Code download |

One deliberate difference: Huawei ships scenario code as a downloadable
archive; this set ships it as a `source_code/<NN>_source_code/` folder in the
repository, numbered to match the scenario document it belongs to.
