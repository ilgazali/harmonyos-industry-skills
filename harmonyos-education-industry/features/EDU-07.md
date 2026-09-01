---
id: EDU-07
title: Swipeable question practice - Swiper over LazyForEach with a shared answer sheet
industry: 04_education
doc: huawei_industry_tree/04_education/docs/07_english_practice.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
sample: huawei_industry_tree/04_education/downloads/EnglishPractice.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit", "@kit.LocalizationKit"]
apis: [Swiper, SwiperController, showNext, showPrevious, cachedCount, LazyForEach, IDataSource, DataChangeListener, notifyDataReload, notifyDataChange, "@Provide", "@Consume", "@Prop", "@Watch", "@StorageProp", Radio, RadioIndicatorType, PromptAction, showToast, "resourceManager.getRawFileContentSync", RelativeContainer, alignRules, Navigation, NavDestination]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0043, HW-04-0044, HW-04-0045, HW-04-0046, HW-04-0047, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **question-at-a-time practice or exam screen**: swipe or
tap to move between items, choose an option, submit, get feedback, and have the
answers survive moving back and forth.

Two things make it worth reading beyond the obvious `Swiper`:

- **`Swiper` is one of only five containers that honour `cachedCount`** with
  `LazyForEach` (the others being `List`, `ListItemGroup`, `Grid` and
  `WaterFlow`). Everything else materialises the whole data source, so a
  thousand-question bank in a `Stack` is a thousand built subtrees.
- The sample demonstrates, by getting it wrong, **the central rule of
  `LazyForEach`**: a child is refreshed only when its *key* changes. State that
  lives outside the data source cannot reach a `LazyForEach` child, no matter
  how observable it is.

## Feature checklist

- Questions load from a rawfile JSON at page entry.
- One question per page in a `Swiper`, non-looping, no indicator.
- Swipe left/right, or the 上一题 / 下一题 buttons, move between questions.
- Selecting an option records the answer; returning to the question shows it
  still selected.
- 提交答案 is enabled only while the question is unanswered-and-unsubmitted, and
  greys out afterwards.
- Submitting shows a green 回答正确 or red 回答错误 toast, outlines the chosen
  option, and locks the radios.
- A running 今日已刷 N 题 counter on the page header and the home page.

## Architecture

One `entry` module, two pages, plus a reusable data source.

```
entry/src/main/ets
├── common
│   ├── DataModel.ets            Question/TabData/ListData + NO_ANSWER, NOT_SUBMITTED, SUBMITTED
│   └── QuestionItem.ets         one question: stem + radio options
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── MainPage.ets             @Entry - Navigation host, owns the shared state
│   └── ExamPage.ets             the Swiper, the three buttons
└── utils
    ├── CommonDataSource.ets     generic IDataSource implementation
    └── DataUtils.ets            rawfile JSON -> Question[]
```

The documented tree matches the zip exactly.

**State is provided from the host page, not from the exam page.** `MainPage`
declares four `@Provide`s and `ExamPage`/`QuestionItem` `@Consume` them:

| Provided | Meaning |
| --- | --- |
| `pageInfos: NavPathStack` | the navigation stack |
| `count: number` | questions answered today - shown on both pages |
| `myAnswer: number[]` | the answer sheet, one 1-based option index per question |
| `isSubmitted: number[]` | per-question state: unset / `NOT_SUBMITTED` / `SUBMITTED` |

That split is deliberate and mostly right: the answer sheet outlives the exam
page, so it belongs to the host. **The mistake is that `isSubmitted` also drives
what a `LazyForEach` child renders** - and that is the one kind of state a
`LazyForEach` child cannot receive (`HW-04-0043`).

**`CommonDataSource<T>` is the generic `IDataSource` worth lifting.** It is a
complete implementation - `addData`, `pushData`, `deleteData`, `changeData`,
`setData`, `refreshDataByIndex`, listener registration - and each mutator calls
the matching `notify*`. Copy this file into any project that needs
`LazyForEach`; the version in this zip is the most complete one in the industry.

## Implementation steps

1. **Implement `IDataSource` once**, generically, with a `listeners` array and a
   `notify*` call in every mutator. Do not reassign the data source object after
   binding it - the guide is explicit that reassigning `dataSource` throws.
