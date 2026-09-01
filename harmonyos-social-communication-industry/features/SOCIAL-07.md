---
id: SOCIAL-07
title: In-chat vote card - a Radio group with a custom indicator, and Progress bars that grow when the ballot is cast
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/07_vote_result_display.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/vote_result_display-0000002274416885
sample: huawei_industry_tree/14_social_communication/downloads/VoteResultDisplay.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Radio, RadioIndicatorType, indicatorBuilder, Progress, ProgressType, ForEach, RelativeContainer, alignRules, "UIContext.getPromptAction", showToast, "@StorageProp", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0013, HW-14-0014, HW-14-0015, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card for a poll embedded in a conversation** - the "vote for the
星 of the month" card, a group deciding on a restaurant, a quick yes/no in a
team channel. The shape is: a header, one row per option carrying a radio
button and a bar, and a submit button that turns the ballot into a result
view.

The transferable mechanics are two. `Radio` with `group` gives mutual
exclusion for free across a `ForEach`, so you never write "uncheck the
others" logic; and `indicatorType: RadioIndicatorType.CUSTOM` with an
`indicatorBuilder` lets the checked state be any component you like - a brand
tick, an avatar, an emoji - without giving up the group behaviour.

The animation is the part people over-engineer. There is no `animateTo` here
and none is needed: `Progress` interpolates between its old and new `value`
by itself, so incrementing a candidate's tally *is* the animation. The same
trick covers a poll, a fundraising thermometer, a quiz score, a storage
meter.

**One caveat before adopting.** The sample's vote arithmetic is wrong in two
independent ways (`HW-14-0013`), and both are in the code a reader is most
likely to copy. Take the structure, rewrite the counting.

## Feature checklist

- A vote card headed 投票 with a banner block over a background image.
- Three meta columns: theme, description, and a deadline rendered as
  `yyyy-mm-dd`, ten days out from now.
- One row per candidate: the name on the left, a radio button on the right,
  and a linear `Progress` bar underneath showing their current tally.
- The radios form one group - selecting a second candidate clears the first.
- The checked indicator is a custom image, not the platform dot.
- Pressing 投票 with nothing selected toasts 请先选择投票对象 and does not
  submit.
- Pressing 投票 with a selection adds one vote to that candidate, animates
  their bar, toasts 投票成功, and swaps the page into result mode.
- In result mode the radios are replaced by a `N票` count per candidate, the
  header gains an icon, and the submit button disappears.

## Architecture

One `entry` module. A single page holds the whole feature; the model layer is
one interface and two helper functions.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model
│  ├── DateConversion.ets           getDatetime (timestamp -> yyyy-mm-dd) + modifyById (increment one tally)
│  └── ICandidateInfo.ets           { id, value, total, name, isCheck }
└── pages
   └── VoteDemo.ets                 @Entry, 319 lines: header, meta row, both candidate lists, submit
```

The documented tree matches the zip.

**The design decision worth copying** is the single `isVoting` boolean that
switches the whole card between ballot and result. `VoteDemo` renders two
sibling `ForEach` blocks over the *same* `specificCandidateList`, one with a
`Radio` per row and one with a `N票` label per row, and the button that flips
`isVoting` is itself inside an `if (this.isVoting)`. Voting and results are
therefore never two pages, never a dialog, and never a navigation - which is
what makes the card droppable into a chat bubble.

**The decision worth avoiding** is putting the vote arithmetic in the radio's
`onClick`. Selecting is not voting: the user is allowed to change their mind
before pressing submit, and the sample charges them a vote for each change
(`HW-14-0013`). Keep selection in `selectedCandidateIndex` and let only the
submit handler touch a tally.

The layout uses `RelativeContainer` with three named children (`Column1`,
`Column2`, `row1`) anchored to each other by `alignRules`. That is the right
container for "header, list, button pinned to the bottom" where the list's
height is unknown - the button anchors to `__container__`'s bottom while the
list anchors to the header's bottom.

## Implementation steps

1. **Model a candidate as `{ id, value, total, name, isCheck }`.** `value` is
   that candidate's tally, `total` is the denominator the bar is drawn
   against.
2. **Render the rows with `ForEach` over the candidate list**, taking both
   `candidate` and `candidateIndex` - the index is what the radio's `value`
   and the group bookkeeping need.
3. **Give every `Radio` the same `group` and a unique `value`.**
   `` value: `Radio${candidateIndex}` `` and `group: 'radioGroup'` is enough;
   the framework enforces single selection.
4. **Set `indicatorType: RadioIndicatorType.CUSTOM` and pass an
   `indicatorBuilder`** that calls a `@Builder` method on the struct. Wrap it
   in an arrow function so `this` still resolves.
5. **In `onClick`, record the selection only.** Set
   `selectedCandidateIndex` and update the `isCheck` flags - do not touch any
   tally here (`HW-14-0013`).
6. **Drive `Progress` from the candidate's own fields**, `value:
   candidate.value` and `total: candidate.total`, so a tally change animates
   the bar with no explicit animation.
7. **Guard the submit on `selectedCandidateIndex === -1`** and toast rather
   than submitting.
8. **On a real submit, increment both the candidate's `value` and every
   candidate's `total`,** then flip `isVoting` to `false` (`HW-14-0013`).
9. **Wrap every resource name in `$r()`.** Six attributes in this file take a
   bare `'app.string.…'` string and are silently ignored (`HW-14-0014`).
10. **Ignore the document's `this.dialogControllerList.open()` line** - no
    such dialog exists in the project (`HW-14-0015`).

## Verified snippets

All snippets are from `VoteResultDisplay.zip`. Corrected forms are marked.

**The radio group — `entry/src/main/ets/pages/VoteDemo.ets`** (corrected, see `HW-14-0013`)

```typescript
@Builder
indicatorBuilder() {
  Image($r('app.media.wy81'));
}

