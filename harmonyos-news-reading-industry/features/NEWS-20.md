---
id: NEWS-20
title: Reading-progress widget - push chapter state to the card with updateForm, jump back with a router event
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/20_read_card.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/read_card-0000002351196153
sample: huawei_industry_tree/11_news_reading/downloads/ReadCard.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.FormKit", "@kit.PerformanceAnalysisKit"]
apis: [FormExtensionAbility, onAddForm, onRemoveForm, onAcquireFormState, "formBindingData.createFormBindingData", "formProvider.updateForm", "formProvider.openFormManager", postCardAction, "@LocalStorageProp", "@StorageLink", "@StorageProp", "preferences.getPreferencesSync", "preferences.removePreferencesFromCacheSync", putSync, getSync, flush, "deviceInfo.sdkApiVersion", bindSheet, "UIAbility.onNewWant"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0031, HW-11-0043, HW-11-0044]
status: verified
---

## When to use

Load this card when your app has **one piece of state the user wants on the home
screen** - where they got to, what is next, what is due - and tapping it must
land them on exactly that thing inside the app. Reading progress here; the same
build serves a "continue watching" tile, a next-task widget, an order-status
card.

Two mechanisms do all the work and they run in opposite directions. Outward:
the app pushes new state to the card with `formProvider.updateForm`, which needs
the `formId` of every live card - so the provider must **persist the ids it is
handed** in `onAddForm` and the app must read that list back. Inward:
`postCardAction({ action: 'router' })` launches a named ability with parameters,
which the ability parses in `onCreate` (cold start) *and* `onNewWant` (warm
start).

The reason to read this card rather than just the widget guide is the awkward
part in between: the card runs in the `FormExtensionAbility` process and the
page runs in the `UIAbility` process, and they share state through
`preferences`. That cross-process detail is where most first implementations go
wrong, and the sample handles it explicitly.

## Feature checklist

- A six-chapter reader page with previous/next controls that grey out at the
  ends.
- A header button opens a bottom sheet previewing the card and offering
  添加卡片 (add card).
- Adding the card calls `formProvider.openFormManager` on API 18 and above, and
  shows an explanatory toast below that.
- The card shows the chapter number, the chapter name and the first line of the
  body, in a 1x2 tile.
- Turning a page in the app updates every card already on the home screen, with
  no interaction from the user.
- Tapping the card opens the app **at the chapter the card is showing**, whether
  the app was closed or already running in the background.
- Removing the card from the home screen drops its id from the provider's list.

## Architecture

