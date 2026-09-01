# Pitfalls

> Generated from `features/*.md`. Source industry: `07_finance_insurance`, 11 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (2)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-07-0054` - Systematic: 2 sample projects declare permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: FIN-08, FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-07-0053` - Systematic: 8 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: FIN-03, FIN-04, FIN-05, FIN-06, FIN-07, FIN-08, FIN-09, FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (52)

### `HW-07-0023` - The RSA private key that protects the app-lock password is hardcoded in the source of a published sample

- Category D, severity blocker, confidence confirmed
- Features: FIN-06
- Document: `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- Why: skData is a complete PKCS#8 RSA private key written out as a byte array literal, and genKeyPairByData reconstructs the pair from it on every encrypt and every decrypt. The key therefore ships inside every build made from this sample, recoverable by anyone who can read the application package. That alone defeats the feature: the app lock exists to stop someone holding the device from reading the account balances behind it, and the ciphertext sits in a plain preferences file in the sandbox, so an attacker who can reach the file also has the key to open it. Because this is a published sample, the situation is worse than a private hardcoded key - this exact pair is public, so every application built on it that does not replace the constants shares one key that anyone can look up. HarmonyOS provides the Asset Store Kit and the Universal Keystore precisely so that key material never has to appear in code.
- Fix: Remove both literals. Generate the pair with cryptoFramework at first run and store the private key through the Universal Keystore, or drop asymmetric encryption entirely in favour of the salted hash described in the related finding.

### `HW-07-0043` - The letter keyboard drops three letters of the alphabet on every shuffle because the row slice bounds are off by one

- Category B, severity blocker, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: Array.slice excludes its end index, so the three ranges cover indices 0-8, 10-17 and 19-24 and leave 9, 18 and 25 out entirely. sliceAbcKeyboardData is called from resortABCKeyboardData, which aboutToAppear calls before the keyboard is first drawn, so the loss happens immediately and on every reshuffle - the initial hardcoded rows that do contain all 26 letters are overwritten before anything is displayed. Because the array is shuffled first, the three excluded letters are different each time and cannot be predicted: a user whose password contains one of them simply cannot type it, and the only recourse is to switch to another keyboard type and back to force a new shuffle. The row lengths also drop from the standard 10 / 9 / 7 to 9 / 8 / 6, so the defect is visible in the layout as well. This is the reference implementation of a secure keyboard for a login screen, where an untypeable password is a complete block on the feature.
- Fix: Set the slice bounds to the row lengths: slice(0, 10), slice(10, 19), slice(19, 26). Better, derive them from the row sizes - const rows = [10, 9, 7] - so the three constants cannot drift apart from the data again, and assert that the rendered key count equals abcMenuData.length.

### `HW-07-0001` - The recognised national ID number and real name are written to the log with debug prefixes and no privacy marking

- Category D, severity high, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: The two hilog calls fire immediately after the card-recognition flow returns, so what they write is the identity-document number and the legal name Vision Kit just read from the user's ID card - the most sensitive pair of values this application handles. Three separate faults compound. The values are concatenated into hilog's third argument, which is the format string, so they carry no %{public} or %{private} marker at all - and the private marker is the mechanism by which hilog redacts a value in release logs, so without it the number is written in clear. The tag is an empty string, so the entries cannot even be filtered out afterwards. And both messages are prefixed 'test ABC' and 'test ABCD', which marks them as debugging scaffolding that was never removed. The same pattern appears in MainPage, where the whole location object and the whole reverse-geocoded address - the user's street address - are serialised into info-level logs. The document's own privacy notice on this screen promises 我司将采用完善的安全措施确保您的信息安全.
- Fix: Delete both hilog calls. If a diagnostic is genuinely needed, log only that recognition succeeded, never the values; where a value must appear, pass it as an argument marked %{private}s and give the entry a real tag. Reduce the location logging to coordinates-free status information.

### `HW-07-0002` - Screenshot protection covers the login page but not the identity screens that display the ID card number

- Category D, severity high, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: The application declares ohos.permission.PRIVACY_WINDOW and ships a WindowUtils helper for setWindowPrivacyMode, and applies it correctly to the login page - so the capability, the permission and the intent are all present. It is then not applied to the two screens that matter more. AuthenticationPage renders the recognised ID number and real name as plain Text, and CredentialsPage hosts a live CardRecognition camera view pointed at the user's identity document. Both are screenshottable and both appear in the recent-tasks thumbnail, while the login form - which shows a masked password field - is protected. For an insurance application whose stated purpose for collecting this data is 办理保安理赔, the protection is applied to the least sensitive of the three screens.
- Fix: Call WindowUtils.setWindowPrivacyModeInPage(context, true) from aboutToAppear and false from aboutToDisappear on AuthenticationPage and CredentialsPage, mirroring LoginPage.

### `HW-07-0012` - The daily K-line branch is guarded by the intraday index variable, so one lookup decides the other chart's data

- Category B, severity high, confidence confirmed
- Features: FIN-03
- Document: `huawei_industry_tree/07_finance_insurance/docs/03_stock_chart_x.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_chart_x-0000002264336070
- Why: The guard on the second block reads assumeCurrentTimeLineIndex, the result of the intraday time lookup performed nine lines earlier, rather than assumeDailyKLineIndex, which is computed on the line immediately above it. The two lookups can disagree, and each way of disagreeing produces a wrong chart. If the configured date is absent from the daily data but the configured time is present in the intraday data, the branch is taken with assumeDailyKLineIndex at -1: currentDailyKLineIndex becomes -1 and slice(0, 0) yields an empty array, so the daily K-line chart renders blank and the fetch timer then rebuilds it one candle at a time from index 0. If the date is present but the time is not, the else branch runs and copies the entire data set, so the chart shows candles dated after the moment the page is meant to represent. Both inputs come from the init_time and init_date string resources, so a developer adapting the sample changes exactly the two values that trigger this.
- Fix: Test the matching variable: if (assumeDailyKLineIndex !== -1). While there, make the two guards consistent - the first uses >= 0 and the second !== -1 for the same condition.

### `HW-07-0015` - Transactions are placed by day-of-month alone, so every month shows the same entries regardless of year or month

