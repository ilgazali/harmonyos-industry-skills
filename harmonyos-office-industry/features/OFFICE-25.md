---
id: OFFICE-25
title: Pinned group announcement - a ComponentContent node added to the window OverlayManager
industry: 05_office
doc: huawei_industry_tree/05_office/docs/25_pinned_notice.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pinned_notice-0000002355181168
sample: huawei_industry_tree/05_office/downloads/PinnedNotice.zip
kits: ["@kit.ArkUI"]
apis: [ComponentContent, "ComponentContent.dispose", "ComponentContent.isDisposed", "UIContext.getOverlayManager", OverlayManager, "OverlayManager.addComponentContent", "OverlayManager.removeComponentContent", "OverlayManager.showComponentContent", "OverlayManager.hideComponentContent", wrapBuilder, WrappedBuilder, "@Builder", "position()", "expandSafeArea", SafeAreaType, HitTestMode, Navigation, NavPathStack, "NavPathStack.pushPath", "@Provide", "@StorageProp", "@StorageLink", "UIContext.px2vp", "window.on('avoidAreaChange')", "window.off('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0137, HW-05-0138, HW-05-0139, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when something must **float above a whole page and stay there
across its internal scrolling and navigation** - a pinned group announcement, a
persistent banner, a floating action strip.

The mechanism is not an `overlay()` attribute on a component (that is OFFICE-04
and OFFICE-24). It is the **window-level `OverlayManager`**: a builder is packaged
into a `ComponentContent` and added to the overlay stack of the whole UI context,
so it is not a child of any page component and is unaffected by their layout.

That buys three things a component overlay cannot give:

- the node survives independently of the page tree it visually sits over;
- it can be **hidden and re-shown** by handle (`hideComponentContent` /
  `showComponentContent`) without being rebuilt;
- it is positioned in window coordinates, so it stays put while the content
  behind it scrolls.

The price is manual lifecycle: what you add you must remove **and dispose**.

## Feature checklist

The application must be able to:

- Package the announcement UI as a `ComponentContent` from a global `@Builder`.
- Pass live page state - the announcement text, the navigation stack, the overlay
  handle - into that builder as a typed argument object.
- Add the node to the window's overlay stack at a chosen index.
- Position the banner at a fixed offset and let it expand into the keyboard safe
  area.
- Let touches pass through the banner to the content beneath it.
- Hide the banner and navigate to the announcement detail page when it is tapped.
- Remove **and dispose** the node when the hosting page is destroyed.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes `topRectHeight` / `bottomRectHeight`, loads `pages/MainPage` |
| `pages/MainPage.ets` | `@Entry`; owns the `NavPathStack`, the `OverlayManager` handle, the `ComponentContent` array and the `Params`; adds the node in `aboutToAppear` and removes it in `aboutToDisappear` |
| `pages/NoticePage.ets` | the announcement detail destination |
| `view/OverlayNodeView.ets` | the global `@Builder OverlayNode(params: Params)` that renders the banner |
| `components/MessageDetail.ets` | the chat content behind the banner |
| `model/Params.ets` | the typed argument object passed into the builder |
| `common/Constants.ets` | sizes and spacings |

Note the directories are `common/` and `pages/` - the document's tree says
`constants/` and `page/` (HW-05-0138).

The `Params` class is the whole contract between the page and the detached
overlay node:

```ts
export class Params {
  text: string = '';
  offset: Position;
  pageInfos: NavPathStack;                    // so the node can navigate
  overlayContent: ComponentContent<Params>[]; // so the node can find itself
  overlayNode: OverlayManager;                // so the node can hide itself
}
```

Because the builder runs outside the page's component tree it has no `this`,
no `@Consume` and no access to page methods - everything it needs has to travel
through this object. Handing it the `OverlayManager` and the array of contents is
what lets the banner hide **itself** on tap.

Lifecycle:

```
MainPage.aboutToAppear
  uiContext.getOverlayManager()                       -> overlayNode
  new ComponentContent(uiContext,
                       wrapBuilder<[Params]>(OverlayNode),
                       params)                        -> componentContent
  overlayNode.addComponentContent(componentContent, 0) // 0 = bottom of the overlay stack
  overlayContent.push(componentContent)                // keep the handle

banner onClick
  overlayNode.hideComponentContent(overlayContent[0])  // hide, do not remove
  pageInfos.pushPath({ name: 'NoticePage' })

MainPage.aboutToDisappear
  overlayContent.pop() -> removeComponentContent(...)  // + dispose() - HW-05-0137
```

`hide` rather than `remove` on tap is deliberate: the node keeps its identity, so
re-showing it later is one call rather than a rebuild. In this sample nothing
re-shows it, which matches the stated behaviour - "若点击查看公告详情并返回，则公告消失"
("if you tap to view the announcement detail and come back, the announcement
disappears").

## Implementation steps

1. **Declare no permission.** The feature is UI composition only; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Write the banner as a global `@Builder`** taking a single typed parameter -
   it must be a top-level function, not a method, so `wrapBuilder` can package
   it.
3. **Define the argument object.** Put everything the detached node needs into
   one class: the display data plus the handles it needs to act (`NavPathStack`,
   `OverlayManager`, the content array).
4. **Get the overlay manager from the UI context.**
   `this.getUIContext().getOverlayManager()`.
5. **Build and add in `aboutToAppear`.**
   `new ComponentContent(uiContext, wrapBuilder<[Params]>(OverlayNode), params)`
   then `addComponentContent(content, 0)`; keep the handle in an array so the
   node can reach it.
6. **Position in window space.** `.position({ x, y })` from the params, plus
   `.expandSafeArea([SafeAreaType.KEYBOARD])` so the banner is not pushed by the
   soft keyboard.
7. **Do not block interaction.** `.hitTestBehavior(HitTestMode.Transparent)` -
   the sample comments this explicitly.
8. **Hide, then navigate,** on tap: `hideComponentContent(content)` followed by
   `pageInfos.pushPath(...)`.
9. **Remove and dispose in `aboutToDisappear`.** `removeComponentContent` detaches
   the node; `dispose()` releases its reference to the backend entity node, which
   the reference requires to avoid a leak (HW-05-0137).
10. **Release the window listener** in `onWindowStageDestroy` (HW-05-0139).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The argument object

`PinnedNotice.zip#entry/src/main/ets/model/Params.ets`

```ts
import { OverlayManager } from '@kit.ArkUI';

export class Params {
  text: string = '';
  offset: Position;
  pageInfos: NavPathStack;
  overlayContent: ComponentContent<Params>[];
  overlayNode: OverlayManager;

  constructor(text: string, pageInfos: NavPathStack, overlayContent: ComponentContent<Params>[],
    overlayNode: OverlayManager, offset: Position) {
    this.text = text;
    this.offset = offset;
    this.pageInfos = pageInfos;
    this.overlayContent = overlayContent;
    this.overlayNode = overlayNode;
  }
}
```

### Creating and attaching the overlay node

`PinnedNotice.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
import { ComponentContent, OverlayManager } from '@kit.ArkUI';
import { OverlayNode } from '../view/OverlayNodeView';

@Entry
@Component
struct MainPage {
  @State text: string = '下周一下午15:00开例会，主要过一下本周的项目进度和接下来的项目安排。';
  @StorageLink('componentOffset') componentOffset: Position = { x: 0, y: 92 };
  pageInfos: NavPathStack = new NavPathStack();
  private uiContext: UIContext = this.getUIContext();
  private overlayNode: OverlayManager = this.uiContext.getOverlayManager();
  private overlayContent: ComponentContent<Params>[] = [];
  @Provide params: Params =
    new Params(this.text, this.pageInfos, this.overlayContent, this.overlayNode, this.componentOffset);

  aboutToAppear(): void {
    let componentContent = new ComponentContent(
      this.uiContext, wrapBuilder<[Params]>(OverlayNode), this.params
    );
    this.overlayNode.addComponentContent(componentContent, 0);
    this.overlayContent.push(componentContent);
  }

  aboutToDisappear(): void {
    let componentContent = this.overlayContent.pop();
    this.overlayNode.removeComponentContent(componentContent);   // no dispose() - HW-05-0137
  }

  build() {
    Navigation(this.pageInfos) {
      Column() {
        MessageDetail();
      }
      .width(Constants.FULL_PERCENT)
      .height(Constants.FULL_PERCENT);
    }
    .hideTitleBar(true)
    .hideToolBar(true)
    .padding({ top: this.uiContext.px2vp(this.topRectHeight) });
  }
}
```

`wrapBuilder<[Params]>(OverlayNode)` is the piece to get right: the type argument
is the **tuple of the builder's parameters**, and it must match the third
`ComponentContent` constructor argument. `addComponentContent(content, 0)` places
the node at index 0 of the overlay stack.

Corrected teardown:

```ts
aboutToDisappear(): void {
  const componentContent = this.overlayContent.pop();
  if (componentContent) {
    this.overlayNode.removeComponentContent(componentContent);
    componentContent.dispose();
  }
}
```

### The detached banner

`PinnedNotice.zip#entry/src/main/ets/view/OverlayNodeView.ets`

```ts
@Builder
export function OverlayNode(params: Params) {
  Row({ space: Constants.SPACE_TEN }) {
    Image($r('app.media.volume'))
      .width(Constants.IMAGE_MORE_WIDTH)
      .height(Constants.IMAGE_MORE_HEIGHT);
    Text(params.text)
      .fontWeight(Constants.FONT_WEIGHT_400)
      .lineHeight(Constants.LINE_HEIGHT_16)
      .fontSize(Constants.FONT_SIZE_14)
      .maxLines(Constants.ONE)
      .layoutWeight(Constants.ONE);
    Image($r('app.media.image_right'))
      .width(Constants.IMAGE_RIGHT_WIDTH)
      .height(Constants.IMAGE_RIGHT_HEIGHT);
  }
  .onClick(() => {
    let componentContent = params.overlayContent[0];
    params.overlayNode.hideComponentContent(componentContent);
    params.pageInfos.pushPath({ name: 'NoticePage' });
  })
  .padding({
    left: Constants.PADDING_LEFT_16,
    right: Constants.PADDING_RIGHT_16
  })
  .position({
    x: params.offset.x,
    y: params.offset.y
  })
  .expandSafeArea([SafeAreaType.KEYBOARD])
  .width(Constants.FULL_PERCENT)
  .height(Constants.BOTTOM_ROW_HEIGHT)
  .backgroundColor($r('app.color.start_window_background'))
  .hitTestBehavior(HitTestMode.Transparent); // 配置浮层不阻塞交互
}
```

Three attributes make the overlay behave: `position()` places it in window
coordinates, `expandSafeArea([SafeAreaType.KEYBOARD])` keeps the soft keyboard
from displacing it, and `hitTestBehavior(HitTestMode.Transparent)` lets taps on
the surrounding area reach the page underneath - the sample comments the last one
explicitly.

Reaching back into the page through `params` is what lets a node with no
component context of its own hide itself and navigate.

## Permissions & config

`PinnedNotice.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the whole feature is ArkUI
node composition. The document has no 权限说明 section, which matches.

`build-profile.json5` pins the SDK to `6.0.0(20)` and enables
`caseSensitiveCheck: true`, which is why the `common/` vs `constants/` and
`pages/` vs `page/` discrepancies in the document's tree matter (HW-05-0138).

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The builder must be a global function.** `wrapBuilder` packages a top-level
  `@Builder`; a method on the struct cannot be wrapped, which is why
  `OverlayNode` lives in its own file.
- **The node has no component context.** No `this`, no `@Consume`, no access to
  page state except through the argument object - which is why `Params` carries
  the `NavPathStack` and the `OverlayManager`.
- **The argument object is captured at construction.** `new ComponentContent(...,
  params)` binds that instance; later changes to page state do not reach the node
  unless it is explicitly updated.
- **`dispose()` is mandatory, not optional.** The reference states that without it
  "memory leaks may occur" because the frontend object keeps its reference to the
  backend entity node.
- **Hide is not remove.** `hideComponentContent` keeps the node in the overlay
  stack so `showComponentContent` can bring it back; only
  `removeComponentContent` detaches it.
- **The overlay is window-scoped.** It floats above every page in that UI context,
  so a node added by one page and not removed remains over the next one.
- **The banner offset is hard-coded.** `@StorageLink('componentOffset')` defaults
  to `{ x: 0, y: 92 }`; it does not derive from the title-bar height or the
  status-bar inset.

## Pitfalls

- **Removing the `ComponentContent` without disposing it is incorrect.** The
  reference is explicit: "be sure to call dispose() on the ComponentContent object
  when you no longer need it ... this lowers the risk of memory leaks". The
  undisposed node also keeps the `Params` object - and through it the
  `NavPathStack` and `OverlayManager` - reachable after the page is gone.
  (HW-05-0137)
- **The project tree's `constants/` and `page/` are incorrect** - the sample ships
  `common/` and `pages/`, which is also what `MainPage.ets` and
  `OverlayNodeView.ets` import, and the build enables `caseSensitiveCheck`.
  (HW-05-0138)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect** -
  release it in `onWindowStageDestroy`. (HW-05-0139)
- Not a separate finding, but worth guarding when adapting: `aboutToDisappear`
  passes `this.overlayContent.pop()` straight into `removeComponentContent`, and
  `pop()` returns `undefined` on an empty array. The official `OverlayManager`
  example uses the same shape, so it is idiomatic - but a `if (componentContent)`
  guard costs nothing and is required anyway once `dispose()` is added.

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` -
  the three `ComponentContent` constructors, and `dispose()` with its
  "memory leaks may occur" warning plus `isDisposed()`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-overlaymanager.md` -
  `addComponentContent12+`, `removeComponentContent12+`,
  `showComponentContent12+` and `hideComponentContent12+`, with the reference
  example this sample follows.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-overlaymanager
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `getOverlayManager()` and `px2vp`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-overlay.md` -
  the component-level `overlay()` attribute, contrasted with the window-level
  overlay used here.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-overlay
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on`/`off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
