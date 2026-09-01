---
id: SOCIAL-35
title: Back-to-latest pill - announce a new message instead of yanking the list to the bottom
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/35_latest_message.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/latest_message-0000002384193145
sample: huawei_industry_tree/14_social_communication/downloads/LatestMessage.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: [List, ListScroller, scrollToIndex, scrollEdge, ScrollAlign, onDidScroll, backToTop, initialIndex, RichEditor, RichEditorController, "window.getLastWindow", keyboardHeightChange, setKeyboardAvoidMode, "resourceManager.getRawFileContentSync", "@Watch", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0075, HW-14-0027, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when **new content arrives at the end of a list the user may
have scrolled away from**. The rule the pattern encodes: if the user is at the
bottom, follow the content; if the user is reading history, do not move the
viewport - float a pill that says 新消息 (new message) and let them decide.

Chat is the canonical case, but the same decision appears in a live comment
feed, a log viewer, and a build console. What makes it a card rather than a
one-liner is that "am I at the bottom" is not a state ArkUI hands you, and every
implementation invents its own answer. **This sample's answer is wrong**
(`HW-14-0075`), which makes it a good place to learn what the right one looks
like: measure distance to the end, not accumulated scroll delta.

The layout half is worth copying as-is: the pill is a sibling of the chat column
inside a `Stack`, offset upward by the current keyboard height so it never hides
under the IME.

## Feature checklist

- A chat page seeded with 17 messages read from `resources/rawfile/data.json`.
- The list opens scrolled to the last message.
- Date separators (6月16日, 7月7日, 刚刚) are injected at fixed indices in the
  list, the last of which lands on the simulated new arrival.
- Five seconds after the page appears, a message from the peer is appended.
- If the user has scrolled away from the bottom, a white pill with a
  down-arrow and 新消息 appears on the right edge instead of the list moving.
- Tapping the pill scrolls to the last message and dismisses the pill.
- If the user is already at the bottom, the list follows the new message
  without a pill (documented; see `HW-14-0075` for why it does not happen).
- Opening the keyboard, focusing the editor, or sending a message all scroll to
  the end; the pill rides up with the keyboard.

## Architecture

One `entry` module, one page. Everything except the data loading is in
`ChatPage.ets`.

```
entry/src/main/ets
├── constants/Constants.ets     Y_OFFSET 360, Y_OFFSET_BIG 570, TIME_OUT 5000, three divider indices
├── entryability/EntryAbility.ets   avoid areas -> AppStorage
├── model/MsgContent.ets        { isSelf: boolean, content: string }
├── pages/ChatPage.ets          @Entry, 314 lines: title bar, list, pill, input box
└── utils
    ├── JsonUtil.ets            getRawFileContentSync('data.json') -> MsgContent[]
    ├── Logger.ets              hilog wrapper
    └── ResourceToStr.ets       getStringSync for the simulated message text
```

The documented tree matches the zip.

**The design decision worth copying** is the `Stack` at the top of `build()`:

```typescript
Stack() {
  Column() { this.titleBar(); ...this.messageList(); this.dialogBox(); }
  if (this.iconIsShow) { /* the pill */ }
}
```

The pill is not inside the list, not a dialog, and not a popup bound to a
component. It is the second child of a `Stack` that fills the page, so it floats
above the chat with no effect on layout and no scroll coupling, and it can be
mounted and unmounted by a plain `if`. Its vertical position is
`top: 560 - this.keyboardHeight`, driven by a `@Watch`-ed `keyboardHeight` state
that a window listener updates - so raising the IME lifts the pill by exactly
the height the list lost.

The choice worth avoiding is in the same file: the "am I near the bottom"
decision is made from a running sum of `onDidScroll` deltas
(`this.currentYOffset += scrollOffset`) compared against two magic pixel
constants. The list starts at the bottom, so the sum starts at 0 and only ever
goes negative as the user scrolls back through history. Both thresholds are
therefore decided at compile time.

## Implementation steps

1. **Load the seed data synchronously in `aboutToAppear`** with
   `resourceManager.getRawFileContentSync` plus a `util.TextDecoder` - a JSON
   rawfile is the cheapest stand-in for a message store.
2. **Open the list at the end** with `List({ initialIndex: this.data.length - 1,
   scroller: this.listScroller })`, not with a `scrollEdge` after mount.
3. **Track whether the end is visible,** not how far the user has scrolled. Use
   the last visible index from `onScrollIndex` (or the scroller's offset against
   the scrollable extent); a delta accumulator can never answer the question
   (`HW-14-0075`).
4. **Branch on arrival**: end visible → `scrollToIndex(last, true,
   ScrollAlign.END)`; end not visible → set the pill flag.
5. **Hide the pill when the user reaches the bottom themselves**, using the same
   predicate - otherwise the only way to dismiss it is to tap it.
6. **Put the pill in a `Stack`,** offset by the live keyboard height.
7. **Subscribe to `keyboardHeightChange` once and unsubscribe in
   `aboutToDisappear`,** and give `getLastWindow` a `.catch` (`HW-14-0027`).
8. **Use `setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE)`** so the IME shrinks
   the page instead of pushing it up - a chat needs its title bar to stay put.
9. **Clear the pending `setTimeout` in `aboutToDisappear`** so a page dismissed
   within five seconds does not push into a dead component.

## Verified snippets

All snippets are from `LatestMessage.zip`. Corrected forms are marked.

**The arrival decision — `entry/src/main/ets/pages/ChatPage.ets`** (corrected, see `HW-14-0075`)

```typescript
@State iconIsShow: boolean = false;
@State lastVisibleIndex: number = 0;          // FIX: replaces @State currentYOffset: number = 0

private isAtBottom(): boolean {
  return this.lastVisibleIndex >= this.data.length - 1;
}

aboutToAppear(): void {
  this.data = JsonUtil.getChatData(this.context.resourceManager);
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
  // Simulate receiving a new message after five seconds
  this.timeId = setTimeout(() => {
    let message: MsgContent = { isSelf: false, content: this.getNewMessage() };
    const wasAtBottom: boolean = this.isAtBottom();   // FIX: decide BEFORE the push
    this.data.push(message);
    Logger.info('push new message');
    if (!wasAtBottom) {                               // FIX: shipped test is currentYOffset < 360
      this.iconIsShow = true;
    } else {
      this.listScroller.scrollToIndex(this.data.length - 1, true, ScrollAlign.END);
    }
    clearTimeout(this.timeId);
  }, Constants.TIME_OUT);
}
```

**The predicate has to be evaluated before the array grows,** which is the
subtlety a rewrite has to keep: once the new message is pushed, the previously
last index is no longer the last index, and any "is the end visible" test
answers about the message that just arrived rather than about where the user
was.

The shipped test is `this.currentYOffset < Constants.Y_OFFSET`, i.e. `< 360`,
against a counter that starts at 0 and accumulates `onDidScroll` deltas. The
list is mounted at its bottom, so scrolling back through history produces
negative deltas and the sum only decreases: the condition is true at 0, true at
-1200, true always. Every arrival raises the pill, including for a user sitting
at the bottom watching the conversation - the exact case the document says gets
an automatic scroll instead.

**The dead auto-hide — same file** (corrected, see `HW-14-0075`)

```typescript
List({ initialIndex: this.data.length - 1, scroller: this.listScroller }) {
  ForEach(this.data, (item: MsgContent, index: number) => { /* bubbles + date dividers */ },
    (item: MsgContent, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`);
}
.backToTop(true)
.onScrollIndex((start: number, end: number) => {      // FIX: replaces the onDidScroll accumulator
  this.lastVisibleIndex = end;
  if (this.isAtBottom()) {
    this.iconIsShow = false;                          // reaching the bottom dismisses the pill
  }
})
.listDirection(Axis.Vertical)
.scrollBar(BarState.Off)
.friction(0.6)
.edgeEffect(EdgeEffect.Spring);
```

The shipped form of this handler is:

```typescript
.onDidScroll((scrollOffset: number) => {
  this.currentYOffset += scrollOffset;
  if (this.currentYOffset > Constants.Y_OFFSET_BIG) {   // > 570: unreachable
    this.iconIsShow = false;
  }
})
```

**Same accumulator, same mistake, opposite direction.** For the sum to exceed
570 the user would have to scroll *forward* past the end of a list that is
already at its end, so this branch never executes and the pill can only be
dismissed by tapping it. Switching to `onScrollIndex` gives the predicate a
meaning that does not depend on where the list happened to start: `end` is the
index of the last item currently rendered, and comparing it against
`data.length - 1` answers "is the newest message on screen" directly.

`backToTop(true)` is worth keeping - it opts the list into the system's
tap-the-status-bar-to-scroll-up gesture, which is the counterpart to this
feature.

**The floating pill — same file** (as shipped)

```typescript
if (this.iconIsShow) {
  Row() {
    Image($r('app.media.ic_move_down')).width(20).height(20).margin({ right: 12 });
    Text($r('app.string.new_message_reminder'))     // 新消息 - "new message"
      .fontColor('rgb(10, 89, 247)')
      .fontSize(17)
      .fontWeight(400);
  }
  .width(140)
  .height(40)
  .padding({ left: 16 })
  .margin({ left: 250, top: 560 - this.keyboardHeight })
  .backgroundColor('rgb(255, 255, 255)')
  .borderRadius({ topLeft: 20, bottomLeft: 20 })     // rounded on the left only: it hugs the edge
  .onClick(() => {
    this.listScroller.scrollToIndex(this.data.length - 1, true, ScrollAlign.END);
    this.iconIsShow = false;
  });
}
```

**`scrollToIndex(last, true, ScrollAlign.END)` is the whole payoff of the
feature**, and the three arguments each matter: the index of the newest message,
`smooth: true` so the jump is an animation the eye can follow rather than a
teleport, and `ScrollAlign.END` so the target settles at the *bottom* of the
viewport - `ScrollAlign.START` would put the newest message at the top with
empty space below it.

The pill is deliberately half-rounded (`topLeft` / `bottomLeft` only): it is
meant to sit flush against the right edge of the screen like a drawer tab.
`margin({ left: 250, top: 560 - keyboardHeight })` positions it with absolute
numbers, which is the one piece here that does not survive a different screen
size - on a tablet or a resized 2in1 window the pill lands in the middle of the
chat. An `.align(Alignment.BottomEnd)` on the `Stack` plus a bottom-relative
offset would be the portable form.

**The keyboard listener — same file** (corrected, see `HW-14-0027`)

```typescript
@State @Watch('onKeyboardChange') keyboardHeight: number = 0;
private keyboardCallback: (h: number) => void = (h: number) => {
  this.keyboardHeight = this.getUIContext().px2vp(h);
};
private currentWindow?: window.Window;

