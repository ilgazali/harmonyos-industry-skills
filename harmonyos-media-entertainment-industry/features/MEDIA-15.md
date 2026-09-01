---
id: MEDIA-15
title: Bullet comments (danmaku) - a timer-driven translate overlay above a Video component
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/15_bullet_comment.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/bullet_comment-0000002284943480
sample: huawei_industry_tree/13_media_entertainment/downloads/BulletCommentDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, util, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0042, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for **danmaku**: comments that fly right-to-left across the
video itself rather than sitting in a list below it. It is the standard
interaction in Chinese video and live-streaming apps, and it appears anywhere
an audience reacts to a timeline - live sport, a webinar, a shared watch
party.

The mechanism is deliberately small: a plain array of comment objects, a
`setInterval` that decrements each object's `translateX`, and a `ForEach` that
renders each one as a `Text` with a `translate` and a `position`. No
`animateTo`, no `Canvas`, no custom node. That is a defensible trade: the
positions change every frame and are recomputed from a physics-free constant
speed, so a declarative animation would have to be cancelled and restarted
constantly.

It also shows the **input/playback coupling** that danmaku demands and that is
easy to forget: focusing the comment box pauses the video, blurring or sending
resumes it, and the scroll loop itself is gated on the play state so comments
freeze when the video does. Take that pattern even if you replace the
rendering.

**Read `HW-13-0042` and the constraints before copying the timing.** The
generator interval and the comment describing it disagree by 3x, and the
recycling scheme leans on a millisecond timestamp as a unique key.

## Feature checklist

- The bundled `sample.mp4` plays from `rawfile` through the `Video` component
  in landscape, with the built-in controls off and a custom bar instead.
- Comments scroll right to left across the top 30% of the screen, in four
  fixed lanes.
- Twenty preset comments cycle in on a timer while the video is playing.
- The user's own comments are visually distinct - the same pill, plus a white
  border.
- Typing focuses the input, which pauses the video; a send button appears only
  once the input has focus.
- Sending appends the comment, clears the input and resumes playback.
- A toggle button hides and shows the whole comment layer without stopping the
  video.
- Pausing the video freezes the comments in place; resuming continues them.
- Comments that leave the left edge are removed from the array; the user's
  backlog is capped at 50.
- Both timers are cleared when their component is destroyed.

## Architecture

One `entry` module, four files, no model layer beyond one class.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       program entry; the ability is declared "orientation": "landscape"
├── entrybackupability/EntryBackupAbility.ets
└── pages
    ├── BulletComment.ets               @Entry, 30 lines: hosts VideoPlayer and owns isPlaying
    ├── BulletCommentItem.ets           the comment class: id, content, colour, lane, translateX, speed
    ├── BulletCommentView.ets           the overlay: the scroll timer and the ForEach that draws it
    └── VideoPlayer.ets                 the Video, the control bar, the input, the preset generator
```

The documented tree matches the zip.

**The design decision worth copying** is the split between the two components
and, in particular, the two separate timers. `VideoPlayer` owns a *generator*
timer that produces new comments; `BulletCommentView` owns a *motion* timer
that moves and retires them. They have completely different natural rates - one
per second versus 60 per second - and putting them in one interval would force
either wasteful comment production or choppy motion. The array is shared by
`@Link`, so the producer appends and the mover filters without either knowing
the other exists.

`isPlaying` is `@Link`-ed all the way down from the `@Entry` page, which is
what lets both timers early-return on a paused video with a single flag.

**The decision worth avoiding** is `forceUpdate`. `BulletCommentItem` is a
plain class, not `@Observed`, so mutating `item.translateX` inside the array
does not notify ArkUI. The sample works around this by toggling a boolean and
rendering an invisible `Text(this.forceUpdate ? ' ' : '  ')` sized 0x0, purely
to dirty the component:

```typescript
// 隐藏的刷新触发器
Text(this.forceUpdate ? ' ' : '  ')
  .size({ width: 0, height: 0 })
