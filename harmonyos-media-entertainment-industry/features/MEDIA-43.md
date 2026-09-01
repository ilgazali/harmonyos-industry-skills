---
id: MEDIA-43
title: Industry FAQ page - a redirect stub, and what to use instead for media and entertainment questions
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/43_practice-audio-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1_2-0000002298352333
sample: none
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: [HW-13-0005, HW-13-0012, HW-13-0062]
status: verified
---

## When to use

Load this card when something points you at the media industry's
**行业常见问题** (industry FAQ) page and you need to know what is actually
there. The answer is: nothing. The page is one sentence saying the content has
moved to the HarmonyOS FAQ portal, with a link. It is the third and last part
of the 音乐应用案例 (music application case) architecture series -
`MEDIA-01` is the architecture narrative, `MEDIA-02` is the index of the
thirty-eight scenario samples, and this page is the FAQ slot that was emptied.

The card exists so that a reader who lands here does not spend time chasing a
document that carries no technical content, and so that the questions the FAQ
slot would have answered have somewhere concrete to go. The three most useful
answers for this industry are not in any Huawei FAQ: they are the three
defects that repeat across the industry's own sample projects, catalogued
below.

**Verification level is low by construction.** There is no sample zip, no code
and no API surface on this page - nothing here was compiled or run. Everything
in the "What the FAQ slot should answer" section comes from reading the other
samples in this industry, and is cross-referenced to the cards that were
verified against their zips.

## Feature checklist

What the page itself promises, in full:

- A heading, 行业常见问题.
- One sentence stating that the industry FAQ content has been migrated to the
  HarmonyOS FAQ, with a 此处 (here) link to the phone FAQ index.

What it does **not** provide:

- No question or answer text of its own, so nothing on the page is searchable
  or citable.
- No anchor or deep link into the FAQ portal - the target is the generic phone
  FAQ index, not a media-and-entertainment section.
- No code, no sample project, no constraints section, no project tree.

## Architecture

Documentation-only page; there is no module, no zip and no downloadable code.
Its place in the series:

```
13_media_entertainment/docs
├── 01_practice-audio-app-architecture-v1.md     音乐应用案例 — layering, module split, software view
├── 02_practice-audio-app-architecture-v1_1.md   关键场景示例 — index of the 38 scenario pages
└── 43_practice-audio-app-architecture-v1_2.md   行业常见问题 — this page, a redirect stub
```

The three share one `practice-audio-app-architecture-v1*` document family and
are meant to be read in order: the architecture, then the scenarios that
implement it, then the questions that come up while doing so. Only the first
two carry content.

**The design decision worth avoiding** is the empty redirect itself. A page
that survives in the navigation tree, keeps its slot in the series and its
position in the sidebar, but forwards to an index rather than to its own
migrated content, costs a reader a click and a search for every question they
arrive with. If a section moves, the pointer should be as specific as what it
replaced - a link to the media FAQ section, or better, the migrated questions
inlined. As it stands, the page's only structural function is to keep the
series numbering intact.

## Implementation steps

There is nothing to implement. What to do instead, in order:

1. **Decide whether the question is about a scenario.** If it is - subtitles,
   headset disconnect, waveform animation, playlist switching, screen
   recording - go to `MEDIA-02`'s index or straight to the feature card; each
   one names its sample zip and the defects found in it.
2. **Decide whether it is about structure.** Layering, HAR versus HAP module
   split, which functionality belongs in the common capability layer: that is
   `MEDIA-01`.
3. **Check the three systematics below before writing new playback code.** They
   are the questions this industry's samples actually raise, and every one of
   them is reproduced in more than one official sample - so code copied from
   any of them inherits the defect.
4. **Only then follow the link** to the HarmonyOS phone FAQ portal, and expect
   to search it rather than land on a media section.

## What the FAQ slot should answer

These are the cross-sample defects found while reviewing this industry. They
are not defects of *this* page - they are the answers a media FAQ would be
most useful for, and each is verified against the sample zips named in it.

