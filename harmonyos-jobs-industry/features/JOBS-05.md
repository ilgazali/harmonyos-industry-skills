---
id: JOBS-05
title: Industry FAQ redirect - the jobs FAQ has moved to the HarmonyOS FAQ portal
industry: 12_jobs
doc: huawei_industry_tree/12_jobs/docs/05_practice-jobs-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-jobs-app-architecture-v1_2-0000002424876885
sample: none
kits: []
apis: []
permissions: []
min_api: 17
modules: []
findings: []
status: verified
---

## When to use

Load this card only to **confirm that there is no jobs-specific FAQ content**
and to know where the content went. The page is a single sentence: the industry
FAQ has been migrated to the general HarmonyOS FAQ portal, with a link to the
phone FAQ index.

This matters for expectation-setting more than for development. The jobs
industry practice guide has the same two-page frame as every other industry in
this tree — a key-scenarios index and an FAQ page — but here the FAQ half is
empty. If you are working through the industry systematically and expecting
per-industry troubleshooting, there is none to find; the answers you want are
in the general portal, indexed by API and by kit rather than by industry.

**Verification level here is low, and honestly so.** There is no sample, no
code, and nothing to compile. The only checkable claim is that the redirect
target exists, and it is an external, frequently reorganised portal index —
not a stable document we can pin.

## Feature checklist

What this page promises:

- One sentence stating that 行业常见问题 (industry FAQ) content has been
  migrated to HarmonyOS FAQ.
- One link, labelled 此处 ("here"), pointing at the HarmonyOS FAQ phone index.
- No scenario content, no code snippet, no project tree, no download.

## Architecture

There is no project and, unusually for this tree, no content either. The whole
page after the frontmatter and title is:

```markdown
行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](.../harmonyos-faqs/faq-phone)前往。
```

**The design decision worth noting** is that the redirect is a leaf page rather
than a removal. Huawei kept the FAQ slot in the industry guide's navigation
tree and filled it with a pointer, which means the sidebar structure stays
uniform across all industries even where an industry has nothing of its own to
say. That is good for crawlers and tables of contents and mildly annoying for a
reader who follows the link expecting jobs-specific answers and lands on a
general phone FAQ index.

For our purposes it means this feature id exists to preserve the one-card-per-
document invariant of the skill pack, not because there is a jobs FAQ to
summarise.

## Implementation steps

There is nothing to implement. What to do instead, when you hit a problem in
one of the three jobs samples:

1. **Check the scenario card first.** Every defect we found in this industry is
   recorded with an `HW-12-xxxx` id on `JOBS-02`, `JOBS-03` or `JOBS-04`,
   including the fix.
2. **Then the API reference in this repo**, under
   `documentation/harmonyos-references/`. The three samples between them lean
   on `notificationManager`, `Navigation`/`NavPathStack`, `Stack`,
   `PanGesture`, `animateTo` and `@ObservedV2`/`@Trace`, and every one of those
   pages is present locally.
3. **Then the guides**, under `documentation/harmonyos-guides/` — in particular
   `notification-enable.md`, which carries the `1600004` refusal contract that
   `JOBS-02`'s sample ignores, and the navigation routing guide behind
   `JOBS-04`'s `routerMap`.
4. **Only then the external FAQ portal** this page points at. It is organised
   by device and by kit, so search it by API name, not by "job app".

## Verified snippets

**The whole page** — `huawei_industry_tree/12_jobs/docs/05_practice-jobs-app-architecture-v1_2.md`
(from the doc — no sample shipped; not compile-verified)

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

行业常见问题 is "industry frequently asked questions"; 已迁移至 is "has been
migrated to"; 请点击此处前往 is "please click here to go there". The link text
is the bare word 此处 ("here"), which carries no information about the
destination — worth knowing if you are indexing these documents by link label,
since this one will index as nothing useful.

The target is the **phone** FAQ index specifically, not a jobs-filtered view
and not the FAQ root. There is no anchor and no query, so nothing about the
jobs industry survives the redirect.

## Pitfalls

No defects found in this document during review. It is a correct, if empty,
redirect: the statement matches the link, and the link resolves.

Two things to be aware of rather than defects:

- The link target is an external portal index that Huawei reorganises; unlike
  the API reference pages, it is not mirrored in this repo's
  `documentation/` tree, so it cannot be pinned or diffed.
- Because the page has no content, this card carries no `HW-12-xxxx` findings.
  That is an absence of content to review, not a clean bill of health for the
  industry — see `JOBS-02` through `JOBS-04` for the four defects we did find.

## References

- `huawei_industry_tree/12_jobs/docs/05_practice-jobs-app-architecture-v1_2.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-jobs-app-architecture-v1_2-0000002424876885
- `JOBS-01` — the key-scenarios index, the other doc-only page in this pack
- `JOBS-02`, `JOBS-03`, `JOBS-04` — the three samples and every finding recorded for this industry
- `documentation/harmonyos-guides/07_application-services/notification-enable.md` — the closest thing to an authoritative FAQ answer for `JOBS-02`'s authorization flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/notification-enable
