---
id: LIFE-23
title: Real-name registration with the CardRecognition system control - pick a document type, scan both sides, confirm
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
sample: huawei_industry_tree/02_convenient_life/downloads/卡证信息识别示例代码.zip
kits: ["@kit.VisionKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit", "@kit.AbilityKit"]
apis: [CardRecognition, CardRecognitionResult, CardType, CardSide, ShootingMode, canIUse, Navigation, NavPathStack, NavDestination, "pageStack.pushPathByName", "pageStack.pop", NavDestinationContext, TextPicker, bindSheet, Provide, Consume, StorageLink, "AppStorage.setOrCreate", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "AvoidAreaType.TYPE_SYSTEM", "AvoidAreaType.TYPE_NAVIGATION_INDICATOR", safeAreaPadding, "UIContext.px2vp", "PromptAction.showToast", RelativeContainer, Extend]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0171, HW-02-0172, HW-02-0173, HW-02-0174, HW-02-0175, HW-02-0176, HW-02-0177, HW-02-0178, HW-02-0179, HW-02-0180, HW-02-0181, HW-02-0182, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when the user has to prove who they are with an official
document - an ID card, a passport, a residence permit - and you want the whole
capture-and-read step handed to the system.

`CardRecognition` is a **full-page ArkUI control from Vision Kit**, not an API
you call. You place it in the tree, tell it which document type to expect, and
it owns the camera, the shooting guidance, the optional gallery pick, and the
extraction. It hands back a structured `CardRecognitionResult`:

```
MainPage                    TextPicker -> a document type
   pushPathByName ------->  CardRecognitionPage
                              CardRecognition({ supportType, cardSide, config, onResult })
                                 the control runs the camera and the OCR
                              onResult -> params.cardInfo.front / .back
                                 { name, idNumber, cardImageUri, originalImageUri, ... }
                              CardData  -> the user confirms
   <------- pop(result)     MainPage shows the verified state
```

**No permission is declared anywhere in this project.** `module.json5` has no
`requestPermissions` block at all, because the control is a system UI that owns
the camera and the gallery on the application's behalf. That is the reason to
reach for it.

**Take a different card if you need the pixels.** `LIFE-22` builds a custom
camera with dual-channel preview and runs OCR itself - it needs
`ohos.permission.CAMERA` and the whole session lifecycle. `LIFE-21` does OCR on
a picked still image. This card gives up all control over the capture UI in
exchange for writing almost no code.

`LIFE-01`'s reference architecture contains the other `CardRecognition` call
site in this industry, and it is the one that uses the current parameter name -
see pitfall 1.

## Feature checklist

- [ ] A document type chosen before the scan page can be opened.
- [ ] `canIUse('SystemCapability.AI.Component.CardRecognition')` around the
      control, **with a fallback branch** (HW-02-0177).
- [ ] The result delivered through **`onResult`**, not the deprecated
      `callback` (HW-02-0171).
- [ ] `params.code` checked, and the failure branch **returns** (HW-02-0178).
- [ ] The confirmation page bound to the recognised values, not to literals
      (HW-02-0173).
- [ ] Nothing from `params.cardInfo` written to the log (HW-02-0172).
- [ ] Avoid areas read after `setWindowLayoutFullScreen` resolves
      (HW-02-0175), and `off('avoidAreaChange')` in `onWindowStageDestroy`
      (HW-02-0174).

## Architecture

Four files, no service layer - the control is the service.

| File | Role |
| --- | --- |
| `pages/MainPage.ets` | `@Entry`. Owns the `NavPathStack`, the `TextPicker` sheet that selects the document type, the sample photo strip, and the verified-state display. Provides `pageStack`, `cardData` and `cardImgUri`. |
| `pages/CardRecognitionPage.ets` | The `NavDestination`. Hosts `CardRecognition` and the confirmation overlay, and writes the result into the consumed model. |
| `components/CardData.ets` | The confirmation form the user submits, drawn *behind* the control in the same `Stack`. |
| `model/DataModel.ets` | `ListData`, `CardInfoData`, and `CardConstants` - the label-to-`CardType` map and the picker's label array. |

