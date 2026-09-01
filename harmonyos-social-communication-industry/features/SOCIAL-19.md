---
id: SOCIAL-19
title: Group solitaire - a chain post edited on its own page and written back into the chat bubble by id
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/19_group_solitaire.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/group_solitaire-0000002320400581
sample: huawei_industry_tree/14_social_communication/downloads/GroupSolitaire.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, util, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0042, HW-14-0043, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card for **a structured message that several people edit in turn** -
群接龙 (group solitaire), where one person posts "who is coming on Saturday?"
and everyone appends their name to a numbered list that keeps living inside
the chat. Sign-up sheets, potlucks, shift swaps, RSVP threads: one bubble,
many authors, an ordered list that grows.

The mechanism is worth more than the feature. A chat bubble is a *reference*
to a record, not the record itself: the bubble carries an id, the record lives
in a store, and tapping the bubble opens a full-screen editor addressed by
that id. When the editor commits, it writes the record *and* refreshes the
bubble. That is the same shape as a poll inside a chat, a shared checklist, a
collaboratively edited location card - anything where the message is a view
onto mutable state.

The sample keeps the store as a plain in-memory array, so there is no
networking and no persistence to distract from the structure. Substituting a
real backend means replacing exactly one array.

## Feature checklist

- A chat page with a message list, a text input, a Send button, and a
  solitaire icon to the left of the input.
- Sending text appends an ordinary bubble.
- Tapping the solitaire icon creates a chain bubble and immediately opens the
  editor for it.
- The editor for a *new* chain shows an editable `#接龙` heading; for an
  *existing* chain it shows the heading as read-only text plus the initiator's
  avatar and caption.
- The list of entries is numbered; only the entries added in this visit are
  editable, earlier ones are plain text.
- A + button appends a new entry pre-filled with 用户 (user).
- Clearing an entry's text and blurring removes that entry.
- Send is enabled only when the list actually differs from what is stored.
  (**Not true in the sample** - see `HW-14-0043`.)
- Cancel, back, or leaving the page discards the edit and removes the bubble
  if the chain was new.
- After Send, the chat bubble renders the heading, the numbered entries, and a
  参与接龙 (join the solitaire) button.

## Architecture

One `entry` module. Two pages, one reusable bubble component, three data
classes, one array helper.

```
entry/src/main/ets
├── Classes
│  ├── Bubble.ets            @Observed chat item: bubbleId, type, user, content, solitaire, changeTag
│  ├── ConstantClass.ets     TypeConstant.TEXT = 1 / .SOLITAIRE = 2
│  └── SolitaireClass.ets    @Observed chain record: solitaireId, example, flags, solitaireArray
├── components/ChatText.ets  @Reusable bubble renderer, text and chain variants
├── entryability/EntryAbility.ets   full-screen window, KeyboardAvoidMode.RESIZE
├── entrybackupability/
├── pages
│  ├── ChatPage.ets          @Entry: Navigation host, message list, composer
│  └── SolitairePage.ets     NavDestination: the chain editor
└── util/ArrayUtil.ets       deepCopy + arrayComparison
```

The documented tree matches the zip. The doc's *code* snippet does not match
it: the doc writes `BubbleID` where the code has `bubbleId`, and
`this.showSolitaire` where the code has `this.usingSolitaire`. Copying the
doc's three-line excerpt verbatim will not compile against the sample.

**The design decision worth copying** is the split between two arrays that
both live on `ChatPage` and are published downward with `@Provide`:

```typescript
@Provide sqlArray: Array<SolitaireClass> = [];   // the store, keyed by solitaireId
@Provide rightData: Array<Bubble> = [];          // the chat transcript
```

`sqlArray` is the pretend database - one `SolitaireClass` per chain, holding
the authoritative entries. `rightData` is the transcript - one `Bubble` per
message, and a chain bubble holds *its own copy* of the chain for rendering.
The two are joined by `solitaireId`. That is why several bubbles can show the
same chain (the initiator's post and each participant's), and why the editor
can find the master record from any of them.

The editor adds a third copy: `usingSolitaire`, a `deepCopy` of the stored
entries that the user actually types into. Cancel is then free - throw the
copy away - and Send is a write of the copy into both the store and the
bubble. **Three copies is one more than most people would write, and each one
earns its place.**

## Implementation steps

1. **Give every bubble a UUID** with `util.generateRandomUUID(true)` at
   creation time, and route on it: `pushPathByName('solitairePage', bubbleId)`.
2. **Create the bubble *before* opening the editor,** with
   `solitaire.isNewBuild = true`. The transcript then already has a slot; the
   editor only has to fill it or remove it.
