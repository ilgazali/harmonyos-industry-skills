---
id: FIN-11
title: Industry FAQ - migrated to the HarmonyOS FAQ site
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/11_practice-insurance-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1_2-0000002263613646
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-07-0011]
status: verified-with-fixes
---

## When to use

Never as a source of technical content - the document has none. This card exists
to record that the industry FAQ slot is empty, so an agent looking for finance
or insurance troubleshooting guidance does not keep searching the architecture
guides for it.

## Feature checklist

Not applicable.

## Architecture

Not applicable.

## Implementation steps

If you are looking for troubleshooting guidance for a finance or insurance app:

1. Check the pitfalls sections of `FIN-01` and `FIN-03` through `FIN-10` first -
   they carry the concrete defects found in the shipped samples, and in this
   industry a disproportionate number of them are security defects.
2. Fall back to `references/base/` for cross-industry conventions.
3. For the security-sensitive parts specifically, the local corpus is the better
   source: `documentation/harmonyos-guides/04_system/` for permissions and
   privacy, and `documentation/harmonyos-guides/08_ai/vision-cardrecognition.md`
   for document scanning.
4. The HarmonyOS FAQ site is not filtered by industry, so search it by symptom -
   component name, API name, error code - rather than by industry.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

There is no finance- or insurance-specific FAQ content in the architecture
guides. This matters more here than in other industries: compliance, key
storage, certificate handling and audit questions are exactly what an industry
FAQ would be expected to answer, and none of it is anywhere in this document set.

## Pitfalls

- **`HW-07-0011` — the FAQ link does not lead to industry content.** The page
  says the industry FAQ has moved to the HarmonyOS FAQ and links to
  `harmonyos-faqs/faq-phone`, the general phone FAQ index, with no anchor or
  filter for finance. The identical stub with the identical target appears in
  the maternity health (`HW-10-0014`), automotive (`HW-01-0012`) and public
  transport (`HW-06-0049`) industries - four of the four reviewed so far - so
  this is shared boilerplate, not a per-industry pointer.

## References

- `documentation/harmonyos-guides/` and `documentation/harmonyos-references/` - the local corpus is a better first stop than the FAQ site for API-level questions.
- `documentation/harmonyos-guides/08_ai/vision-cardrecognition.md` - card recognition, referenced from the FAQ answer
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/vision-cardrecognition
