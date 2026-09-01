---
id: SPORT-14
title: Scan a medicine barcode - the system scan UI feeding a half-modal sheet
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/14_scan_to_add_medication.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_to_add_medication-0000002396859981
sample: huawei_industry_tree/03_sports_health/downloads/ScanToAddMedication.zip
kits: ["@kit.ScanKit", "@kit.ArkUI", "@kit.LocalizationKit", "@kit.BasicServicesKit"]
apis: ["scanBarcode.startScanForResult", "scanBarcode.ScanOptions", "scanBarcode.ScanResult", "scanCore.ScanType", bindSheet, SheetSize, "UIContext.getHostContext", "resourceManager.getStringSync", "UIContext.getPromptAction", showToast, ForEach, List, "@State"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0051, HW-03-0052, HW-03-0053, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when the app should **let the user enter a product by scanning
it** - medicines here, and equally groceries for a food diary, equipment,
books, tickets, anything with a barcode.

The single most useful fact on this card: **the default scan UI needs no
camera permission.** The scan guide states it - "For the default UI of code
scanning, the system-level camera permission has been pre-granted and the
camera remains securely accessible when the UI is opened. It means that you do
not need to request the camera permission again." So `startScanForResult`
gives a full scanning experience - preview, gallery entry, torch prompt - with
an empty `requestPermissions`, and this sample correctly declares none.

The second idea is the **two-sheet flow**: an action sheet that offers the
scan, and a result sheet that shows what came back. Each is a `bindSheet` on a
boolean, and the handoff between them is two assignments.

## Feature checklist

- A "my medicines" list on the main page.
- An add button opening a half-modal sheet of entry methods.
- One of those methods launches the system barcode scanner.
- A successful scan closes the action sheet and opens a result sheet showing
  the matched product.
- Confirming adds the medicine to the list; the unmatched case and the
  unimplemented methods each produce a toast.

## Architecture

One `entry` module, one page, two model files.

```
entry/src/main/ets
├── common/Constants.ets                sizes, opacities, spacings
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── MedicationScanInfo.ets          codeStr, image, info - all ResourceStr
│   └── MockData.ets                    the stub catalogue + getMedicationInfo(code)
└── pages/MedicationAssistantPage.ets   the list, both sheets, the scan call
screenshots/
├── 37457656785686786.png               scannable test barcodes
└── 63567246789848537.png
```

The documented tree matches the zip. The note pointing at the test barcodes
spells the directory `/sceenshots`; it is `screenshots` (`HW-03-0053`).

**The sheet handoff is two booleans:**

```
add button ──> showAddMedicationSheet = true      (how do you want to add it?)
                        │
                   "scan" tapped ──> startScanForResult
                        │
                 result arrives ──> showAddMedicationSheet = false
                                    showRecognitionResultSheet = true
                        │
              "add to list" ──> medications.push(...) ; both sheets false
```

The two sheets are bound at different levels - the action sheet on the add
`Button`, the result sheet on the page's root `Column` - because the first is
dismissed while the second opens, and a sheet bound to a component that is
being hidden cannot open its successor.

**The catalogue is a stub with a real interface.** `getMedicationInfo(code, resourceManager)`
returns `MedicationScanInfo | undefined`, so the not-found path exists in the
signature and the page has to handle it - which is what a lookup against a
real product database would also return.

## Implementation steps

1. **Declare no permissions.** The default scan UI is pre-granted; asking for
   `ohos.permission.CAMERA` here is wrong.
2. **Bind the action sheet to the add button** and the result sheet to the
   page root.
3. **Launch the scanner with `startScanForResult`**, passing the host context.
4. **Take `result.originalValue`** as the barcode and look it up.
5. **Swap the sheets in the resolve handler**, and handle both the not-found
   case and the rejection.
6. **Identify controls by resource id, not by rendered text** (`HW-03-0051`).
7. **Key list rows on the entry, not on the product** (`HW-03-0052`).

## Verified snippets

All snippets are from `ScanToAddMedication.zip`. Corrected forms are marked.

**Launching the scanner — `entry/src/main/ets/pages/MedicationAssistantPage.ets`** (as shipped)

```typescript
import { scanBarcode, scanCore } from '@kit.ScanKit';

startDefaultScan() {
  const options: scanBarcode.ScanOptions = { /* scanTypes, enableMultiMode, enableAlbum */ };
  try {
    scanBarcode.startScanForResult(this.getUIContext().getHostContext(), options)
      .then((result: scanBarcode.ScanResult) => {
        this.showAddMedicationSheet = false;          // close the "how" sheet
        this.showRecognitionResultSheet = true;       // open the "what" sheet
        this.currentMedication = getMedicationInfo(result.originalValue, this.resourceManager);
        if (this.currentMedication === undefined) {
          this.showRecognitionResultSheet = false;    // nothing matched: fall back to a toast
          // ...
        }
      })
      .catch((error: BusinessError) => { /* logged */ });
  } catch (error) { /* logged */ }
}
```

**Both a `try` and a `catch` are needed.** `startScanForResult` can throw
synchronously on a bad context or bad options, and reject asynchronously when
the user backs out of the scanner or the scan fails - two different failures
that arrive through two different channels. The sample handles both, which is
more than most samples in this corpus manage.

**`result.originalValue` is the raw barcode string.** `ScanResult` also
carries the code type and the corner points; for a product lookup only the
value matters.

Passing `this.getUIContext().getHostContext()` rather than a stored context is
correct here - the scan UI is launched as an ability, so it needs the host
`UIAbilityContext`.

**The two sheets — same file** (as shipped)

```typescript
@State showAddMedicationSheet: boolean = false;
@State showRecognitionResultSheet: boolean = false;
@State medications: MedicationScanInfo[] = [];
@State currentMedication: MedicationScanInfo | undefined = undefined;

build() {
  Column() {
    // ... the list of added medications
    Button($r('app.string.add_medication'))
      .bindSheet($$this.showAddMedicationSheet, this.addMedicationSheetBuilder(), { /* ... */ })
  }
  .bindSheet($$this.showRecognitionResultSheet, this.recognitionResultSheetBuilder(), { /* ... */ })
}
```

**The `$$` two-way binding is required on both.** `bindSheet` writes the flag
back when the user swipes the sheet away, so a one-way binding would leave it
true and the sheet unable to reopen.

Binding the second sheet to the page root rather than to the button is the
detail that makes the handoff work: the action sheet is being dismissed at the
moment the result sheet opens, and a sheet whose anchor is disappearing cannot
present another.

**Looking the product up — `entry/src/main/ets/model/MockData.ets`** (as shipped)

```typescript
export function getMedicationInfo(code: string,
  resourceManager: resourceManager.ResourceManager | undefined): MedicationScanInfo | undefined {
  for (let i = 0; i < MEDICATION_INFO_MOCK_DATA.length; i++) {
    if (resourceManager?.getStringSync(MEDICATION_INFO_MOCK_DATA[i].codeStr.id) === code) {
      return MEDICATION_INFO_MOCK_DATA[i];
    }
  }
  return undefined;
}
```

**Returning `undefined` rather than a placeholder** is the right contract: an
unrecognised barcode is a real outcome, and the page's handling of it is what
a production lookup against a product database would also need.

The catalogue itself is `ResourceStr` throughout - `MedicationScanInfo` holds
`codeStr`, `image` and `info` as resources - so the demo data is fully
localisable, unlike the model in `SPORT-13`.

**Dispatching a sheet button — same file** (corrected, see `HW-03-0051`)

```typescript
.onClick(() => {
  // FIX: the sample compares getStringSync(text.id) with getStringSync($r('app.string.scan').id)
  if (text.id !== $r('app.string.scan').id) {
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.wait_to_complished'),
      offset: { dx: 0, dy: Constants.HEIGHT_NORMAL_2 }
    });
  } else {
    this.startDefaultScan();
  }
})
```

## Permissions & config

**None, and that is correct.** `module.json5` declares no
`requestPermissions`, because the default scan UI carries a pre-granted
system-level camera permission. Declaring `ohos.permission.CAMERA` here would
prompt the user for something the app does not need.

A **custom** scanning UI - one that renders its own preview stream - is a
different capability and does need the camera permission.

Test barcodes ship in the project's `screenshots/` directory
(`37457656785686786.png` and `63567246789848537.png`), so the scan path can be
exercised on a device by pointing it at the screen.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Requires a device with a camera**; the scan UI cannot be exercised on the
  emulator.
- The catalogue is two stubbed products in `MockData.ets`. Any other barcode
  returns `undefined` and the page falls back to a toast - there is no manual
  entry path and no network lookup.
- Only the scan entry method is implemented; the other buttons in the action
  sheet raise a not-yet-implemented toast.
- The list is component state, so the medication history does not survive the
  page.

## Pitfalls

- **`HW-03-0051` — the sheet buttons decide what to do by comparing their
  resolved display text.** Two buttons sharing a caption in any locale would
  cross their actions, and if `resourceManager` is undefined both sides of the
  comparison are `undefined`, so the equality holds regardless of which button
  was pressed. Both ids are already in hand.
- **`HW-03-0052` — list rows are keyed on the barcode,** so adding the same
  medicine twice - the ordinary case for a purchase history - produces two
  rows with the same key, and an unavailable resource manager makes every key
  `undefined`.
- **`HW-03-0053` — the document sends the reader to `/sceenshots`** for the two
  test barcodes; the directory is `screenshots`. Those two images are the only
  barcodes the stubbed catalogue recognises, so a reader who cannot find them
  cannot exercise the feature at all.

## References

- `documentation/harmonyos-guides/02_media/scan-scanbarcode.md` - the default scan UI and its pre-granted camera permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-scanbarcode
- `documentation/harmonyos-references/04_media/scan-scanbarcode-api.md` - `startScanForResult`, `ScanOptions`, `ScanResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-scanbarcode-api
- `documentation/harmonyos-guides/03_application-framework/arkts-sheet-page.md` - `bindSheet` and half-modal pages
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-sheet-page
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - the sheet options
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getStringSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - the key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `SPORT-09` - the other `bindSheet` sample in this industry, and the same `PhotoViewPicker`-style permission-free system UI pattern
