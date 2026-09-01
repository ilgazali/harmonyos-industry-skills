---
id: EDU-10
title: Bullet comments over a course video - timer-driven danmaku with colour, lane and tag settings
industry: 04_education
doc: huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
sample: huawei_industry_tree/04_education/downloads/VideoCourseBulletComment.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.CryptoArchitectureKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [Video, VideoController, onPrepared, onUpdate, onFinish, onAreaChange, setTimeout, clearTimeout, setInterval, clearInterval, translate, position, "@ObservedV2", "@Trace", ForEach, Stack, Toggle, "window.setPreferredOrientation", "display.getDefaultDisplaySync", "cryptoFramework.createRandom", generateRandomSync, animateTo]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0068, HW-04-0069, HW-04-0070, HW-04-0071, HW-04-0072, HW-04-0073, HW-04-0074, HW-04-0075, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for **danmaku over a video** - comments that scroll right to left
across the picture while it plays, with the viewer able to send their own,
choose a colour, restrict them to the top or bottom band, and turn a name tag
on.

ArkUI has no bullet-comment component, so the whole thing is built by hand from
three moving parts: a **generator** that adds comments on a randomised
`setTimeout` chain, an **animator** that advances every comment's x offset, and
a **layer** that renders them absolutely positioned over the `Video`.

Read this card as much for the shape as for the code: the sample gets the
architecture right - a singleton `@ObservedV2` model shared by the video, the
comment layer and the settings panel - and gets almost every detail of the
rendering wrong, in ways that are worth recognising because they are the classic
ArkUI performance traps.

## Feature checklist

- Comments appear on their own at a randomised 500-2000 ms cadence, cycling a
  fixed phrase list.
- Each scrolls right to left at a constant speed and is dropped once it exits.
- The viewer can type a comment and send it; it renders bold, more opaque and
  above the generated ones.
- A settings drawer offers a colour swatch row, a 默认/顶部/底部 lane selector,
  and a 显示昵称 toggle.
- Comments pause with the video and clear when it stops.
- The player forces landscape while it is open and restores portrait on exit.

## Architecture

One `entry` module. The interesting split is between the two singletons and the
components that consume them.

```
entry/src/main/ets
├── components
│   ├── VideoPlayerComponent.ets        Video + comment layer + control bar + settings
│   ├── BulletCommentViewComponent.ets  THE LAYER: the ForEach and the animation timer
│   ├── BulletSettingComponent.ets      colour / lane / tag settings drawer
│   ├── ControlBarComponent.ets
│   ├── CourseComponent.ets
│   ├── CourseListComponent.ets
│   └── IntroductionComponent.ets
├── constants/Constants.ets             every tunable number
├── model
│   ├── BulletCommentItem.ets           one comment: content, colour, x, y, speed
│   └── TrainInfoModel.ets
├── pages
│   ├── EntrancePage.ets                @Entry
│   ├── CourseDetailPage.ets            the video course page
│   └── Empty.ets
└── util
    ├── BulletCommentUtils.ets          singleton: the comment list + the generator
    ├── VideoUtils.ets                  singleton: VideoController + play/pause/stop
    └── RandomUtils.ets                 lane and delay randomness
```

The documented tree matches the zip.

**Two `@ObservedV2` singletons, and the pairing is the design.**
`VideoUtils.videoPlay()` starts the video **and** the comment generator;
`videoPause()` stops both; `videoStop()` stops both and clears the list. That is
why the comment layer never has to know about the `VideoController` and the
settings drawer never has to know about either - they all reach the same
`BulletCommentUtils.instance`.

| Owner | `@Trace` state | Read by |
| --- | --- | --- |
| `BulletCommentUtils` | `bulletComments`, `bulletColor`, `positionYIndex`, `showTag` | comment layer, settings drawer |
| `VideoUtils` | `duration`, `curTime` | control bar |

**Each comment carries its own geometry.** `BulletCommentItem` holds
`positionX` (a vp offset fed to `translate`), `positionY` (a percentage fed to
`position`), `speed`, `color` and `isUserBulletComment`. The layer is a `Stack`
and every comment is absolutely placed inside it - no layout, just two numbers
per node.