- Category B, severity high, confidence confirmed
- Features: FIN-04
- Document: `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- Why: The fill loop reads getDate() and nothing else - the year and month of each record are never compared against the y and m the method was called with. Every record in sourceData is therefore dropped into the currently displayed month at its day-of-month position. The shipped data is dated June 2025 while aboutToAppear opens the calendar on today, so on any device the calendar shows six June 2025 transactions spread across the current month, and paging to any other month shows the very same six again. The totals per cell, the monthly summary and the day-detail list are all built from this, so every number the screen displays is wrong. calcFlows takes y and m as parameters and uses them for the cell layout, which makes the omission in the fill loop the clear defect rather than a design choice.
- Fix: Filter before placing: const d = this.sourceData[i].payDate; if (d.getFullYear() !== y || d.getMonth() !== m - 1) { continue; } then index by d.getDate().

### `HW-07-0016` - Two cell lookups index the array without bounds checks, so a day number larger than the month crashes the page

- Category B, severity high, confidence confirmed
- Features: FIN-04
- Document: `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- Why: Both expressions compute a cell index as day + startMissingDays - 1 and dereference it immediately. At the point of the fill loop the array holds startMissingDays + daysOfThisMonth entries, so any day greater than daysOfThisMonth runs off the end and the call becomes undefined.addDetails(...). Two ordinary paths reach that state. A transaction dated the 29th, 30th or 31st while February is displayed indexes past the end. And the tap handler passes item.day straight into calcFlows, where item may be a supplemental cell belonging to the previous or next month - so tapping the 31 that pads the start of a 30-day month asks for index 31 in an array that stops at 30, and the immediate .changeCellStyle() on the undefined result throws. Neither index is checked, and the second one is dereferenced twice in a row.
- Fix: Clamp and check: const idx = day + startMissingDays - 1; if (idx < 0 || idx >= thisFlows.length) { continue; } in the loop, and guard the default selection the same way before calling changeCellStyle. Clamp the tapped day to the target month's length before passing it to calcFlows.

### `HW-07-0019` - Length truncation ignores the text being replaced, so typing over a full selection discards the character

- Category B, severity high, confidence confirmed
- Features: FIN-05
- Document: `huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
- Why: The insertion replaces the selected range: the result is text[0..left] + value + text[right..]. But both the overflow test and the truncation length are computed against this.text.length, the length before the replacement, as though the insertion only ever added characters. Whenever a selection is active the calculation is therefore too pessimistic and drops characters that would have fitted. The extreme case is reachable in two taps with the sample's own limit of 8: type eight digits, select all eight, then type a nine. text.length + value.length is 9, which exceeds 8, so length becomes 8 - 8 = 0, value.substring(0, 0) is empty, and the two surrounding substrings are empty because the whole string was selected - the field is left completely blank and the digit the user pressed is gone. The same six lines are duplicated verbatim in onPaste, so pasting over a selection loses the paste the same way.
- Fix: Compute the room from what survives: const kept = this.text.length - (this.rightCaretPos - this.leftCaretPos); const room = this.maxLength - kept; then truncate value to room. Extract the shared logic so onInput and onPaste cannot drift apart.

### `HW-07-0024` - The lock password is stored reversibly and decrypted back to plaintext to be compared

- Category D, severity high, confidence confirmed
- Features: FIN-06
- Document: `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- Why: An authentication secret only needs to be verifiable, not recoverable, so encrypting it is the wrong primitive: it creates a key that must be protected forever and a plaintext that can be reconstructed. This sample reconstructs it constantly. TextLoginPage, ImageLoginPage and PassSettingPage each decrypt on entry, and MinePage decrypts both the text password and the gesture password in aboutToAppear simply to hold them in component state - so merely opening the home screen materialises both cleartext credentials in memory, with no login involved. Combined with the key being a literal in the same file, the stored value is plaintext with extra steps. The document presents this as the security design of the practice, under a heading about protecting 个人财产隐私.
- Fix: Store a salted hash: generate a random salt per user, keep it beside the digest, and verify by hashing the attempt with the same salt and comparing. cryptoFramework provides the digest primitives. Nothing in the flow needs the original password back.

### `HW-07-0025` - Credentials are persisted with Preferences rather than the Asset Store Kit provided for secrets

- Category D, severity high, confidence confirmed
- Features: FIN-06
- Document: `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- Why: Preferences is a lightweight key-value store backed by an ordinary file in the application sandbox - it offers persistence, not protection, and the guidance describes it for application configuration rather than for secrets. HarmonyOS ships a separate component for this case: the Asset Store Kit stores short, sensitive items such as passwords and tokens, binds them to the device, and can gate access on user authentication. Using Preferences here means the protection of the app-lock password rests entirely on the encryption layer, whose private key is a literal in the same project - so the two decisions compound: the wrong store, secured by a key anyone can read. The parameter is even named passWord, so what is being stored was never in doubt.
- Fix: Move both credentials to the Asset Store Kit, which also removes the need to hand-roll encryption for them.

### `HW-07-0026` - On a first run the home page decrypts the string 'null', and the absence of a password is read as a password being set

- Category B, severity high, confidence confirmed
- Features: FIN-06
- Document: `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- Why: getString returns the literal string 'null' when a key has never been written, and nothing anywhere checks for it. On the very first launch MinePage.aboutToAppear passes 'null' straight into deciphering, which runs hexStrToUint8Array over a non-hexadecimal string and hands the result to an RSA decrypt that cannot succeed. There is no try/catch in deciphering, none around the call, and aboutToAppear is async, so the failure surfaces as an unhandled rejection during the first render of the application's home screen. The two lines below compound it: 'null' is a non-empty string and therefore truthy, so isSetText and isSetImage are both set to true when no password has ever been configured - the application believes it is locked with a credential that does not exist. A sentinel that is a truthy string is the root of both halves.
- Fix: Return undefined rather than the string 'null' when a key is absent, guard each call - if (!stored) { return; } - before decrypting, and wrap deciphering in try/catch so a corrupt or missing value degrades to no-password-set instead of throwing.

### `HW-07-0028` - Every money figure is rounded up, so a floating-point artefact of one part in ten thousand billion becomes a whole extra cent

