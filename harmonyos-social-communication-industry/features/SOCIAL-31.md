---
id: SOCIAL-31
title: Phone numbers inside a message - Natural Language Kit entity extraction into tappable spans
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/31_auto_phone_number_recognition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_phone_number_recognition-0000002317048484
sample: huawei_industry_tree/14_social_communication/downloads/PhoneNumberRecognize.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.NaturalLanguageKit", "@kit.PerformanceAnalysisKit"]
apis: ["textProcessing.getEntity", EntityType, "Entity.charOffset", "Entity.text", "call.hasVoiceCapability", "call.makeCall", startAbility, Want, Span, ImageSpan, onAreaChange, TextArea, TextAreaController, showToast, position]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0067, HW-14-0068, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when text a user typed or received **contains something
actionable** - a phone number, an address, a courier number, a verification
code - and it should become tappable without the user selecting it. The pattern
is: run `textProcessing.getEntity` over the string, split the string at the
entity boundaries, and rebuild it as a `Text` of `Span`s where the entity spans
carry their own styling and click handler.

The extractor covers ten entity types (`DATETIME`, `EMAIL`, `EXPRESS_NO`,
`FLIGHT_NO`, `LOCATION`, `NAME`, `PHONE_NO`, `URL`, `VERIFICATION_CODE`,
`ID_NO`), all on-device, so the same three lines that make a phone number
callable make a tracking number openable or a verification code
one-tap-copyable. Only the action menu changes. That is the generalisation
worth taking from this card - the segmentation and rendering half is identical
for every type.

**Read `HW-14-0067` before adopting it.** The sample's segmentation is
`String.split()` on the matched text, which is wrong in a way that a demo
message never exposes and a real chat exposes immediately - and the published
document ships the same code, so copying from either source reproduces the
defect.

## Feature checklist

- A full-screen text editor opens focused, ready to type.
- A done button in the header switches the page from editing to rendered mode.
- On done, phone numbers in the text are found and drawn blue and underlined;
  the rest of the text stays black.
- Tapping a number opens a small two-item menu positioned next to it.
- 呼叫 (call) checks that the device can make voice calls and raises the system
  dialler pre-filled with the number.
- 新建联系人 (new contact) starts the system Contacts app on its save-contact
  page with the number filled in.
- On a device with no telephony, 呼叫 shows a toast instead.
- Tapping anywhere else in the rendered text returns to editing with the full
  message intact.

## Architecture