State flows through `@Provide` / `@Consume`, declared on `MainPage` and
consumed two levels down:

```ts
// MainPage
@Provide('pageStack') pageStack: NavPathStack = new NavPathStack();
@Provide cardData: CardInfoData = { cardType: '', name: '', id: '' };
@Provide cardImgUri: string = '';
```

The scan page is registered by `routerMap`, and `main_pages.json` lists only
`pages/MainPage`:

```json
// entry/src/main/resources/base/profile/route_map.json
{
  "routerMap": [
    {
      "name": "CardRecognitionPage",
      "pageSourceFile": "src/main/ets/pages/CardRecognitionPage.ets",
      "buildFunction": "cardRecognitionPageBuilder",
      "data": { "description" : "this is CardRecognitionPage" }
    }
  ]
}
```

The layout of the scan page is worth noting because it is not obvious: the
confirmation form is drawn **first** and the recognition control is stacked on
top of it. When the control finishes and removes itself, the form underneath is
already populated and visible - there is no second navigation.

```ts
Stack({ alignContent: Alignment.Top }) {
  Stack() { CardData(); }                     // the form, always present
  if (canIUse(...)) { CardRecognition({...}) }  // the control, on top
}
```

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Map the picker labels to `CardType` once.**

   ```ts
   export class CardConstants {
     static readonly CARD_ID_LIST: Record<string, CardType> = {
       '二代身份证': CardType.CARD_ID,                 // second-generation ID card
       '护照': CardType.CARD_PASSPORT,                 // passport
       '港澳台居民身份证': CardType.CARD_AUTO,          // HK/Macao/Taiwan resident ID
       '台湾居民来往大陆通行证': CardType.CARD_AUTO      // Taiwan mainland travel permit
     };
     static readonly CARD_ARR: string[] = [
       '二代身份证', '护照', '港澳台居民身份证', '台湾居民来往大陆通行证'];
   }
   ```

   `CARD_AUTO` is the fallback for the two document types that have no dedicated
   enum value - the control detects the layout itself.

2. **Select the type in a sheet.** `bindSheet` with `$$` two-way binding, and a
   `TextPicker` over the label array:

   ```ts
   .bindSheet($$this.useSelect, this.cardSelect(), {
     height: SheetSize.MEDIUM,
     backgroundColor: Color.White,
     title: { title: $r('app.string.select_card_type') }
   })
   ```

3. **Gate the entry point on a selection.** The scan page must not open with no
   document type, or `supportType` resolves to `undefined`:

   ```ts
   .onClick(() => {
     if (this.selectText) {
       this.pageStack.pushPathByName('CardRecognitionPage', this.selectText, (popInfo) => {
         this.isRealName = popInfo.result as string;
       });
     } else {
       this.promptAction.showToast({ message: $r('app.string.select_card_type'), ... });
     }
   })
   ```

   Pop a value whose type matches what the receiver declares - the shipped code
   pops a `Resource` into a field declared `string` (HW-02-0179).

4. **Read the parameter in `onReady`.** The navigation parameter arrives on the
   `NavDestinationContext`, not as a constructor argument:

   ```ts
   .onReady((context: NavDestinationContext) => {
     this.pageStack = context.pathStack;
     this.cardType = context.pathInfo.param as string;
   })
   ```

5. **Guard the control, and handle the negative branch** (HW-02-0177):

   ```ts
   if (canIUse('SystemCapability.AI.Component.CardRecognition')) {
     CardRecognition({ /* step 6 */ });
   } else {
     // the shipped sample has no else branch and leaves the user on a dead form
     this.unsupportedNotice();
   }
   ```

6. **Declare the control with `onResult`** (HW-02-0171):

   ```ts
   CardRecognition({
     supportType: CardConstants.CARD_ID_LIST[this.cardType],
     cardSide: CardSide.DEFAULT,
     cardRecognitionConfig: {
       defaultShootingMode: ShootingMode.MANUAL,
       isPhotoSelectionSupported: true
     },
     onResult: ((params: CardRecognitionResult) => {
       // step 7
     })
   })
   ```

   `cardSide: CardSide.DEFAULT` lets the control decide which sides to ask for.
   `isPhotoSelectionSupported: true` adds the gallery option next to the
   shutter, which is what makes this flow work without a camera permission
   *and* without a picker of your own.

