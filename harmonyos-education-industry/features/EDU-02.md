---
id: EDU-02
title: Key scenario index - the catalogue of the eighteen education feature documents
industry: 04_education
doc: huawei_industry_tree/04_education/docs/02_practice-educate-app-architecture_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture_1-0000002328370237
sample: none (index document)
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Load this card only to **find the right feature card**. `02_关键场景示例`
(key scenario examples) is the navigation page of the education industry: it
carries nothing but a list of the eighteen feature documents that hang off the
architecture practice in `EDU-01`.

There is no code, no zip and no technique here. If you know what you are
building, go straight to the feature card; if you do not, read the map below.

## Feature checklist

Nothing to implement. This document is verified as an index: all eighteen links
resolve to a document present in this industry, and no feature document in the
industry is missing from the list.

## Architecture

The industry has three document kinds, and this is the middle one:

```
EDU-01  practice-educate-app-architecture-v1     the architecture + framework code (has a zip)
EDU-02  practice-educate-app-architecture_1      THIS - the index of the 18 feature docs
EDU-03..EDU-20                                   one feature each (each has its own zip)
EDU-21  practice-educate-app-architecture-v1_2   the FAQ page, now a redirect
```

The map, by the product problem each one solves:

| Card | 中文标题 | What it is for |
| --- | --- | --- |
| `EDU-03` | 基于DataPanel实现双向滑块调节分数区间 | Two-handle range slider over a DataPanel - filter schools by score band |
| `EDU-04` | 双向滚动课程表 | A timetable that scrolls in both axes with locked header and gutter |
| `EDU-05` | 人物介绍展开与收起 | Expand/collapse a long biography with a trailing "more" control |
| `EDU-06` | 堆叠式单词卡片 | Stacked, swipeable word cards |
| `EDU-07` | 刷题页面滑动 | Swipe between practice questions |
| `EDU-08` | 试题PDF转长图保存 | Render a PDF paper into one long image and save it |
| `EDU-09` | PDF文件导入与打开 | Import a PDF from the file picker and open it |
| `EDU-10` | 视频课程弹幕发送与设置 | Bullet comments over a course video, with settings |
| `EDU-11` | 口语跟读训练 | Read-aloud practice - record, play back, score |
| `EDU-12` | 网络状态实时呈现 | Live network status in the UI |
| `EDU-13` | 课程提醒日历同步 | Write a course reminder into the system calendar |
| `EDU-14` | 课程学习进度展示 | Course progress display |
| `EDU-15` | 学籍级联选择 | Cascading province/city/district or school-year picker |
| `EDU-16` | 媒体设备状态检查 | Check camera and microphone availability before a lesson |
| `EDU-17` | 单词拼写练习 | Spelling drill |
| `EDU-18` | 学习进度折线图统计 | Study-time line chart |
| `EDU-19` | 球状标签动画 | Spherical tag-cloud animation |
| `EDU-20` | 扫描提交作业 | Scan homework with the camera and submit it |

## Implementation steps

None. To use the index:

1. Identify the product problem.
2. Take the card from the table above.
3. Read `EDU-01` first for module boundaries, `AppStorage`-published
   `NavPathStack` routing and the permission baseline - every feature card
   assumes that shell.

## Verified snippets

None - the document contains no code.

## Permissions & config

None.

## Constraints

- The document is a link list only: no 简介 (introduction), no code, no zip, and
  the crawl records `images: 0`.
- The eighteen links point at the public `architecture-guides` site. The local
  copies under `huawei_industry_tree/04_education/docs/` are the crawl of those
  same pages; each local file's `source:` frontmatter is the URL listed here.
- The list does **not** include `EDU-01` or `EDU-21`, which are siblings of this
  page rather than entries under it.

## Pitfalls

None found. Every one of the eighteen URLs matches the `source:` frontmatter of
a document present in this industry, and every feature document in the industry
appears in the list - the index is neither stale nor short.

## References

- `EDU-01` - the architecture practice and framework code these eighteen features plug into
- `EDU-21` - the sibling FAQ page, now a redirect to the HarmonyOS FAQ
