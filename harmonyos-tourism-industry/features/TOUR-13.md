---
id: TOUR-13
title: Tourism industry FAQ - a redirect to the HarmonyOS FAQ
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/13_practice-tourist-park-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1_2-0000002359070945
sample: NO ZIP
kits: []
apis: []
permissions: []
min_api: 24
modules: []
findings: []
status: verified
---

## When to use

Do not. This page holds no content.

It is the third and last of the architecture-level documents in the tourist
park practice, and its entire body is a migration notice:

> 行业常见问题内容已迁移至HarmonyOS FAQ，请点击此处前往。
> (The industry FAQ content has been migrated to the HarmonyOS FAQ; click here
> to go there.)

The link points at `harmonyos-faqs/faq-phone`, outside this corpus.

Load `TOUR-01` for the architecture, `TOUR-02` for the scenario catalog.

## Feature checklist

Nothing to implement. The page is a stub by design, not by omission - the
content moved rather than being lost, and the notice says where.

## Architecture

None. Eleven lines, no code, no images, no sample.

Its place in the industry:

| Document | Card | State |
| --- | --- | --- |
| 旅游住宿应用案例 | `TOUR-01` | the architecture and the framework code |
| 关键场景示例 | `TOUR-02` | the catalog of ten scenario samples |
| 行业常见问题 | `TOUR-13` (this card) | migrated away; a redirect only |

## Implementation steps

None.

## Verified snippets

None - the document contains no code.

## Permissions & config

None.

## Constraints

- The target of the redirect is the general HarmonyOS phone FAQ, not a
  tourism-specific page, so nothing industry-specific survives here. Anything
  a reader needed from the old 行业常见问题 has to be found by search in the
  general FAQ.
- Map Kit troubleshooting - the failure mode this industry hits most, since
  five of its samples draw a map - is covered instead by
  `documentation/harmonyos-guides/07_application-services/map-faq-1.md`
  through `map-faq-4.md`. Those are the pages to reach for, and the cards
  `TOUR-01`, `TOUR-06`, `TOUR-07` and `TOUR-12` cite them.

## Pitfalls

None in the document itself. The corpus-level gap is worth stating: because
this page is empty, **there is no tourism-specific troubleshooting anywhere in
this industry**, and the two failures its samples actually ship with - a
missing `client_id` and a missing `INTERNET` permission, between them
responsible for `HW-09-0034`, `HW-09-0035`, `HW-09-0045` and `HW-09-0072` -
are exactly what an industry FAQ would have caught.

## References

- `huawei_industry_tree/09_tourism/docs/13_practice-tourist-park-app-architecture-v1_2.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1_2-0000002359070945
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - map not displayed
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/07_application-services/map-faq-3.md` - map component questions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-3
- `documentation/harmonyos-guides/07_application-services/map-faq-4.md` - map lifecycle and teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-4
- `TOUR-01` - the architecture document this page belongs to
- `TOUR-02` - the scenario catalog