7. **Handle the result: fail closed, then fill the model** (HW-02-0178,
   HW-02-0172):

   ```ts
   hilog.info(0x0001, TAG, `params code: ${params.code}`);
   if (params.code !== 200) {
     this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_failed') });
     this.pageStack.pop();
     return;                       // the shipped sample has no return here
   }
   if (params.cardInfo?.front !== undefined) {
     this.cardDataSource.push(params.cardInfo?.front);
     this.cardData.cardType = this.cardType;
     this.cardData.name = this.cardDataSource[0].name;
     this.cardData.id = this.cardDataSource[0].idNumber;
     this.cardImgUri = this.cardDataSource[0].cardImageUri;
   }
   if (params.cardInfo?.back !== undefined) {
     this.cardDataSource.push(params.cardInfo?.back);
   }
   // do NOT log params.cardInfo - it carries the name, the number and the image URI
   ```

   `200` is the success code. `front` and `back` are both optional, and the
   front is the one that carries the identity fields.

8. **Render what was recognised** (HW-02-0173). The shipped confirmation page
   displays literals; bind it to the model instead:

   ```ts
   Column() {
     Text(this.cardData.cardType).textFancy();
     Divider().strokeWidth(0.5);
     Text(this.cardData.name).textFancy();        // shipped: Text('***')
     Divider().strokeWidth(0.5);
     Text(this.cardData.id).textFancy();          // shipped: Text('123456789877654321')
     Divider().strokeWidth(0.5);
   }
   ```

9. **Pop the confirmation back to the home page.**

   ```ts
   Button($r('app.string.button_put_info'))
     .onClick(() => {
       this.realName = $r('app.string.haven_real_name');
       this.pageStack.pop(this.realName);
     })
   ```

10. **Set up the immersive layout correctly** (HW-02-0175, HW-02-0174). The
    shipped `EntryAbility` reads the avoid areas before the layout change has
    taken effect and never unsubscribes:

    ```ts
    windowClass.setWindowLayoutFullScreen(true).then(() => {
      // read the areas here, after the immersive layout is in effect
      let bottom = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
      AppStorage.setOrCreate('bottomRectHeight', bottom.bottomRect.height);
      let top = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
      AppStorage.setOrCreate('topRectHeight', top.topRect.height);

      windowClass.on('avoidAreaChange', (data) => { /* later changes only */ });
    }).catch((err: BusinessError) => {
      hilog.error(0x0000, 'testTag', 'setWindowLayoutFullScreen failed: ' + JSON.stringify(err));
    });
    ```

    ```ts
    onWindowStageDestroy(): void {
      this.windowClass?.off('avoidAreaChange');
    }
    ```

11. **Consume the heights as padding.** `MainPage` reads them with
    `@StorageLink` and converts px to vp at the point of use:

    ```ts
    .safeAreaPadding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight),
      left: 16,
      right: 16
    })
    ```

    `getWindowAvoidArea` returns px; `safeAreaPadding` takes vp. The conversion
    is not optional.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`卡证信息识别示例代码.zip#entry/src/main/ets/pages/CardRecognitionPage.ets:36-58` -
the control stacked over the confirmation form, behind the availability guard:

```ts
  build() {
    NavDestination() {
      Stack({ alignContent: Alignment.Top }) {
        Stack() {
          CardData();
        }
        .size({
          width: '100%',
          height: '100%'
        });

        //Card Recognition Control
        if (canIUse('SystemCapability.AI.Component.CardRecognition')) {
          CardRecognition({
            supportType: CardConstants.CARD_ID_LIST[this.cardType], //Types of cards
            cardSide: CardSide.DEFAULT, // identify Pages
            //Card certificate recognition configuration item.
            cardRecognitionConfig: {
              defaultShootingMode: ShootingMode.MANUAL,
              isPhotoSelectionSupported: true
            },
            //Card recognition result;Deprecated starting with API version 19. Use onResult instead.
            callback: ((params: CardRecognitionResult) => {
```

