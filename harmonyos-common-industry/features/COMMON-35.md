---
id: COMMON-35
title: Category title to content linkage - tap a title and scroll the content list to that section
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/35_title_and_content_linkage.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/title_and_content_linkage-0000002365395897
sample: huawei_industry_tree/19_common_technical_solutions/downloads/ScrollDemo.zip
kits: ["@kit.ArkUI"]
apis: [List, ListItem, Scroller, "Scroller.scrollToIndex", ScrollAlign, "List.listDirection", "List.lanes", "List.chainAnimation", "List.edgeEffect", "List.initialIndex", "List.scroller", Image, "Image.sourceSize", "Image.objectFit", "UIContext.getMeasureUtils", "MeasureUtils.measureText", "UIContext.px2vp", "component .safeAreaPadding", "@StorageProp", ForEach]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0106, HW-19-0107, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a screen has **a horizontal strip of category titles above a
long vertical content list**, and tapping a title must jump the content to that
section - a gallery grouped by album, a menu grouped by course, a store page
grouped by department.

The mechanism is two `List`s, two `Scroller`s, and one shared index. It is
deliberately simpler than the tab-bar sample of COMMON-30: there is no
continuous underline animation, no gesture tracking, and the linkage is one-way -
title drives content, not the reverse.

## Feature checklist

The application must:

- Render the titles in a horizontally scrolling `List` and the content in a
  vertical `List`, each bound to its own `Scroller`.
- Track the selected title index in state.
- On tap: scroll the **title** strip so the tapped title and its neighbours are
  visible, scroll the **content** list to the matching section, and update the
  selected index.
- Align the scrolled-to content section deliberately - the sample centres it with
  `ScrollAlign.CENTER`.
- Mark the selected title with an underline sized to the measured text width.
- Size image decoding from the rendered size, not from an arbitrary constant
  (HW-19-0106).

## Architecture

Single-module project (`entry` HAP), two source files plus the ability:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | full-screen layout; publishes `topSafeHeight` / `bottomSafeHeight` from `TYPE_SYSTEM` and `TYPE_NAVIGATION_INDICATOR`, and keeps them current through `avoidAreaChange` |
| `ShowData.ets` | `ShowData` (a group: name plus details), `ShowDataDetail` (image plus caption), and the `showDatas` fixture |
| `pages/Index.ets` | everything else: both lists, both scrollers, the title measurement and the selection state |

**Two scrollers, one index.**

| Field | Role |
| --- | --- |
| `scrollerForTitle` | bound to the horizontal title `List` |
| `scrollerForList` | bound to the vertical content `List` |
| `currIndex` | the selected title; drives the title colour and the underline |
| `titleLength: Length[]` | per-title measured text width, used as the underline width |

**The tap handler does three things**, and the first is the non-obvious one:

```ts
if (index > 3) {
  this.scrollerForTitle.scrollToIndex(index, true);
} else {
  this.scrollerForTitle.scrollToIndex(0, true);
}
this.scrollerForList.scrollToIndex(index, true, ScrollAlign.CENTER);
this.currIndex = index;
```

The title strip scrolls **itself** so the tapped title is not stranded at the
edge of the viewport. The `> 3` threshold is a "first half / second half" rule
for the visible window; both the document and the code comment mark it as a value
to tune - 该数值根据实际设置 ("set this value according to the actual situation").

**The underline is measured, not guessed.** `aboutToAppear` runs each group name
through `MeasureUtils.measureText` at the same font size the `Text` uses, converts
the px result to vp, and stores it:

```ts
this.titleLength[index] = this.getUIContext().px2vp(Number(this.getUIContext().getMeasureUtils().measureText({
  textContent: item.groupName,
  fontSize: 14
}))) + 2;
```

so the indicator is exactly as wide as its label plus 2 vp, for titles of any
length.

**Content layout.** Each group renders its name and then a nested `List` with
`.lanes(3)` - a three-column grid of thumbnails - and the outer list gets
`.chainAnimation(true)` and `.edgeEffect(EdgeEffect.Spring)` for the scroll feel.

## Implementation steps

1. **Model the data as groups of details** so one index addresses both lists.
2. **Create two `Scroller`s** and bind one to each `List`; a `Scroller` cannot be
   shared between scrollable components.
3. **Measure the titles once** in `aboutToAppear` with `getMeasureUtils()
   .measureText({ textContent, fontSize })`, using the same font size as the
   rendered `Text`, and convert with `px2vp`.
4. **Build the title strip**: `List({ space, scroller: scrollerForTitle })` with
   `.listDirection(Axis.Horizontal)` and `.scrollBar(BarState.Off)`.
