---
id: OFFICE-02
title: Key-scenario index for the office industry - the 29 scenario guides and what each one solves
industry: 05_office
doc: huawei_industry_tree/05_office/docs/02_practice-office-app-architecture-v1-5_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-5_1-0000002267128277
sample: none (architecture / index document)
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when you know **what office behaviour you need** but not **which
scenario guide describes it**. The source document is the 关键场景示例 ("key
scenario examples") index of the office-industry architecture guide: a flat list
of 29 links, one per scenario, with no prose of its own.

Use it as a routing table: pick the row that matches the requirement, then load
the matching `OFFICE-nn` card for the verified implementation. It carries no
code and no configuration - the scenario cards do.

## Feature checklist

This document is an index, so the "feature" it delivers is coverage. It must:

- List every key scenario the office-industry guide ships, with a stable link to
  each one.
- Stay in sync with the scenario pages: no dead entry, no scenario page missing
  from the list.
- Sit under the same architecture-guide parent as the framework document
  (OFFICE-01) and the FAQ document (OFFICE-32), which are deliberately not part
  of this list.

Verified: the document contains exactly 29 links, and they map one-to-one onto
`03_location_permissions.md` through `31_schedule_share.md` in this repository.
Every link title matches the `title:` frontmatter of its target page, and every
URL matches the target's `source:` URL. No entry is missing and no entry points
at a page that does not exist.

## Architecture

There is no runtime architecture. The document sits at one level of the office
architecture-guide tree:

```
practice-office-app-architecture-v1        综合办公应用案例   -> OFFICE-01 (framework code + ZIP)
practice-office-app-architecture-v1-5_1    关键场景示例       -> OFFICE-02 (this index)
  |- location_permissions                  考勤打卡位置获取    -> OFFICE-03
  |- ... 27 more scenarios ...
  '- schedule_share                        协同办公日程管理    -> OFFICE-31
practice-office-app-architecture-v1_2      行业常见问题       -> OFFICE-32 (FAQ)
```

The three architecture-level documents (OFFICE-01, OFFICE-02, OFFICE-32) ship no
sample project of their own; each of the 29 scenario documents does.

## Implementation steps

There is nothing to implement from this document. The step is selection:

1. Identify the capability group your requirement falls into (see the table
   below).
2. Open the corresponding scenario document under
   `huawei_industry_tree/05_office/docs/`.
3. Load the matching `OFFICE-nn` feature card in this skill for the verified
   snippets, permission list and known pitfalls.
4. If the requirement spans several scenarios (for example "meeting invite that
   also writes a system schedule and shows a banner notification"), load each
   card; they were reviewed independently and their permission lists are
   additive.

Scenario map, grouped by capability:

| Group | Scenarios (document number - title) |
| --- | --- |
| Attendance & location | 03 考勤打卡位置获取 (attendance check-in location) |
| Documents & PDF | 04 PDF在线安全预览 (secure online PDF preview); 06 公文审批-画板签名、文件预览下载 (approval with canvas signature); 08 文件下载预览 (file download and preview); 12 电子章添加和保存 (e-stamp add and save); 13 选择文件打开方式 (choose how to open a file); 28 PDF预览模式下添加或删除批注 (add/delete PDF annotations) |
| Mail & messaging | 10 邮件附件添加和预览 (mail attachment add and preview); 15 消息稍后处理 (handle message later); 25 群聊页面置顶公告 (pinned group announcement); 14 跑马灯横幅通知 (marquee banner notification) |
| Contacts & organisation | 05 个人名片页 (personal card page); 17 多层级嵌套企业通讯录 (nested corporate directory); 22 添加特别关注 (special following); 26 多级组织架构菜单 (multi-level org chart); 29 来电显示企业员工信息 (caller identity from employee info) |
| Camera & imaging | 07 证件照拍摄-蒙版效果 (ID photo capture with mask); 16 证件照标签设置及推荐 (ID photo tag recommendation); 19 水印相机 (watermark camera); 09 自动生成访客管理二维码 (visitor QR code) |
| Notes & voice | 20 笔记中图片插入及编辑 (insert and edit images in notes); 21 语音笔记播放 (voice note playback); 23 录音笔记实时转文字 (live speech-to-text notes) |
| Meetings & schedule | 11 自定义弹窗发布会议并添加系统日程 (publish meeting, add system schedule); 18 线上会议主副窗口切换 (main/sub window switching); 27 待办事项批量同步至日历 (batch sync to-dos to calendar); 31 协同办公日程管理 (collaborative schedule management) |
| Security & compliance | 24 应用背景水印 (application background watermark); 30 考试切后台提示 (exam background-switch warning) |

## Verified snippets

None. The document contains no code block, and it ships no sample project, so
there is no ZIP to quote from. Every snippet for this industry lives in the
scenario cards OFFICE-03 through OFFICE-31.

## Permissions & config

None. The document declares no permission and no `module.json5` fragment.
Permissions are declared per scenario; the ones that appear across this industry
and are verified in the individual cards include calendar read/write, reminder
publication, location, camera and media access. Take each list from the scenario
card you are actually implementing - do not aggregate them here.

## Constraints

- The document is an index only: it has no 简介, no 实现思路 and no
  代码下载链接 section, which is expected for this page type and is not a defect.
- It covers the 29 scenario pages only. The framework code and its ZIP are in
  OFFICE-01; industry FAQ material is in OFFICE-32. Neither is reachable from
  this page.
- Two entries carry a `#一多适配` ("multi-device adaptation") suffix in their
  link text (20 笔记中图片插入及编辑, 21 语音笔记播放). The suffix is part of the
  target pages' own titles, so it is a scope marker on those scenarios, not a
  broken anchor.
- The links are absolute `https://developer.huawei.com/consumer/cn/...` URLs and
  point at the Simplified Chinese site.

## Pitfalls

No findings were recorded for this document. The checks that were run and
passed:

- Link count against scenario document count: 29 links, 29 scenario documents,
  one-to-one.
- Link target existence: every URL matches the `source:` frontmatter of an
  existing document in `huawei_industry_tree/05_office/docs/`.
- Link text against target title: every link text equals the target document's
  `title:` frontmatter.
- Scope boundary: the three architecture-level documents are correctly excluded
  from the list.

## References

- `huawei_industry_tree/05_office/docs/02_practice-office-app-architecture-v1-5_1.md` -
  the index document itself.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-5_1-0000002267128277
- `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md` -
  the office framework document (OFFICE-01), the parent of this index.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- `huawei_industry_tree/05_office/docs/32_practice-office-app-architecture-v1_2.md` -
  the industry FAQ document (OFFICE-32), the sibling of this index.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1_2-0000002263504048
- `huawei_industry_tree/05_office/docs/03_location_permissions.md` through
  `31_schedule_share.md` - the 29 link targets verified above.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