// ...
ForEach(this.specificCandidateList, (candidate: ICandidateInfo, candidateIndex: number) => {
  Column() {
    Row() {
      Text(candidate.name);
      Radio({
        value: `Radio${candidateIndex}`,
        group: 'radioGroup',
        indicatorType: RadioIndicatorType.CUSTOM,
        indicatorBuilder: () => {
          // 自定义单选框样式
          this.indicatorBuilder();
        }
      })
        .checked(this.specificCandidateList[candidateIndex].isCheck)
        .onChange((isChecked: boolean) => {
          this.specificCandidateList[candidateIndex].isCheck = isChecked;
        })
        .onClick(() => {
          this.selectedCandidateIndex = candidateIndex;
          this.specificCandidateList.forEach((item, index) => {
            item.isCheck = index === candidateIndex;
          });
          // FIX: the sample does `this.total++` here - selection is not a vote
        })
        .margin(0);
    }
    .justifyContent(FlexAlign.SpaceBetween)
    .width($r('app.string.fullWidth'));
```

**`group` is what makes this a poll rather than four checkboxes.** Every
`Radio` in the loop joins `'radioGroup'`, so the framework unchecks the
previous one when a new one is picked; `value` only has to be unique within
that group, which is why the index is enough. Because `checked()` is bound to
the model's `isCheck`, the visual state survives a re-render - and because the
`onClick` handler also rewrites every `isCheck`, the model and the group agree
even when the framework's own uncheck fires in a different order.

`indicatorType: RadioIndicatorType.CUSTOM` swaps the platform dot for whatever
`indicatorBuilder` draws - here `$r('app.media.wy81')`. The builder is passed
as an arrow function that calls the struct's `@Builder` method rather than
being passed directly; that indirection is what keeps `this` bound to the
component.

The removed line is the defect. `this.total++` in `onClick` charges a vote for
every *selection*, so flipping between two candidates three times adds six
votes before anyone has pressed submit (`HW-14-0013`).

**The bar and the denominator — same file** (corrected, see `HW-14-0013`)

```typescript
Row() {
  Progress({
    value: this.specificCandidateList[candidateIndex].value,
    total: candidate.total,
    type: ProgressType.Linear
  })
    .color('rgba(10, 89, 247, 1)')
    .backgroundColor('rgba(10, 89, 247, 0.15)')
    .style({ strokeWidth: 6, enableScanEffect: false })
    .width($r('app.string.fullWidth'));   // FIX: the header columns above pass this as a bare string
}
.width($r('app.string.fullWidth'));

// model/DateConversion.ets — corrected: the denominator must move with the tally
export function modifyById(id: number, arr: ICandidateInfo[]): void {
  if (id >= 0 && id < arr.length) {
    arr[id].value++;
    arr.forEach((candidate: ICandidateInfo) => {
      candidate.total++;                  // FIX: absent - every bar is drawn against a frozen 20
    });
  }
}
```

**`Progress` animates itself.** Changing `value` on a `Progress` transitions
the fill rather than jumping, which is the entire "进度条动效" the document's
title promises - there is no `animateTo` in this project. `strokeWidth: 6`
sets the bar thickness and `enableScanEffect: false` turns off the sweeping
highlight, which would read as "loading" rather than "result" on a poll.
`color` and `backgroundColor` are the same blue at full and 15% alpha, so the
unfilled remainder reads as the same quantity, not as a separate track.

The denominator is the bug. Each candidate's `total` is snapshotted from
`this.total` when the array literal is constructed, so it is `20` forever
while `this.total` drifts upward on selection clicks. The bars therefore never
change proportion relative to a growing electorate: casting the 21st vote
still draws against 20. Incrementing every candidate's `total` inside
`modifyById` keeps the model self-consistent with a single edit
(`HW-14-0013`).

**The header columns — same file** (corrected, see `HW-14-0014`)

```typescript
Column({ space: 6 }) {
  Text($r('app.string.themeStr'))
    .fontSize(18)
    .fontColor('rgba(0, 0, 0, 1)')
    .width($r('app.string.fullWidth'))        // FIX: shipped as .width('app.string.fullWidth')
    .fontWeight($r('app.float.fontWeight'))   // FIX: shipped as .fontWeight('app.float.fontWeight')
    .textAlign(TextAlign.Center);
  Text($r('app.string.voteResStr'))
    .fontSize(16)
    .fontWeight(400)
    .fontColor('rgba(0, 0, 0, 0.5)')
    .width($r('app.string.fullWidth'))        // FIX: same
    .textAlign(TextAlign.Center);
};
```

**A bare `'app.string.fullWidth'` is not a resource, it is a length string.**
ArkUI accepts `string` for `width`, so the compiler is happy and the runtime
tries to parse `app.string.fullWidth` as a dimension, fails, and drops the
setting. The column then sizes to its content instead of the full width, and
`textAlign(TextAlign.Center)` centres text inside a box that is already
exactly as wide as the text - so the three meta columns lose their centring.
`fontWeight` behaves the same way. Six attributes across lines 121-160 have
this defect, in a file that uses `$r()` correctly a dozen times elsewhere,
which is the signature of a hand edit (`HW-14-0014`).

**The submit button — same file** (as shipped)

```typescript
Button($r('app.string.voteString'))
  .height(44)
  .onClick(() => {
    if (this.selectedCandidateIndex === -1) {
      // 未选择投票对象
      this.getUIContext().getPromptAction().showToast({
        message: '请先选择投票对象',
        duration: 1000
      });
      return;
    } else {
      // 已选择投票对象
      this.isVoting = false;
      modifyById(this.selectedCandidateIndex, this.specificCandidateList);
      this.getUIContext().getPromptAction().showToast({
        message: '投票成功',
        duration: 1000
      });
    }
  })
```

**This is the whole state machine.** `selectedCandidateIndex === -1` is the
"nothing picked" sentinel and the only validation; `modifyById` applies the
ballot; `isVoting = false` swaps the card into result mode, which
simultaneously removes this button (it lives inside `if (this.isVoting)`) and
replaces the radio rows with vote counts. Nothing re-locks the ballot beyond
that, because the button no longer exists to be pressed.

**The document adds a line that is not here.** Its step-3 snippet calls
`this.dialogControllerList.open()` between `modifyById` and the toast. There
is no dialog controller, no `CustomDialog` and no dialog of any kind anywhere
in the project - the result is shown by re-rendering the card, and a reader
who copies the document gets an undefined-property crash on their first
successful vote (`HW-14-0015`).

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Everything is local
state; there is no network, no storage and no media access.

`deviceTypes` is `["phone", "tablet", "2in1"]`. Avoid areas are consumed with
`@StorageProp` and converted at the point of use:

```typescript
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;
// ...
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

This is the correct form - the ability writes px into `AppStorage`, the page
converts with `px2vp` where it is applied, and a missing key falls back to a
typed `0` rather than `undefined`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **There is no backend and no persistence.** Four candidates with seeded
  tallies of 3/7/4/3 are hardcoded in `VoteDemo`; a relaunch resets
  everything, and nothing prevents the same device voting again after
  restart.
- The deadline is `getDatetime(Date.now() + 864000000)` - always exactly ten
  days from now, recomputed on every render. It is decoration, not a rule:
  nothing checks it before accepting a vote.
- `RadioIndicatorType.CUSTOM` replaces the checked indicator only; the
  unchecked state is still the platform's, so a custom design needs to look
  right against the default empty circle.
- **Unfiled observation - the two `ForEach` blocks have no key generator.**
  Both fall back to ArkUI's default key, which is acceptable for a fixed
  four-element array but will not survive a candidate list that can be
  reordered or appended to.
- The `id` field on `ICandidateInfo` is never read - `modifyById` takes an
  array *index*, not an id, despite the name.

## Pitfalls

- **`HW-14-0013` (B/low, confirmed) — the vote total is incremented on
  selection and never reaches the denominator.** `VoteDemo.ets:195-201` does
  `this.total++` inside the radio's `onClick`, so merely switching between
  candidates inflates it; meanwhile each candidate's `total` was snapshotted
  at construction (`:30-59`) and `Progress` reads `candidate.total`, so the
  denominator stays 20 forever. Fix: move the increment to the vote action and
  update `candidate.total` there.
- **`HW-14-0014` (B/low, confirmed) — six attributes receive raw
  resource-name strings instead of `$r()`.** `VoteDemo.ets:121-160` has
  `.width('app.string.fullWidth')` and `.fontWeight('app.float.fontWeight')`
  as literal strings; the header columns silently lose their full-width and
  centred layout. Fix: wrap them in `$r()`.
- **`HW-14-0015` (D/low, confirmed) — the document calls a dialog the sample
  does not have.** Step 3's snippet inserts
  `this.dialogControllerList.open()` into the submit handler; the shipped code
  (`VoteDemo.ets:280-297`) shows a toast and re-renders. Fix the snippet, or
  readers will hunt for a result dialog that was never built.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-radio.md` - `Radio`, `group`, `RadioIndicatorType` and `indicatorBuilder`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-radio
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress`, `ProgressType.Linear`, `strokeWidth` and `enableScanEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-container-relativecontainer.md` - `alignRules` and anchoring to `__container__`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-relativecontainer
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and its default key behaviour
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
