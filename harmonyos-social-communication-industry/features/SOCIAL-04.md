---
id: SOCIAL-04
title: Friend-match reveal - an irregular Grid with per-index transition effects, over a similarity score that is never used
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/04_preference_search.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/preference_search-0000002236253766
sample: huawei_industry_tree/14_social_communication/downloads/PreferenceSearch.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [Grid, GridLayoutOptions, onGetRectByIndex, irregularIndexes, GridItem, align, transition, TransitionEffect, asymmetric, combine, rotate, scale, translate, opacity, "UIContext.animateTo", PlayMode, setInterval, clearInterval, "@Watch", "@StorageProp", Navigation, NavDestination, NavPathStack, replacePathByName, pushPath, "NavDestinationMode.DIALOG", TipsDialog, CustomDialogController, "resourceManager.getStringValue", hilog]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0009, HW-14-0010, HW-14-0001, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card for a "searching…" reveal** - the screen that spends a few
seconds pretending to look for something and then flies results in one at a
time. Friend matching here, but the same shape covers finding nearby players,
matching a ride, pairing a device, or any wait you want to make feel like work
being done.

Two techniques are worth taking. The first is `GridLayoutOptions` with
`onGetRectByIndex`, which lets you hand-place items into a non-uniform grid by
index while still using `ForEach` - a big hero cell, a stack of half-width
cells, a wide footer cell, all in one container with no nested layout. The
second is per-index `TransitionEffect`: each avatar gets its own entrance
direction, so six items arriving on a 1 s timer read as a converging swarm
rather than a list appending.

**Read `HW-14-0009` before adopting the matching itself.** The sample computes
a similarity score for every candidate and then throws it away - the result is
always the input list in insertion order. The animation is the working part of
this sample; the algorithm is decoration.

## Feature checklist

- A match page with a title, an empty-state illustration, and two capsule
  buttons: 偏好 (preferences) and 开始匹配 (start matching).
- 偏好 opens a preference sheet as a dialog-mode `NavDestination` with eight
  interest tiles that toggle on tap; closing it writes the selection back into
  the user model.
- 开始匹配 with no preferences selected raises a `TipsDialog` instead.
- With preferences selected, a loading component appears: a pulsing icon and a
  匹配中 label whose trailing dots cycle `.` → `..` → `...`.
- One candidate avatar appears per second, each flying in from its own
  direction with its own rotate/scale/translate effect.
- After six candidates the timer stops, the loading component disappears, and
  the whole grid settles upward by 58 vp.
- Both buttons are inert while a match is running.

## Architecture

One `entry` module. Two routed pages plus one animation component; the entry
page exists only to redirect.

```
entry/src/main/ets
├── component/LoadingAnimationComponent.ets  pulsing icon + animated "匹配中..." dots
├── constant/CommonConstant.ets              every literal: durations, grid templates, opacities
├── entryability/EntryAbility.ets            window setup, avoid areas -> AppStorage
├── model/UserModel.ets                      UserData, PreferenceData, MYSELF, FRIENDS[6], ALL_PREFERENCE[8]
├── pages
│  ├── FriendMatchPage.ets                   the match screen: grid, transitions, the timer (313 lines)
│  ├── Index.ets                             @Entry, 29 lines: replacePathByName('FriendMatchPage')
│  └── PreferencesPage.ets                   the 8-tile preference sheet, NavDestinationMode.DIALOG
└── util/UserPreferenceUtil.ets              similarityCompute + produceArray
```

The documented tree matches the zip file-for-file, but its two page comments
are swapped: it labels `FriendMatchPage.ets` 主页面 (home page) and `Index.ets`
好友匹配页面 (friend match page), which is backwards - `Index.ets` is the
`@Entry` shell and `FriendMatchPage.ets` is the match screen. Minor, but this
industry has a documented pattern of trees that do not survive contact with the
zip (`HW-14-0001`).

**The design decision worth copying** is that `Index.ets` renders nothing:

```typescript
@Entry
@Component
struct Index {
  private mainPathStack: NavPathStack = new NavPathStack();

  aboutToAppear() {
    this.mainPathStack.replacePathByName('FriendMatchPage', {}, true);
  }

  build() {
    Navigation(this.mainPathStack)
      .hideNavBar(true)
      .hideTitleBar(true)
      .hideBackButton(true)
      .hideToolBar(true);
  }
}
```

