---
id: MEDIA-29
title: Speed lock - long-press to run at 2x, slide down to keep it after release
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/29_lock_speed.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/lock_speed-0000002317513156
sample: huawei_industry_tree/13_media_entertainment/downloads/LockSpeed.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [GestureGroup, GestureMode, LongPressGesture, PanGesture, "GestureEvent.fingerList", Video, VideoController, PlaybackSpeed, currentProgressRate, display, "@StorageLink", window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0066, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a full-screen player needs **two levels of fast-forward**:
a transient one that lasts only as long as the finger is down, and a sticky one
the user commits to by continuing the same gesture. This is the short-video
convention - press and hold the edge of the screen to skim at 2x, and if you
want to stay at 2x, slide down without lifting.

The transferable mechanic is `GestureGroup(GestureMode.Sequence, ...)`. A
sequence group fires its gestures in order: the long press must succeed before
the pan is even considered, so a plain drag scrolls the feed and only a
*held* drag reaches the lock logic. The same shape covers every "press, then
drag to commit or cancel" interaction - hold-to-record with slide-to-cancel in
a messenger, press-and-drag volume or brightness on a player, hold-and-drag to
pin a card.

The state design is the second transferable part: the lock is a separate
boolean that *overrides* the transient rate rather than replacing it, so the
gesture-end handler that resets the transient rate to 1x cannot accidentally
undo the lock. **Read `HW-13-0066` before shipping the threshold** - the sample
decides "far enough down" against an absolute 750 vp, which does not exist on
many phones.

## Feature checklist

- A full-screen looping video plays on entry with the system controls hidden.
- Two invisible 20%-wide columns sit at the left and right edges; both carry the
  same gesture group.
- Long-pressing either column raises playback to 2x, shows a 倍速播放中
  (playing at speed) pill near the top, and hides the right-hand action rail.
- If the video was paused, the long press also resumes it.
- While the press is held, a bar at the bottom tells the user to slide down to
  lock, and its icon and wording change once the finger passes the threshold.
- Releasing past the threshold locks the rate: a 已锁定 (locked) pill shows for
  three seconds and playback stays at 2x after the finger is lifted.
- Sliding past the threshold again while locked unlocks it and returns to 1x.
- Releasing without reaching the threshold returns to 1x immediately.
- Tapping anywhere else toggles play/pause; a cancelled gesture also drops back
  to 1x.
- Sending the app to the background writes `isPlay = false` into `AppStorage`.

## Architecture

One `entry` module, one page, two components, one constants file. There is no
model or view-model layer - the whole feature is five `@State` fields on
`ShortVideo`.

```
entry/src/main/ets
├── common/Constants.ets           layout numbers + LOCKED_RATE + LOCKED_POSITION
├── component/
│   ├── ShortVideo.ets             the Video, the gesture columns, all five overlays (451 lines)
│   └── VideoDes.ets               the title/description block over the bottom-left corner
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage, isPlay on background
├── entrybackupability/
└── pages/LockSpeed.ets            @Entry, twelve lines: a Flex holding ShortVideo
```

The documented tree matches the zip exactly, file for file.

**The design decision worth copying** is that the locked rate is never written
into the transient rate. Two fields carry the speed:

```typescript
@State isLocked: boolean = false;                                        // sticky
@State curRate: PlaybackSpeed = PlaybackSpeed.Speed_Forward_1_00_X;      // transient
// ...
currentProgressRate: this.isLocked ? Constants.LOCKED_RATE : this.curRate
```

`curRate` belongs to the gesture: the long press sets it to 2x, every exit path
(`onActionEnd` below the threshold, `onCancel`) sets it back to 1x. `isLocked`
belongs to the user's decision and outlives the gesture. Because the `Video`
component *derives* its rate from both, the reset in the gesture handler is
harmless while locked - there is no ordering hazard between "restore the speed
on release" and "keep the speed because it is locked", which is exactly the bug
that appears when a single `rate` field is mutated by both concerns. Compare
`MEDIA-17`, where one `speedIndex` field carries both roles and starts life at
the wrong value (`HW-13-0045`).

**Worth avoiding**: the left and right gesture columns are 55 lines of
copy-pasted `GestureGroup`, identical except that the right column drops the
empty `onActionStart`. Extract it into one `@Builder edgeGesture()` used twice;
as shipped, any fix to the lock logic has to be applied in two places.

## Implementation steps

1. **Lay two narrow, invisible columns over the video**, `20%` wide and full
   height, inside a `Row` with `FlexAlign.SpaceBetween` and a `Blank()` between
   them. They are pure gesture surfaces - no background, no children.
2. **Attach a sequence gesture group**:
   `GestureGroup(GestureMode.Sequence, LongPressGesture({ repeat: true }), PanGesture())`.
   Order matters - in `GestureMode.Sequence` the pan is only recognised after
   the long press has fired, so an ordinary swipe never reaches the lock code.
3. **In `LongPressGesture.onAction`, guard on `event.repeat`** so the rate is
   raised on the repeat ticks of a held press, and resume playback if the video
   was paused when the press started.
4. **Track the finger in `PanGesture.onActionUpdate`** by reading
   `event.fingerList[i].localY` - `localY` is relative to the gesture column, so
   it is already the "how far down the screen" number the lock needs.
5. **Decide the lock in `onActionEnd`**, not in `onActionUpdate`: passing the
   threshold does nothing until the finger is lifted, which is what makes the
   gesture cancellable by sliding back up.
6. **Derive the threshold from the measured height of the gesture column, not
   from a constant** (`HW-13-0066`). Capture the height with `onAreaChange` and
   compare against `height * ratio`.
7. **Toggle, do not set**: the same "release past the threshold" event locks
   when unlocked and unlocks when locked, so the two branches differ only in the
   value they leave in `isLocked` and `curRate`.
8. **Show the confirmation for three seconds** with a `setTimeout` that clears
   `isShow`, and gate its `visibility` on `this.isShow && !this.isAccelerate` so
   the lock pill never overlaps the speed pill.
9. **Bind the player to both flags** with
   `currentProgressRate: this.isLocked ? Constants.LOCKED_RATE : this.curRate`,
   and reset only `curRate` from `onCancel`.

## Verified snippets

All snippets are from `LockSpeed.zip`. Corrected forms are marked.

**The sequence gesture group — `entry/src/main/ets/component/ShortVideo.ets`** (as shipped, left column)

```typescript
Column()
  .width(Constants.GESTURE_WIDTH)      // '20%'
  .height(Constants.FULL_HEIGHT)
  .gesture(
    GestureGroup(GestureMode.Sequence,
      LongPressGesture({ repeat: true })
        .onAction((event: GestureEvent | undefined) => {
          // 快进图标显隐变化 - swap the two chevrons in the speed pill
          this.leftRateOpacity = 1;
          this.rightRateOpacity = 0;
          if (event) {
            if (event.repeat) {
              this.isAccelerate = true;
              this.curRate = PlaybackSpeed.Speed_Forward_2_00_X;
              if (!this.isPlay) {
                this.isPlay = !this.isPlay;
                this.controller.start();
              }
            }
          }
        }),
      PanGesture()
        .onActionUpdate((event: GestureEvent) => {
          // 获取拖动位置的y坐标 - the finger's Y inside this column
          for (let i = 0; i < event.fingerList.length; i++) {
            this.positionY = event.fingerList[i].localY;
          }
        })
    )
      .onCancel(() => {
        this.isAccelerate = false;
        this.curRate = PlaybackSpeed.Speed_Forward_1_00_X;
      })
  );
```

**`GestureMode.Sequence` is the whole trick.** In a sequence group the child
gestures are recognised strictly in order and the group only continues while
the earlier ones stay satisfied, so the `PanGesture` here is not a general drag
handler - it is "drag, while still holding". That is why the columns can sit on
top of a scrollable feed without stealing swipes.

`LongPressGesture({ repeat: true })` fires `onAction` repeatedly for as long as
the press is held. The `event.repeat` guard means the rate is raised on those
repeat ticks rather than on the first recognition, which is what gives the small
delay users expect before a hold "engages". `onCancel` on the *group* covers
the case where the system takes the gesture away mid-press - without it the
video would be stranded at 2x with no gesture left to end it.

**The lock decision — same file** (corrected, see `HW-13-0066`)

```typescript
@State gestureHeight: number = 0;                     // FIX: measured, not assumed

// on the gesture Column:
  .onAreaChange((_oldValue: Area, newValue: Area) => {
    this.gestureHeight = newValue.height as number;
  })

PanGesture()
  .onActionEnd(() => {
    const lockedPosition = this.gestureHeight * Constants.LOCKED_RATIO;   // FIX: was 750 vp flat
    if (this.positionY >= lockedPosition && !this.isLocked) {
      this.isLocked = true;
      this.isShow = true;
      setTimeout(() => {
        this.isShow = false;
      }, 3000);
      this.curRate = PlaybackSpeed.Speed_Forward_2_00_X;
    } else if (this.positionY >= lockedPosition && this.isLocked) {
      this.isLocked = false;
      this.isShow = true;
      setTimeout(() => {
        this.isShow = false;
      }, 3000);
      this.curRate = PlaybackSpeed.Speed_Forward_1_00_X;
    } else {
      this.curRate = PlaybackSpeed.Speed_Forward_1_00_X;
    }
    this.isAccelerate = false;
    this.positionY = 0;
  })
```

**The threshold must be relative.** As shipped the comparison is
`this.positionY >= Constants.LOCKED_POSITION` with `LOCKED_POSITION = 750`,
a vp literal. The gesture column is `height('100%')` of a full-screen page, so
on a device whose usable height is under 750 vp the condition can never be
true and the feature - the entire point of the sample - is unreachable; on a
tall device it fires well above the hint bar the user is aiming at. Measuring
the column with `onAreaChange` and keeping a ratio constant instead makes the
same code work on a phone, a tablet and a resized 2in1 window.

Note the third branch: releasing *without* reaching the threshold drops
`curRate` back to 1x but leaves `isLocked` untouched. That is the branch that
makes the two-field design pay off - a locked user who long-presses and
releases short of the threshold stays locked.

**The player, driven by two flags — same file** (as shipped)

```typescript
Video({
  src: $r('app.media.video3'),
  controller: this.controller,
  currentProgressRate: this.isLocked ? Constants.LOCKED_RATE : this.curRate
})
  .width(Constants.FULL_WIDTH)
  .height(this.screenHeight - Constants.VIDEO_HEIGHT)
  .loop(true)
  .autoPlay(true)
  .muted(false)
  .controls(false);
```

`currentProgressRate` takes the `PlaybackSpeed` enum, and `Constants.LOCKED_RATE`
is `PlaybackSpeed.Speed_Forward_2_00_X` - the locked speed is a named constant,
not a repeated enum reference, so a product decision to lock at 3x is one edit.
`.controls(false)` is required: the built-in control bar would carry its own
speed affordance and fight the gesture. `.autoPlay(true)` with `.loop(true)`
makes the page a feed cell rather than a player screen.

**The hint bar that teaches the gesture — same file** (as shipped)

```typescript
@Builder
BottomBar() {
  if (this.isAccelerate) {
    if (this.isLocked) {
      Row() {
        Image(this.positionY < Constants.LOCKED_POSITION ? $r('app.media.ic_slide_down') :
          $r('app.media.ic_unlocked'))
          .width(24)
          .margin({ top: 10 });
        Text(this.positionY < Constants.LOCKED_POSITION ? $r('app.string.slide_down_up_unlock_rate') :
          $r('app.string.up_unlock_rate'))
          .fontColor(Color.White)
          .margin({ top: 10 });
      }
      .width(Constants.BOTTOM_BAR_WIDTH)
      .margin({
        top: this.screenHeight - Constants.BOTTOM_BAR_MARGIN,
        left: (this.screenWidth - Constants.BOTTOM_BAR_WIDTH) / 2
      });
    }
    // ... the unlocked branch is the same Row with ic_locked / up_lock_rate
  } else {
    Image($r('app.media.bottom'))          // the ordinary tab bar image
      .width(Constants.FULL_WIDTH)
      .height(80);
  }
}
```

**The hint bar is the only feedback the gesture has.** It reads `positionY`
live during the pan, so the icon flips from 下滑 (slide down) to 松手锁定
(release to lock) the moment the finger crosses the threshold, and the wording
flips again when the current state is already locked - four combinations from
two booleans and one number. Copy the pattern, but note that this bar compares
against the same absolute `LOCKED_POSITION`: the corrected relative threshold
from `HW-13-0066` has to be applied here too, or the promise on screen and the
condition in `onActionEnd` disagree.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; the only media is a
bundled `app.media.video3`, so nothing needs a grant.

`EntryAbility` does the immersive setup: `setColorMode(COLOR_MODE_LIGHT)`,
`setWindowLayoutFullScreen(true)`, and it publishes the status-bar and
navigation-bar avoid-area heights into `AppStorage` under `topRectHeight` /
`bottomRectHeight`. Two things about that are worth knowing before copying it:

- the heights are stored **in px** without a `px2vp` conversion, and
- `ShortVideo` never reads either key. It measures the screen itself with
  `display.getAllDisplays` and converts with `px2vp`, so the avoid-area
  publication in this sample is dead code.

`onBackground` writes `AppStorage.setOrCreate('isPlay', false)`, which the
`@StorageLink('isPlay')` in `ShortVideo` picks up so the play button reappears
when the user comes back.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `display.getAllDisplays` is **asynchronous** and is called from
  `aboutToAppear`, so `screenWidth` and `screenHeight` are `0` for the first
  frame. Everything positioned from them - the video height, the centred pills,
  the bottom bar - lays out at the wrong place for one frame.
- Overlay positions are hardcoded vp literals (`top: 352` for the play button,
  `bottom: 103` for the action rail, `top: 100` for the pills). They are tuned
  for one phone size and are the second thing to make relative after
  `HW-13-0066`.
- `avoidAreaChange` is registered in `onWindowStageCreate` and never released in
  `onWindowStageDestroy` - the same boilerplate leak that recurs across this
  industry's samples.
- The action rail, the search icon, the recommend badge and the description
  block are all toasts (`app.string.toast_tips`). This is a gesture sample, not
  a feed.

## Pitfalls

- **`HW-13-0066`** (B/medium, probable): the lock threshold is an absolute
  `750` vp compared against `localY`, while the gesture column is `100%` of the
  screen height - on devices with less than 750 vp of usable height the lock can
  never be triggered, and on tall devices it triggers far above the hint bar.
  Fix: measure the column (`onAreaChange`) and compare against
  `height * ratio`; apply the same change in `BottomBar`.
- No systematic finding from elsewhere in this industry names `LockSpeed`; the
  speed-state defect `HW-13-0045` and the prepare-polling leak `HW-13-0046`
  both belong to the `AVPlayer`-based siblings (`MEDIA-17`, `MEDIA-32`) and do
  not apply here, because this sample drives a plain `Video` component with no
  state machine and no polling.

## References

- `huawei_industry_tree/13_media_entertainment/docs/29_lock_speed.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/lock_speed-0000002317513156
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-combined-gestures.md` - `GestureGroup` and `GestureMode.Sequence`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `repeat`, `duration`, and the `onAction` event
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `fingerList`, `localY` and the action callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `currentProgressRate`, `PlaybackSpeed`, `VideoController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `MEDIA-17` - the same long-press-to-2x interaction on an `AVPlayer`, and the single-field speed state this sample avoids
- `MEDIA-32` - the other `AVPlayer` player in this industry, with the same speed menu
