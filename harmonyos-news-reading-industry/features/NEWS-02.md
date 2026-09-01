---
id: NEWS-02
title: Key scenario index - the 24 news and reading samples that plug into the app skeleton
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/02_practice-news-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1_1-0000002231929048
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

Load this card when you know **what** you want to build in a news or reading
app but not **which** of the industry's samples covers it. This page is the
industry's table of contents: 24 links, each to one scenario document with its
own downloadable project, sitting between the architecture narrative
(`NEWS-01`) and the FAQ page (`NEWS-27`).

It is a navigation page, so it carries no code and no project of its own.
Its value is the shape of the list: it tells you what Huawei considers the
solved problems of this industry, and by omission what it does not. Read as a
group, the 24 scenarios split cleanly into four concerns - compliance,
reading ergonomics, text interaction, and shelf/session management - and if
your feature does not fall into one of those, no sample exists for it.

Because the page is a link list, the verification level here is lower than for
a card with a zip: nothing was compiled, and what is asserted below is the
mapping between the list and the cards in this pack, checked link by link
against `huawei_industry_tree/11_news_reading/docs/`.

## Feature checklist

What the page promises, and what was checked:

- 24 links, all to `developer.huawei.com/consumer/cn/doc/architecture-guides/`.
- Each link resolves to a scenario document that exists in this repo under
  `huawei_industry_tree/11_news_reading/docs/`.
- The list order matches the crawl order, so link *n* is document `NN = n + 2`
  and card `NEWS-(n+2)`.
- Every scenario document except the two architecture pages and the FAQ page is
  reachable from here - no orphans in the industry tree.
- The page itself declares no scenario, no constraints and no environment
  requirements; each linked document carries its own.

## Architecture

Documentation only. No project, no module, no `module.json5`.

Structurally the industry tree is three architecture pages plus 24 scenarios:

```
huawei_industry_tree/11_news_reading/docs
├── 01_practice-news-app-architecture-v1.md      the skeleton + NewsSolutionDemo   -> NEWS-01
├── 02_practice-news-app-architecture-v1_1.md    THIS PAGE: the scenario index     -> NEWS-02
├── 03..26_*.md                                  24 scenarios, each with a zip     -> NEWS-03 .. NEWS-26
└── 27_practice-news-app-architecture-v1_2.md    industry FAQ (redirects out)      -> NEWS-27
```

**The design decision worth copying** - in the documentation, not the code - is
that the index holds nothing but links. It states no constraint, repeats no
snippet and gives no summary, so it cannot drift out of step with the pages it
points at. Compare `NEWS-01`, whose embedded project tree *does* duplicate
information from its sample and *has* drifted (`HW-11-0003`). An index that
only indexes is an index that stays correct.

The cost is that the list is unordered and unlabelled: 24 Chinese titles with
no grouping, so finding "the one about highlighting" means reading all of them.
The table below is the grouping the page does not provide.

## Implementation steps

1. **Start from `NEWS-01`** for the module layout, the feed and the navigation
   stack. Every scenario below assumes a host app of roughly that shape.
2. **Find your concern in the table**, then load that card - not this one.
   Each scenario card carries its own zip-verified snippets, constraints and
   findings.
3. **Check the API baseline before mixing.** `NEWS-01` targets API 24 /
   DevEco 6.1.1; the scenario samples target API 20 / DevEco 6.0.0 and are
   written against a plain `entry` module, so lifting one into the layered
   skeleton means moving its code into a feature HAR and re-pointing its
   imports at `@ohos/common`.
4. **Expect to supply the data layer yourself.** Every sample in this industry
   ships static or mock data; none of them makes a network call.

## Verified snippets

*(from the doc — no sample shipped; not compile-verified)*

The entire body of the page is one list. This is its first three entries,
verbatim:

```markdown
- **[未成年人内容过滤](https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325)**
- **[频道选择](https://developer.huawei.com/consumer/cn/doc/architecture-guides/channel_selection-0000002270325497)**
- **[字体大小调节](https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282)**
```

The URL slug is the stable identifier, not the title: `new_minors_protection`,
`channel_selection`, `set_app_font_size`. Titles are Chinese and have been
edited between revisions; slugs have not. When matching a card to a Huawei page,
match on the slug.

Resolved against this repo, the 24 links are:

