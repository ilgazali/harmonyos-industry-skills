---
id: UTIL-47
title: Industry FAQ page - a redirect, and what the 45 utilities samples actually get wrong
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/47_practice-tools-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1_2-0000002263801246
sample: none
kits: []
apis: []
permissions: []
modules: []
findings: []
status: verified
---

## When to use

**Load this card when you were sent to the utilities industry's 行业常见问题
(industry FAQ) page and need to know where the answers really are.** The page
itself is one sentence: the content has moved to the HarmonyOS FAQ portal, here
is a link. There is no sample, no snippet and no guidance left on it.

So this card does two jobs. First, it routes: for a concrete question, the
answer is almost always in one of the 45 scenario cards in this pack, and the
table below maps the recurring question shapes onto them. Second, it carries
what the FAQ page would say if it were written from the samples: after
reviewing all 45 of them, the same handful of mistakes recur across unrelated
features, and knowing those five patterns is worth more than any single card.

Treat it as an index with an errata section, not as a technical reference.
Everything below the routing table is drawn from the review of the sibling
samples, not from this page.

## What the page promises

- One line of prose: 行业常见问题内容已迁移至HarmonyOS FAQ，请点击此处前往
  ("the industry FAQ content has been migrated to the HarmonyOS FAQ, click
  here to go there").
- A single external link to `harmonyos-faqs/faq-phone`.
- No scenario description, no 实现思路 (implementation approach), no 约束与限制
  (constraints), no project tree, no code download - the four sections every
  other page in this industry carries.

The link target is outside the industry tree and was not crawled, so nothing on
the other side of it has been reviewed. Anything you take from it is
unverified by this pack.

## Architecture

Documentation-only page; no sample project ships with it. Its place in the
industry tree:

```
15_utilities
├── 01_practice-tools-app-architecture-v1.md   the architecture narrative + LogTemp template
├── 02_practice-tools-app-architecture-v1_1.md 关键场景示例 - the index of scenarios (no zip)
├── 03..46_*.md                                44 scenario pages, each with a sample zip
└── 47_practice-tools-app-architecture-v1_2.md this page - FAQ, redirected away
```

**The structural point worth knowing** is that this industry has three
doc-level pages and 44 scenario pages, and only the scenario pages carry
verified code. `UTIL-01` is the one place where an *architecture* page ships a
runnable template (the LogTemp printing app), and it is also the page whose
defects propagate furthest, because its `module.json5` is what readers copy
first (`HW-15-0001`: seven declared permissions including an unused
`DISTRIBUTED_DATASYNC`, the restricted `WRITE_IMAGEVIDEO`, and the deprecated
`READ_MEDIA`/`WRITE_MEDIA` pair). If you arrived at this FAQ page from the
architecture guide, `UTIL-01` and `UTIL-02` are the pages you actually wanted.

## Where the answers are

| If the question is about | Go to |
|---|---|
| App structure, printing, the industry template | `UTIL-01`, `UTIL-02` |
| Canvas animation, gauges, radar sweeps, LED text | `UTIL-03`, `UTIL-05`, `UTIL-26` |
| Sensors: compass, spirit level, orientation | `UTIL-22`, `UTIL-23`, `UTIL-04` |
| Camera and image decoding pipelines | `UTIL-17`, `UTIL-41`, `UTIL-38` |
| Photos, EXIF and album saves | `UTIL-37`, `UTIL-27`, `UTIL-31` |
| Poster and share-card generation | `UTIL-46`, `UTIL-08` |
| Files: unzip, TaskPool queries, downloads, resume | `UTIL-19`, `UTIL-16`, `UTIL-35`, `UTIL-34` |
| Networking: state, IP, speed test, ping | `UTIL-09`, `UTIL-32`, `UTIL-05`, `UTIL-40` |
| Wi-Fi and NFC | `UTIL-18`, `UTIL-21`, `UTIL-42` |
| Crypto: RSA, HMAC, one-time passwords | `UTIL-29`, `UTIL-30`, `UTIL-33` |
| Input methods and text entry | `UTIL-45`, `UTIL-13`, `UTIL-15` |
| Widgets, float balls, drawers, drag and drop | `UTIL-11`, `UTIL-12`, `UTIL-07`, `UTIL-25` |
| Local databases (RDB and KVStore) | `UTIL-43` |
| 3D scenes | `UTIL-44` |

## Cross-sample systematics

Five defect patterns recur across samples that share no code. Each was raised
as a systematic finding on one feature and lists the others as instances; check
your own code against all five before shipping anything built from this pack.

1. **Subscriptions and native resources are never released.** The most common
   defect in the industry, in ten samples. Sensors keep sampling after the page
   is gone (`HW-15-0004` spirit level, `HW-15-0055` compass), camera
   `ImageReceiver` pipelines are rebuilt on every re-entry without teardown
   (`HW-15-0044`, systematic over two samples), NFC `hceCmd` is re-subscribed
   on every tab show so one APDU gets N responses (`HW-15-0087`), KVStore
   result sets are never closed until the eight-handle limit is hit
   (`HW-15-0088`), an `AVPlayer` is created and never released (`HW-15-0006`),
   and the IME's listeners are off'd with fresh closures or only under a debug
   flag (`HW-15-0095`). **Rule: every `on(...)`, `open(...)` and `create(...)`
   needs its pair on the exit path, with the original callback reference.**
2. **Permission results are ignored.** `HW-15-0078` is systematic over three
   samples: `requestPermissionsFromUser` is treated as granted whenever no
   error came back, but a refusal returns `authResults: -1` with **no** error -
   so the refusal path proceeds into `getCurrentLocation` or a camera session
   and fails later with a raw exception. Related: `MEDIA_LOCATION` declared but
   never requested, silently redacting the EXIF GPS the feature is about
   (`HW-15-0079`); a restricted `WRITE_IMAGEVIDEO` that is auto-denied without
   an ACL (`HW-15-0063`). **Rule: branch on `data.authResults`, and prefer a
   security component (`SaveButton`) or a picker over a declared permission.**
3. **Promises with no `catch` on the routine failure path.** Not exotic errors -
   cancel and back-navigation. `taskpool.execute` rejects on page-exit cancel
   and nobody listens (`HW-15-0038`); an rcp pause rejects the in-flight fetch
   and it surfaces as an unhandled rejection (`HW-15-0075`); one failed group
   deadlocks a batch download forever (`HW-15-0069`); `getDefaultNet` and
   `linkAddresses[0]` are dereferenced blind (`HW-15-0071`); and `closeSync` is
   called on a `file` that is still `undefined` after a cancelled picker
   (`HW-15-0016`, systematic over three samples) - so the cleanup error
   replaces the real one.
4. **`(item: string) => item` key generators over object arrays.** `HW-15-0039`,
   in a `LazyForEach` designed for 100 000 rows and in a `ForEach` whose
   per-item delete depends on the key. Every row stringifies to
   `[object Object]`, so diffing and item reuse are broken and deletes hit the
   wrong row. **Rule: return a real id (`item.path`), never the item.**
5. **Geometry hardcoded for one display.** A float ball starting at
   `(1100, 2300)` px, off-screen on smaller devices (`HW-15-0035`); an IME hot
   zone pinned to `top: 1613, width: 1200` and a panel resized to `100x100`
   (`HW-15-0094`); device adaptation done by exact width/height pairs with a
   fallback ratio for everything else. **Rule: derive from
   `display.getDefaultDisplaySync()` and the component's own measured size.**

A sixth pattern is not a code defect but affects anyone reading these pages:
**the documents drift from the zips.** Ten findings in this industry are
document-only - a project tree naming `NfcUtils.ets` where the zip ships
`NfcUtil.ets` (`HW-15-0008`), a snippet referencing a constant that does not
exist (`HW-15-0019`), a misspelled constant (`HW-15-0009`), constraints
claiming API 20 for a sample targeting 5.0.5(17) (`HW-15-0003`), a reference
link that resolves to nothing (`HW-15-0002`), a described behaviour (average
colour) that the code does not implement (`HW-15-0030`). **Read the zip, not
the page** - and where a page's snippet reproduces a defect verbatim, as doc 45
and doc 46 both do, copying the documented code is how the defect spreads.

## Verified snippets

**None.** This page carries no code - the only content is the redirect line
(from the doc - no sample shipped; not compile-verified):

```markdown
行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

For code, open any of the 45 cards in this pack backed by a sample zip. Their
snippets are all taken from the extracted zip, with corrections marked against
the finding that motivated them.

## Constraints

- The page has no 约束与限制 section of its own. Per-sample constraints live on
  each scenario card; the industry baseline is API Version 20 Release,
  HarmonyOS 6.0.0 Release SDK and DevEco Studio 6.0.0 Release, with `UTIL-23`
  the documented exception (`HW-15-0003`).
- The FAQ portal behind the link is outside this review's scope. Nothing there
  is verified by this pack.
- 45 of the 47 pages ship a sample zip - all 44 scenario pages plus `UTIL-01`.
  The two without one are `UTIL-02` (the scenario index) and this page.
  `UTIL-45` is the exception in the other direction: one zip, two complete
  projects.

## Pitfalls

- **No defects were raised against this document.** It is one accurate sentence
  and a working link; there is nothing in it to be wrong.
- The risk it carries is navigational: it is the last page of the industry
  architecture guide, so a reader following the guide in order ends on a
  redirect and may never reach the scenario pages that hold the actual content.
  Use the routing table above.
- Do not treat the absence of findings here as a quality signal. It reflects
  that the page contains no technical claims - not that the industry's guidance
  is defect-free. The 99 findings recorded across `HW-15-0001`..`HW-15-0099`
  say otherwise.

## References

- `huawei_industry_tree/15_utilities/docs/47_practice-tools-app-architecture-v1_2.md` - this page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1_2-0000002263801246
- `UTIL-01` - the industry architecture narrative and its runnable template
- `UTIL-02` - 关键场景示例, the scenario index this pack's cards are built from
- `huawei_industry_tree/15_utilities/docs/README.md` - the crawled document list