`replacePathByName` rather than `pushPath` means the empty shell never enters
the back stack, so a back press from the match page exits the app instead of
landing on a blank `Navigation`. Every real screen is then a `NavDestination`,
which is what lets `PreferencesPage` be `NavDestinationMode.DIALOG` - a sheet
that overlays its parent, keeps the parent mounted, and dismisses with
`pathStack.pop()` - without a separate dialog mechanism.

The consequence is that the parent stays alive underneath, which is why
`PreferencesPage` can write straight into the shared `UserData` object in
`aboutToDisappear` and the match page simply sees the mutation.

## Implementation steps

1. **Make the entry a shell** that `replacePathByName`s onto the first real
   route, and hide every `Navigation` chrome affordance.
2. **Declare the sheet as `NavDestinationMode.DIALOG`** so the match page stays
   mounted behind it.
3. **Pass the user model as the route param** and read it in `onReady` from
   `ctx.pathInfo.param`; write the selection back in `aboutToDisappear`.
4. **Describe the irregular layout with `GridLayoutOptions`**: list every
   hand-placed index in `irregularIndexes` and return
   `[rowStart, columnStart, rowSpan, columnSpan]` from `onGetRectByIndex`.
5. **Give each `GridItem` its own `align()` and `margin()` by index** for the
   final pixel nudges, rather than trying to express them in the grid template.
6. **Attach `transition()` per index** and give the *outer* effect an
   `.animation({...})` - combined effects inherit it, but without one on the
   outer effect nothing animates at all.
7. **Also set a `transition()` on the Grid itself**, otherwise the children's
   exit animations do not play when the parent re-renders.
8. **Drive arrivals with `setInterval` and stop it from an `@Watch` on the
   index**, not from inside the interval body.
9. **Sort the candidates by the computed similarity before rendering** - the
   sample computes and discards it (`HW-14-0009`).
10. **Parenthesise the correlation denominator and centre by the mean**, not by
    the raw selection count (`HW-14-0010`).

## Verified snippets

All snippets are from `PreferenceSearch.zip`. Corrected forms are marked.

**The irregular grid - `entry/src/main/ets/pages/FriendMatchPage.ets`** (as shipped)

```typescript
gridLayoutOptions: GridLayoutOptions = {
  regularSize: [1, 1],
  irregularIndexes: [0, 1, 2, 3, 4, 5],
  onGetRectByIndex: (index: number) => {
    switch (index) {
      case 0: // 首项
        return [0, 0, 1, 2]; // [rowStart, columnStart, rowSpan, columnSpan]
      case 5: // 末项
        return [5, 0, 2, 2];
      default:
        const COL = (index + 1) % 2;
        const ROW = Math.floor((index - 1) / 2) * 2 + 1;
        return [ROW, COL, 2, 1]; // 中间项占两行一列
    }
  }
};

@Builder
userMatchGrid() {
  Grid(this.scroller, this.gridLayoutOptions) {
    ForEach(this.bestMatchUser, (bestUser: UserData, index: number) => {
      GridItem() {
        Image(bestUser.avatar)
          .width(CommonConstant.AVATAR_HEIGHT)
          .height(CommonConstant.AVATAR_HEIGHT)
          .margin(this.getItemMargin(index));
      }
      .align(this.judgeAlignment(index))
      .transition(this.getItemTransitionAnimation(index));
    }, (bestUser: UserData) => {
      return bestUser.userId;
    });
  }
  .columnsTemplate(CommonConstant.USER_GRID_COLUMNS_TEMPLATE)   // '1fr 1fr'
  .rowsTemplate(CommonConstant.USER_GRID_ROWS_TEMPLATE)         // 7 rows
  // 此处是确保各个头像的退场动画正常运作
  .transition(TransitionEffect.OPACITY.animation({ duration: CommonConstant.ONE_SECOND_DURATION, curve: Curve.Ease })
    .combine(TransitionEffect.translate({ y: this.markTranslateY })));
}
```

**`onGetRectByIndex` is a placement function, not a hint.** It returns an
absolute rectangle in grid coordinates, so the six avatars occupy a 7-row ×
2-column board by hand: index 0 spans both columns of row 0, index 5 spans both
columns of rows 5-6, and 1-4 each take one column across two rows, alternating
sides via `(index + 1) % 2`. Every index is listed in `irregularIndexes`;
`regularSize` only applies to indices that are not.