**And that is where it comes apart.** `BulletCommentItem` is a *plain* class -
no `@ObservedV2`, no `@Trace` - so the per-frame `positionX` writes are invisible
to state management. The only reason anything moves is that the animation tick
reassigns the whole `bulletComments` array afterwards (`HW-04-0070`), which
combined with a missing `ForEach` key generator (`HW-04-0069`) rebuilds every
node 60 times a second.

## Implementation steps

1. **Model a comment as an observable class.** Decorate `BulletCommentItem`
   `@ObservedV2` and its `positionX`/`positionY` `@Trace`, so moving one comment
   re-renders one node.
2. **Give it a unique id from a counter,** not `Date.now()` (`HW-04-0072`).
3. **Hold the list in an `@ObservedV2` singleton** with `@Trace bulletComments`,
   and mutate it only through array APIs or whole-array assignment - never
   `.length = 0` (`HW-04-0073`).
4. **Generate on a self-rescheduling `setTimeout` chain,** guarded by a boolean
   the stop path clears:
   ```typescript
   const GENERATE = (): void => {
     if (!this.isGeneratingNewComment) { return; }
     // ... choose a lane, push a comment ...
     this.commentTimer = setTimeout(GENERATE, RandomUtils.getDelay());
   };
   this.isGeneratingNewComment = true;
   GENERATE();
   ```
   The flag, not `clearTimeout`, is what makes this stoppable - a timer already
   in flight checks it on entry.
5. **Compute the lane as `100 / POSITION_DIVISION` percent per slot.** The
   sample's 12 slots at 33 % each put nine of them off-screen (`HW-04-0068`).
6. **Start each comment at the layer's own width,** measured with
   `onAreaChange` - not at the display height (`HW-04-0071`).
7. **Animate declaratively.** Prefer `animateTo`/`keyframeAnimation` on
   `translate` per comment over a 16 ms `setInterval` that mutates every item and
   re-renders the list (`HW-04-0070`).
8. **Key the `ForEach` explicitly** on the comment id.
9. **Pair every timer with its clear** in `aboutToDisappear`, and stop the
   generator when the video pauses.

## Verified snippets

All snippets are from `VideoCourseBulletComment.zip`. Corrected forms are marked.

**The shared model — `entry/src/main/ets/util/BulletCommentUtils.ets`** (corrected, see `HW-04-0073`)

```typescript
@ObservedV2
export class BulletCommentUtils {
  @Trace bulletComments: BulletCommentItem[] = [];
  @Trace bulletColor: Color = Constants.COLORS[0];
  @Trace positionYIndex: number = 0;      // 0 default, 1 top, 2 bottom
  @Trace showTag: boolean = false;
  private isGeneratingNewComment: boolean = false;
  private commentTimer: number | null = null;

  sendBulletComment(input: string) {
    if (!input.trim()) { return; }
    this.bulletComments.push(
      new BulletCommentItem(input, true, this.showTag, this.positionIndex, this.bulletColor));
    if (this.bulletComments.length > Constants.BULLET_COMMENTS_MAX) {
      this.bulletComments = this.bulletComments.slice(1);
    }
  }

  bulletCommentGenerator() {
    const GENERATE = (): void => {
      if (!this.isGeneratingNewComment) {
        return;                                  // the stop flag, checked on entry
      }
      if (this.commentTimer) {
        clearTimeout(this.commentTimer);
      }
      if (this.positionYIndex === 0) {
        this.positionIndex = RandomUtils.getRandom();
      } else if (this.positionYIndex === 1) {
        this.positionIndex = RandomUtils.getTop();
      } else if (this.positionYIndex === 2) {
        this.positionIndex = RandomUtils.getBottom();
      }
      const randomText = this.commentsData[BulletCommentUtils.bulletCommentIndex];
      BulletCommentUtils.bulletCommentIndex =
        (BulletCommentUtils.bulletCommentIndex + 1) % this.commentsData.length;
      this.bulletComments.push(new BulletCommentItem(randomText, false, false, this.positionIndex));
      this.commentTimer = setTimeout(GENERATE, RandomUtils.getDelay());
    };
    this.isGeneratingNewComment = true;
    GENERATE();
  }

  stopBulletCommentGenerator() {
    this.isGeneratingNewComment = false;
    if (this.commentTimer) {
      clearTimeout(this.commentTimer);
      this.commentTimer = null;
    }
  }

  endGenerator() {
    this.bulletComments = [];        // FIX: sample writes this.bulletComments.length = 0,
  }                                  //      which @Trace does not observe
}
```

