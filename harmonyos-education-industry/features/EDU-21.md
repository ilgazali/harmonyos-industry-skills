---
id: EDU-21
title: Industry FAQ - a redirect page with no content of its own
industry: 04_education
doc: huawei_industry_tree/04_education/docs/21_practice-educate-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1_2-0000002298147617
sample: none (redirect document)
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-04-0154]
status: verified-with-fixes
---

## When to use

Do not. This card exists so that the industry's document set is completely
accounted for.

`21_practice-educate-app-architecture-v1_2.md` is the 行业常见问题 (industry
common issues) chapter of the education practice, and its entire body is one
sentence saying the content has moved to the HarmonyOS FAQ. There is no
technique here, no code, no sample project.

If you are looking for education-specific troubleshooting, it is not in this
industry's documentation. Use the feature cards `EDU-01` … `EDU-20`, whose
**Pitfalls** sections carry the defects actually found in this industry's
samples - which is the closest thing to an industry FAQ that exists.

## Feature checklist

Nothing to implement.

## Architecture

The education industry has three document kinds. This is the third:

```
EDU-01           the architecture practice + framework code (has a zip)
EDU-02           the index of the 18 feature documents
EDU-03..EDU-20   one feature each (each has its own zip)
EDU-21           THIS - the FAQ chapter, now a redirect
```

The whole document, after the frontmatter and the heading:

```
行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

*(The industry FAQ content has been migrated to the HarmonyOS FAQ; click here to
go there.)*

**The same page exists in all nineteen industries,** with the identical sentence
and the identical destination (`HW-04-0154`) - `09_tourism` has it as document
13, `08_children_education` as 17, `07_finance_insurance` as 11, and so on. The
only per-industry difference is the source URL recorded in the frontmatter.

The destination, `harmonyos-faqs/faq-phone`, is the general phone FAQ, organised
by device and capability rather than by industry - so nothing at the other end is
specific to education either.

## Implementation steps

None.

## Verified snippets

None - the document contains no code.

## Permissions & config

None.

## Constraints

- The document has none of the sections its siblings have: no 场景介绍, no
  实现思路, no 约束与限制, no 工程目录, no 参考文档, no code download.
- `images: 0` in the frontmatter; there are no assets.
- The redirect target lies outside `architecture-guides`, so it is not part of
  the crawled corpus and cannot be reviewed here.
- `EDU-02`, the industry's index page, does not list this document - it is a
  sibling of the architecture practice, not an entry under it.

## Pitfalls

- **`HW-04-0154` — the chapter promises industry-specific questions and
  delivers a link to the general phone FAQ,** identically in all nineteen
  industries. Nothing at the destination is filtered to education, so a reader
  following it has to search the whole FAQ without knowing what they were meant
  to find.
- **Do not treat this document as a gap in the review.** It was read in full;
  there is simply nothing in it.

## References

- `EDU-01` - the architecture practice this chapter belongs to
- `EDU-02` - the index of the industry's eighteen feature documents
- The **Pitfalls** section of every card in this industry - the concrete,
  evidenced problems that a real industry FAQ would collect
- `huawei_industry_tree/09_tourism/docs/13_practice-tourist-park-app-architecture-v1_2.md` - the same page in another industry, for comparison
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1_2-0000002359070945