That last comment is the sample telling you not to use the line it is sitting
on - see pitfall 1.

`卡证信息识别示例代码.zip#entry/src/main/ets/pages/CardRecognitionPage.ets:64-76` -
front and back handled separately, the front carrying the identity fields:

```ts
              // Front of ID photo
              if (params.cardInfo?.front !== undefined) {
                this.cardDataSource.push(params.cardInfo?.front);
                this.cardData.cardType = this.cardType;
                this.cardData.name = this.cardDataSource[0].name;
                this.cardData.id = this.cardDataSource[0].idNumber;
                this.cardImgUri = this.cardDataSource[0].cardImageUri;
                //To protect personal privacy, personal information cannot be printed.
              }
              // Back of ID photo
              if (params.cardInfo?.back !== undefined) {
                this.cardDataSource.push(params.cardInfo?.back);
              }
```

`卡证信息识别示例代码.zip#entry/src/main/ets/model/DataModel.ets:30-42` - the
label-to-type map and the picker's range in one place:

```ts
export class CardConstants {
  static readonly CARD_ID_LIST: Record<string, CardType> = {
    '二代身份证': CardType.CARD_ID,
    '护照': CardType.CARD_PASSPORT,
    '港澳台居民身份证': CardType.CARD_AUTO,
    '台湾居民来往大陆通行证': CardType.CARD_AUTO
  };
  static readonly CARD_ARR: string[] = [
    '二代身份证',
    '护照',
    '港澳台居民身份证',
    '台湾居民来往大陆通行证'];
}
```

`卡证信息识别示例代码.zip#entry/src/main/ets/pages/MainPage.ets:106-133` - the
document-type picker inside the sheet:

```ts
  @Builder
  cardSelect() {
    Column() {
      TextPicker({ range: CardConstants.CARD_ARR, selected: this.selectIndex })
        .canLoop(false)
        .onChange((value: string | string[], index: number | number[]) => {
          this.selectIndex = index as number;
        })
        .defaultPickerItemHeight(60)
        .width('100%')
        .divider({} as DividerOptions)
        .backgroundColor($r('sys.color.white'))
        .zIndex(1);
      Button($r('app.string.button_checkin'))
        .width('100%')
        .onClick(() => {
          this.selectText = CardConstants.CARD_ARR[this.selectIndex];
          this.cardType = CardConstants.CARD_ID_LIST[this.selectText];
          this.useSelect = false;
        });
    }
```

`卡证信息识别示例代码.zip#entry/src/main/ets/pages/MainPage.ets:236-250` - the
entry point, gated on a selected type, with the pop result wired back:

```ts
              .onClick(() => {
                if (this.selectText) {
                  this.pageStack.pushPathByName('CardRecognitionPage', this.selectText, (popInfo) => {
                    this.isRealName = popInfo.result as string;
                  });
                } else {
                  this.promptAction.showToast({
                    message: $r('app.string.select_card_type'),
                    alignment: Alignment.Bottom,
                    offset: { dx: 0, dy: -64 },
                    duration: 800
                  });
                }
              });
```

`卡证信息识别示例代码.zip#entry/src/main/ets/pages/MainPage.ets:298-307` - the
avoid-area heights converted and applied as safe-area padding:

```ts
    .width('100%')
    .height('100%')
    .safeAreaPadding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight),
      left: 16,
      right: 16
    })
    .mode(NavigationMode.Stack)
    .backgroundColor('rgb(246, 247, 249)');
```

## Permissions & config

**No permissions.** `卡证信息识别示例代码.zip#entry/src/main/module.json5` has no
`requestPermissions` block, and the document has no 权限说明 section. The
`CardRecognition` control owns the camera and the gallery access, so the
application never requests either. This is the feature's main advantage over
`LIFE-22`.