One `entry` module with two abilities and one extension - the widget is not a
separate module.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       parses want.parameters.params -> AppStorage('chapter')
├── entrybackupability/EntryBackupAbility.ets
├── entryformability/EntryFormAbility.ets  the provider: onAddForm stores the formId, seeds the card
├── model/FormData.ets                  the payload class sent to the card
├── model/FormInfo.ets                  formId/formName/formDimension, logged on add
├── pages/ReadPage.ets                  @Entry: the reader, saveHistory(), the add-card sheet
├── readcard/ReadCard.ets               the card UI, @Entry(localStorage) + postCardAction
├── utils/Logger.ets
├── utils/PreferencesUtil.ets           the cross-process store: formId list + reading state
└── views/BottomView.ets                chapter bar, @StorageLink('chapter')
```

`resources/base/profile/form_config.json` declares one form, `ReadCard`, with
`isDynamic: true`, `defaultDimension: "1*2"` and `updateEnabled: false`.

The documented tree matches the zip.

**The design decision worth copying** is that `AppStorage['chapter']` is the
single source of truth for "which chapter is on screen", and that three
independent writers and readers agree on it:

- `EntryAbility.onCreate` / `onNewWant` write it from the card's router
  parameters,
- `ReadPage` binds it as `@StorageLink('chapter') bookmark`,
- `BottomView` binds the *same key* as `@StorageLink('chapter') bookmark`, not
  as a `@Link` from its parent.

That last point is the one to notice. The chapter bar could have taken the
bookmark as a `@Link` prop, and then the ability's deep link would have had to
travel page -> child. By having both bind the AppStorage key directly, a card tap
that lands in `onNewWant` updates the page and the navigation bar in the same
frame, and the page needs no `onNewWant` plumbing of its own. `@StorageLink` is
the right tool exactly when a value has a writer outside the component tree -
and an ability launched by a widget is precisely that.

**The second decision worth copying** is in `PreferencesUtil.getPreferences`:

```typescript
getPreferences(context: Context): preferences.Preferences {
  preferences.removePreferencesFromCacheSync(context, MY_STORE);
  return preferences.getPreferencesSync(context, { name: MY_STORE });
}
```

Dropping the cache before every read looks wasteful until you remember the
`FormExtensionAbility` is a *different process* from the `UIAbility`. Each holds
its own in-memory `Preferences` instance over the same file; without the evict,
the provider would keep serving the formId list it read when it started, and a
card added after the app last touched preferences would never receive an
update. This is the correct fix for cross-process `preferences`, and it is the
line most implementations are missing when their widget silently stops
refreshing.

## Implementation steps

1. **Declare the form extension** in `module.json5` with `"type": "form"` and a
   `ohos.extension.form` metadata entry pointing at
   `$profile:form_config`, and declare the card itself in `form_config.json`
   with `isDynamic: true` and the dimensions you support.
2. **Persist the formId in `onAddForm`.** The `want` carries
   `ohos.extra.param.key.form_identity`; store it in a list under a known key.
   Without this the app has nothing to address `updateForm` at.
3. **Seed the new card from the same store** in the same callback, so a card
   added mid-book shows the current chapter rather than chapter one.
4. **Remove the id in `onRemoveForm`,** or the app will keep calling
   `updateForm` on dead cards forever.
5. **Write reading state and push it in one function.** `saveHistory(chapter)`
   writes the four fields to preferences, then loops the stored formIds building
   a `FormData` per id and calling `formProvider.updateForm(formId, formMsg)`.
6. **Call it on every state change** - the chapter bar's buttons pass
   `onChildClick` back up to `saveHistory`.
7. **Bind the card's fields with `@LocalStorageProp`** against the
   `LocalStorage` the card is entered with. `formBindingData` fields land in that
   storage by name, so the property names must match `FormData`'s fields
   exactly.
8. **Send the router event from the card** with `postCardAction(this, { action:
   'router', abilityName: 'EntryAbility', params: {...} })`.
9. **Parse the params in both entry points.** `want.parameters.params` is a JSON
   **string**, so it needs `JSON.parse`; handle it in `onCreate` for a cold
   start and `onNewWant` for a warm one. Handling only `onCreate` is the classic
   bug - the card works once, then stops navigating.
10. **Gate `openFormManager` on the API level.** It needs API 18; the sample
    reads `deviceInfo.sdkApiVersion` and falls back to a toast.

## Verified snippets

All snippets are from `ReadCard.zip`.

**The provider - `entry/src/main/ets/entryformability/EntryFormAbility.ets`** (as shipped)

```typescript
onAddForm(want: Want): formBindingData.FormBindingData {
  if (!want || !want.parameters) {
    return formBindingData.createFormBindingData('');
  }
  let formId = want.parameters['ohos.extra.param.key.form_identity'] as string;
  let formName = want.parameters['ohos.extra.param.key.form_name'] as string;

  let util = PreferencesUtil.getInstance();
  let preferences = util.getPreferences(this.context);
  // 保存 FormId
  util.addFormId(preferences, formId);
  if (formName === 'ReadCard') {
    let formData = new FormData(formId);
    formData.bookTitle = preferences.getSync('bookTitle', 0) as string;
    formData.bookChapter = preferences.getSync('bookChapter', 0) as number;
    formData.bookChapterName = preferences.getSync('bookChapterName', 0) as string;
    formData.bookBody = preferences.getSync('bookBody', 0) as string;
    return formBindingData.createFormBindingData(formData);
  }
  return formBindingData.createFormBindingData('');
}

