# Pitfalls

> Generated from `features/*.md`. Source industry: `08_children_education`, 17 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-08-0120` - Systematic: 14 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: KIDS-02, KIDS-03, KIDS-04, KIDS-05, KIDS-06, KIDS-08, KIDS-09, KIDS-10, KIDS-11, KIDS-12, KIDS-13, KIDS-14, KIDS-15, KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (119)

### `HW-08-0023` - Every duration is labelled in minutes and computed in seconds, so a limit the parent sets as 40 minutes locks the app after 40 seconds.

- Category B, severity blocker, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The sample's whole purpose is limiting how long a child uses the app, and it is wrong by a factor of sixty in the direction that makes it useless: 'Duration of use: 40 minutes' locks the screen after forty seconds, and the '20 minute' rest that follows is over in twenty. A parent configuring this trusts the label, gets a lock roughly a minute into use, and concludes the app is broken rather than that the unit is. Nothing anywhere converts minutes to seconds -- there is no multiplication by 60 in the project -- so both halves of the cycle are equally short.
- Fix: Convert once, where the picker value is read: 'this.useTime = this.checkedList[value + ''].valueOf() * 60;' and the same for restTime, so everything downstream stays in the seconds the clock uses. Alternatively hold the durations in milliseconds throughout and drop the '* 1000' at the two consumers.

### `HW-08-0001` - The expression generator loops forever on the UI thread if the secure random number fails, because the failure value -1 produces an expression whose result can never be positive.

- Category B, severity high, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The sample handles a cryptoFramework failure by returning a sentinel that the caller cannot tell apart from a legitimate value, and the sentinel then drives an unbounded loop on the UI thread. The result is not a wrong answer but a frozen application: the entry page never finishes aboutToAppear, so nothing renders and the app has to be killed. The loop is unbounded even on the success path -- it regenerates until the answer happens to be positive -- so there is no iteration cap to fall back on.
- Fix: Have doRandBySync signal failure distinctly (throw, or return undefined) and let checkAbs propagate it instead of feeding it back into the generator. Bound the retry loop with an attempt count and fall back to a fixed expression when it is exhausted, and move the generation off aboutToAppear so a slow or failing call cannot block first paint.

### `HW-08-0024` - Every preference is read with a string default although all stored values are numbers, and the reference says a type mismatch returns the default, so nothing saved is ever read back.

- Category B, severity high, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: A number was stored and a string default is supplied, so by the documented rule the read returns 'No info' even when the value is present. The three fields typed number therefore hold a string, and 'as number' is a compile-time assertion that converts nothing, so the mismatch never surfaces as an error. MainPage.aboutToAppear then computes 'this.futureRest - this.currentTime', which is NaN, and both comparisons against NaN are false, so the lock is not restored. For an anti-addiction control that is the failure that matters: the limit survives inside a session and is erased by closing and reopening the app, which is the first thing a child locked out of it will try. The same mismatch makes the first-open branch in EntryAbility permanently take the else arm, rewriting the flag on every launch.
- Fix: Give the read a default of the stored type and no cast -- 'store.getSync(storeKey, 0) as number' -- or better, add typed helpers (getNumber with a numeric default, getString with a string one) so the default and the declared return type cannot drift apart. Signal absence with a sentinel the caller can test, such as 0 or -1, rather than a string.

### `HW-08-0025` - The table converting a picked duration to a number is wrong in two of its six entries, so choosing 20 stores 10 and choosing 80 stores 90.

- Category B, severity high, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The table exists only to turn the picked label into the number behind it, so its entries should be identities; two are not, and the two wrong ones are at the ends of the range where a parent is most likely to be deliberate. Picking the shortest use period, 20, silently halves it to 10; picking the longest, 80, quietly grants 90. Because the displayed text is taken from the picker and not from the stored value, the settings screen goes on reporting the number the parent chose, so the discrepancy is invisible from inside the app. A Record whose every entry maps a numeric string to the same number is also redundant: parsing the string would be correct by construction and could not drift.
- Fix: Delete checkedList and parse the picked value -- 'this.useTime = Number.parseInt(value + '', 10);' -- so the stored number is the displayed one by construction.

### `HW-08-0026` - The lock lifts only on an exact tick equality, and the reference states that tick does not fire while the app is in the background or the screen is locked.

- Category B, severity high, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The rest period is the interval during which a child is being told to put the device down, so the app being backgrounded or the screen being locked is the expected behaviour, not an edge case -- and the reference says the tick does not fire in either state. Miss the single tick where elapsedTime equals restReal and isClick is never set, the 'to set up' text stays disabled, and the overlay cannot be dismissed for the rest of the process. The comparison is also exact rather than a threshold, so any dropped or coalesced tick has the same effect. The recovery a user would try, force-quitting and reopening, is the one that removes the lock entirely.
- Fix: Compare with '>=' rather than '===' so a missed tick still releases the lock, and do not rely on the tick at all: recompute the remaining rest from the stored end time in aboutToAppear and in the ability's onForeground, and clear isShow there when the period has passed.

### `HW-08-0035` - The board is painted once from onReady using dimensions set by an unordered async callback, and nothing repaints when those dimensions arrive.

- Category C, severity high, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: Two independent asynchronous events have to land in one order and nothing enforces it. If the display callback resolves after onReady, drawCheckerboard runs with sideLen, spaceLen and boardLen still 0: strokeRect draws a zero-sized rectangle, all thirty-eight lines collapse onto the same point, and the child is left looking at a plain yellow square. When the real dimensions arrive a moment later nothing redraws, because canvas painting is imperative -- a @State change re-runs build, and build issues no drawing commands. The @State decorators are what makes this look safe: they suggest the geometry drives the rendering, and it does not. The same values are also what the tap handler divides by, so in that state every tap computes Infinity and is silently rejected by the bounds check, leaving a board that cannot be drawn on either.
- Fix: Take the size from the canvas rather than from a separate async source: compute the dimensions in onReady from the context's own width and height, or cache the component size in onAreaChange and draw from there. If an async source must be kept, redraw when it lands -- give sideLen a @Watch that calls drawCheckerboard once the canvas is ready.

### `HW-08-0043` - Every puzzle piece nests a Path inside a Canvas, although the Canvas reference states it accepts no child components, and the Canvas itself draws nothing.

- Category C, severity high, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: Canvas is the imperative drawing surface: it exists to be handed a rendering context and painted from onReady, and its reference says it holds no children. Path is the declarative alternative and needs no host -- it renders itself from its commands, stroke and fill attributes, which is exactly what is happening here. So every Canvas in this sample is an empty drawing surface wrapped around a component that draws itself, in a documented arrangement the component does not support. The cost is paid seven times over per grid and the page renders three grids of seven pieces, so twenty-one drawing surfaces are allocated and left unused. The Canvas is also what carries the 20 by 100 size that clips the pieces' touch targets.
- Fix: Delete the Canvas wrapper and return the Path directly from each builder, moving the width and height onto the Path. If a container is wanted for layout, use Stack or Column, which do accept children.

### `HW-08-0044` - A piece's draggable area is 20 by 100 while the shape drawn inside it is up to 300 by 300, so most of a visible piece cannot be grabbed.

- Category C, severity high, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: Hit testing follows the component's laid-out box, not the pixels a child component happens to paint outside it, so the region that responds to a drag is the 20 by 100 GridItem while the triangle the child sees is fifteen times its area. Dragging works only from a narrow vertical strip at the piece's left edge; a touch anywhere else on the visible shape either does nothing or grabs whichever piece's strip is underneath, which in a seven-piece tangram is frequently the wrong one. This is a puzzle whose entire interaction is picking a shape up, so the mismatch is the interaction failing rather than a cosmetic overflow. The same sizes also make the intended sizing intent unreadable: nothing in the layout expresses how large a piece is meant to be.
- Fix: Size each container to the path it holds -- a 150-unit triangle needs a 150 by 150 box -- so the layout box, the visible shape and the touch target coincide. Deriving both the path commands and the width/height from one per-piece dimension keeps them from drifting apart again.

### `HW-08-0045` - The vertical scale factor is computed as the square of the height ratio, so every piece is positioned wrongly on any screen that is not the design size.

- Category B, severity high, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: A scale factor converts design-sheet units to device units and must be linear; squaring it means the error grows with the distance from the design size. On a screen ten per cent taller than the 809.1428 reference every y coordinate is multiplied by 1.21 instead of 1.1, so a piece whose target sits 500 units down is placed about fifty units too low, while the horizontal coordinate beside it is scaled correctly. The three stacked grids -- the draggable pieces, the outline of their start positions and the target silhouette -- all use the same factor, so the whole board stretches vertically together and the puzzle still looks coherent; what breaks is the snap test, which compares unscaled coordinates against unscaled targets while the child is aiming at positions that have been moved. The width line immediately above shows the correct form.
- Fix: Compute the height factor the same way as the width factor: 'this.scaleHeight = this.screenHeight / standardHeight;'.

### `HW-08-0046` - Drag deltas measured on screen are added to design-sheet coordinates that are then scaled again at render time, so a piece does not follow the finger.

- Category B, severity high, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: info.offsetX is the distance the finger actually travelled on the device, while offsetXItem holds design-sheet units that are multiplied by scaleWidth before they reach position(). Adding one to the other mixes the two spaces, so the piece moves scaleWidth times as far as the finger: on a wider screen it runs ahead of the touch and on a narrower one it lags behind, and with the squared vertical factor the two axes drift by different amounts. The hardcoded bounds on the line above are in the same mixed space, so the drag region does not correspond to the visible board either. The design was to convert at the boundary -- the document shows exactly that, with scaling helpers on both axes -- and the shipped code assigns the raw values instead.
- Fix: Keep one coordinate space. Either convert the gesture delta into design units before adding it (divide by scaleWidth and scaleHeight), or store positions in device units and scale only the constant targets when they are loaded.

### `HW-08-0053` - Every touch-move re-strokes the whole accumulated path, so one drawn line costs work quadratic in its length and its colour builds up where it has been painted most.

- Category C, severity high, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: stroke(path) paints the whole path it is given, not the part added since the last call, so a stroke of n move events issues 1 + 2 + ... + n segment draws instead of n. A child drawing a single unhurried line across the board generates hundreds of move events, which is tens of thousands of segment draws for one line, all on the UI thread inside the touch handler -- the frame budget this has to fit into is what determines whether the ink keeps up with the finger. The visual consequence is separate and worse: an anti-aliased stroke composites partially transparent pixels at its edges, so re-painting the same segment n times accumulates coverage, and the beginning of a line ends up darker and visibly thicker than its end. The same file already shows the fix for the eraser branch, which draws only at the current point.
- Fix: Stroke only the new segment: keep the previous point, and on each move do beginPath(), moveTo(prevX, prevY), lineTo(x, y), stroke() -- then update the previous point. Keep the Path2D only if the completed stroke has to be replayed later.

### `HW-08-0068` - The safe-area machinery is present but gutted: the change listener has empty branches, the avoid areas are never read, the two bound fields are never used, and the inset is a hardcoded 24.

- Category B, severity high, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: Four pieces of a working mechanism are present and the connections between them have been removed, so nothing reports that it is broken. The window is laid out full screen, which makes padding the application's responsibility; the value it pads with is 24, which is not the status bar height on any particular device, so the title bar sits too high or too low on all of them and under the status bar on any device whose inset exceeds 24. The listener is the worst part: it is registered for the life of the process, is never released -- the string off does not appear in the file -- and its two branches do nothing at all, so it costs a callback on every layout change to accomplish nothing. The two @StorageProp fields complete the illusion by declaring that the page consumes insets it never reads.
- Fix: Restore the chain from the sibling sample: read both avoid areas in onWindowStageCreate, convert with px2vp, write them into AppStorage, fill in the listener body to keep them current, release it in onWindowStageDestroy, and replace padding({top:24}) with the bound topRectHeight. If the insets are genuinely not wanted, delete the listener and both @StorageProp fields.

### `HW-08-0076` - The random move index divides by 255 instead of 256, so a maximum byte selects one past the end of the array and the undefined it returns passes the null guard into a crash.

- Category B, severity high, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: Each iteration has a one-in-256 chance of drawing the maximum byte, and the shuffle runs a hundred of them, so roughly a third of games hit it -- 1 - (255/256)^100 is about 32 per cent. When it fires, blocks[undefined] is undefined and reading .x throws a TypeError out of an async arrow function that nobody awaits, so it surfaces as an unhandled rejection rather than an error the app can report. The two lines after the loop never run, Start.ets:141-142, 'this.blocks = currentBlocks;' and 'this.isShuffle = !this.isShuffle;', so this.blocks keeps the ordered array that initBlocksArray built: the child is shown a puzzle that is already solved. A sliding-tile game that starts finished one time in three is the whole feature failing, and the cause -- a 255 where 256 belongs -- is a single character.
- Fix: Divide by the number of byte values, not the largest one: 'let index = Math.floor((randomByte / 256) * possibleMoves.length);' or, avoiding floating point entirely, 'let index = randomByte % possibleMoves.length;'. Reject bytes in the biased tail if uniformity matters. Separately, make the guard reject undefined as well as null, and await the shuffle so a failure is not silent.

### `HW-08-0083` - When the allowance is used up the ring gauge is driven to a full 100, so the one indicator on the screen reads full at the moment nothing is left.

- Category B, severity high, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: The ring is a remaining-quota gauge: it empties as the child uses data, and reaching zero is the event the whole feature exists to show. The negative arm makes it do the opposite, snapping the ring from nearly empty to completely full at the instant the allowance runs out, so the strongest visual signal on the screen inverts precisely when it matters. A parent glancing at a full ring reads plenty of data left. The number in the centre contradicts it -- it is negative by then -- and the colour changes, but a filled ring beside a red colour is ambiguous rather than alarming, and nothing marks the ring as an overflow state. Writing 0 instead of 100 would make the gauge consistent with both the number and the colour.
- Fix: Drive the gauge with the clamped remaining fraction: 'value: Math.max(0, Math.min(100, this.tempData * 100 / (this.dataNumber * Constants.GB_TO_MB)))'. If an over-quota state should look distinct, show a full ring in the used colour only with an explicit over-limit label.

### `HW-08-0091` - The map module declares no permissions at all, so the network permissions Map Kit needs to load tiles are absent and the map this sample exists to show cannot render.

- Category D, severity high, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: Map tiles, the trace overlay and the reverse-geocoded address are all fetched over the network, and none of them can be without ohos.permission.INTERNET. Both are system_grant permissions, so declaring them costs nothing at runtime -- there is no dialog and no user decision -- and omitting them produces the exact failure the troubleshooting page opens with: a blank map with a permission-denied entry in the log and no error surfaced in the app. Every other failure mode in this sample is invisible on top of that one, because the page has no content besides the map.
- Fix: Add a requestPermissions block to module.json5 declaring ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO with usedScene.when set to always, as the Map Kit FAQ specifies.

### `HW-08-0092` - A one-second interval started in aboutToAppear is never cleared and has no stop condition, so reverse-geocoding calls continue for the life of the process.

- Category B, severity high, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: The timer is the sample's only moving part and nothing ever stops it. Two separate defects compound: the end of the simulated journey clamps rather than terminating, so the loop keeps issuing one reverse-geocoding request per second for a coordinate that will never change again; and there is no teardown hook, so the interval survives the page being destroyed and keeps a closure over the component alive with it. The result is a permanent once-a-second network call for a screen the user has left -- on a child's device, which is exactly where battery and data matter, in an app whose sibling feature monitors that data usage. The timer id is already stored in a field, so the fix has nothing left to work out.
- Fix: Call clearInterval(this.timer) from an aboutToDisappear hook, and stop the timer when the track ends rather than clamping: 'if (this.index >= MAP_LOCATION.length) { clearInterval(this.timer); return; }'.

### `HW-08-0097` - Raw file descriptors are opened for every pronunciation tap and never closed, although the reference states explicitly that they must be.

- Category B, severity high, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: The two pronunciation buttons are the feature: a child taps them repeatedly, once for the Chinese reading and once for the English, on every card, and each tap opens a descriptor that is never returned. File descriptors are a per-process limit, so the leak does not degrade gracefully -- it accumulates silently through a session of ordinary use until the process cannot open another, at which point getRawFdSync throws and audio stops working with no indication of why. The reference states the obligation in the same paragraph as the signature, and the sample has four call sites and no close on any path, including its release() method.
- Fix: Close each descriptor once the player no longer needs it: keep the RawFileDescriptor alongside the player and call context.resourceManager.closeRawFdSync(url) when the source is replaced, when playback completes, and in release().

### `HW-08-0104` - One field holds the distance from home in metres in two places and in hundreds of metres in a third, and the alarm message prints all of them with an m suffix.

- Category B, severity high, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: The same field means two things a hundred-fold apart depending on which code path last wrote it. getDistance divides by a hundred, so after it runs a child two kilometres from home is recorded as 20 and compared against a threshold expressed in metres -- the alarm is silently disabled, since 20 is below any plausible safe radius. The two haversine paths write metres, so whether the alarm fires depends on whether the last update came from the continuous callback or from the one-shot lookup. This is a child-safety alarm whose entire behaviour is that comparison, and the message shown to the parent labels the number m in every case, so the wrong readings are indistinguishable from the right ones. The helper's own doc comment claiming kilometres while the body returns metres is what makes the unit unrecoverable from reading the code.
- Fix: Fix the unit at the one site that is wrong -- 'this.distanceToHome = Math.floor(map.calculateDistance(location, homeLocation));' -- correct the haversine doc comment to metres, and give the field a name that states its unit, such as distanceToHomeMeters.

### `HW-08-0105` - The network permission the document itself lists as required is the one permission the module does not declare, in a sample built on Map Kit.

- Category D, severity high, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: The sample gets four of the five permissions its own document specifies and misses the one that lets it reach the network, which is what the map tiles, the coordinate conversion and the address lookup all need. Because GET_NETWORK_INFO is present, the app can still observe that a network exists -- and it does, showing a toast when there is none -- so it reports the network as available while being unable to use it, which is the most confusing of the failure modes. INTERNET is system_grant, so declaring it costs no dialog and no user decision; the omission is pure oversight and is contradicted by the document shipped beside the code.
- Fix: Add ohos.permission.INTERNET to requestPermissions with usedScene.when set to always, alongside the GET_NETWORK_INFO entry that is already there.

### `HW-08-0110` - A class decorated @ObservedV2 with @Trace properties is held in @State and @Link, which the guide states is a compile-time error.

- Category C, severity high, confidence confirmed
- Features: KIDS-15
- Document: `huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
- Why: The two state management systems are alternatives, not layers: V2 observes property writes through @ObservedV2 and @Trace, V1 observes assignment through @State and @Link, and the guide forbids one class being subject to both. Here the grid items are V2 objects living in V1 containers, so the observation path the model was written for -- writing item.ifOnclick in a tap handler and having that one tile repaint -- is not the path the containing components use, and the framework rejects the combination. The tap handler at GridFocusCom.ets:58 does exactly that write, 'item.ifOnclick = false;', so the feature the V2 decorators exist to provide is the one the pairing breaks.
- Fix: Pick one system. Either convert the two components to @ComponentV2 and replace @State/@Link with @Local/@Param, keeping @ObservedV2 and @Trace; or drop to V1 throughout, decorating the class @Observed and binding the item in the child with @ObjectLink.

