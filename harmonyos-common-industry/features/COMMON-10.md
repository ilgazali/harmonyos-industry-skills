---
id: COMMON-10
title: Personalised launch guide - a Swiper walkthrough with a custom indicator and a start button that fades in on the last page
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/10_custom_swiper_guide_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_swiper_guide_page-0000002282653913
sample: huawei_industry_tree/19_common_technical_solutions/downloads/CustomSwiperGuidePage.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [Swiper, SwiperController, "Swiper.onChange", "Swiper.onGestureSwipe", "Swiper.onAnimationStart", "Swiper.onAnimationEnd", SwiperAnimationEvent, "UIContext.animateTo", "component .opacity", "component .visibility", "component .hitTestBehavior", HitTestMode, Navigation, NavPathStack, "NavPathStack.pushPathByName", "NavDestination.onReady", NavDestinationContext, routerMap, "window.getWindowProperties", "window.setWindowLayoutFullScreen", "UIContext.px2vp", "@Provide", "@Link", "@Prop"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0016, HW-19-0017, HW-19-0018, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the product needs a **first-launch onboarding walkthrough**:
two or three full-screen illustrations the user swipes through, a custom page
indicator instead of the built-in dots, and a "立即启动" ("Start now") button that
appears only on the last page and takes the user into the app.

The interesting part is not the Swiper - it is the **transition of the button**.
The button must not pop in at the end of the animation; its opacity has to track
the drag in real time, in both directions, and it must not be tappable while it
is still transparent.

## Feature checklist

The application must:

- Show the guide pages in a full-screen `Swiper` with looping off and autoplay
  off.
- Replace the built-in indicator (`.indicator(false)`) with a custom component
  whose selected dot is a rounded bar and whose unselected dots are circles.
- Track the selected page and drive the custom indicator from it.
- Fade the start button in as the user drags from the second-to-last page onto
  the last one, proportionally to the drag offset.
- Fade it back out when the user drags away from the last page - both directions
  (HW-19-0016).
- Settle the opacity with an animation when the swipe animation ends, in both the
  "landed on last page" and "landed elsewhere" cases.
- Hide the custom indicator on the last page and show it elsewhere.
- Prevent touches on the button while it is transparent.
- Navigate to the main tab page when the button is tapped.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | reads the window width into `AppStorage` **before** loading content, sets full-screen layout, publishes and maintains the status-bar / navigation-indicator insets, then loads `pages/SwiperGuide` |
| `constants/StyleConstants.ets` | all magic numbers - `LAST_INDEX = 2`, `SECOND_TO_LAST_INDEX = 1`, durations - plus `TAB_INFO`, `TABINFO` and `GRID_DATA` |
| `components/CustomGuide.ets` | the custom indicator: `CustomGuide` (a `Row` of `GuideItem`s) and `GuideItem` (selected = rounded bar, unselected = `Circle`) |
| `pages/SwiperGuide.ets` | the guide page itself: the `Swiper`, the four swipe callbacks, the opacity state and the start button |
| `pages/TabView.ets` | the destination: `NavDestination` with `Tabs` and a custom tab bar |
| `pages/HomePage.ets` | content of the first tab |
| `resources/base/profile/route_map.json` | maps route name `TabView` to `tabViewBuilder` |

**State flow.** Two state variables carry the whole effect:

- `@State penetrability: number` - 1 means "button fully hidden", 0 means "button
  fully shown". The button binds `opacity(1 - this.penetrability)`, so the name is
  inverted relative to what it drives.
- `@State visible: Visibility` - visibility of the custom indicator.
- `@Provide selectedIndex: number` in the page, consumed as `@Link` by
  `CustomGuide` and passed down to `GuideItem`.

**Event flow.** The `Swiper` fires three relevant callbacks:

1. `onChange(index)` records `selectedIndex`, which redraws the indicator.
2. `onGestureSwipe(index, extraInfo)` runs **during** the drag. Dragging left off
   page 1 (`currentOffset < 0`) sets
   `penetrability = 1 + currentOffset / windowWidth` - a linear ramp from 1 to 0
   as the drag covers a full page width - and hides the indicator. Dragging right
   off page 2 (`currentOffset > 0`) resets `penetrability = 0` and shows the
   indicator again.
3. `onAnimationEnd(index, extraInfo)` runs when the settle animation finishes and
   snaps the state to its final value inside `animateTo`: `penetrability = 0` if
   the Swiper landed on the last page, `1` otherwise; and the indicator is hidden
   iff `selectedIndex === 2`.

`hitTestBehavior` is the piece that makes the invisible button harmless: it is
`HitTestMode.Default` only when `penetrability === 0`, and `HitTestMode.None`
otherwise, so a fully transparent button cannot swallow the swipe.

## Implementation steps

1. **Publish the window width before loading the page.** In
   `onWindowStageCreate`, `windowStage.getMainWindowSync().getWindowProperties().windowRect.width`
   into `AppStorage`, then `loadContent('pages/SwiperGuide')`. Keep it fresh if
   the window can be resized (HW-19-0018).
2. **Register the destination route.** `"routerMap": "$profile:route_map"` in
   `module.json5`, one entry naming `TabView` and its exported
   `tabViewBuilder`.
3. **Build the Swiper.** One `Column` per guide page, `.loop(false)`,
   `.autoPlay(false)`, `.indicator(false)`, `.cachedCount(2)`,
   `.curve(Curve.Linear)`.
4. **Add the custom indicator** as a sibling in a `Stack({ alignContent:
   Alignment.Bottom })`, bound to the page index via `@Provide` / `@Link`.
5. **Add the start button** in the same `Stack`, bound to
   `opacity(1 - penetrability)` and
   `hitTestBehavior(penetrability === 0 ? HitTestMode.Default : HitTestMode.None)`.
6. **Wire `onGestureSwipe` for the live ramp**, with **both** direction branches
   (HW-19-0016).
7. **Wire `onAnimationEnd` for the settle**, running the state change inside
   `this.getUIContext().animateTo({ duration, curve: Curve.EaseOut, iterations: 1,
   playMode: PlayMode.Normal }, () => { ... })`, with an `else` branch that
   restores `penetrability = 1` when the Swiper did not land on the last page.
8. **Navigate on tap** with `this.pathStack.pushPathByName('TabView', null)`, and
   have `TabView` capture its own stack in `onReady((context) => this.pathStack =
   context.pathStack)`.
9. **Use a constant log tag and matching format identifiers** if you keep the
   debug logging (HW-19-0017).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The Swiper and its callbacks

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/ets/pages/SwiperGuide.ets`

```ts
@Entry
@Component
struct SwiperGuide {
  pathStack: NavPathStack = new NavPathStack();
  private swiperController: SwiperController = new SwiperController();
  @Provide selectedIndex: number = StyleConstants.ZERO; // 初始化被选定的guide下标
  @State penetrability: number = StyleConstants.ONE;
  @State visible: Visibility = Visibility.Visible;
  private windowWidth: number = StyleConstants.ZERO;

  aboutToAppear(): void {
    this.windowWidth = this.getUIContext().px2vp(AppStorage.get('windowWidth') as number);
  }

  build() {
    Navigation(this.pathStack) {
      Stack({ alignContent: Alignment.Bottom }) {
        Swiper(this.swiperController) {
          // three Columns, each holding one full-width Image
        }
        .cachedCount(StyleConstants.CACHED_COUNT)
        .index(StyleConstants.SWIPER_INDEX)
        .autoPlay(false)
        .interval(StyleConstants.SWIPER_INTERVAL)
        .loop(false)
        .duration(StyleConstants.SWIPER_DURATION)
        .itemSpace(StyleConstants.SWIPER_ITEMSPACE)
        .indicator(false)
        .curve(Curve.Linear)
        .onChange((index: number) => {
          this.selectedIndex = index;
        })
        .onGestureSwipe((index: number, extraInfo: SwiperAnimationEvent) => {
          if (index === StyleConstants.SECOND_TO_LAST_INDEX && extraInfo.currentOffset < StyleConstants.ZERO) {
            this.penetrability = 1 + extraInfo.currentOffset / this.windowWidth;
            this.visible = Visibility.None;
          }
          if (index === StyleConstants.LAST_INDEX && extraInfo.currentOffset > StyleConstants.ZERO) {
            this.penetrability = 0;
            this.visible = Visibility.Visible;
          }
        })
        .onAnimationEnd((index: number, extraInfo: SwiperAnimationEvent) => {
          if (index === StyleConstants.LAST_INDEX) {
            this.getUIContext().animateTo({
              duration: StyleConstants.ANIMATETO_DURATION,
              curve: Curve.EaseOut,
              iterations: StyleConstants.ANIMATETO_ITERATIONS,
              playMode: PlayMode.Normal,
              onFinish: () => {
                hilog.info(0x0000, 'testTag', 'play end');
              }
            }, () => {
              this.penetrability = StyleConstants.ZERO;
            });
          } else {
            this.getUIContext().animateTo({
              duration: StyleConstants.ANIMATETO_DURATION,
              curve: Curve.EaseOut,
              iterations: StyleConstants.ANIMATETO_ITERATIONS,
              playMode: PlayMode.Normal,
              onFinish: () => {
                hilog.info(0x0000, 'testTag', 'play end');
              }
            }, () => {
              this.penetrability = StyleConstants.ONE;
            });
          }
          if (this.selectedIndex === 2) {
            this.visible = Visibility.None;
          } else {
            this.visible = Visibility.Visible;
          }
        });

        CustomGuide({ selectedIndex: $selectedIndex })
          .margin({ bottom: $r('app.string.custom_guide_botton_margin') })
          .visibility(this.visible);

        Button($r('app.string.start'))
          .width($r('app.float.start_button_width'))
          .height($r('app.float.start_button_height'))
          .margin({ bottom: $r('app.string.start_button_botton_margin') })
          .hitTestBehavior(this.penetrability === StyleConstants.ZERO ? HitTestMode.Default : HitTestMode.None)
          .opacity(1 - this.penetrability)
          .backgroundColor($r('app.color.start_button_background_color'))
          .fontColor(Color.White)
          .onClick(() => {
            this.pathStack.pushPathByName('TabView', null);
          });
      }
      .width(StyleConstants.FULL_WIDTH)
      .height(StyleConstants.FULL_HEIGHT);
    }
    .hideTitleBar(true);
  }
}
```

### The custom indicator

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/ets/components/CustomGuide.ets`

```ts
import { StyleConstants, TAB_INFO } from '../constants/StyleConstants';

@Component
export struct CustomGuide {
  @Link selectedIndex: number;

  build() {
    Row({ space: StyleConstants.CUSTOM_GUIDE_SPACE }) {
      ForEach(TAB_INFO, (index: number) => {
        GuideItem({
          guideIndex: index,
          selectedIndex: $selectedIndex,
        });
      });
    }
    .alignItems(VerticalAlign.Center)
    .justifyContent(FlexAlign.Center)
    .height($r('app.float.custom_guide_height'));
  }
}

@Component
struct GuideItem {
  @Prop guideIndex: number;
  @Link selectedIndex: number;

  build() {
    Column() {
      Column() {
        if (this.selectedIndex === this.guideIndex) {
          Row() {}
            .width($r('app.float.circle_selected_width'))
            .height($r('app.float.circle_height'))
            .borderRadius(StyleConstants.CIRCLE_SELECTED_BORDER_RADIUS)
            .backgroundColor($r('app.color.circle_select_color'));
        } else {
          Circle()
            .width($r('app.float.circle_width'))
            .height($r('app.float.circle_height'))
            .fill($r('app.color.circle_unselect_color'));
        }
      }
      .justifyContent(FlexAlign.Center);
    };
  }
}
```

### Publishing the window width before the page loads

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let windowClass: window.Window = windowStage.getMainWindowSync();
let rect: window.Rect = windowClass.getWindowProperties().windowRect;
AppStorage.setOrCreate('windowWidth', rect.width);

windowClass.setWindowLayoutFullScreen(true).then(() => { /* ... */ });

let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
AppStorage.setOrCreate('bottomRectHeight', windowClass.getWindowAvoidArea(type).bottomRect.height);
type = window.AvoidAreaType.TYPE_SYSTEM;
AppStorage.setOrCreate('topRectHeight', windowClass.getWindowAvoidArea(type).topRect.height);

windowClass.on('avoidAreaChange', (data) => { /* keeps both heights current */ });

windowStage.loadContent('pages/SwiperGuide', (err) => { /* ... */ });
```

### Route table and destination

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "TabView",
      "pageSourceFile": "src/main/ets/pages/TabView.ets",
      "buildFunction": "tabViewBuilder",
      "data": { "description": "this is TabView" }
    }
  ]
}
```

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/ets/pages/TabView.ets`

```ts
@Builder
export function tabViewBuilder() {
  TabView();
}

@Component
export struct TabView {
  pathStack: NavPathStack = new NavPathStack();

  build() {
    NavDestination() {
      Tabs({ barPosition: BarPosition.End }) {
        TabContent() { Column() { HomePage(); }; };
        TabContent() {};
        TabContent() {};
      }
      .vertical(false)
      .scrollable(false)
      .layoutWeight(StyleConstants.ONE)
      .barHeight($r('app.float.bar_height'));

      CustomTabBar();
    }
    .hideTitleBar(true)
    .onReady((context: NavDestinationContext) => {
      this.pathStack = context.pathStack;
    });
  }
}
```

## Permissions & config

**No permissions are required** - the feature is pure UI, and the shipped
`module.json5` declares no `requestPermissions` block.

`CustomSwiperGuidePage.zip#CustomSwiperGuidePage/entry/src/main/module.json5`:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "routerMap": "$profile:route_map"
  }
}
```

`main_pages.json` lists only `pages/SwiperGuide` - the guide is the launch page
and everything after it is a Navigation destination, not a page entry.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The index constants are hardcoded to three pages.** `LAST_INDEX = 2` and
  `SECOND_TO_LAST_INDEX = 1` in `StyleConstants.ets` must be updated together
  with the number of `Swiper` children; nothing derives them from the child
  count.
- **`onGestureSwipe` only fires for finger drags.** Programmatic page changes
  (`SwiperController.showNext()`) go straight to `onAnimationStart` /
  `onAnimationEnd`, so a UI that also offers a "next" button must handle the
  opacity there too.
- **`windowWidth` is captured once.** With `phone`, `tablet` and `2in1` all
  declared, a resize or fold makes the divisor stale (HW-19-0018).
- **`.indicator(false)` is mandatory for this design** - otherwise the built-in
  dots render underneath the custom indicator.
- **`opacity(0)` alone does not stop touches.** `hitTestBehavior` is what makes
  the hidden button transparent to the gesture; keep both bound to the same
  state.

## Pitfalls

- **The document's `onGestureSwipe` snippet is incomplete, which is incorrect.**
  It shows only the forward branch; the sample also handles
  `index === LAST_INDEX && currentOffset > 0`, which restores `penetrability` and
  the indicator when the user drags back off the last page. Without it the
  "real-time gradient" the document promises works in one direction only.
  (HW-19-0016)
- **The document's `onAnimationEnd` snippet also omits the `else` branch.** The
  sample animates `penetrability` back to `1` when the Swiper settles on any page
  other than the last, and separately hides or shows the indicator based on
  `selectedIndex`. Copying only the `if` leaves the button faded in permanently
  after the first visit to the last page.
- **`hilog.info(0x0000, index.toString(), '')` and
  `hilog.info(0x0000, 'testTag', 'index: ', +index)` are incorrect.** The tag must
  be a constant identifying the service or class, and "The number and type of
  parameters must map to the identifier in the format string"; as written the
  values are never printed. (HW-19-0017)
- **`penetrability` is inverted relative to what it names.** `1` hides the button
  and `0` shows it, because the binding is `opacity(1 - penetrability)`. Read
  every comparison in the file with that in mind.
- **`ForEach(TAB_INFO, (index: number) => ...)`** iterates the *values* of
  `TAB_INFO`, which happen to be `[0, 1, 2]`. The parameter is the item, not the
  loop index; it works only because the array holds its own indices.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` -
  the `Swiper` component, `onChange`, `onGestureSwipe`, `onAnimationStart`,
  `onAnimationEnd` and `SwiperAnimationEvent`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `UIContext.animateTo` and its `AnimateParam` fields.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-uicontext-uicontext#animateto
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - `hilog.info`
  parameter contract: tag semantics, format identifiers, and the requirement that
  arguments map to identifiers.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` and `TYPE_NAVIGATION_INDICATOR`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `Navigation`, `NavPathStack.pushPathByName` and the `routerMap` profile.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_swiper_guide_page-0000002282653913