| # | Title | Slug | Doc | Card |
| --- | --- | --- | --- | --- |
| 1 | 未成年人内容过滤 (minors content filtering) | `new_minors_protection` | `03_` | `NEWS-03` |
| 2 | 频道选择 (channel selection) | `channel_selection` | `04_` | `NEWS-04` |
| 3 | 字体大小调节 (font size adjustment) | `set_app_font_size` | `05_` | `NEWS-05` |
| 4 | Base64格式图片保存 (save a Base64 image) | `base64_image_save` | `06_` | `NEWS-06` |
| 5 | 阅读翻页方式调节 (page-turn mode) | `page_flip_page` | `07_` | `NEWS-07` |
| 6 | 返回阅读列表顶部 (back to top) | `move_to_top` | `08_` | `NEWS-08` |
| 7 | AI朗读 (AI read-aloud) | `ai_recitation` | `09_` | `NEWS-09` |
| 8 | 广告窗口插入 (ad insertion) | `ad_loading` | `10_` | `NEWS-10` |
| 9 | 构建热搜榜单列表页 (trending list page) | `hot_search` | `11_` | `NEWS-11` |
| 10 | 音量键翻页 (volume-key paging) | `volume_key_turn_page` | `12_` | `NEWS-12` |
| 11 | 文本标记高亮显示 (text marking) | `text_marker_ability` | `13_` | `NEWS-13` |
| 12 | 加载预置数据库刷新文章列表 (preset database) | `preset_book` | `14_` | `NEWS-14` |
| 13 | 打字机效果 (typewriter effect) | `typewriter_effect` | `15_` | `NEWS-15` |
| 14 | H5页面适配应用内字体大小设置 (H5 font size) | `h5_fontsize` | `16_` | `NEWS-16` |
| 15 | 正则匹配高亮关键字 (regex keyword highlight) | `regular_highlight` | `17_` | `NEWS-17` |
| 16 | 阅读放大镜 (reading magnifier) | `magnifier` | `18_` | `NEWS-18` |
| 17 | 涂抹识别文字并复制 (smear to recognise and copy) | `erase_recognize` | `19_` | `NEWS-19` |
| 18 | 阅读记录卡片 (reading record card) | `read_card` | `20_` | `NEWS-20` |
| 19 | 阅读时长看板 (reading time dashboard) | `time_statistics` | `21_` | `NEWS-21` |
| 20 | 书架设置 (bookshelf import, delete, search) | `bookshelf_demo` | `22_` | `NEWS-22` |
| 21 | 小说段落评论 (per-paragraph comments) | `novel_read_review` | `23_` | `NEWS-23` |
| 22 | 图文动态编辑发布 (rich post composer) | `auto_flip_read` | `24_` | `NEWS-24` |
| 23 | 小说阅读自动翻页 (auto page turn) | `automatic_page_turn` | `25_` | `NEWS-25` |
| 24 | 定时关闭文字转语音播报 (TTS sleep timer) | `text_reader` | `26_` | `NEWS-26` |

Grouped by concern, which is how to actually use the list:

- **Compliance and account** - `NEWS-03` (minors mode), `NEWS-10` (ads).
- **Reading ergonomics** - `NEWS-05`, `NEWS-16` (font size, in-app and in H5),
  `NEWS-07`, `NEWS-12`, `NEWS-25` (page turning by gesture, volume key, timer),
  `NEWS-18` (magnifier), `NEWS-09`, `NEWS-26` (read-aloud and its sleep timer).
- **Text interaction** - `NEWS-13` (marking), `NEWS-17` (regex highlight),
  `NEWS-19` (smear-to-OCR), `NEWS-15` (typewriter), `NEWS-23` (paragraph
  comments), `NEWS-24` (composing a post).
- **Shelf, feed and session** - `NEWS-04` (channels), `NEWS-08` (back to top),
  `NEWS-11` (trending), `NEWS-14` (preset database), `NEWS-20` (record card),
  `NEWS-21` (time dashboard), `NEWS-22` (bookshelf), `NEWS-06` (image save).

Entry 22 is worth a note: the title is 图文动态编辑发布 (a rich-text post
composer) but the slug is `auto_flip_read`. The document behind it is the
composer, so the slug is a leftover from a page that was repurposed. Match on
the title for that one.

## Pitfalls

- **No defects found.** Every link resolves, the order matches the document
  numbering, and the page asserts nothing that could be wrong. The
  slug/title mismatch on entry 22 is on Huawei's side of the link and does not
  break navigation.
- The page is a hub with no content of its own, so it should never be cited as
  the source for a technique. Cite the scenario card.

## References

- `huawei_industry_tree/11_news_reading/docs/02_practice-news-app-architecture-v1_1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1_1-0000002231929048
- `NEWS-01` - the architecture and the framework sample the 24 scenarios extend
- `NEWS-27` - the industry FAQ page, a sibling of this index rather than an entry in it
- `huawei_industry_tree/11_news_reading/docs/index.json` - the crawl manifest that fixes the document numbering used above
