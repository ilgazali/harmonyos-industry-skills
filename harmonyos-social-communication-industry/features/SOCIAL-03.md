---
id: SOCIAL-03
title: Hold-to-talk voice input - streaming ASR results into a RichEditor, with drag-up to cancel
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/03_voice_to_text_forchat.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016
sample: huawei_industry_tree/14_social_communication/downloads/VoiceToTextForChatDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreSpeechKit", "@kit.CryptoArchitectureKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: ["speechRecognizer.createEngine", startListening, setListener, onResult, finish, shutdown, RichEditorController, addTextSpan, deleteSpans, getCaretOffset, setCaretOffset, getSpans, ComponentContent, "PromptAction.openCustomDialog", closeCustomDialog, showToast, GestureGroup, "GestureMode.Sequence", LongPressGesture, PanGesture, bindContentCover, "abilityAccessCtrl.requestPermissionsFromUser", "media.AVRecorder", getAudioCapturerMaxAmplitude, "cryptoFramework.createRandom", setKeyboardAvoidMode, hilog]
permissions: [ohos.permission.MICROPHONE]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0005, HW-14-0006, HW-14-0007, HW-14-0008, HW-14-0003, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a chat input bar needs a press-and-hold voice button that
types instead of sending audio** - the 按住 说话 (hold to talk) control, but
producing editable text rather than a voice message. A long-press starts a Core
Speech Kit session, partial results are written into the live `RichEditor` as
they arrive, and releasing finalises them so the user can still edit before
sending.

The transferable mechanism is **streaming replacement in a rich-text
controller.** ASR delivers a growing hypothesis, not append-only tokens, so each
partial result must *replace* the previous one: remember the caret offset where
the transcription began and the length of what was last written, then delete
that exact span before writing the new one. The same shape covers live
translation, live captions, or any incremental model output rendered into an
editable field.

**Read `HW-14-0005` and `HW-14-0006` before adopting it.** The drag-up-to-cancel
gesture in this sample neither stops the recogniser nor scopes its deletion, so
cancelling leaves the microphone live and destroys the user's whole draft. Both
are three-line fixes, but they are on the path every user takes.

## Feature checklist

- A chat page with a message list and a rich input bar (text + inline emoji);
  the leftmost icon toggles between keyboard and voice input mode.
- In voice mode the input bar becomes a 语音识别输入 (speech recognition input)
  strip; tapping it raises a full-screen voice overlay.
- Press and hold on the overlay starts recognition; the label flips from
  按住 说话 to 松开 停止 (release to stop).
- A floating dialog shows the running transcription and a live waveform driven
  by the recorder's real amplitude.
- Recognised text streams into the chat input in place, replacing the previous
  partial result each time.
- Dragging the finger up onto the cancel target abandons the input; releasing
  anywhere else commits it.
- A 20 s watchdog closes the dialog and stops recognition with a 录音时间到
  (recording time is up) toast.
- Without the microphone permission the long press shows 无麦克风权限 and does
  nothing else.

## Architecture

One `entry` module, one page and four views. The engine and the recorder are
each wrapped in a plain class under `utils/`, neither of which is an ArkUI
component.

```
entry/src/main/ets
├── entryability/EntryAbility.ets      window setup, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Chat.ets                     @Entry: message List + CustomRichEditor; owns @Provide state
├── utils
│  ├── AudioRecorder.ets               a second AVRecorder, used only for the waveform amplitude
│  ├── Interface.ets                   EditMenuAction / MessageType enums, MsgContent
│  ├── RequestPermission.ets           generic requestPermissionsFromUser helper
│  └── SpeechRecognizer.ets            thin wrapper over speechRecognizer: init/start/stop/shutdown
└── view
   ├── CustomRichEditor.ets            the input bar; exports the shared `richController` singleton
   ├── EmojiView.ets                   emoji grid, addImageSpan into the same controller
   ├── SpeechView.ets                  the voice overlay: gestures, placeholder, span surgery
   └── VoiceToTextView.ets             the floating dialog: waveform + transcription preview
```

The documented tree matches the zip.