3. **Resolve the bubble in `onReady`** by comparing the route param against
   each `bubbleId`, then branch: new chain pushes a fresh `SolitaireClass`
   into the store, existing chain looks up the record by `solitaireId`.
4. **Work on a `deepCopy` of the stored entries,** never on the stored array
   itself.
5. **Do not pre-arm the Send button.** `isChange` must stay `false` until an
   edit really changes the array (`HW-14-0043`).
6. **Recompute `isChange` on blur** by comparing the working copy against the
   stored array.
7. **On Send, write both sides** - the bubble's copy and the store's record -
   then flip `changeTag` to force the reusable bubble to re-render.
8. **On `onHidden` without a Send, roll back**: pop the store entry if the
   chain was new, and pop the bubble.
9. **Fix `deepCopy`'s object branch before reusing it** - it calls
   `obj.length()` and throws on anything that is not an array
   (`HW-14-0042`).

## Verified snippets

All snippets are from `GroupSolitaire.zip`. Corrected forms are marked.

**Creating the chain and routing to it — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
Image($r('app.media.Solitaire'))
  .height(24)
  .onClick(() => {
    this.textController.stopEditing();
    // Create a new bubble class and redirect to the connector page
    let newSolitaire: Bubble = new Bubble();
    newSolitaire.type = TypeConstant.SOLITAIRE;
    newSolitaire.user = false;
    newSolitaire.bubbleId = util.generateRandomUUID(true);
    newSolitaire.solitaire = new SolitaireClass();
    // Create a new solitaire class
    newSolitaire.solitaire.isNewBuild = true;
    this.rightData.push(newSolitaire);
    this.navPathStack.pushPathByName('solitairePage', newSolitaire.bubbleId);
  });
```

**The bubble exists before the editor opens, and that is deliberate.** It is
pushed into `rightData` with `isNewBuild = true` and `isMessageSend` still
`false`, and `ChatText` renders nothing at all for a chain bubble whose
`isMessageSend` is false. So the transcript has a reserved, invisible slot
while the user is editing. Cancelling simply pops it; committing sets
`isMessageSend = true` and it appears. No placeholder flicker, no separate
"pending" collection.

`util.generateRandomUUID(true)` - the `true` asks for a cryptographically
secure source - is the only identity mechanism in the sample. Note that the
route carries the *bubble* id, not the chain id: the same chain can be reached
from several bubbles, so the editor needs to know which bubble it was opened
from in order to write back into the right one.

**Resolving the route and seeding the editor — `entry/src/main/ets/pages/SolitairePage.ets`** (corrected, see `HW-14-0043`)

```typescript
.onReady(() => {
  this.selectId = JSON.stringify(this.navPathStack.getParamByName('solitairePage')[0]);
  for (let index = 0; index < this.rightData.length; index++) {
    // Find the incoming bubble based on the selectId
    if (this.selectId === '"' + this.rightData[index].bubbleId + '"') {
      this.indexBubbleNumber = index;
      // Determine whether it is a new connector.
      if (this.rightData[index].solitaire.isNewBuild) {
        // If it is a new bubble, push it to the database.
        let solitaire: SolitaireClass = new SolitaireClass();
        solitaire.isNewBuild = true;
        solitaire.example = '接龙\n';
        solitaire.solitaireArray = [];
        solitaire.solitaireId = util.generateRandomUUID(true);
        this.indexSQLNumber = this.sqlArray.push(solitaire) - 1;
        this.usingSolitaire = deepCopy(this.sqlArray[this.sqlArray.length - 1].solitaireArray);
      } else {
        for (let index2 = 0; index2 < this.sqlArray.length; index2++) {
          // If it is not a new bubble, find the data from sqlArray.
          if (this.sqlArray[index2].solitaireId === this.rightData[index].solitaire.solitaireId) {
            this.usingSolitaire = deepCopy(this.sqlArray[index2].solitaireArray);
            this.exampleString = this.sqlArray[index2].example;
            this.indexSQLNumber = index2;
          }
        }
      }
    }
  }
  this.usingSolitaire.push('用户');
  // FIX: sample sets `this.isChange = true;` here, arming Send before any edit
  // FIX: sample also sets `this.inputStringId = this.usingSolitaire.length;` (off by one, unread)
})
```

**Two indices and a working copy are the whole editor state.**
`indexBubbleNumber` is where to write the display copy back;
`indexSQLNumber` is where to write the master record. Both are resolved once,
here, so every later handler is an array index rather than another search.

The string comparison is the ugly part worth understanding before you copy it:
`getParamByName` returns the parameter as an `Object`, `JSON.stringify` turns
the string `"abc"` into the five-character `"abc"` *with quotes*, and the
comparison then rebuilds the quotes on the other side with
`'"' + bubbleId + '"'`. It works, and it is fragile. `getParamByName(...)[0]
as string` compared directly against `bubbleId` says the same thing without
the quoting dance.

`usingSolitaire.push('用户')` gives the user a pre-filled row to type over -
good affordance. The line after it in the shipped code, `isChange = true`, is
what turns that affordance into a bug: Send lights up before the user has done
anything, and pressing it publishes a chain whose only entry is the literal
placeholder 用户. The greyed-out `fontColor(this.isChange ? Color.Black :
'#919294')` and the `if (this.isChange)` guard exist precisely to prevent
that, and are defeated by one line.

**Committing to both copies — same file** (as shipped)

```typescript
Text($r('app.string.Send'))
  .fontColor(this.isChange ? Color.Black : '#919294')
  .onClick(() => {
    if (this.isChange) {
      this.getUIContext().getFocusController().clearFocus();
      // Update rightData display data
      this.rightData[this.indexBubbleNumber].solitaire.solitaireArray = deepCopy(this.usingSolitaire);
      this.rightData[this.indexBubbleNumber].solitaire.example =
        this.exampleString.replace(/[\r\n]+$/, '');            // 清除末尾换行符
      this.rightData[this.indexBubbleNumber].solitaire.solitaireId =
        this.sqlArray[this.indexSQLNumber].solitaireId;
      this.rightData[this.indexBubbleNumber].solitaire.isNewBuild = false;
      this.rightData[this.indexBubbleNumber].solitaire.isMessageSend = true;
      // Update sqlArray display data
      this.sqlArray[this.indexSQLNumber].solitaireArray = deepCopy(this.usingSolitaire);
      this.sqlArray[this.indexSQLNumber].example = this.exampleString.replace(/[\r\n]+$/, '');
      this.sqlArray[this.indexSQLNumber].isNewBuild = false;
      this.exampleString = '#接龙\n';
      this.rightData[this.indexBubbleNumber].changeTag =
        !this.rightData[this.indexBubbleNumber].changeTag;
      this.isSend = true;
      this.addSolitaireNum = 1;
      this.navPathStack.pop();
    }
  });
