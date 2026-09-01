---
id: AUTO-08
title: Industry FAQ - migrated to the HarmonyOS FAQ site
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/08_practice-auto-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1_2-0000002297842985
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-01-0012]
status: verified-with-fixes
---

## When to use

Never as a source of technical content - the document has none. This card exists
to record that the industry FAQ slot is empty, so an agent looking for
automotive troubleshooting guidance does not keep searching the architecture
guides for it.

## Feature checklist

Not applicable.

## Architecture

Not applicable.

## Implementation steps

If you are looking for troubleshooting guidance for an automotive app:

1. Check the pitfalls sections of `AUTO-01` and `AUTO-03` through `AUTO-07`
   first - they carry the concrete defects found in the shipped samples.
2. Fall back to `references/base/` for cross-industry conventions.
3. For Map Kit and Scan Kit problems specifically, the local corpus under
   `documentation/harmonyos-guides/07_application-services/` is more useful than
   the FAQ site.
4. The HarmonyOS FAQ site is not filtered by industry, so search it by symptom -
   component name, API name, error code - rather than by industry.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

There is no automotive-specific FAQ content in the architecture guides.

## Pitfalls

- **`HW-01-0012` — the FAQ link does not lead to industry content.** The page
  says the industry FAQ has moved to the HarmonyOS FAQ and links to
  `harmonyos-faqs/faq-phone`, the general phone FAQ index, with no anchor or
  filter for automotive. The identical stub with the identical target appears in
  the maternity health industry (`HW-10-0014`), so this is shared boilerplate,
  not a per-industry pointer.

## References

- `documentation/harmonyos-guides/` and `documentation/harmonyos-references/` - the local corpus is a better first stop than the FAQ site for API-level questions.
