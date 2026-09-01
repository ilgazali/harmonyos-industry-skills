---
id: MAT-05
title: Industry FAQ - migrated to the HarmonyOS FAQ site
industry: 10_maternity_health
doc: huawei_industry_tree/10_maternity_health/docs/05_practice-health-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1_2-0000002332396545
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-10-0014]
status: verified-with-fixes
---

## When to use

Never as a source of technical content - the document has none. This card exists
to record that the industry FAQ slot is empty, so an agent looking for
maternity-health troubleshooting guidance does not keep searching the
architecture guides for it.

## Feature checklist

Not applicable.

## Architecture

Not applicable.

## Implementation steps

If you are looking for troubleshooting guidance for a maternity health app:

1. Check the pitfalls sections of `MAT-01`, `MAT-03` and `MAT-04` first - they
   carry the concrete defects found in the shipped samples.
2. Fall back to `references/base/` for the cross-industry conventions.
3. The HarmonyOS FAQ site is unfiltered by industry, so search it by symptom
   (component name, API name, error code) rather than by industry.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

There is no maternity-health-specific FAQ content in the architecture guides.

## Pitfalls

- **`HW-10-0014` — the FAQ link does not lead to industry content.** The page
  says the industry FAQ has moved to the HarmonyOS FAQ and links to
  `harmonyos-faqs/faq-phone`, the general phone FAQ index, with no anchor or
  filter for maternity health. Do not expect industry-specific answers behind
  that link.

## References

- `documentation/harmonyos-guides/` and `documentation/harmonyos-references/` - the local corpus is a better first stop than the FAQ site for API-level questions.