5. **Build the title item** as a `Column` of the `Text` plus a conditional
   underline `Column` whose width is the measured length.
6. **Wire the tap**: scroll the strip, scroll the content with an explicit
   `ScrollAlign`, then set the index.
7. **Build the content list**: `List({ initialIndex: 0, scroller: scrollerForList })`,
   one `ListItem` per group, each containing the group heading and a nested
   `List().lanes(3)` of thumbnails.
8. **Size the thumbnails' decode resolution from their rendered size**
   (HW-19-0106).
9. **Pad for the system insets** with the published safe heights.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Measuring the title widths

`ScrollDemo.zip#ScrollDemo/entry/src/main/ets/pages/Index.ets`

```ts
@State currIndex: number = 0;
@State titleLength: Length[] = [];
private data: ShowData[] = showDatas;
private scrollerForTitle: Scroller = new Scroller();
private scrollerForList: Scroller = new Scroller();

aboutToAppear(): void {
  this.data.forEach((item, index) => {
    this.titleLength[index] = this.getUIContext().px2vp(Number(this.getUIContext().getMeasureUtils().measureText({
      textContent: item.groupName,
      fontSize: 14
    }))) + 2;
  });
}
```

### The title item and the linkage

`ScrollDemo.zip#ScrollDemo/entry/src/main/ets/pages/Index.ets`

```ts
@Builder
groupNameBar(name: string, index: number) {
  Column() {
    Text(name)
      .fontSize(14)
      .fontColor(this.currIndex === index ? '#0A59F7' : '#cd202021')
      .onClick(() => {
        if (index > 3) {
          // 标题根据点击List的index进行滚动，当index>3（该数值根据实际设置），即点击位置处于列表后半部分，列表滚动到index位置。
          this.scrollerForTitle.scrollToIndex(index, true);
        } else {
          // 标题根据点击List的index进行滚动，当index<=（该数值根据实际设置），即点击位置处于列表前半部分，列表滚动到首位，也就是根据点击，展示附近位置的标题。
          this.scrollerForTitle.scrollToIndex(0, true);
        }
        // 根据当前标题index，使用scrollToIndex滚动到下方List组件的指定Index位置，设置对齐方式为ScrollAlign.CENTER。
        this.scrollerForList.scrollToIndex(index, true, ScrollAlign.CENTER);
        this.currIndex = index;
      })
      .padding({ bottom: 5 });
    // 自定义选中文本底部导航条
    if (index === this.currIndex) {
      Column().width(this.titleLength[this.currIndex]).height(1).backgroundColor('#0A59F7');
    }
  }.height(21);
}
```

### The two lists

`ScrollDemo.zip#ScrollDemo/entry/src/main/ets/pages/Index.ets`

```ts
Row() {
  List({ space: 30, scroller: this.scrollerForTitle }) {
    ForEach(this.data, (item: ShowData, index: number) => {
      ListItem() {
        this.groupNameBar(item.groupName, index);
      };
    }, (item: number) => JSON.stringify(item));      // FIX (HW-19-0107)
  }
  .listDirection(Axis.Horizontal)
  .padding({ left: 20, right: 20 })
  .scrollBar(BarState.Off);
}
.margin({ top: 20 })
.width('100%')
.height(22);

// ... a Line separator ...

Column() {
  List({ initialIndex: 0, scroller: this.scrollerForList }) {
    ForEach(this.data, (item: ShowData) => {
      ListItem() {
        this.showDataImage(item);
      };
    }, (item: number) => JSON.stringify(item));      // FIX (HW-19-0107)
  }
  .chainAnimation(true)
  .edgeEffect(EdgeEffect.Spring);
}.layoutWeight(1);
```

### One content group (as shipped - see HW-19-0106)

`ScrollDemo.zip#ScrollDemo/entry/src/main/ets/pages/Index.ets`

```ts
@Builder
showDataImage(item: ShowData) {
  Column() {
    Row() {
      Text(item.groupName)
        .fontWeight(FontWeight.Bold)
        .fontSize(18)
        .lineHeight(24);
    }
    .justifyContent(FlexAlign.Start)
    .margin({ bottom: 10 })
    .width('100%');

    List({ space: 5 }) {
      ForEach(item.details, (itemDetail: ShowDataDetail) => {
        ListItem() {
          Column() {
            Image(itemDetail.image)
              .margin({ right: 5 })
              .height(100)
              .borderRadius(5)
              .objectFit(ImageFit.Fill)
              .width('95%')
              .sourceSize({ width: 30, height: 30 });   // FIX: decode at the rendered size

            Text(itemDetail.desc)
              .maxLines(2)
              .fontSize(14)
              .margin({ top: 5 })
              .fontColor('#cd202021')
              .textOverflow({ overflow: TextOverflow.Ellipsis });
          }
          .alignItems(HorizontalAlign.Start);
        };
      });
    }
    .lanes(3);
  }
  .margin({ top: 20, left: 20, right: 20 });
}
```