onRemoveForm(formId: string) {
  PreferencesUtil.getInstance().removeFormId(this.context, formId);
}
```

**`onAddForm` has two jobs and both are mandatory.** It registers the id - this
is the only moment the system ever tells you a card exists - and it returns the
card's first payload. Skip the registration and `updateForm` has no target; skip
the seeding and every newly added card renders its `@LocalStorageProp` defaults
(chapter 0, empty strings) until the user next turns a page.

`formName` is checked because one `FormExtensionAbility` can back several
declared forms; the branch is what lets you add a second card later without the
first one's payload leaking into it.

`onRemoveForm` is the symmetric half. It is easy to leave out because nothing
visibly breaks - the ids just accumulate and `updateForm` starts failing
silently on every one of them.

**Pushing an update - `entry/src/main/ets/pages/ReadPage.ets`** (as shipped)

```typescript
//保存信息至卡片
saveHistory(chapter: number) {
  let util = PreferencesUtil.getInstance();
  let preferences = util.getPreferences(this.getUIContext().getHostContext()!);
  util.preferencesPut(preferences, 'bookChapter', chapter);
  util.preferencesPut(preferences, 'bookTitle', this.bookName);
  util.preferencesPut(preferences, 'bookChapterName', this.bookList[chapter].bookChapterName);
  util.preferencesPut(preferences, 'bookBody', this.bookList[chapter].bookBody);
  let idArr = PreferencesUtil.getInstance().getFormIds(preferences);
  if (idArr.length > 0) {
    idArr.forEach((formId: string) => {
      let formData = new FormData(formId);
      formData.bookTitle = preferences.getSync('bookTitle', 0) as string;
      formData.bookChapter = preferences.getSync('bookChapter', 0) as number;
      formData.bookChapterName = preferences.getSync('bookChapterName', 0) as string;
      formData.bookBody = preferences.getSync('bookBody', 0) as string;
      let formMsg: formBindingData.FormBindingData = formBindingData.createFormBindingData(formData);
      formProvider.updateForm(formId, formMsg).then(() => {
        Logger.info(`updateForm success.`);
      }).catch((error: Error) => {
        Logger.error(`updateForm failed: ${JSON.stringify(error)}`);
      });
    });
  }
}
```

**Write first, then read back, then push.** The order matters: preferences is
the durable record that `onAddForm` will read when a *future* card is added, and
the `updateForm` calls are a best-effort broadcast to the cards that exist now.
If a push fails - card removed a moment ago, provider not running - the
persisted state is still correct and the next `onAddForm` or system refresh
recovers.

Note that `formData` is rebuilt per id even though the payload is identical.
That is not redundancy for its own sake: `FormBindingData` is consumed by the
transport, and reusing one instance across several `updateForm` calls is not
something the API promises to support.

`getFormIds` defaults to `['']` when the key is absent, so on a device with no
card added the loop still runs once and calls `updateForm('')`. It fails into
the `catch` and logs, which is harmless but noisy - default to `[]` instead.

**The card and its router event - `entry/src/main/ets/readcard/ReadCard.ets`** (as shipped)

```typescript
let localStorage = new LocalStorage();

@Entry(localStorage)
@Component
export struct ReadCard {
  @LocalStorageProp('formId') formId: string = '';
  @LocalStorageProp('bookChapter') bookChapter: number = 0;
  @LocalStorageProp('bookChapterName') bookChapterName: string | Resource = '';
  @LocalStorageProp('bookBody') bookBody: string | Resource = '';