Required in `module.json5`:

```json5
"deviceTypes": ["phone", "tablet", "2in1"],
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map",
```

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

`oh-package.json5` has empty `dependencies` at both levels - Vision Kit is an
SDK kit, and there is no AppGallery Connect configuration of any kind.

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **The control may be absent.** The sample itself guards on
  `canIUse('SystemCapability.AI.Component.CardRecognition')`, so the capability
  is not present on every device the module declares support for
  (`phone`, `tablet`, `2in1`). Plan the negative branch (HW-02-0177).
- **Only two document types have a dedicated `CardType`.** In this sample's map,
  `二代身份证` resolves to `CARD_ID` and `护照` to `CARD_PASSPORT`; the two
  Hong Kong / Macao / Taiwan documents both fall back to `CARD_AUTO`.
- **`supportType` is looked up by label.** `CardConstants.CARD_ID_LIST[...]`
  returns `undefined` for any string not in the map, so the navigation parameter
  must be one of `CARD_ARR`'s four entries and nothing else.
- **`params.cardInfo.front` and `.back` are both optional.** The identity
  fields (`name`, `idNumber`, `cardImageUri`) come from the front.
- **`params.code === 200` is success.** Any other value means no result.
- **`getWindowAvoidArea` returns px.** Convert with `px2vp` before using the
  values as padding, as `MainPage.ets:300-305` does.
- **The recognition result is personal data.** Name, document number and both
  image URIs arrive in one object; treat the whole `cardInfo` as sensitive
  (HW-02-0172).

## Pitfalls

1. **HW-02-0171 - the sample uses the `callback` parameter its own comment
   marks as deprecated.** `CardRecognitionPage.ets:57` reads
   `//Card recognition result;Deprecated starting with API version 19. Use
   onResult instead.` and line 58 is `callback: ((params: CardRecognitionResult)
   => {`. The project targets API 20 (`build-profile.json5`,
   `"compatibleSdkVersion": "6.0.0(20)"`). The replacement is already in use in
   this same industry: `LIFE-01`'s sample calls the control with
   `onResult: (async (params: CardRecognitionResult) => {`
   (`Life_Framework_Code_V1.zip#features/home/src/main/ets/pages/CredentialsPage.ets:173`).
   Rename the property; the body is unchanged.

2. **HW-02-0182 - the document's snippet copies the deprecated parameter and
   drops the warning.** The document contains exactly one code block
   (`23_card_information_recognition.md:28-52`), it shows `callback:` at line 35,
   and it reproduces neither the deprecation comment nor the `canIUse` guard
   that wraps the control in the ZIP. The document is therefore strictly worse
   than the code it links to. Read the ZIP, not the page.

3. **HW-02-0172 - a privacy comment is contradicted six lines later.**
   `CardRecognitionPage.ets:71` says
   `//To protect personal privacy, personal information cannot be printed.` and
   `:77-78` print exactly that:
   `hilog.info(0x0001, TAG, \`params cardInfo front: ${JSON.stringify(params.cardInfo?.front)}\`);`
   and the same for `.back`. The serialised object holds the name, the document
   number and the image URIs. Both calls pass the interpolated message as the
   hilog format string, so nothing is filtered. Delete both lines; `:59` and
   `:63` already log `params.code` and `params.cardType`, which is enough.

