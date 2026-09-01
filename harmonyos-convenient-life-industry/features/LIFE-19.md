---
id: LIFE-19
title: Paste an address, split it into fields - PasteButton plus on-device entity extraction from Natural Language Kit
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
sample: huawei_industry_tree/02_convenient_life/downloads/AddressExtraction.zip
kits: ["@kit.NaturalLanguageKit", "@kit.BasicServicesKit", "@kit.IMEKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [PasteButton, "pasteboard.getSystemPasteboard", "systemPasteboard.getData", "PasteData.getPrimaryText", "textProcessing.getEntity", "textProcessing.Entity", EntityType, "entity.text", "entity.charOffset", "entity.type", "entity.jsonObject", RichEditor, RichEditorController, "controller.addTextSpan", "controller.deleteSpans", RichEditorOptions, TextInput, TextInputController, "inputMethod.getController", "controller.stopInputSession", onTouch, TouchEvent, TouchType, expandSafeArea, "util.format"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0134, HW-02-0135, HW-02-0136, HW-02-0137, HW-02-0138, HW-02-0139, HW-02-0140, HW-02-0141, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for the **paste-and-parse address form**: the user copies
"张三 13800000000 广东省深圳市南山区科技园南路 100 号" from a chat, taps one
button, and the recipient name, phone number, region and street land in separate
fields.

Two HarmonyOS features do the work, and both are worth knowing on their own:

- **`PasteButton`** is a *security component*. Tapping it grants the application
  one-shot access to the clipboard with no permission declaration and no user
  prompt - the tap **is** the consent. Reading the pasteboard any other way
  needs a permission this project does not have.
- **`textProcessing.getEntity`** is on-device entity extraction. You give it a
  string and a list of `EntityType`s and it returns typed spans with a
  structured `jsonObject` payload - for an address, the administrative
  decomposition.

The hard part is neither API: it is turning one `LOCATION` entity (or two) into
a region field and a street field. That classification is where the sample is
weakest (`HW-02-0137`, `HW-02-0138`).

Take this for delivery addresses, contact import, invoice details - anywhere a
user has structured data as one blob of text. For extraction from a photo rather
than text, see `LIFE-21`.

## Feature checklist

- A `RichEditor` showing the pasted raw text.
- A `PasteButton` that clears the form, reads the clipboard, echoes the text
  into the editor and runs extraction.
- Extraction fills four fields: recipient name, telephone, region, detailed
  address.
- The region and the street are separated even when the model returns them as
  one `LOCATION` span, by cutting at 区 or 县.
- Every field stays editable afterwards.
- A 清空 (clear) control resets the form and the editor.
- Tapping the background dismisses the keyboard.

## Architecture

One `entry` module, one page, one constants file.

```
entry/src/main/ets
├── constants/Constants.ets        the jsonObject key names and the two split characters
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets    (doc spells it "entrybackupablility" - HW-02-0140)
└── pages/Index.ets                THE CARD: the editor, the paste button, the form, the parser
```

**The data flow is one line of text in, five `@State` strings out:**

```
PasteButton.onClick
  clear addressName / phoneNumber / provinceAddress / detailedAddress / companyName
  controller.deleteSpans()                             wipe the editor
  systemPasteboard.getData() -> data.getPrimaryText()  the raw text
  controller.addTextSpan(text)                         echo it
  textProcessing.getEntity(text, { entityTypes: [NAME, PHONE_NO, LOCATION] })
    -> Entity[] -> formatEntityResult(...)             assigns the four fields
```

**Clearing before reading is not tidiness, it is correctness.** The parser only
ever *assigns* fields it finds; it never blanks one it does not. Without the
reset, pasting an address with no company name leaves the previous paste's
company on screen. The document's snippet omits that reset (`HW-02-0140`).

**The `LOCATION` classification** is the interesting logic. The model may return
the address as one span or as two (region, then street), and each span carries a
`jsonObject` describing which administrative levels it contains:

```
one span with province+city+district and a following LOCATION  -> it is the region
a span marked adornLocation, or the first span seen            -> cut it at 区 / 县
anything else                                                  -> it is the street
```

## Implementation steps

1. **Use `PasteButton`, not a plain button plus a pasteboard read.** The
   security component is what makes the clipboard access permissionless.
2. **Reset every field and the editor before reading,** so a second paste cannot
   inherit the first one's values.
3. **Ask for exactly the entity types you will use** - `NAME`, `PHONE_NO`,
   `LOCATION`. Each extra type is work the model does for nothing.
4. **Handle the rejection of both promises** (`HW-02-0136`). A `try` around a
   call that returns a promise catches nothing.
5. **Parse `entity.jsonObject` once per entity and read its fields,** rather
   than substring-searching the serialised form (`HW-02-0137`).
6. **Cut the region from the street at the first separator only,** using
   `indexOf`/`slice` rather than `split` (`HW-02-0138`).
7. **Keep parsing separate from applying** - a pure function that returns the
   four values, and a caller that writes them to state (`HW-02-0139`).
8. **Give each `TextInput` its own controller,** or none (`HW-02-0135`).
9. **Dismiss the keyboard from a background tap only,** testing
   `event.type === TouchType.Down`, not from an `onTouch` on the container that
   holds the fields (`HW-02-0134`).

## Verified snippets

All snippets are from `AddressExtraction.zip`. Corrected forms are marked.

**Paste and extract - `AddressExtraction.zip#entry/src/main/ets/pages/Index.ets:140`** (corrected, see `HW-02-0136` and `HW-02-0141`)

```typescript
PasteButton()
  .onClick(async () => {
    // reset first - the parser only assigns what it finds, never clears
    this.addressName = '';
    this.phoneNumber = '';
    this.provinceAddress = '';
    this.detailedAddress = '';
    this.companyName = '';
    this.controller.deleteSpans();

    const systemPasteboard = pasteboard.getSystemPasteboard();   // FIX: sample holds this at module scope
    await systemPasteboard.getData()
      .then((data) => {
        const text = data.getPrimaryText();
        this.inputText = text;
        if (this.inputText !== '') {
          this.controller.addTextSpan(this.inputText, { style: { fontSize: 14 } });
          textProcessing.getEntity(this.inputText,
            { entityTypes: [EntityType.NAME, EntityType.PHONE_NO, EntityType.LOCATION] })
            .then((result) => {
              if (result) {
                this.applyEntities(result);
              }
            })
            .catch((err: BusinessError) => {                     // FIX: sample has a try, which
              hilog.error(0x0000, TAG, 'getEntity failed: %{public}s', err.message);   // catches nothing
            });
        }
      })
      .catch((err: BusinessError) => {                           // FIX: sample has no catch here at all
        hilog.error(0x0000, TAG, 'getData failed: %{public}s', err.message);
      });
  });
```

**`PasteButton` is a security component, and that is the point.** The tap is the
user's grant, so the application reads the clipboard without declaring
`ohos.permission.READ_PASTEBOARD` and without a system dialog - which is why
this project's `module.json5` has no `requestPermissions` block at all. The
grant is scoped to that one tap; the same `getData()` called from an ordinary
button would fail.

**The `try` in the shipped code cannot work.** Its only statement is
`textProcessing.getEntity(...)`, which returns a promise immediately; a
rejection arrives later, outside the `try`. The `catch` fires only if `getEntity`
throws synchronously - which is not how it fails.

**`addTextSpan` after `deleteSpans`** is how the `RichEditor` is used as a
display surface rather than an input: the pasted text replaces whatever was
there, in one style.

**Classifying the entities - same file, line 409** (corrected, see `HW-02-0137`, `HW-02-0138` and `HW-02-0139`)

```typescript
// FIX: the sample calls this formatEntityResult, returns a debug string, and
//      assigns the page's @State from inside the loop. Split the two jobs.
applyEntities(entities: textProcessing.Entity[]): void {
  for (let i = 0; i < entities.length; i++) {
    const entity = entities[i];
    if (entity.type === EntityType.NAME) {
      this.addressName = entity.text;
    } else if (entity.type === EntityType.PHONE_NO) {
      this.phoneNumber = entity.text;
    } else if (entity.type === EntityType.LOCATION) {
      // FIX: the sample calls entity.jsonObject.toString().includes(...) eight times
      const attrs: Record<string, string> = JSON.parse(entity.jsonObject.toString());
      const isAdorn = attrs[Constants.ENTRY_ATTACH_ADORN_LOCATION] !== undefined;
      const isFullRegion = !isAdorn &&
        attrs[Constants.ENTRY_ATTACH_CORE_LOCATION] !== undefined &&
        attrs[Constants.ENTRY_ATTACH_CORE_PROVINCE] !== undefined &&
        attrs[Constants.ENTRY_ATTACH_CORE_CITY] !== undefined &&
        (attrs[Constants.ENTRY_ATTACH_CORE_DISTRICT] !== undefined ||
         attrs[Constants.ENTRY_ATTACH_CORE_COUNTY] !== undefined);

      if (isFullRegion && (i + 1) < entities.length && entities[i + 1].type === EntityType.LOCATION) {
        // a complete region span, with the street following as its own entity
        this.provinceAddress = entity.text;
      } else if (this.provinceAddress === '' || isAdorn) {
        // one span carrying both: cut at the first 区 or 县
        this.splitRegionFromStreet(entity.text);
      } else {
        this.detailedAddress = entity.text;
      }
    }
  }
}

splitRegionFromStreet(text: string): void {
  // FIX: the sample uses split() and takes [1], losing everything after a second separator
  for (const sep of [Constants.ENTRY_TEXT_SPLIT_ZONE, Constants.ENTRY_TEXT_SPLIT_COUNTY]) {
    const at = text.indexOf(sep);
    if (at >= 0) {
      this.provinceAddress = text.slice(0, at + 1);
      this.detailedAddress = text.slice(at + 1);
      return;
    }
  }
  this.detailedAddress = text;
}
```

**The lookahead is the key insight.** A `LOCATION` span that already contains
province + city + district is only the *region* if another `LOCATION` follows to
carry the street - otherwise it is the whole address and has to be cut. That is
what `(i + 1) < entities.length && entities[i + 1].type === EntityType.LOCATION`
tests, and it is the one part of the shipped classifier that is genuinely well
judged.

**`adornLocation` versus `coreLocation`** is the model's own distinction between
a decorative place reference and an administrative one; a span marked `adorn`
always needs cutting rather than being taken as a region.

**Substring-searching a serialised object cannot tell a key from a value.**
`includes('city')` matches a payload whose *value* contains "city" just as
happily as one with that key - ordinary for a romanised address. Parsing once
also removes eight `toString()` calls per entity.

**Splitting on 区 loses text.** `split('区')` on a district name that itself
contains 区 followed by a road name containing it yields three fragments and the
code keeps only the second. `indexOf` + `slice` cuts once and keeps the
remainder whole.

**Dismissing the keyboard - same file, line 49** (corrected, see `HW-02-0134`)

```typescript
Column() {
  this.topBar();
  this.textInput();
  this.addressPadding();
  Blank();
  this.bottomBar();
}
// FIX: the sample attaches this to the Column that CONTAINS the five TextInputs,
//      with no phase test - touch events bubble, so tapping a field ends the
//      input session the tap was opening, three times per tap.
.onTouch((event: TouchEvent) => {
  if (event.type === TouchType.Down) {
    inputMethod.getController().stopInputSession()
      .catch((err: BusinessError) => { hilog.error(0x0000, TAG, '%{public}s', err.message); });
  }
})
.width('100%')
.height('100%')
.expandSafeArea([SafeAreaType.SYSTEM]);
```

**Touch bubbling is what makes this wrong.** The reference is explicit: "Touch
events bubble by default and can be consumed by multiple components." Every tap
on a `TextInput` also reaches the ancestor's handler, and `onTouch` fires for
`Down`, `Move` and `Up` - so a single tap on a field calls
`stopInputSession()` three times. The reference also notes the API "can be
called only when the edit box is attached to the input method", which an
arbitrary background tap does not satisfy.

**`expandSafeArea([SafeAreaType.SYSTEM])`** with no edge argument expands into
both the status bar and the navigation bar - the declarative alternative to the
`AppStorage` inset dance used by most samples in this industry.

**The form fields - same file, line 203** (corrected, see `HW-02-0135`)

```typescript
Row() {
  Text($r('app.string.consignee_name')).fontSize(12);
  TextInput({ text: this.addressName })          // FIX: sample passes one shared
    .width('100%').height('100%')                //      textInputController to all five
    .backgroundColor(Color.Transparent)
    .onChange((value: string) => {
      this.addressName = value;                  // one-way in, explicit write-back out
    });
}
.height(40)
.backgroundColor('#0d000000')
.borderRadius('8');
```

**Each field is a one-way bind plus an explicit `onChange`,** not `$$`. That is
deliberate here: the parser writes the state, and the user's edits write it back
- `$$` would work too, but the explicit handler makes the two write paths
visible. Unlike `LIFE-15`, this sample does wire every field's write-back.

The shared `TextInputController` on all five fields is the same defect as
`LIFE-08`'s two-field version, at five.

## Permissions & config

**None** - and that is the notable part.
`AddressExtraction.zip#entry/src/main/module.json5` declares no
`requestPermissions` block at all:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

Reading the clipboard normally requires a permission; `PasteButton` replaces it
with the user's tap. Entity extraction runs on device, so it needs neither a
network permission nor a service to be enabled - unlike the Map Kit samples in
this industry.

Root `build-profile.json5` targets `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 93-95).
- The extraction is tuned for **Chinese** addresses: the region/street cut is
  the literal characters 区 and 县, and the `jsonObject` keys are the model's
  Chinese administrative levels.
- Only `NAME`, `PHONE_NO` and `LOCATION` are requested. A postcode or a company
  name in the pasted text is not extracted - the company field is never filled
  by the parser, only by hand.
- The parser assigns but never clears, so the caller must reset the form first.
- `getPrimaryText()` returns only text; an image or a URI on the clipboard
  yields an empty string and the extraction is skipped.
- The confirm and 邀请填写 buttons in the bottom bar have no handlers, and the
  back arrow and the save-address checkbox are decorative.
- A `LOCATION` span that the model marks `adornLocation` is always treated as
  needing a cut; there is no path that treats it as a pure region.

## Pitfalls

- **`HW-02-0134` - the keyboard-dismiss `onTouch` sits on the container that
  holds the fields.** Touch events bubble and `onTouch` fires for every phase, so
  tapping a field ends the input session three times. Test
  `event.type === TouchType.Down` and put the handler somewhere the fields are
  not inside.
- **`HW-02-0135` - one `TextInputController` is bound to all five fields.**
- **`HW-02-0136` - neither promise has a `catch`,** and the `try` around
  `getEntity` catches nothing because the call returns rather than throws. A
  failed paste leaves the form cleared, silently.
- **`HW-02-0137` - the classifier substring-searches
  `entity.jsonObject.toString()`** for key names, eight times per entity. Parse
  the payload once and read its fields.
- **`HW-02-0138` - `split('区')[1]` discards everything after a second
  occurrence.** Cut at the first separator with `indexOf` and `slice`.
- **`HW-02-0139` - `formatEntityResult` is typed as a formatter and is actually
  the function that fills the form;** its returned string is only logged.
- **`HW-02-0140` - the tree says `entrybackupablility`,** and the step-1 snippet
  omits the five-field reset and the `addTextSpan` that make a second paste
  correct.
- **`HW-02-0141` - `pasteboard.getSystemPasteboard()` runs at module load.**
- **Do not read the clipboard without `PasteButton`.** The security component is
  what removes the permission requirement; a plain button needs
  `ohos.permission.READ_PASTEBOARD`, which this project does not declare.
- **Do not skip the reset before a paste.** The parser is additive.
- **Do not assume one `LOCATION` entity.** The model may return the region and
  the street separately, which is exactly what the lookahead branch handles.

## References

- `documentation/harmonyos-references/02_application-framework/ts-security-components-pastebutton.md` - `PasteButton` and the permissionless one-shot clipboard grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-pastebutton
- `documentation/harmonyos-references/07_ai/natural-language-text-processing-api.md` - `textProcessing.getEntity`, `Entity` (`text`, `charOffset`, `type`, `jsonObject`), `EntityType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/natural-language-text-processing-api
- `documentation/harmonyos-references/03_system/js-apis-pasteboard.md` - `getSystemPasteboard`, `getData`, `PasteData.getPrimaryText`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pasteboard
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController.addTextSpan`, `deleteSpans`, `placeholder`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `onTouch`, event bubbling, `TouchType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod.md` - `getController().stopInputSession()` and its focus precondition
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` - `expandSafeArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `LIFE-21` - the same extraction problem starting from a photo instead of text
- `LIFE-22`, `LIFE-23` - the industry's card-scanning scenarios
- `LIFE-01` - the industry shell this page would sit in
