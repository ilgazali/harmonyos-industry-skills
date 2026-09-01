---
id: JOBS-01
title: Key scenarios index - the three jobs-industry samples and how they compose into one app
industry: 12_jobs
doc: huawei_industry_tree/12_jobs/docs/01_practice-jobs-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-jobs-app-architecture-v1_1-0000002386519613
sample: none
kits: []
apis: []
permissions: []
min_api: 17
modules: []
findings: []
status: verified
---

## When to use

Load this card when you are **starting a job-search or recruitment app and want
to know which parts Huawei has already worked out**, or when you have landed on
one of the three scenario cards and need to see where it sits. This page is the
industry's table of contents: three links, no prose, no sample.

Its value is not in what it says but in what it implies about the shape of the
product. The three scenarios Huawei chose are the notification ask, the
recommendation deck, and the posting-to-recruiter-to-catalogue flow — the
opening moment, the browsing loop, and the conversion path. That is a complete
skeleton of a job app's first session, and the omissions are as informative as
the inclusions: there is no login, no résumé upload, no chat, no search
backend. Everything the samples show is client-side presentation.

Treat this card as a routing table. The substance lives in `JOBS-02`,
`JOBS-03` and `JOBS-04`, all of which ship real, extracted, compile-checked
projects. Verification here is limited to the links resolving and the titles
matching the pages they point at.

## Feature checklist

What this page promises, and nothing more:

- Three links, each to one key-scenario document in the jobs industry practice
  guide.
- 应用内授权通知弹窗 (in-app notification authorization popup) → `JOBS-02`.
- 推荐岗位-层叠卡片滑动切换动效 (recommended jobs, stacked-card swipe
  animation) → `JOBS-03`.
- 招聘信息及岗位列表展示 (recruitment information and job list display) →
  `JOBS-04`.
- Each target document carries its own scenario intro, implementation notes,
  constraints, project tree and a downloadable sample zip.

## Architecture

There is no project. The page is eleven lines of markdown with a bullet list.

The architecture worth describing is the one the three samples describe
together, because they do not share a codebase but they do share conventions:

```
JOBS-02  NotificationAuthorizationPopup   entry/  pages/MainPage.ets            API 20
JOBS-03  PositionSlidingWindow            entry/  pages/PositionSlidingPage.ets API 20
JOBS-04  RecruitmentInformation           entry/  pages/{Main,JobList,PersonalIntroduction}.ets  API 17
```

Every sample is a single `entry` module with `entryability` +
`entrybackupability` + `pages`, no HAR, no HSP, no shared library, and no
network layer of any kind. Every one of them declares an empty permission set.
Every one of them puts its window into layout-full-screen mode in
`onWindowStageCreate`, reads the two avoid areas, and publishes them into
`AppStorage` under `topRectHeight` / `bottomRectHeight` for the pages to pad
with.

**The design decision worth copying** is that last convention — avoid areas as
`AppStorage` keys — but only in `JOBS-04`'s form, where the ability converts to
vp before storing. `JOBS-02` stores px and converts at the read site, which is
also correct but repeats the conversion. `JOBS-03` stores px and then attempts
to convert into its own `@StorageProp`, a one-way binding whose local write is
overwritten by the next `avoidAreaChange`. Three samples, three treatments of
the same six lines: pick `JOBS-04`'s and apply it everywhere.

**The design decision worth avoiding** is treating these as composable pieces.
They are three separate demos with three different `MainPage`s, and two of them
target a different SDK baseline than the third. Assembling them into one app
means picking a single baseline (API 20, since `JOBS-02` and `JOBS-03` need
it), unifying the avoid-area handling, and giving `JOBS-03`'s deck a real data
source — which it does not have.

## Implementation steps

1. **Start from `JOBS-04`.** It has the navigation host, the `routerMap`
   profile and the multi-page structure, so it is the only one of the three
   that can absorb the others rather than being absorbed.
2. **Fold `JOBS-02`'s authorization ladder into the first page** you actually
   ship, keeping the ask on a screen that shows jobs. Apply `HW-12-0001`'s fix
   before you do: detect `1600004`, persist the refusal, and put the settings
   sheet behind a user action.
3. **Register `JOBS-03`'s deck as a `NavDestination`** rather than an `@Entry`
   page, and apply `HW-12-0002` first — an unobserved card array is not
   something to build on.
4. **Raise the baseline to API 20** across the merged project;
   `JOBS-04`'s `compatibleSdkVersion` of `5.0.5(17)` is the odd one out.
5. **Standardise the avoid-area boilerplate** on the vp-at-the-ability form,
   and release the `avoidAreaChange` subscription in `onWindowStageDestroy` —
   none of the three samples does.
6. **Add the layer none of them have.** All job data in all three samples is a
   hardcoded array in a constants file. There is no repository, no DTO, no
   error state and no loading state anywhere in this industry's samples.

## Verified snippets

**The whole page** — `huawei_industry_tree/12_jobs/docs/01_practice-jobs-app-architecture-v1_1.md`
(from the doc — no sample shipped; not compile-verified)

```markdown
# 关键场景示例

- **[应用内授权通知弹窗](https://developer.huawei.com/consumer/cn/doc/architecture-guides/notification_authorization_popup-0000002386582149)**
- **[推荐岗位-层叠卡片滑动切换动效](https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225)**
- **[招聘信息及岗位列表展示](https://developer.huawei.com/consumer/cn/doc/architecture-guides/recruitment_and_job_list_display-0000002456685322)**
```

All three links resolve, and each target has both a project tree and a code
download. The titles match the samples: the notification popup ships
`NotificationAuthorizationPopup`, the sliding cards ship
`PositionSlidingWindow`, the job list ships `RecruitmentInformation`.

Note that the sample zip for the first scenario is published under a Chinese
filename, `应用内授权通知弹窗示例代码.zip`, while the other two use the ASCII
project names. If you are scripting the downloads for this industry, that
asymmetry will break a naive filename assumption.

## Pitfalls

No defects were found in this document during review. The three links resolve
and name their targets correctly.

The defects in what it links to are recorded on the scenario cards:
`HW-12-0001` (`JOBS-02`, the every-launch settings sheet), `HW-12-0002` and
`HW-12-0003` (`JOBS-03`, the unobserved card array and the stale swipe
direction), and `HW-12-0004` (`JOBS-04`, the invalid font weight). Two of the
three samples ship with a medium-severity defect, so do not treat an index
entry as an endorsement of the code behind it.

## References

- `huawei_industry_tree/12_jobs/docs/01_practice-jobs-app-architecture-v1_1.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-jobs-app-architecture-v1_1-0000002386519613
- `JOBS-02` — in-app notification authorization
- `JOBS-03` — the stacked recommendation deck
- `JOBS-04` — job detail, recruiter and company job list
- `JOBS-05` — the industry FAQ pointer, the other doc-only page in this pack
