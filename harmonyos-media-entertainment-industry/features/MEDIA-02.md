---
id: MEDIA-02
title: Key-scenario index - the 40 media samples that fill in the audio app architecture
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/02_practice-audio-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1_1-0000002273374777
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

Load this card when you know **what** you need to build in a media app but not
**which** sample demonstrates it. The page is the industry's routing table: a
single list of 40 links, each pointing at one narrow scenario with its own zip,
sitting between the architecture narrative (`MEDIA-01`) and the individual
feature cards (`MEDIA-03` … `MEDIA-42`).

It is worth loading first for a second reason. The samples behind these links
were reviewed as a set, and several defects turned out to be **systematic** -
the same mistake copied across five, six or seven of them. Those are listed
under Pitfalls below with the ids to check before you copy any media sample in
this industry, whichever one you land on.

This card is documentation-only: the page ships no code and no zip, so nothing
here is compile-verified. The mapping to card ids is, however, exact - the 40
links are in the same order as documents 03 through 42 in this industry's
tree, one for one.

## Feature checklist

What the page promises, and what it actually is:

- A flat list of 40 scenario links, unordered and ungrouped, with no
  descriptions.
- Every link resolves to a separate architecture-guide page with its own
  实现思路 (implementation approach), 约束与限制 (constraints), 工程目录
  (project tree) and a downloadable zip.
- No overlap and no duplicates: each of the 40 is a distinct scenario.
- The list is the complete set of key scenarios for the media and
  entertainment industry - `MEDIA-01` (the architecture) and `MEDIA-43`
  (行业常见问题, a stub redirecting to the HarmonyOS FAQ) are not in it.

## Architecture

No sample project. The page is one Markdown list, 40 items long.

Its structure is worth naming anyway, because it determines how to use this
industry pack: **the architecture page describes one app, and each scenario
page solves one problem inside it.** `MEDIA-01` shows the tab shell, the
static player manager, the AppStorage state object and the AVSession wiring;
the 40 scenario samples are the parts it stubs out - real file import, real
subtitles, real casting, real compression.

**The design decision worth copying** from this arrangement is the split
itself. Each scenario sample is a single-`entry` HAP with three to eight ArkTS
files and no shared framework, so it can be read in ten minutes and lifted
whole. That is deliberately the opposite of `MEDIA-01`, which is a 21-file
skeleton with no working media resolution. When you build the real app, take
the shell from `MEDIA-01` and the mechanisms from these - not the reverse.

The grouping the page does not give you, by theme:

| Theme | Cards |
| --- | --- |
| Playback control and lifecycle | MEDIA-11, MEDIA-12, MEDIA-17, MEDIA-18, MEDIA-24, MEDIA-29, MEDIA-42 |
| Player UI and chrome | MEDIA-05, MEDIA-07, MEDIA-08, MEDIA-25, MEDIA-26, MEDIA-28, MEDIA-33, MEDIA-39 |
| Media metadata and editing | MEDIA-03, MEDIA-04, MEDIA-14, MEDIA-20, MEDIA-23, MEDIA-32, MEDIA-35, MEDIA-36 |
| Audio-specific | MEDIA-10, MEDIA-13, MEDIA-27, MEDIA-30, MEDIA-31, MEDIA-38 |
| Social and feed | MEDIA-15, MEDIA-16, MEDIA-22 |
| Networking and download | MEDIA-09, MEDIA-30, MEDIA-34 |
| Native (C++/OpenGL/NAPI) | MEDIA-36, MEDIA-37, MEDIA-40 |
| Other | MEDIA-06, MEDIA-19, MEDIA-21, MEDIA-41 |

## Implementation steps

There is nothing to implement from this page. Use it as an index:

1. **Map the link to the card.** The list order is the document order:
   list item *n* is document *n+2* and card `MEDIA-(n+2)`. The first link,
   获取视频信息, is `03_get_video_info.md` and `MEDIA-03`; the last,
   蓝牙耳机佩戴检测…, is `42_demo_audio_session.md` and `MEDIA-42`.