**`HW-13-0005` — "why does my app slow down after playing a few videos?"**
Five samples create an `AVPlayer` or a `SoundPool` and never call `release()`
anywhere in the project: `VideoPlayList`, `VideoPlayerResumeDemo` (a sample
whose entire subject is player lifecycle), `XComponentTransition`,
`VideoSubtitle` and `SoundPoolBeatDemo`. Players hold native decoder and audio
resources; the official playback guides end every lifecycle with `release()`.
Release each player in the page's `aboutToDisappear` or the ability's
`onDestroy`. See `MEDIA-18`, `MEDIA-24`, `MEDIA-41`.

**`HW-13-0012` — "why do file descriptors accumulate until the process dies?"**
Seven samples take a descriptor with `resourceManager.getRawFd` or
`getRawFdSync` and never release it with `closeRawFd`: `HarmonyMusic` (per
track change), `AVPlayerAudio`, `SpeedPlay` (per rawfile video), `VideoPlayList`
(per swipe), `Mirror`, `opengl-offscreen-render` and `demo_AudioSession`. The
last of these also closes a resource-manager descriptor with `fs.closeSync`,
which is the wrong call for it. Raw HAP-resource descriptors live for the
lifetime of the process - unbounded in a feed-style player. Pair every
`getRawFd` with a `closeRawFd` after the consumer has been released. See
`MEDIA-01`, `MEDIA-42`.

**`HW-13-0062` — "why does the end of every track click or echo?"**
Four samples render the final partial audio chunk without clearing what was in
the buffer before it: `AudioVisualization`, `PCMTranscode`, `demo_AudioSession`
and the native `demo_HttpAudioRender`. The shapes differ - an ignored
`readSync` return, an unclamped read length, an `fread` with `size`/`nmemb`
transposed - but the symptom is one frame of stale audio at the end of every
playback. Zero the remainder of the buffer past the bytes actually read, or
render only those bytes. See `MEDIA-27`, `MEDIA-36`, `MEDIA-30`, `MEDIA-42`.

## Verified snippets

None. There is no sample and no code on this page. Its entire body, quoted
from the document (from the doc - no sample shipped; not compile-verified):

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

The one thing worth reading out of it is the target: `harmonyos-faqs/faq-phone`
is the phone FAQ index for all industries, not a media-and-entertainment
section. There is no fragment identifier, so the link cannot land on the
migrated questions even if they still exist under that root.

## Constraints

- No sample, no API surface, no minimum API version - nothing on this page is
  version-dependent.
- The linked FAQ portal is external and versioned independently of this
  document tree; it is not mirrored under `documentation/` in this repo, so it
  cannot be consulted offline.
- The claims in "What the FAQ slot should answer" are verified against the
  named sample zips, not against this page.

## Pitfalls

- **The page is a dead-end pointer.** It keeps a slot in the architecture
  series and in the sidebar while carrying no content, and its link resolves to
  a generic phone FAQ index with no media section and no anchor. Treat it as
  navigation, never as a source.
- **`HW-13-0005` (B/medium, confirmed)** — systematic: five media samples
  create an `AVPlayer`/`SoundPool` and never release it. Fix: release in the
  page or ability teardown.
- **`HW-13-0012` (B/low, confirmed)** — systematic: `getRawFd`/`getRawFdSync`
  descriptors are never closed with `closeRawFd` in seven media samples. Fix:
  pair every acquisition with `closeRawFd` after the player is released.
- **`HW-13-0062` (B/low, confirmed)** — systematic: the final partial audio
  chunk leaves stale bytes in the render buffer in four samples. Fix: zero the
  tail, or render only the bytes actually read.
- **Do not cite this page as the source of an answer.** Anything attributed to
  the "industry FAQ" has to be traced to the FAQ portal or to one of the
  scenario documents; this page cannot corroborate it.

## References

- `huawei_industry_tree/13_media_entertainment/docs/43_practice-audio-app-architecture-v1_2.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1_2-0000002298352333
- `MEDIA-01` - 音乐应用案例, the architecture narrative this page closes
- `MEDIA-02` - 关键场景示例, the index of the industry's scenario samples
- `MEDIA-18`, `MEDIA-24`, `MEDIA-41` - instances of the unreleased-player systematic
- `MEDIA-27`, `MEDIA-30`, `MEDIA-36`, `MEDIA-42` - instances of the stale-tail render-buffer systematic
