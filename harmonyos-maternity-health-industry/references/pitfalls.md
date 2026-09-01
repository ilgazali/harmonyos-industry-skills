# Pitfalls

> Generated from `features/*.md`. Source industry: `10_maternity_health`, 5 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-10-0033` - Systematic: 2 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: MAT-03, MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (32)

### `HW-10-0024` - Duplicate-date check calls getDay() instead of getDate(), so records on different days overwrite each other

- Category B, severity blocker, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: getDay() returns the day of the week, 0 to 6, not the day of the month; getDate() is the day of the month. The condition therefore reads 'same month and same weekday'. Any two dates in the same month that share a weekday - 5 August and 12 August, for instance - are treated as one record, and the second save silently overwrites the first in growthRecordList, heightList, weightList and dateList. The year is never compared either, so a measurement taken in March 2026 overwrites the March 2025 entry. In a growth chart whose whole purpose is a trend over time, this destroys user data on a completely ordinary input.
- Fix: Compare the calendar day: item.getFullYear() === d.getFullYear() && item.getMonth() === d.getMonth() && item.getDate() === d.getDate(). Better, normalise every record to a yyyy-MM-dd key and compare that.

### `HW-10-0025` - Hardcoded index-6 offset between the record list and the chart arrays produces negative indices and silent data loss

- Category B, severity blocker, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: heightList, weightList and dateList are each seeded with six demo values while growthRecordList starts empty, so an index into dateList is offset by exactly six from the same record's position in growthRecordList. The code hardcodes that six. Whenever the matched index is below 6 - that is, whenever the user edits or inserts before any of the six seeded points - index - 6 is negative. In the assignment branch, growthRecordList[-1] = record creates a plain string-keyed property on the array object: the record is not in the list, length does not change, and it never renders. In the splice branch, a negative start counts back from the end of the array, so the record is inserted near the end instead of at the front, and the record list falls permanently out of step with the three chart arrays it is supposed to parallel.
- Fix: Drop the parallel arrays. Keep growthRecordList as the only stored state, seed it with the six demo records so no offset exists, and compute heightList, weightList and dateList from it with map() wherever the chart needs them.

### `HW-10-0001` - LazyDataSource.addData() appends N items but emits a single notifyDataAdd, desynchronising LazyForEach from totalCount()

- Category B, severity high, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: concat() appends every element of 'data' to the end of the array, but notifyDataAdd(index) tells LazyForEach that exactly one item was inserted at position 'index'. After the call totalCount() has grown by data.length while the framework has created only one child component, so the data source and the rendered list disagree. The parameter 'index' is also ignored: items always land at the end no matter what the caller passes. The load-more path in FriendCircleDetailPage.ets:99 hits this with a six-element array.
- Fix: Insert at the requested position and notify once per element: this.dataArray.splice(index, 0, ...data); data.forEach((_, k) => this.notifyDataAdd(index + k));

### `HW-10-0002` - Removing a selected photo always removes the first one because findIndex compares item.uri with itself

- Category B, severity high, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The arrow function parameter shadows any outer binding, so 'item.uri === item.uri' compares a value with itself and is true for the very first element. findIndex therefore always returns 0 and deleteData(0) removes the first selected photo regardless of which delete badge the user tapped. The correct form of the same comparison exists 100 lines below in AlbumItemComp, where it reads 'item.uri === this.albumData.uri'.
- Fix: Bind the row item in the LazyForEach item generator and compare against it: LazyForEach(this.checkedAlbumData, (row: IPicData) => { ... arrayList.findIndex((item: IPicData) => item.uri === row.uri) ... }).

### `HW-10-0003` - onShown re-imports the whole system album without clearing, so the picker grid duplicates every time the page is shown again

- Category B, severity high, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: initAlbumDataResource() only calls this.albumData.addData(i, [...]) in a loop over every asset returned by photoAccessHelper; it never clears the data source first. NavDestination.onShown fires every time the page becomes visible again - returning from the camera picker, or popping back from another page - so the complete album is appended a second, third and fourth time. The list grows without bound and every photo is rendered repeatedly.
- Fix: Clear the source before refilling (call this.albumData.clear() at the start of initAlbumDataResource), or load once from aboutToAppear instead of onShown.

### `HW-10-0004` - Calendar builds Date objects from non-padded strings, shifting the whole month grid by one column in months 10-12 west of UTC

- Category B, severity high, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: currMonth is not zero-padded, so months 1-9 produce '2026-8-01', which is not an ISO 8601 date and is parsed as local time, while months 10-12 produce '2026-10-01', which is valid ISO 8601 and is parsed as UTC midnight. Reproduced with TZ=America/New_York: new Date('2026-10-01').getDay() returns 3 (Wed) instead of the correct 4 (Thu), while new Date('2026-8-01').getDay() correctly returns 6. The leading blank columns are therefore off by one for October, November and December for every user behind UTC, so every date lands under the wrong weekday - in a period and ovulation tracker that is a wrong-day prediction. The same construction drives the pregnancy trimester arithmetic at lines 46, 70 and 82, where the timezone skew can push a boundary day into the wrong trimester.
- Fix: Build dates from numeric components instead of strings: new Date(this.currYear, this.currMonth - 1, 1). Apply the same change at lines 46, 70 and 82.

### `HW-10-0015` - Image preview opens a gallery of every photo ever added instead of the photos of the tapped diary entry

- Category B, severity high, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: imageList is a single page-level accumulator: every diary entry's images are appended to it and it is never partitioned per entry. ImageViewerComponent is handed that whole flat array, so opening any photo lets the user swipe through the photos of unrelated entries from other dates. On top of that, selectIdx is computed with imageList.indexOf(item), which returns the first occurrence: if the user picks the same photo for two entries - trivially easy, the picker allows it - tapping it in the second entry opens the first entry's position instead.
- Fix: Pass childItem.imageContent to ImageViewerComponent instead of the global list, and take the index from the inner ForEach position rather than indexOf: ForEach(childItem.imageContent, (item: string, idx: number) => ... this.imageList = childItem.imageContent!; this.selectIdx = idx; ...).

### `HW-10-0026` - Date picker range is hardcoded to a window that has already expired, so no current date can be recorded

- Category B, severity high, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: The picker only offers dates between 5 August 2025 and 31 December 2025. selectedDate is initialised to new Date(), the current day, which already falls outside that window, so the picker opens on a value it cannot represent and the user cannot choose today or any date in the current year. Anyone building and running the sample after 2025 finds the add-record flow unusable, which is the only way to get data into the chart the document is about. A fixed literal window in shipped sample code has no expiry warning attached to it.
- Fix: Compute the range: start from the earliest meaningful date for the child and set end to new Date(), so the window follows the clock instead of a literal.

### `HW-10-0005` - LazyDataSource.findDataIndex() can never find anything because the predicate result is discarded

- Category B, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The inner arrow function has a block body and no return statement, so it evaluates to undefined for every element. findIndex therefore never sees a truthy value and returns -1 unconditionally. The predicate is also handed a wrapper object { item } rather than the element itself, so a caller written against the normal Array.findIndex contract would not work either. Nothing in the sample calls it yet, which is why the defect is invisible, but this is framework code the document tells developers to build on.
- Fix: return this.dataArray.findIndex((item: T) => callback(item)); and type the parameter as (item: T) => boolean instead of Function.

### `HW-10-0006` - LazyForEach key generators call JSON.stringify, which the official code-linter rule forbids

- Category C, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The rule @performance/hp-arkui-no-stringify-in-lazyforeach-key-generator states: 'Do not use stringify in the key generator function that uses LazyForEach for component reuse.' Serialising the whole object on every key computation allocates a long string per row per pass, which is exactly the scroll frame-loss cost that the 方案设计 section of this document tells the developer to avoid by following the long-list loading practice. In FriendCircleDetailPage the index is concatenated as well, so the key of an unchanged comment changes whenever an item is inserted above it, violating the key-consistency requirement stated in the LazyForEach guide.
- Fix: Use an identifying field as the key: (item: IPicData) => item.uri and (item: CommentData) => item.id. Never fold the index into a key for a list that supports insertion.

### `HW-10-0007` - Tab underline is measured at font weight 700 but the label is rendered at 500, so the indicator never centres correctly

- Category C, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: getCurrTextWidth() measures with FONT_WEIGHT_SEVEN (700), but tabBuilder renders the selected label with FONT_WEIGHT_FIVE (500) and unselected labels with FONT_WEIGHT_FOUR (400). Bold text measures wider than medium text, so the measured width always exceeds the rendered width and the centring term (measuredWidth - underlineWidth) / 2 pushes the underline right of the label centre. Separately, onAnimationStart positions the underline at targetIndexInfo.left, which belongs to targetIndex, but computes the centring offset from getCurrTextWidth(index) - the source tab. All four demo labels are two Chinese characters, so the widths coincide and both defects stay hidden until a developer changes the labels, which is exactly what the document instructs them to do.
- Fix: Measure with the weight actually rendered (FONT_WEIGHT_FIVE for the selected tab), and pass targetIndex rather than index to getCurrTextWidth() inside onAnimationStart.

### `HW-10-0008` - Predicted period and ovulation days are drawn in every month because only isMenes is gated on the displayed month

- Category B, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: isMenes is correctly suppressed when the user browses away from the current month, but isForecastMenes and isOvulation are computed from the day-of-month number alone. Swiping to any other month therefore repaints the same predicted-period and ovulation days there, so a user scrolling through the year sees an identical forecast in every single month. For a menstrual tracker that forecast is the core output of the feature.
- Fix: Apply the same month guard to all three fields, or better, key the constants to absolute dates rather than day-of-month numbers so a forecast can span a month boundary.

### `HW-10-0009` - LazyDataSource rebuilds every row on append, contradicting the long-list performance practice the document cites

- Category C, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: appendArrayData adds a single element but calls notifyDataReload(), which tells LazyForEach to discard and rebuild every child component instead of creating one. pushArrayData is worse: clear() already fires a reload, then each pushData fires its own notifyDataAdd, then a third reload fires at the end, so loading n rows costs n+2 notifications and two full rebuilds. The 方案设计 section of this document tells the developer to follow the long-list loading optimisation practice, and the framework code shipped with it does the opposite. All seven feed lists in the sample load through pushArrayData.
- Fix: appendArrayData should call this.notifyDataAdd(this.dataArray.length - 1). pushArrayData should replace the backing array in one step and fire a single notifyDataReload().

### `HW-10-0011` - Project tree lists SearchInputComp.ets but the sample ships SerachInputComp.ets

- Category E, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The file name in the shipped project is misspelled - 'Serach' instead of 'Search'. The document silently prints the corrected spelling, so a developer following the tree cannot find the file and any import path copied from the document fails to resolve. The export in common/Index.ets carries the misspelling too, so the defect is in the sample and the document conceals it.
- Fix: Rename the file and its export to SearchInputComp.ets in the sample project so it matches the document.

### `HW-10-0012` - Document scopes the practice to phones but the sample declares phone, tablet and 2in1 device types

- Category E, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The document states three times that this practice targets a phone: the hardware requirement says Huawei phone, the introduction plans a phone Entry HAP, and the software view says the product customisation layer covers one phone end. The shipped manifest also declares tablet and 2in1, so the app is distributed to and installed on devices whose layouts the document never discusses. The sample does carry breakpoint handling for md and lg, which suggests the manifest is right and the document is stale, but a reader cannot tell which side to trust.
- Fix: Either narrow deviceTypes to ['phone'] to match the stated scope, or update the document to describe the multi-device scope and the breakpoint strategy the sample already implements.

### `HW-10-0013` - Software view claims a common capability layer with routing, networking and database HARs that the sample does not contain

- Category E, severity medium, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The sample ships exactly one common HAR and it contains only components, constants, model, pages and three utils: BreakPointUtil, DropPostingPageUtil and LoggerUtil. There is no routing module, no network module and no database module - navigation uses a raw NavPathStack in the entry module and all content is hardcoded arrays in model files. Only LoggerUtil corresponds to anything in the list. The note under 应用框架代码 says the code is partial, but the software view is presented as the architecture of this practice, and that layered structure is the main thing a developer takes from an architecture guide.
- Fix: State in 软件视图设计 which common capability modules fall outside the downloadable sample, or ship minimal routing, network and database HARs so the layered architecture the document teaches is actually demonstrated.

### `HW-10-0016` - TextInput two-way binding attaches a number state to the text parameter, which is documented as ResourceStr

- Category A, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: The TextInput reference defines TextInputOptions.text as ResourceStr, that is string or Resource, and says two-way binding through $$ is supported since API version 10. The sample binds vHeight and vWeight, both declared number. With $$ the component writes the edited text back into the bound variable, so at runtime the state holds a string while its declared type says number, and the save guard 'this.vHeight <= 0 || this.vHeight >= 200' then runs relational comparisons on a string. Diary.ets in the same project binds a correctly typed string to TextArea, so this is a defect rather than a project convention.
- Fix: Declare the inputs as strings and convert once on save: @State vHeight: string = ''; then const h = Number(this.vHeight); validate Number.isFinite(h) before building HealthInfo.

### `HW-10-0017` - Document says duplicate time points are collapsed to the latest record, but the code keeps every record for a date

- Category E, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: getData() deduplicates the formatted date strings, so each date appears once as a ListItemGroup header - that part matches. But the grouping loop then pushes every content item whose date equals that header into tempArr, so all records for a date are kept and rendered, not just the latest. A developer reading 保留最新数据 would expect an earlier same-day weight measurement to be replaced by a later one, and would build persistence around that assumption. The behaviour the code implements is grouping, not deduplication of records.
- Fix: Reword the step to describe grouping - each date appears once as a header and all of its records are listed beneath it - or implement the stated behaviour by keeping only the most recent record per date in getData().

### `HW-10-0018` - Timeline grouping key is a locale-formatted display string hardcoded to zh-CN

- Category B, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: Records are stored with date set to dateFormat(params.date), and grouping compares that stored display string against a freshly formatted header string. The whole grouping therefore depends on Intl producing byte-identical output for the same day on every call. The locale is pinned to zh-CN, so the timeline headers stay Chinese no matter what language the device is set to - a localisation defect on its own in a template meant to be adapted - and the moment a developer changes the locale or dateStyle to localise the app, previously stored records stop matching the new headers and silently disappear from their groups.
- Fix: Store an ISO key on the record (date.getFullYear() + '-' + ... zero-padded) and group on it; call Intl only inside timeLineHead() to render the header, using the system locale rather than a hardcoded 'zh-CN'.

### `HW-10-0019` - getData() rebuilds the whole timeline quadratically on every save and mutates the caller's state array in place

- Category C, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: Three compounding problems. First, the nested forEach scans the entire content list once per unique date, so the cost is dates x records - quadratic in the size of a timeline that by design grows for years. Second, the caller pushes to this.dates on every save and never deduplicates, so the dates array keeps a separate entry for every record and is re-sorted in full each time. Third, dates.sort() sorts in place, mutating the caller's @State dates array as a side effect that the name getData does not advertise; a reader would not expect a getter to reorder component state.
- Fix: Build the groups in a single pass with a Map<string, IRecordList[]> keyed by the date key, sort the keys once, and take a copy before sorting: const sorted = [...dates].sort(...).

### `HW-10-0020` - Diary save accepts a completely empty entry while the growth record page validates its input

- Category B, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: The two record types are entered through sibling pages that reach the same timeline, but only one of them validates. Tapping save on an untouched diary page pops a result with an empty content string and an empty image array, so the timeline gains a group header and a blank row that the user cannot distinguish or explain. The empty row also enters the dates array and so permanently affects the grouping. The correct pattern is already present ten files away in the same project.
- Fix: Guard the diary save the same way: if (!this.contents.trim() && this.toAddImages.length === 0) { showToast(...); return }.

### `HW-10-0021` - Height and weight labels are hardcoded Chinese in the timeline while the same labels are string resources in the record page

- Category F, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: Every other user-visible string in this project goes through the resource system - titles, placeholders, button labels, and the very same height and weight labels and cm/kg units on the entry page. The timeline row builds them as hardcoded Chinese literals in code instead. The result is an app that cannot be localised without editing source, and a sample that demonstrates two contradictory conventions for the same two labels in one project.
- Fix: Replace the literals with the existing resources and interpolate only the numbers, for example Text($r('app.string.timeline_height', childItem.healthyInfo.height)).

### `HW-10-0022` - The industry ships two contradictory image-picking patterns with very different permission costs and neither document mentions the trade-off

- Category F, severity medium, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: Both documents belong to the same industry and both solve 'let the user attach photos to a post'. This one uses PhotoViewPicker, a security component that returns read-only URIs and needs no permission at all - the shipped module.json5 declares none. The architecture practice MYDemo instead enumerates the entire album with getAssets() and therefore has to request ohos.permission.READ_IMAGEVIDEO up front, gating the whole composer behind a permission dialog. Neither document says why, so a developer reading both cannot tell which is recommended, and the natural reading - that attaching photos needs READ_IMAGEVIDEO - is wrong for the common case.
- Fix: State in both documents that PhotoViewPicker is the default for attaching a bounded selection and needs no permission, and that getAssets() with READ_IMAGEVIDEO is only for rendering a custom in-app album browser.

### `HW-10-0027` - Seed dates are built from non-padded strings, mixing local-time and UTC parsing inside the same array

- Category B, severity medium, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: '2025-3-5' is not an ISO 8601 date - ISO requires two-digit month and day - so it falls into implementation-defined parsing and is read as local time, while '2025-12-31' in the same file is valid ISO and is read as UTC midnight. The two forms therefore differ by the device's UTC offset. Reproduced with TZ=America/New_York: new Date('2025-12-31') resolves to 30 December local, one day earlier than intended, while the non-padded literals resolve to the day named. The date picker's end bound is consequently a day short, and the axis labels rendered by XValueFormatterMonth from these seeds can be off by one for users behind UTC. The same construction appears in the GrowthRecord view model default.
- Fix: Replace every date literal with numeric-component construction, for example new Date(2025, 2, 5) and new Date(2025, 11, 31).

### `HW-10-0028` - Every list row performs synchronous resource lookups on the UI thread instead of using $r()

- Category C, severity medium, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: getStringByNameSync is a blocking resource-manager call. Placing it inside a ForEach item builder means six lookups per row - height prefix and suffix, weight prefix and suffix, user label and date format - executed on the UI thread on every render pass of every row, and repeated for both tabs. The very same file uses $r('app.string....') declaratively for the chart headers thirty lines above, so the sample demonstrates both the cheap and the expensive way to read one string and gives no reason for the difference. Declarative resources also let the framework re-resolve on a locale change, which the sync calls do not.
- Fix: Use $r('app.string.name') for the plain labels, and $r('app.string.name', arg) or the resource's own format placeholders for the interpolated ones, so no synchronous lookup happens during layout.

### `HW-10-0029` - The @Watch handler shown in the document writes the same state it is registered to watch

- Category C, severity medium, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: max carries a @Watch that points at onDataChange, and onDataChange assigns to max on its first line. Every write to max therefore schedules another invocation of its own handler. The loop terminates today only because ArkUI suppresses the notification when the assigned value equals the current one, so the second pass writes an identical number and stops. That is termination by coincidence, not by design: any change that makes the recomputation depend on something that moves between passes turns it into repeated re-entry, and a reader copying the snippet from 实现思路 step 3 inherits the pattern without the warning. The document reproduces this exact handler as the recommended refresh mechanism.
- Fix: Remove @Watch from max and keep it on the real inputs only: @Prop @Watch('onDataChange') trendData and @LocalStorageLink @Watch('onDataChange') dateList. Declare max as a private field, since nothing renders it directly.

### `HW-10-0010` - Deprecated @ohos.measure module imported but never used, alongside a general mix of legacy @ohos.* and current @kit.* import paths

- Category A, severity low, confidence confirmed
- Features: MAT-01
- Document: `huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
- Why: The 'measure' binding is dead: the file measures text through this.uiContext.getMeasureUtils().measureText() and never touches it. The module it points at is deprecated - the reference page for @ohos.measure marks MeasureText.measureText as 'supported since API version 9 and deprecated since API version 18' and gives the current import as 'import { MeasureText } from "@kit.ArkUI"'. The project declares compatibleSdkVersion 6.0.0(20), well past that deprecation. The same file also reaches @ohos.arkui.UIContext directly, and PostingPage.ets mixes @ohos.multimedia.cameraPicker and @ohos.base with @kit.MediaLibraryKit and @kit.ArkData in a single file, so the sample teaches two conflicting import conventions at once.
- Fix: Delete the unused measure import and route the remaining @ohos.* imports through their Kit entry points: @kit.ArkUI for ComponentUtils, @kit.CameraKit for cameraPicker and camera, @kit.BasicServicesKit for BusinessError.

