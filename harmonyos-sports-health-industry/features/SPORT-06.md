---
id: SPORT-06
title: Match scorer - a TextTimer countdown with tap-to-score panels
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/06_match_scorer.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/match_scorer-0000002329513545
sample: huawei_industry_tree/03_sports_health/downloads/MatchScorer.zip
kits: ["@kit.ArkUI"]
apis: [TextTimer, TextTimerController, isCountDown, count, format, onTimer, start, pause, reset, onTouch, TouchEvent, TouchType, TextInput, onSubmit, "@State", "@Link", "@Builder", ImageInterpolation]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0026, HW-03-0027, HW-03-0028, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for a **scoreboard the user operates during a live event** -
basketball, table tennis, badminton, any match with a clock and two sides that
accumulate points. The document names those three.

Two things are worth taking:

- **`TextTimer` in countdown mode** with a `TextTimerController`, which gives
  you a clock with pause, resume and reset for free - no interval, no handle
  to leak. Compare `SPORT-03`, which needs its own `setInterval` because it is
  driving a drawing rather than showing time.
- **Score buttons driven by `onTouch` rather than `onClick`**, so the press
  can be shown immediately on `Down` and the point awarded on `Up`. That
  separation is what makes a scoreboard feel responsive when taps come fast -
  and it is what makes handling `Cancel` mandatory (`HW-03-0026`).

## Feature checklist

- A countdown clock, ten minutes by default, in mm:ss.
- Play/pause toggling the clock, and a reset that returns it to the full
  period.
- Two team panels, each with an editable team name and a running score.
- Three quick-score buttons per team: +1, +2, +3.
- Each button highlights while held and awards its points on release.
- Scores are capped rather than allowed to run away.

## Architecture

One `entry` module, one page, one reusable panel.

```
entry/src/main/ets
├── constant/Constants.ets       TIME_COUNT, MAX_SCORE, colours, spacings
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/MatchView.ets          @Entry - the clock, the controller, two ScorerViews
    └── view/ScorerView.ets      one team: name, score, three score buttons
```

The documented tree spells the backup directory `entrybackupablility`
(`HW-03-0028`).

**The page owns the clock, the panel owns nothing.** `MatchView` holds the
single `TextTimerController` and both scores; `ScorerView` takes its score by
`@Link` and pushes points back into it. So the same panel component serves
both teams with no duplication and no shared state between them:

```
MatchView
├── TextTimer + TextTimerController        the clock, owned here
├── ScorerView({ teamScore: $homeScore })  @Link
└── ScorerView({ teamScore: $awayScore })  @Link
```

**Team names are edited in place** - a `Text` swapped for a `TextInput` on an
`isEditing` flag, with `onSubmit` switching back. That avoids a dialog for a
one-field edit.

## Implementation steps

1. **Configure the clock declaratively**: `TextTimer({ isCountDown: true, count, controller })`
   with a `format` for the display.
2. **Drive it through the controller** - `start()`, `pause()`, `reset()` - and
   keep one boolean for which icon to show.
3. **Observe the end of the countdown with `onTimer`** (`HW-03-0027`).
4. **Build the score buttons with `onTouch`**, setting the pressed flag on
   `Down`, awarding on `Up`, and clearing on `Cancel` (`HW-03-0026`).
5. **Cap the score** with `Math.min` at the point of increment.
6. **Bind the score with `@Link`** so one panel component works for both
   teams.

## Verified snippets

All snippets are from `MatchScorer.zip`. Corrected forms are marked.

**The clock — `entry/src/main/ets/pages/MatchView.ets`** (corrected, see `HW-03-0027`)

```typescript
@State isPause: boolean = false;
textTimerController: TextTimerController = new TextTimerController();

TextTimer({ isCountDown: true, count: Constants.TIME_COUNT, controller: this.textTimerController })
  .format(Constants.TIME_FORMAT)
  .fontSize($r('app.float.font_size_38'))
  .onTimer((utc: number, elapsedTime: number) => {          // FIX: absent in the sample
    if (elapsedTime >= Constants.TIME_COUNT) {
      this.textTimerController.pause();
      this.isPause = false;
      // signal the end of the period
    }
  })
```

**`isCountDown: true` with `count` in milliseconds** is the whole
configuration - `TIME_COUNT` is 600000, ten minutes. The component keeps the
elapsed time itself, so there is no state to hold and nothing to clean up;
`onTimer` is the only way to read it back, which is why the end of the period
cannot be detected without it.

**Play, pause and reset — same file** (as shipped)

```typescript
Image(this.isPause ? $r('app.media.pause') : $r('app.media.play'))
  .interpolation(ImageInterpolation.High)
  .onClick(() => {
    if (this.isPause) {
      this.textTimerController.pause();
    } else {
      this.textTimerController.start();
    }
    this.isPause = !this.isPause;
  });

// the reset control
.onClick(() => {
  this.textTimerController.reset();
  this.isPause = false;
});
```