2. **Load the questions in `aboutToAppear`** with
   `resourceManager.getRawFileContentSync` + `buffer.from(...).toString()` +
   `JSON.parse`, then `setData`, which fires `notifyDataReload`.
3. **Size the answer arrays to the question count** at the same moment, filled
   with `NO_ANSWER`. The sample leaves them empty, so every guard written
   against `0` sees `undefined` (`HW-04-0044`).
4. **Wrap `LazyForEach` in a `Swiper`** with `.cachedCount(2)`, `.loop(false)`,
   `.indicator(false)` and `.index(this.index)` bound to a `@State` the
   `onChange` writes back.
5. **Key on something unique and cheap** - a question id, or `index` folded into
   the key. Not the stem (`HW-04-0046`).
6. **Put per-question mutable state inside the `Question` objects**, update it
   through `changeData(index, updated)` and make the key generator depend on it,
   so the child is actually refreshed. Anything else - an `@Consume` array, a
   `@Prop`, a `@Watch` - will not reach the child (`HW-04-0043`).
7. **Drive paging from the `SwiperController`**: `showPrevious()` /
   `showNext()`. Do not assign to the bound `index` for this; the controller
   animates, the assignment jumps.
8. **Mark `QuestionItem` `@Reusable`.** `LazyForEach` supports node reuse and a
   question card is exactly the uniform, high-churn subtree reuse is for.

## Verified snippets

All snippets are from `EnglishPractice.zip`. Corrected forms are marked.

**The Swiper — `entry/src/main/ets/pages/ExamPage.ets`** (corrected, see `HW-04-0043`, `HW-04-0046`)

```typescript
private testData = new CommonDataSource<Question>();
private examPaperSwiperController: SwiperController = new SwiperController();
@State index: number = 0;

aboutToAppear(): void {
  const listData: Question[] = getDataFromJSON(this.context, 'singlequestion.json');
  this.testData.setData(listData);
  this.myAnswer = new Array<number>(listData.length).fill(NO_ANSWER);      // FIX: sample leaves
  this.isSubmitted = new Array<number>(listData.length).fill(NOT_SUBMITTED); // both empty
}

Swiper(this.examPaperSwiperController) {
  LazyForEach(this.testData, (item: Question, index: number) => {
    QuestionItem({ index, item, onSelectOption: () => { /* ... */ } })
  }, (item: Question, index: number) => `${index}-${item.select}-${item.submitted}`)
  // FIX: sample keys on item.title alone, which never changes - so the child never refreshes
}
.index(this.index)
.cachedCount(2)
.indicator(false)
.loop(false)
.onChange((event: number) => { this.index = event; })
```

**The key is the refresh trigger, and that is the whole lesson here.** The
guide states it twice: *"LazyForEach uses the generated key value to determine
whether to refresh child components. Components with unchanged key values will
not be refreshed"*, and *"ensure the onDataChange API of DataChangeListener
generates new key values different from previous ones to trigger component
re-rendering."*

So notifying the data source is **not sufficient** while the key is constant,
and passing state through `@Prop` from outside the data source does **nothing**.
The sample does the latter and the right/wrong toast never fires.

`cachedCount(2)` keeps the neighbours built so a swipe is instant; `loop(false)`
stops question 10 wrapping to question 1, which would misrepresent progress.

**The three controls — same file** (corrected, see `HW-04-0045`)

```typescript
Button({ type: ButtonType.Capsule }) { Text($r('app.string.Previous_question')) }
  .onClick(() => { this.examPaperSwiperController.showPrevious(); })

Button({ type: ButtonType.Capsule }) { Text($r('app.string.submit_answer')) }
  // FIX: sample reads .enabled(this.myAnswer[this.index] === 0 || ...),
  //      which enables submit precisely when nothing has been chosen
  .enabled(this.isSubmitted[this.index] !== SUBMITTED && this.myAnswer[this.index] !== NO_ANSWER)
  .onClick(() => {
    this.isSubmitted[this.index] = SUBMITTED;
    this.count++;
  })

Button({ type: ButtonType.Capsule }) { Text($r('app.string.Next_question')) }
  .onClick(() => { this.examPaperSwiperController.showNext(); })
```