### `HW-08-0111` - The shuffle divides the random byte by 255, so a maximum byte selects one index past the range, and the guard beside it admits exactly that value.

- Category B, severity high, confidence confirmed
- Features: KIDS-15
- Document: `huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
- Why: Each iteration has a one-in-256 chance of drawing the maximum byte, and a five-by-five grid runs twenty-four of them, so about nine per cent of games are affected. When it fires on the first iteration a tile is created with undefined as its number, renders the text undefined, and can never match this.searchNumber -- so the game becomes unwinnable, which for a timed concentration exercise means the child works through the grid and the timer never stops. Later iterations do not corrupt the array but still select outside the correct range, biasing the shuffle away from uniform. The guard is the part that makes it reachable: comparing j to numbers.length instead of to i is precisely the bound that lets the invalid index through.
- Fix: Divide by the number of byte values rather than the largest one and bound the index correctly: 'const j = randOutput.data[0] % (i + 1);' -- which cannot exceed i -- and drop the guard, or make it 'if (j <= i)'. Draw all the bytes in one generateRandomSync call before the loop instead of creating a generator per iteration.

### `HW-08-0115` - The die is a random byte modulo six, which is not a fair die: four of the six faces come up about 2.4 per cent more often than the other two.

- Category B, severity high, confidence confirmed
- Features: KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/16_die_rolling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
- Why: Fairness is the entire specification of a die, and this one is measurably not fair. Reducing a uniform value from a range that is not a multiple of six concentrates the remainder on the low faces: 1 through 4 come up with probability 43/256 and 5 and 6 with 42/256, a 2.4 per cent excess that is stable and reproducible rather than noise. Choosing cryptoFramework over Math.random shows the author cared about the quality of the source and then discarded it at the last step, which is the more instructive part -- a secure generator does not make a biased reduction unbiased. The document repeats the method, so the bias propagates to every reader who copies the snippet rather than the reasoning.
- Fix: Reject the biased tail before reducing: draw a byte, discard it if it is 252 or above, and take the remainder otherwise -- 'let b: number; do { b = rand.generateRandomSync(1).data[0]; } while (b >= 252); let num = b % 6;'. The loop terminates with probability 1 and rejects fewer than 2 per cent of draws.

### `HW-08-0002` - The secure random helper asks for 24 bytes and derives every value from the first one, because toString on a Uint8Array yields a comma-separated list that parseInt stops reading at the first comma.

- Category B, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: Twenty-three of the twenty-four requested bytes are discarded, so every digit and every operator in the verification expression comes from a single byte. That byte is a value in 0-255, and 256 is not a multiple of 10, so "% 10" makes the digits 0 to 5 more likely than 6 to 9 -- the modulo bias the byte count was presumably meant to avoid. The comment carried over from the reference sample says a 24-byte random number is being generated, which is not what the code consumes. The null check is also the wrong one: generateRandomSync returns a DataBlob, so comparing it to null does not detect an empty result, which is the case the guide checks for.
- Fix: Read a single byte and reject values in the biased tail before taking the modulus, or request one byte at a time and retry while the value is at or above the largest multiple of num below 256. Check randData.data.length !== 0 as the guide does, and drop the 24-byte comment.

### `HW-08-0003` - Rect is imported from @ohos.application.AccessibilityExtensionAbility instead of @kit.AccessibilityKit, pulling an accessibility-framework type into an app that has no accessibility extension.

- Category A, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The Rect this imports describes the screen bounds of an accessibility element, and it is being used to name a status bar and navigation bar height in an app that declares no AccessibilityExtensionAbility. The window module already publishes its own Rect for exactly this purpose, and the heights the app actually uses are plain numbers read from getWindowAvoidArea and held in AppStorage as topRectHeight and bottomRectHeight. So the import brings in an unrelated system module, in the superseded @ohos path form, for two fields that never hold a value.
- Fix: Delete both imports and both fields. The avoid-area heights are already carried as numbers through AppStorage; nothing in this sample needs a Rect. If a rectangle type is genuinely wanted, take window.Rect from @kit.ArkUI.

### `HW-08-0004` - The settings page reads its navigation parameter under the route name Index while the route pushed is named Setting, and the parameter it would receive is built from two fields that are never assigned.

- Category B, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: No route named Index exists, so getParamByName returns an empty array and the cast to Array<Param> hides that. The parameter is therefore never delivered, and the field that would hold it, params, is never read in Setting's build, so the mismatch produces no visible symptom and will survive until someone tries to use it. The push side is equally empty: both members of the Param are undefined for the lifetime of the page, so even with the name corrected nothing would arrive. Setting does not need them -- it reads the avoid-area heights from AppStorage at lines 24-25, which is where they actually live.
- Fix: Delete the params field and the Param argument to pushPathByName; Setting already gets its avoid-area heights from AppStorage. If a parameter is wanted later, read it under the same name the route was pushed with.

### `HW-08-0005` - The carousel of animation cards wraps each slide in a ListItem, which the reference states may only be a child of List or ListItemGroup.

- Category C, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: ListItem is not a generic wrapper; it is a List child that exists to carry list-specific behaviour such as swipe actions, selection and the group layout, and it is laid out by its List parent. Placed under Swiper it contributes nothing but an extra node in the tree for every slide, and it puts the component outside the parent relationship its own documentation requires, so its layout behaviour under Swiper is not defined by either component's contract. The slide content is a Column and needs no wrapper at all.
- Fix: Remove the ListItem and let the Column be the direct child of the ForEach inside Swiper.

### `HW-08-0006` - Each reduction pass evaluates the expression twice, and the string it returns describes a different list from the one it returns beside it.

- Category B, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The pair (list, data) is supposed to describe one expression, and it does not: data is produced from a list one reduction ahead of the list being returned, and is the empty string whenever the extra call finds nothing left to reduce. getRandomListAndResult at Utils.ets:197-199 does read that string as its result -- "listData = CalculateUtil.checkHaveAddAndSubAndExe(list);
  list = listData[0];
  let resultData = listData[1];" -- and only escapes returning it because the loop immediately below overwrites resultData from the list. The verification answer therefore depends on that overwrite always firing. Meanwhile every reduction step runs twice and decrements the same id fields twice, inside a loop that the entry page blocks on.
- Fix: Call the reduction once and destructure the pair: "const step = CalculateUtil.mulAndDiv(list); list = step[0]; data = step[1];", and the same for addAndSub. Build the new list without mutating the caller's Calculator objects.

### `HW-08-0007` - The UIContext is passed down as @Provide typed UIContext | undefined and received as @Consume typed UIContext, which the guide says causes implicit conversion and application exceptions.

- Category C, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The provider genuinely can hold undefined -- Profile.ets:78 assigns it only in aboutToAppear -- so the consumer's declared type is a claim the provider does not honour, and the non-null assertions in Setting suppress the check that would have caught it. The pairing is also unnecessary: any custom component can call this.getUIContext() for itself, which is exactly what Profile.ets:78 does to obtain the value it then forwards.
- Fix: Delete the @Provide/@Consume pair and call this.getUIContext() in Setting, as Profile already does. If the value must be shared, declare both sides with the same type and drop the non-null assertions.

### `HW-08-0008` - The avoidAreaChange listener registered when the window stage is created is never unregistered, and the call is not guarded although the API can throw.

- Category B, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The listener is registered against the main window and holds a closure that writes into AppStorage; the ability's own comment at onWindowStageDestroy says UI resources are to be released there, and this one is not. The registration is also the only window call in the file left unguarded -- setWindowLayoutFullScreen above it has a catch, and the reference wraps both on and off in try/catch for the documented 401 -- so a parameter failure here propagates out of onWindowStageCreate and the content is never loaded.
- Fix: Keep the callback in a field, call windowClass.off('avoidAreaChange', callback) from onWindowStageDestroy, and wrap the registration in try/catch as the reference example does.

### `HW-08-0013` - The tab index is bound with $$ to an undecorated member, so the selected-tab highlight never moves off the tab the app starts on.

- Category C, severity medium, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: $$ keeps the plain variable and the component's internal state in sync, but only a decorated variable propagates that change into the UI. tabindex is undecorated, so when the user switches tabs the field is updated and nothing re-renders: the blue highlight stays on the tab whose index was the initial value, 1, which is 我的. The colour is the only cue distinguishing the selected tab from the other one, so the tab bar reports the wrong tab as active for the rest of the session -- and the highlight is precisely what the two-way binding was added to drive.
- Fix: Decorate the field, "@State tabindex: number = 1;", leaving the $$ binding as it is.

### `HW-08-0014` - The handwriting canvas is created without RenderingContextSettings, so anti-aliasing is off for every freehand stroke the app exists to draw.

- Category C, severity medium, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: Anti-aliasing is exactly what this sample needs and the one setting it does not pass. The product is a calligraphy practice board: its entire visible output is diagonal and curved freehand strokes 20 units wide, drawn segment by segment from touch coordinates, and those are the strokes that show stair-stepping most when aliased. The saved result is then shown back to the child on the completion page as a record of their handwriting, so the jagged edges are preserved into the reward screen. The guide grid drawn underneath at lineWidth 0.5 is affected worse still, since sub-pixel diagonal hairlines without anti-aliasing break into dotted lines.
- Fix: Construct the context with anti-aliasing enabled: "private settings: RenderingContextSettings = new RenderingContextSettings(true);" and "private canvasContext: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);".

### `HW-08-0015` - savePicture opens a file and closes it only on the success path, so any failure between the two leaks the descriptor and throws out of the click handler.

- Category B, severity medium, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: openSync has already acquired a descriptor by the time the decode and the write run. If buffer.from rejects its input or writeSync fails -- a full sandbox is the ordinary case, since the function writes a new file on every one of the five steps and never deletes any -- closeSync is skipped and the descriptor is held for the life of the process. The same throw then escapes savePicture and the onClick that called it, so the failure is silent to the user: the page does not advance, no message appears, and the child taps the arrow again, opening another descriptor each time. Descriptors are a per-process limit, so repeated failures exhaust them rather than degrading gracefully.
- Fix: Close in a finally block so the descriptor is released on every path, check the result of the base64 split before decoding, and catch the BusinessError so a failed save surfaces to the user instead of aborting the handler.

### `HW-08-0017` - The writing board always draws from touches[0] without the emptiness check the reference requires and without distinguishing fingers by id, so a resting hand redirects the stroke.

- Category C, severity medium, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: This is a writing board for children, so a hand resting on the screen while the other hand writes is the normal way it will be held, and every one of those contacts becomes an element of touches. The handler tracks no finger identity, so whichever contact happens to be at index 0 supplies the coordinates: lineTo jumps to the resting palm and back, drawing a stroke across the character the child is tracing. The same omission makes the two contacts indistinguishable on lift, so the path is closed by whichever finger rises first. The emptiness check the reference asks for is the second half: touch is read before the event type is known, so any event arriving with an empty touches array leaves touch undefined and the Down and Move branches dereference it.
- Fix: Check event.touches.length before use, take the coordinates from changedTouches for the event that actually fired, and record the id of the finger seen on Down so Move and Up only act on that same id. Handle TouchType.Cancel alongside Up so an interrupted touch closes its path.

### `HW-08-0018` - The ability keeps a permanent avoidAreaChange subscription feeding two AppStorage keys that no component in the app ever reads.

- Category B, severity medium, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: A window listener that outlives the window is a leak on its own, and here it is a leak with no purpose: it exists to keep two values current that nothing consumes, so the whole block -- two getWindowAvoidArea calls, four AppStorage writes and a permanent subscription -- is dead. The document labels this file 屏幕窗口沉浸式布局页 (the immersive window layout page), which is the capability the code appears to implement and does not: the ability does call setWindowLayoutFullScreen(true) and hides the navigation indicator, so the pages really are drawn under the system bars, and neither page pads for them. The measurements that would have fixed that are computed and discarded.
- Fix: Either consume the keys -- bind topRectHeight and bottomRectHeight with @StorageProp in WriteBoard and Finish and apply them as padding -- or delete the block entirely. Whichever is kept, hold the callback in a field and call windowClass.off('avoidAreaChange', callback) from onWindowStageDestroy.

### `HW-08-0019` - The completion screen is registered as a launchable page and decorated @Entry while being built as a navigation destination, and launching it as a page would leave its @Consume without a provider.

- Category C, severity medium, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: The two roles contradict each other. As a navigation destination Finish is correct: it is built inside WriteBoard, so the @Provide is above it and its NavDestination receives the pushed parameter through onReady. As a page it is not: a page is the root of its own component tree, nothing above it provides pageInfos, and the @Consume has no match. Registering it in main_pages.json declares that launching it that way is supported, and the @Entry decorator is what makes it launchable, so the sample ships a page that cannot work. The pairing also misleads a reader about the pattern being demonstrated, which is precisely that a destination is not a page.
- Fix: Remove @Entry from Finish and drop pages/Finish from main_pages.json, leaving WriteBoard as the only page and Finish as a destination built by the navDestination builder.

### `HW-08-0027` - Leaving the settings page rewrites the lock deadline in memory without persisting it, and does so even when the user changed nothing.

- Category B, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: Two copies of the deadline now disagree. The in-memory @Provide is pushed forward every time the page is closed, while the persisted copy is only updated by the save button, so the value the app runs on and the value it would restore after a restart diverge. The callback also fires on any exit, including the back gesture after the parent looked at the screen and changed nothing, and including the entry from the lock overlay -- so simply opening and closing settings extends the child's remaining time by a fresh use period. That is the opposite of what the screen is for, and it bypasses the enabled() guard the same computation is deliberately placed behind on the button.
- Fix: Delete the onVisibleAreaChange handler; the save button already performs this computation and persists it. If a deadline must be recomputed on exit, guard it on the same setUse and setRest flags and persist it through savePreferenceInfo like the button does.

### `HW-08-0028` - The ability selects a start page through a first-launch branch and then loads a hardcoded one, and the variable it discards names a page that does not exist.

- Category B, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The block reads as a first-launch routing decision -- send a returning user to the main page, send a first-time user somewhere else -- and none of it takes effect, because loadContent is passed a literal. What it does still do is write the first-open flag on every launch, since the condition can never be true. The discarded default points at pages/RestPage, which does not exist anywhere in the sample, so the intent cannot even be recovered from the code: a reader cannot tell whether a rest page was removed or never written. If the branch were wired up as written, a first launch would try to load a missing page.
- Fix: Either delete the block and load pages/MainPage directly, or pass the computed value -- 'windowStage.loadContent(loadPageUrl, ...)' -- and give the first-launch arm a page that exists.

### `HW-08-0029` - The clock is seconds since midnight, so any use or rest period that runs past midnight ends immediately or never.

- Category B, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The two quantities being subtracted are times of day, so their difference is only a duration while both fall on the same day. A period started before midnight produces a futureRest above 86399, and the next reading restarts near zero, so the difference becomes a value close to a full day: the app schedules a setTimeout up to twenty-four hours out and treats the child as having almost a day of use remaining. Reopening after midnight during what should be a rest gives the mirror image, a negative difference that fails both tests, so the lock is silently dropped. Late-evening use is the case a screen-time limit exists for, which is exactly the window where the arithmetic stops meaning anything.
- Fix: Store an absolute instant rather than a time of day -- Date.now() in milliseconds -- and compare instants. That removes the midnight discontinuity and makes the stored deadline meaningful across launches.

### `HW-08-0030` - Preferences are written by an async function that is never awaited, and the save button pops the page before calling it.

- Category B, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: flush is what puts the value on disk, and nothing waits for it or learns whether it failed: a rejected promise from put or flush is unhandled, so a save that does not happen is indistinguishable from one that does. The ordering at the save button makes it worse by starting the write after the component has been popped, so the one persisted value that matters -- the lock deadline -- is written during teardown, from a handler whose component is going away. Since this is the state that is supposed to survive the app being closed, a silently dropped flush removes the limit entirely.
- Fix: Make the handler async and await the save before popping, and catch BusinessError so a failed write is reported rather than discarded: 'await PreferencesClass.savePreferenceInfo(...); this.pageInfos.pop();'.

### `HW-08-0031` - Context is imported from the access control module rather than from Ability Kit.

- Category A, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: abilityAccessCtrl is the permission-checking module; this sample requests no permissions and calls nothing in it. Reaching into it for Context ties the app's storage layer to an unrelated module through a superseded @ohos path, in a file whose only dependency should be ArkData. The correct source is @kit.AbilityKit, which is already imported by the ability in the same project, and which the access-control reference itself uses when it needs the type.
- Fix: Import the type from the kit that owns it: 'import { common } from '@kit.AbilityKit';' and type the parameter as common.Context or common.UIAbilityContext.

### `HW-08-0032` - The avoidAreaChange listener registered when the window stage is created is never released.

- Category B, severity medium, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: The callback is registered against the main window and holds a closure that writes into AppStorage; the ability's own comment on the destroy hook says UI resources are released there, and this one is not. Unlike the neighbouring samples the two keys are genuinely consumed here -- MainPage and SetTimePage both bind them with @StorageLink and apply them as padding -- so the subscription has a purpose and only its release is missing. The registration is also the one window call in the file left unguarded, while setWindowLayoutFullScreen above it has a catch.
- Fix: Keep the callback in a field, call windowClass.off('avoidAreaChange', callback) from onWindowStageDestroy, and wrap the registration in try/catch as the reference example does.

### `HW-08-0036` - The board is sized from the physical display rather than from the canvas it is drawn on, so it overflows any window smaller than the screen.

- Category C, severity medium, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: The canvas is a component inside a window, and the window is not the screen. On 2in1 and tablet the app can be resized or run side by side, on a foldable the window changes with the hinge, and in split-screen it is a fraction of the display -- in every one of those the board is computed for the full screen and painted into a smaller square, so the right and bottom edges of a 19-line grid fall outside the visible area. The tap handler then maps touches through the same oversized spaceLen, so the coordinates it computes do not correspond to the intersections the child can see. The component's own size is available and is what aspectRatio(1) has already resolved; reading the display instead discards it. The full-screen layout compounds it, since the top of the board sits under the status bar with only the 20 vp borderMargin between them.
- Fix: Derive sideLen from the canvas: read the context's width and height inside onReady, or cache the component size with onAreaChange as the sibling handwriting sample does, and redraw when it changes. Pad for the avoid area, or drop the full-screen layout, so the first row of the grid is not under the status bar.

### `HW-08-0037` - The display callback declares an error parameter, never reads it, and dereferences the data array on the next line.

- Category B, severity medium, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: This is an AsyncCallback, so failure arrives as a populated err with data left unusable, and the sample reads data[0].width regardless. A throw inside the callback is not caught by anything -- there is no try/catch and the callback is not a promise -- so it surfaces as an unhandled error while the page has already been laid out, leaving the three geometry fields at zero and the board blank with no indication of why. The reference example on the very page this API is documented on shows the guard, and the same file's own hilog import is unused, so nothing is logged either.
- Fix: Follow the reference: test err.code and return before touching data, check data.length, and log the failure. Better still, take the geometry from the canvas and remove the dependency on this call entirely.

### `HW-08-0038` - The grid is stroked without ever beginning a path, so each line re-strokes every line drawn before it and a 19-line board costs 741 segment strokes instead of 38.

- Category C, severity medium, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: stroke draws the entire current path, and moveTo/lineTo append to it rather than replacing it, so without beginPath the path only ever grows. The last stroke of the loop redraws all thirty-eight lines, the one before it thirty-seven, and so on: the work is quadratic in the board size for a drawing that is linear. Every segment is also painted up to thirty-eight times, which on a translucent or anti-aliased stroke compounds the coverage at each pixel and thickens the lines unevenly across the board -- the first line drawn receives the most passes. The same file demonstrates the correct discipline eleven lines further down, where each stone begins its own path.
- Fix: Call this.context.beginPath() at the top of each iteration, or build the whole grid as one path and stroke it once after the loop, which is the cheaper of the two.

### `HW-08-0039` - The font-size search returns two steps below the size it rejected, so every stone number is drawn a step smaller than it should be, and on a small board the value goes negative.

- Category B, severity medium, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: The search is correct up to the final step and then discards a size that was measured to fit, so the move number inside every stone is drawn one increment smaller than the stone can hold -- on a board where the stone is about sixteen units across that is a visible fraction of the glyph. The subtraction is unguarded, so when the first size tried already overflows -- a small window, a folded screen, or the zero-geometry state this sample can start in -- the function returns -1 and the canvas is handed '-1vp' as a font size. The loop has no iteration cap either, so it relies entirely on the measured height eventually exceeding boardLen; with boardLen at zero or negative that condition is met on the first pass, which is the same path that produces the negative result.
- Fix: Return fontSize - 1, clamped to a minimum of 1, and give the loop an upper bound so a degenerate boardLen cannot drive it: 'for (let fontSize = 1; fontSize <= MAX_FONT_SIZE; fontSize++) { ... } return 1;'.

### `HW-08-0047` - The document's drag handler calls two scaling helpers that exist nowhere in the sample, hiding the coordinate-space bug the shipped code has.

- Category E, severity medium, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: The document shows the correct design and the sample ships something else, so the two disagree on the one point that determines whether a dragged piece tracks the finger. A reader comparing them cannot tell which is authoritative: the helper names are plausible and specific enough to look like real code, and searching the sample for them finds nothing, with no note explaining the difference. The step-2 snippet is also incomplete in a way that matters -- its onActionUpdate computes two locals and never assigns them, so as printed the drag does nothing at all.
- Fix: Quote the handler as it ships, including the bounds check and the assignments; or add the two scaling helpers to the sample so the published snippet is accurate, which would also fix the coordinate-space defect.

### `HW-08-0048` - Three grids declare a column template and then position every one of their items absolutely, so the grid layout is bypassed entirely, and the comment above it names a different column count.

- Category C, severity medium, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: A Grid exists to compute positions from a row and column template, and position() overrides that computation for every child, so the container is doing layout work whose entire result is discarded. What remains is three nested Grid components, each iterating seven items and measuring a four-column template, to achieve what a Stack with absolutely positioned children does directly -- and Stack is the container documented for exactly this. The stale comment compounds it: a reader adjusting the board reads five columns, counts four in the template, and cannot tell which was intended, when in fact neither number affects anything on screen.
- Fix: Replace each Grid/GridItem pair with a Stack holding positioned children, and delete the columnsTemplate and the comment. If the Grid is kept for another reason, correct the comment to match the template.

### `HW-08-0049` - The avoidAreaChange listener registered when the window stage is created is never released, and one of the two values it maintains is bound but never used.

- Category B, severity medium, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: The callback is registered against the main window and holds a closure writing into AppStorage; the ability's own comment on the destroy hook says UI resources are released there, and this one is not. The unused bottom value is the visible half of the same carelessness: the page is laid out full screen, so the navigation indicator overlaps the bottom of the board, and the value that would pad for it is bound to a field the layout never reads. The pieces' start positions run to y 185 in design units and the targets to 510, so the lower part of the board is where the overlap lands.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) from onWindowStageDestroy. Add the bottom inset to the safeAreaPadding call, or drop the unused @StorageProp.

### `HW-08-0054` - The eraser clears a fixed 20 by 20 square anchored at the touch point rather than centred on it, and ignores the brush size the child selected.

- Category C, severity medium, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: Passing the touch point as the rectangle's upper-left corner puts the whole erased square below and to the right of the finger, so a child rubbing out a mark watches the ink disappear from beside where they are pointing and has to aim ten units up and left to hit it. The offset is a constant twenty units, which on a drawing board is roughly a fingertip -- large enough that the eraser feels broken rather than imprecise. The fixed size is the second half: the app offers a brush from 1 to 10 and applies it to the pen, so a child who has drawn a fine line gets an eraser twenty times its width, with no way to make it smaller.
- Fix: Centre the rectangle on the touch and scale it with the selected size: 'const r = this.paintSize * 2; this.context.clearRect(x - r / 2, y - r / 2, r, r);'.

### `HW-08-0055` - Clearing the board passes a hardcoded device resolution to a canvas that is sized as a percentage, so the numbers are meaningless and only happen to be large enough.

- Category B, severity medium, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: The two numbers describe a phone's pixel dimensions and are being used where the surrounding code uses vp, so they are not the canvas size on any device -- they merely exceed it on all of them, which is why the button appears to work. That makes the call a coincidence rather than a clear, and it hides the actual requirement: nothing in the file records how large the drawing surface is. On a tablet or a 2in1 window wider than 1080 vp the rectangle would stop covering the canvas and the clear would leave ink along the right edge, with no error and no obvious cause.
- Fix: Clear the canvas's own extent: read this.context.width and this.context.height, or cache the component size from onAreaChange as the sibling handwriting sample does, and pass those.

### `HW-08-0056` - The custom-colour button in the pen sheet has no click handler, so it is a fully styled control that does nothing.

- Category B, severity medium, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: The button is rendered at full opacity with a brand-coloured label beside a working confirm button, so nothing distinguishes it from a live control -- a child taps it and the sheet neither changes nor closes. The sample already has a convention for features that are only mocked up, applied four buttons earlier in the same file, and this one does not use it. The gap matters more here than for the canvas buttons because this sheet is where the pen colour is chosen, so a user reaching for a colour that is not in the twenty-four presets is left with a dead end rather than a message.
- Fix: Give the button the same ONLY_DISPLAY toast the canvas buttons use, or disable it with enabled(false) so it reads as inactive.

### `HW-08-0057` - Eight visible labels and the placeholder toast are Chinese literals in a constants file, and the two locale directories the app ships contain no application strings at all.

- Category B, severity medium, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: The interface encodes the split: img is a Resource and txt beside it is not, so every label on the toolbar a child looks at is unreachable from the resource system while the icon above it is not. The locale directories make it worse rather than better, because they announce that the app is localised and then contain nothing: the eleven strings that do go through $r have no en_US translation, so an English device shows Chinese for those too. The result is an app that ships two empty locale folders and eight labels that could not be translated even if they were filled in.
- Fix: Type ButtonType.txt as ResourceStr and move the eight labels and the toast text into base/element/string.json, then populate en_US/element/string.json with every application key rather than only the scaffold three.

### `HW-08-0058` - The avoidAreaChange listener is never released, and the bottom inset it maintains is bound by the page but never applied.

- Category B, severity medium, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: A window listener that outlives the window is a leak, and the ability's own destroy hook is the place the comment says UI resources are released. The unused bottom inset is the visible half: the page is laid out full screen and its lowest element is the four-button toolbar, so the navigation indicator sits on top of the very controls the child taps to change colour and brush. The value that would move them clear is already computed, already converted and already bound to a field on this component.
- Fix: Hold the callback in a field and call windowClass.off('avoidAreaChange', callback) from onWindowStageDestroy. Add bottom: this.bottomRectHeight to the page's padding, or drop the unused @StorageProp.

### `HW-08-0061` - Eight alignRules calls are applied to children of Column and Row, and the reference states the attribute only takes effect inside a RelativeContainer.

- Category C, severity medium, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: Eight attribute calls state the layout intent -- centre this in its parent -- and none of them does anything, because the containers are Column and Row, which position children by their own main-axis and cross-axis rules. The centring that is actually visible comes from the Column defaults, so the page looks approximately right and the attributes read as the mechanism achieving it. Anyone changing the layout will adjust the alignRules first and see no effect, and the anchor '__container__' reinforces the misreading by naming a relative-layout concept that has no meaning here.
- Fix: Delete the eight alignRules calls and set alignment on the containers -- alignItems and justifyContent on the Column and Row -- or wrap the card in a RelativeContainer if relative anchoring is genuinely wanted.

### `HW-08-0062` - The page writes popup visibility straight into the imported JSON module, so annotation state is process-global and survives the page being destroyed.

- Category B, severity medium, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: An ES module object is created once per process and shared by every importer, so the display flags are not this component's state -- they are the module's. @State on a reference into it observes the reference, not the fields, so the decorators give a false impression of ownership while the writes land in shared data. The effect is that leaving the page and returning restores whatever popup flags were last written rather than the initial values in the JSON, and a second instance of the page would share them. The reassignment on line 187 makes the coupling explicit: it takes the array back from the module every tap, which is only meaningful because the two are the same object.
- Fix: Copy the data on entry -- 'this.testValue = JSON.parse(JSON.stringify(data.content[0]))' in aboutToAppear, or map the JSON into freshly constructed objects -- so the imported module stays immutable and the flags belong to the component.

### `HW-08-0063` - The handlers that look like toggles always evaluate to true, so tapping an annotated word a second time cannot close its popup.

- Category B, severity medium, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: The negation makes the two handlers read as toggles, and they are not: the reset that precedes it, which exists to close any other open popup, also clears the flag being toggled, so every tap opens and no tap closes. A child tapping the title to dismiss its own annotation gets the same popup reopened. The behaviour is masked because the popups also close on an outside tap through onStateChange, so the only symptom is that the word itself is not a dismiss target. The body handler, written three lines differently, shows the intended form and is the one that is honest about what happens.
- Fix: Capture the previous value before the reset and toggle against that -- 'const wasOpen = this.title.display; ...reset...; this.title.display = !wasOpen;' -- or assign true directly, as the body handler does, and accept that a second tap does not close.

### `HW-08-0064` - The page expands into both system safe areas and pads for neither, so the back button and title sit under the status bar.

- Category C, severity medium, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: expandSafeArea is an opt-in: it moves the component's drawing area under the system bars, and the component then owns the job of keeping its content clear of them. Here nothing does, so the header -- the back button and the screen title, the two controls a child needs first -- is drawn into the status bar region at the top of the window. The bottom edge is expanded too, and the poem card's 40-unit bottom margin is the only thing between the last line of the poem and the navigation indicator. Because the sample never reads an avoid area, there is no value available to fix it with at the point of use.
- Fix: Either drop the expandSafeArea call and let the framework inset the page, or keep it and pad the content: read both avoid areas in the ability, store them in AppStorage, and apply them as padding on the root Column.

### `HW-08-0065` - Underline widths and popup widths are character counts multiplied by hand-fitted constants, so both drift away from the text they measure.

- Category B, severity medium, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: The underline is meant to sit exactly beneath the word it annotates, and its length is derived from a per-character width that was measured once by hand for one font size. Three different multipliers appear because three font sizes do, and a fourth expression exists purely to undo letterSpacing -- so the relationship between text and rule is encoded in four places and in the data file. Any change to font size, letter spacing or typeface breaks all of them at once, and so does the system font-size accessibility setting, which scales text without touching these constants: the orange rule then runs short or long under every annotated word. The popup width has the same problem with a harder edge, since the 220 cap truncates the author's 65-character biography into a fixed box.
- Fix: Measure instead of counting: size the Divider from getMeasureUtils().measureTextSize on the word with the same font and letter spacing, and let the popup size itself to its message rather than multiplying a stored character count. The width and num fields can then come out of data.json.

### `HW-08-0069` - Circles are swept with an end angle of 360 where the API takes radians, so the arc runs more than fifty-seven full turns.

- Category B, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: The value is a degree measure passed to a radian parameter, and it survives only because sweeping far more than one turn still produces a closed circle. Nothing about the code says so: a reader sees 0 to 360 and concludes the API takes degrees, then writes 180 for a semicircle and gets a shape swept through twenty-eight turns instead of half of one. The same misreading is already in the published document, so it propagates to anyone who copies the snippet rather than the sample. Math.PI * 2 is exact, self-documenting and the same length to type.
- Fix: Sweep a full turn in the unit the API takes: arc(x, y, r, 0, Math.PI * 2).

### `HW-08-0070` - Beginning any shape wipes the entire canvas, so only one shape can ever exist and any freehand drawing is destroyed by the next shape.

- Category B, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: The document titles this sample Canvas绘制固定图形 (drawing fixed shapes with Canvas) and the toolbar offers three of them, which sets the expectation that shapes can be combined -- a rectangle and a circle next to each other, or a shape added to a freehand drawing. Instead each new shape destroys everything on the board, so the canvas holds exactly one object at a time and a child who draws a picture and then reaches for the circle tool loses the picture with no warning and no undo. The clear is also indistinguishable in code from the clear-all button, using the same two literals, so its purpose -- making room for the live preview to be redrawn -- is not recoverable from reading it.
- Fix: Do not clear on Down. Erase only the previous preview, which the Move handler already attempts, or draw the in-progress shape on a second overlaid Canvas and commit it to the main one on Up, which removes the need to erase anything.

### `HW-08-0071` - The live shape preview is erased by painting the background colour over it, so dragging a shape scrubs away whatever was already drawn underneath.

- Category C, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: A bitmap canvas has no layers, so erasing the previous preview necessarily removes everything else in the same region: a freehand line, an earlier shape, a template. The erase area is the whole disc or rectangle swept so far, so dragging outward wipes a growing area of the picture, and dragging back in does not restore it. The two mechanisms disagree as well -- clearRect punches a transparent hole while fill stamps opaque colour -- which only looks equivalent because this page sets the canvas background to the same canvasColor; give the canvas a template image, as the sibling drawing sample does, and the rectangle would reveal it while the circle would paint over it.
- Fix: Preview on a second Canvas stacked over the first: clear the whole preview surface each Move, draw the in-progress shape there, and stroke it onto the main canvas once on Up. Nothing then has to be erased from the picture.

### `HW-08-0072` - Freehand drawing re-strokes the whole accumulated path on every touch-move, so one line costs work quadratic in its length.

- Category C, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: stroke(path) paints the entire path it is handed, not the part added since the last call, so a stroke of n move events issues 1 + 2 + ... + n segment draws instead of n -- all inside the touch handler on the UI thread, which is the budget that decides whether ink keeps up with the finger. With anti-aliasing on there is a visible consequence too: partially transparent edge pixels accumulate coverage each time they are repainted, so the beginning of a line ends up darker and thicker than its end. The shape branch in the same handler already draws only the current shape and does not have this problem.
- Fix: Keep the previous point and stroke only the new segment: beginPath(), moveTo(prevX, prevY), lineTo(x, y), stroke(), then advance the previous point.

### `HW-08-0073` - The eraser clears a fixed 20 by 20 square anchored at the touch point rather than centred on it, and ignores the brush size.

- Category C, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: Passing the touch point as the rectangle's upper-left corner puts the erased square entirely below and to the right of the finger, so a child rubbing out a mark watches ink vanish from beside where they are pointing. Twenty units is about a fingertip, large enough that the tool reads as broken rather than imprecise. The fixed size is the second half: the app offers a brush from 1 to 10 and applies it to the pen, so erasing a fine line takes a tool twenty times its width with no way to narrow it. The two stale-coordinate sites are worse still, erasing at a point from an earlier gesture entirely.
- Fix: Centre the rectangle on the touch and scale it with the selected size, and read the coordinates from the current event at every site: 'const r = this.paintSize * 2; this.context.clearRect(x - r / 2, y - r / 2, r, r);'.

### `HW-08-0074` - Nine visible labels are Chinese literals in a constants file, and the two locale directories the app ships contain no application strings.

- Category B, severity medium, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: The interface encodes the split: the icon on each toolbar button is a Resource and the caption beside it is not, so every label a child reads on the toolbar is unreachable from the resource system. The locale directories make it worse rather than better, because they announce that the app is localised and then contain nothing: even the strings that do go through $r have no en_US translation, so an English device shows Chinese throughout. The result is an app shipping two empty locale folders and nine captions that could not be translated even if those folders were filled.
- Fix: Type the label field as ResourceStr and move the nine literals into base/element/string.json, then populate en_US/element/string.json with every application key rather than only the scaffold three.

### `HW-08-0077` - Shuffling awaits a hundred separate one-byte crypto calls in sequence while the solved picture is on screen.

- Category B, severity medium, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: One byte is needed per move and a hundred are needed in total, so a single generateRandom(100) would supply the whole shuffle in one call; instead a generator is constructed and a promise awaited a hundred times over, each round trip serialised behind the last. Nothing covers the gap: the component has already rendered the pieces in order, so the child watches the finished picture until the loop completes and the board snaps into its shuffled state. Because the promise is neither awaited nor caught by the caller, there is also no point at which the app knows the shuffle is still running, and no way to show a spinner or block the first tap.
- Fix: Draw all the bytes once before the loop -- 'const bytes = (await rand.generateRandom(100)).data;' -- and index into them, or use generateRandomSync since a hundred bytes is trivial work. Await the shuffle before the board becomes interactive.

### `HW-08-0078` - The board's ForEach chooses between two identical expressions and supplies no key generator, so refreshes rely on a side effect and every tile is rebuilt on every move.

- Category C, severity medium, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: The ternary evaluates to this.blocks either way, so as an expression it is a no-op; what makes the refresh happen is the incidental read of this.isShuffle inside build, which registers the dependency. The mechanism therefore works by side effect and the code that appears to implement it does nothing, so anyone simplifying the ternary breaks the board without touching anything that looks load-bearing. The missing key generator is the second half: with the default key derived from the serialised item, every tile whose x or y changed gets a new key on each move, so the framework discards and rebuilds those components instead of updating them -- and the tap handler additionally replaces the whole array with a spread copy, which invalidates all of them.
- Fix: Give the ForEach a stable key generator -- '(item: Block) => item.id.toString()' -- since ids never change, and drop the ternary. Keep the refresh honest by making the pieces observe their own data, so neither the array copy nor the boolean toggle is needed.

### `HW-08-0079` - The board's pixel dimensions are hardcoded to 1100 by 1100 while the picture is user-selectable, so any image of another size is tiled wrongly.

- Category B, severity medium, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: The tiles are cut by positioning one background image behind a grid of fixed-size frames, so the offsets are only correct when the frame grid matches the image's real pixel dimensions. The sample states that dependency in a comment and then provides a picture picker that cannot satisfy it: choosing a picture that is not exactly 1100 by 1100 leaves the background scaled or cropped against offsets computed for a different size, so the tiles show overlapping or misaligned fragments and the assembled picture never lines up. The constraint is invisible to whoever adds an image to the resource folder, since nothing validates it and nothing fails loudly.
- Fix: Measure the chosen image and set w and h from it -- resolve the resource and read its size, or store the dimensions alongside each entry in the picture list -- and recompute blockWidth and blockHeight when the picture changes. Alternatively size the board from the layout and use backgroundImageSize so the image is scaled to the grid rather than the grid to the image.

### `HW-08-0080` - Each tile renders from a one-way @Prop copy while writing through to the parent's array, so the view is kept in step by replacing the array and toggling an unrelated boolean.

- Category C, severity medium, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: A @Prop is a one-way copy taken when the child is built, so mutating this.blocks[idx] in the parent's array does not update this.block that the tile draws from -- the tile's own state is stale the instant it is swapped. Two blunt instruments compensate: replacing the array with a spread copy, which invalidates every tile rather than the two that moved, and toggling isShuffle, which is a refresh flag rather than a piece of game state. Both are needed because neither alone is the real mechanism, and together they mean a single tap rebuilds the entire board. The data model is what forces this: Block is an interface, so no field on it can be observed.
- Fix: Make the tile observe its own data: declare Block as an @Observed class and bind it in SetPiece with @ObjectLink instead of @Prop. Swapping two tiles then updates exactly those two, and neither the array copy nor the isShuffle toggle is needed.

### `HW-08-0084` - The monitored interface is the hardcoded string wlan0 at four call sites, so a feature presented as a data-allowance monitor measures Wi-Fi traffic.

- Category B, severity medium, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: wlan0 is the wireless LAN interface, so what is being counted is traffic over Wi-Fi -- which is the traffic that does not consume a data allowance. A monitor built to tell a parent how much of the child's plan is left therefore reports the one figure that has no bearing on it, and reports nothing at all about mobile usage. The hardcoding also removes any chance of noticing: the name appears as a bare literal four times with no constant, no configuration and no fallback, and because neither call is guarded, a device or state where wlan0 is absent produces a rejected promise that nothing catches rather than a message the user can act on.
- Fix: Name the interface once as a constant and pick it from the active connection rather than assuming it -- query the cellular interface for a data allowance, or sum the interfaces that matter. Wrap both calls in try/catch so a missing interface is reported instead of rejecting silently.

### `HW-08-0085` - The baseline reading is written once and never revisited, and nothing handles the current reading falling below it, so the app can report more data remaining than the allowance.

- Category B, severity medium, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: The interface counters the baseline is drawn from are cumulative for the current session of the interface, not for the lifetime of the app, so any event that restarts them leaves the stored baseline larger than the live reading. restData is then negative, and because tempData subtracts it from the allowance, the app reports more remaining than the parent ever configured -- the ring is driven past its maximum and the number in the middle exceeds the quota. Nothing detects this, because the only test applied to tempData is whether it is below zero. The one-shot write makes it permanent: firstData is rewritten only when it is exactly 0, which after the first launch it never is, so there is no path that re-baselines. There is also no way to reset the counter for a new billing month, which a data allowance needs by definition.
- Fix: Detect the counter going backwards -- if rx + tx is less than firstData, re-baseline to the current reading and persist it -- and clamp restData at zero. Store the baseline with the period it belongs to and offer an explicit reset so a new month starts clean.

### `HW-08-0086` - Pull-to-refresh dismisses its own spinner after a fixed 500 ms without waiting for the reading it triggered.

- Category B, severity medium, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: The spinner is the app's statement that a reading is in progress, and it is being dismissed on a timer that has no relationship to the reading. Two awaited calls can take longer than 500 ms on a busy device, in which case the spinner retracts while the old figure and the old timestamp are still on screen and the new ones appear afterwards with no indication -- so a gesture that appears to have completed silently updates the display a moment later. If they finish sooner the spinner is held for no reason. Because the promise is also uncaught, a rejected read leaves the previous value in place and the spinner still retracts, reporting success for a refresh that failed.
- Fix: Await the work and clear the flag when it is done: '.onRefreshing(async () => { try { await this.tempDataMonitor(); } catch (e) { /* surface it */ } finally { this.isRefreshing = false; } })'.

### `HW-08-0087` - The allowance is parsed with parseFloat and stored unvalidated, so clearing the field makes every derived figure NaN.

- Category B, severity medium, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: NaN propagates silently through every arithmetic operation that touches it, so an empty or mistyped allowance turns the remaining figure into NaN, prints NaN in the middle of the ring through toFixed, and makes the gauge comparison false so the ring jumps to the full-100 arm. Nothing rejects the value at any stage: the save button is always enabled, the field accepts arbitrary text because no inputType is set, and there is no lower or upper bound, so zero or a negative allowance is equally accepted and makes the percentage a division by zero or an inverted scale. The state is also persistent for the session, since dataNumber is only ever written here.
- Fix: Restrict the field with inputType: InputType.NUMBER_DECIMAL, and validate before committing: reject NaN and non-positive values, keep the save button disabled until the input parses, and leave dataNumber unchanged otherwise.

### `HW-08-0088` - Context is imported from the access control module rather than from Ability Kit.

- Category A, severity medium, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: abilityAccessCtrl is the permission-checking module and this sample checks no permissions, so the storage utility is tied to an unrelated system module through a superseded @ohos path for a type that Ability Kit owns. The correct source is @kit.AbilityKit, which the access-control reference itself uses when it needs the type. The same mistake appears in the screen-time sample of this industry, in a file with the same name and the same four helpers, so it travels by copy.
- Fix: Import the type from the kit that owns it: 'import { common } from '@kit.AbilityKit';' and type the parameter as common.Context.

### `HW-08-0093` - A real Map Kit client_id is committed into the sample instead of a placeholder.

- Category D, severity medium, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: client_id identifies the developer account the map requests are billed and rate-limited against, so publishing a live one in a downloadable sample invites every reader to issue Map Kit traffic under someone else's project. It also breaks the sample in the quieter direction: a reader who builds it unchanged sees a working map, never performs the AppGallery Connect setup the document spends a section describing, and discovers the missing step only when the credential is revoked or throttled. The neighbouring sample in the same corpus demonstrates the right form, so the convention exists and this file departs from it.
- Fix: Replace the value with a mask and a comment pointing at the AppGallery Connect steps, as MotionTrajectory does, and revoke the published identifier.

### `HW-08-0094` - The trail animation and the address readout run on two unsynchronised clocks over the same journey, so the text describes a place the marker is not at.

- Category B, severity medium, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: One journey is being played twice at different speeds. The marker travels the trail in five seconds under the overlay's own animation while the caption underneath steps through the same coordinates in seven one-second jumps, so after the first tick the two are describing different points and by the end the marker has been stationary for two seconds while the address is still catching up. The panel is the only text on the screen and its whole purpose is to say where the child is, so a mismatch with the marker is the feature failing quietly. The step arithmetic is the reason it cannot line up even in principle: floor(n/6) + 1 does not divide the track into six equal parts for any n, which is why the clamp branch exists at all.
- Fix: Drive both from one clock: compute the tick interval from animationDuration and the number of samples (animationDuration / samples), or drop the interval and update the address from the overlay's own animation progress so the caption is derived from the marker rather than run alongside it.

### `HW-08-0095` - The reverse-geocoding result is indexed and non-null asserted without checking that any address came back.

- Category B, severity medium, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: A reverse-geocoding lookup returning nothing is an ordinary outcome -- a coordinate at sea, in an unmapped area, or a service that declines the request -- and it arrives as a resolved promise with an empty array, not as a rejection, so the catch beside it does not cover the case. data[0] is then undefined and reading placeName throws inside a then handler, producing an unhandled rejection rather than an error the page can show. The optional fields are the milder half: when locality or subLocality is absent the template renders the literal text undefined into the panel, which is the only caption on the screen.
- Fix: Guard before use -- 'if (!data || data.length === 0) { return; }' -- and fall back to a placeholder for each optional field rather than asserting: 'this.address = data[0].placeName ?? '';'.

### `HW-08-0098` - The player's initialize method assigns a newly created AVPlayer to its own parameter instead of the field, so the branch cannot work, and the method is never called.

- Category B, severity medium, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: The method has two paths and only the one that receives a player already works; the path that creates one leaves the object it created unreachable, so callbacks are registered on a player the class cannot subsequently play, pause or release, and every public method silently returns because the field is still null. Nothing surfaces because the method is dead -- the page reaches the player through getAVPlayer instead -- so the class ships two initialisation routines, one correct and one broken, with no indication of which is intended. The parameter is also declared non-optional while the body's whole purpose is handling its absence.
- Fix: Assign the field in both branches -- 'this.avPlayer = await media.createAVPlayer(); this.setupCallbacks(this.avPlayer, context);' -- and mark the parameter optional. Better, delete initialize and keep getAVPlayer as the single entry point.

### `HW-08-0099` - Playing a second word routes through the error handler, because no path returns the player to idle and the idle branch exists only to re-apply the source after a reset.

- Category B, severity medium, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: The design makes the error path part of normal operation. After the first sound finishes the player is left in a post-playback state, and the next tap assigns fdSrc from there; whatever the player does with that, the only route back to a playable state is the error callback calling reset, which produces idle, which re-reads the static audioUrl and starts the chain again. That is why the idle branch re-applies a source at all -- it is compensating, not initialising. The consequence is that a tap's success depends on an error being raised and swallowed, every switch between words writes an error line to the log, and any change to the error handler breaks playback rather than error reporting.
- Fix: Reset explicitly before setting a new source: in the tap handler, 'await instance.reset();' and then assign fdSrc once the player reports idle, so the transition is driven by the app rather than recovered from an error.

### `HW-08-0100` - The card advances from onActionStart while onActionEnd is empty, so a five-unit twitch flips the card and a deliberate swipe cannot be completed or abandoned.

- Category C, severity medium, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: Committing on recognition rather than on release inverts how a card carousel is expected to behave. The smallest horizontal movement that clears the recogniser's threshold changes the card, so a child resting a finger on the stack while looking at it loses their place, and there is no way to back out: once onActionStart has fired the index has already moved, and dragging back the other way does nothing because no further handler runs. The card also does not follow the finger, since nothing updates an offset during the drag -- the animation plays to completion independently of the gesture that is still in progress. The empty onActionEnd is where all of that belongs, and it is present and unused.
- Fix: Track the drag in onActionUpdate to move the card with the finger, and commit in onActionEnd only when the accumulated offsetX passes a threshold, animating back to the current card otherwise. Give PanGesture an explicit distance so the recogniser does not fire on a twitch.

### `HW-08-0101` - The player keeps all of its mutable state in static fields beside a singleton instance, and releasing the instance leaves every static stale.

- Category B, severity medium, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: The class is a singleton, so the statics buy nothing that instance fields would not -- and they cost the one guarantee a singleton offers. release() nulls the instance so the next getInstance() constructs a fresh object, and that object inherits state, isPlaying, duration, time and curAudioId from the player that was just released: a new instance can report itself as playing before it has a player. Three of the eight are never read at all (curAudioId is written in pause and stop and read nowhere; playMode and playIndex have accessors and no consumers), so the class also publishes a surface that does not exist. Making them public compounds it, since the page mutates audioUrl directly and the state handler reads it back, putting the coordination between the page and the player in a mutable global.
- Fix: Make the mutable fields instance fields and expose them through methods, so release() disposes of them with the instance. Delete curAudioId, playMode and playIndex if nothing consumes them, and pass the audio URL as an argument rather than through a static.

### `HW-08-0106` - The Map Kit client_id is nested inside the ability instead of at module level, and carries a live value rather than a placeholder.

- Category D, severity medium, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: Map Kit reads client_id from the module's metadata, so a copy nested inside an ability is not where the framework looks for it -- the same misplacement that the sports framework sample makes. The second half is that the value is real: publishing a live credential in a downloadable sample means every reader who builds it issues Map Kit traffic against someone else's project, and never performs the AppGallery Connect setup the document spends a section describing, so the missing step surfaces only when the identifier is revoked or throttled. Two samples in this corpus show the correct form -- one for placement, one for masking -- and this file matches neither.
- Fix: Move the metadata array up to module level, beside abilities, and replace the value with a mask plus a comment pointing at the AppGallery Connect steps. Revoke the published identifier.

### `HW-08-0107` - Two different distance implementations coexist for the same measurement, and the hand-written one is used where the document prescribes the Map Kit call.

- Category B, severity medium, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: One quantity, the distance from home, is computed by two different implementations on two different code paths, so the answer depends on which path last ran -- and they do not agree, because the hand-written version models the Earth as a sphere of radius 6371 km while the platform version does not, and because only the hand-written path applies a coordinate transform to its inputs first. The document names the platform call as the mechanism, so a reader following the document finds the sample using something else for the alarm itself. Carrying a hand-rolled geodesy helper in a sample that already depends on Map Kit also adds a maintenance surface for no benefit, and its doc comment declares kilometres where the body returns metres.
- Fix: Delete the haversine helper and use map.calculateDistance on all three paths, passing the coordinates in the system it expects so the transform is unnecessary.

### `HW-08-0108` - Continuous positioning is requested at a one-second interval and kept running in the background, which is the highest cost the API can be asked for.

- Category B, severity medium, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: A one-second interval is the shortest a continuous request accepts, and it is being asked for by a feature whose only question is whether a child has crossed a radius measured in hundreds of metres -- a threshold that cannot be crossed meaningfully faster than once every ten or twenty seconds on foot. The cost is paid continuously and in the background, on a child's device, where the GPS radio is the largest single battery draw available; and because each fix also triggers a reverse-geocoding request, the sample makes a network call every second as well. The safety benefit of the extra frequency is nil, since the alarm threshold is coarse, while the cost is the device running flat -- which disables the alarm entirely.
- Fix: Raise the interval to match the threshold -- an interval of 10 to 30 seconds is ample for a several-hundred-metre radius -- and only reverse-geocode when the safe state actually changes rather than on every fix.

### `HW-08-0112` - A secure random generator is constructed inside the shuffle loop, once per swap, for a single byte each time.

- Category B, severity medium, confidence confirmed
- Features: KIDS-15
- Document: `huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
- Why: createRandom allocates a cryptographic random source, and the loop allocates twenty-four of them to obtain twenty-four bytes -- a single generateRandomSync(24) before the loop supplies the same entropy from one generator. The cost lands in aboutToAppear, on the UI thread, immediately before the grid is first laid out, and again every time the child restarts or switches between the three by three, four by four and five by five layouts, which is the app's main interaction. The sibling sliding-puzzle sample in this industry makes the same mistake asynchronously, awaiting a hundred separate one-byte calls.
- Fix: Create one generator and draw all the bytes before the loop: 'const bytes = cryptoFramework.createRandom().generateRandomSync(numbers.length).data;' and index into it with the loop counter.