One `entry` module, one page, one model class, one logger. The entire feature
is 246 lines of `TextEdit.ets`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets     full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/DataModel.ets               TextArr: data, isPhoneNumber, pox, poy
├── pages/TextEdit.ets                @Entry - editor, recognition, rendering, menu, actions
└── utils/Logger.ets                  hilog wrapper
```

The documented tree matches the zip exactly.

The state model is worth reading closely, because it is where the defect
lives. There are two source strings and one array:

- `originalText` - the string recognition runs over.
- `originalText1` - the string the `TextArea` is seeded with.
- `textArr: TextArr[]` - the rendered segments, each a chunk of text plus an
  `isPhoneNumber` flag plus a screen position.

`TextArr` carrying `pox`/`poy` is the interesting piece: the menu has to appear
next to the tapped number, and a `Span` has no geometry of its own.

**The design decision worth copying** is the zero-sized `ImageSpan` used as a
position probe. `Span` supports no layout callbacks, but `ImageSpan` does, so
the sample emits an `ImageSpan` of width 0 and height 0 immediately before each
phone-number `Span` and reads `onAreaChange` → `newValue.globalPosition` off
it. The probe is invisible, costs one node, and gives the exact baseline
coordinate where the number starts. That is a genuinely useful trick for
anchoring anything to inline text.

**The decision worth avoiding** is in the same file: `originalText` is both the
input to recognition and the scratch variable the segmentation loop consumes.
See `HW-14-0067`.

## Implementation steps

1. **Import from the kit, not the module**:
   `import { EntityType, textProcessing } from '@kit.NaturalLanguageKit';`
2. **Call `getEntity(text, { entityTypes: [EntityType.PHONE_NO] })`** inside a
   `try/catch` - it is async and it throws with `err.code` / `err.message`.
3. **Segment by `charOffset`, not by `split()`** (`HW-14-0067`). Every `Entity`
   carries `text` and `charOffset`; walk a cursor through the original string
   and slice between offsets.
4. **Never mutate the source string during segmentation** (`HW-14-0067`). Keep
   the message immutable so that returning to the editor and pressing done
   again re-runs recognition over the same text.
5. **Render as one `Text` of `Span`s**, not a `Row` of `Text`s, so the message
   wraps as a single paragraph. `wordBreak(WordBreak.BREAK_ALL)` is needed on
   both the editor and the rendered form to break long numeric runs.
6. **Emit a zero-sized `ImageSpan` before each entity span** and capture
   `newValue.globalPosition.x/y` in `onAreaChange` to anchor the menu.
7. **Clamp the menu's x against the container width** so a number at the right
   edge does not push the popup off screen; subtract the status-bar height from
   y because `globalPosition` is measured from the window, not the page.
8. **Use a length resource, not a string resource, for `borderRadius`**
   (`HW-14-0068`).
9. **Guard the call with `call.hasVoiceCapability()`** and fall back to a toast
   - tablets and 2in1 devices are in `deviceTypes` and have no telephony.

## Verified snippets

All snippets are from `PhoneNumberRecognize.zip`. Corrected forms are marked.

**Recognition and segmentation — `entry/src/main/ets/pages/TextEdit.ets`** (corrected, see `HW-14-0067`)

```typescript
.onClick(async () => {
  this.pop = false;
  try {
    let result = await textProcessing.getEntity(this.originalText,
      { entityTypes: [EntityType.PHONE_NO] }); // 实体抽取,设置实体的类别为手机号实体
    // FIX: walk the entity offsets over an unmutated source string.
    // The sample instead did `this.originalText.split(result[i].text)` and
    // reassigned `this.originalText = ms1[1]` on every pass.
    const source: string = this.originalText;
    const segments: TextArr[] = [];
    let cursor: number = 0;
    for (const entity of result) {
      const start: number = entity.charOffset;
      if (start > cursor) {
        segments.push(new TextArr(source.substring(cursor, start), false, 0, 0));
      }
      segments.push(new TextArr(entity.text, true, 0, 0));
      cursor = start + entity.text.length;
    }
    if (cursor < source.length) {
      segments.push(new TextArr(source.substring(cursor), false, 0, 0));
    }
    this.textArr = segments;
    this.isEditing = false;
  } catch (err) {
    Logger.error(`getEntity errorCode: ${err.code}, errorMessage: ${err.message}`);
  }
});
```

**`split()` is the wrong tool and `charOffset` is right there.** `split(needle)`
cuts at *every* occurrence, but the shipped loop consumes only `ms1[0]` and
`ms1[1]`: with the same number written twice - "call me on 138xxxx or
138xxxx" - the array has three parts, the third is dropped, and the next
iteration splits `undefined` into spans that literally render the word
`undefined`. Offsets have none of that ambiguity: each `Entity` says where it
starts, the gaps between entities are the plain-text runs, and repeated,
overlapping or adjacent matches all fall out correctly.

The second half of the fix is immutability. The shipped code overwrote
`this.originalText` with the remaining tail on each pass, so after one
successful recognition the state variable holds only the text after the last
number. Tap the rendered text to go back to editing, then press done without
typing anything - `onChange` never fires, so `originalText` is still the
truncated tail, and recognition runs over it. The front of the message is gone.
The corrected form reads `originalText` once into a local and never writes it.

Note also that the corrected version drops `this.textArr = []` at the top and
the `result.length !== 0` branch: building into a local array and assigning at
the end covers the empty-result case (`segments` becomes the whole string) and
leaves the previous render untouched if `getEntity` throws.

**Rendering with the position probe — same file** (as shipped)

```typescript
Text() {
  ForEach(this.textArr, (item: TextArr, index: number) => {
    if (item.isPhoneNumber) {
      ImageSpan($r('app.media.startIcon'))
        .width(0)
        .height(0)
        .onAreaChange((oldValue: Area, newValue: Area) => { // 获取ImageSpan的位置
          this.textArr[index].pox = newValue.globalPosition.x!;
          this.textArr[index].poy = newValue.globalPosition.y!;
        });
      Span(item.data)
        .fontColor(Color.Blue)
        .decoration({
          type: TextDecorationType.Underline,
          style: TextDecorationStyle.SOLID,
          color: Color.Blue
        })
        .onClick(() => {
          this.phoneNum = this.textArr[index].data;
          this.pop = true;
          this.cIndex = index;
        });
    } else {
      Span(item.data)
        .fontColor(Color.Black)
        .onClick(() => {
          this.isEditing = true;
        });
    }
  });
}
.wordBreak(WordBreak.BREAK_ALL);
```

**The zero-sized `ImageSpan` is a measuring device, not an image.** `Span` has
no `onAreaChange` - it is not an independent layout node - so there is no
direct way to learn where a run of inline text landed. `ImageSpan` is a layout
node that participates in the same text flow, so placing one with
`width(0).height(0)` immediately before the number gives its exact start
coordinate at no visual cost. `globalPosition` is relative to the window, which
is why the menu's `y` later subtracts `px2vp(topRectHeight)`.

Everything is inside a single `Text` container. That is what lets a long
message wrap normally with numbers embedded mid-line; a `Row` of separate
`Text`s would break the paragraph at every entity. The non-number spans carry
their own `onClick` returning to the editor, and the enclosing `Row` has a
second `onClick` for the gaps - guarded by `if (!this.pop)` so that the tap
which dismisses an open menu does not also reopen the editor.

**The anchored menu — same file** (corrected, see `HW-14-0068`)

```typescript
if (this.pop) {
  Column() {
    Row() {
      Text($r('app.string.call'))          // 呼叫
        .fontSize(14)
        .alignSelf(ItemAlign.Start);
    }
    .width($r('app.string.percent_one_hundred'))
    .padding({ left: 15 })
    .onClick(() => {
      let result: boolean = call.hasVoiceCapability(); // 判断设备是否具备语音通话能力
      if (result) {
        call.makeCall(this.phoneNum, (err: BusinessError) => { // 打电话
          if (!err) {
            Logger.info('make call success.');
          } else {
            Logger.error('make call fail, err is:' + JSON.stringify(err));
          }
        });
      } else {
        this.promptAction.showToast({ message: $r('app.string.reminder') });
      }
    });
    // ... 新建联系人 row
  }
  .backgroundColor(Color.White)
  .height(70)
  .width(110)
  .borderRadius(10)                        // FIX: sample had $r('app.string.EntryAbility_desc')
  .shadow(ShadowStyle.OUTER_DEFAULT_LG)
  .position({
    x: Number(this.textArr[this.cIndex].pox) > this.rowSize - 124 ? this.rowSize - 124 :
      this.textArr[this.cIndex].pox,
    y: Number(this.textArr[this.cIndex].poy) - this.context.px2vp(this.topRectHeight) + 5
  }) // 限制弹窗的位置
  .zIndex(1);
}
```

**The positioning arithmetic is the reusable part.** `x` is the probe's x
unless that would put the 110 vp menu past the container's right edge, in which
case it is pinned to `rowSize - 124` (`rowSize` comes from the container's own
`onAreaChange`). `y` converts from window coordinates to page coordinates by
subtracting the status-bar inset that the page's own padding already added.
`zIndex(1)` lifts the menu over the text, and `position` takes it out of the
`Column`'s flow so it does not push the layout around.

`.borderRadius($r('app.string.EntryAbility_desc'))` in the shipped code points
at a string resource whose value is the literal word `description`. It is a
valid `Resource` so the compiler accepts it, and an invalid `Length` so the
runtime silently drops the rounding - the menu ships with square corners
nobody intended. The project has no radius float resource; either add one to
`float.json` or use a literal, as above.

**Handing the number to the system — same file** (as shipped)

```typescript
.onClick(() => {
  let want: Want = {
    bundleName: 'com.ohos.contacts',
    abilityName: 'com.ohos.contacts.MainAbility',
    parameters: {
      'phoneNumber': this.phoneNum, // 需新增联系人电话
      'contactName': '',            // 需新增联系人姓名
      'pageFlag': 'page_flag_save_contact'
    }
  };
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  context.startAbility(want); // 拉起联系人并传入参数新建联系人
});
```

**Neither action needs a permission,** and that is the design. `call.makeCall`
raises the system dialler with the number pre-filled and lets the *user* press
call, so it needs no `ohos.permission.PLACE_CALL`; the explicit
`hasVoiceCapability()` check exists because this module also targets `tablet`
and `2in1`. The contact path is a plain explicit `Want` at the Contacts app -
the three `parameters` keys are the documented contract for its save-contact
page - so no contacts permission is needed either. Prefer both of these over
the permission-holding equivalents whenever a user confirmation step is
acceptable.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Entity extraction runs
on-device inside Natural Language Kit, the dialler is raised for user
confirmation, and Contacts is started through an explicit `Want`.

`deviceTypes` is `phone`, `tablet`, `2in1` - which is exactly why
`hasVoiceCapability()` is checked before `makeCall`. There is no `routerMap`;
`TextEdit` is the single `@Entry` page.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Entity extraction is not supported on the emulator.** Test on a device.
- Input is capped at **1,000 characters**, and supported languages are
  simplified Chinese, English and traditional Chinese.
- The Contacts `Want` is hardcoded to `com.ohos.contacts`; on a device without
  that bundle `startAbility` fails, and its result is not checked here.
- Only `PHONE_NO` is requested. Adding types to the `entityTypes` array is the
  whole change needed for URLs, emails or courier numbers - but `TextArr` has a
  single `isPhoneNumber` boolean, so a second type needs it turned into an enum.
- The menu is 110 x 70 vp with two fixed items; there is no copy action despite
  it being the obvious third.
- `pox`/`poy` are typed `Length` but hold numbers, hence the `Number(...)`
  casts in the position expression.

## Pitfalls

- **`HW-14-0067`** (B/medium, confirmed): segmentation uses
  `originalText.split(entity.text)` and consumes only `ms1[0]`/`ms1[1]`, so a
  repeated number drops the tail and propagates `undefined` into the rendered
  spans; the same pass overwrites `originalText` with the remainder, so
  done → tap → done loses the front of the message. The published document
  ships the identical code. Fix: segment by `entity.charOffset` over an
  immutable source string.
- **`HW-14-0068`** (B/low, confirmed): `.borderRadius($r('app.string.EntryAbility_desc'))`
  passes a string resource whose value is `description` where a `Length` is
  required, so the menu's rounded corners are silently dropped. Fix: point at a
  float/length resource (or a literal).

## References

- `documentation/harmonyos-guides/08_ai/natural-language-getentity.md` - the ten entity types, the emulator and 1,000-character constraints, and `Entity.charOffset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/natural-language-getentity
- `documentation/harmonyos-references/07_ai/natural-language-text-processing-api.md` - `textProcessing.getEntity`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/natural-language-text-processing-api
- `documentation/harmonyos-references/07_ai/natural-language-api.md` - Natural Language Kit overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/natural-language-api
- `documentation/harmonyos-references/03_system/js-apis-call.md` - `hasVoiceCapability`, `makeCall`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-call
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-uiabilitycontext.md` - `startAbility`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-want.md` - explicit `Want` and `parameters`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-want
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-span.md` - `Span`, `ImageSpan`, `decoration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-span
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-show-hide.md` - `onAreaChange` and `Area.globalPosition`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-show-hide
