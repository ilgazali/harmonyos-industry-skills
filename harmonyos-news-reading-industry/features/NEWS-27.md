---
id: NEWS-27
title: Industry FAQ - the news-reading practice's third page, now a redirect to the HarmonyOS FAQ portal
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/27_practice-news-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1_2-0000002298237753
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

Load this card when you have followed a link to 行业常见问题 (industry FAQ) from
the news-reading practice and want to know whether there is anything behind it.
There is not. **The page is a one-line redirect**: its entire body says the FAQ
content has moved to the HarmonyOS FAQ site and offers a link to
`harmonyos-faqs/faq-phone`. No scenario, no snippet, no sample zip.

Its value is positional, not informational. The news-reading practice is a
three-page frame - `practice-news-app-architecture-v1` (the architecture),
`v1_1` (the index of key scenarios), `v1_2` (this FAQ stub) - and the
twenty-four feature documents hang off the middle one. Knowing that the third
page is empty is what stops you looking for industry-specific answers that were
never written here.

So use this card as the **navigation entry** for the industry: it maps the
three architecture pages onto the cards that carry real content, and tells you
which card to open for a given problem. When you actually need a HarmonyOS
answer, the FAQ portal it points at is a general phone-development FAQ with no
news-reading section - the industry-specific knowledge lives in the sibling
cards, not there.

Verification level for this card is **document-only**. There is no zip, nothing
was compiled, and the single claim the page makes (that a link resolves) is all
that could be checked.

## Feature checklist

What the page promises, in full:

- The 行业常见问题 heading exists in the practice's table of contents.
- The body states that the FAQ content has migrated to HarmonyOS FAQ.
- A 此处 (here) link points to `developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone`.

That is the complete contract. Anything a reader expects beyond it - known
issues for reader apps, Reader Kit troubleshooting, speech-panel caveats -
is not on this page and never was.

## Architecture

Documentation-only. No project, no module, no `module.json5`.

The structure worth carrying away is the practice's own:

```
11_news_reading/docs
├── 01_practice-news-app-architecture-v1.md     NEWS-01  the architecture narrative
│                                                        (modules, layering, HAP/HAR split)
├── 02_practice-news-app-architecture-v1_1.md   NEWS-02  the index: 24 links, nothing else
├── 03_..26_ (24 scenario documents)            NEWS-03..NEWS-26
└── 27_practice-news-app-architecture-v1_2.md   NEWS-27  this page: a redirect
```

`01` is the only one of the three that carries design content: the four-module
split (新闻 news / 视频 video / 直播 live / 我的 profile), the layering rule
that product customisation is one phone HAP while feature modules and common
capabilities are HARs, and the logical view of the dependencies. `02` is a pure
link list - the ordering of `NEWS-03` through `NEWS-26` in this pack is `02`'s
ordering. `27` is this stub.

**The design decision worth copying** is the frame itself, not this page: one
architecture document that fixes the module boundaries, one index that stays
flat while scenarios are appended, and scenario pages that each ship a
standalone runnable zip. It is why twenty-four features could be added over
time without the architecture page changing - and, less happily, why nothing
reconciles them. Each zip is its own single-`entry` project; none of them
demonstrates the HAR layering that `01` prescribes.

## Implementation steps

There is nothing to implement. The steps below are how to use the pack.

1. **Read `NEWS-01` first** for the module split and the HAP/HAR layering; it
   is the only page that states the intended shape of the application.
2. **Treat `NEWS-02` as the catalog** - it is the industry's own ordering of
   the scenarios, mirrored by the card numbering here.
3. **Pick the card by problem, not by document title** (see the map below).
4. **Do not look here for FAQs.** The link goes to the general phone
   development FAQ; nothing on it is specific to news or reading apps.
5. **Expect single-module samples.** Every scenario zip in this industry is one
   `entry` module, so lifting two features into one app means merging two
   projects, not adding two HARs.

## Verified snippets

*(from the doc — no sample shipped; not compile-verified)*