### `HW-08-0113` - The flag guarding the tap handler is named for having started and is tested for being false, and the win branch rewinds the counter it has just advanced.

- Category B, severity medium, confidence confirmed
- Features: KIDS-15
- Document: `huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
- Why: A boolean called isStart that must be false for the game to accept taps inverts the reading of every site that touches it, and the page and the component disagree on the name -- ifStart on one side, isStart on the other -- so neither spelling can be searched for reliably. The counter rewind is the same kind of confusion made concrete: searchNumber is the number the child must click next, the branch fires when it has passed the last tile, and instead of leaving it there the code sets it back to the last tile's number. That value still matches the final tile, so the completion branch is re-entered on any further tap and textTimerController.pause() is called again on an already-paused timer.
- Fix: Rename the flag for what it gates -- isPaused, or isWaitingToStart -- and use it without negation, spelling it the same way in the page and the component. Leave searchNumber past the end when the grid is complete, and guard the completion branch so it runs once.

### `HW-08-0116` - The play-count check compares a field that is initialised to one and never written, so the condition it guards is always true and nothing is counted.

- Category B, severity medium, confidence confirmed
- Features: KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/16_die_rolling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
- Why: The field, its comment and the comment above the callback all describe a counter that limits how many times the animation repeats, and there is no counter -- the guard is a constant expression. It works for the shipped behaviour only because the intended count happens to be one, so stopping after the first loop and stopping unconditionally coincide. Anyone changing gifLoopCount to 3 to make the die tumble three times gets no change at all, because the value is compared rather than decremented, and the animation still stops after one loop. That is the worst shape for a constant: it is presented as configuration, it is documented as configuration, and it is inert.
- Fix: Count the loops: keep a mutable counter, increment it in setLoopFinish, and compare it against gifLoopCount -- 'this.loopsDone++; if (this.loopsDone >= this.gifLoopCount) { this.gifAutoPlay = false; ... }' -- resetting it when a new roll starts. If a single loop is always wanted, delete the field and the test.

### `HW-08-0117` - The GIF controller holds a decoded buffer and a hardware decoder that are released on every replacement but never when the page goes away.

- Category B, severity medium, confidence confirmed
- Features: KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/16_die_rolling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
- Why: The sample demonstrates that it knows the controller owns releasable resources -- it calls destroy() before replacing one, twice -- and then omits the one place where the replacement never comes. What is left holding is a fully decoded animation buffer plus, because setOpenHardware(true) is set, a hardware decoding path, which is a scarcer resource than memory: hardware decoders are a small per-device pool shared across applications. Leaving the page therefore leaks both, and returning to it allocates another in aboutToAppear. The destroy() in aboutToAppear at line 39 is also of no use for this, since it acts on the freshly constructed field created by the initialiser rather than on anything that was ever loaded.
- Fix: Add 'aboutToDisappear(): void { this.model.destroy(); }' so the live controller is released with the page, and drop the destroy() at the top of aboutToAppear, which acts on an empty controller.

### `HW-08-0009` - The keypad iterates a string array but declares its item as a number.

- Category B, severity low, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The declared item type contradicts the array it comes from, and the two uses hide it in opposite directions: num.toString() is a no-op on a string, and this.inputData += num concatenates rather than adding because the value really is a string. The comparison that decides the verification, Profile.ets:65, is a string comparison against the computed answer, so the code only works because the annotation is wrong. Anyone taking the annotation at face value and switching arrNumber to a number array would turn that concatenation into arithmetic and break the gate.
- Fix: Type the item as string to match arrNumber, and give the ForEach an explicit key generator.

### `HW-08-0010` - Half the user-visible text is resolved through string resources and the other half is hardcoded Chinese literals in the page.

- Category B, severity low, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: Both conventions are present in the same builder call, so the file demonstrates the correct one and then does not follow it. Every label a child actually reads on the profile screen -- the two tab names, the three list rows and the three card titles -- is unreachable from string.json, which makes the sample untranslatable in exactly the places a children's app would need to be. Movie is declared with "name: string" at Objects.ets:74-81 while its sibling field rec is a Resource, so the model encodes the same split.
- Fix: Move the eight literals into string.json and pass $r references, and type Movie.name and the builder parameters as ResourceStr so the model cannot carry a bare literal.

### `HW-08-0011` - The divide-by-zero guard tests for an operator the generator never produces, so it can never fire and would still miss the case if division were enabled.

- Category B, severity low, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The guard is written against a symbol that is not in the alphabet, so it never fires and the regeneration loop it controls always exits on its first pass. That is harmless today only because division is equally unreachable. The two facts point in opposite directions for whoever enables division next: the natural way to do that is to add the Chinese division sign to arrSymFir, and the guard compares against the ASCII '/', so it would still not fire and mulAndDiv's else arm would divide by a zero operand. The loop also carries the cost of a full regeneration check on every expression for a condition that cannot occur.
- Fix: Compare against the same symbol the generator emits -- take it from arrSymFir rather than repeating a literal -- and add the division sign to arrSymFir and to the branch in mulAndDiv together, so the guard and the operation cannot drift apart again.

### `HW-08-0012` - The documented project tree omits the backup extension ability that the sample ships and registers.

- Category E, severity low, confidence confirmed
- Features: KIDS-02
- Document: `huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
- Why: The 工程目录 section is the reader's map of the sample, and it is missing an entry point. A backup extension ability is not incidental -- it is what runs when the system backs up and restores this app's data, it is declared in module.json5 with its own profile, and a reader comparing the document against the unpacked project has to work out on their own whether the extra directory belongs there or is left over.
- Fix: Add the entrybackupability directory and EntryBackupAbility.ets to the 工程目录 listing.

