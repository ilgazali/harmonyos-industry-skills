---
id: UTIL-06
title: Ringtone setting - stage an audio file into the sandbox, then hand its path to Ringtone Kit
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/06_ringtone.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ringtone-0000002284229333
sample: huawei_industry_tree/15_utilities/downloads/铃声设置示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.RingtoneKit"]
apis: ["ringtone.startRingtoneSetting", "picker.AudioViewPicker", "picker.AudioSelectOptions", "fs.listFileSync", ListFileOptions, "fs.openSync", "fs.writeSync", "fs.copyFileSync", "fs.copyDirSync", "fs.mkdirSync", "fs.accessSync", "fs.closeSync", "resourceManager.getRawFileListSync", "resourceManager.getRawFileContentSync", List, ListItem, swipeAction, "UIContext.showAlertDialog", "@StorageLink"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0022, HW-15-0023, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the app must **let the user set a system ringtone** -
alarm, incoming call, message or notification - from audio the app ships or
audio the user picks off the device. The whole feature is two moves: get the
audio into your own sandbox as a real file, then call
`ringtone.startRingtoneSetting(context, path, name)` and let the system sheet
do the rest.

The important constraint is the one that shapes the code: **Ringtone Kit takes
a sandbox path, not a URI and not a resource.** A `rawfile` you shipped is not
a file on disk, and a URI handed back by the audio picker points into the
media store, not into your app. Both have to be materialised under
`context.filesDir` first. That is why a "ringtone" sample is 80% file I/O.

It generalises to any feature that hands a file to a system service by path -
wallpaper setting, sharing a packed image (see `UTIL-08`), attaching to a
system share sheet. The staging step, the picker, the directory listing and
the swipe-to-delete list are all reusable; **the delete is not** - read
`HW-15-0022` first.

## Feature checklist

- Two sections: 推荐铃声 (recommended ringtones, from the app's own rawfiles)
  and 选择音乐 (choose music, added from the device).
- Each section is a scrolling `List` capped at 30% of the page height.
- Names longer than 12 characters are truncated with an ellipsis; tapping the
  name opens an alert dialog with the full name.
- The first and last rows of each list get rounded outer corners; the ones
  between are square, so a list reads as one card.
- A 刷新 (refresh) link re-lists the sandbox audio directory.
- A 添加 (add) link opens the system audio picker and copies the chosen file
  into the sandbox.
- The gear icon on a row raises the system ringtone-setting sheet for that
  file.
- Swiping a row left reveals a red 删除 (delete) button.
- On first launch the six bundled mp3s are copied out of `rawfile` into
  `filesDir/audio/`.

## Architecture

One `entry` module, four source files. The page is the feature.

```
entry/src/main/ets
├── common/Constants.ets             paths ('audio/', '/audio/'), suffix filters, percent literals
├── entryability/EntryAbility.ets    full screen, status-bar height -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/SettingRingtone.ets        the whole feature, 332 lines
└── utils/Logger.ets                 hilog wrapper
```

`entry/src/main/resources/rawfile/audio/` holds `audio1.mp3` … `audio6.mp3`.
There is **no** `resources/resfile` directory - which matters, see
`HW-15-0023`. The documented tree matches the zip.

**The design decision worth copying** is that the sandbox directory, not an
in-memory array, is the source of truth for the first list. `onPageShow` calls
`findListFileSync(this.audioDir)` and assigns the result straight to
`@State audioData`, so the UI is a projection of the filesystem and a
`刷新` link is enough to resynchronise it after anything - a copy, an
external change, a return from the ringtone sheet.

The trap is that the second list does not follow the same rule.
`audioDataPhone` is only ever appended to in the picker callback, and nothing
re-derives it, so picked files survive on disk but vanish from the list on
every relaunch. And because the first list *is* derived from the directory,
the delete button that only edits the array is undone by the very next
refresh (`HW-15-0022`). One list is honest about where its data lives and the
other is not; copy the first.

## Implementation steps

1. **Stage the bundled audio.** Enumerate `rawfile/audio` with
   `getRawFileListSync`, read each entry with `getRawFileContentSync`, and
   write the `ArrayBuffer` to `filesDir/audio/<name>` with an
   `openSync` + `writeSync` + `closeSync` in a `finally`.
2. **Do not also try to copy from `resourceDir`.** The sample's `copyRawfile`
   does, the project ships no `resfile`, and the call throws on every launch
   (`HW-15-0023`).
3. **List the sandbox directory with a suffix filter** (`.mp3`, `.wav`,
   `.OGG`) and `recursion: false`. Keep the copy destination flat - a nested
   `audio/audio/` is invisible to a non-recursive lister.
4. **Add device audio through `picker.AudioViewPicker`.** The picker returns
   URIs; open the URI read-only, read `file.name` off the returned `File`, and
   `copyFileSync(file.fd, dest.fd)` into the sandbox. Close both descriptors
   in a `finally`.
5. **Call `ringtone.startRingtoneSetting(context, sandboxPath, fileName)`** on
   the gear tap and let the system sheet choose the category. Build the path
   from the same constant the copy used, or the two lists disagree.
6. **Wrap the row in `ListItem().swipeAction({ end: ... })`** with a `@Builder`
   returning the delete button.
7. **Unlink the file in the delete handler**, not just the array entry
   (`HW-15-0022`).
8. **Re-derive both lists in `onPageShow`** so the picker-added files survive
   a relaunch.

## Verified snippets

All snippets are from `铃声设置示例代码.zip` (`Ringtone/`). Corrected forms
are marked.

**Staging rawfiles into the sandbox — `entry/src/main/ets/pages/SettingRingtone.ets`** (as shipped)

```typescript
copyRawFileListToSdcard() {
  // 1. read the raw data out of rawfile
  let rawFileAll = this.context?.resourceManager.getRawFileListSync(Constants.MUSIC_PATH_0); // 'audio'
  rawFileAll?.forEach(str => {
    let rawFile = this.context?.resourceManager.getRawFileContentSync(Constants.MUSIC_PATH_1 + str); // 'audio/'
    let arrayBuffer: ArrayBuffer = rawFile?.buffer as ArrayBuffer;

    // 2. build the sandbox destination path
    let sandboxPath = this.context?.filesDir + Constants.MUSIC_PATH_2 + str;   // '/audio/'
    let fd = fs.openSync(sandboxPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    try {
      // 3. write the bytes
      let bytesWritten = fs.writeSync(fd.fd, arrayBuffer, {
        offset: 0,
        length: arrayBuffer.byteLength
      });
      Logger.info('copy rawfile' + bytesWritten + 'to' + sandboxPath);
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      Logger.error('copy rawfile failed with error message: ' + err.message + ' , error code: ' + err.code);
    } finally {
      // 4. always close the descriptor
      fs.closeSync(fd);
    }
  });
}
```

**A rawfile is not a file.** It lives inside the HAP, and neither Ringtone Kit
nor any other path-taking system service can reach it, so the only way to
offer bundled audio as a ringtone is to materialise it under `filesDir`. The
three-constant dance (`'audio'` for the list call, `'audio/'` for the content
call, `'/audio/'` for the sandbox path) is the API's doing:
`getRawFileListSync` wants a directory with no trailing slash,
`getRawFileContentSync` wants a path relative to `rawfile`, and the sandbox
path is concatenated onto `filesDir` which has no trailing slash of its own.

`openSync` with `READ_WRITE | CREATE` creates the file if it is missing and
opens it in place if it is not - so this runs unconditionally on every
`aboutToAppear` and simply rewrites the same six files. That is wasteful but
safe. `closeSync` in the `finally` is the part not to skip: an fd leaked per
launch is an fd leaked forever.

**Pulling a file off the device — same file** (as shipped)

```typescript
findAudioFromPhone() {
  let audioSelectOptions = new picker.AudioSelectOptions();
  let audioPicker = new picker.AudioViewPicker(this.context);
  audioPicker.select(audioSelectOptions).then(async (audioSelectResult: Array<string>) => {
    let file = fs.openSync(audioSelectResult[0], fs.OpenMode.READ_ONLY);   // the picker returns a URI
    let fileName = file.name;
    let audioPath: string = `${this.context?.filesDir}/${fileName}`;
    this.audioDataPhone[this.audioDataPhone.length] = fileName;
    let file2 = fs.openSync(audioPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    try {
      fs.copyFileSync(file.fd, file2.fd);
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      Logger.error('copy directory failed with error message: ' + err.message + ', error code: ' + err.code);
    } finally {
      fs.closeSync(file);
      fs.closeSync(file2);
    }
  }).catch((err: BusinessError) => {
    Logger.error('AudioViewPicker.select failed with err: ' + JSON.stringify(err));
  });
}
```

**The picker is why this feature needs no permissions.** `AudioViewPicker`
runs in its own process and hands back a URI the app is granted temporary read
access to; there is no `READ_MEDIA` in `module.json5` and none is needed. That
is the pattern to copy whenever a user picks their own content - a declared
media permission for the same job would be rejected in review.

Two details to fix when adapting it. `fs.openSync` on a picker URI works, but
the URI is transient: it must be copied now, inside this callback, not stored
for later. And the copy target is `filesDir/<name>` while the bundled files go
to `filesDir/audio/<name>` - hence the two different path expressions in the
two gear handlers below. Picking a file whose name collides with an existing
one silently overwrites it, and picking one twice appends a duplicate row.

**Raising the ringtone sheet — same file** (as shipped)

```typescript
Image($r('app.media.setting'))
  .width(20)
  .height(22)
  .align(Alignment.Center)
  .onClick(() => {
    try {
      let audioPath = this.context.filesDir + Constants.MUSIC_PATH_2 + item;  // '/audio/' + name
      let fileName = item;
      ringtone.startRingtoneSetting(this.context, audioPath, fileName).then(res => {
        Logger.info('startRingtoneSetting success', res.toString());
      });
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      Logger.info(err.message + err.code);
    }
  });
```

**This is the entire Ringtone Kit surface the sample uses.** Three arguments -
the UIAbility context, the sandbox path, and the display name the sheet shows
- and the system takes over: it presents the category chooser (call, alarm,
notification, message), applies the selection, and resolves. The app never
touches system settings, never asks for a ringtone permission, and never plays
the audio itself.

The `try` around it catches synchronous argument errors only; the returned
promise has no `.catch`, so a rejection (missing file, unsupported format)
goes unobserved. Add one. Note also that the second list's copy of this
handler uses `filesDir + '/' + item` because picked files land one directory
higher - a good argument for routing every staged file through one helper.

**Swipe-to-delete — same file** (corrected, see `HW-15-0022`)

```typescript
@Builder
DeleteButton0(item: string) {
  Button($r('app.string.delete'))
    .width(80)
    .height(Constants.FULL_PERCENT)
    .backgroundColor(Color.Red)
    .onClick(() => {
      let index = this.audioData.indexOf(item);
      if (index !== -1) {
        fs.unlinkSync(this.context.filesDir + Constants.MUSIC_PATH_2 + item); // FIX: absent in the sample
        let newData = [...this.audioData];
        newData.splice(index, 1);
        this.audioData = newData;                    // reassign, do not splice in place
      }
    });
}

// attached to the row:
ListItem() {
  // ... the Row with the name and the gear
}
.swipeAction({
  end: this.DeleteButton0(item)
});
```

**`swipeAction({ end })` is the whole gesture** - no pan handler, no offset
maths; ArkUI reveals whatever the builder emits when the item is dragged from
the trailing edge. Giving the button `height('100%')` is what makes it fill
the row rather than float in it.

The reassignment (`this.audioData = newData`) rather than an in-place
`splice` is deliberate and correct: `@State` on an array observes assignment,
so mutating in place would not re-render. But without the `unlinkSync` the
file is still on disk, and since `onPageShow` and the 刷新 link both rebuild
`audioData` from `listFileSync`, every deleted ringtone comes straight back.
The delete looks like it works only for as long as nobody refreshes.

**The dead resfile copy — same file** (corrected, see `HW-15-0023`)

```typescript
aboutToAppear(): void {
  this.copyRawFileListToSdcard();          // FIX: was `this.copyRawfile().then(() => ...)`
}

// FIX: delete copyRawfile() entirely - it targets this.context.resourceDir + '/audio/',
// the project ships no resources/resfile, and fs.copyDirSync throws on every launch.
// The error is swallowed by its own catch, which is why the sample appears to work:
// copyRawFileListToSdcard() in the .then() is what actually stages the audio.
```

The shipped `copyRawfile` also has a second bug behind the first: `copyDirSync`
with mode `0` copies the *directory*, so even with a `resfile/audio` present
the result would be `filesDir/audio/audio/`, one level below what
`findListFileSync` looks at with `recursion: false`. Two defects stacked in a
path that never executes successfully - the cleanest fix is removal.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, and that is the
correct design: the audio picker grants transient access to exactly the file
the user chose, and Ringtone Kit's sheet is itself the user's consent for the
setting. `deviceTypes` is `["phone", "tablet"]`.

`EntryAbility` publishes the status-bar height into `AppStorage`; the page
reads it with `@StorageLink('topRectHeight')` and converts with `px2vp` into
the title's top margin. `@StorageLink` is two-way here where `@StorageProp`
(one-way) is what the page actually needs - it only ever reads the value.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `ringtone.startRingtoneSetting` requires a **sandbox path**. Passing a
  picker URI, a `$rawfile` reference or a media-store URI does not work.
- The suffix filter is `['.wav', '.OGG', '.mp3']` - case-sensitive as written,
  so a `.ogg` or `.MP3` file staged into the directory is silently not listed.
- Both lists are capped at `constraintSize({ minHeight: '30%', maxHeight: '30%' })`,
  which is a fixed 30% of the page whatever the content - a single ringtone
  still occupies the full block.
- `audioDataPhone` is never re-derived from disk, so device-picked entries
  disappear from the list on relaunch even though the files remain staged.
- There is no preview playback in the sample despite it being a ringtone
  picker; the only audition is the system sheet's own.

## Pitfalls

- **`HW-15-0022`** (B/medium, confirmed): swipe-delete is fake - the handler
  removes only the in-memory entry and nothing calls `fs.unlinkSync`, while
  `onPageShow` and the 刷新 link rebuild `audioData` from the directory, so
  every "deleted" ringtone reappears within the same session. Fix: unlink the
  sandbox file (or do not offer delete).
- **`HW-15-0023`** (B/low, confirmed): `copyRawfile()` copies from
  `context.resourceDir` (`resfile`), which the project does not ship, so
  `copyDirSync` throws on every `aboutToAppear`; the error is swallowed and
  the app works only because of the separate rawfile copy chained after it.
  Even with a `resfile` present, `copyDirSync` would nest `audio/audio/` where
  the non-recursive lister cannot see it. Fix: drop the resfile branch.

## References

- `documentation/harmonyos-references/04_media/ringtone-ringtone.md` - `ringtone.startRingtoneSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ringtone-ringtone
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `writeSync`, `copyFileSync`, `copyDirSync`, `listFileSync`, `ListFileOptions`, `unlinkSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItem`, `swipeAction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `UTIL-08` - the other sample in this industry that stages a file into the sandbox before handing it to a system service