The flag reads as "the clock is running, so the button offers pause" rather
than "the clock is paused" - worth renaming to `isRunning` when adapting this,
since the current name inverts what a reader expects.

`reset()` returns the timer to `count` and leaves it stopped, so setting the
flag to `false` afterwards correctly restores the play icon.

`ImageInterpolation.High` on a small control icon is a detail worth copying:
it keeps a scaled-down asset from looking ragged.

**The score buttons — `entry/src/main/ets/view/ScorerView.ets`** (corrected, see `HW-03-0026`)

```typescript
@State isClick: boolean[] = [false, false, false];
@Link teamScore: number;

@Builder
addScore(score: number, index: number) {
  Column() {
    Text(`+${score}分`)
      .fontColor(this.isClick[index] ? $r('app.color.font_color_blue') : Constants.DARK_FONT_COLOR)
  }
  .backgroundColor(Constants.BUTTON_BACKGROUND_COLOR)
  .borderRadius($r('app.float.radius_20'))
  .onTouch((event: TouchEvent) => {
    if (event) {
      if (event.type === TouchType.Down) {
        this.isClick[index] = true;                   // show the press immediately
      } else if (event.type === TouchType.Up) {
        this.teamScore = Math.min(this.teamScore + score, Constants.MAX_SCORE);
        this.isClick[index] = false;
      } else if (event.type === TouchType.Cancel) {   // FIX: absent in the sample
        this.isClick[index] = false;                  // no points, but clear the highlight
      }
    }
  })
}
```

**`onTouch` over `onClick` is the deliberate choice.** A click fires once, at
the end; a scoreboard wants the button to light up the instant the finger
lands, because the operator is watching the court and not the screen. The cost
is owning the whole touch lifecycle - and `Cancel` is a real terminal state,
reached whenever the finger slides off the button or a scroll claims the
gesture.

**One flag per button in an array indexed by the builder's parameter** is what
lets a single `@Builder` serve +1, +2 and +3 without three sets of state.

`Math.min` at the increment caps the score at `MAX_SCORE` (9999) rather than
validating afterwards - the right place, since the increment is the only
writer.

**One panel, two teams — `MatchView.ets` and `ScorerView.ets`** (as shipped)

```typescript
// ScorerView
@State teamName: ResourceStr = $r('app.string.name_null');
@State isEditing: boolean = false;
@Link teamScore: number;

// editing the team name in place
if (!this.isEditing) {
  Text(this.teamName)
    .maxLines(Constants.TEAMNAME_MAX_LINE)
    .textOverflow({ overflow: TextOverflow.Ellipsis })
} else {
  TextInput({ placeholder: $r('app.string.name_null'), text: this.teamName })
    .onChange((value: string) => { this.teamName = value; })
    .onSubmit(() => { this.isEditing = false; })
}
```

`@Link` for the score and `@State` for the name is the right division: the
score belongs to the match and is read by the page, the name belongs to the
panel and nobody outside needs it.

## Permissions & config

**None.** The sample declares no `requestPermissions` - it is a clock and two
counters.

No routing configuration; a single `@Entry` page.

`Constants.ets` holds `TIME_COUNT` (600000 ms), `MAX_SCORE` (9999), the time
format and the colour and spacing values.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`onTimer` does not fire in the background or on a locked screen.** The
  reference states it: "This event is not triggered when the screen is locked
  or the application is running in the background." A scoreboard that must
  survive the screen locking needs a real clock source, not the component's.
- **Nothing is persisted.** Scores and the elapsed time live in component
  state, so backgrounding the app far enough to destroy the page loses the
  match.
- The period is a single ten-minute countdown - there are no quarters, no
  half-time and no overtime, and `reset()` returns to the same fixed count.
- Scores only increase. There is no correction for a mis-tap, which a live
  scorer needs.

## Pitfalls

- **`HW-03-0026` — the score buttons handle only `Down` and `Up`,** so a touch
  ending in `Cancel` - a finger sliding off, a scroll claiming the gesture -
  leaves the button highlighted permanently.
- **`HW-03-0027` — the countdown reaches zero unobserved.** No `onTimer`
  handler exists, so the end of the period produces no buzzer, no state change
  and no lockout of the scoring buttons, although the countdown is the
  document's first stated feature.
- **`HW-03-0028` — the documented tree says `entrybackupablility`;** the
  directory is `entrybackupability`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimer`, `isCountDown`, `count`, `format`, `onTimer`, `TextTimerController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `TouchEvent`, `TouchType.Down`, `Up`, `Cancel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInput`, `onChange`, `onSubmit`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-guides/03_application-framework/arkts-link.md` - `@Link`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- `SPORT-03` - the other timer in this industry, driving a drawing instead of a clock
- `SPORT-10` - the tournament bracket that consumes match results