### `HW-08-0016` - Each saved page is written with a .jpeg extension while the data written is PNG, because toDataURL is called with no arguments.

- Category B, severity low, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: The file the sample leaves in the sandbox claims to be a JPEG and contains a PNG. Nothing inside this sample notices, because Finish.ets:26 renders it with Image(url), which decodes by content rather than by extension. It matters at the first point the file leaves the app -- shared to a parent, exported to the gallery, uploaded, or opened by any tool that trusts the extension -- which is the natural next feature for a screen whose purpose is showing the child's finished work. The two formats also differ in what they preserve: the strokes are flat colour on a flat background, which is what PNG is for, so the correct fix is to keep PNG and rename the file rather than to switch the export to JPEG.
- Fix: Name the file .png to match the default, or pass the format explicitly -- toDataURL('image/png') -- and derive the extension from the same value rather than repeating it as a literal.

### `HW-08-0020` - Seven of the eleven strings in the resource file belong to a different sample and are referenced nowhere, while this sample's own button label is a hardcoded literal.

- Category B, severity low, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: The resource file is a leftover from a different project. Four of the dead entries are not text at all but numbers and percentages -- 12, 14, 90%, 100% -- stored as strings, which is a layout convention this sample does not use and which the integer.json beside it would be the place for anyway. The result is that the one file a translator or reviewer opens describes a colour palette with a doodle pen and a placeholder toast, none of which exists here, while the two things that are on screen -- the clear button and the characters being practised -- are not in it. palette_toast in particular is a stub message telling a developer to implement a feature, shipped in a sample where no feature calls it.
- Fix: Delete the seven palette_ entries, and add the clear button's label so both buttons resolve through $r.