**A self-rescheduling `setTimeout` beats `setInterval` here** because the
interval is randomised per comment - `getDelay()` returns 500-2000 ms, so the
cadence is irregular the way real danmaku is. `setInterval` can only be uniform.

**The stop flag is doing the real work.** `clearTimeout` alone cannot stop a
callback that has already fired; the `if (!this.isGeneratingNewComment) return;`
on entry is what guarantees the chain ends. This is the pattern to copy for any
self-rescheduling timer.

**Three ways to mutate a `@Trace` array, and only two of them work.** `push`
(array API) and whole-array assignment are observed; `length = 0` is a plain
property write and is not. The guide is explicit: *"If an array API is used to
operate numberArr, the change caused can be observed."* The sample uses all
three and the unobserved one is on the stop path, so the comments freeze on
screen when playback stops.

**The comment model — `entry/src/main/ets/model/BulletCommentItem.ets`** (corrected, see `HW-04-0068`, `HW-04-0071`, `HW-04-0072`)

```typescript
@ObservedV2                                  // FIX: sample class is undecorated
export class BulletCommentItem {
  private static nextId: number = 0;
  readonly id: number = BulletCommentItem.nextId++;   // FIX: sample uses new Date().getTime()
  content: Resource | string;
  color: string | Color;
  @Trace positionY: number;                  // FIX: sample fields are untraced
  @Trace positionX: number;
  speed: number = Constants.MOVEMENT_SPEED;
  isUserBulletComment: boolean = false;

  constructor(content: Resource | string, isUserBulletComment: boolean, showTag: boolean,
              index: number, color?: string | Color) {
    this.content = showTag ? TAG + content : content;
    this.color = color ?? '#FFFFFF';
    // FIX: sample multiplies by COMMENT_POSITION_PERCENT = 33 with POSITION_DIVISION = 12,
    //      so nine of twelve lanes fall outside the 30%-tall layer
    this.positionY = (index % Constants.POSITION_DIVISION) * (100 / Constants.POSITION_DIVISION);
    const context = AppStorage.get('context') as UIContext;
    this.positionX = context.px2vp(Constants.INITIAL_POSITIONX);
  }
}
```

**The lane arithmetic is the bug most worth remembering.** The code comment says
"the screen is divided into 12 equal parts"; twelve equal parts of 100 % is
8.33 % each, not 33 %. At 33 % the lanes run 0, 33, 66, 99 … 363 percent of the
comment layer, so only the first three are inside it - and `getBottom()`, which
picks lanes 8-11, produces nothing visible at all. A percentage-based `position`
inside a fixed-height layer has no clamping; anything past 100 % simply draws
outside.

**Two different units in one object.** `positionY` is a **percentage** consumed
by `position`; `positionX` is a **vp offset** consumed by `translate`. Mixing
them is deliberate - the lane must scale with the layer height, the horizontal
travel must not - but it means the two must never be computed the same way.
`Constants.INITIAL_POSITIONX` is `display.getDefaultDisplaySync().height`, which
is the wrong axis (`HW-04-0071`).

**The layer — `entry/src/main/ets/components/BulletCommentViewComponent.ets`** (corrected, see `HW-04-0069`, `HW-04-0070`)

