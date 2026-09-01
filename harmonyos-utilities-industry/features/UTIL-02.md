---
id: UTIL-02
title: Key-scenario index - the utilities industry's 44 sample links and how they map onto the print skeleton
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/02_practice-tools-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1_1-0000002236204802
sample: none
kits: []
apis: []
permissions: []
findings: []
status: verified
---

## When to use

Load this card when you know *what* you need to build in a utility app but not
*which* sample shows it - a compass, an NFC tag writer, a floating tool ball,
a bearing dial, a zip extractor. The page it documents (关键场景示例, "key
scenario samples") is the utilities industry's index: 44 links, no prose, no
code, no download of its own.

Its practical value is as a routing table. `UTIL-01` gives you the app
skeleton - the HAR layout, the routeMap navigation, the tab shell - and this
page is the catalogue of small features you drop into that skeleton. Read it
as "which card do I open next", not as a technical document.

**Verification level is low by construction.** There is no sample zip and no
code on the page, so nothing here is compile-verified. What the review did
verify is that the list resolves: every entry has a corresponding crawled
document in this repo and a feature card in this pack. Defects live on the
individual scenario cards, not here.

## Feature checklist

What the page promises, and what was checked:

- A flat list of 44 scenario links under the utilities architecture guide.
- Each link points at an `architecture-guides` page with its own scenario
  description, implementation notes and, in almost all cases, a sample zip.
- The scenarios are single-feature, not app-sized: one animation, one sensor,
  one crypto routine, one UI control each.
- No ordering by theme - sensors, crypto, networking and UI controls are
  interleaved in what looks like publication order.
- All 44 entries resolve to crawled documents in
  `huawei_industry_tree/15_utilities/docs/` as `03_*` through `46_*`, in the
  same order as they appear on the page.

## Architecture

There is no project. The page is a leaf of the industry documentation tree:

```
15_utilities/docs
├── 01_practice-tools-app-architecture-v1.md    the print app skeleton  -> UTIL-01
├── 02_practice-tools-app-architecture-v1_1.md  THIS PAGE: 44 links     -> UTIL-02
├── 03_radar_scan_effect.md .. 46_poster_gen.md the scenarios           -> UTIL-03..UTIL-46
└── 47_practice-tools-app-architecture-v1_2.md  the follow-on index     -> UTIL-47
```

The 44 entries, grouped by what they actually teach (the page itself does not
group them):

| Theme | Cards |
| --- | --- |
| Drawing, animation and custom controls | `UTIL-03` radar sweep · `UTIL-04` azimuth sector · `UTIL-05` speed gauge · `UTIL-07` bottom drawer · `UTIL-22` compass · `UTIL-25` drag and pinch · `UTIL-26` LED scroll text · `UTIL-39` swiper conference grid |
| Sensors and hardware | `UTIL-21` NFC read/write · `UTIL-23` spirit level · `UTIL-42` HCE tag emulation · `UTIL-36` native audio inner-record |
| Networking | `UTIL-09` network state · `UTIL-18` Wi-Fi QR connect · `UTIL-31` batch download · `UTIL-32` IP address · `UTIL-35` rcp pause/resume · `UTIL-40` ping |
| Files, media and images | `UTIL-08` web poster · `UTIL-10` navbar colour from image · `UTIL-19` unzip · `UTIL-27` H5 base64 save · `UTIL-34` PC upload · `UTIL-37` photo geotag · `UTIL-38` camera with location · `UTIL-41` decode to formats · `UTIL-46` poster composition |
| Text entry and input method | `UTIL-13` immersive keyboard · `UTIL-15` auto-advance fields · `UTIL-45` IME candidate box |
| Crypto | `UTIL-29` RSA · `UTIL-30` HMAC · `UTIL-33` dynamic password |
| App integration and system surfaces | `UTIL-11` desktop card · `UTIL-12` floating tool ball · `UTIL-14` app-store review page |
| Data and concurrency | `UTIL-16` TaskPool file query · `UTIL-43` local database |
| Utilities and misc | `UTIL-06` ringtone · `UTIL-17` VIN scan · `UTIL-20` calculator · `UTIL-24` custom timer · `UTIL-28` date calculator · `UTIL-44` 3D model drag-rotate |

**The design decision worth understanding** is why this page exists at all.
The architecture guide (`UTIL-01`) is deliberately a skeleton: it ships the
module layout and three tab shells, and explicitly says the content is local
placeholder data. The scenario pages are where the actual techniques live, and
none of them ships a skeleton of its own - each is a single `entry` module
with one page. The split is intentional: **one card teaches structure, 44
cards teach behaviours, and they are meant to be combined**, with the scenario
code landing inside a feature HAR of the `UTIL-01` layout.

Two consequences follow from that split, and both matter when you pull a
scenario into a real project:

- Scenario samples do their own window and avoid-area setup in
  `EntryAbility`. Dropped into a multi-module app, that work is already done
  by the host ability - the scenario's copy has to be deleted, not merged.
- Scenario samples are single-module, so their `Constants.ets` is local. In
  the `UTIL-01` layout the equivalent lives in the `Basic` HAR and is shared;
  merging two scenarios naively gives you two competing constants files.

## Implementation steps

1. **Start from `UTIL-01`** for the module layout, the `Navigation` +
   `NavPathStack` routing and the breakpoint plumbing. Do not start from a
   scenario sample - none of them has a structure worth growing.
2. **Pick the scenarios you need from the table above** and open those cards
   before the documents; each card carries the zip-verified code and the
   defects found in it.
3. **Move each scenario's page into a feature HAR**, add it to that HAR's
   `route_map.json` with an exported `@Builder`, and push it by name. That is
   the only integration step the scenario samples do not show, because none of
   them has more than one module.
4. **Delete the scenario's `EntryAbility` window setup** and read
   `topRectHeight` / `bottomRectHeight` from `AppStorage` as the host ability
   publishes them.
5. **Merge permissions deliberately.** Each scenario declares its own; the
   union is not the right answer, and `UTIL-01`'s own block is already over-
   declared (`HW-15-0001`).
6. **Check the card's Pitfalls before copying any scenario.** Roughly two
   quarters of the scenario cards in this pack carry at least one
   medium-or-higher B-category finding, and several - `UTIL-16`'s
   `LazyForEach` keying, the `finally`-`closeSync` family - recur across
   samples rather than being one-offs.

## Verified snippets

**No sample ships with this page**, and the page contains no code block at
all - it is 44 list items. The nearest thing to a snippet is the list itself,
reproduced here in its documented order with the card each entry maps to
(from the doc - no sample shipped; not compile-verified):

```text
雷达扫描动画                        -> UTIL-03   radar scan animation
方位角与夹角测量                    -> UTIL-04   azimuth and included angle
测速仪表盘自定义                    -> UTIL-05   custom speed gauge
铃声设置                            -> UTIL-06   ringtone settings
底部抽屉滑动效果                    -> UTIL-07   bottom drawer
网页海报生成及分享                  -> UTIL-08   web poster generate and share
网络状态信息查询                    -> UTIL-09   network state query
导航栏背景变色                      -> UTIL-10   navbar colour from image
应用卡片添加至桌面                  -> UTIL-11   add app card to desktop
应用内悬浮工具球                    -> UTIL-12   in-app floating tool ball
...  (44 entries, matching docs/03_* .. docs/46_* one for one)
车架号扫描识别                      -> UTIL-17   VIN scanning
TaskPool文件查询                    -> UTIL-16   TaskPool file query
如何实现图片合成生成海报的功能      -> UTIL-46   poster composition
```

The one structural observation worth recording: **the order is publication
order, not difficulty or theme order**, and the last dozen entries
(`UTIL-35` onward - rcp download resume, native audio inner-record, photo
geotagging, 3D model rotation, IME candidate boxes) are noticeably more
specialised than the first dozen. If you are reading the list top to bottom
looking for a starting point, the first ten are the general-purpose UI
techniques.

Three entries are also not native to this industry, which the page does not
say: `38_convenient-life-v1_2-ts_133` comes from the convenient-life tree,
`43_educate-v1_1-ts_23` from the education tree, and `41_tools-v1_2-ts_59` is
a v1.2 tools page. They are cross-listed here because the technique applies,
not because the sample was written for a utility app.

## Constraints

- **Documentation only.** No sample zip, no code, nothing to compile or run.
- The verification behind this card is link resolution and mapping, not code
  review. Every technical claim about a scenario belongs on that scenario's
  own card.
- The list is a snapshot. Huawei adds entries to this page over time; the
  crawl behind this repo captured 44, numbered `03` to `46` in the docs
  directory.
- `UTIL-47` documents the follow-on index page
  (`practice-tools-app-architecture-v1_2`) and carries further scenarios not
  listed here - check both indexes when hunting for a technique.

## Pitfalls

- **No defects were found in this document.** It is a link list; the links
  resolve and the order matches the crawled documents.
- The risk this page carries is indirect: it presents 44 samples as a flat,
  equal menu with no indication that many of them contain defects. Thirty-four
  of the scenario cards in this pack carry a medium-or-higher B-category
  finding, and the skeleton this index sits under (`UTIL-01`) carries a HIGH
  one (`HW-15-0010`). Read the target card, not just the target document.

## References

- `huawei_industry_tree/15_utilities/docs/02_practice-tools-app-architecture-v1_1.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1_1-0000002236204802
- `UTIL-01` - the print app skeleton these scenarios are meant to be dropped into
- `UTIL-47` - the follow-on key-scenario index page
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - what moving a scenario page into a feature HAR involves
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