2. **Read the target card, not the page.** Each feature card carries the
   zip-verified snippets and the defects found in that sample; the linked
   Huawei page carries the author's intent, which the two sometimes disagree
   about (`HW-13-0016` is one such case).
3. **Check the systematic ids below** before copying any of the samples -
   they cut across the set and are not repeated in every card.
4. **Check the sample's real `compatibleSdkVersion`** in
   `build-profile.json5` rather than trusting the 约束与限制 section
   (`HW-13-0004`).

## Verified snippets

*(from the doc - no sample shipped; not compile-verified)*

The whole page is this shape, 40 times:

```markdown
- **[获取视频信息](https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_video_info-0000002273254857)**
- **[Videocompressor实现视频压缩功能](https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292)**
- **[滑块及进度条自定义](https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_slider-0000002284049981)**
```

The mirrored copies live under
`huawei_industry_tree/13_media_entertainment/docs/`, numbered `03_` to `42_`
after the slug in each URL (`get_video_info` → `03_get_video_info.md`), and
the zips under `.../downloads/` under the sample's own name
(`PageMediaMeta.zip`, `VideoCompressDemo.zip`, …), which is **not** derivable
from the slug - `MEDIA-03`'s page is `get_video_info` but its zip is
`PageMediaMeta.zip`. Use each card's `sample:` field rather than guessing.

## Pitfalls

The page itself carries no defects. These are the systematic findings raised
against the samples it links to; each is anchored on one card but names
several projects, so check them wherever you enter the set.

- **`HW-13-0012` (B/low, confirmed) - raw file descriptors are never closed.**
  `resourceManager.getRawFd`/`getRawFdSync` without a matching `closeRawFd`,
  in seven samples (HarmonyMusic, AVPlayerAudio, SpeedPlay, VideoPlayList,
  Mirror, opengl-offscreen-render, demo_AudioSession). Anchored on `MEDIA-01`.
- **`HW-13-0015` (B/medium, probable) - descriptors closed while an async
  consumer still holds them.** A `finally` block closes the fd that an
  un-awaited `AVImageGenerator`, `SoundPool`, vibrator or image decode is
  still reading, in five samples. Anchored on `MEDIA-04`.
- **`HW-13-0039` (B/medium, confirmed) - `closeSync` on a possibly-undefined
  file in `finally`.** Turns a routine dialog refusal or a failed open into a
  secondary `TypeError` that masks the original error, in five samples.
  Anchored on `MEDIA-14`.
- **`HW-13-0005` (B/medium, confirmed) - AVPlayer/SoundPool created and never
  released.** Zero `.release()` calls anywhere in five projects, including the
  one whose entire subject is player lifecycle. Anchored on `MEDIA-18`.
- **`HW-13-0032` (B/medium, probable) - `new UIContext()`.** A detached
  instance whose `getHostContext()` has no host, feeding resource loading,
  media-library access and sandbox paths in four samples; document 11 copies
  the same code. Anchored on `MEDIA-11`.
- **`HW-13-0002` (B/medium, confirmed) - `getAssets` with `READ_IMAGEVIDEO`
  declared but never requested at runtime.** The query fails silently; the
  picker URI would have needed no permission at all. Anchored on `MEDIA-03`.
- **`HW-13-0004` (E/low, confirmed) - constraints claim API 20 while the zip
  targets 16 or 17,** in three of these pages. Anchored on `MEDIA-18`.
- **`HW-13-0003` (E/low, confirmed) - documented project trees that do not
  match the zip** in letter case or file extension (`Entryability.ets`,
  `Logger.ets`, `index.d.ets`), which matters because these projects enable
  `caseSensitiveCheck`. Anchored on `MEDIA-13`.

## References

- `huawei_industry_tree/13_media_entertainment/docs/02_practice-audio-app-architecture-v1_1.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1_1-0000002273374777
- `MEDIA-01` - the architecture this list of scenarios fills in
- `MEDIA-03` … `MEDIA-42` - one card per link, in list order
- `huawei_industry_tree/13_media_entertainment/docs/index.json` - the mirrored doc index, including document 43 (行业常见问题, now a redirect to the HarmonyOS FAQ) which this page omits