```

It works, but it re-renders the entire overlay 60 times a second. The
supported form is `@Observed` on `BulletCommentItem` plus `@ObjectLink` in a
child component, so only the comments that actually moved are marked dirty.

## Implementation steps

1. **Model a comment as a class with its own motion state** - `translateX`
   seeded at the right edge, a `speed`, a lane, and a flag for "the user wrote
   this".
2. **Assign lanes round-robin from a static counter** so consecutive comments
   never overlap, and derive `positionY` as a percentage of the overlay
   height.
3. **Put the overlay in a `Stack` over the `Video`** with an explicit
   `position` and height, so comments occupy a band rather than the full
   frame.
4. **Move every comment on one interval** and let a single `filter` retire the
   ones past the left edge - one array pass does both jobs.
5. **Gate both timers on `isPlaying`** with an early `return`, so pausing the
   video freezes the comments without stopping and restarting timers.
6. **Clear each interval in the `aboutToDisappear` of the component that
   created it** - the two timers live in different components and each must
   clean up its own.
7. **Set the generator interval to the rate you document** - the sample's
   comment says 每3秒生成一条 but passes 1000 ms (`HW-13-0042`).
8. **Pause on input focus, resume on blur and on send**, and reveal the send
   button only while the input is active so the control bar stays sparse.
9. **Cap the array.** Retirement handles comments that scroll away, but a user
   spamming the send button while paused never triggers it; the sample slices
   at 50.

## Verified snippets

All snippets are from `BulletCommentDemo.zip`.

**The comment model — `entry/src/main/ets/pages/BulletCommentItem.ets`** (as shipped)

```typescript
export class BulletCommentItem {
  private static trackIndex: number = 0;
  id: number;
  content: Resource | string;
  color: string;
  positionY: number;
  translateX: number = 1000;                 // 初始位置(右侧): start off-screen right
  speed: number;                             // vp per tick
  isUserBulletComment: boolean = false;

  constructor(content: Resource | string, isUserBulletComment: boolean = false) {
    this.id = new Date().getTime();
    this.content = content;
    this.color = '#FFFFFF';
    BulletCommentItem.trackIndex = (BulletCommentItem.trackIndex + 1) % 4;
    this.positionY = BulletCommentItem.trackIndex * 33;   // four lanes at 0/33/66/99 %
    this.speed = 2;
    this.isUserBulletComment = isUserBulletComment;
  }
}
```

**The static `trackIndex` is what keeps comments from colliding.** Lane
assignment is a class-level round robin, not a random number, so four
consecutive comments occupy four different lanes by construction - a random
lane would clump. `positionY` is a percentage (0, 33, 66, 99) of the overlay's
own height, so the lane spacing scales with the band rather than with the
screen.

`content` is typed `Resource | string` because the presets are `$r(...)`
string resources and the user's are raw text. That union goes straight into
`Text(item.content)`, which accepts both - a small thing that saves a branch
at render time and keeps the presets translatable.

Two weaknesses are worth knowing. `translateX = 1000` is a hardcoded start
position in vp, not the measured width, so on a wider window comments appear
already partly on screen. And `id` is `new Date().getTime()` - millisecond
resolution - while `.key(item.id.toString())` uses it as the ArkUI key; two
comments created in the same millisecond share a key.

**The motion loop — `entry/src/main/ets/pages/BulletCommentView.ets`** (as shipped)

```typescript
@Component
export struct BulletCommentView {
  @State forceUpdate: boolean = false;
  @Link bulletComments: BulletCommentItem[];
  @Link isPlaying: boolean;
  @Link showBulletComment: boolean;
  private timerId: number = 0;

  aboutToAppear() {
    this.startAnimation();
  }

  aboutToDisappear() {
    clearInterval(this.timerId);
  }