Note the layered positioning: the grid template places the *cells*,
`align(this.judgeAlignment(index))` places the avatar within its cell
(`End` for index 1, `Top` for index 2, `Center` otherwise), and
`getItemMargin(index)` applies the final negative-margin nudges (`{ left: -6 }`,
`{ right: -15, top: -9 }`). That is three mechanisms for one arrangement, and
it is why the layout is pinned to exactly six items - a seventh candidate would
fall through `default:` in all three switches.

The `transition()` on the `Grid` itself is not decorative. The comment says it
outright: without an animation on the parent, the children's exit animations do
not play when the parent re-renders. `markTranslateY` doubles as the settle: it
starts at 58 and is set to 0 by the watcher when matching completes, which
slides the whole board up as the loading component disappears.

**Per-index entrance effects - same file** (as shipped, trimmed)

```typescript
getItemTransitionAnimation(index: number): TransitionEffect {
  switch (index) {
    case 0:
      return TransitionEffect.asymmetric(TransitionEffect.rotate({ x: 1, angle: 360 })
        .animation({ duration: CommonConstant.ONE_SECOND_DURATION })
        .combine(TransitionEffect.scale({ x: 0.5, y: 0.5 })
          .animation({ duration: CommonConstant.ONE_SECOND_DURATION })),
        TransitionEffect.rotate({ y: 1, angle: -360 }).animation({ duration: CommonConstant.ONE_SECOND_DURATION })
          .combine(TransitionEffect.scale({ x: 1.5, y: 1.5 })
            .animation({ duration: CommonConstant.ONE_SECOND_DURATION })));
    case 1:
      return TransitionEffect.translate({ x: 58, y: -58, z: -58 })
        .animation({ duration: CommonConstant.ONE_SECOND_DURATION })
        .combine(TransitionEffect.opacity(0));
    case 2:
      return TransitionEffect.translate({ x: -58, y: -58, z: -58 })
        .animation({ duration: CommonConstant.ONE_SECOND_DURATION })
        .combine(TransitionEffect.opacity(0));
    default:
      return TransitionEffect.translate({ y: CommonConstant.TRANSLATE_Y })
        .animation({ duration: CommonConstant.ONE_SECOND_DURATION })
        .combine(TransitionEffect.opacity(0).animation({ duration: CommonConstant.ONE_SECOND_DURATION }));
  }
}
```

**`asymmetric` and `combine` are the two composition operators.**
`asymmetric(appear, disappear)` splits what is by default one symmetric effect
into two, so the first and last avatars spin in one direction on entry and the
other on exit. `combine` chains effects onto the same component - a rotate *and*
a scale, a translate *and* a fade.

The rule that catches people is where `.animation()` goes. The **outer** effect
must carry animation parameters or nothing animates; a `combine`d inner effect
inherits them unless it needs different ones. `case 1` relies on that -
`opacity(0)` has no `.animation()` and rides the translate's 1 s duration -
while the `default` branch spells it out on both. Both forms are correct; the
mistake is omitting it on the outermost effect.

The translate offsets are `±58` on x, y **and z**, matched to
`AVATAR_HEIGHT = 58`: each avatar starts exactly one avatar-width outside its
final cell, on the diagonal away from the centre, and slightly behind in depth.
That is what makes six independently-timed arrivals read as one converging
motion.

**The arrival timer - same file** (as shipped)

```typescript
getMatchUsers() {
  this.markTranslateY = CommonConstant.MARK_TRANSLATE_Y;
  this.bestMatchUser = [];
  this.waitMatchUsers = [];
  // 通过定时任务模拟寻找好友的过程
  this.userLoadTimerId = setInterval(() => {
    this.waitMatchUsers.push(FRIENDS[this.loadIndex]);
    this.loadIndex++;
    // 筛选好友
    this.bestMatchUser = UserPreferenceUtil.similarityCompute(this.myself, this.waitMatchUsers);
  }, CommonConstant.ONE_SECOND_DURATION);
}

loadIndexChange(changedPropertyName?: string) {
  if (changedPropertyName === 'loadIndex') {
    if (this.loadIndex === CommonConstant.MAX_PAGE_USER) {
      this.isLoading = false;
      this.markTranslateY = 0;
      clearInterval(this.userLoadTimerId);
      this.loadIndex = 0;
    }
  }
}
```

**The stop condition lives in an `@Watch`, not in the interval body.**
`@Watch('loadIndexChange') @State loadIndex: number` fires the handler on every
increment, and the handler owns `clearInterval`, the loading flag and the
`markTranslateY` settle. That keeps the timer callback to one job - produce the
next candidate - and puts every end-of-run side effect in one place.