- Category B, severity high, confidence confirmed
- Features: FIN-07
- Document: `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- Why: Math.ceil rounds away from zero unconditionally, and the docstring confirms this is deliberate. Every currency value on the detail page passes through it: the headline monthly payment, the loan amount, the total interest, and all three columns of every instalment row. Two consequences. First, the schedule is systematically overstated - a borrower is shown a monthly payment and a total interest higher than the arithmetic gives, which for a repayment schedule is the wrong direction to err in. Second, ordinary binary floating-point noise is promoted to real money: reproduced with node, formatNumber(0.1 + 0.2) returns '0.31', because the sum is 0.30000000000000004 and the ceiling of 30.000000000000004 is 31. Any computed value that lands a fraction of a cent above a boundary gains a full cent, and the instalment list has one such value per month for up to 360 months.
- Fix: Round half-up on the cent: Math.round(val * 100) / 100, or better, do the arithmetic in integer cents throughout and format at the edge. State the rounding rule in the document, since for a loan calculator it is a product decision rather than a formatting detail.

### `HW-07-0030` - The password dialog verifies nothing - any six digits reveal the full card number

- Category D, severity high, confidence confirmed
- Features: FIN-08
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: The handler accumulates digits and tests only this.codeText.length === VERIFY_CODE_LENGTH. The value of codeText is never compared with anything - it is cleared on the next line and discarded. So entering any six digits at all closes the dialog, sets isValidatePass, and replaces the masked account with the full sixteen-digit card number. The document describes this component as a 密码验证弹窗 and the feature as 通过密码验证后一键复制卡号 - copying the card number after passing password verification - so the control the practice is named after does not exist. This differs from the insurance architecture practice in this same industry, which also accepts any password but says so plainly in its own text; here nothing warns the reader, so a developer adopting the pattern inherits a verification screen that verifies nothing.
- Fix: Compare the entered code against a credential before granting access - at minimum a stored salted digest as described in the app-lock card, and for a real card number a server-side check. Until then, state in the document that the dialog is UI only, as the architecture practice does.

### `HW-07-0031` - Screenshot protection is switched off on the same lines that reveal the card number

- Category D, severity high, confidence confirmed
- Features: FIN-08
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: The sample enables privacy mode when the PIN dialog opens and disables it four lines before assigning the real card number to the state the page renders. The result is exactly inverted: the six digits the user types are protected from screenshots and screen recording, and the sixteen-digit card number those digits unlock is not. The unmasked number then stays on screen, screenshottable and present in the recent-tasks thumbnail, for as long as the user remains on the page. The document highlights 在弹窗开启页面设置防截屏模式 as a feature of this practice, which makes the placement the point of the exercise rather than an incidental detail. The same misplacement appears in this industry's architecture practice, where privacy mode covers the login form but not the identity screens.
- Fix: Keep privacy mode enabled while isValidatePass is true and the account is unmasked; release it in aboutToDisappear or when the number is re-masked, not on successful entry.

### `HW-07-0034` - The key shuffle uses Array.sort with a random comparator, which is not a uniform shuffle - digit 0 lands first twice as often as chance

- Category D, severity high, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: Array.prototype.sort requires a comparator that is consistent - the same pair must always compare the same way, and the ordering must be transitive. A comparator returning a fresh random sign satisfies neither, so the permutation it yields is not uniform: the outcome depends on the sort algorithm's access pattern and leaves elements strongly correlated with their starting positions. Measured over 200,000 shuffles of a ten-element array with this exact comparator, digit 0 finishes in the first position 19.4 percent of the time instead of the 10 percent uniformity requires, digit 9 finishes in the first two positions only about 6 percent of the time each, and the largest deviation from uniform across the whole position matrix is 93.7 percent. That matters here more than it would anywhere else: the entire purpose of an out-of-order keypad is to break the link between where a finger lands and which digit was pressed, so an observer with touch positions - a shoulder surfer, a smudge pattern, a recorded video - retains a strong prior on the PIN. Using a cryptographic random source for the comparator does not repair this; the bias is in the sorting, not in the numbers.
- Fix: Replace both with Fisher-Yates: for (let i = arr.length - 1; i > 0; i--) { const j = Math.floor(this.keyboardController.createSecurityRandom() * (i + 1)); [arr[i], arr[j]] = [arr[j], arr[i]]; } - which needs exactly n-1 random draws and is uniform by construction.

### `HW-07-0040` - The headline monthly payment is the first instalment, which is not the monthly payment under two of the four repayment methods

- Category B, severity high, confidence confirmed
- Features: FIN-07
- Document: `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- Why: The summary card reads 每月应还, the amount due each month, in thirty-point bold, and fills it from index 0 of the schedule. That is correct only for the two level-payment methods. Under 等额本金 the payment declines every month, so the headline overstates the later instalments - by month 240 the borrower pays 4183.68 against a headline of 8250.00, nearly double. Under 先息后本 it is worse in the other direction: the headline shows the interest-only instalment of 4083.33 while the final month is 1004083.33, so the single largest obligation in the loan, the principal itself, is absent from the summary that is meant to summarise it. Both methods are offered in the same picker as the two the headline is valid for, and nothing on the screen tells the user which case they are in. The instalment table below carries the true figures, so the page contradicts itself.
- Fix: Branch on repaymentMethod: keep the single figure for 等本等息 and 等额本息, and for the other two show a range - first and last instalment - or relabel to 首期应还. The repayment method is already on mLoanInfo, so no new state is needed.

### `HW-07-0044` - The length cap ignores the text the insertion replaces, so typing over a full selection at the cap loses the keystroke

- Category B, severity high, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: The insertion is a replacement: it keeps the text before leftCaretPos and after rightCaretPos and drops what is between them. The cap check ignores that middle range, so a replacement is measured as though it were appended to the full existing text. Select all sixteen characters of a password at the cap and type one letter: room computes to zero, the typed character is truncated away, and the surviving text is the empty prefix plus the empty suffix, so the field is cleared and the keystroke is silently discarded. The user sees a password box that empties itself when they try to correct it. The identical logic is duplicated in onPaste, so pasting over a selection fails the same way, and fixing one place leaves the other broken. The custom stock keyboard in this same industry ships the same controller with the same defect.
- Fix: Compute room from what remains: const kept = this.text.length - (this.rightCaretPos - this.leftCaretPos); const room = this.maxLength - kept; then truncate to Math.max(0, room). Extract the replacement into one private method and have both onInput and onPaste call it.