  build() {
    Row() {
      Column() {
        Row() {
          Text('第' + (this.bookChapter + 1) + '章:')   // "Chapter N:"
          Text(this.bookChapterName)
        }
        Text(this.bookBody).maxLines(1)
      }
      .alignItems(HorizontalAlign.Start)
    }
    .height('100%')
    .onClick(() => {
      postCardAction(this, {
        action: 'router',
        abilityName: 'EntryAbility',
        params: {
          formId: this.formId,
          targetChapter: this.bookChapter   //书籍章节
        }
      });
    });
  }
}
```

**`@Entry(localStorage)` plus `@LocalStorageProp` is the card's entire data
binding.** Everything a `FormBindingData` carries is deposited into that
`LocalStorage` under the field names of the object you passed to
`createFormBindingData`. There is no callback and no explicit refresh: a
successful `updateForm` rewrites the keys and the card re-renders. The
consequence is that `FormData`'s field names are a contract with the card's
property names, and a rename on one side silently produces a card showing
defaults.

`postCardAction` is only available inside a card. `action: 'router'` starts the
named ability in the card's own bundle; the `params` object is what arrives as
`want.parameters.params`. `formId` is included so a future handler could
distinguish which of several cards was tapped.

**Receiving the jump - `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
onCreate(want: Want): void {
  // 获取卡片信息
  if (want?.parameters?.params) {
    // want.parameters.params 对应 postCardAction() 中 params 内容
    let params: Record<string, Object> = JSON.parse(want.parameters.params as string);
    this.chapter = params.targetChapter as number;
  }
  AppStorage.setOrCreate('chapter', this.chapter);
  const bundleName = this.context.abilityInfo.bundleName;
  AppStorage.setOrCreate('bundleName', bundleName);
}

onNewWant(want: Want): void {
  if (want?.parameters?.params) {
    let params: Record<string, Object> = JSON.parse(want.parameters.params as string);
    this.chapter = params.targetChapter as number;
    AppStorage.setOrCreate('chapter', this.chapter);
  }
}
```

**Both callbacks, or the feature works exactly once.** `onCreate` runs on a cold
start; if the app is already alive in the background - which it usually is,
right after the user left it - the launch arrives at `onNewWant` instead, and an
implementation that only handles `onCreate` leaves the reader on whichever
chapter it was showing before. The parse is duplicated deliberately rather than
shared, and the payload is a JSON **string**, not an object: `want.parameters
.params as string` then `JSON.parse`.

`AppStorage.setOrCreate('chapter', ...)` before `loadContent` in the cold-start
path means `ReadPage`'s `@StorageLink` picks up the target chapter on its very
first build - no flash of chapter one.

**Adding the card from inside the app - `ReadPage.ets`** (as shipped)

```typescript
Button($r('app.string.add_card'))
  .onClick(() => {
    this.saveHistory(this.bookmark);
    let sdkApiVersionInfo: number = deviceInfo.sdkApiVersion;
    if (sdkApiVersionInfo >= 18) {
      let want: Want = {
        bundleName: this.bundleName,
        abilityName: 'EntryFormAbility',
        parameters: {
          'ohos.extra.param.key.form_dimension': 1,
          'ohos.extra.param.key.form_name': 'ReadCard',
          'ohos.extra.param.key.module_name': 'entry'
        },
      };
      try {
        formProvider.openFormManager(want);
        this.isShow = false;
      } catch (error) {
        Logger.error(`catch error, code: ${(error as BusinessError).code}, message: ${(error as BusinessError).message}`);
      }
    } else {
      this.getUIContext().getPromptAction().showToast({
        message: $r('app.string.not_api18')
      });
    }
  });