`bestMatchUser` is reassigned to a **new array** each tick, which is what makes
the `@State` diff visible and lets `ForEach`'s `userId` key generator introduce
exactly one new child - hence exactly one entrance animation. Re-sorting the
existing array in place would not re-render.

Two rough edges: the ranking call inside the tick rescores every
previously-matched user from scratch each second (O(n²) over the run,
irrelevant at n=6), and `clearInterval` is only reachable through the watcher -
`aboutToDisappear` does not clear it.

**The similarity computation - `entry/src/main/ets/util/UserPreferenceUtil.ets`** (corrected, see `HW-14-0009`, `HW-14-0010`)

```typescript
public static similarityCompute(mainUser: UserData, otherUsers: UserData[]): UserData[] {
  if (otherUsers.length <= 1) {
    return otherUsers;
  }

  let collaborative: number[][] = [];
  let userSimArray: UserSimCalModel[] = [];
  collaborative.push(UserPreferenceUtil.produceArray(mainUser));
  for (let u of otherUsers) {
    collaborative.push(UserPreferenceUtil.produceArray(u));
    userSimArray.push(new UserSimCalModel(u));
  }

  // FIX: centre by the MEAN of the 0/1 vector, not by the raw selection count
  let mainMean: number = mainUser.preferences.size / ALL_PREFERENCE.length;

  for (let i = 1; i < collaborative.length; i++) {
    let numerator: number = 0;
    let denominator1: number = 0;
    let denominator2: number = 0;

    let otherMean: number = otherUsers[i - 1].preferences.size / ALL_PREFERENCE.length;   // FIX
    for (let j = 0; j < ALL_PREFERENCE.length; j++) {
      numerator += (collaborative[0][j] - mainMean) * (collaborative[i][j] - otherMean);
      denominator1 += Math.pow(collaborative[0][j] - mainMean, 2);
      denominator2 += Math.pow(collaborative[i][j] - otherMean, 2);
    }

    // FIX: parenthesise - the sample computes (n / sqrt(d1)) * sqrt(d2)
    let d: number = Math.sqrt(denominator1) * Math.sqrt(denominator2);
    userSimArray[i - 1].sim = d === 0 ? 0 : numerator / d;   // FIX: guard the degenerate all-or-none case
  }

  // FIX: absent in the sample - rank by the score that was just computed
  userSimArray.sort((a, b) => b.sim - a.sim);

  return userSimArray.map<UserData>((value) => {
    return value.userData;
  });
}

private static produceArray(user: UserData): number[] {
  let userPref: number[] = [];
  let userPreferences: Map<number, PreferenceData> = user.preferences;
  for (let pref of ALL_PREFERENCE) {
    if (userPreferences.has(pref.id)) {
      userPref.push(1);
    } else {
      userPref.push(0);
    }
  }
  return userPref;
}
```

**`produceArray` is the right idea and the rest is where it goes wrong.** Each
user becomes a fixed-length 0/1 vector over the eight `ALL_PREFERENCE` ids -
positional, so index `j` means the same interest for everyone, and comparable
between any two users regardless of how many each selected. That is the correct
encoding for a Pearson-style correlation.

Three corrections turn it back into one. **The sort is the important one**: the
shipped code assigns `sim` to every `UserSimCalModel` and then maps straight
back to `userData` in insertion order, so the return value is byte-for-byte the
input array. 根据偏好匹配好友 (match friends by preference) is the sample's
stated purpose and it has literally no effect - selecting every interest or one
interest produces the same six avatars in the same order (`HW-14-0009`).

The other two are arithmetic (`HW-14-0010`). `numerator / Math.sqrt(d1) *
Math.sqrt(d2)` parses left-to-right as `(numerator / sqrt(d1)) * sqrt(d2)`,
multiplying by the second denominator instead of dividing - the result is not
bounded to `[-1, 1]` and is not a correlation. And the centring term subtracts
`preferences.size` - a count between 0 and 8 - from vector elements that are 0
or 1; the mean of a 0/1 vector is `size / 8`, so a user who picked 5 interests
has every element centred to `-5`/`-4` instead of `0.375`/`-0.625`. Finally,
selecting **all** or **none** of the eight makes the centred vector all zeros
and the denominator 0, so the raw expression yields `NaN` - hence the
`d === 0` guard, without which the sort order is unspecified.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, which is correct -
the sample touches no sensitive resource. Worth stating because two other
samples in this industry (`18_save_draft_on_exit`, `34_drag_image_sort`) ship
dead `LOCATION` / `APPROXIMATELY_LOCATION` constants from the same project
template with no matching feature (`HW-14-0003`). This one is clean.