  private startAnimation() {
    if (this.timerId) {
      clearInterval(this.timerId);              // never stack two intervals
    }
    this.timerId = setInterval(() => {
      if (!this.isPlaying) {
        return;                                 // paused video freezes the comments in place
      }
      let needsUpdate = false;
      this.bulletComments.forEach(item => {
        const NEWX = item.translateX - item.speed;
        if (NEWX !== item.translateX) {
          item.translateX = NEWX;
          needsUpdate = true;
        }
      });
      const BEFORELENGTH = this.bulletComments.length;
      this.bulletComments = this.bulletComments.filter(item => item.translateX > -20);
      if (needsUpdate || this.bulletComments.length !== BEFORELENGTH) {
        this.forceUpdate = !this.forceUpdate;   // dirty the component; see Architecture
      }
    }, 16);                                     // 60fps
  }
}
```

**One pass does three jobs**: advance, decide whether anything changed, and
retire. The `filter` reassigns `this.bulletComments`, which - because it is a
`@Link` to an array - is the one mutation ArkUI does observe on its own; the
`forceUpdate` toggle exists only for the in-place `translateX` writes, which it
does not.

The `if (!this.isPlaying) return` is the right way to pause. Clearing and
recreating the interval on every play/pause would lose the phase and risk
leaking a timer on a fast toggle; an early return keeps one timer for the life
of the component and costs a comparison per frame.

`clearInterval(this.timerId)` at the top of `startAnimation` is defensive but
correct: `aboutToAppear` can run again if the component is re-created, and a
stacked interval would double the scroll speed with no visible cause.

`translateX > -20` retires a comment 20 vp past the left edge rather than at
0, which covers the text's own width well enough for short comments. A long
comment is retired while its tail is still visible - measure the text if you
need this exact.

**The preset generator — `entry/src/main/ets/pages/VideoPlayer.ets`** (corrected, see `HW-13-0042`)

```typescript
private static bulletCommentIndex: number = 0;
private presetBulletComments = [
  $r('app.string.comment1'), $r('app.string.comment2'), /* ... 20 in total ... */
  $r('app.string.comment20')
];
private bulletCommentInterval: number = 0;

aboutToAppear() {
  this.startBulletCommentGenerator();
  let initBulletComments = [$r('app.string.comment1'), $r('app.string.comment2')];
  this.bulletComments = initBulletComments.map(text => new BulletCommentItem(text));
}

aboutToDisappear() {
  clearInterval(this.bulletCommentInterval);
}

// 根据播放时间均匀生成弹幕
private startBulletCommentGenerator() {
  // 每3秒生成一条
  this.bulletCommentInterval = setInterval(() => {
    if (!this.isPlaying) {
      return;
    }
    let randomText = this.presetBulletComments[VideoPlayer.bulletCommentIndex];
    VideoPlayer.bulletCommentIndex =
      (VideoPlayer.bulletCommentIndex + 1) % this.presetBulletComments.length;
    this.bulletComments = [...this.bulletComments, new BulletCommentItem(randomText)];
  }, 3000);                                     // FIX: sample passes 1000 under a "每3秒" comment
}
```

**Two comments seed the screen immediately and the rest arrive on the timer.**
Without those two, the first three seconds of the video show an empty overlay
and the feature looks broken - a cheap trick worth keeping in any demo of a
time-based stream.

The append is `[...this.bulletComments, item]`, a new array rather than a
`push`, because `@State`/`@Link` on an array observes reassignment, not
mutation. That is the same reason the motion loop needs `forceUpdate` and the
retirement `filter` does not.

The index is a **static** on the component, not an instance field, so the
preset rotation survives re-entering the page - which is either a nice touch
or a leak of state between page instances, depending on your view. It is also
why `bulletCommentIndex` never resets.

The interval is the finding (`HW-13-0042`): the comment says one comment every
3 s, the code passes 1000 ms, so the twenty presets recycle every 20 seconds
instead of every minute and the screen is three times as busy as designed.

**Input coupled to playback — same file** (as shipped)

```typescript
TextInput({ text: this.bulletCommentInput, placeholder: $r('app.string.placeholder') })
  .onFocus(() => this.onInputFocus())
  .onBlur(() => this.resumePlayback())
  .onChange((value: string) => {
    this.bulletCommentInput = value;
  })

// 点击输入框后才有弹幕发送按钮
if (this.isInputActive) {
  Button() {
    Image($r('app.media.send')).width(24).height(24)
  }
  .onClick(() => {
    this.sendBulletComment();
    this.isInputActive = false;
    this.showControls = false;                 // hide the bar so the comment is visible
  })
}

private sendBulletComment() {
  if (this.bulletCommentInput.trim()) {
    this.bulletComments = [...this.bulletComments, new BulletCommentItem(this.bulletCommentInput, true)];
    this.bulletCommentInput = '';
    if (this.bulletComments.length > 50) {
      this.bulletComments = this.bulletComments.slice(1);   // cap the backlog
    }
  }
  this.resumePlayback();                                    // 发送后恢复播放
}