**`showPrevious`/`showNext` versus assigning `index`.** Both move the `Swiper`;
only the controller runs the page transition. The `@State index` exists to
*read* the current page (for the enable conditions and the header), and
`onChange` is what keeps it in step with a swipe - it is an output, not the
input.

**Why the button appears to work while the card does not.** `.enabled()` is
evaluated in `ExamPage`'s own `build`, so it does react to `isSubmitted`
changing. The `QuestionItem` next to it does not, because it sits behind
`LazyForEach`'s key check. Same state, same page, two different refresh rules -
that asymmetry is what makes `HW-04-0043` easy to miss in testing.

**One question — `entry/src/main/ets/common/QuestionItem.ets`** (corrected, see `HW-04-0044`)

```typescript
@Reusable                                   // FIX: absent in the sample
@Component
export struct QuestionItem {
  @Consume myAnswer: number[];
  public index: number = 0;
  @State item: Question = { title: '', key: 0, option: [], analysis: '', select: 0 };
  @State selected: number = 0;
  onSelectOption: VoidCallback = () => {};
  private tagList: string[] = ['A', 'B', 'C', 'D', 'E'];

  aboutToAppear(): void {
    const previous = this.myAnswer[this.index];
    if (previous !== undefined && previous !== NO_ANSWER) {   // FIX: sample tests !== 0 only,
      this.selected = previous;                               // and assigns undefined into a number
    }
  }

  build() {
    Column() {
      Text((this.index + 1) + '.' + this.item.title)

      Column() {
        ForEach(this.item.option, (option: string, i: number) => {
          Row() {
            Radio({ value: `${i}`, group: 'radioGroup' + this.index,
                    indicatorType: RadioIndicatorType.TICK })
              .enabled(this.isSubmitted !== SUBMITTED)
              .checked(this.selected === i + 1)
              .onChange((isChecked: boolean) => {
                if (isChecked) {
                  this.myAnswer[this.index] = i + 1;   // 1-based: 0 means "no answer"
                  this.selected = i + 1;
                  this.onSelectOption();
                }
              })
            Text(this.tagList[i] + '.')
            Text(option)
          }
          .border({
            width: this.isSubmitted === SUBMITTED && this.selected === i + 1 ? 1 : 0,
            color: this.selected === this.item.key ? Color.Green : Color.Red
          })
        })
      }
    }
  }
}
```

**`group: 'radioGroup' + this.index` is load-bearing.** Radios are grouped by
name across the whole page, and `cachedCount(2)` means three questions' radios
are alive at once. A shared group name would let selecting an option on question
3 clear question 2's selection. Deriving the group from the question index keeps
each card's radios independent.

**Answers are 1-based on purpose.** `NO_ANSWER = 0` is the sentinel, so
`i + 1` on write and `selected === i + 1` on read. The padding trick on the
selected row (`17` instead of `18`) absorbs the 1 vp border so the row does not
shift when it gains an outline.

**The generic data source — `entry/src/main/ets/utils/CommonDataSource.ets`** (as shipped)

```typescript
export class CommonDataSource<T> implements IDataSource {
  private listeners: DataChangeListener[] = [];
  originDataArray: T[] = [];

  totalCount(): number { return this.originDataArray.length; }
  getData(index: number) { return this.originDataArray[index]; }

  changeData(index: number, data: T): void {
    this.originDataArray.splice(index, 1, data);
    this.notifyDataChange(index);          // the hook the exam page should be using
  }

  setData(dataArray?: T[]) {
    this.originDataArray = dataArray ?? [];
    this.notifyDataReload();
  }

  registerDataChangeListener(listener: DataChangeListener): void {
    if (this.listeners.indexOf(listener) < 0) { this.listeners.push(listener); }
  }
  unregisterDataChangeListener(listener: DataChangeListener): void {
    const pos = this.listeners.indexOf(listener);
    if (pos >= 0) { this.listeners.splice(pos, 1); }
  }
}
```

**`setData` replaces the array, it does not rebind the source.** The guide warns
that "reassigning dataSource (first parameter) causes an exception" - meaning the
`LazyForEach(this.testData, ...)` expression must keep pointing at the same
object. Mutating `originDataArray` inside it and calling `notifyDataReload` is
the supported way to swap the whole contents, and it is why `testData` is a
`private` field created once at declaration rather than in `aboutToAppear`.