### `HW-10-0014` - Industry FAQ page has no content and redirects to the unfiltered phone FAQ index instead of the maternity health FAQs

- Category E, severity low, confidence confirmed
- Features: MAT-05
- Document: `huawei_industry_tree/10_maternity_health/docs/05_practice-health-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1_2-0000002332396545
- Why: The page is a migration stub whose entire body is one sentence and one link. The link target is faq-phone, the general phone FAQ index for all of HarmonyOS, with no anchor, category filter or search term for maternity health. A developer who opens 行业常见问题 from the maternity health architecture guide is therefore handed an unfiltered list and has no way to reach the industry FAQ content the sentence promises. The same stub appears in other industry guides, so the redirect target is generic rather than per-industry.
- Fix: Point the link at the maternity-health-filtered FAQ view, or list the migrated questions inline with direct links so the industry context is not lost.

### `HW-10-0023` - Weight has no upper bound while height does, and the height input carries an empty onChange handler

- Category B, severity low, confidence confirmed
- Features: MAT-03
- Document: `huawei_industry_tree/10_maternity_health/docs/03_growth_record_timeline.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_timeline-0000002270223453
- Why: Height is bounded to (0, 200) but weight only has to be greater than zero, so a value of 9999 is accepted and rendered into the timeline as '体重 9999kg'. The 200 is also a bare literal in the condition while every other dimension in the project comes from CommonConstants. Separately, the height TextInput declares an onChange callback with an empty body: it is dead code left over from before the $$ two-way binding was introduced, and the weight input next to it has none, so the sample shows both spellings of the same thing.
- Fix: Add a weight upper bound, move both limits into CommonConstants alongside the other layout constants, and delete the empty onChange.

