---
id: MAT-03
title: Baby growth record timeline
industry: 10_maternity_health
doc: huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
sample: huawei_industry_tree/10_maternity_health/downloads/RecordTimeLine.zip
kits: ["@kit.ArkUI", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItemGroup, NavPathStack.pushPathByName, NavPathStack.pop, photoAccessHelper.PhotoViewPicker, photoAccessHelper.PhotoSelectOptions, Intl.DateTimeFormat, bindSheet, DatePicker, PromptAction.showToast]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-10-0015, HW-10-0016, HW-10-0017, HW-10-0018, HW-10-0019, HW-10-0020, HW-10-0021, HW-10-0022, HW-10-0023, HW-10-0033]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to present **dated user records grouped under
date headers on a vertical timeline** - baby growth logs, feeding records,
symptom diaries, antenatal visit history, medication logs. The pattern is not
specific to maternity: any "add an entry, see it grouped by day" feature uses
the same `List` + `ListItemGroup` structure.

It handles two record shapes on one timeline: a **structured measurement**
(height and weight) and a **free-form diary** (text plus images).

## Feature checklist

- A timeline where each date appears once as a group header with a visual node,
  and all of that date's records hang under it.
- Newest date first.
- Two entry paths from a toolbar: a measurement form and a diary composer.
- Both entry pages let the user pick the record date, defaulting to today.
- The diary attaches images through the system picker, with no permission
  prompt.
- Tapping an attached image opens a full-screen previewer that can be swiped.
- The measurement form validates its input and refuses out-of-range values.
- The toolbar background becomes opaque once the content scrolls under it.

## Architecture

Single `entry` module - this is a feature sample, not an app skeleton.

```
entry/src/main/ets
├── components/BindSheetComponent.ets   date picker sheet
├── components/ImageViewerComponent.ets full-screen image previewer
├── model/DataInfoModel.ets             IRecordList, IDateList, HealthInfo, popRes
├── pages/TimeLine.ets                  host page, owns the NavPathStack
├── pages/GrowthRecord.ets              height and weight form
├── pages/Diary.ets                     text and image composer
└── utils/DateUtils.ets                 format and group
    utils/ImgUtil.ets                   PhotoViewPicker wrapper
```

Data flow is **push-with-callback**, not shared state. `TimeLine` owns
`@Provide('pageInfo') pathStack`. It pushes an entry page with
`pushPathByName(name, param, onPop)`; the entry page collects input and returns
it with `pathStack.pop(result)`. The `onPop` callback then folds the result into
the timeline. The entry pages hold no reference to the timeline's data.

> The `onPop` callback **fires only when `pop()` is called with a result** - the
> reference states it is "triggered only when the result parameter is set in
> pop, popToName, or popToIndex". A system back gesture therefore does not
> invoke it, which is why the sample can dereference `res.result` without a
> guard. Keep that contract if you copy this pattern: if you ever add a
> `pop()` without a result on the cancel path, the callback still will not
> fire, but any `pop(someResult)` on cancel would reach the same handler.

## Implementation steps

1. **Model three shapes.** `IRecordList` is one record (user, optional
   `healthyInfo`, optional `textContent`, optional `imageContent`, date, time);
   `IDateList` is `{ date, content: IRecordList[] }`, one group; `popRes` is the
   payload the entry pages return.
2. **Push the entry page with an `onPop` callback** and fold the result into the
   record list from inside that callback.
3. **Keep a machine-readable date key on every record** and group on it. Do not
   group on the formatted display string (`HW-10-0018`).
4. **Render with `List` + `ListItemGroup`**, one group per date, the date header
   as the `header` builder, and a left border on each `ListItem` to draw the
   timeline spine.
5. **Attach images with `PhotoViewPicker`** - no permission needed.
6. **Scope the image previewer to the entry that was tapped** (`HW-10-0015`).
7. **Validate both entry forms** before popping (`HW-10-0020`, `HW-10-0023`).

