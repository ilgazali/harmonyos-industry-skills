---
id: FOOD-06
title: Food industry FAQ page - a redirect stub, and where the answers actually are
industry: 17_food
doc: huawei_industry_tree/17_food/docs/06_practice-food-app-architecture-v1_4.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-architecture-v1_4-0000002287672312
sample: none
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when something points you at the food industry's 行业常见问题
(industry FAQ) page and you want to know whether it is worth opening. It is
not: **the page has been emptied**. Its entire body is one sentence saying the
FAQ content has moved to the HarmonyOS FAQ site, followed by a link.

This card exists to absorb that dead end. It records what the page is, and
then does the job the page no longer does - routing a food-app question to the
sibling card that answers it, and listing the questions this industry's own
samples turn out to raise.

Nothing here is compile-verified: there is no sample project, and the source
document contains no code. Treat the routing table as navigation and the
recurring-issue list as a checklist, not as verified guidance - the verified
guidance is in the cards it points to.

## Feature checklist

What the page promises, in full:

- A single paragraph stating that industry FAQ content has migrated to
  HarmonyOS FAQ.
- One outbound link to the phone FAQ index.

There is no scenario, no sample, no permission section, no project tree and no
constraints section. The document is eleven lines long including its
front matter.

## Architecture

Documentation-only page, one of two such pages in this industry:

```
17_food/docs
├── 01_practice-food-app-design-demo-v1.md      FOOD-01, food ordering app template (sample)
├── 02_practice-food-menu-app-demo-v1.md        FOOD-02, recipe app template (sample)
├── 03_practice-food-app-architecture-v1_1.md   FOOD-03, key-scenario index (no sample)
├── 04_map_navigation.md                        FOOD-04, store address and route (sample)
├── 05_custom_refresh.md                        FOOD-05, custom refresh and load-more (sample)
└── 06_practice-food-app-architecture-v1_4.md   FOOD-06, this page (no sample, no content)
```

`FOOD-03` and `FOOD-06` are the two index pages of the set, and they fail in
opposite ways. `FOOD-03` still holds real navigation - it is the page that
links `04` and `05`. `FOOD-06` holds nothing: the migration removed the
content and left neither a topic list nor deep links into the FAQ site, only a
link to the phone FAQ front page.

**The structural decision worth noting** - as a warning, not a pattern - is
that the migration kept the page in the architecture-guide navigation tree
after emptying it. A reader following the industry sidebar in order reaches a
page that costs a click and returns nothing. If you maintain a documentation
tree, an emptied page should either carry the topic list it used to have with
links to where each topic now lives, or be removed from the tree.

## Implementation steps

There is no feature to implement. The steps are navigational:

1. **Do not look for food-specific answers on the HarmonyOS FAQ site.** It is
   the general phone FAQ index; nothing on it is scoped to this industry.
2. **Route the question to a sibling card instead** using the table below.
3. **Check the recurring-issue list** before shipping - the defects in this
   section were found across independent samples in this one industry, so they
   are the closest thing to a real FAQ this material has.
4. **For anything genuinely platform-level** (signing, ArkTS syntax, DevEco
   build errors), the general FAQ site is the right destination and no
   industry page is needed.

**Where food-app questions are actually answered:**

| Question | Card |
| --- | --- |
| How do I structure a whole ordering app - home, cart, orders, mine? | `FOOD-01` |
| Huawei ID login, scan-to-order, service cards, in-app map | `FOOD-01` |
| How do I structure a recipe app - feed, search, categories, basket? | `FOOD-02` |
| Login / payment components, ads, membership | `FOOD-02` |
| Which key-scenario samples exist for this industry? | `FOOD-03` |
| How do I show a store's location and start navigation? | `FOOD-04` |
| Static map image, Petal Maps handoff, AGC map service setup | `FOOD-04` |
| How do I build a custom pull-to-refresh and load-more feed? | `FOOD-05` |
| `WaterFlow` + `LazyForEach` + `IDataSource` wiring | `FOOD-05` |

## Verified snippets

The document's complete body (from the doc — no sample shipped; not
compile-verified):

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

That is the whole page. 行业常见问题 is "industry frequently asked questions";
内容已迁移至 is "the content has been migrated to"; 请点击此处前往 is "please
click here to go there". The link target is the generic phone FAQ index, not
a food section of it - so the migration lost the industry scoping, not just
the page.

## Recurring questions this industry actually raises

Drawn from the review of the four samples in this industry. Each is a defect
found in shipped Huawei sample code, which makes it a question a developer
copying that code will eventually have.

- **Permissions declared in `module.json5` do not match the document or the
  code.** All four samples diverge somewhere: unused `GYROSCOPE`,
  `ACCELEROMETER` and `GET_NETWORK_INFO` in the ordering template
  (`HW-17-0005`), an undocumented `APP_TRACKING_CONSENT` in the recipe app
  (`HW-17-0016`), an undocumented and unused `GET_NETWORK_INFO`
  (`HW-17-0021`), and a map sample that declares nothing at all despite
  needing `INTERNET` (`HW-17-0022`). Audit the array against both the document
  and the call sites before shipping.
- **`ohos.permission.LOCATION` cannot be declared on its own.** The ordering
  template's document says to declare only it (`HW-17-0006`); the platform
  guide requires `APPROXIMATELY_LOCATION` alongside it.
- **Permission checks that loop and overwrite.** `PermissionsUtil` honours
  only the last permission in the list and ignores the request result
  (`HW-17-0004`). Accumulate across the array; never assign per iteration.
- **Kit calls with no error path.** Map Kit calls have neither `.catch` nor
  `try/catch` although the reference documents `1002600004` "The Map
  permission is not enabled" (`HW-17-0024`), and ad-service catch blocks log a
  copy-pasted string while discarding the real error (`HW-17-0019`).
- **Project trees in the documents do not match the zips.** Wrong directory
  names (`HW-17-0008`), wrong file names and comments (`HW-17-0017`), and
  snippets that name a function the sample spells differently
  (`HW-17-0007`). Read the zip, not the document, when the two disagree.
- **Reference links in these documents rot.** Two cited pages do not resolve
  anywhere in the documentation tree (`HW-17-0013`, `HW-17-0020`).
- **Navigation implemented as `push` where `pop` is meant.** Returning to the
  main page stacks a new instance instead of unwinding (`HW-17-0009`).
- **Never sign payments on the device.** The recipe template hardcodes a full
  RSA-2048 merchant private key in client code (`HW-17-0014`). This is the one
  item on the list that is not a bug to fix but a design to discard.

## Constraints

- No sample project, no code, no permissions, no API level. The
  document carries none of the standard 约束与限制 (constraints) section that
  every other page in this industry has.
- The one outbound link leaves the architecture-guide tree entirely, so
  nothing beyond it is covered by this review.
- Verification level for this card is **document-only**: the page was read and
  its claim confirmed. Everything in the routing table and the recurring-issue
  list is verified in the card it cites, not here.

## Pitfalls

- No defects were recorded against this document. A page that says one true
  thing and stops has little surface to be wrong on - the cost is what it
  omits, not what it states. See the recurring-questions section above for the
  defects this page would have been the natural home for.

## References

- `huawei_industry_tree/17_food/docs/06_practice-food-app-architecture-v1_4.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-architecture-v1_4-0000002287672312
- `FOOD-03` - the other index page in this industry, and the one that still works
- `FOOD-01`, `FOOD-02` - the two application templates
- `FOOD-04`, `FOOD-05` - the two key-scenario samples
