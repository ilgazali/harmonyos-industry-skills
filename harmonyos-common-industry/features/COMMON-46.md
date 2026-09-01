---
id: COMMON-46
title: Scroll-to-top for H5 pages - a draggable ArkUI button layered over a Web component
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/46_web_scroll_to_top.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
sample: huawei_industry_tree/19_common_technical_solutions/downloads/WebScrollToTop.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI"]
apis: [Web, "webview.WebviewController", "WebviewController.scrollTo", "Web.onScroll", "Web.fileAccess", "Web.geolocationAccess", "Web.expandSafeArea", Stack, PanGesture, animateToImmediately, "curves.springMotion", ".position", ".visibility", ".onAreaChange", "@Link", "@Prop", "@State", "$rawfile"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0149, HW-19-0150, HW-19-0151, HW-19-0152, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a `Web` component shows long pages and the user needs a
**scroll-to-top affordance that works on any page** - including pages you do not
control and cannot inject script into.

The key idea is that the button is **not part of the H5 content**. It is an ArkUI
component stacked over the `Web`, driven by the native `onScroll` event and
acting through `WebviewController.scrollTo`. No JavaScript injection, no
cooperation from the page, no per-site adaptation. The same pattern applies to
any overlay control a hybrid app needs over web content - share, refresh, reader
mode.

## Feature checklist

- `Stack` with the `Web` first and the overlay second.
- `Web.onScroll` → show the button past a threshold.
- `WebviewController.scrollTo(0, 0, duration)`, guarded (HW-19-0149).
- Optional: `PanGesture` to let the user move the button, with a spring-animated
  edge snap at drag end.
- Initial button position derived from the measured container, not from
  constants (HW-19-0150).

## Architecture

Single-module project (`entry` HAP), two components:

| File | Responsibility |
| --- | --- |
| `pages/Index.ets` | the `Stack`, the `Web`, the scroll threshold, container measurement |
| `views/ScrollToTopButton.ets` | the floating button, the drag gesture, the snap animation |
| `resources/rawfile/index.html` | the demo page (316 lines, remote images) |
| `resources/rawfile/icon/search.svg` | a local asset the page references relatively |

**The stacking order is the whole layout.** `Stack({ alignContent:
Alignment.BottomEnd })` renders children back-to-front, so the `Web` declared
first sits underneath and the button declared second floats above it. The
`alignContent` is then overridden by the button's own `.position()`, so it has no
visible effect - a leftover of the intended design.

**Three pieces of state flow parent → child, with one flowing back.**

| Variable | Direction | Why |
| --- | --- | --- |
| `isShow` | `@Link`, two-way | the parent shows it on scroll; the button hides itself on tap |
| `containerWidth` / `containerHeight` | `@Prop`, one-way | measured by the parent, used by the child's clamp |
| `webviewController` | constructor argument | the child needs to act on the web view |

`isShow` is the one case in this industry where `@Link` is genuinely warranted:
the child writes it in `onClick` and the parent must see that write.

**The threshold is a plain field, not state.** `showButtonLimit: number = 300`
never changes, so it is correctly left undecorated.

**Two `onAreaChange` handlers, both guarded.** The parent measures the `Stack`,
the child measures itself, and both compare old and new dimensions before writing
- `onAreaChange` fires on every layout pass, so the guard prevents a write-render
loop.

**The drag is three gesture callbacks and one animation.** `onActionStart`
snapshots the current offsets, `onActionUpdate` applies the delta live,
`onActionEnd` clamps inside `animateToImmediately` with `curves.springMotion()` -
so the snap-to-edge is a spring rather than a jump, and it is delivered
immediately rather than waiting for the next VSync.

**The demo page needs the network.** More than twenty `<img>` elements load from
`consumer.huawei.com` over https. That is what gives the page enough height to
scroll past 300 vp and make the button appear - and it is why `INTERNET` is
declared, which the document never mentions (HW-19-0151).

## Implementation steps

1. **Stack the overlay over the Web.**
   ```ts
   Stack({ alignContent: Alignment.BottomEnd }) {
     Web({ src: ..., controller: this.webviewController })
     ScrollToTopButton({ webviewController: ..., isShow: this.isScrollToTopShow, ... })
   }
   ```
2. **Measure the container** in the `Stack`'s `onAreaChange`, guarding against
   unchanged dimensions, and pass the size down.
3. **Watch the scroll**: `.onScroll((event) => { this.isScrollToTopShow =
   event.yOffset > this.showButtonLimit; })`.
4. **Show and hide** with `.visibility(this.isShow ? Visibility.Visible :
   Visibility.Hidden)` - which keeps the component in the layout, so its measured
   size stays valid while hidden.
5. **Act on tap**: `scrollTo(0, 0, duration)` inside `try/catch`, then hide
   (HW-19-0149).
6. **Position from the measurement** rather than from constants (HW-19-0150),
   then let `PanGesture` move it and clamp at `onActionEnd`.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The stacked layout and the scroll threshold

`WebScrollToTop.zip#WebScrollToTop/entry/src/main/ets/pages/Index.ets`

```ts
import { webview } from '@kit.ArkWeb';
import { ScrollToTopButton } from '../views/ScrollToTopButton';

@Entry
@Component
struct Index {
  private webviewController: WebviewController = new webview.WebviewController();
  private showButtonLimit: number = 300;
  @State isScrollToTopShow: boolean = false;
  @State containerWidth: number = 0;
  @State containerHeight: number = 0;

  build() {
    Column() {
      Stack({ alignContent: Alignment.BottomEnd }) {
        Web({
          src: $rawfile('index.html'),
          controller: this.webviewController,
        })
          .fileAccess(false)
          .geolocationAccess(false)
          .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
          .onScroll((event) => {
            if (event.yOffset > this.showButtonLimit) {
              this.isScrollToTopShow = true;
            } else {
              this.isScrollToTopShow = false;
            }
          });
        ScrollToTopButton({
          webviewController: this.webviewController,
          isShow: this.isScrollToTopShow,
          containerWidth: this.containerWidth,
          containerHeight: this.containerHeight
        });
      }
      .onAreaChange((oldValue: Area, newValue: Area) => {
        if (oldValue.width !== newValue.width || oldValue.height !== newValue.height) {
          this.containerWidth = newValue.width as number;
          this.containerHeight = newValue.height as number;
        }
      });
    }
    .height('100%')
    .width('100%');
  }
}
```

Note `.fileAccess(false)` and `.geolocationAccess(false)` - both capabilities
explicitly denied because the demo page needs neither. That is the right default
for a web view showing content you did not write.

### The floating button

`WebScrollToTop.zip#WebScrollToTop/entry/src/main/ets/views/ScrollToTopButton.ets`

```ts
import { curves } from '@kit.ArkUI';

@Component
export struct ScrollToTopButton {
  @State webviewController: WebviewController | null = null;
  @Link isShow: boolean;
  @Prop containerWidth: number;
  @Prop containerHeight: number;
  @State private offsetX: number = 296;      // FIX (HW-19-0150): derive from containerWidth
  @State private offsetY: number = 623;      // FIX (HW-19-0150): derive from containerHeight
  @State private buttonX: number = 0;
  @State private buttonY: number = 0;
  private buttonWidth: number = 48;
  private buttonHeight: number = 48;
  private buttonPadding: number = 16;

  build() {
    Row() {
      Image($r('app.media.top_button_icon'))
        .width($r('app.float.top_icon_size'))
        .height($r('app.float.top_icon_size'));
    }
    .justifyContent(FlexAlign.Center)
    .alignItems(VerticalAlign.Center)
    .width($r('app.float.top_button_size'))
    .height($r('app.float.top_button_size'))
    .borderRadius($r('app.float.top_button_radius'))
    .backgroundColor(Color.White)
    .visibility(this.isShow ? Visibility.Visible : Visibility.Hidden)
    .onClick(() => {
      this.webviewController?.scrollTo(0, 0);   // FIX (HW-19-0149): try/catch + duration
      this.isShow = false;
    })
    .position({ x: this.offsetX, y: this.offsetY })
    .onAreaChange((oldValue: Area, newValue: Area) => {
      if (oldValue.width !== newValue.width || oldValue.height !== newValue.height) {
        this.buttonWidth = newValue.width as number;
        this.buttonHeight = newValue.height as number;
      }
    })
```

The corrected click handler:

```ts
.onClick(() => {
  try {
    this.webviewController?.scrollTo(0, 0, 300);
    this.isShow = false;
  } catch (error) {
    hilog.error(0x0000, 'ScrollToTop', `scrollTo failed: ${(error as BusinessError).code}`);
  }
})
```

### The drag and the spring snap

`WebScrollToTop.zip#WebScrollToTop/entry/src/main/ets/views/ScrollToTopButton.ets`

```ts
.gesture(
  PanGesture()
    .onActionStart(() => {
      this.buttonX = this.offsetX;
      this.buttonY = this.offsetY;
    })
    .onActionUpdate((event: GestureEvent) => {
      this.offsetX = this.buttonX + event.offsetX;
      this.offsetY = this.buttonY + event.offsetY;
    })
    .onActionEnd(() => {
      animateToImmediately({ curve: curves.springMotion() }, () => {
        // 限制按键在x方向上的最大、最小值，防止按键超出可视窗口
        if (this.offsetX >= 0) {
          let x = this.offsetX + this.buttonWidth / 2;
          this.offsetX =
            x >= this.containerWidth / 2 ? this.containerWidth - this.buttonWidth - this.buttonPadding :
              this.buttonPadding;
        } else {
          this.offsetX = this.buttonPadding;
        }
        // 限制按键在y方向上的最大、最小值，防止按键超出可视窗口
        if (this.offsetY >= 0) {
          this.offsetY =
            this.offsetY + this.buttonHeight > this.containerHeight ? this.containerHeight - this.buttonHeight :
              this.offsetY;
        } else {
          this.offsetY = 0;
        }
      });
    })
);
```

Read the horizontal clamp carefully - it is the interesting part. It takes the
button's **centre** (`offsetX + buttonWidth / 2`), compares it with the
container's midpoint, and snaps to whichever edge that centre is nearer. That is
the standard floating-action-button behaviour, expressed in four lines. The
vertical clamp only bounds the value; it does not snap.

`onActionStart` snapshotting into `buttonX`/`buttonY` is what makes the drag
cumulative: `GestureEvent.offsetX` is measured from the gesture's start, not from
the last frame, so it must be added to the position the button held when the
gesture began.

### The demo page

`WebScrollToTop.zip#WebScrollToTop/entry/src/main/resources/rawfile/index.html`

```html
<img src="./icon/search.svg" alt="search">
<img src="https://consumer.huawei.com/content/dam/huawei-cbg-site/cn/mkt/launch/241126/plp/phones/new/huawei-phones-camera.jpg" ...>
```

Both reference styles work from `$rawfile`: the relative `./icon/search.svg`
resolves inside the rawfile directory (the file is shipped at
`resources/rawfile/icon/search.svg`), and the https images need
`ohos.permission.INTERNET`.

## Permissions & config

`WebScrollToTop.zip#WebScrollToTop/entry/src/main/module.json5`:

```json5
"deviceTypes": ["phone"],
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- **`ohos.permission.INTERNET`** - normal, granted at install. Required here, and
  genuinely used: the demo page loads its images from `consumer.huawei.com`.
  Declared without `reason`/`usedScene`, which is permitted for a normal
  permission. The document does not mention it (HW-19-0151).

`build-profile.json5` pins `6.0.0(20)`. Button dimensions come from
`resources/base/element/float.json`: `top_button_size` 48, `top_icon_size` 24,
`top_button_radius` 80.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `scrollTo`'s `duration` parameter
  is API 14+; `Web.onScroll` is API 9+.
- **`scrollTo` throws 17100001** when the `WebviewController` is not associated
  with a `Web` component, and 401 on a bad parameter.
- **`scrollTo` coordinates are in vp** and negative values are treated as 0.
- **Omitting `duration` disables the animation** - the scroll becomes an instant
  jump.
- **`.position()` overrides the parent's `alignContent`.** Mixing the two is
  contradictory; pick one.
- **`Visibility.Hidden` keeps the component in the layout** (unlike `None`), which
  is what keeps `onAreaChange` measurements valid while the button is hidden.
- **`onAreaChange` fires on every layout pass** - always compare before writing
  state from it.
- **`GestureEvent.offsetX/offsetY` are cumulative from the gesture start**, not
  per-frame deltas.
- **Devices.** `phone` only, per `module.json5`.

## Pitfalls

- **`scrollTo` is called with no error handling and no duration, which is
  incorrect** - it is documented to throw 17100001 and 401, and without a duration
  the page jumps instead of scrolling. (HW-19-0149)
- **The button's initial position is hard-coded to (296, 623), which is
  incorrect** - every other position it takes is computed from the measured
  container, and in a window narrower than 344 vp the button renders outside it
  where no pan gesture can reach it. (HW-19-0150)
- **The document has no 权限说明 section although the sample declares INTERNET,
  which is incomplete** - and without the permission the demo page is too short to
  scroll, so the feature never appears. (HW-19-0151)
- **The 工程目录 tree writes `entrybackupablility` and gives `pages` and `views`
  the same last-child marker, which is incorrect.** (HW-19-0152)
- **Do not put the overlay inside the H5 page.** The entire value of this pattern
  is that it works on pages you do not control; a JavaScript-injected button has
  to be re-tested against every site.
- **Do not use `Visibility.None` for the hidden state** unless you also stop
  relying on the component's measured size - `None` removes it from the layout.
- **`@State webviewController` is unnecessary.** The controller is passed in once
  and never reassigned; a plain field expresses that better and avoids wrapping a
  framework object in an observation proxy.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `scrollTo(x, y, duration?)`, its vp units, the API 14+ duration, and error codes
  17100001 / 401.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animatetoimmediately.md` -
  `animateToImmediately` and how it differs from `animateTo` (no VSync wait).
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-explicit-animatetoimmediately
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-events -
  `onScroll` and its `yOffset`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-layout-development-stack-layout -
  `Stack` and `alignContent`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-gesture-settings -
  `gesture`, `PanGesture`, `GestureEvent`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-visibility -
  `Visibility.Visible` / `Hidden` / `None`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