```

**`changeTag` is the load-bearing line.** `ChatText` is `@Reusable` with an
`@ObjectLink bubble: Bubble`, so it re-renders when an *observed property of
the bubble itself* changes. Rewriting `bubble.solitaire.solitaireArray` mutates
a nested object; `@Observed`/`@ObjectLink` only tracks the first level, so the
list would not refresh. Toggling a boolean directly on the `Bubble` is the
cheap, explicit way to say "this bubble changed" - a manual invalidation
token. It is a workaround for a real limitation of nested observability, and
it is the pattern to reach for before restructuring your model.

`isSend = true` is the other half of the transaction: it tells the `onHidden`
handler that this exit was a commit, not an abandonment, so the rollback is
skipped.

```typescript
.onHidden(() => {
  this.controller.stopEditing();
  if (!this.isSend) {
    if (this.sqlArray[this.indexSQLNumber].isNewBuild) {
      this.sqlArray.pop();       // undo the record the editor speculatively created
    }
    this.indexSQLNumber = -1;
    this.indexBubbleNumber = -1;
    this.rightData.pop();        // undo the reserved bubble
  }
  this.exampleString = '#接龙\n';
  this.isSend = false;
})
```

`onHidden` rather than `aboutToDisappear` is right for a `NavDestination`:
a popped destination is hidden whether or not it is destroyed, so this is the
one callback that fires on Cancel, on the back gesture, and on the Send path
alike. Both rollbacks are `pop()` calls, which is only safe because the
speculative record and the speculative bubble are always the last elements -
true here, and the first thing that breaks if two editors are ever open at
once.

**The array helper — `entry/src/main/ets/util/ArrayUtil.ets`** (corrected, see `HW-14-0042`)

```typescript
export function deepCopy(obj: ESObject): ESObject {
  if (typeof obj !== 'object' || obj === null) {
    return obj;
  }
  let copy: ESObject;
  if (Array.isArray(obj)) {
    copy = [];
    for (let i = 0; i < obj.length; i++) {
      copy[i] = deepCopy(obj[i]);
    }
  } else {
    copy = {};
    for (let key of Object.keys(obj)) {     // FIX: sample loops `i < obj.length()`
      copy[key] = deepCopy(obj[key]);       //      and uses obj[i] as the key
    }
  }
  return copy;
}

/*
Determine whether the array has been modified
 */