**Reading a rawfile — `entry/src/main/ets/utils/DataUtils.ets`** (as shipped)

```typescript
import { buffer } from '@kit.ArkTS';

export function getDataFromJSON(context: Context, fileName: string) {
  let result: Question[] = [];
  try {
    const value: Uint8Array = context.resourceManager.getRawFileContentSync(fileName);
    const str = buffer.from(value.buffer).toString();
    result = JSON.parse(str) as Question[];
  } catch (err) {
    hilog.error(0x0000, 'testTag', `err msg is ${err.message}`);
  }
  return result;
}
```

`getRawFileContentSync` returns a `Uint8Array`; `buffer.from(value.buffer)`
wraps the underlying `ArrayBuffer` and `toString()` decodes it as UTF-8. The
failure path returns an empty array, which renders an empty `Swiper` with no
message - acceptable for a sample, worth a visible error state in an app.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block. The
question bank is a rawfile inside the HAP:

```
entry/src/main/resources/rawfile/singlequestion.json
```

Ten single-choice items, each `{ title, key, option[], analysis, select }`,
where `key` is the 1-based index of the correct option.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`cachedCount` only works inside `List`, `ListItemGroup`, `Grid`, `Swiper`
  and `WaterFlow`.** In any other container `LazyForEach` builds everything at
  once, and the lazy loading is nominal.
- The bank is a bundled rawfile: fixed at build time, no paging, no server.
- The `analysis` field is parsed and never displayed - the explanation the data
  carries is not shown after submitting.
- `ExamPage.aboutToDisappear` clears `myAnswer` and `isSubmitted` but leaves
  `count`, so leaving the exam discards the answer sheet while the day's tally
  survives.
- Questions have at most five options (`tagList` is `A`-`E`); a sixth would
  index past the end.
- `EntryAbility` registers `avoidAreaChange` and never releases it - the same
  boilerplate defect recorded as `HW-04-0008` in `EDU-01`.

## Pitfalls

- **`HW-04-0043` — `isSubmitted` cannot reach a `LazyForEach` child.** The key
  is `item.title`, which never changes, so the child is never refreshed: the
  `@Prop` stays at its creation value, `@Watch('onSubmit')` never fires, the
  right/wrong toast never appears and submitted radios stay editable. This is
  the defect to understand before reusing anything on this page. State that
  drives a `LazyForEach` child must live in the data source **and** appear in
  the key.
- **`HW-04-0044` — the answer arrays are provided empty,** so `myAnswer[index]`
  is `undefined` rather than `NO_ANSWER`. `QuestionItem.aboutToAppear`'s
  `!== 0` guard therefore passes and assigns `undefined` into a field declared
  `number`.
- **`HW-04-0045` — the submit button is enabled when no option is selected.**
  `myAnswer[this.index] === 0` means "unanswered"; it is used as a reason to
  enable. It is masked today only by the arrays being empty - fix `HW-04-0044`
  first and this one becomes live.
- **`HW-04-0046` — the key is the whole question stem.** Unique in these ten
  items, not guaranteed in a real bank, and expensive to hash. Fold in an id or
  the index.
- **`HW-04-0047` — the avoid areas are bound with `@StorageLink`,** a two-way
  binding, where every other sample in this industry uses `@StorageProp`. Nothing
  writes them today; a later edit that did would corrupt the shared value
  app-wide.
- **`QuestionItem` is not `@Reusable`.** A uniform card cycling through a
  `Swiper` is the canonical reuse case, and `LazyForEach` supports it directly.
- **Do not reassign the data source object.** `LazyForEach(this.testData, ...)`
  must keep the same instance for the component's life; swap the contents
  through `setData`.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - key generation rules, "components with unchanged key values will not be refreshed", duplicate-key behaviour, and the `cachedCount` container list
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, `DataChangeListener`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `SwiperController`, `showNext`, `showPrevious`, `cachedCount`, `loop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-radio.md` - `group`, `RadioIndicatorType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-radio
- `documentation/harmonyos-guides/03_application-framework/arkts-reusable.md` - `@Reusable` with `LazyForEach`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-reusable
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - `@Provide`/`@Consume`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `EDU-01` - the exam page in the framework sample, which has the same "state written next to the component that renders it" problem in a different form