### `HW-08-0021` - drawLine takes a rendering context as a parameter and then sets the pen on a different object, so the parameter is only half honoured.

- Category B, severity low, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: The signature promises the function draws on whichever context it is given, and the body only does so for the geometry. Because every caller happens to pass this.canvasContext the two objects are identical today, so the mistake is invisible; the moment a second canvas is introduced -- a preview thumbnail, an export surface, the completion screen re-rendering a page -- the guide grid appears on the passed context with the wrong line width and colour, taken from whatever the main board was last set to. The width and height read at the top come from ctx, which makes the inconsistency plain within three lines of each other.
- Fix: Set the pen on the parameter: ctx.lineWidth = 0.5 and ctx.strokeStyle = Color.Gray, so the function touches nothing but the context it was handed.

### `HW-08-0022` - The third implementation step shows a call that is not the API, and sends the reader to the atomic service documentation for a method the sample uses from the standard reference.

- Category E, severity low, confidence confirmed
- Features: KIDS-03
- Document: `huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
- Why: The snippet is the one a reader copies for the feature the step is about, and it neither compiles nor matches the sample it claims to describe -- the attribute name is wrong and the callback that does all the work is missing. The prose in step 1 repeats the same misspelling, so the document names the API incorrectly twice. The link then points at the atomic service reference, which is a different product surface with its own constraints, for an application that is not an atomic service; a reader following it lands outside the documentation set the rest of the page uses.
- Fix: Replace the step 3 snippet with the real call, onTouch with its handler, or drop it and refer back to step 1. Point the clearRect link at the CanvasRenderingContext2D reference the rest of the document uses, and correct ontouch to onTouch in the step 1 prose.

### `HW-08-0033` - The preference helpers take the store name and the key in opposite orders, and the delete helper never flushes.

- Category B, severity low, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: Two adjacent functions in one class taking the same pair of strings in opposite orders is a trap that produces no error: a transposed call reads from a store named after a key, gets nothing, and returns the default. The call sites happen to be correct today, which is why nothing has surfaced. The delete helper carries a second defect that is dormant only because it is unused -- it removes the pair from the in-memory instance and never persists that, so the entry reappears on the next launch -- and its async marker promises the caller a completion it does not represent.
- Fix: Use one parameter order across all three helpers, or take a single options object so the two names cannot be transposed. Add 'await store.flush();' after deleteSync.

### `HW-08-0034` - The implementation section's code block lists bare API names instead of calls, including one that invokes an event handler as a static method.

- Category E, severity low, confidence confirmed
- Features: KIDS-04
- Document: `huawei_industry_tree/08_children_education/docs/04_control_usage_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
- Why: This block is the only code the document offers for the feature it describes, and a reader cannot take anything from it: the call that schedules the lock is shown without the callback and the delay that constitute the mechanism, and the countdown handler is shown as a static method of a component, which is not how any event attribute in ArkUI is used. The neighbouring documents in this industry quote real fragments from their samples; this one lists identifiers. The three implementation steps above the block describe the design accurately, so the gap is entirely in the sample code.
- Fix: Replace the identifier list with the real fragments from the sample: the setTimeout call in MainPage.setTime, the aboutToDisappear clearTimeout, and the TextTimer element with its isCountDown, count and onTimer.

