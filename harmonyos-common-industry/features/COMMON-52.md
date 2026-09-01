---
id: COMMON-52
title: Industry FAQ redirect - the common-technical-solutions FAQ has moved to the HarmonyOS FAQ
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/52_practice-common-app-architecture-v1_35.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_35-0000002263801938
sample: none
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: [HW-19-0178, HW-19-0179]
status: verified-with-fixes
---

## When to use

Do not load this card for implementation guidance - there is none here. Load it
only to know **where the industry FAQ content went** and why looking for it in
this industry's document set is a dead end.

This is an architecture/overview entry with no sample project. The whole document
is one sentence:

> 行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
> ("The industry FAQ content has been migrated to the HarmonyOS FAQ; please click
> here to go there.")

## Feature checklist

Not applicable - this document describes no feature. What it establishes:

- The `19_common_technical_solutions` industry has **no FAQ document of its own**;
  it was removed and replaced by a link.
- The link target is the generic HarmonyOS phone FAQ, shared with all other
  industries (HW-19-0179).
- The target is **outside the local documentation mirror** - `documentation/`
  holds only `harmonyos-guides` and `harmonyos-references`, with no
  `harmonyos-faqs` tree - so this content cannot be consulted from this
  repository.

## Architecture

There is no project to describe. What matters structurally is the document's
place in the set.

**The industry has 52 documents; 51 of them are scenarios.** Each of those
carries the same eight sections - 场景介绍, 效果预览, 实现思路, 约束与限制,
权限说明 (where applicable), 工程目录, 参考文档, 代码下载 - and each ships a
downloadable sample. This one has none of them and no archive under `downloads/`
(HW-19-0178).

**It is the industry's tail entry by convention.** Every industry in the tree ends
with a `practice-<industry>-app-architecture` document, and in all nineteen of
them that document is now this same stub:

| industry | file |
| --- | --- |
| `01_auto` | `08_practice-auto-app-architecture-v1_2.md` |
| `02_convenient_life` | `31_practice-convenient-life-app-architecture-v1_2.md` |
| `03_sports_health` | `15_practice-sports-health-app-architecture-v1_2.md` |
| `04_education` | `21_practice-educate-app-architecture-v1_2.md` |
| `05_office` | `32_practice-office-app-architecture-v1_2.md` |
| … | … |
| `19_common_technical_solutions` | `52_practice-common-app-architecture-v1_35.md` |

Nineteen distinct source URLs, nineteen industry-specific titles, and one shared
destination (HW-19-0179).

**What that means in practice.** Questions that the scenario documents raise -
which API level a capability needs, why a permission is restricted, what a device
must support - have no industry-level answer document. They must be resolved
against the reference and guide pages each scenario cites, which is what the other
51 cards in this set do.

## Implementation steps

Not applicable. If you arrived here looking for FAQ content:

1. **For an API question**, go to the reference page the relevant scenario
   document cites - each card in this set lists them under References, and the
   local mirror at `documentation/harmonyos-references/` usually holds the page.
2. **For a how-to question**, go to `documentation/harmonyos-guides/`.
3. **For a genuine FAQ**, follow the link to
   `https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone`,
   knowing it is not scoped to this industry.

## Verified snippets

None. There is no sample project for this document, and the document contains no
code.

The complete body, quoted from
`huawei_industry_tree/19_common_technical_solutions/docs/52_practice-common-app-architecture-v1_35.md`:

```markdown
# 行业常见问题

> Kaynak: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_35-0000002263801938

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

## Permissions & config

Not applicable - no sample, no manifest, no permissions.

## Constraints

- **No sample.** The document declares `images: 0` in its front matter and has no
  `代码下载` section; no archive corresponds to it under
  `huawei_industry_tree/19_common_technical_solutions/downloads/`.
- **No version floor.** Unlike every scenario document in this industry, there is
  no 约束与限制 section, so no API level, SDK or DevEco Studio version is stated.
- **The destination is not mirrored locally.** `documentation/` contains
  `harmonyos-guides` and `harmonyos-references` only.
- **The destination is not industry-specific.** It is the same page all nineteen
  industries link to.

## Pitfalls

- **This entry is listed like a scenario document but contains none of the
  sections one has, which is incorrect for the listing it appears in** - a reader
  cannot tell it is a redirect until they open it. (HW-19-0178)
- **All nineteen industry FAQ redirects share one generic destination, which is
  incorrect** - the industry dimension the titles promise is lost at the link.
  (HW-19-0179)
- **Do not treat the absence of an industry FAQ as an absence of constraints.**
  The per-scenario 约束与限制 and 权限说明 sections are where those live in this
  industry, and the other 51 cards in this set record them.
- **Do not cite this document as evidence for anything.** It contains one
  sentence and one link; there is no statement here to reason from.

## References

- The redirect target:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone
- The document's own source:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_35-0000002263801938
- The industry's scenario documents, which carry the constraints and permissions
  this page does not: `huawei_industry_tree/19_common_technical_solutions/docs/`
  01 through 51, covered by cards COMMON-01 through COMMON-51.