private onInputFocus() {
  this.isInputActive = true;
  if (this.isPlaying) {
    this.controller.pause();
    this.isPlaying = false;
  }
}

private resumePlayback() {
  this.isInputActive = false;
  this.controller.start();
  if (!this.isPlaying) {
    this.isPlaying = true;
  }
}
```

**Pausing on focus is not politeness, it is correctness.** The comment the
user is typing is anchored to a moment in the video; if the video runs on
while the keyboard is up, the comment lands seconds late. Pausing also stops
the generator and the scroll loop, both gated on the same flag, so the screen
is quiet while the user types.

`isPlaying` is written here **and** by the `Video` component's `onUpdate`
(`if (event.time > 0 && !this.isPlaying) this.isPlaying = true`) and
`onFinish`. Three writers on one flag is the fragile part of this design: a
pause issued from `onInputFocus` is re-set to true by the next `onUpdate` tick
if the player has not actually stopped yet. Prefer driving the flag from the
player's events alone and treating focus as a request to pause.

The send button appearing only when `isInputActive` is a real space saving on
a landscape control bar - and it doubles as the affordance that tells the user
the input took focus.

## Permissions & config

**None.** The sample declares no `requestPermissions`. The video is a
`$rawfile('sample.mp4')` inside the package, so there is no file access and no
network.

The one configuration worth noting is on the ability itself:

```json5
"abilities": [
  {
    "name": "EntryAbility",
    "srcEntry": "./ets/entryability/EntryAbility.ets",
    "exported": true,
    "orientation": "landscape",
    ...
  }
]
```

`"orientation": "landscape"` locks the app to landscape at the manifest level
rather than by calling `setPreferredOrientation` at runtime. For a
single-purpose video app that is the simpler choice and avoids a visible
rotation on launch. The overlay's `translateX = 1000` start position is
implicitly tuned to that landscape width.

The comment strings are `app.string.comment1` .. `comment20` resources, so the
preset track is translatable; the user's own comments are raw strings, as they
must be.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Four lanes only, at a fixed 33% spacing. A busy stream overlaps comments in
  the same lane - there is no collision test against the previous comment's
  tail.
- `speed` is a constant 2 vp per 16 ms tick (about 125 vp/s) for every comment,
  so a long comment and a short one travel at the same rate and never overtake.
  Real danmaku scales speed by text length so all comments clear the screen in
  the same time.
- `translateX` starts at a hardcoded 1000 vp. On a window wider than that,
  comments appear already on screen; on a much narrower one they idle
  off-screen for a while.
- `id` is a millisecond timestamp used as the ArkUI `key`; two comments created
  within the same millisecond collide.
- The `ForEach` has no key generator, so it falls back to index keying over an
  array that is filtered from the front every frame.
- The whole overlay re-renders 60 times a second because motion is applied by
  mutating plain objects and dirtying the component with a hidden `Text`. Use
  `@Observed` / `@ObjectLink` if the comment count is large.
- The next-track button and the back button are decorative; there is one video
  and no navigation.
- The comment array is in memory only. There is no network, no persistence and
  no association between a comment and a playback timestamp - a real danmaku
  system stores the video offset with each comment and replays them on seek.

## Pitfalls

- **`HW-13-0042`** (D/low, confirmed) — the generator's own comment says
  每3秒生成一条 (one every 3 s) and the interval passed to `setInterval` is
  `1000`. Comments arrive three times faster than designed, so the twenty
  presets recycle every 20 seconds and repetition becomes obvious. Fix: pass
  `3000`, or correct the comment to match the intended rate.

## References

- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - the `translate` attribute the motion is built on
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position`, used for the lane offset
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `VideoController`, `onPrepared`, `onUpdate`, `onFinish`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `onFocus` / `onBlur` / `onChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and why the key generator matters here
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-page-transition-animation.md` - the page-transition reference the document cites for `translate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-page-transition-animation
- `MEDIA-16` - the comment sheet for the same interaction when comments are a list rather than an overlay