### `HW-10-0030` - Dead state fields carrying a comment that contradicts the refresh mechanism the document teaches

- Category B, severity low, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: Neither field is read anywhere: this.weight and this.nowWeight appear only at their declarations, and the file's other references are all to this.weightList. The comment describes a resize-to-force-redraw workaround that the sample does not implement and does not need, because the chart is refreshed through model.notifyDataSetChanged() and model.invalidate() - which is exactly what 实现思路 step 3 of this document presents as the refresh mechanism. A reader who trusts the comment will conclude that mpchart cannot be refreshed by data and will build the resize hack the sample deliberately avoided. The field is also named weight, colliding with the domain meaning of weight used throughout the same file. AddRecord.ets carries a matching unused pageStack field alongside the pathStack it actually uses.
- Fix: Delete weight, nowWeight and the misleading comment from MainPage.ets, and delete the unused pageStack field from AddRecord.ets.

### `HW-10-0031` - Save validation only rejects exactly zero, so negative heights and weights are accepted

- Category B, severity low, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: The guard tests equality with zero rather than a plausible range. A negative or absurdly large value passes straight through into the record and into the chart series, where it distorts the y-axis for every other point and makes the trend unreadable. The sibling practice in this same industry, MAT-03, bounds height to (0, 200) on its own entry form, so the industry ships two different validation standards for the same two measurements.
- Fix: Replace the equality tests with range checks and hoist the bounds into CommonConstants next to the other constants.

### `HW-10-0032` - Constraints section omits the third-party charting dependency, which is pinned only to a floating caret range

- Category E, severity low, confidence confirmed
- Features: MAT-04
- Document: `huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
- Why: The entire feature depends on @ohos/mpchart - without it there is no chart - yet 约束与限制 lists only SDK, system and IDE versions. The package is mentioned in 实现思路 and linked in 参考文档, but not as a constraint, so a reader scanning the constraints for what the sample needs will miss that an ohpm package must resolve. The version is also a caret range, so any future 3.x is accepted, while the SDK next to it is pinned exactly to 6.0.0(20). A behaviour change in a later mpchart 3.x silently reaches anyone who builds the sample later, and the document records no version that was actually verified.
- Fix: Add the dependency and its verified version to 约束与限制, and pin the sample to an exact version rather than a caret range.

