---
id: PHOTO-32
title: Industry FAQ page - a redirect stub that closes the photography architecture guide
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/32_practice-photo-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1_2-0000002298361249
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

**Load this card only to answer "what is on the photography FAQ page?"** The
answer is: one sentence and one link. The page carries no scenario, no code and
no sample, and it is the last of the three-page frame that wraps this industry's
guide.

The frame is worth knowing even though this page is empty. The photography
architecture guide is published as three shell pages plus twenty-nine scenario
pages:

- `01_practice-photo-app-architecture-v1.md` - 拍摄美化应用案例, the architecture
  narrative: Stage model, one phone-form Entry HAP, feature modules as HARs, the
  six-tile home grid. Card `PHOTO-01`, which also anchors this industry's
  systematic findings.
- `02_practice-photo-app-architecture-v1_1.md` - 关键场景示例, a bare index of the
  twenty-nine scenario links. Card `PHOTO-02`.
- `32_practice-photo-app-architecture-v1_2.md` - 行业常见问题, **this page**: the
  FAQ, moved out to the HarmonyOS FAQ site.

If you are looking for photography answers, go to the scenario cards
(`PHOTO-03` … `PHOTO-31`) or to the platform FAQ this page points at. If you are
looking for the *recurring* problems in this industry's samples, the Pitfalls
section below lists them, because that is the material a real FAQ page would
have been made of.

## Feature checklist

There is no feature. What the page promises, in full:

- A heading, 行业常见问题 (industry FAQ).
- One sentence stating that the content has been migrated to the HarmonyOS FAQ.
- One link, labelled 此处 ("here"), to `harmonyos-faqs/faq-phone`.
- `images: 0` in the frontmatter - the page has no screenshots, no diagrams and
  no code fences.

Verification level: **document only, and a document with no technical content.**
Nothing here was compiled, run, or cross-checked against a sample, because there
is nothing to check beyond whether the link resolves.

## Architecture

No sample project. No zip is published for this page and none is expected.

```
huawei_industry_tree/18_photography/docs
├── 01_practice-photo-app-architecture-v1.md     the architecture narrative (PHOTO-01)
├── 02_practice-photo-app-architecture-v1_1.md   the scenario index (PHOTO-02)
├── 03_image_filter.md … 31_audio-v1_2-ts_50.md  twenty-nine scenario pages
└── 32_practice-photo-app-architecture-v1_2.md   this page: FAQ, redirected out
```

**The design decision worth copying** - and it is a documentation decision, not
a code one - is that the FAQ was *removed* rather than duplicated. Platform FAQs
age badly inside an industry guide: the same question about permission dialogs
or `XComponent` surfaces belongs to the platform, not to photography, and a copy
inside twenty industry guides is twenty copies to keep current. Pointing at one
canonical FAQ is right.

The decision worth avoiding is leaving the empty shell in the navigation tree.
The page still appears in the sidebar as a peer of pages carrying real
scenarios, so a reader who follows the guide in order ends on a dead stub. A
redirect at the URL level, or a deletion with the link folded into page 01,
would cost the reader one click instead of one dead end.

## Implementation steps

Nothing to implement. What to do with this page:

1. **Do not cite it as a source for anything.** It contains no statement about
   HarmonyOS beyond the existence of the FAQ site.
2. **Follow the link for platform-level questions** - permission dialogs,
   window and avoid areas, device compatibility. It is the general phone FAQ,
   not a photography one.
3. **For photography questions, use the scenario cards.** Camera session
   lifecycle: `PHOTO-01`, `PHOTO-31`. Recording: `PHOTO-06`, `PHOTO-25`,
   `PHOTO-30`. Orientation: `PHOTO-29`. Permissions: `PHOTO-10`.
4. **When auditing an industry guide, treat a page like this as a signal, not a
   defect.** An empty terminal page usually means the guide was generated from a
   shared template; the same template produced the wrong-topic URL slugs
   documented in `HW-18-0008`.

## Verified snippets

There is no sample. The page's entire body, quoted from
`huawei_industry_tree/18_photography/docs/32_practice-photo-app-architecture-v1_2.md`
(from the doc — no sample shipped; not compile-verified):

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

That is the whole page. 行业常见问题 is "industry frequently asked questions";
the sentence reads "the industry FAQ content has been migrated to the HarmonyOS
FAQ, please click *here* to go there." The target is `harmonyos-faqs/faq-phone`
- the **phone** FAQ, a platform-wide page, not a photography section of one. So
the migration was not a move of photography-specific answers into a new home;
the industry-specific FAQ simply stopped existing.