```typescript
@Component
export struct BulletCommentViewComponent {
  @Link isPlaying: boolean;
  @Link showBulletComment: boolean;
  private bulletUtil: BulletCommentUtils = BulletCommentUtils.instance;
  private timerId: number = 0;

  aboutToAppear() { this.startAnimation(); }
  aboutToDisappear() { clearInterval(this.timerId); }     // paired - this part is right

  build() {
    Stack() {
      if (this.showBulletComment) {
        ForEach(this.bulletUtil.bulletComments, (item: BulletCommentItem) => {
          if (item.isUserBulletComment) {
            Text(item.content).bulletCommentText(item)
              .fontWeight(FontWeight.Bold).opacity(0.8).zIndex(1)   // the viewer's own, on top
          } else {
            Text(item.content).bulletCommentText(item).opacity(0.5)
          }
        }, (item: BulletCommentItem) => item.id.toString())   // FIX: sample passes no key generator
      }
    }
    .width(Constants.FULL_WIDTH)
    .height(Constants.HEIGHT_COMMENT_PERCENT)
  }

  private startAnimation() {
    if (this.timerId) { clearInterval(this.timerId); }
    this.timerId = setInterval(() => {
      if (!this.isPlaying) { return; }
      this.bulletUtil.bulletComments.forEach(item => {
        item.positionX -= item.speed;          // observed once BulletCommentItem is @Trace-d
      });
      // FIX: the sample reassigns the array here on EVERY tick, purely to force a re-render:
      //   this.bulletUtil.bulletComments = this.bulletUtil.bulletComments.filter(...)
      const survivors = this.bulletUtil.bulletComments.filter(item => item.positionX > -20);
      if (survivors.length !== this.bulletUtil.bulletComments.length) {
        this.bulletUtil.bulletComments = survivors;      // only when something actually left
      }
    }, 16);
  }
}

@Extend(Text)
function bulletCommentText(item: BulletCommentItem) {
  .fontSize(Constants.FONT_SIZE_SMALL)
  .fontColor(item.color)
  .translate({ x: `${item.positionX}`, y: 0 })    // horizontal travel
  .position({ y: `${item.positionY}%` })          // lane
  .borderRadius(Constants.HALF_PERCENT)
  .key(item.id.toString())                        // a TEST id, not the ForEach key
}
```

**Two things about the key, and they are different things.** `.key()` is a test
and inspection identifier. The `ForEach` diff key is the **third argument** to
`ForEach`, which the sample omits - so the default
`index + '__' + JSON.stringify(item)` applies, and `JSON.stringify` includes
`positionX`, which changes every frame. The result is a full teardown and
rebuild of every comment node 60 times a second, in the one component on the
page that must stay smooth.

**The unconditional array reassignment is a symptom, not a design.** It exists
because `BulletCommentItem` is not observable, so nothing else would trigger a
re-render. Decorate the model and the reassignment can be conditional - or, far
better, replace the whole interval with a declarative `animateTo` per comment and
let the framework interpolate.

**`aboutToDisappear` clears the interval and `VideoPlayerComponent.aboutToDisappear`
pauses the video** (which stops the generator). Both timers are paired; this is
the part of the lifecycle handling the sample gets right.

**Settings, read straight off the singleton — `entry/src/main/ets/components/BulletSettingComponent.ets`** (as shipped)

```typescript
private bulletUtil: BulletCommentUtils = BulletCommentUtils.instance;

ForEach(Constants.COLORS, (color: Color) => {
  Stack()
    .width(color === this.bulletUtil.bulletColor ? Constants.SIZE_COLOR_BLOCK_SEL
                                                 : Constants.SIZE_COLOR_BLOCK_DEF)
    .backgroundColor(color)
    .onClick(() => { this.bulletUtil.bulletColor = color; })
})

Toggle({ type: ToggleType.Switch, isOn: this.bulletUtil.showTag })
  .onChange((isOn: boolean) => { this.bulletUtil.showTag = isOn; })
```

**No `@Link`, no `@Provide`, no props.** The drawer holds the singleton directly
and writes `@Trace` fields on it; the generator reads the same fields on its next
tick. That is the payoff of putting the model in an `@ObservedV2` singleton
rather than threading it through the component tree - three components, no
plumbing.