**The design decision worth copying** - with one caveat - is the exported
controller singleton at the top of `CustomRichEditor.ets`:

```typescript
export const richController: RichEditorController = new RichEditorController();
```

`SpeechView` and `EmojiView` both import it directly. That is why a voice
transcription and a tapped emoji land in the same editor at the same caret
without any prop plumbing, `@Link`, or callback chain between three sibling
components. For a controller - an imperative handle, not state - a module-level
singleton is the right call: it is not rendered, so it does not need to be
observable.

The caveat is that it is a *module* singleton, so exactly one `RichEditor` may
exist in the app - two chat tabs would silently share one caret.

Note also that the sample runs **two audio consumers at once**: Core Speech
Kit's own capture (pcm/16k/mono) for the words, and a separate
`media.AVRecorder` (AAC/48k/stereo, written to `filesDir` and unlinked on stop)
purely to poll `getAudioCapturerMaxAmplitude()` every 200 ms for the waveform
bars. The ASR engine exposes no amplitude, so a decorative waveform costs you a
second microphone stream and a temp file - know that before you copy it.

## Implementation steps

1. **Create the engine once, in `aboutToAppear`**, with
   `recognizerMode: 'long'` for dictation-length input, and `shutdown()` it in
   `aboutToDisappear`.
2. **Request `ohos.permission.MICROPHONE` up front** and keep the result in
   `isPermitted`; gate the long-press on it. Check *every* element of
   `authResults`, do not return inside the first iteration (`HW-14-0008`).
3. **Capture the caret offset before the first result arrives**
   (`fillPlaceHolder`), and insert a dim `...` placeholder only when the editor
   already has content.
4. **Track whether that placeholder was actually inserted**, and delete exactly
   those 3 characters only if it was (`HW-14-0007`).
5. **Replace, do not append, on each partial result**: delete
   `[caretOffset, caretOffset + lastLength)` then `addTextSpan` at
   `caretOffset`, and remember the new length.
6. **On `isFinal`, re-read the caret offset and reset the length to 0**, so the
   next utterance starts a fresh replacement window instead of overwriting the
   finalised text.
7. **Wire the gestures as `GestureMode.Sequence`**: `LongPressGesture` first,
   then `PanGesture`, so a drag is only interpreted after the press has been
   held.
8. **In the cancel branch, stop the recogniser** - not just the UI
   (`HW-14-0005`).
9. **Delete only the voice-inserted range on cancel**, never call
   `deleteSpans()` with no argument (`HW-14-0006`); and clear the 20 s watchdog
   in the dialog's `onWillDisappear`, which the sample does correctly.

## Verified snippets

All snippets are from `VoiceToTextForChatDemo.zip`. Corrected forms are marked.

**The engine wrapper - `entry/src/main/ets/utils/SpeechRecognizer.ets`** (as shipped)

```typescript
import { speechRecognizer } from '@kit.CoreSpeechKit';

export class SpeechRecognizer {
  private engineParams: speechRecognizer.CreateEngineParams = {
    language: 'zh-CN',
    online: 1,
    extraParams: { 'locate': 'CN', 'recognizerMode': 'long' }
  };
  private asrEngine?: speechRecognizer.SpeechRecognitionEngine;
  private sessionId: string = 'SpeechRecognizer_' + Date.now();

  public async intiEngine() {
    this.asrEngine = await speechRecognizer.createEngine(this.engineParams);
  }

  public stop() {
    this.asrEngine?.finish(this.sessionId);
  }

  public shutdown() {
    this.asrEngine?.shutdown();
  }

  private startListening() {
    let recognizerParams: speechRecognizer.StartParams = {
      sessionId: this.sessionId,
      audioInfo: {
        audioType: 'pcm',
        sampleRate: 16000,
        soundChannel: 1,
        sampleBit: 16
      },
      extraParams: {
        recognitionMode: 0,
        maxAudioDuration: 60000,
        vadBegin: 500,
        vadEnd: 10000,
      }
    };
    this.asrEngine?.startListening(recognizerParams);
  }
}
```