For contrast, the sibling shell page that does carry content - the scenario
index at `02_practice-photo-app-architecture-v1_1.md` - is a flat list of
twenty-nine links whose last three entries are the samples this review batch
covered:

```markdown
- **[基于重力传感器实现相机图标和照片拍摄角度跟随设备旋转](…/camera_twist-0000002552826219)**
- **[自定义相机退后台保持画中画录制](…/pipwindow_recorder-0000002522526388)**
- **[使用OpenGL标准化设备坐标绘制矩形人脸框](…/audio-v1_2-ts_50-0000002407536358)**
```

The third entry is the one whose slug names an unrelated domain - `audio-…` for
an OpenGL face-detection page - which is how the index itself carries the
evidence for `HW-18-0008`.

## Pitfalls

The page has no defects of its own: the link resolves and the sentence is
accurate. What follows is what a photography FAQ page **should** have contained,
drawn from the systematic findings this industry's samples actually produced.
Every id below is anchored on another card; none is a defect in this page.

- **"Why is my preview black at startup?"** — `HW-18-0041` (B/medium, probable):
  camera init races the `XComponent` surface. Four samples fire `cameraShooting`
  from a permission callback or a fixed 200 ms timer while `surfaceId` is still
  `''`. Gate init on a non-empty surface id and drive it from `onLoad`.
- **"Why does switching cameras sometimes report the camera as occupied?"** —
  `HW-18-0010` (B/medium, confirmed): fire-and-forget teardown races the
  rebuild in nine samples. `releaseCamera()`-style helpers await nothing and
  null nothing, and callers invoke them un-awaited right before configuring a
  new session. Await `stop` → `close` → `release`, then reset the fields.
- **"Why does the app still open the camera after the user denies the
  permission?"** — `HW-18-0030` (B/medium, confirmed): six samples ignore
  `requestPermissionsFromUser`'s `authResults` and build the pipeline anyway.
  Check the results and show a refused state.
- **"Why does the permission dialog never appear?"** — `HW-18-0007` (D/high,
  confirmed): two recorder samples request `ohos.permission.MEDIA_LOCATION`
  without declaring it, and one undeclared name rejects the *entire* request, so
  CAMERA and MICROPHONE are never granted. Request only what you declare.
- **"Why was my app rejected for permissions the sample shipped with?"** —
  `HW-18-0004` (D/medium, confirmed): nine samples declare the restricted
  `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` although their code deliberately uses
  the permission-free `SaveButton` and picker flows. Delete both entries.
- **"Why are my recordings empty or corrupt?"** — `HW-18-0029` (B/high,
  confirmed): three recorder samples close the AVRecorder's target fd in a
  `finally` immediately after `prepare()`, before recording starts. Keep the
  file open until `stop()` completes, then close it once.
- **"Why are my photos saved sideways?"** — `HW-18-0027` (B/low, confirmed):
  three samples hardcode `rotation: ROTATION_0` in `PhotoCaptureSetting`; one of
  them computes the sensor rotation and then ignores it. See `PHOTO-29` for the
  gravity mapping and `PHOTO-30` for `videoOutput.getVideoRotation`.
- **"Why does pinch-zoom stop responding at the limits?"** — `HW-18-0023`
  (B/low, confirmed): three samples clamp against a `zoomRatioRange` that is
  stale or never assigned, because the value returned by the session builder is
  discarded at every call site.
- **"Why does this page's URL say `insurance` / `audio`?"** — `HW-18-0008`
  (E/low, confirmed): six architecture pages across the crawled industries are
  published under slugs cloned from unrelated industries' page shells. In
  photography it hits `26_insurance-v1_2-ts_32` and `31_audio-v1_2-ts_50`, and
  the latter also ships the wrong sample zip (see `PHOTO-31`).

## References

- `huawei_industry_tree/18_photography/docs/32_practice-photo-app-architecture-v1_2.md` - this page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1_2-0000002298361249
- `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md` - the architecture narrative the guide opens with
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- `huawei_industry_tree/18_photography/docs/02_practice-photo-app-architecture-v1_1.md` - the scenario index linking all twenty-nine feature pages
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1_1-0000002247958506
- `PHOTO-01` - the architecture card, and the anchor for this industry's systematic findings
- `PHOTO-02` - the scenario index card
- `PHOTO-29`, `PHOTO-30`, `PHOTO-31` - the three scenarios this page follows in the sidebar