Note the settings apply to **future** comments only: `bulletColor` and
`showTag` are read in `sendBulletComment`, and `positionYIndex` in the generator.
Comments already on screen keep their colour and lane.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block - the
video is a bundled rawfile (`$rawfile('sample.mp4')`) and nothing touches the
network or the gallery.

The player forces landscape for its lifetime:

```typescript
aboutToAppear() {
  this.windowClass = AppStorage.get('windowClass');
  this.windowClass?.setPreferredOrientation(window.Orientation.LANDSCAPE);
}
aboutToDisappear() {
  this.videoVM.videoPause();
  this.windowClass?.setPreferredOrientation(window.Orientation.PORTRAIT);
}
```

Restoring the orientation on the way out is mandatory - `setPreferredOrientation`
is a window-level setting, not a page-level one.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The video is `entry/src/main/resources/rawfile/sample.mp4`; there is no network
  source and no `AVPlayer` - this uses the simpler `Video` component.
- The phrase pool is twenty string resources cycled in order by a **static**
  index on `BulletCommentUtils`, so it does not reset between courses.
- The list is capped at `BULLET_COMMENTS_MAX = 20` **only on the user-send path**;
  the generator's `push` is bounded solely by comments leaving the screen.
- **There is no lane-collision avoidance.** Two comments can be assigned the same
  lane milliseconds apart and will overlap; real danmaku engines track per-lane
  occupancy.
- Speed is a single constant (`MOVEMENT_SPEED = 2` vp per tick) - no per-comment
  variation, and no relation to the text's width, so long comments and short ones
  travel at the same rate and can visually collide.
- The animation is timer-driven at a fixed 16 ms, so it does not adapt to the
  device's refresh rate.

## Pitfalls

- **`HW-04-0068` — nine of the twelve lanes are off-screen.**
  `POSITION_DIVISION = 12` with `COMMENT_POSITION_PERCENT = 33` gives lanes at
  0 % … 363 % of a 30 %-tall layer. 底部弹幕 mode (lanes 8-11) shows nothing at
  all. Use `100 / POSITION_DIVISION`.
- **`HW-04-0069` — the `ForEach` has no key generator,** so the default key
  embeds `positionX` and changes every frame, rebuilding every node at 60 Hz.
  `.key()` is a test id and does not substitute.
- **`HW-04-0070` — the animation is a 16 ms `setInterval` that reassigns the
  whole array every tick,** because `BulletCommentItem` is undecorated and
  nothing else would re-render. Decorate the model, or animate declaratively.
- **`HW-04-0071` — `INITIAL_POSITIONX` is the display *height*.** Comments start
  more than a screen width too far right and are invisible for seconds.
- **`HW-04-0072` — ids are `Date.now()` timestamps** and collide for comments
  created in the same millisecond - which the generator and the send button can
  easily do.
- **`HW-04-0073` — `endGenerator` clears with `.length = 0`,** which `@Trace`
  does not observe, so stopping the video leaves the comments frozen on screen.
- **`HW-04-0074` — the document prints `bulletCommentGenerator` twice** with
  different halves elided, neither with balanced braces, and the step-3 version
  omits the stop flag entirely.
- **`HW-04-0075` — the cosmetic randomness comes from `cryptoFramework`,** with a
  new `Random` allocated per value, several times a second.
- **`if (positionX !== item.positionX)` is dead code** - `speed` is a non-zero
  constant, so the values always differ.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - the default key generator and the "unique and persistent key" rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2`, `@Trace`, and which array operations are observed
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `translate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position` and percentage offsets
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setTimeout`, `setInterval`, `clearTimeout`, `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `VideoController`, `onPrepared`, `onUpdate`, `onFinish`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setPreferredOrientation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `EDU-04`, `EDU-06` - the same missing or unstable `ForEach` key defect in this industry
- `EDU-01` - the `AVPlayer` course-video player, for when the source is a network URL rather than a rawfile