**Four parameters carry the design.** `recognizerMode: 'long'` selects
continuous dictation rather than short-command recognition - short mode would
close the session after the first utterance. `recognitionMode: 0` is microphone
input (as opposed to feeding audio buffers yourself). The `audioInfo` block is
not free-form: Core Speech Kit's microphone mode accepts pcm/16000/mono/16-bit
and this is that exact combination. And `vadEnd: 10000` means ten seconds of
silence before the engine decides you are done - generous on purpose, so a
thinking pause does not truncate a message.

`maxAudioDuration: 60000` is the hard ceiling that makes `HW-14-0005` matter:
after a cancel that never calls `finish()`, the session stays open for the
remainder of that minute and keeps calling back. Note `stop()` is
`finish(sessionId)` - it ends the session, and `start()` then re-runs
`setListener` + `startListening` on the same id, which is stamped once at
construction from `Date.now()`.

**Streaming a partial result into the editor - `entry/src/main/ets/view/SpeechView.ets`** (corrected, see `HW-14-0007`)

```typescript
private placeholderInserted: boolean = false;   // FIX: absent in the sample

private startSpeechRecognizer() {
  this.fillPlaceHolder();
  this.speechRecognizer.start((result) => {
    this.insertSpan(result.result);
    this.speechText = result.result;
    if (result.isFinal) {
      this.finalText += result.result;
      this.caretOffset = richController.getCaretOffset();
      this.speechTextLength = 0;
    }
    // and a second isFinal branch feeding the floating dialog:
    // contentNode?.update(isFinal ? finalText : finalText + speechText)
  });
}

private insertSpan(text: string) {
  if (text.length <= 0) {
    return;
  }
  if (this.speechTextLength === 0) {
    richController.addTextSpan(text, { offset: this.caretOffset });
  } else {
    richController.deleteSpans({ start: this.caretOffset, end: this.caretOffset + this.speechTextLength });
    richController.addTextSpan(text, { offset: this.caretOffset });
  }
  this.speechTextLength = text.length;
}

private fillPlaceHolder() {
  this.caretOffset = richController.getCaretOffset();
  this.placeholderInserted = richController.getSpans().length > 0;   // FIX
  if (this.placeholderInserted) {
    richController.addTextSpan('...', {
      offset: this.caretOffset,
      style: { fontColor: '#4D242E3E', fontSize: 13 }
    });
    richController.setCaretOffset(this.caretOffset);
  }
}

private stopSpeechRecognizer() {
  this.placeHolder = '';
  this.speechRecognizer.stop();
  if (this.placeholderInserted) {                                    // FIX: was unconditional
    richController.deleteSpans({ start: this.caretOffset, end: this.caretOffset + 3 });
    this.placeholderInserted = false;
  }
}
```

**`insertSpan` is the whole streaming trick.** `speechTextLength` remembers how
many characters the previous partial result wrote; the next one deletes exactly
that range and rewrites from the same `caretOffset`. Because the anchor never
moves during an utterance, the replacement is stable even as the hypothesis
grows from "今" to "今天天气". On `isFinal` the anchor advances to the new caret
and the length resets to 0, which promotes the finalised text out of the
replacement window and starts a fresh one behind it.

The correction is the placeholder bookkeeping. `fillPlaceHolder` inserts `...`
**only when the editor already had spans**, but `stopSpeechRecognizer` deleted
three characters unconditionally - so on the first use with an empty editor
those three characters come off the head of the freshly inserted
transcription. A boolean is enough; hardcoding `3` is acceptable only because
the placeholder is a literal.

**Press, drag and cancel - same file** (corrected, see `HW-14-0005`, `HW-14-0006`)

