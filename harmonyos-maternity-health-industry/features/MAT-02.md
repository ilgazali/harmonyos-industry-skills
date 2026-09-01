---
id: MAT-02
title: Key scenario index for the maternity health industry
industry: 10_maternity_health
doc: huawei_industry_tree/10_maternity_health/docs/02_practice-health-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1_1-0000002270340425
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Do not load this card on its own. It is the industry's scenario index: it
records which feature-level practices Huawei publishes for maternity health, so
that a router or an agent picking a card knows the full set. Load `MAT-01` for
the architecture and then the card for the screen being built.

## Feature checklist

Huawei publishes exactly two feature-level practices for this industry:

| Card | Scenario | Sample |
|---|---|---|
| `MAT-03` | Baby growth record timeline (宝宝成长记录时间轴) | `RecordTimeLine.zip` |
| `MAT-04` | Child growth curve record (儿童生长曲线记录) | `BabyGrowthRecordCurve.zip` |

Everything else in a maternity health app - the community feed, the health
calendar, the post composer, the profile area - is covered by the architecture
card `MAT-01` and its sample project, not by a dedicated practice document.

## Architecture

Not applicable. The source document is a two-entry link list with no technical
content of its own.

## Implementation steps

None. Follow `MAT-01` first, then `MAT-03` or `MAT-04`.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

The published feature set for this industry is small. If the app needs
vaccination scheduling, antenatal appointment reminders, feeding logs or
device-cloud sync, there is no HQ practice document for it; build on the
`MAT-01` architecture and the common technical solutions skill instead.

## Pitfalls

No defects found in this document. Both links resolve to the documents they
name, and the two titles match the `title` frontmatter of `MAT-03` and `MAT-04`.

## References

- `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