4. **HW-02-0173 - the confirmation page shows fabricated data.**
   `CardData.ets:69` is `Text('***')` and `:73` is `Text('123456789877654321')`,
   where the recognised name and number belong. The values are there:
   `CardRecognitionPage.ets:68-69` writes `cardData.name` and `cardData.id` into
   the `@Provide` object that `CardData` consumes at `:20` - and grepping the
   ZIP shows neither field is ever read. Every scan therefore shows the same
   asterisks and the same eighteen-digit number, and the submit button confirms
   them. The document says the recognised information reaches this page
   (`识别后将卡证信息传入信息提交页面。` - "After recognition, the card
   information is passed to the information submission page"), and it does; the
   page just does not render it. Bind the two `Text` components to
   `this.cardData.name` and `this.cardData.id`.

5. **HW-02-0177 - the `canIUse` guard has no fallback.**
   `CardRecognitionPage.ets:48-81` wraps the control in the guard with no `else`.
   The `CardData` form behind it is rendered unconditionally at `:38-45`, so on
   a device without the capability the user lands on a real-name form that never
   fills, with a submit button that pops back and marks the account verified.
   Add an `else` that says so and pops.

6. **HW-02-0178 - the failure branch pops without returning.**
   `CardRecognitionPage.ets:60-62` is
   `if (params.code !== 200) { this.pageStack.pop(); }` and execution falls
   through into the `cardInfo` handling and the logging. A failed scan can still
   write into the model of a page that is already gone, and the user gets no
   explanation. Add the `return` - and the message.

7. **HW-02-0174 - `on('avoidAreaChange')` is never unsubscribed.**
   `EntryAbility.ets:66-74` registers the listener; `onWindowStageDestroy`
   (`:77-80`) only logs, and `off(` appears nowhere in the file. The window
   reference documents the counterpart:
   `off(type: 'avoidAreaChange', callback?: Callback<AvoidAreaOptions>): void -
   Unsubscribes from the events indicating changes to the system avoid area of
   the current window.` Keep the window handle and unsubscribe in
   `onWindowStageDestroy`.

8. **HW-02-0175 - the avoid areas are read before the immersive layout is in
   effect.** `EntryAbility.ets:48` calls
   `windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(...)` and
   `:56` and `:61` call `getWindowAvoidArea` outside that chain, immediately.
   The reference gives the signature as
   `setWindowLayoutFullScreen(isLayoutFullScreen: boolean): Promise<void>` -
   the promise exists because the layout is not applied when the call returns,
   and the immersive layout is precisely what turns the status bar and the
   navigation bar into avoid areas. Move both reads into the `.then()`.

9. **HW-02-0179 - a `Resource` is popped as the navigation result and cast to
   `string`.** `CardData.ets:22` declares
   `@State realName: string | Resource = ''`, `:105` assigns
   `$r('app.string.haven_real_name')` to it and `:106` pops it;
   `MainPage.ets:54` declares `@State isRealName: string = ''` and `:240` does
   `this.isRealName = popInfo.result as string`. It only works because every use
   of `isRealName` is a truthiness test (`:152`, `:189`, `:214`, `:222`,
   `:230`). Pop a boolean or a plain string and render the localized text at the
   point of display, as `:153` already does.

10. **HW-02-0176 - `ListItem` is used inside a `Row`.**
    `MainPage.ets:285-291` builds the four photo-quality hints as
    `Row() { ForEach(this.data, (item: ListData) => { ListItem() { ... } }) }`,
    and there is no `List` in the file. The reference is absolute: "The parent
    of this component can only be `List` or `ListItemGroup`." The wrapper buys
    nothing - the strip is four static images - so drop it and put the
    `@Builder` output straight in the `Row`.

11. **HW-02-0180 - `cardDataSource` is `@Provide` with no consumer.**
    `CardRecognitionPage.ets:30` declares it as provided state; grepping the
    project finds no `@Consume cardDataSource` anywhere. It reads as though the
    recognition buffer were shared with the confirmation component, which is
    exactly the wrong impression given pitfall 4. Make it a private field.

12. **HW-02-0181 - `MainPage.cardType` is computed and never read.**
    `MainPage.ets:55` declares `@State cardType: CardType = CardType.CARD_AUTO`
    and `:123` assigns the resolved enum, but nothing reads it. What crosses the
    navigation boundary is the display label (`:239`,
    `pushPathByName('CardRecognitionPage', this.selectText, ...)`), and
    `CardRecognitionPage.ets:50` resolves the enum from that label a second
    time. Pick one: pass the resolved `CardType`, or delete the field.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- CardRecognition control reference:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/vision-card-recognition
- Card recognition guide:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/vision-cardrecognition
- ListItem (allowed parent components):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- hilog (`{public}` vs `{private}`, format string):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- NavDestination:
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