### `HW-08-0040` - The two comments naming the grid lines are swapped, in the sample and in the document that quotes it.

- Category B, severity low, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: The comments are the only thing distinguishing two nearly identical four-line blocks whose difference is the argument order, and they name the wrong one each time. Someone changing the grid -- adding the star points a Go board needs, drawing the edge lines thicker, or handling a non-square board -- reads the label rather than re-deriving which coordinate is held constant, and edits the wrong block. The error is duplicated into the published document, so a reader who never opens the sample still receives it.
- Fix: Swap the two comments so each names the line its block draws.

### `HW-08-0041` - Stones are drawn with an arc of 6.28 radians instead of a full turn, leaving the stroked outline open.

- Category B, severity low, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: fill implicitly closes the region, so the disc is solid, but stroke draws the path as it stands and leaves a gap at the three-o'clock position where the arc stopped short. On a white stone against the yellow board that gap is the outline's only contrast, so the notch is visible at the exact point every stone starts. Writing the constant as a two-decimal literal is what causes it: Math.PI * 2 is exact, shorter, and cannot be mistyped. The same literal will be copied into any board game derived from this sample, which the document explicitly invites -- it names gomoku and Chinese chess as the other targets.
- Fix: Sweep a full turn -- this.context.arc(realX, realY, this.boardLen / 2, 0, Math.PI * 2) -- or call closePath() before stroking.