```typescript
GestureGroup(GestureMode.Sequence,
  LongPressGesture({ repeat: false })
    .onAction(() => {
      if (this.isPermitted) {
        this.isLoosen = '松开  停止';
        this.startSpeechRecognizer();
        this.contentNode =
          new ComponentContent<string>(this.uiContext, wrapBuilder<[string]>(voiceToTextViewBuilder),
            this.speechText);
        this.uiContext.getPromptAction().openCustomDialog(this.contentNode, {
          alignment: DialogAlignment.Center,
          showInSubWindow: true,
          onDidAppear: () => {
            // 20 s watchdog: toast 录音时间到, close the dialog, stopSpeechRecognizer(), isShow = false
            this.timerId = setInterval(async () => { /* ... */ }, 20000);
          },
          onWillDisappear: () => {
            clearInterval(this.timerId);
          }
        });
      } else {
        this.uiContext.getPromptAction().showToast({ message: '无麦克风权限' });
      }
    })
    .onActionEnd(() => {
      if (this.contentNode) {
        this.uiContext.getPromptAction().closeCustomDialog(this.contentNode);
      }
      this.isLoosen = '按住  说话';
      this.stopSpeechRecognizer();
      this.isShow = false;
    }),
  PanGesture()
    .onActionUpdate((event: GestureEvent) => {
      for (let i = 0; i < event.fingerList.length; i++) {
        this.positionY = event.fingerList[i].localY;
        this.positionX = event.fingerList[i].localX;
      }
    })
    .onActionEnd(() => {
      if (this.positionY >= -200 && this.positionY <= -50 && this.positionX >= 140 && this.positionX <= 200) {
        this.dragPosition = 1;
      } else {
        this.dragPosition = -1;
      }
      if (this.dragPosition === 1) {
        this.speechText = '';
        this.isShow = false;
        this.speechRecognizer.stop();                     // FIX: cancel must stop the engine
        richController.deleteSpans({                      // FIX: was rangeless deleteSpans()
          start: this.caretOffset,
          end: this.caretOffset + this.speechTextLength + (this.placeholderInserted ? 3 : 0)
        });
        this.speechTextLength = 0;
        this.placeholderInserted = false;
      } else {
        this.isLoosen = '按住  说话';
        this.stopSpeechRecognizer();
      }
      this.positionY = 0;
      this.positionX = 0;
      if (this.contentNode) {
        this.uiContext.getPromptAction().closeCustomDialog(this.contentNode);
      }
      this.isShow = false;
    })
)
```

**`GestureMode.Sequence` is what makes press-then-drag one gesture.** In a
sequence group the gestures must be recognised in declaration order, so the
`PanGesture` only becomes eligible after the `LongPressGesture` has already
fired - a bare swipe over the overlay does nothing, and the cancel target only
exists while the user is holding.

The two corrections are on the cancel path, which the shipped code treats as a
pure UI reset. `speechRecognizer.stop()` (i.e. `finish(sessionId)`) is called
**only** in the else branch, so cancelling leaves the session live for the rest
of `maxAudioDuration` and every later `onResult` still runs `insertSpan` into
the input the user just cancelled. And `deleteSpans()` with no argument means
"clear everything" - the sample uses that same rangeless form deliberately in
`sendMessage()` to empty the box after a send, so on cancel it destroys any
typed draft the voice input was appended to. `EmojiView.deleteSpans()` in the
same project gets this right: `{ start: offset - 1, end: offset }`.

The cancel target is a raw `localY -200..-50, localX 140..200` rectangle,
matched against the same literals in the overlay's `Image` selector - four
magic numbers in two places; extract them before reuse.

**The permission helper - `entry/src/main/ets/utils/RequestPermission.ets`** (corrected, see `HW-14-0008`)

```typescript
import { abilityAccessCtrl, common, Permissions } from '@kit.AbilityKit';

export async function requestPermission(permissions: Permissions[],
  context: common.UIAbilityContext): Promise<boolean> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let data = await atManager.requestPermissionsFromUser(context, permissions);
  let grantStatus: Array<number> = data.authResults;
  for (let i = 0; i < grantStatus.length; i++) {
    if (grantStatus[i] !== 0) {
      return false;        // FIX: only the failure returns inside the loop
    }
  }
  return grantStatus.length > 0;   // FIX: shipped code returns true on the first granted element
}
```