export function arrayComparison(ary1: Array<string>, ary2: Array<string>): boolean {
  let isEqual: boolean = false;
  if (ary1.length === ary2.length) {
    for (let index = 0; index < ary1.length; index++) {
      if (ary1[index] !== ary2[index]) {
        isEqual = true;
      }
    }
  } else {
    isEqual = true;
  }
  return isEqual;
}
```

**The object branch of `deepCopy` has never run.** `obj.length()` is a method
call on a property that does not exist on plain objects, and the loop would
index `obj[0]`, `obj[1]`… as *keys* even if it did. The sample only ever
copies `Array<string>`, so the array branch carries every real call and the
defect stays latent - until someone reuses the helper for the `SolitaireClass`
it sits next to, at which point the advertised generic helper throws.
`Object.keys` is the two-line fix.

`arrayComparison` is shipped correct and named backwards: the local is called
`isEqual` and it is set to `true` when the arrays *differ*. The call site
reads `this.isChange = arrayComparison(stored, working)`, which is right.
Rename it `hasChanged` before the next reader trusts the local.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - the sample is
entirely local, with no network, no storage, and no media access.

`EntryAbility` sets two things that matter for a chat UI:

```typescript
const WIN = await windowStage.getMainWindow();
WIN.setWindowLayoutFullScreen(true);
windowStage.loadContent('pages/ChatPage', (err) => {
  let uiContext: UIContext = windowStage.getMainWindowSync().getUIContext();
  uiContext.setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
  // ...
});
```

`KeyboardAvoidMode.RESIZE` shrinks the page instead of sliding it up, which is
what keeps the message list scrollable while the keyboard is open. Unlike the
other samples in this industry, this one does *not* read avoid areas into
`AppStorage`; it uses fixed top margins (`margin({ top: 36 })`,
`margin({ top: 40 })`) instead, which is the weaker choice on devices with a
different status-bar height.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Nothing is persisted or transmitted.** `sqlArray` is an in-memory array on
  `ChatPage`; killing the app loses every chain. The class is named after a
  database it does not have.
- There is only one participant. `Bubble.user` exists to left/right-align
  bubbles and is set to `false` everywhere, so all messages render on one
  side; the 参与接龙 button simulates a second person by creating another
  bubble owned by the same user.
- The rollback path assumes the speculative record and bubble are the last
  elements of their arrays. Concurrent edits, or any message arriving while
  the editor is open, would make `pop()` remove the wrong thing.
- Only entries added during the current visit are editable - the guard is
  `index >= this.usingSolitaire.length - this.addSolitaireNum`. Earlier
  entries are rendered as `Text` and cannot be corrected.
- The heading is editable only while `isNewBuild` is true, so the initiator
  gets one chance to write the prompt and it is fixed thereafter.

## Pitfalls

- **`HW-14-0043` (B/low, confirmed) — `onReady` hardcodes `isChange = true`,**
  arming the Send control before any edit. Combined with the
  `usingSolitaire.push('用户')` on the line above, the user can immediately
  publish a chain whose only new entry is the literal placeholder. The greyed
  `fontColor` and the `if (this.isChange)` guard exist to prevent exactly
  this. Fix: drop the assignment and let the blur handler's
  `arrayComparison` decide. (The neighbouring `inputStringId =
  usingSolitaire.length` is off by one and never read.)
- **`HW-14-0042` (B/low, confirmed) — `deepCopy`'s object branch calls
  `obj.length()`,** which throws on any non-array object, and treats element
  values as keys. Latent today because the helper is only ever called with
  string arrays. Fix: iterate `Object.keys(obj)`.
- **Not filed:** the doc's snippet uses `BubbleID` and `showSolitaire`, names
  that do not exist in the shipped code (`bubbleId`, `usingSolitaire`). The
  three-line excerpt in 实现思路 will not compile against the sample.
- **Not filed:** `arrayComparison`'s local is named `isEqual` and holds the
  opposite. The function returns "has changed".
- **Not filed:** `ChatText` declares `@State rightData: Array<Bubble> = []`
  and pushes the participant's new bubble into it, but `ChatPage` passes its
  own `rightData` into that `@State` at construction. A `@State` initialised
  from a parent is a one-time copy of the reference here; the code works
  because arrays are shared by reference, not because the binding is correct.
  `@Link` or `@ObjectLink` is what was meant.

## References

- `huawei_industry_tree/14_social_communication/docs/19_group_solitaire.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/group_solitaire-0000002320400581
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `pushPathByName`, `getParamByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `onReady`, `onHidden`, `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - why nested mutations do not refresh, and what `changeTag` is working around
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-guides/03_application-framework/arkts-reusable.md` - `@Reusable` components and their re-render rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-reusable
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `generateRandomUUID`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List` / `ListItem` and `ForEach`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
