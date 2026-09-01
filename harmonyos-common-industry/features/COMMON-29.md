---
id: COMMON-29
title: User agreement and privacy policy consent - a half-modal sheet gate with full-screen detail covers, persisted
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/29_user_agreement_and_privacy_policy.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_agreement_and_privacy_policy-0000002331953689
sample: huawei_industry_tree/19_common_technical_solutions/downloads/UserAgreementAndPrivacyPolicy.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit"]
apis: ["component .bindSheet", SheetSize, SheetType, DismissSheetAction, DismissReason, "component .bindContentCover", "PersistentStorage.persistProp", "@StorageLink", "@StorageProp", Navigation, NavPathStack, "NavPathStack.pushPathByName", routerMap, "UIAbilityContext.terminateSelf", Span, "component .textAlign", setInterval, clearInterval]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0090, HW-19-0091, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must **block first use until the user accepts
the user agreement and privacy policy** - the compliance gate every consumer app
ships: a sheet on first launch, links inside it that open the full texts, an
Agree button that lets the user in, and a Cancel that closes the application.

The distinctive part is the layering: a **half-modal sheet** for the consent
prompt and **full-screen covers** for the two detail texts, bound to a component
inside the sheet - so the detail page opens over the sheet without dismissing it.

## Feature checklist

The application must:

- Persist whether consent has been given, and read it before the first frame.
- Show the consent sheet when consent is missing; go straight to the home page
  when it is present.
- Make the sheet non-dismissable by ordinary means: no close button, and back
  press closes the application rather than the sheet.
- Render the consent paragraph as one `Text` of `Span`s so the two agreement
  names can be individually coloured and tappable.
- Open each detail text as a full-screen cover from its `Span`, with a back
  control.
- Write the consent state and navigate to the home page on Agree.
- Terminate the application on Cancel.
- Localise the consent text (HW-19-0090).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `common/Constants.ets` | layout constants, `AGREED_STATE`, `DELAY_TIME`, `ROUTER = 'Home'`, and the agreement / policy body text |
| `common/CardData.ets`, `model/Model.ets`, `views/CardView.ets` | the home-page content behind the gate |
| `pages/Index.ets` | the whole gate: persisted state, the sheet, the two covers, Agree / Cancel |
| `pages/Home.ets` | the destination after consent |
| `resources/base/profile/route_map.json` | route `Home` |

**Persistence is registered at module scope**, before the component exists:

```ts
// 初始化PersistentStorage
PersistentStorage.persistProp('agreedState', 0);
PersistentStorage.persistProp('flag', false);
```

This is the correct ordering - the two `persistProp` calls run when the module is
first imported, so the persisted values are in AppStorage before
`@StorageLink('agreedState')` binds to them. (Contrast COMMON-11 and COMMON-17,
where the same call is made too late.)

**Three modal layers**, each bound to a different component:

1. `bindSheet($$this.isShow, this.agreementAndPolicy, { height: SheetSize.FIT_CONTENT,
   preferType: SheetType.BOTTOM, showClose: false, blurStyle: BlurStyle.Regular, ... })`
   on the welcome `Column`.
2. `bindContentCover(this.isShowAgreement || this.isShowPolicy, ...)` on the
   `Text` **inside** the sheet - which is what lets a detail page open on top of
   the sheet rather than replacing it.
3. The `Navigation` / `NavPathStack` beneath everything, used only once consent
   is given.

**The gate is closed properly.** `showClose: false` removes the sheet's own close
affordance, and `onWillDismiss` intercepts the back press:

```ts
onWillDismiss: ((dismissSheetAction: DismissSheetAction) => {
  if (dismissSheetAction.reason === DismissReason.PRESS_BACK) {
    // 退出应用
    dismissSheetAction.dismiss();
    this.context.terminateSelf(...);
    this.flag = true;
  }
})
```

So back does not dismiss the sheet into an unguarded page - it ends the
application, which is the only correct answer when consent is refused.

**The clickable agreement names** come from splitting the paragraph into five
`Span`s and colouring / handling only indices 1 and 3
(`AGREEMENT_INDEX = 1`, `PRIVACY_INDEX = 3`).

## Implementation steps

1. **Register the persisted properties at module scope**, above the `@Entry`
   struct, so they are in AppStorage before any `@StorageLink` binds.
2. **Bind them**: `@StorageLink('agreedState') agreedState: number`.
3. **Decide in `aboutToAppear`**: consented -> `pushPathByName('Home', null,
   false)`; otherwise `isShow = true`. Do not add a second, timer-based path to
   the same decision (HW-19-0091).
4. **Bind the sheet** with `SheetSize.FIT_CONTENT`, `SheetType.BOTTOM` and
   `showClose: false`.
5. **Intercept the back press** in `onWillDismiss`, check
   `reason === DismissReason.PRESS_BACK`, and terminate.