`deviceTypes` is `["phone", "tablet", "2in1"]`, but the layout is not
responsive: the grid is a fixed `70%` width by `58 * 7` vp height with
hand-placed cells and hardcoded 58 vp offsets.

`routerMap` is `$profile:route_map`, with `FriendMatchPage` and
`PreferencesPage` bound to their exported `@Builder` functions.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The layout is hardcoded to exactly six candidates.** `MAX_PAGE_USER`,
  `USER_GRID_ROWS_TEMPLATE` (7 rows), `onGetRectByIndex`, `judgeAlignment`,
  `getItemMargin` and `getItemTransitionAnimation` all enumerate indices 0-5.
  A different candidate count needs all six changed together.
- `FRIENDS` is six hardcoded `UserData` objects with empty preference maps, so
  **every candidate has an empty preference vector** in the shipped data - even
  with the ranking fixed, all six score identically until real preference data
  is supplied.
- `PreferencesPage` uses `selectedIds[index] === item.id` as its selected test
  and writes `-1` to deselect, which relies on `PreferenceData.id` equalling its
  array index in `ALL_PREFERENCE` - true today and undocumented. Its
  `aboutToDisappear` mutates the shared `MYSELF` object directly, and there is
  no persistence: the selection is lost on app restart.
- The 匹配中 dot animation wraps `this.text += '.'` in `animateTo`, but a text
  content change is not an animatable property - that `animateTo` is inert and
  only the interval matters. The icon's `opacity` animation with
  `iterations: -1, playMode: PlayMode.Alternate` is the real one.

## Pitfalls

- **`HW-14-0009`** (B/high, confirmed): `similarityCompute` assigns `sim` to
  every candidate and then returns `userSimArray.map(v => v.userData)` in
  insertion order - no sort, no filter anywhere in the project. The sample's
  headline feature has zero effect on the output. Fix: sort `userSimArray` by
  `sim` descending before mapping.
- **`HW-14-0010`** (B/medium, confirmed): the formula is wrong twice.
  `numerator / Math.sqrt(d1) * Math.sqrt(d2)` evaluates as
  `(n / sqrt(d1)) * sqrt(d2)` - the second denominator multiplies instead of
  divides; and the vectors are centred by `preferences.size` (0-8) rather than
  by the mean `size / ALL_PREFERENCE.length`. All-or-none selections give a zero
  denominator and produce `NaN`/`Infinity`. Fix: parenthesise, centre by the
  mean, guard the zero denominator.
- **`HW-14-0003`** (D/low, confirmed): the industry's copy-pasted permission
  template. This sample is clean, but sibling samples built from the same
  template carry unused declarations and dead `LOCATION` constants - check
  `module.json5` and any `REQUEST_PERMISSIONS` constant before reusing the
  scaffold.
- **`HW-14-0001`** (E/low, confirmed): documented trees in this industry are
  not reliably regenerated. Here the file list is right but the `Index.ets` and
  `FriendMatchPage.ets` comments are swapped; four other docs list files their
  zips do not contain.
- `clearInterval(this.userLoadTimerId)` is only reachable from the `@Watch`
  handler at `loadIndex === 6`. Navigating away mid-match (the 偏好 button is
  guarded by `isLoading`, but the system back gesture is not) leaves a 1 s timer
  running against a detached component and pushing into `waitMatchUsers`. Clear
  it in `aboutToDisappear` too.
- `similarityCompute` short-circuits on `otherUsers.length <= 1`, so the first
  candidate is never scored. Harmless while the score is unused; after fixing
  `HW-14-0009` it is the reason the first arrival can outrank a better match.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `GridLayoutOptions`, `irregularIndexes`, `onGetRectByIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-griditem.md` - `align` inside a grid cell
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-griditem
- `documentation/harmonyos-references/02_application-framework/ts-transition-animation-component.md` - `TransitionEffect`, `asymmetric`, `combine`, `animation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-transition-animation-component
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo`, `iterations`, `PlayMode.Alternate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `NavDestinationMode.DIALOG` and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `replacePathByName` and route maps
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `SOCIAL-01` - the industry index this card hangs off