### `HW-07-0003` - Precise location is declared in the manifest but never requested, so the app claims a permission it never uses

- Category D, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: ohos.permission.LOCATION is a user-grant permission, so declaring it does nothing on its own: it must also be requested at runtime before it has any effect. A search of the whole project finds the string 'ohos.permission.LOCATION' only in the manifest - the single runtime request array holds APPROXIMATELY_LOCATION and nothing else, and no other module requests anything. The precise permission is therefore inert code that nonetheless appears in the application's declared permission list, which is what users and app-store reviewers read. For an insurance application, declaring precise location that is never asked for is exactly the kind of over-declaration a privacy review flags, and it costs the app nothing to remove because the approximate permission is all the code uses.
- Fix: Remove ohos.permission.LOCATION from products/phone/src/main/module.json5, or request it alongside APPROXIMATELY_LOCATION if precise positioning is actually wanted.

### `HW-07-0004` - The shared inset value is provided as a number and consumed as a string by six of its seven consumers

- Category B, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: @Provide and @Consume are bound by name and are expected to agree on type. Here one provider publishes a number - the status-bar height converted to vp - and six consumers across five modules declare it as a string, while a seventh declares it correctly as a number. The value flows regardless and the pages render, because padding and margin accept a Length that may be either, so the mismatch is invisible at runtime and permanent in the source. Insurance.ets proves the author knew the right type. The practical cost is that six declarations lie about what they hold: anyone doing string work on the value - concatenating a unit, comparing, passing it where a string is required - gets a number instead, and the compiler cannot warn because the declaration says otherwise.
- Fix: Change the six string declarations to number so all eight sites agree with the provider.

### `HW-07-0005` - Card recognition handles only the success code, so a cancelled or failed scan leaves the user stranded on the scanner

- Category B, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: The callback acts on 200 and silently discards everything else. CardRecognitionResult reports cancellation, recognition failure and permission problems through the same code field, so on any of those the callback returns having done nothing at all: the scanner view stays on screen with no message, no retry prompt and no way back except the system gesture, and AuthenticationPage never learns the scan did not happen. The literal 200 is also unexplained, with no named constant and no comment saying what other values exist. The onDisAppear handler beside it is an empty function, which is where a cancellation could have been detected and is not.
- Fix: Branch on the code: pop with the result on success, pop without one on cancellation, and show the failure reason otherwise. Name the codes rather than testing a bare 200, and either implement onDisAppear or remove it.

### `HW-07-0006` - A failure to enable screenshot protection is undetectable, because neither the window lookup nor the mode change is checked

- Category B, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: Both calls return promises and neither is handled. getLastWindow can reject, and setWindowPrivacyMode requires ohos.permission.PRIVACY_WINDOW and rejects when it is not held - so if that declaration is ever dropped, or the call fails for any other reason, the login screen renders without screenshot protection and absolutely nothing in the application knows. This is a security control failing open and failing silently, in a helper whose only job is to apply it. The caller receives void, so it cannot check either. For a control of this kind the failure needs to be at least logged, and arguably needs to block the screen from showing sensitive content.
- Fix: Return the promise, await the inner call and add a catch: static async setWindowPrivacyModeInPage(...): Promise<boolean> { try { const w = await window.getLastWindow(context); await w.setWindowPrivacyMode(isFlag); return true; } catch (err) { Logger.error(...); return false; } }

### `HW-07-0007` - The secure keyboard keys are keyed by JSON.stringify of an item the click handler mutates in place

- Category C, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: Two problems that reinforce each other. The key generator serialises the whole key-attribute object, which the ForEach guidance warns against because a complex object produces a long string and consumes more memory - here on every key of a full alphabetic keyboard, on every render pass. Worse, the click handler mutates item.label directly on an element of an @Prop array, and because the mutated field is part of the serialised key, the key changes and the framework rebuilds the item. So the redraw for the caps-lock icon works only as a side effect of the anti-pattern in the key generator: fix the key to something stable and the caps-lock icon silently stops updating. Mutating an element of a one-way @Prop copy is also not a supported way to drive a re-render.
- Fix: Key on an identifying field, for example (item: IKeyAttribute) => item.value ?? item.type, and drive the caps-lock icon from the observed curKeyboardType in the builder instead of mutating item.label.

### `HW-07-0008` - The software view says phone and tablet, the hardware requirement says phone, and the manifest declares three device types

- Category E, severity medium, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: Three statements in one document give three different answers, and the manifest gives a fourth. The introduction plans for a phone; the software view adds tablet twelve lines later; the hardware requirements say Huawei phone; the shipped manifest declares phone, tablet and 2in1. The module directory is even named products/phone, which reads as a commitment to the narrowest of the four. The sample does ship a BreakpointSystem and breakpoint constants in commons/basic, so some multi-device intent is real - which makes the phone-only statements the stale ones, but a reader has no way to tell which sentence to believe.
- Fix: Settle on the scope the breakpoint code actually supports, state it once in 简介, and make 软件视图设计, 硬件要求 and the manifest agree with it.

### `HW-07-0013` - An empty catch around the whole data load leaves both charts blank with no message and no log

