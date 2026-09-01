---
id: FIN-07
title: Loan calculator - repayment schedules for four methods
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
sample: huawei_industry_tree/07_finance_insurance/downloads/LoanCalculator.zip
kits: ["@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [Navigation, NavPathStack, NavDestination, onReady, TextInput.inputFilter, onWillInsert, InsertValue, TextPicker, bindSheet, List, ForEach, Scroller]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-07-0028, HW-07-0029, HW-07-0040, HW-07-0041, HW-07-0042, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to **turn a few numeric inputs into a long
computed schedule** - loan repayments, instalment plans, deposit interest,
amortisation of any kind. Two things transfer beyond the finance case: the
input form with per-keystroke validation, and the money-formatting rules that
this sample gets wrong in ways worth learning from.

## Feature checklist

- Four inputs: amount (text), term (picker), annual rate (text), method (picker).
- The Calculate button activates only when all four are filled.
- Per-keystroke validation: at most two decimals, no leading zero, amount and
  rate capped.
- A detail page showing a summary card plus a per-month instalment table.
- Four repayment methods: 等本等息 flat, 等额本息 annuity, 等额本金 equal
  principal, 先息后本 interest-first.

## Architecture

One module, two pages, one calculation function.

```
entry/src/main/ets/
├── constant/Constants.ets      LoanConstants / CommonConstants / StyleConstants / ToastConstants
├── model/LoanInfo.ets          amount, installments (years), yearRate, repaymentMethod
├── model/MonthRepayInfo.ets    month, monthlyAmount, monthlyBase, monthlyInterest
├── pages/BasePage.ets          @Entry - the input form
├── pages/DetailPage.ets        NavDestination - summary + schedule
└── util/Util.ets               calEqualBaseInterest, formatNumber, insertStr, showToast
```

Navigation carries the whole `LoanInfo` as the route parameter, and the
destination reads it in `onReady`:

```typescript
this.pathStack.pushPathByName('DetailPage', userLoanInfo);
// ...
}.onReady((context: NavDestinationContext) => {
  this.pathStack = context.pathStack;
  this.mLoanInfo = context.pathInfo.param as LoanInfo;
  this.mTotalInterest = calEqualBaseInterest(this.mLoanInfo, this.mMonthInfoArr);
})
```

## Implementation steps

1. **Model the inputs as one interface** and push it as the route parameter, so
   the detail page has no state of its own to keep in sync.
2. **Validate in `onWillInsert`, not `onChange`** - rejecting a keystroke before
   it lands avoids the flicker of accepting then reverting.
3. **Compute the schedule once in `onReady`**, not in `build`.
4. **Branch the four methods inside one loop** over the total month count.
5. **Format money in one place**, and get the rounding right (`HW-07-0028`).
6. **Render the schedule with `LazyForEach`** - it can be 240 rows
   (`HW-07-0042`).

## Verified snippets

From `LoanCalculator.zip`. Corrected forms are marked.

**Per-keystroke validation — `pages/BasePage.ets`** (as shipped)

```typescript
TextInput({ placeholder: ... })
  .type(InputType.NUMBER_DECIMAL)
  .inputFilter('^\\d*\\.?\\d{0,2}$', () => {
    showToast(this.getUIContext(), ToastConstants.TIP_MAX_TWO_DECIMAL);
  })
  .onWillInsert((info: InsertValue) => {
    if (info.insertOffset === 0 && info.insertValue === '0') {
      showToast(this.getUIContext(), ToastConstants.TIP_NO_START_WITH_ZERO);
      return false;                                    // keystroke rejected
    }
    // build the prospective value and test *that*, not the current one
    let willLoanAmount = Number(insertStr(this.filledLoanAmount, info.insertValue, info.insertOffset));
    if (willLoanAmount > MAX_LOAN_AMOUNT) {
      showToast(this.getUIContext(), ToastConstants.TIP_NO_BILLION);
      return false;
    }
    return true;
  })
```

**This is the reusable idea.** `inputFilter` handles the *shape* with a regex;
`onWillInsert` handles the *range*, which a regex cannot express. The helper
`insertStr(old, ins, idx)` builds the text as it would be after the insert, so
the bound is tested against the value the user is about to have rather than the
one they currently have — which matters because the caret may be anywhere.

**Filled-state check — same file** (corrected, see `HW-07-0041`)

```typescript
@State filledLoanAmount: string = '';
@State filledYearRate: string = '';        // FIX: shipped code initialises this to
                                           // CommonConstants.PLEASE_SELECT, the picker
                                           // placeholder, although the control is a TextInput
@State filledInstallment: string = CommonConstants.PLEASE_SELECT;   // genuine picker
@State filledRepayMethod: string = CommonConstants.PLEASE_SELECT;   // genuine picker

private updateFilledState() {
  let isFilledAmount = this.filledLoanAmount !== '';
  let isFilledYearRate = this.filledYearRate !== '';                // FIX: shipped code compares
                                                                    // against PLEASE_SELECT
  let isFilledInstallment = this.filledInstallment !== CommonConstants.PLEASE_SELECT;
  let isFilledRepayMethod = this.filledRepayMethod !== CommonConstants.PLEASE_SELECT;
  this.isAllFilled = isFilledAmount && isFilledInstallment && isFilledYearRate && isFilledRepayMethod;
}
```

**Text fields are checked for emptiness; pickers are checked against their
placeholder.** Mixing the two is the defect in `HW-07-0041` - a text field can
never return to the placeholder value once typed in, so the check passes for an
empty field and `Number('')` silently becomes `0`.

**The four repayment methods — `util/Util.ets`** (as shipped)

```typescript
let totalMonthNum = loanInfo.installments * 12;
let monthRate = loanInfo.yearRate / 12 / 100;
let repaidBase = 0;

let monthBase = loanInfo.loanAmount / totalMonthNum;   // 等本等息 / 等额本金 default
let monthInterest = loanInfo.loanAmount * monthRate;   // 等本等息 / 先息后本 default
let monthRepayAmount = monthBase + monthInterest;      // 等本等息 default

for (let i = 1; i <= totalMonthNum; i++) {
  switch (loanInfo.repaymentMethod) {
    case LoanConstants.RM_BASE_EQUAL_INTEREST:   // 等额本息 - level payment
      if (i === 1) {
        let monthRateRepeat = Math.pow((1 + monthRate), totalMonthNum);
        let repayScaleFactor = monthRateRepeat / (monthRateRepeat - 1);
        monthRepayAmount = loanInfo.loanAmount * monthRate * repayScaleFactor;
      }
      monthInterest = (loanInfo.loanAmount - repaidBase) * monthRate;
      monthBase = monthRepayAmount - monthInterest;
      break;
    case LoanConstants.RM_EQUAL_BASE:            // 等额本金 - declining payment
      monthInterest = (loanInfo.loanAmount - repaidBase) * monthRate;
      monthRepayAmount = monthBase + monthInterest;
      break;
    case LoanConstants.RM_INTEREST_FIRST:        // 先息后本 - interest only, then balloon
      monthBase = i === totalMonthNum ? loanInfo.loanAmount : 0;
      monthRepayAmount = monthBase + monthInterest;
      break;
    // 等本等息 needs no case - the initial values already describe it
  }
  totalInterest += monthInterest;
  repaidBase += monthBase;
  monthRepayInfos.push({ monthlyAmount: monthRepayAmount, month: i,
                         monthlyBase: monthBase, monthlyInterest: monthInterest });
}
```

Note the shape: the three variables are pre-loaded with the flat-interest case,
so the fourth method needs no branch at all, and the annuity factor is computed
once on the first iteration rather than every month.

**Money formatting** (corrected, see `HW-07-0028`, `HW-07-0029`)

```typescript
// FIX: shipped code is `Math.ceil(val * 100) / 100 + ''`, which rounds every
// amount up and drops trailing zeros. Two consequences it produces today:
//   0.1 + 0.2  -> '0.31'      (float noise promoted to a whole cent)
//   910616.19  -> '910616.2'  (the promised two decimals are not applied)
export function formatNumber(val: number | undefined): string {
  if (val === undefined) {
    return '';
  }
  return (Math.round(val * 100) / 100).toFixed(2);
}
```

For anything a user will reconcile against a bank statement, do the arithmetic
in integer cents and format only at the edge.

**The summary headline** (corrected, see `HW-07-0040`)

```typescript
// FIX: shipped code always shows mMonthInfoArr[0].monthlyAmount under the label
// 每月应还, which is true only for the two level-payment methods. Under 等额本金
// the payment declines; under 先息后本 the final month carries the whole principal.
Text(LoanConstants.REPAY_MONTHLY)
Text(this.isLevelPayment()
  ? formatNumber(this.mMonthInfoArr[0].monthlyAmount)
  : `${formatNumber(this.mMonthInfoArr[0].monthlyAmount)} - ${formatNumber(this.lastAmount())}`)
```

**The schedule list** (corrected, see `HW-07-0042`)

```typescript
// FIX: shipped code is ForEach(this.mMonthInfoArr, ...) with no key generator,
// building up to 240 ListItems and their children up front.
List({ scroller: new Scroller() }) {
  LazyForEach(this.scheduleSource, (item: MonthRepayInfo) => {
    ListItem() {
      Row() {
        this.ListItemText(item.month.toString(), StyleConstants.LIST_FIRST_COLUMN_WIDTH);
        this.ListItemText(formatNumber(item.monthlyAmount), StyleConstants.LIST_COLUMN_WIDTH);
        this.ListItemText(formatNumber(item.monthlyBase), StyleConstants.LIST_COLUMN_WIDTH);
        this.ListItemText(formatNumber(item.monthlyInterest), StyleConstants.LIST_COLUMN_WIDTH);
      }.width(StyleConstants.MATCH_PARENT).height(StyleConstants.LIST_ROW_HEIGHT);
    };
  }, (item: MonthRepayInfo) => item.month.toString());
}
.cachedCount(10)
.divider({ strokeWidth: '1px', color: '#33000000' });
```

## Permissions & config

**None.** The sample declares no permissions - everything is local arithmetic.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Term picker offers 1, 2, 3, 4, 5, 10, 15 and 20 years - so up to 240 rows.
- Amount capped at 99,999,999; annual rate capped at 100.
- `installments` is a **year** count in `LoanInfo`, multiplied by 12 inside the
  calculation. The picker labels are strings like `'20年'`, parsed with
  `.split('年')[0]`.
- The calculation fills its second argument by `push` rather than returning an
  array, so calling it twice on the same array appends a second schedule.

## Pitfalls

- **`HW-07-0028` — every money figure is rounded up.** `Math.ceil(val * 100) / 100`
  is deliberate per its docstring, so a schedule is systematically overstated,
  and float noise becomes real money: `0.1 + 0.2` formats as `'0.31'`.
- **`HW-07-0029` — the two promised decimals never appear.** The same function
  concatenates a number, so `910616.19` renders as `910616.2` and consecutive
  rows in the table have different widths.
- **`HW-07-0040` — the headline is the first instalment, not the monthly
  payment.** For 先息后本 over 20 years at 4.9% on 1,000,000 it reads 4083.33
  while the final month is 1,004,083.33.
- **`HW-07-0041` — the annual rate is validated against a picker placeholder.**
  Type a rate, clear the field, and the app calculates at 0% with zero interest.
- **`HW-07-0042` — the schedule uses a keyless `ForEach`,** building every one
  of up to 240 rows before the first screenful is drawn.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `LazyForEach`, `IDataSource`, `cachedCount`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` keys
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `inputFilter`, `onWillInsert`, `InsertValue`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textpicker.md` - `TextPicker`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textpicker
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-jump.md` - `pushPathByName`, `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-jump
- `FIN-04` - the income/expense calendar, for the other date-and-money page in this industry
- `FIN-10` - the expenditure pie chart, which duplicates its data the same way