6. **Build the paragraph from `Span`s**, colouring and handling only the two
   agreement names.
7. **Bind the full-screen covers to that `Text`**, choosing the builder from
   which flag is set, and reset the flag in `onDisappear`.
8. **On Agree**: set the persisted state, hide the sheet, push the home route.
   **On Cancel**: `terminateSelf`.
9. **Externalise the legal text into resources** so it follows the device locale
   (HW-19-0090).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Persisted state and the launch decision

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/pages/Index.ets`

```ts
// 初始化PersistentStorage
PersistentStorage.persistProp('agreedState', 0);
PersistentStorage.persistProp('flag', false);

@Entry
@Component
struct Index {
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageLink('agreedState') agreedState: number = 1;
  @StorageLink('flag') flag: boolean = true;
  @State isShow: boolean = false;
  @State isShowAgreement: boolean = false;
  @State isShowPolicy: boolean = false;
  intervalID: number = 0;
  @Provide('pathStack') pathStack: NavPathStack = new NavPathStack();
  private context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;

  aboutToAppear(): void {
    if (this.agreedState === Constants.AGREED_STATE[1]) {
      this.pathStack.pushPathByName(Constants.ROUTER, null, false);
    } else {
      this.isShow = true;
    }

    // 拉起协议                                    // FIX (HW-19-0091): redundant, leaks a timer
    this.intervalID = setInterval(() => {
      if (!this.flag) {
        this.isShow = true;
        this.flag = !this.flag;
      }
      if (this.intervalID) {
        clearInterval(this.intervalID);
      }
    }, Constants.DELAY_TIME);
  }
}
```

### The consent sheet, and closing the app on back

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/pages/Index.ets`

```ts
.bindSheet($$this.isShow, this.agreementAndPolicy, {
  height: SheetSize.FIT_CONTENT,
  preferType: SheetType.BOTTOM,
  showClose: false,
  blurStyle: BlurStyle.Regular,
  onWillDismiss: ((dismissSheetAction: DismissSheetAction) => {
    if (dismissSheetAction.reason === DismissReason.PRESS_BACK) {
      // 退出应用
      dismissSheetAction.dismiss();

      this.context.terminateSelf((err) => {
        if (err.code) {
          return;
        }
      });
      this.flag = true;
    }
  }),
  onDisappear: (() => {
    this.isShow = false;
  })
})
.width(Constants.FULL_PERCENT)
.height(Constants.FULL_PERCENT);
```

### The tappable agreement names and the detail covers

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/pages/Index.ets`

```ts
Text() {
  ForEach(Constants.AGREEMENT_AND_PRIVACY_CONTENT_LIST, (textItem: string, index: number) => {
    Span(textItem)
      .fontSize($r('app.float.font_size_14'))
      .fontWeight(FontWeight.Normal)
      .fontColor(index === Constants.AGREEMENT_INDEX || index === Constants.PRIVACY_INDEX ?
        Constants.FONT_COLOR[2] : Constants.FONT_COLOR[1])
      .lineHeight($r('app.float.line_height_19'))
      .onClick(() => {
        if (index === Constants.AGREEMENT_INDEX) {
          this.isShowAgreement = true;
        } else if (index === Constants.PRIVACY_INDEX) {
          this.isShowPolicy = true;
        }
      });
  });
}
.textAlign(TextAlign.JUSTIFY)
.width(Constants.FULL_PERCENT)
.bindContentCover(this.isShowAgreement || this.isShowPolicy,
  this.isShowAgreement ? this.userAgreementBuilder() : this.privacyPolicyBuilder(), {
    onDisappear: (() => {
      if (this.isShowAgreement) {
        this.isShowAgreement = false;
      } else {
        this.isShowPolicy = false;
      }
    })
  });
```

### Agree and Cancel

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/pages/Index.ets`

```ts
Row() {
  Button($r('app.string.cancel'))
    .fontColor(Constants.FONT_COLOR[2])
    .backgroundColor(Constants.BUTTON_COLOR)
    .onClick(() => {
      this.context.terminateSelf((err) => {
        if (err.code) {
          return;
        }
      });
      this.flag = true;
    });

  Button($r('app.string.agree'))
    .fontColor($r('app.color.font_color_white'))
    .onClick(async () => {
      this.agreedState = Constants.AGREED_STATE[1];
      this.isShow = false;
      this.pathStack.pushPathByName(Constants.ROUTER, null);
    });
}
.justifyContent(FlexAlign.SpaceBetween);
```

### The detail cover

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/pages/Index.ets`

```ts
@Builder
privacyPolicyBuilder(): void {
  Column() {
    Row() {
      Image($r('app.media.back'))
        .onClick(() => {
          this.isShowPolicy = false;
        });
      Text($r('app.string.privacy_title'))
        .fontSize($r('app.float.font_size_20'))
        .fontWeight(FontWeight.Bold);
    }
    .height($r('app.float.height_56'));

    Scroll() {
      Text() {
        Span(Constants.PRIVACY_DETAIL[0])
        // ...
      };
    };
  };
}
```

### The consent text (as shipped - see HW-19-0090)

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/ets/common/Constants.ets`