### Safe-area publication (done correctly here)

`ScrollDemo.zip#ScrollDemo/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let windowClass: window.Window = windowStage.getMainWindowSync(); // 获取应用主窗口
windowClass.setWindowLayoutFullScreen(true);

let topSafeHeight =
  windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height;
let bottomSafeHeight =
  windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height;
AppStorage.setOrCreate('bottomSafeHeight', bottomSafeHeight);
AppStorage.setOrCreate('topSafeHeight', topSafeHeight);

// 3. 注册监听函数，动态获取避让区域数据
windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topSafeHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomSafeHeight', data.area.bottomRect.height);
  }
});
```

Consumed in the page as `padding({ top: px2vp(this.topSafeHeight), bottom:
px2vp(this.bottomSafeHeight) })` - the raw px stays in AppStorage and the
conversion happens at the point of use, which is the pattern COMMON-12 gets wrong
(HW-19-0024).

## Permissions & config

**No permissions are required** and none are declared - the feature is layout and
scrolling over a packaged data fixture.

`ScrollDemo.zip#ScrollDemo/entry/src/main/module.json5` declares the usual
`EntryAbility` with the home skill and an `EntryBackupAbility`; there is no
`requestPermissions` block and no `routerMap`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **A `Scroller` cannot be shared** between scrollable components, which is why
  there are two.
- **`scrollToIndex` ignores out-of-range values silently** - a negative index or
  one past the last item performs no scrolling at all.
- **`ScrollAlign` is worth choosing deliberately.** `CENTER` puts the target
  section in the middle of the viewport; the `List` default is `START`.
- **`sourceSize` is the decode resolution.** "This attribute works only when the
  target size is smaller than the source size", and it does not apply to SVG
  images or `PixelMap` objects.
- **`measureText` returns px**, hence the `px2vp` in `aboutToAppear`; the font
  size passed to it must match the rendered `Text`, or the underline will be the
  wrong width.
- **Title widths are measured once.** A dynamic data set, a font-size change or a
  system font-scale change would need re-measurement.
- **The linkage is one-way.** Scrolling the content does not move the title strip
  or update `currIndex`; add an `onScrollIndex` handler on the content list if
  the reverse direction is wanted.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`sourceSize({ width: 30, height: 30 })` on a 100 vp thumbnail is incorrect.**
  The image is decoded at 30 pixels and then stretched with `ImageFit.Fill`, so
  every gallery thumbnail is visibly blurred. Derive the decode size from the
  rendered size. (HW-19-0106)
- **Both key generators are typed `(item: number)` while the array holds
  `ShowData`, which is incorrect** - and neither declares the index, so the
  framework appends it, the fallback the ForEach guide warns about. Type the
  parameter and key on a stable field. (HW-19-0107)
- **The `index > 3` threshold is a magic number**, acknowledged as tunable by both
  the document and the code comment. It encodes an assumption about how many
  titles fit on screen; on a tablet or in landscape it is wrong in a way nothing
  detects.
- **`titleLength[this.currIndex]` is read, not `titleLength[index]`.** Inside the
  `if (index === this.currIndex)` branch the two are equal, so it works - but the
  indirection makes the builder look dependent on state it does not need.
- **`Line().startPoint([20, 0]).endPoint([500, 0])`** hardcodes a 500 vp
  separator, which is narrower than the screen on a tablet and wider than it on a
  small phone.
- **`this.data = showDatas` aliases the module-level fixture.** Nothing mutates it
  here, but a page that did would edit the shared constant.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` -
  `List`, `scroller`, `initialIndex`, `space`, `listDirection`, `lanes`,
  `chainAnimation`, `edgeEffect`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` -
  `Scroller`, `scrollToIndex` and its parameters, `ScrollAlign`, and the
  one-scroller-per-component rule.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` -
  `sourceSize` ("Sets the decoding size of the image. This attribute works only
  when the target size is smaller than the source size."), `objectFit`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the `keyGenerator` signature and the automatic index-appending fallback.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-UIContext.md` -
  `getMeasureUtils().measureText` and `px2vp`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-uicontext
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` / `TYPE_NAVIGATION_INDICATOR`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/title_and_content_linkage-0000002365395897