```

**`saveHistory` runs *before* the sheet opens the form manager**, so the
preferences store is current by the time `onAddForm` reads it in the other
process. Without that ordering the freshly added card shows whatever the last
page turn left behind.

The three `parameters` keys name the form to add - dimension, form name and
module - and must match `form_config.json`; `form_dimension: 1` is the enum
value for the declared `1*2` tile. `bundleName` comes from
`@StorageProp('bundleName')`, published by the ability from
`this.context.abilityInfo.bundleName` rather than hardcoded, which is the right
habit for a sample people will rename.

The `deviceInfo.sdkApiVersion >= 18` guard is version detection done properly:
the API is queried at runtime and the branch degrades to an explanation, rather
than the app crashing on an older device.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

```json5
// resources/base/profile/form_config.json
{
  "name": "ReadCard",
  "src": "./ets/readcard/ReadCard.ets",
  "uiSyntax": "arkts",
  "colorMode": "auto",
  "isDynamic": true,
  "isDefault": true,
  "updateEnabled": false,
  "scheduledUpdateTime": "10:30",
  "updateDuration": 1,
  "defaultDimension": "1*2",
  "supportDimensions": ["1*2"]
}
```

- `isDynamic: true` is required for an ArkTS card that receives `updateForm`
  pushes and sends `postCardAction`.
- `updateEnabled: false` disables *system-scheduled* refresh, which is the right
  choice here: the app pushes on every page turn, so a timer would only burn
  wakeups. Note that `scheduledUpdateTime` and `updateDuration` are therefore
  inert - they are only read when `updateEnabled` is `true`.
- Only `1*2` is supported, so the `form_dimension: 1` passed to
  `openFormManager` is the sole valid value.

`module.json5` declares `EntryFormAbility` as a second `extensionAbilities`
entry alongside the backup ability, with `"type": "form"` and its
`ohos.extension.form` metadata.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `openFormManager` additionally needs
  API 18, which the sample checks.
- **`targetChapter` from the card is trusted unchecked.** `ReadPage` indexes
  `this.bookList[this.bookmark]` directly; a card left on the home screen after
  the chapter list shrinks - or a stale card from an older install - indexes past
  the end. Clamp the parsed value against the current list length.
- **`Resource` values travel through `preferences` and `formBindingData` typed
  `as string`.** `bookChapterName` and `bookBody` are `$r(...)` objects written
  into the store and cast to `string` on the way out. It renders because `Text`
  accepts both, but the declared types are not the truth; substitute
  server-supplied strings and the casts become correct while the storage layer
  changes shape underneath you.
- The chapter list is six literal string resources in `ReadPage`. There is no
  content source, no scroll position within a chapter, and the card shows only
  the first line.
- `PreferencesUtil.addFormId` calls `preferencesPut`, which already flushes, and
  then flushes again.
- `BottomView` takes `bookLength` as a `@Link` although the value never changes;
  `@Prop` is the correct decorator for a read-only input.
- The evict-then-read strategy in `getPreferences` is correct for cross-process
  sharing but costs a file read on every call, and `saveHistory` calls it once
  per page turn. Fine at this scale; measure before applying it to a hot path.

## Pitfalls

- No defects were recorded against this document or sample during review. The
  cross-process, ordering and bounds concerns above are design notes, not
  findings.

## References

- `huawei_industry_tree/11_news_reading/docs/20_read_card.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/read_card-0000002351196153
- `documentation/harmonyos-references/02_application-framework/js-apis-app-form-formprovider.md` - `updateForm`, `openFormManager`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-form-formprovider
- `documentation/harmonyos-references/02_application-framework/js-apis-app-form-formbindingdata.md` - `createFormBindingData` and how fields reach the card
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-form-formbindingdata
- `documentation/harmonyos-guides/03_application-framework/arkts-ui-widget-event-router.md` - `postCardAction` with `action: 'router'` and the `want.parameters.params` string
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-ui-widget-event-router
- `documentation/harmonyos-guides/03_application-framework/arkts-ui-widget-update-by-status.md` - provider-driven card refresh
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-ui-widget-update-by-status
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `removePreferencesFromCacheSync`, `flush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, used for the add-card panel
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `NEWS-18` - the same reader shell and `BottomView`, without the widget