The signature takes `Permissions[]`, so this reads as a general helper - and it
is imported as one. But the shipped loop has `return true` in the `else` branch,
which decides the whole result from `authResults[0]` and never looks at the
rest. With `['MICROPHONE']` it happens to be correct; with two permissions it
reports success when the first is granted and the second refused. The identical
shape appears in the `BluetoothSPP` sample's `PermissionsUtil.ets`, whose
`checkPermissions` additionally *overwrites* its accumulator each iteration so
only the last permission decides.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "always"
    },
  }
]
```

- `MICROPHONE` is `user_grant`, so `reason` and `usedScene` are mandatory and
  `microphone_reason` must exist in `resources/base/element/string.json`.
- `when: "always"` is **stronger than this feature needs.** Recognition only
  runs while the user holds the button and there is no continuous task or
  background ability; `"inuse"` is the correct scope. Same copy-pasted
  permission-block carelessness as `HW-14-0003`, here as over-scoping rather
  than an entirely unused declaration.
- The permission is requested eagerly in `SpeechView.aboutToAppear`, i.e. when
  the voice overlay first mounts rather than at app launch - the right moment,
  since the dialog appears with the feature the user just reached for.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `online: 1` in `CreateEngineParams` selects the on-device engine; `language`
  is fixed to `'zh-CN'` with `locate: 'CN'`. Other locales need both changed.
- `maxAudioDuration` is 60 s per session; the sample's own watchdog cuts in at
  20 s.
- The `RichEditorController` is a module-level singleton, so exactly one
  `RichEditor` instance may be live.
- The waveform's `calcHeight` calls `cryptoFramework.createRandom()` **once per
  bar per frame**; a CSPRNG is an expensive source of decoration. And
  `AudioRecorder.deleteAudioFile` calls `closeSync`/`unlinkSync` with no
  try/catch, so a stop before the file was created throws.

## Pitfalls

- **`HW-14-0005`** (B/medium, confirmed): the drag-to-cancel branch clears UI
  state only - `speechRecognizer.stop()` lives solely in the else branch, so the
  microphone stays live for the rest of the 60 s session and later `onResult`
  callbacks keep transcribing into the cancelled input. Fix: call `stop()` in
  the cancel branch too.
- **`HW-14-0006`** (B/medium, confirmed): the cancel branch calls
  `richController.deleteSpans()` with no range, which clears the entire editor.
  Type a draft, add voice, cancel - the draft is gone. Fix: delete only
  `[caretOffset, caretOffset + speechTextLength]` (plus the placeholder if one
  was inserted), as `EmojiView` already does.
- **`HW-14-0007`** (B/medium, confirmed): `stopSpeechRecognizer` always deletes
  3 characters to remove a `...` placeholder that `fillPlaceHolder` inserts only
  when `getSpans().length > 0`. First use on an empty editor chops three
  characters off the transcription. Fix: track a `placeholderInserted` flag.
- **`HW-14-0008`** (B/low, confirmed): `requestPermission` returns from inside
  the first loop iteration, so a multi-permission call is decided by
  `authResults[0]`. Same defect in `BluetoothSPP/PermissionsUtil.ets`, whose
  checker also overwrites its accumulator per iteration. Fix: return `false`
  in-loop, `true` after.
- **`HW-14-0003`** (D/low, confirmed): the industry's copy-pasted permission
  template. Here it shows as `when: "always"` on a permission only needed
  `"inuse"`; in four sibling samples it shows as entirely unused `INTERNET` /
  `VIBRATE` declarations and dead `LOCATION` constants. Audit
  `requestPermissions` against actual call sites before shipping.
- Two microphone consumers run concurrently (the ASR engine and a second
  `AVRecorder` for the waveform). Where microphone access is serialised, the
  decorative recorder can starve the recogniser.

## References

- `documentation/harmonyos-references/07_ai/hms-ai-speechrecognizer.md` - `createEngine`, `StartParams`, `RecognitionListener`, `finish`, `shutdown`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/hms-ai-speechrecognizer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController`, `addTextSpan`, `deleteSpans`, `getCaretOffset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureMode.Sequence`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `openCustomDialog` / `closeCustomDialog` with `ComponentContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser` and `authResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.MICROPHONE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the user_grant request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `SOCIAL-01` - the industry index this card hangs off