## Verified snippets

All snippets are from `RecordTimeLine.zip`. Corrected forms are marked.

**Timeline structure — `pages/TimeLine.ets`** (as shipped)

```typescript
List() {
  ForEach(this.dateList, (item: IDateList) => {
    ListItemGroup({ header: this.timeLineHead(item.date) }) {
      ForEach(item.content, (childItem: IRecordList) => {
        ListItem() {
          Column({ space: 2 }) {
            if (childItem.textContent) {
              Text(childItem.textContent).padding(CommonConstants.TIMELINE_DATA_PADDING)
            }
            if (childItem.healthyInfo) {
              Row() {
                Text($r('app.string.timeline_height', childItem.healthyInfo.height)).width('50%')
                Text($r('app.string.timeline_weight', childItem.healthyInfo.weight))
              }.padding(CommonConstants.TIMELINE_DATA_PADDING)
            }
          }
        }
        .border({ width: { left: 1 } })   // the timeline spine
      })
    }
  })
}
```

The `.border({ width: { left: 1 } })` on every `ListItem` is what draws the
vertical line; the group header carries the node marker. That is the whole
timeline trick - no custom drawing.

> The height and weight `Text` above are shown with string resources. The
> shipped file hardcodes Chinese literals there - see `HW-10-0021`.

**Push and collect — `pages/TimeLine.ets`** (as shipped)

```typescript
this.pathStack.pushPathByName('growth', '', (res) => {
  const params = res.result as popRes
  const curTime = dateUtils.curTimeFormat()
  this.contentList.unshift({
    user: this.userName,
    healthyInfo: params.bodyData,
    date: dateUtils.dateFormat(params.date),
    time: curTime,
  })
  this.dates.push(params.date)
  this.dateList = dateUtils.getData(this.dates, this.contentList)
})
```

**Return a result — `pages/GrowthRecord.ets`** (as shipped)

```typescript
Button($r('app.string.save'))
  .width(CommonConstants.FULL_WIDTH)
  .type(ButtonType.Capsule)
  .onClick(() => {
    if (this.vHeight <= 0 || this.vHeight >= 200 || this.vWeight <= 0) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.toast_msg_3') })
      return
    }
    const result: popRes = {
      bodyData: new HealthInfo(this.vHeight, this.vWeight),
      date: this.selectedDate
    }
    this.pathStack.pop(result)
  })
```

Copy the shape - validate, toast, `return`, then `pop(result)` - but bind the
inputs to string state, not number (`HW-10-0016`), and bound the weight too
(`HW-10-0023`).

**Attaching images with no permission — `utils/ImgUtil.ets`** (as shipped)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';

export async function selectImages(maxSelectNumber: number) {
  const photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = maxSelectNumber;
  const photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  let photoSelectResult = await photoViewPicker.select(photoSelectOptions);
  return photoSelectResult.photoUris;
}
```

**This is the pattern to prefer.** `PhotoViewPicker` is a security component:
the system renders the gallery, the app receives read-only URIs, and
`module.json5` declares **no permission at all**. Only reach for
`getPhotoAccessHelper().getAssets()` plus `ohos.permission.READ_IMAGEVIDEO` when
you must render your own in-app album browser (`HW-10-0022`).

**Grouping by date — `utils/DateUtils.ets`** (corrected, see `HW-10-0018` and `HW-10-0019`)

```typescript
// FIX: the shipped version groups on a zh-CN display string and rescans the
// whole content list once per date. Group on a machine key, in one pass.
dateKey(date: Date): string {
  const m = `${date.getMonth() + 1}`.padStart(2, '0');
  const d = `${date.getDate()}`.padStart(2, '0');
  return `${date.getFullYear()}-${m}-${d}`;
}