### `HW-08-0042` - The full-screen window call is neither awaited nor caught, and the ability ships with the placeholder application label 'label'.

- Category B, severity low, confidence confirmed
- Features: KIDS-05
- Document: `huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
- Why: A rejected promise with no catch is an unhandled rejection, and this one governs whether the page is laid out under the system bars -- the layout the whole board geometry then assumes. If it fails the app keeps running with a different usable area and nothing records why. The label is smaller but reaches further: it is the name under the icon on the launcher, so the sample installs as an app called 'label'. Both are the kind of placeholder a reader copies without noticing, since neither produces a warning.
- Fix: Attach then/catch to setWindowLayoutFullScreen as the sibling samples do, and give EntryAbility_label a real name in base/element/string.json with locale variants alongside it.

### `HW-08-0050` - The function that snaps a piece to its target is named animatePieceToTarget and performs no animation, although the document presents the snap as an animateTo effect.

- Category B, severity low, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: The name states a behaviour the body does not implement, so a reader looking for the snap animation finds a function that already claims to be it and stops. What the child sees is a piece jumping instantly from where they released it to the target, which in a game aimed at young children is the moment that most wants easing -- it is the feedback that the piece was accepted. The file already imports and uses animateTo correctly a hundred lines above for the rotation, so the mechanism is present and only this call site was left out.
- Fix: Wrap the two assignments in this.uiContext.animateTo with a short duration and an ease-out curve, matching the rotation animation, or rename the function to setPiecePosition so it no longer promises what it does not do.

### `HW-08-0051` - The piece component declares six state fields that are never read or written, seeded with magic numbers that contradict the ones its parent uses.

- Category B, severity low, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: Six state variables that nothing reads still cost an observation slot each and, more damagingly, tell the next reader that the component is screen-aware when it is not -- the scaling is entirely the parent's job. The two magic numbers make it worse by disagreeing with the parent's design size in the third decimal, so a reader trying to find the authoritative reference size finds two. The unused @Prop and the inert @Observed are the same pattern: isPlaced is copied into every piece on every render and never consulted, and @Observed without a matching @ObjectLink observes nothing.
- Fix: Delete the six unused @State fields and the unused @Prop isPlaced. Remove @Observed from PuzzlePieceItem unless an @ObjectLink is introduced to pair with it.

### `HW-08-0052` - The tangram's constants live in a file named CalendarConstants.ets, and the document's project tree repeats the name without noticing.

- Category E, severity low, confidence confirmed
- Features: KIDS-06
- Document: `huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
- Why: The file is the one place a reader goes to change the puzzle -- the seven shapes, their start positions and the four solution variants are all in it -- and its name says calendar, which is a different sample in this same corpus. The three exported classes are all named for what they hold, so the mismatch is only in the filename, which is exactly the part a reader searches by. Reproducing it in the published project tree removes the last chance to catch it, and the comment beside it, 静态常量类 (static constants class), describes the contents without noting that the name does not.
- Fix: Rename the file to PuzzleConstants.ets, update the two imports, and correct the 工程目录 entry in the document.