- Category B, severity medium, confidence confirmed
- Features: FIN-03
- Document: `huawei_industry_tree/07_finance_insurance/docs/03_stock_chart_x.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_chart_x-0000002264336070
- Why: This one try block covers both rawfile reads, both JSON parses and the initial realtime update - every input the page has. Its catch body is empty, so a missing or renamed rawfile, a malformed JSON payload, or a failure inside updateRealtimeData all produce the same outcome: both charts stay empty, the fetch timer keeps ticking against empty arrays, and nothing is written anywhere. The page renders its axes and title normally, so the failure is indistinguishable from a slow load. This is the only empty catch in an otherwise carefully written sample - the rest of the project checks its guards, and the surrounding code even uses ?. and || [] defensively on the parse results, which makes the silent catch the one gap in an otherwise complete chain.
- Fix: Log the error and surface a state: catch (err) { Logger.error(`loadInitialData failed: ${JSON.stringify(err)}`); this.loadFailed = true; } and render an error or retry affordance when loadFailed is set.

### `HW-07-0020` - A three-way guard tests one condition three times and turns a limit of zero into no limit at all

- Category B, severity medium, confidence confirmed
- Features: FIN-05
- Document: `huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
- Why: The three clauses are not three checks. typeof x === 'undefined' and x === undefined are the same test written two ways, and !x already subsumes both, so the second and third clauses can never be reached. The field is declared number with an initialiser of Infinity, so undefined is a state the type does not permit in the first place. What the guard does achieve is a real defect: !this.maxLength is also true for 0, so a caller that sets maxLength to 0 to block input entirely has it silently rewritten to Infinity and gets unlimited input - the exact opposite of what was asked for. The check also runs on every keystroke and mutates the object's own configuration from inside a handler.
- Fix: Delete the guard. If a caller might supply an invalid value, validate once where maxLength is assigned - the sample sets it in TextInputComponent - and use Number.isFinite there rather than truthiness.

### `HW-07-0021` - Every keyboard row keys its ForEach by serialising the whole key object

- Category C, severity medium, confidence confirmed
- Features: FIN-05
- Document: `huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
- Why: Five ForEach sites in the keyboard module serialise a KeyboardMenu object to build each key's identity, so a full alphabetic keyboard performs about thirty JSON.stringify calls on every render pass, each producing a string far longer than the value it identifies. The ForEach guidance in the rendering-control document warns about exactly this: when the item is a complex object, serialising it to a JSON string produces a long string that consumes more memory. A keyboard re-renders on every keystroke and on every layout change, so this is a hot path. This is the third document in this industry with the same pattern - the insurance architecture practice and this one - which suggests it is being copied between keyboard samples rather than reconsidered.
- Fix: Key on the character or icon the button represents, which is already unique within a row, and include the row index if uniqueness across rows is needed.

### `HW-07-0027` - The document's snippet computes the ciphertext and then stores the plaintext variable

- Category E, severity medium, confidence confirmed
- Features: FIN-06
- Document: `huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
- Why: The block is presented as one continuous flow - encrypt, then persist - and is the only code the document shows for either step. The encryption half computes text, the hexadecimal ciphertext. The persistence half then writes passWord, a variable the snippet never defines and which by its name holds the password itself. As printed, the sample encrypts a value and stores a different one, so a reader following the document literally persists the plaintext password. The shipped project does connect the two, through an encryption() helper whose result is passed to PreferencesUtils.setString, but the document shows neither of those functions, so the join is invisible and the printed code is wrong in the most consequential way a security snippet can be.
- Fix: Show the real call chain from the archive - encryption(...) returning the hex string, then PreferencesUtils.setString(key, that string, context) - so the value that is stored is visibly the value that was encrypted.

### `HW-07-0029` - The currency formatter does not produce two decimal places, so the repayment columns show ragged amounts

- Category B, severity medium, confidence confirmed
- Features: FIN-07
- Document: `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- Why: The function divides by 100 and concatenates, which yields a plain number-to-string conversion and drops trailing zeros - so the two decimal places its own docstring promises appear only when the value happens to need them. The detail page renders the instalment schedule as three aligned currency columns, and consecutive rows therefore differ in width and decimal count: one row reads 1234.5, the next 1234.56, a third 1235. The headline total is affected too - for the sample's own scenario the total interest of 910616.19 is displayed as 910616.2, which reads as a different, rounder number than it is. No thousands separator is applied either, so a seven-figure amount arrives as an unbroken digit run.
- Fix: Return (Math.round(val * 100) / 100).toFixed(2), and group the integer part with a thousands separator for amounts of this size.

### `HW-07-0032` - The card number is written to the system clipboard with default sharing and never cleared

- Category D, severity medium, confidence confirmed
- Features: FIN-08
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: PasteData carries a property that limits how far the content may travel - shareOption, with values restricting an item to the originating application, to the local device, or allowing it to cross devices. The sample sets none, so a bank card number lands on the system clipboard under the default policy, readable by any application that reads the clipboard and eligible for whatever cross-device clipboard the user has enabled. Nothing clears it afterwards either, so the number stays available until something else overwrites it, long after the user has finished with it. Everything else about this feature treats the card number as sensitive - a verification dialog, a masked display, an anti-screenshot mode - and then it is handed to the least controlled channel on the device with no restriction at all.
- Fix: Set the property before writing - pasteData.setProperty with shareOption limited to the local device or the application - and clear the entry afterwards, on a short timer or in aboutToDisappear, rather than leaving it resident.

### `HW-07-0035` - A fresh cryptographic random generator is constructed for every single comparison in the sort

- Category C, severity medium, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: createRandom() constructs a new cryptographic random generator object, and the function is the comparator, so it runs on every comparison the sort performs - on the order of n log n times. For the alphabetic keyboard that is roughly a hundred generator constructions and a hundred synchronous single-byte draws for one shuffle, and the shuffle re-runs every time the keyboard is raised and every time the user switches case. generateRandomSync is a blocking call into the crypto framework, and this all happens on the UI thread at the moment the keyboard is being presented, which is the one moment the user is waiting for it. A Fisher-Yates shuffle over one reused generator needs n-1 draws instead - about twenty-five for the same keyboard - and can take them in a single batched call.
- Fix: Hold one generator as a field on the controller, and draw the bytes you need in one generateRandomSync(n) call before shuffling rather than one per comparison.

### `HW-07-0036` - A minimum slice angle is set unconditionally, silently inflating any category below 2.78 percent of the total

- Category B, severity medium, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: setMinAngleForSlices forces every slice to occupy at least the given angle regardless of its value, so any category worth less than 10/360 of the total - 2.78 percent - is drawn larger than its share and every other slice is compressed to make room. The document introduces this chart as the way for a user to understand 支出分布及比重, the distribution and the proportion of their spending, which is precisely the property the setting breaks. With the six bundled values the smallest slice is 20.9 degrees, so the setting has no visible effect today and the distortion is invisible during development; it activates the moment real data contains a small category, which for a personal expenditure breakdown is the common case rather than the exceptional one. The value 10 is also a bare literal with no constant and no comment explaining the intent.
- Fix: Remove the call, or group categories below a threshold into a single Other slice and let every drawn slice keep its true angle. If a minimum is kept for legibility, note in the document that slices below the threshold are no longer to scale.

### `HW-07-0037` - The chart and the legend list are fed from two separate copies of the same hardcoded data

- Category B, severity medium, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: The same six figures are declared independently in the page and in the chart component, and each renders from its own copy - the page draws the composition list beneath the chart, the component draws the pie. Nothing connects them, so editing one leaves the pie and the list describing different spending. That is the failure mode a reader will hit first, because replacing the mock figures with real ones is the first thing anyone does with this sample, and the page is the obvious place to look. The component already takes its labels and colours as inputs, so the plumbing to pass values in exists; only this array was duplicated instead.
- Fix: Declare the array once on the page and pass it into PieChartComponent as a property alongside the labels it already receives.

### `HW-07-0038` - The composition bars are scaled against the largest category while the pie beside them is scaled against the total

- Category B, severity medium, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: Progress uses this.values[0], the first and largest entry, as its total, so each bar shows the category as a fraction of the biggest category rather than as a share of spending. The pie immediately above uses the sum. The two therefore give different answers for the same row on the same screen: the second category fills 52.5 percent of its bar while occupying 20.7 percent of the pie, and the largest category fills its bar completely while accounting for 39.5 percent of the total. A reader comparing the bar with the slice concludes one of them is wrong. Relying on index 0 also silently assumes the array is sorted descending - reorder the data and the first bar stops being full and later bars overflow their scale.
- Fix: Compute the sum once and pass it as the total: const sum = this.values.reduce((a, b) => a + b, 0); then Progress({ value: value, total: sum }). If a relative-to-largest scale is genuinely wanted, derive it with Math.max rather than index 0 and label the section so the two encodings are not read as the same thing.

### `HW-07-0041` - The annual rate is validated against a picker placeholder it can never hold again, so an emptied rate field is calculated as zero percent

- Category B, severity medium, confidence confirmed
- Features: FIN-07
- Document: `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- Why: filledYearRate starts as the picker placeholder 请选择 and is checked against it, but the control bound to it is a TextInput, so the moment the user types anything the state leaves that sentinel and can never return to it. Type a rate and then clear the field and the check still reports the rate as filled: the Calculate button turns active, Number('') yields 0, and the detail page presents a full repayment schedule at zero percent with a total interest of zero, as though the loan were free. The identical check one line above uses !== '' for the loan amount, which is the correct test for a text field, so the two inputs on the same form are validated by different rules. The same sentinel comparison appears in toastHasUnfilled, whose 请输入年利率 branch is therefore unreachable once the field has been touched.
- Fix: Test the rate for emptiness like the amount - this.filledYearRate !== '' - and initialise it to '' rather than to the picker placeholder, since it is not a picker. Reject a rate of zero explicitly if a zero-interest loan is not intended.

