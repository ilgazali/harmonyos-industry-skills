---
id: TRANS-09
title: Industry FAQ - migrated to the HarmonyOS FAQ site
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/09_practice-bus-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1_2-0000002591551329
sample: none
kits: []
apis: []
permissions: []
min_api: 24
modules: []
findings: [HW-06-0049]
status: verified-with-fixes
---

## When to use

Never as a source of technical content - the document has none. This card exists
to record that the industry FAQ slot is empty, so an agent looking for
public-transport troubleshooting guidance does not keep searching the
architecture guides for it.

## Feature checklist

Not applicable.

## Architecture

Not applicable.

## Implementation steps

If you are looking for troubleshooting guidance for a transit app:

1. Check the pitfalls sections of `TRANS-01` and `TRANS-03` through `TRANS-08`
   first - they carry the concrete defects found in the shipped samples.
2. Fall back to `references/base/` for cross-industry conventions.
3. For Map Kit and Location Kit problems specifically, the local corpus under
   `documentation/harmonyos-guides/04_application-services/` and
   `documentation/harmonyos-guides/07_application-services/` is more useful than
   the FAQ site. Note that
   `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md`
   is a stub in this corpus, so `GeoAddress` field semantics must be checked
   against the live reference.
4. The HarmonyOS FAQ site is not filtered by industry, so search it by symptom -
   component name, API name, error code - rather than by industry.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

There is no public-transport-specific FAQ content in the architecture guides.

## Pitfalls

- **`HW-06-0049` — the FAQ link does not lead to industry content.** The page
  says the industry FAQ has moved to the HarmonyOS FAQ and links to
  `harmonyos-faqs/faq-phone`, the general phone FAQ index, with no anchor or
  filter for transit. The identical stub with the identical target appears in
  the maternity health (`HW-10-0014`) and automotive (`HW-01-0012`) industries -
  three of the three industries reviewed so far - so this is shared boilerplate,
  not a per-industry pointer.

## References

- `documentation/harmonyos-guides/` and `documentation/harmonyos-references/` - the local corpus is a better first stop than the FAQ site for API-level questions.
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - geolocation manager
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
