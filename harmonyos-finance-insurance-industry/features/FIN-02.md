---
id: FIN-02
title: Key scenario index for the finance and insurance industry
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/02_financial-insurance-v1-7_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/financial-insurance-v1-7_1-0000002264232984
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Do not load this card on its own. It is the industry's scenario index: it
records which feature-level practices Huawei publishes for finance and
insurance, so a router or an agent picking a card knows the full set. Load
`FIN-01` for the architecture, then the card for the screen being built.

## Feature checklist

Huawei publishes eight feature-level practices for this industry - the largest
feature set of any industry reviewed so far:

| Card | Scenario | Sample |
|---|---|---|
| `FIN-03` | Stock intraday and daily K-line charts (股票行情走势分时线与日K线) | `StockChartX.zip` |
| `FIN-04` | Daily income and expenditure calendar (每日收支日历图) | `CalendarOfIncomeAndExpense.zip` |
| `FIN-05` | Custom stock-code keyboard (自定义股票键盘) | `StockKeyboard.zip` |
| `FIN-06` | App lock and data encryption (用户登录验证-自定义应用锁、数据加密) | `CustomAppLock.zip` |
| `FIN-07` | Loan calculator (贷款计算器) | `LoanCalculator.zip` |
| `FIN-08` | One-tap bank card number copy via Pasteboard (Pasteboard实现一键复制银行卡号) | `CardNumberCopy.zip` |
| `FIN-09` | Randomised-order keyboard (随机乱序键盘设置) | `OutOrderKeyboard.zip` |
| `FIN-10` | Expenditure distribution pie chart (绘制支出分布饼图) | `ExpenditurePieChart.zip` |

Three of the eight are security practices - the app lock, the randomised
keyboard, and the clipboard copy - which is what marks this industry out.
`FIN-01` adds a fourth with its fixed-layout secure keyboard.

Everything else in a finance app - the tab shell, the product catalogue, policy
and claim management, identity verification - is covered by `FIN-01`.

## Architecture

Not applicable. The source document is an eight-entry link list with no
technical content of its own.

## Implementation steps

None. Follow `FIN-01` first, then the relevant feature card.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

The published set is presentation- and input-heavy. Payment, transfer, biometric
authentication, certificate pinning and secure key storage have no HQ practice
document in this industry, even though `FIN-01`'s architecture assumes an
authenticated user throughout.

## Pitfalls

No defects found in this document. All eight links resolve to the documents they
name, and each link's text matches the `title` frontmatter of its target
exactly.

One thing worth knowing if you script against these documents: this page's slug
is `financial-insurance-v1-7_1`, which does **not** follow the
`practice-<industry>-app-architecture-v1_1` shape that seventeen of the nineteen
industries use. It names its own industry correctly, unlike the public transport
index, which is served from `financial-insurance-v1-6_1` (`HW-06-0017`).

## References

- `huawei_industry_tree/07_finance_insurance/docs/03_stock_chart_x.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_chart_x-0000002264336070
- `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- `huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
- `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