### `HW-07-0042` - The repayment schedule renders up to 240 rows with a keyless ForEach instead of LazyForEach

- Category C, severity medium, confidence confirmed
- Features: FIN-07
- Document: `huawei_industry_tree/07_finance_insurance/docs/07_loan_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/loan_calculator-0000002352881396
- Why: ForEach builds a component tree for every element up front, so selecting the twenty-year term materialises 240 ListItems, each with four Text children, before the first screenful is shown - roughly a thousand components for a list that displays about ten rows at a time. With no key generator supplied the framework falls back to its default, which serialises each item to build the key, so every entry in the schedule is passed through JSON.stringify on each render pass. The array is also @State and is filled by push inside calEqualBaseInterest, so the render pass runs against the growing array. LazyForEach exists for exactly this shape - a long, uniform, scrollable list - and the practice is published as the reference implementation of a repayment schedule, which is one of the longer lists a finance app draws.
- Fix: Wrap the schedule in an IDataSource and use LazyForEach with cachedCount, keyed on item.month. If ForEach is kept, supply the key generator explicitly: (item: MonthRepayInfo) => item.month.toString().

### `HW-07-0045` - A three-clause guard tests one condition three times and rewrites a cap of zero into no cap at all

- Category B, severity medium, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: The three clauses are not three checks. !this.maxLength already covers undefined, and the second and third clauses are the same test written two ways, so the guard reduces to !this.maxLength. What it actually does is rewrite any falsy cap to Infinity, which means a caller setting maxLength to 0 - the one value that unambiguously asks for no input at all - gets unlimited input instead, the exact opposite. The field is declared number with a default of Infinity, so neither undefined case the guard defends against can occur under the type system. It also mutates the caller's setting from inside an input handler rather than validating it where it is assigned. The same guard appears in the custom stock keyboard controller in this industry, so the pattern is being propagated.
- Fix: Delete the guard. If a caller may legitimately pass undefined, widen the field to number | undefined and normalise once at assignment, keeping 0 as a valid cap.

### `HW-07-0046` - The random helper divides a byte by 255, so it can return exactly 1 and is not the half-open range its callers will assume

- Category B, severity medium, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: The function is named as the project's secure replacement for Math.random and is exported from the controller for general use, but it does not match Math.random's contract: one byte in 256 produces exactly 1.0. Any caller that turns it into an index the obvious way - Math.floor(r * n) - gets n back once every 256 draws, which is one past the end of the array. The sample's only current caller subtracts 0.5 for a sort comparator and so does not trip on it, which is what makes this dangerous: the defect is invisible where it is used and fires in the first correct use of the helper, including the Fisher-Yates shuffle this keyboard needs anyway. Dividing by 256 is also the arithmetically right mapping, since a byte has 256 equally likely values, not 255.
- Fix: Divide by 256 so the result is uniform over [0, 1), and document the range on the method. Take the bytes in one generateRandomSync(n) call from a generator held on the controller rather than constructing one per draw.

### `HW-07-0050` - The total that every percentage label is divided by is a hardcoded literal, and the total shown on screen is a hardcoded string resource

- Category B, severity medium, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: The sum of the six figures is written out by hand in two more places on top of the two copies of the values array: as a number field on the chart component, which is the divisor for every percentage label the chart draws, and as a literal string inside string.json, which is the 总支出 amount shown at the top of the screen. Nothing recomputes either. Replace the mock data with real spending and the pie still divides by 3174.3, so the labels no longer sum to anything near 100 percent and each one is wrong by the ratio of the old total to the new, while the headline goes on displaying the old total entirely. Putting a computed figure in a string resource is the more damaging half: string.json is where translators and reviewers work, it is not where anyone looks for arithmetic, and the value has no unit or context to mark it as data rather than copy.
- Fix: Compute the total once - const total = this.values.reduce((a, b) => a + b, 0) - pass it to the chart component, and render the headline from that value rather than from a string resource. Keep only the currency symbol and formatting in resources.

### `HW-07-0051` - Each ListItem holds two children against the documented single-child rule, and the divider is positioned by a 50vp top margin

- Category C, severity medium, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: ListItem is documented to contain a single child component, and this one is given two - the composition row and a conditional Divider. The two are then stacked rather than laid out, which is why the Divider needs a 50vp top margin to clear a row that happens to be 50vp tall: the separator is placed by a magic number that duplicates LIST_ITEM_HEIGHT, so changing the row height silently moves every divider off its line. List already provides a divider attribute that draws separators between items with correct insets and no extra components, and the last-item check the code performs by hand is exactly what that attribute does for free. The ForEach also supplies no key generator, so the framework falls back to serialising each entry.
- Fix: Put only the composition row inside ListItem and set the separators with List's divider attribute: .divider({ strokeWidth: 1, startMargin: 40 }). Key the ForEach on the label.

### `HW-07-0009` - Possibly-undefined card fields are passed into a constructor that declares them as strings

- Category B, severity low, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: Both fields are declared string | undefined and are assigned through optional chaining from cardInfo?.front, so undefined is a reachable value whenever recognition returns a partial result - a card read where the number came through but the name did not, for instance. They are then handed straight to a constructor whose parameters are declared string. The receiving page assigns them into @State strings and renders them, so a partial recognition produces a screen showing the literal text undefined where the user's name should be. The class on the receiving side is a second, separate Info declaration with placeholder defaults 'a' and 'b', so the two ends of this hand-off do not even share a type.
- Fix: Guard before popping - if (!front?.idNumber || !front?.name) { report and return; } - and share one Info declaration between the two pages instead of declaring it twice.

### `HW-07-0010` - The privacy notice shown on the identity screen contains a character error

- Category E, severity low, confidence confirmed
- Features: FIN-01
- Document: `huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
- Why: The two characters are homophones and the wrong one is a common input-method slip, but this is the notice that tells the user what their identity-document data will be used for and promises that it will be protected. It is the closest thing in the sample to a privacy statement, it is rendered directly beneath the control that scans an ID card, and it reads 同事我司将采用完善的安全措施 - colleague, our company will adopt comprehensive security measures. The string is also a hardcoded literal in the page rather than a resource, so it cannot be corrected or translated without editing source.
- Fix: Correct 同事 to 同时 and move the notice into string.json.