### `HW-08-0059` - beginPath and closePath are called on the rendering context while all stroking goes through a Path2D, so neither call affects anything.

- Category B, severity low, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: beginPath and closePath operate on the context's internal current path, which is what the no-argument stroke() would draw. This sample uses the overload that takes an explicit Path2D, so the context's current path stays empty for the whole session and both calls are no-ops. They are not harmless: they state that the context's path is being managed, which sends a reader looking for the pairing that governs the stroke, and closePath in particular implies the shape is being closed on release -- a freehand line should not be. The two conventions, context path and Path2D, are alternatives and this file half-adopts both.
- Fix: Pick one. Either drop the Path2D and use context.beginPath/moveTo/lineTo/stroke, or keep the Path2D and delete the context's beginPath and closePath calls.

### `HW-08-0060` - A grid sets rowsGap twice in a row with different values, and a number constant shares its name with a colour array exported from the same file.

- Category B, severity low, confidence confirmed
- Features: KIDS-07
- Document: `huawei_industry_tree/08_children_education/docs/07_draw_board.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
- Why: The duplicated rowsGap is dead code that reads as a deliberate value, so someone tuning the canvas-colour grid changes the first one and sees nothing happen. The name collision is the more confusing of the two: PEN_COLOR means the list of available colours in one place and the index of the default colour in the other, they live in the same file, and both are imported into the same page, so a reader has to check the qualifier on every use to know which is meant. The default index 19 is also a bare number whose meaning -- black, the twentieth entry of the array -- is not stated anywhere.
- Fix: Delete the first rowsGap. Rename the constant to DEFAULT_PEN_COLOR_INDEX, or derive it, so it cannot be confused with the colour list.

### `HW-08-0066` - Component ids are taken from the displayed word, which the reference requires the developer to guarantee unique.

- Category B, severity low, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: Deriving an identifier from displayed content hands the uniqueness guarantee to the poem. It holds for the four annotated words shipped here and does not hold in general: repetition is a normal device in classical Chinese verse, and any poem that annotates the same word twice produces two components claiming one id. The attribute is also not used for anything in this sample -- no getInspectorByKey, no focus control, no test hook references it -- so the risk is taken for no benefit.
- Fix: Drop the id, or build one that cannot collide, such as `${index}-${idx}` from the two ForEach indices already in scope.

### `HW-08-0067` - The same four-line loop that closes every open annotation is copied into four handlers.

- Category B, severity low, confidence confirmed
- Features: KIDS-08
- Document: `huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
- Why: Closing every popup before opening one is the single rule this page runs on, and it is written out four times, so the rule lives nowhere. The fourth copy already differs from the other three -- it resets the body annotations but not the title and author flags -- which is exactly the drift duplication produces, and it means a popup dismissed by an outside tap leaves the title and author flags in whatever state they were in. Adding a fourth annotatable element to the template would mean finding and editing all four sites.
- Fix: Extract a closeAllPopups() method that clears the body flags along with the title and author flags, and call it from all four places.

### `HW-08-0075` - The triangle is stroked twice, and the tool identity is a bare number in which 3 means freehand.

- Category B, severity low, confidence confirmed
- Features: KIDS-09
- Document: `huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
- Why: Stroking the same path twice paints its anti-aliased edge pixels twice, so the triangle's outline is darker and slightly thicker than the rectangle's and the circle's drawn with the same pen -- a visible inconsistency between three tools that should match, repeated on every move event of the drag. The magic numbers are the reason the extra call is easy to miss: 3 for freehand and less-than-3 for a shape is a convention with no name, and the default value 3 tells a reader nothing about which tool the app starts in. An enum would make the trailing stroke's scope obvious.
- Fix: Delete the stroke() inside the triangle branch and let the trailing call cover all three shapes. Replace the numbers with an enum -- ShapeTool.Freehand, Rectangle, Circle, Triangle -- and branch on that.

### `HW-08-0081` - getValidMoves is declared with one parameter and called with two, and the only @Observed class in the project is never instantiated or paired with an @ObjectLink.

- Category B, severity low, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: Typing a field as Function erases its signature, so the compiler accepts any argument list and the extra currentBlocks argument reads as though the function needs the board state when it works purely from the index and the column count. A reader changing the move rules will look for the parameter that is not there. The @Observed decorator has a matching problem: it takes effect by wrapping a class constructor, and an object literal is not constructed through it, so the config is a plain object and the decorator observes nothing -- which is masked here because @StorageLink already propagates whole-object assignment.
- Fix: Declare the arrow function with a real signature -- 'private getValidMoves(emptyIndex: number): number[]' -- and drop the second argument at the call site. Either construct FigMessage with new and pair it with @ObjectLink where a child needs to observe it, or remove @Observed and declare FigMessage as an interface.

### `HW-08-0082` - The documented project tree names a constants file and a model directory that do not exist under those names in the sample.

- Category E, severity low, confidence confirmed
- Features: KIDS-10
- Document: `huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
- Why: The 工程目录 section is the reader's map, and two of its seven entries cannot be found by the names given: searching the sample for CalendarConstants returns nothing, and following the model path fails on the missing plural. The first mistake is traceable -- CalendarConstants.ets is the real filename in the neighbouring tangram sample, so the tree was copied between documents -- which means a reader who does find a CalendarConstants.ets in this corpus finds the wrong sample's file.
- Fix: Correct the two entries to constants/Constants.ets and models/Pieces.ets.

### `HW-08-0089` - The timestamp helper round-trips a Date through its own milliseconds, calls the result Beijing time, and hardcodes a Chinese locale.

- Category B, severity low, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: Three of the five statements do nothing: getTime followed by new Date returns the same moment, and the two variables named for Beijing hold whatever the device's zone is. A reader is told the app normalises to a fixed zone and it does not, which matters for a refresh timestamp shown to a parent who may not be in that zone. The hardcoded locale is the part that is visibly wrong: the surrounding labels come from string resources and follow the device language, and this one string is pinned to Chinese formatting, so a device set to another language shows a zh-CN clock among translated labels.
- Fix: Reduce the function to a formatter over new Date(), and take the locale from the system rather than a literal. If a fixed zone is genuinely wanted, pass timeZone: 'Asia/Shanghai' in the options so the name matches the behaviour.

### `HW-08-0090` - The delete helper is declared async, removes the entry without persisting it, and is never called.

- Category B, severity low, confidence confirmed
- Features: KIDS-11
- Document: `huawei_industry_tree/08_children_education/docs/11_data_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
- Why: The helper removes the pair from the in-memory instance and stops there, so the entry returns on the next launch -- the opposite of what a delete means to its caller. The async marker compounds it by promising a completion the body never represents, since nothing inside it is awaited. It is dormant only because it is unused, and the one value this app stores is the traffic baseline that can never be reset, which is exactly what this helper would be called for.
- Fix: Add 'STORE.flushSync();' after the delete, matching the save helper, and drop the async marker.

### `HW-08-0096` - The marker animation is built from a single image and set to repeat forever, and neither it nor the trace overlay is released.

- Category B, severity low, confidence confirmed
- Features: KIDS-12
- Document: `huawei_industry_tree/08_children_education/docs/12_map_location.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
- Why: PlayImageAnimation exists to cycle a sequence, and a sequence of one has nothing to cycle -- the marker is animated in name only, so the pulsing effect the panel's radio-wave icon suggests never appears, and the reader is left with an animation API demonstrated in a form that cannot show what it does. The infinite repeat count then keeps that empty animation running for as long as the marker exists, and since the page never disposes of the marker, the overlay or the controller, everything the map allocated stays allocated. The comment on the line above admits the sample is a placeholder for the reader's own images, which is the right instinct and would have been the place to note that more than one is needed.
- Fix: Push the frames the animation is meant to cycle, or drop the animation if a static marker is intended. Add an aboutToDisappear that stops the marker animation and clears the trace overlay before the controller goes away.

### `HW-08-0102` - Two of the three captions on every card are hardcoded strings while the third on the same object is a resource.

- Category B, severity low, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: One object carries an image and a Chinese name through the resource system and the pinyin and English name outside it, so a single card is half localisable. The English name is the field that matters most: this is a bilingual vocabulary app whose whole purpose is presenting a word in two languages, and the English half cannot be adapted for a learner whose second language is not English. The pinyin literals also carry hand-inserted padding -- 'lǎo    hǔ' and 'kǎo    lā ' contain runs of spaces used to align the two syllables under the characters above, which is layout encoded into the data and breaks at any other font size.
- Fix: Type pinyin and englishName as ResourceStr and move all three strings into string.json, then handle the syllable alignment in the layout rather than with padding spaces in the data.

### `HW-08-0103` - The singleton accessor is async with nothing to await, and the error log passes its message in the tag parameter.

- Category B, severity low, confidence confirmed
- Features: KIDS-13
- Document: `huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
- Why: The async marker forces every caller to await a value that is already available, so four call sites in the page are written as asynchronous handlers for no reason and the accessor cannot be used from a synchronous context such as a field initialiser. The log call is the more damaging of the two: hilog's tag is a short identifier used for filtering, and putting a serialised BusinessError into it means the one message reporting a player failure has an unfilterable tag and an empty body -- so the error path that this player depends on for normal operation logs nothing readable.
- Fix: Drop async from getInstance and return MediaPlayer directly. Correct the log call to hilog.error(0x0000, 'MediaPlayer', `player error: ${err.message}`).

### `HW-08-0109` - The alarm message the parent sees is a hardcoded Chinese template literal, in a page whose every other string is a resource.

- Category B, severity low, confidence confirmed
- Features: KIDS-14
- Document: `huawei_industry_tree/08_children_education/docs/14_distance_alarm.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
- Why: This is the one string the feature exists to produce: the alarm a parent reads when a child has left the safe area. Every other piece of text on the page is translatable and this one is not, so on a device in any other language the app presents an untranslated warning at the only moment it matters. The metre suffix is baked into the same literal, so the unit cannot be localised either -- and it is currently applied to a value that is not always in metres.
- Fix: Move the message into string.json with a parameter and resolve it: '$r('app.string.distance_warning', Math.floor(this.distanceToHome))', matching the internet_switch toast beside it.

### `HW-08-0114` - The grid iterates its tiles with no key generator, so the framework falls back to a key derived from the serialised item.

- Category C, severity low, confidence confirmed
- Features: KIDS-15
- Document: `huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
- Why: Without an explicit key the default is derived from the index and the serialised item, so the key of a tile changes the moment its ifOnclick flag flips -- which happens on every correct tap. The framework then treats the tile as a different child and rebuilds it rather than updating it, discarding the very optimisation the @Trace decorators on the model were added to enable. The array is also replaced wholesale on restart and on a grid-size change, which invalidates every key at once. A stable key is available for free, since the numbers are unique within a grid.
- Fix: Key the loop on the tile's number: 'ForEach(this.gridFocusArr, (item: GridFocusItem) => { ... }, (item: GridFocusItem) => item.textNumber.toString());'.

### `HW-08-0118` - Two components each set the same attribute twice with different values, so the first call in each pair is dead.

- Category B, severity low, confidence confirmed
- Features: KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/16_die_rolling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
- Why: A duplicated attribute reads as a deliberate value, so someone adjusting the die's size edits the '100%' and sees nothing happen, and someone adjusting the button text edits the 12. The font-size pair is the more misleading of the two because the two calls are three lines apart with other attributes between them, so neither is obviously a leftover -- a reader scanning the chain finds 12 first and stops. Both are silent: the framework applies the last value with no warning.
- Fix: Delete the first .width('100%') and the first .fontSize(12), leaving one value per attribute.

### `HW-08-0119` - The module that imports the third-party GIF library declares no dependencies, and the root manifest pins a looser version than the document specifies.

- Category E, severity low, confidence confirmed
- Features: KIDS-16
- Document: `huawei_industry_tree/08_children_education/docs/16_die_rolling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
- Why: The manifest of the module that uses the package does not mention it, so the dependency is invisible at the level where the import lives -- a reader inspecting entry/oh-package.json5 to find out what the module needs is told it needs nothing. The version is the second half: the document pins 2.1.1 exactly and says the solution is based on that version, while the manifest accepts any 2.x release, so a fresh install can resolve to a later minor the sample was never tested against. For a sample whose single external dependency is the feature -- the animation is the whole app -- the two statements about it should agree.
- Fix: Declare '@ohos/gif-drawable' in entry/oh-package.json5 alongside the import that needs it, and pin the version the document names rather than a caret range.

