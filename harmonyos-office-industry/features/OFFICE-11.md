---
id: OFFICE-11
title: Publish a meeting from custom dialogs and write it into the system calendar
industry: 05_office
doc: huawei_industry_tree/05_office/docs/11_conference_release.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
sample: huawei_industry_tree/05_office/downloads/ConferenceRelease.zip
kits: ["@kit.CalendarKit", "@kit.CoreFileKit", "@kit.PreviewKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["@CustomDialog", CustomDialogController, DatePicker, TimePicker, TextPicker, "DatePicker.onDateChange", "calendarManager.getCalendarManager", "CalendarManager.editEvent", "calendarManager.Event", "calendarManager.EventType", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", "picker.DocumentViewPicker", "DocumentViewPicker.select", "picker.DocumentSelectOptions", "fs.openSync", "fs.readSync", "fs.writeSync", "fs.closeSync", "fs.OpenMode", ReadOptions, WriteOptions, "fileUri.getUriFromPath", "filePreview.openPreview", "filePreview.PreviewInfo", "util.TextDecoder.create", "TextDecoder.decodeToString", "util.format", "@StorageLink", "AppStorage.setOrCreate", "UIContext.getPromptAction", "PromptAction.openToast", Checkbox, Navigation, NavPathStack]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry]
findings: [HW-05-0063, HW-05-0064, HW-05-0065, HW-05-0066, HW-05-0067, HW-05-0068, HW-05-0069, HW-05-0070, HW-05-0184, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app has to **compose a meeting invitation on a
single form and then hand it to the system calendar**, so the reminder survives
the app being closed.

Three sub-problems make up the scenario:

- **Chained custom dialogs for the schedule.** Date, start time and duration are
  each collected in their own `@CustomDialog`, and confirming one opens the next
  - the date dialog's confirm button opens the time dialog. Each dialog writes
  straight into a shared `@Link meetingFormData: FormData`.
- **An attachment copied into the sandbox.** `DocumentViewPicker` returns a URI,
  the bytes are copied into `filesDir` under a same-named file, and the copy is
  opened with Preview Kit. The picked name may arrive percent-encoded, so it is
  decoded first.
- **Publication into the system calendar.** The completed form is turned into a
  `calendarManager.Event` and passed to `CalendarManager.editEvent`.

The button is two-stage: the first tap "creates" the meeting locally, the second
publishes it to the calendar.

## Feature checklist

The application must be able to:

- Collect topic, date, start time, duration, format (video or on-site), venue,
  optional password, attendees and an announcement on one page.
- Open a date dialog, then chain into the time dialog and the duration dialog.
- Show the selected date with its correct weekday label.
- Store the date as a zero-padded `yyyy-MM-dd` string so it can be re-parsed.
- Toggle between video conference and on-site meeting with mutually exclusive
  checkboxes.
- Pick attendees from a list.
- Attach a local file through `DocumentViewPicker`, decode a percent-encoded
  file name, copy the bytes into the app sandbox, and preview the copy.
- Request `READ_CALENDAR` and `WRITE_CALENDAR`, verify the grant, and only then
  build a `CalendarManager`.
- On publish, build a `calendarManager.Event` with a title, start/end timestamps,
  a reminder offset and a description, and pass it to `editEvent`.
- Report failure at each step - picker, copy, preview and calendar write.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/PublishMeetingPage`, requests the two calendar permissions, publishes the `CalendarManager` into `AppStorage` under `calendarMgr` |
| `pages/PublishMeetingPage.ets` | the whole form, the dialog controllers, `uploadAttachment`, and the publish button |
| `views/CustomDialog.ets` | `CustomDatePickerDialog`, `CustomTimePickerDialog`, `CustomDurationPickerDialog` |
| `model/FormData.ets` | the `FormData` interface and the `MeetingFormat` enum |
| `model/AlterAttendees.ets` | the attendee row type |
| `utils/FileUtils.ets` | `findMimeType`, `createFile`, `readWriteFile`, `preview`, `utfFileNameToString` |
| `utils/TimestampUtils.ets` | `timeStrToTimestamp` - `'HH:mm'` to milliseconds since midnight |

State flows one way and back through a single object: the page owns
`meetingFormData: FormData` and passes it to each dialog as `@Link`, so a dialog
writes the user's choice directly into the page's state with no callback
plumbing. The `CalendarManager` travels the other way, from the ability into the
page, through `@StorageLink('calendarMgr')`.

Publish flow:

```
EntryAbility.onWindowStageCreate
  -> requestPermissionsFromUser([READ_CALENDAR, WRITE_CALENDAR])
  -> AppStorage['calendarMgr'] = calendarManager.getCalendarManager(context)

PublishMeetingPage
  date field   -> customDatePickerDialogController.open()
                    DatePicker.onDateChange -> meetingFormData.selectedDate
                    confirm -> customTimePickerDialogController.open()
                                 TimePicker -> meetingFormData.startTime / endTime
  duration     -> customDurationPickerDialogController.open() -> meetingFormData.duration
  attachment   -> uploadAttachment()
                    DocumentViewPicker.select -> uri
                    utfFileNameToString(name)  -> attachmentName
                    filesDir + '/' + name      -> attachmentUri
                    readWriteFile(srcUri, dest) -> copy into the sandbox
                    preview(name, getUriFromPath(attachmentUri), context)
  publish (2nd tap)
                 -> Event { title, type: NORMAL, startTime, endTime, reminderTime: [10], description }
                 -> calendarMgr.editEvent(event) -> id > 0 -> toast
```

Timestamps are assembled by adding a time-of-day offset to a midnight base:
`new Date(selectedDate + '-00:00').getTime() + timeStrToTimestamp(startTime)`.
That makes the zero-padding of `selectedDate` load-bearing (HW-05-0069).

## Implementation steps

1. **Declare the calendar permissions.** `ohos.permission.READ_CALENDAR` and
   `ohos.permission.WRITE_CALENDAR` in `module.json5` with
   `"when": "inuse"` - the sample does this correctly and it matches the
   document's 权限说明 section.
2. **Request them at startup and check the result.** In `onWindowStageCreate`,
   `requestPermissionsFromUser`, then inspect `data.authResults` and only publish
   the `CalendarManager` into `AppStorage` when every entry is `0`
   (HW-05-0065).
3. **Define one shared form object.** A `FormData` interface holding every field,
   held as `@State` on the page and passed to each dialog as `@Link`.
4. **Chain the dialogs.** Each `@CustomDialog` takes `@Link meetingFormData` and
   an optional `controller?: CustomDialogController`; its confirm button closes
   itself and opens the next controller.
5. **Format the date defensively.** Build `yyyy-MM-dd` with **string** padding,
   not `Number('0' + m)` (HW-05-0069), and index the weekday array so it lines up
   with `Date.getDay()`'s Sunday-first numbering (HW-05-0070).
6. **Pick the attachment.** `new picker.DocumentViewPicker(context).select(new
   picker.DocumentSelectOptions())`, take element `0`, and split the URI on `/`
   for the file name.
7. **Decode the file name.** A picked name can arrive percent-encoded; decode the
   stem and keep the extension from the **last** dot (HW-05-0068).
8. **Copy into the sandbox.** Open the source `READ_ONLY` and the destination
   `READ_WRITE | CREATE | TRUNC` (HW-05-0067), then loop
   `readSync`/`writeSync` with a 4 KB buffer, advancing `ReadOptions.offset` by
   the bytes read. Close both handles in `finally`.
9. **Pass the destination path, not the file name.** `readWriteFile(srcUri,
   attachmentUri)` - the sample passes `attachmentName`, so the copy lands
   somewhere other than the path it previews (HW-05-0063).
10. **Derive the MIME type from the extension**, using the official Preview Kit
    extension/MIME table (HW-05-0064), and return `''` for anything unknown - the
    reference states the system then infers from the URI suffix.
11. **Report every failure.** The picker rejection, both file catches and the
    preview rejection all need real handlers (HW-05-0066).
12. **Publish.** Build the `calendarManager.Event` and call
    `calendarMgr?.editEvent(event)`; treat `id > 0` as success and keep the
    `.catch()` the sample already has.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Permission request and CalendarManager publication

`ConferenceRelease.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
import { abilityAccessCtrl, common, ConfigurationConstant, Permissions, UIAbility } from '@kit.AbilityKit';
import { calendarManager } from '@kit.CalendarKit';

let mContext: common.UIAbilityContext | null = null;

onWindowStageCreate(windowStage: window.WindowStage): void {
  AppStorage.setOrCreate<window.WindowStage>('Global.windowStage', windowStage);
  windowStage.loadContent('pages/PublishMeetingPage', (err, data) => {
    if (err.code) {
      hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
      return;
    }
    windowStage.getMainWindowSync().setWindowBackgroundColor('#F1F3F5');
  });
  mContext = this.context;
  const permissions: Permissions[] = ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR'];
  let atManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(mContext, permissions).then(() => {
    // authResults is never inspected - see HW-05-0065
    AppStorage.setOrCreate('calendarMgr', calendarManager.getCalendarManager(mContext));
  }).catch((error: BusinessError) => {
    hilog.error(0xFFFF, '', `get Permission error, error. Code: ${error.code}, message: ${error.message}`);
  });
}
```

Corrected gate:

```ts
atManager.requestPermissionsFromUser(mContext, permissions)
  .then((data: PermissionRequestResult) => {
    if (data.authResults.every((status) => status === 0)) {
      AppStorage.setOrCreate('calendarMgr', calendarManager.getCalendarManager(mContext));
    } else {
      hilog.warn(0x0000, 'ConferenceRelease', 'calendar permission denied');
    }
  })
  .catch((error: BusinessError) => { /* ... */ });
```

### The chained date dialog

`ConferenceRelease.zip#entry/src/main/ets/views/CustomDialog.ets`

```ts
@CustomDialog
export struct CustomDatePickerDialog {
  @Link meetingFormData: FormData;
  controller?: CustomDialogController;
  @State isLunar: boolean = false;
  @State title: string = '';
  private customTimePickerDialogController: CustomDialogController | null = new CustomDialogController({
    builder: CustomTimePickerDialog({ meetingFormData: this.meetingFormData }),
    cornerRadius: 20,
    maskColor: '#75000000'
  });

  build() {
    Column() {
      Text(this.title);
      DatePicker({
        start: new Date('1970-1-1'),
        end: new Date('2100-1-1'),
        selected: new Date(this.meetingFormData.selectedDate)
      })
        .lunar(this.isLunar)
        .onDateChange((value: Date) => {
          let y: number = value.getFullYear();
          let m: number = value.getMonth() + 1;
          let hm = m < 10 ? Number('0' + m) : m;      // padding lost - HW-05-0069
          let d: number = value.getDate();
          let hd = d < 10 ? Number('0' + d) : d;
          const WEEK_DAYS = ['星期一', '星期二', '星期三', '星期四', '星期五', '星期六', '星期日'];
          this.title = util.format('%s年%s月%s日 %s', y, m, d, WEEK_DAYS[value.getDay()]);  // off by one - HW-05-0070
          this.meetingFormData.selectedDate = y + '-' + hm + '-' + hd;
        });

      Row() {
        Button('取消').buttonsStyle().onClick(() => {
          this.controller?.close();
          this.cancel();
        });
        Divider().vertical(true).height(40).opacity(0.6);
        Button('确定').buttonsStyle().onClick(() => {
          this.controller?.close();
          this.confirm();
          this.customTimePickerDialogController?.open();   // chain into the next dialog
        });
      };
    };
  }
}
```

Corrected date formatting and weekday lookup:

```ts
const hm = m < 10 ? `0${m}` : `${m}`;
const hd = d < 10 ? `0${d}` : `${d}`;
const WEEK_DAYS = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六'];
this.title = util.format('%s年%s月%s日 %s', y, m, d, WEEK_DAYS[value.getDay()]);
this.meetingFormData.selectedDate = `${y}-${hm}-${hd}`;
```

The neighbouring time dialog in the same file already shows the correct string
padding idiom:

```ts
let sh = '';
if (h < 10) { sh = '0' + h.toString(); } else { sh = h.toString(); }
let sm = '';
if (m < 10) { sm = '0' + m.toString(); } else { sm = m.toString(); }
this.meetingFormData.startTime = sh + ':' + sm;
```

### Attachment pick, copy and preview

`ConferenceRelease.zip#entry/src/main/ets/pages/PublishMeetingPage.ets`

```ts
uploadAttachment() {
  let documentSelectOptions = new picker.DocumentSelectOptions();
  let documentPicker = new picker.DocumentViewPicker(this.context);
  documentPicker.select(documentSelectOptions).then((documentSelectResult: Array<string>) => {
    let fileStrArr = documentSelectResult[0].split('/');
    let fileName = fileStrArr[fileStrArr.length - 1];
    this.meetingFormData.attachmentName = utfFileNameToString(fileName);
    let srcDir = documentSelectResult[0];

    this.meetingFormData.attachmentUri =
      this.filesDir + '/' + this.meetingFormData.attachmentName;
    readWriteFile(srcDir, this.meetingFormData.attachmentName);   // wrong argument - HW-05-0063
    preview(this.meetingFormData.attachmentName, fileUri.getUriFromPath(this.meetingFormData.attachmentUri),
      this.context);

  }).catch(() => {                                                 // empty - HW-05-0066
  });
}
```

Corrected call:

```ts
readWriteFile(srcDir, this.meetingFormData.attachmentUri);
```

`ConferenceRelease.zip#entry/src/main/ets/utils/FileUtils.ets`

```ts
export function readWriteFile(srcDir: string, destFileUri: string): void {
  createFile(destFileUri);
  let srcFile = fs.openSync(srcDir, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);   // HW-05-0067
  let destFile = fs.openSync(destFileUri,
    fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  try {
    let bufSize = 4096;
    let readSize = 0;
    let buf = new ArrayBuffer(bufSize);
    let readOptions: ReadOptions = {
      offset: readSize,
      length: bufSize
    };
    let readLen = fs.readSync(srcFile.fd, buf, readOptions);
    while (readLen > 0) {
      readSize += readLen;
      let writeOptions: WriteOptions = {
        length: readLen
      };
      fs.writeSync(destFile.fd, buf, writeOptions);
      readOptions.offset = readSize;
      readLen = fs.readSync(srcFile.fd, buf, readOptions);
    }
  } catch (err) {                                                  // empty - HW-05-0066
  } finally {
    fs.closeSync(srcFile);
    fs.closeSync(destFile);
  }
}
```

The chunked read/write loop itself is the part worth reusing: advance
`ReadOptions.offset` by the accumulated `readSize` and pass the actual `readLen`
as `WriteOptions.length`, so a short final chunk is not padded. Corrected open
modes:

```ts
const srcFile = fs.openSync(srcDir, fs.OpenMode.READ_ONLY);
const destFile = fs.openSync(destFileUri,
  fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE | fs.OpenMode.TRUNC);
```

### MIME lookup and preview

`ConferenceRelease.zip#entry/src/main/ets/utils/FileUtils.ets`

```ts
export function findMimeType(fileName: string): string {
  let suffixStr: string[] = fileName.split('.');
  let suffix: string = suffixStr[suffixStr.length - 1];
  switch (suffix) {
    case 'txt':  return 'text/plain';
    case 'jpg':  return 'image/jpeg';
    case 'png':  return 'image/png';
    case 'mp4':  return 'video/mp4';
    case 'pdf':  return 'application/pdf';
    case 'doc':  return 'application/msword';
    case 'docs': return 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'; // HW-05-0064
    case 'xls':  return 'application/vnd.ms-excel';
    case 'xlsx': return 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet';
    case 'ppt':  return 'application/vnd.ms-powerpoint';
    case 'pptx': return 'application/vnd.openxmlformats-officedocument.presentationml.presentation';
    case 'csv':  return 'text/comma-separated-values';                                             // HW-05-0064
    default:     return '';
  }
}

// preview file
export function preview(fileName: string, fileUri: string, context: Context) {
  let fileInfo: filePreview.PreviewInfo = {
    title: fileName,
    uri: fileUri,
    mimeType: findMimeType(fileName)
  };
  let files: Array<filePreview.PreviewInfo> = [];
  files.push(fileInfo);
  filePreview.openPreview(context, files, 0).then(() => {
  }).catch(() => {                                                 // empty - HW-05-0066
  });
}
```

Note the list overload: `openPreview(context, files, 0)` takes an
`Array<PreviewInfo>` plus a start index, which is what a multi-attachment
meeting would need.

### Percent-encoded file-name decoding

`ConferenceRelease.zip#entry/src/main/ets/utils/FileUtils.ets`

```ts
export function utfFileNameToString(utfFileName: string): string {
  let str1 = utfFileName.split('.')[0];
  let strArr = str1.split('%');
  if (strArr.length <= 1) {
    return utfFileName;
  }
  for (let i = 1; i < strArr.length; i++) {
    strArr[i] = '0x' + strArr[i];
  }
  let hexArr: number[] = [];
  for (let i = 1; i < strArr.length; i++) {
    hexArr.push(Number(strArr[i]));
  }
  let utf8ArrayFileName: Uint8Array = new Uint8Array(hexArr);
  let textDecoderOptions: util.TextDecoderOptions = { fatal: false, ignoreBOM: true };
  let decodeToStringOptions: util.DecodeToStringOptions = { stream: false };
  let textDecoder = util.TextDecoder.create('utf-8', textDecoderOptions);
  let retStr = textDecoder.decodeToString(utf8ArrayFileName, decodeToStringOptions);

  return retStr + '.' + utfFileName.split('.')[1];    // first-dot split - HW-05-0068
}
```

The `util.TextDecoder` + `Uint8Array` route is the reusable idea; the
`split('.')[0]` / `[1]` bookkeeping around it is not.

### Publishing to the system calendar

`ConferenceRelease.zip#entry/src/main/ets/pages/PublishMeetingPage.ets`

```ts
@StorageLink('calendarMgr') calendarMgr: calendarManager.CalendarManager | null = null;

Button(this.meetingIsCreated ? $r('app.string.releasing_meeting') : $r('app.string.creating_meeting'),
  { type: ButtonType.Capsule })
  .onClick(() => {
    if (this.meetingIsCreated) {
      let eventInfo: calendarManager.Event = {
        title: this.context.resourceManager.getStringSync($r('app.string.meeting').id) +
        this.meetingFormData.topic,
        type: calendarManager.EventType.NORMAL,
        startTime: new Date(this.meetingFormData.selectedDate + '-00:00').getTime() +
        timeStrToTimestamp(this.meetingFormData.startTime),
        endTime: new Date(this.meetingFormData.selectedDate + '-00:00').getTime() +
        timeStrToTimestamp(this.meetingFormData.endTime),
        reminderTime: [10],
        description: this.meetingFormData.meetingNotice
      };
      this.calendarMgr?.editEvent(eventInfo).then((id: number): void => {
        hilog.info(0xFFFF, '', `create Event id = ${id}`);
        if (id > 0) {
          this.promptAction.openToast({
            message: $r('app.string.meeting_released'),
            offset: { dx: 0, dy: 28 }
          });
        }
      }).catch((err: BusinessError) => {
        hilog.info(0xFFFF, '', `Failed to create Event. Code: ${err.code}, message: ${err.message}`);
      });
    } else {
      this.meetingIsCreated = true;
      this.promptAction.openToast({ message: $r('app.string.meeting_created'), offset: { dx: 0, dy: 28 } });
    }
  });
```

`ConferenceRelease.zip#entry/src/main/ets/utils/TimestampUtils.ets`

```ts
export function timeStrToTimestamp(timeStr: string): number {
  let addSeconds: number = 0;
  let strArr = timeStr.split(':');
  let hour = Number(strArr[0]);
  let minute = Number(strArr[1]);
  addSeconds = hour * 60 * 60 * 1000 + minute * 60 * 1000;
  return addSeconds;
}
```

## Permissions & config

`ConferenceRelease.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "pages": "$profile:main_pages",
    "requestPermissions": [
      {
        "name": "ohos.permission.READ_CALENDAR",
        "reason": "$string:reason",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      },
      {
        "name": "ohos.permission.WRITE_CALENDAR",
        "reason": "$string:reason",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      }
    ]
  }
}
```

Notes on the config:

- The declared set matches the document's 权限说明 section exactly, and both
  entries carry a complete `usedScene` with `when` - verified consistent, and
  better than the OFFICE-07 sample which omits `when` (HW-05-0041).
- Both are `user_grant`, so the runtime request in `EntryAbility` is mandatory -
  declaring them is not enough.
- **No file or storage permission.** `DocumentViewPicker` grants access to the
  single file the user picked, and the copy target is the app's own `filesDir`.
- `build-profile.json5` pins `compatibleSdkVersion` / `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Calendar access is user-granted.** Both permissions are requested at
  ability start-up; if the user denies, `getCalendarManager` still returns an
  object but the write will not succeed, so the grant must be checked.
- **`editEvent` is the publication path used here.** It is not covered by the
  reference pages available in this repository, so treat its exact behaviour -
  in particular whether it presents a system UI - as unverified and confirm
  against the current Calendar Kit reference before relying on it.
- **Preview Kit region and devices.** Preview is available only in the Chinese
  mainland and on phones, tablets and 2-in-1 devices; the attachment list still
  works elsewhere, only the preview does not.
- **The date string is the integration point.** `selectedDate` is produced by the
  dialog and re-parsed by the publish button as
  `new Date(selectedDate + '-00:00')`, so its format is a contract between two
  files - keep it a zero-padded `yyyy-MM-dd`.
- **Attachment names may be percent-encoded.** The URI segment returned by the
  document picker is not always the display name, which is why the sample
  decodes it before using it as a sandbox file name.
- **One attachment at a time.** `FormData` holds a single `attachmentName` /
  `attachmentUri` pair, though `preview` already takes an array and would support
  more.

## Pitfalls

- **`readWriteFile(srcDir, this.meetingFormData.attachmentName)` is incorrect.**
  The second parameter is the destination path; passing the bare file name copies
  the bytes somewhere other than the `attachmentUri` that the very next line
  previews. Pass `attachmentUri`. (HW-05-0063)
- **`case 'docs':` is incorrect** - the OOXML Word extension is `docx`, so real
  Word attachments fall through to the empty default - and `csv` should map to
  `text/csv`, not `text/comma-separated-values`. (HW-05-0064)
- **Publishing the `CalendarManager` without checking `authResults` is
  incorrect.** `requestPermissionsFromUser` resolves on denial too; gate the
  manager creation on every entry being `0`. (HW-05-0065)
- **Four empty error handlers are incorrect.** The two `catch (err) {}` blocks in
  `FileUtils`, the empty `.catch(() => {})` on the picker and the empty
  `then`/`catch` on `openPreview` between them hide the entire attachment
  pipeline - which is exactly why HW-05-0063 produces no visible symptom.
  (HW-05-0066)
- **Opening the source `READ_WRITE | CREATE` and the destination without `TRUNC`
  is incorrect.** A missing source path is silently created as an empty file, and
  re-attaching a shorter file leaves the previous file's trailing bytes in the
  sandbox copy. (HW-05-0067)
- **`split('.')[0]` / `split('.')[1]` in `utfFileNameToString` is incorrect** for
  any name containing more than one dot: the name is truncated at the first dot
  and the wrong segment is used as the extension. Use `lastIndexOf('.')`.
  (HW-05-0068)
- **`Number('0' + m)` is incorrect** as zero-padding - `Number('08')` is `8`, so
  the leading zero is discarded and `selectedDate` becomes `2026-8-5`. Keep it a
  string, as the time dialog in the same file does. (HW-05-0069)
- **Indexing a Monday-first weekday array with `Date.getDay()` is incorrect** -
  `getDay()` returns `0` for Sunday, so every label in the date dialog title is
  shifted by one day. Reorder the array to start at Sunday. (HW-05-0070)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/07_application-services/preview-introduction.md` -
  the supported extension/MIME table used to check `findMimeType`, plus the
  region, device and emulator limits.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/preview-introduction
- `documentation/harmonyos-references/03_application-services/preview-arkts.md` -
  `openPreview` overloads including the `Array<PreviewInfo>` form, and the
  `mimeType` inference rule for an empty string.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/preview-arkts
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `OpenMode` flags (`READ_ONLY`, `READ_WRITE`, `CREATE`, `TRUNC`), `readSync` /
  `writeSync` with `ReadOptions` / `WriteOptions`, and `closeSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` -
  `DocumentViewPicker.select` and `DocumentSelectOptions`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  checking `authResults` after `requestPermissionsFromUser`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  the meaning of `authResults`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and `when` for user_grant permissions.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