### `HW-07-0011` - Industry FAQ page has no content and redirects to the unfiltered phone FAQ index instead of the finance FAQs

- Category E, severity low, confidence confirmed
- Features: FIN-11
- Document: `huawei_industry_tree/07_finance_insurance/docs/11_practice-insurance-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1_2-0000002263613646
- Why: The page is a migration stub whose entire body is one sentence and one link. The link target is faq-phone, the general phone FAQ index for all of HarmonyOS, with no anchor, category filter or search term for finance or insurance. A developer who opens 行业常见问题 from the insurance architecture guide is handed an unfiltered list and has no way to reach the industry FAQ content the sentence promises. This is byte-for-byte the same stub and the same target already recorded in maternity health, automotive and public transport - four of the four industries reviewed so far - so the redirect is shared boilerplate rather than a per-industry pointer. It matters more here than elsewhere: a finance application has security and compliance questions that a generic phone FAQ will not answer.
- Fix: Point the link at the finance-filtered FAQ view, or list the migrated questions inline with direct links so the industry context survives the migration.

### `HW-07-0014` - A field named uiContext holds a UIAbilityContext, and is used only as a resource-manager handle

- Category B, severity low, confidence confirmed
- Features: FIN-03
- Document: `huawei_industry_tree/07_finance_insurance/docs/03_stock_chart_x.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_chart_x-0000002264336070
- Why: getUIContext() returns a UIContext and getHostContext() returns the ability's Context, so the cast yields a common.UIAbilityContext - a different type from UIContext, reached by calling through one to get the other. Naming it uiContext makes every use site read as though a UIContext is in play, which is the type a developer would then try to pass to APIs that genuinely want one. Every one of its four uses immediately reaches for .resourceManager and nothing else, so the field is really a resource-manager handle wearing the wrong name. The same misleading name appears in the transit industry's shortcut practice, so it is spreading between samples rather than being a one-off.
- Fix: Rename to abilityContext, or hold what is actually used: private resources = (this.getUIContext().getHostContext() as common.UIAbilityContext).resourceManager;

### `HW-07-0017` - The date picker's lower bound is the one date in the file built from a string

- Category B, severity low, confidence confirmed
- Features: FIN-04
- Document: `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- Why: '2000-1-1' is not an ISO 8601 date - ISO requires two-digit month and day - so it falls into implementation-defined parsing and is read as local time, whereas a zero-padded '2000-01-01' would be read as UTC. The resulting instant therefore shifts with the device timezone. As a picker lower bound the practical consequence is small, but the inconsistency is the point: this file otherwise builds every one of its eleven other Date values from numeric components, which is the correct and unambiguous form, and the one string construction sits in the snippet the document prints as step 3. The maternity health industry has three confirmed defects from exactly this construction, so it is worth not teaching it here.
- Fix: Use new Date(2000, 0, 1).

### `HW-07-0018` - The year and month fields carry dead initial values that also disagree with the bundled data

- Category B, severity low, confidence confirmed
- Features: FIN-04
- Document: `huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
- Why: Both fields are overwritten on the first line of aboutToAppear before anything reads them, so the literals never take effect. They are also wrong on their own terms: month is one-based here, since calcFlows converts with m - 1, so 5 means May, while every bundled record is dated with new Date(2025, 5, ...) where the month argument is zero-based and therefore June. A reader trying to work out which month the sample is meant to open on gets one answer from the field, another from the data, and a third from the code that overwrites both. The literals are the kind of leftover that later becomes a real default when someone removes the aboutToAppear assignment.
- Fix: Drop the initial values, or set them from the data the sample ships and let aboutToAppear override only when the data covers the current month.