```ts
/**
 * Agreement and privacy content
 */
static readonly AGREEMENT_AND_PRIVACY_CONTENT_LIST: string[] = [
  '亲爱的用户感谢您选择成长记录！\n为了加强对您个人信息的保护，根据最新的法律法规要求，我们更新了',
  '《用户协议》',
  '和',
  '《隐私政策》',
  '，请您仔细阅读并确认。本系统非常重视对用户信息、用户隐私的保护...'
];
/**
 * Agreement`s index in Agreement and privacy content
 */
static readonly AGREEMENT_INDEX: number = 1;
/**
 *Privacy`s index in Agreement and privacy content
 */
static readonly PRIVACY_INDEX: number = 3;
```

## Permissions & config

**No permissions are required** and none are declared - the gate is pure UI plus
`PersistentStorage`.

`UserAgreementAndPrivacyPolicy.zip#UserAgreementAndPrivacyPolicy/entry/src/main/module.json5`
declares `"deviceTypes": ["phone", "tablet", "2in1"]`, the usual `EntryAbility`
with the home skill, an `EntryBackupAbility`, and
`"routerMap": "$profile:route_map"` for the `Home` destination. There is no
`requestPermissions` block.

`entry/src/main/resources` contains `base`, `dark`, `en_US` and `zh_CN`, with 16
localised strings per locale - the sheet title, the buttons and the two detail
titles, but not the consent body (HW-19-0090).

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`persistProp` must precede any binding of the key.** Here it is at module
  scope, which is the reliable place; putting it in `onWindowStageCreate`'s
  `loadContent` callback (as COMMON-11 and COMMON-17 do) is too late.
- **`showClose: false` plus `onWillDismiss` is what makes the sheet a gate.**
  Either one alone leaves a way past it.
- **`DismissReason.PRESS_BACK` is only one reason.** `onWillDismiss` also fires
  for slide-down and mask taps; this sheet has `mask` left at its default and
  `SheetSize.FIT_CONTENT`, so decide deliberately what the other reasons should
  do.
- **`bindContentCover` is bound to a component inside the sheet.** That is
  deliberate - binding it to the page's outer `Column` would put the detail cover
  beneath the sheet.
- **The cover content is chosen by a ternary on two booleans.** Only one may be
  true at a time; the `onDisappear` handler restores that invariant.
- **`terminateSelf` is asynchronous.** The `this.flag = true` that follows it runs
  during termination; do not add work after it that must complete.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The agreement and privacy text is hardcoded Chinese in `Constants.ets`,
  which is incorrect for a localised app.** The project ships `en_US` and
  `zh_CN` resources for the surrounding chrome, so an English-locale user is
  shown English buttons above Chinese consent text - and legal updates applied to
  the resource files silently miss this array. (HW-19-0090)
- **The `setInterval` block in `aboutToAppear` is incorrect.** It is a one-shot
  disguised as an interval, it is never cancelled in `aboutToDisappear`, and on
  the only launch where its guard passes the `else` branch above has already set
  `isShow`. Delete it. (HW-19-0091)
- **`@StorageLink('agreedState') agreedState: number = 1` has a misleading local
  default.** The persisted default registered at module scope is `0`; the `1`
  here would mean "already agreed" if the key were ever missing. Keep the two
  defaults consistent.
- **`Cancel` and back press both terminate, but only back press goes through
  `onWillDismiss`.** The two paths duplicate the `terminateSelf` + `flag` code;
  extract it if you extend the flow.
- **The consent state is a single number, not a version.** A revised policy
  cannot re-prompt users who already agreed - store the agreed policy version
  rather than a boolean-ish `0`/`1`.
- **`AGREEMENT_INDEX` and `PRIVACY_INDEX` are positions in the text array.**
  Editing the paragraph without updating the two indices silently makes the wrong
  span clickable.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-sheet-transition -
  `bindSheet`, `SheetSize`, `SheetType`, `showClose`, `onWillDismiss`,
  `DismissSheetAction` and `DismissReason`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-modal-transition -
  `bindContentCover` and its `onDisappear`.
- `documentation/harmonyos-guides/03_application-framework/arkts-persiststorage.md` -
  `PersistentStorage.persistProp` and the ordering requirement this sample gets
  right.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-persiststorage
- `documentation/harmonyos-guides/01_getting-started/resource-categories-and-access.md` -
  localised resource qualifiers and `$r()` access.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/resource-categories-and-access
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-uiabilitycontext.md` -
  `terminateSelf`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-state-management -
  application-level state management, linked by the document.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_agreement_and_privacy_policy-0000002331953689