**The entire body of `27_practice-news-app-architecture-v1_2.md`:**

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

Eleven lines including the front matter, of which one is content: "the industry
FAQ content has been migrated to HarmonyOS FAQ, please click *here* to go
there." No 场景介绍, no 约束与限制, no 工程目录, no 代码下载 - the four sections
every other document in this industry carries.

**The catalog it should have pointed at — from `02_practice-news-app-architecture-v1_1.md`:**

```markdown
- **[未成年人内容过滤](https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325)**
- **[频道选择](https://developer.huawei.com/consumer/cn/doc/architecture-guides/channel_selection-0000002270325497)**
- **[字体大小调节](https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282)**
- ...
- **[小说阅读自动翻页](https://developer.huawei.com/consumer/cn/doc/architecture-guides/automatic_page_turn-0000002415970301)**
- **[定时关闭文字转语音播报](https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_reader-0000002525367690)**
```

Twenty-four bullets, each a bare link, in the order reproduced by this pack's
card numbers. When a new scenario is published it is appended here, which is
why the last two entries - auto page turn and the speech sleep timer - are also
the two highest card numbers with samples.

## Card map

| Problem | Cards |
| --- | --- |
| Overall structure, module split, layering | `NEWS-01`, `NEWS-02` |
| Reading surface: font size, flip mode, magnifier | `NEWS-05`, `NEWS-07`, `NEWS-16`, `NEWS-18` |
| Reader Kit: preset books, bookshelf, auto page turn | `NEWS-14`, `NEWS-22`, `NEWS-25` |
| Page turning by input: volume keys, timer | `NEWS-12`, `NEWS-25` |
| Speech: read-aloud panel, sleep timer | `NEWS-09`, `NEWS-26` |
| Text selection, marking, highlighting, OCR | `NEWS-13`, `NEWS-17`, `NEWS-19`, `NEWS-23` |
| Feed and list: channels, ads, hot search, scroll-to-top | `NEWS-04`, `NEWS-08`, `NEWS-10`, `NEWS-11` |
| Content compliance and accounts | `NEWS-03` |
| Media and sharing: Base64 save, article editor, cards | `NEWS-06`, `NEWS-20`, `NEWS-24` |
| Statistics and effects | `NEWS-15`, `NEWS-21` |

## Constraints

- The document has no 约束与限制 section - the API and SDK floors it would
  normally state are absent here. Take them from the scenario card you are
  actually using; every zip-backed card in this industry is API Version 20
  Release / HarmonyOS 6.0.0 Release SDK / DevEco Studio 6.0.0 Release.
- The target of the link is the general phone FAQ, not a news-reading FAQ. It
  is not versioned alongside this practice and can change without any change
  here.
- Nothing on this page was compile-verified, because there is nothing to
  compile.
- The industry's samples do not follow `NEWS-01`'s own layering guidance: all
  are single-`entry` projects with no HARs.

## Pitfalls

- **No defects were filed against this document.** The redirect is accurate and
  the link resolves; there is nothing else to be wrong.
- **Do not treat an empty FAQ as "no known issues".** Twenty-five of this
  industry's twenty-eight findings sit on the scenario documents that `02`
  indexes (the other three are on the architecture page itself) - notably permission blocks copy-pasted between unrelated
  samples (`HW-11-0027`) and background declarations without the matching
  continuous task (`HW-11-0028`, hit twice). The absence of an FAQ page means
  those were never collected anywhere upstream.
- **Do not cite this page as an architecture source.** `NEWS-01` is the
  architecture; this is its third tab.

## References

- `huawei_industry_tree/11_news_reading/docs/27_practice-news-app-architecture-v1_2.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1_2-0000002298237753
- `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md` - the architecture page behind `NEWS-01`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- `huawei_industry_tree/11_news_reading/docs/02_practice-news-app-architecture-v1_1.md` - the scenario index behind `NEWS-02`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1_1-0000002231929048
- `NEWS-01` - module split, layering, the HAP/HAR decision
- `NEWS-02` - the scenario catalog in the industry's own order