### `HW-07-0022` - The keyboard module ships no dark-mode resources although the two modules around it do

- Category B, severity low, confidence confirmed
- Features: FIN-05
- Document: `huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
- Why: The keyboard is the one module in this project whose resources directory has no dark qualifier, while the entry module and the shared common module both have one. A custom keyboard covers the lower third of the screen and defines its own key backgrounds, key text colours and panel background, so it is precisely the component that looks wrong when the system switches to a dark theme - the surrounding page follows the theme and the keyboard does not. Since this module is packaged as a reusable HAR, whatever application imports it inherits the gap.
- Fix: Add keyboard/src/main/resources/dark with the colour set the keyboard uses, matching the qualifiers the other two modules already provide.

### `HW-07-0033` - The masked placeholder is one character longer than the number it masks

- Category B, severity low, confidence confirmed
- Features: FIN-08
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: The masked placeholder is seventeen characters - five leading digits, eight asterisks, four trailing digits - while the number it is replaced by is a sixteen-digit card number. The text therefore reflows when verification succeeds, and the mask misrepresents the length of the value, which for a card number is information the user may check against the physical card. Both strings are also hardcoded in the page rather than derived from one another, so a real implementation replacing the number has to remember to re-count the asterisks; masking should be computed from the value instead.
- Fix: Derive the mask from the number - keep the first four or six and last four digits and fill the middle with as many asterisks as the number has remaining digits - so the two are the same length by construction.

### `HW-07-0039` - The third-party charting dependency is on a floating version range and is absent from the constraints

- Category E, severity low, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: The whole feature depends on @ohos/mpchart - without it there is no chart - yet 约束与限制 lists only SDK, system and IDE versions. The package is named in 场景介绍 and linked to ohpm, but not as a constraint, so a reader scanning the constraints for what the sample needs will miss that an ohpm package must resolve. The caret range accepts any future 3.x while the SDK beside it is pinned exactly, so a behaviour change in a later mpchart release reaches anyone who builds the sample later and the document records no version that was actually verified. The maternity health industry's growth-curve practice carries the identical defect against the same package at ^3.0.11, so the two samples also disagree about which version of it they need.
- Fix: Add the dependency and its verified version to 约束与限制, and pin the sample to an exact version rather than a caret range.

### `HW-07-0047` - Each keyboard component unsubscribes every listener on the shared event id, not its own

- Category A, severity low, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: Both keyboard layouts subscribe to the same event id, 20250807, and both tear down with the single-argument off, which the reference states removes every subscriber of that id. Whichever component is destroyed first therefore cancels the other's subscription as well as its own. Nothing misbehaves as shipped, because the three layouts are switched with visibility rather than conditional rendering and so are destroyed together, but that is a property of the current layout code rather than of the cleanup: change the visibility switch to an if - the normal way to avoid building three keyboards - and the surviving layout silently stops reshuffling, with no error anywhere. The id is also a bare number in app scope, so anything else in the app that uses it loses its listeners too.
- Fix: Keep the registered callback in a field and pass it to emitter.off(eventId, callback) in aboutToDisappear so each component removes only what it added.

### `HW-07-0048` - A key's background colour is chosen by an expression that ands a colour resource with a boolean

- Category B, severity low, confidence confirmed
- Features: FIN-09
- Document: `huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
- Why: && binds tighter than the conditional operator, so the false branch of the first conditional is ($r(normal) && isFuncMenu(index)) ? funcColor : normal. A Resource object is always truthy, so that sub-expression collapses to isFuncMenu(index) and the right colours are produced - the line works by accident, not by design. As written it reads as though a colour is being combined with a predicate, the middle $r('app.color.keyboard_normal_menu_color') looks like a value being returned when it is only being tested and discarded, and the whole selection depends on an operator precedence rule that the layout gives no hint of. Anyone editing this line to add a fourth key style will almost certainly get a different result than they intend.
- Fix: Write it as a nested conditional with parentheses, or replace it with a helper - keyColor(index): Resource - that returns the finish, function or normal colour and is testable on its own.

### `HW-07-0049` - The password dot renders undefined for empty slots because the ternary's false branch repeats the expression it just tested

- Category B, severity low, confidence confirmed
- Features: FIN-08
- Document: `huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
- Why: The two branches of the conditional are the masking symbol and the very expression that was just tested for falsiness, so the false branch can only ever evaluate to undefined - reading an index past the end of the entered digits. Text is declared to take string | Resource, and undefined satisfies neither; the index access is typed string only because the array is not read under strict index checking, so the type system does not catch it. The intent is plainly an empty slot, which is what an empty string gives, and writing it that way also makes the line say what it means instead of appearing to render the digit it is trying to hide.
- Fix: Return an empty string from the false branch.

### `HW-07-0052` - The bottom tab bar cannot switch tabs and the state that colours the selected tab is never written

- Category B, severity low, confidence confirmed
- Features: FIN-10
- Document: `huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
- Why: onContentWillChange returns false for every request, so the Tabs container refuses every switch and the two neighbouring tabs can never be opened - which is defensible for a single-feature demo, except that nothing says so. What makes it a defect rather than a simplification is that the code around it is written as though switching worked: selectedIndex is declared @State, is read to pick the selected and unselected label colours, and is never assigned, so the highlight is frozen on index 1 and the @State decorator drives no update. A reader adopting this page gets a tab bar that looks interactive, taps that do nothing with no feedback, and a state variable whose purpose only becomes clear once they discover it is dead.
- Fix: Either drive it properly - drop onContentWillChange and set selectedIndex from onChange - or make the intent explicit: render a static bar, make selectedIndex a plain constant rather than @State, and note in the document that only the 统计 tab is implemented.