getData(content: IRecordList[]): IDateList[] {
  const groups = new Map<string, IRecordList[]>();
  content.forEach(item => {
    const bucket = groups.get(item.dateKey);
    if (bucket) { bucket.push(item); } else { groups.set(item.dateKey, [item]); }
  });
  return Array.from(groups.keys())
    .sort((a, b) => b.localeCompare(a))          // newest first
    .map(key => ({ date: key, content: groups.get(key)! }));
}
```

Format for display only in the header builder, using the system locale rather
than a pinned `'zh-CN'`.

**Toolbar that reacts to scroll — `pages/TimeLine.ets`** (as shipped)

```typescript
.backgroundColor(this.isChange ? $r('app.color.toolbar_background') : Color.Transparent)
.position({ y: 0 })
.zIndex(1)
```

with `toolBarHeight = CommonConstants.TOOLBAR_HEIGHT + this.topRectHeight + 8`
computed in `aboutToAppear`, so the bar absorbs the status-bar inset.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` block,
and it does not need one - image selection goes through `PhotoViewPicker`.

Inset heights come from `AppStorage`:

```typescript
@StorageProp('topRectHeight') topRectHeight: number = 0
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- All state is in-memory. There is no persistence: every record is lost when the
  page is destroyed. Wire up your own storage - the document does not cover it.
- `entry/src/mock/mock-config.json5` is an empty object; the mock scaffolding is
  present but unused, and the timeline starts empty.
- The record date is chosen by the user with no upper bound, so future-dated
  records sort to the top of the timeline.

## Pitfalls

- **`HW-10-0015` — the image previewer is not scoped to the entry.** All diary
  images accumulate into one page-level `imageList`, so tapping a photo lets the
  user swipe through every other entry's photos. `indexOf` also returns the
  first match, so a photo attached to two entries always opens the first one.
  Pass `childItem.imageContent` and use the `ForEach` index.
- **`HW-10-0016` — `TextInput` `text` is `ResourceStr`, not `number`.** The
  shipped growth form binds `$$this.vHeight` where `vHeight: number`, so the
  state holds a string at runtime while its type says number and the range guard
  compares strings. `Diary.ets` binds a string correctly - follow that one.
- **`HW-10-0017` — the document promises deduplication it does not implement.**
  Step 2 says duplicate time points are collapsed and the latest data retained;
  the code groups all records under one header and keeps every one. Do not build
  persistence on the documented behaviour.
- **`HW-10-0018` — never group on a formatted date string.** The key is produced
  by `Intl.DateTimeFormat('zh-CN', ...)`. Headers stay Chinese regardless of the
  device language, and the day a developer localises the format, stored records
  stop matching their headers and vanish from the timeline.
- **`HW-10-0019` — `getData()` is quadratic and mutates state.** It rescans the
  full record list once per date, the `dates` array accumulates a duplicate
  entry per record forever, and `dates.sort()` reorders the caller's `@State`
  array as a hidden side effect.
- **`HW-10-0020` — the diary composer has no validation.** Tapping save on an
  untouched page creates a blank timeline row and a date group for it. The
  correct guard already exists on the growth form.
- **`HW-10-0021` — the timeline hardcodes Chinese labels and units** while the
  entry page uses `$r('app.string.*')` for the same two labels. The app cannot
  be localised without editing source.
- **`HW-10-0022` — two image-picking patterns across this industry.** This
  sample uses `PhotoViewPicker` with no permission; `MAT-01`'s sample enumerates
  the album with `getAssets()` and requires `ohos.permission.READ_IMAGEVIDEO`.
  Neither document explains the trade-off. Default to the picker.
- **`HW-10-0023` — weight is unbounded and there is a dead `onChange`.** Height
  is checked against `(0, 200)`, weight only against `> 0`, so `9999kg` renders
  into the timeline. The bare `200` should be a constant.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitemgroup.md` - `ListItemGroup`, the `header` builder
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitemgroup
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `pushPathByName`, `pop`, `PopInfo`; the `onPop` trigger condition is documented here
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInputOptions.text` is `ResourceStr`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` default key generation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