onKeyboardChange() {
  if (this.keyboardHeight !== 0) {
    this.listScroller.scrollEdge(Edge.End);
  }
}

aboutToAppear(): void {
  window.getLastWindow(this.context).then(currentWindow => {
    this.currentWindow = currentWindow;                       // FIX: keep the handle
    currentWindow.on('keyboardHeightChange', this.keyboardCallback);
  }).catch((err: BusinessError) => {                          // FIX: shipped promise has no catch
    Logger.error(`getLastWindow failed: ${err.code} ${err.message}`);
  });
}

aboutToDisappear(): void {
  this.currentWindow?.off('keyboardHeightChange', this.keyboardCallback);  // FIX: absent
  if (this.timeId !== undefined) {
    clearTimeout(this.timeId);
  }
}
```

**`@Watch` on the keyboard height turns one measurement into two behaviours.**
The height is consumed declaratively by the pill's `top` margin, and imperatively
by `onKeyboardChange`, which scrolls the list to the end whenever the IME opens
so the message being replied to stays visible. Because `keyboardHeight` is
stored in vp (`px2vp` on the raw px the event carries), it can be subtracted
directly from a vp margin.

The listener registration is the shipped code's other defect and it is not local
to this sample: `HW-14-0027` records the same missing `off` in
`H5Interception`'s rich editor and in `DropToSendImageAndText`, which registers
a fresh listener every time a chat opens. Each registration outlives its
component and keeps firing into a destroyed struct for the window's lifetime.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; the sample reads one
rawfile and one string resource and talks to nothing.

`window.getLastWindow(this.context)` is the only system API in play and needs no
permission. Resource directories are `base`, `en_US`, `zh_CN`; the message text,
the date separators and the pill label are all string resources, and the
simulated arrival is fetched through `getStringSync` rather than being a literal.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The "new message" is a single `setTimeout` fired once per page appearance.
  There is no socket and no repeat, so the pill can be observed exactly once per
  launch - keep that in mind when testing a fix.
- The date separators are hardcoded indices (`JUNE_SIXTEENTH = 0`,
  `JULY_SEVENTH = 6`, `JUST_NOW = 17`) matched against the `ForEach` index, not
  derived from any timestamp; `MsgContent` has no time field. `JUST_NOW = 17` is
  exactly the index the simulated arrival takes in the 17-message seed, which is
  why the 刚刚 separator appears with it.
- `Y_OFFSET` (360) and `Y_OFFSET_BIG` (570) are half-screen and
  above-half-screen distances in vp for one reference device. Even after the
  predicate is fixed, thresholds in absolute vp do not transfer across form
  factors.
- The pill's position is absolute (`left: 250`, `top: 560`) and assumes a phone
  in portrait.
- The `ForEach` key stringifies each message plus its index, so every re-render
  pays a `JSON.stringify` per row.

## Pitfalls

- **`HW-14-0075`** (B/medium, confirmed): both scroll thresholds are compared
  against `currentYOffset`, an accumulated `onDidScroll` delta that starts at 0
  on a list mounted at its end and can only go negative. `< 360` is therefore
  always true and `> 570` never true, so a user sitting at the bottom gets the
  pill instead of the documented auto-scroll, and the pill can only be dismissed
  by tapping it. Fix: derive the predicate from distance to the list end -
  `onScrollIndex`'s last visible index, or the scroller's offset against the
  scrollable extent.
- **`HW-14-0027`** (B/low, confirmed): systematic - the
  `window.on('keyboardHeightChange', ...)` listener is registered per component
  and never removed, and the `getLastWindow` promise carries no `.catch`.
  `LatestMessage` is one of three social samples with this exact shape. Fix:
  store the callback, call `currentWindow.off` in `aboutToDisappear`, and handle
  the rejection.

## References

- `huawei_industry_tree/14_social_communication/docs/35_latest_message.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/latest_message-0000002384193145
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `initialIndex`, `onScrollIndex`, `onDidScroll`, `backToTop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `scrollToIndex`, `scrollEdge`, `ScrollAlign`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController`, `getSpans`, `deleteSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getLastWindow`, `on`/`off('keyboardHeightChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `SOCIAL-12` - the systematic `keyboardHeightChange` finding (`HW-14-0027`) and its other instances
