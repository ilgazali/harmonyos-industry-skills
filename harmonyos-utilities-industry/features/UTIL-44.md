---
id: UTIL-44
title: Orbit camera for a loaded 3D model - drag to rotate, pinch to zoom, driven by quaternions
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/44_rotate_model.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/rotate_model-0000002556688551
sample: huawei_industry_tree/15_utilities/downloads/RotateModel.zip
kits: ["@kit.AbilityKit", "@kit.ArkGraphics3D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [deviceInfo, hilog, scene, window]
permissions: []
min_api: 22
modules: [entry (entry)]
findings: [HW-15-0091, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when a `.glb`/`.gltf` model has to be inspectable by hand** -
a car, an apartment, a product on a shop page - and `Scene.load` has left you
with a static render. `Scene` gives you a camera; it does not give you a
controller. This sample is the missing controller: about 200 lines of
quaternion maths plus a touch handler that turns finger deltas into camera
orientation.

The key idea is that **the model never moves.** Everything you see is the
camera orbiting a fixed target point at a fixed distance. Drag changes the
camera's rotation quaternion; pinch changes the orbit radius; the position is
recomputed from both. That is the standard orbit-camera model and it
generalises to any 3D preview - it is the same code whether the subject is a
car, a floor plan or a molecule.

Quaternions rather than Euler angles are not a flourish here. Composing two
axis rotations as angles produces gimbal lock the moment the user drags past
the pole; composing them as quaternions does not, and re-normalising each frame
keeps the accumulated float error from shearing the view.

## Feature checklist

- `store.glb` loads from `rawfile` on page show and renders in a `Component3D`.
- The 3D surface is only created after the load resolves - no empty view flash.
- One finger dragging rotates the camera around the model, horizontally and
  vertically.
- Two fingers pinching moves the camera closer and further away.
- Releasing either gesture clears the recorded coordinates so the next gesture
  starts from where the finger lands, not from the last one.
- On API 22 or later the camera's vignette post-process is switched off.

## Architecture

One `entry` module, one page, one helper class, one model asset.

```
entry/src/main/ets
├── common/utils.ets            free quaternion functions + OrbitCameraHelper (the whole controller)
├── entryability/EntryAbility.ets
├── entrybackupability/
└── pages/Index.ets             @Entry: Scene.load, Component3D, onTouch + PinchGesture wiring
entry/src/main/resources/rawfile/store.glb    the model
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that `OrbitCameraHelper` knows nothing
about ArkUI or about `Scene`. It takes `TouchEvent`s and `GestureEvent`s in and
answers two questions - `GetCameraPosition()` and `GetCameraRotation()` - in
plain `Vec3`/`Quaternion` values. The page is the only thing that touches the
camera:

```typescript
.onTouch((event: TouchEvent) => {
  this.orbitCamera.HandleTouchEvent(event);
  if (this.cam) {
    this.cam.position = this.orbitCamera.GetCameraPosition();
    this.cam.rotation = this.orbitCamera.GetCameraRotation();
  }
})
```

That split is what makes the helper reusable: swap the renderer, keep the
controller. It also means the helper is unit-testable without a device, which
matters for maths this easy to get subtly wrong.

The internal state is deliberately small - `cameraRotation_` (a quaternion),
`orbitDistance_` (a scalar), `cameraTargetPosition_` (the orbit centre), the
last touch point, and a `state` flag distinguishing press from pinch. Position
is never stored; it is derived on every read from rotation and distance.

## Implementation steps

1. **Load the scene in `onPageShow`, guarded by a null check** so a return to
   the page does not reload the model.
2. **Create the camera from the scene's `SceneResourceFactory`,** enable it,
   and only then publish `sceneOpt` - `Component3D` is rendered inside
   `if (this.sceneOpt)`, so it never mounts against a half-built scene.
3. **Set the initial camera z to the helper's `orbitDistance_`,** not to an
   unrelated constant - the sample uses `5` against an orbit distance of `3`,
   so the first touch snaps the zoom (`HW-15-0091`).
4. **Feed every touch phase into the helper** - `Down`, `Move`, `Up` and
   `Cancel`. Dropping `Cancel` leaves the state machine stuck in "pressed"
   when the system takes the gesture away.
5. **Gate `OnMove` on `state === 0` and a single touch** so a second finger
   landing hands control to the pinch gesture instead of yanking the rotation.
6. **Compose the two axis rotations as quaternions and re-normalise** every
   update.
7. **Reset the state to `-1` (none) when the pinch ends**, not to `1` (pinch) -
   the shipped constant blocks all rotation until the finger presses again
   (`HW-15-0091`).
8. **Clear `pinchDistance_` on pinch end** so the next pinch re-anchors on the
   current distance rather than compounding the previous scale.

## Verified snippets

All snippets are from `RotateModel.zip`. Corrected forms are marked.

**The rotation update - `entry/src/main/ets/common/utils.ets`** (as shipped)

```typescript
//更新相机旋转的四元数
UpdateCameraRotation(dx: number, dy: number): void {
  let rotationX: Quaternion = angleAxis(dx, { x: 0.0, y: -1.0, z: 0.0 });
  let rotationY: Quaternion = angleAxis(dy, quatMultVec3(this.cameraRotation_,
    { x: -1.0, y: 0.0, z: 0.0 }));
  this.cameraRotation_ = normalizeQuat(quatMultQuat(quatMultQuat(rotationX, rotationY), this.cameraRotation_));
}

//获取相机旋转后的最终位置
GetCameraPosition(): Vec3 {
  let ret = quatMultVec3(this.cameraRotation_, { x: 0.0, y: 0.0, z: this.orbitDistance_ });
  return {
    x: ret.x + this.cameraTargetPosition_.x, y: ret.y + this.cameraTargetPosition_.y,
    z: ret.z + this.cameraTargetPosition_.z
  };
}
```

**The two axes are not symmetric, and that asymmetry is the point.** Horizontal
drag rotates about the **world** up axis `(0, -1, 0)`, so left-right always
means the same thing to the user no matter how the model is currently tilted.
Vertical drag rotates about the **camera's own** right axis - obtained by
pushing `(-1, 0, 0)` through the current rotation with `quatMultVec3` - so
up-down is always "tilt towards me", relative to where the camera is now. Use
world axes for both and the control feels drunk after the first horizontal
turn.

`GetCameraPosition` is the whole orbit in three lines: take the vector
`(0, 0, orbitDistance_)` - straight back from the target - rotate it by the
camera's quaternion, and add the target position. Rotation and radius fully
determine where the camera sits, so there is no position state that can drift
out of agreement with the orientation.

`normalizeQuat` is called on every update, and its `len <= 0.0` guard returns
the identity quaternion rather than dividing by zero. Skipping the
normalisation works for a few dozen frames and then visibly skews the scene as
float error accumulates through the repeated multiplication.

**The touch state machine - same file** (corrected, see `HW-15-0091`)

```typescript
//gesture status 当前手势状态 0--按下 1--捏合 -1--无操作
state: number = -1; // 0 press, 1 pinch

HandleTouchEvent(event: TouchEvent): void {
  if (event.type === TouchType.Down) {
    this.OnPress(event);
  }
  if (event.type === TouchType.Up || event.type === TouchType.Cancel) {
    this.OnRelease();
  }
  if (event.type === TouchType.Move) {
    this.OnMove(event);
  }
}

OnMove(event: TouchEvent): void {
  if (event.touches.length === 1 && this.state === 0) {
    let sensitivity: number = 0.01;
    this.deltaX_ = event.touches[0].x - this.x_;
    this.deltaY_ = event.touches[0].y - this.y_;
    this.x_ = event.touches[0].x;      // re-anchor every frame: deltas, never a total
    this.y_ = event.touches[0].y;
    this.UpdateCameraRotation(sensitivity * this.deltaX_, sensitivity * this.deltaY_);
  }
}

HandlePinchGestureActionEnd(): void {
  this.pinchDistance_ = undefined;
  this.state = -1;                     // FIX: the sample sets 1 ('pinch'), which blocks OnMove
}
```

**`OnMove` re-anchors `x_`/`y_` on every frame**, so `deltaX_` is the movement
since the last event, not since the press. That is what makes `sensitivity`
a constant rate rather than an accelerating one, and it is why `OnRelease`
zeroing the coordinates is harmless - the next `Down` writes them before any
`Move` reads them.

The `state` guard is how the class arbitrates between the two gestures:
rotation only runs while `state === 0` (pressed) with exactly one touch.
`HandlePinchGestureActionUpdate` sets `state = 1` so the trailing single-finger
`Move` events during a two-finger gesture cannot also rotate the camera. But
`HandlePinchGestureActionEnd` sets `state = 1` **again** instead of `-1`, which
the field's own comment defines as 无操作 (no operation). After any pinch, the
state stays "pinch" and one-finger drag does nothing until a fresh `Down`
resets it to `0` (`HW-15-0091`). The `Up` following the pinch does call
`OnRelease`, which sets `-1` - so whether rotation recovers depends on the
touch/gesture event ordering, which is exactly the kind of thing not to rely
on.

`pinchDistance_` is cleared on end so the next pinch re-reads the current
`orbitDistance_` as its baseline; without that, `pinchDistance_ * (1 / scale)`
would compound against a stale anchor and each pinch would multiply the last.

**Scene load and camera setup - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0091`)

```typescript
Init(): void {
  if (this.scene == null) {
    // 加载场景资源，支持.gltf和.glb格式
    Scene.load($rawfile('store.glb'))
      .then(async (result: Scene) => {
        this.scene = result;
        let rf: SceneResourceFactory = this.scene.getResourceFactory();
        this.cam = await rf.createCamera({ 'name': 'Camera' });
        // 去除暗角
        if (deviceInfo.sdkApiVersion >= 22 && this.cam.postProcess) {
          this.cam.postProcess.vignette = undefined;
        }
        this.cam.enabled = true;
        this.cam.position.z = this.orbitCamera.orbitDistance_;   // FIX: the sample hardcodes 5
        this.sceneOpt = { scene: this.scene, modelType: ModelType.SURFACE } as SceneOptions;
      }).catch((error: Error) => {
      console.error('Scene load failed:', error);
    });
  }
}
```

**Three details carry this.** The `this.scene == null` guard makes `onPageShow`
idempotent - it fires on every return to the page, and a second `Scene.load`
would build a second scene graph. `sceneOpt` is assigned **last**, after the
camera exists, because `Component3D` is inside an `if (this.sceneOpt)` and
mounting it before the camera is configured renders one frame of an unlit,
uncontrolled scene. And the `deviceInfo.sdkApiVersion >= 22` check around
`postProcess.vignette` is the correct pattern for an API added in the version
this sample targets - the field does not exist below 22, so a bare assignment
would throw on an older device even though `compatibleSdkVersion` allows it.

The hardcoded `position.z = 5` against `OrbitCameraHelper.orbitDistance_ = 3`
is the second half of `HW-15-0091`: the model is first drawn from 5 units away,
and the instant the user touches it `GetCameraPosition()` recomputes from
`orbitDistance_` and the camera jumps to 3. A visible snap on the very first
interaction, from two numbers that were meant to be one.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the model is a bundled
`rawfile` and nothing touches the network, storage or sensors.

What does matter in configuration:

- `compatibleSdkVersion` targets 22 - `ArkGraphics3D`'s `Component3D`,
  `Scene.load` and `postProcess` are the constraint, not any permission.
- `store.glb` lives in `entry/src/main/resources/rawfile/` and is addressed as
  `$rawfile('store.glb')`. `.gltf` works the same way, but a `.gltf` with
  external `.bin`/texture files needs all of them in `rawfile` too.
- `ModelType.SURFACE` renders the scene into its own surface. The alternative,
  `ModelType.TEXTURE`, composites into the ArkUI texture tree - slower, but
  required if UI has to be drawn over the model with correct blending.

## Constraints

- API Version 22 Release or later; HarmonyOS 6.0.2 Release SDK or later;
  DevEco Studio 6.0.2 Release or later. This is a **higher floor than the rest
  of this industry pack**, which sits on API 20.
- 3D rendering needs a real device; `Component3D` does not render on the
  emulator.
- The orbit target is fixed at the origin (`cameraTargetPosition_` is
  `{0,0,0}` and never written), so there is no panning - a model whose pivot is
  not at its centre will orbit around the wrong point.
- `orbitDistance_` is unclamped. A hard pinch can drive it to a huge value or
  through zero and turn the camera inside out; clamp it to a sensible range
  before shipping.
- `sensitivity` is a bare `0.01` inside `OnMove`, unscaled by screen density,
  so the same swipe rotates further on a lower-density display.
- No reset control: once the user has tumbled the model there is no way back to
  the initial view short of leaving the page (and even that is blocked by the
  `scene == null` guard, which keeps the old camera).
- The `PinchGesture` is declared with `fingers: 2`; three-finger input does
  nothing.

## Pitfalls

- **`HW-15-0091`** (B/low, confirmed): `HandlePinchGestureActionEnd` sets
  `this.state = 1` - the value the field's own comment defines as 捏合 (pinch) -
  where it should be `-1` (无操作). Rotation is gated on `state === 0`, so
  after a pinch the drag is dead until a new press re-arms it. The same
  finding covers the initial camera distance: `Index.ets` sets
  `cam.position.z = 5` while `OrbitCameraHelper.orbitDistance_` is `3`, so the
  first touch recomputes the position and the view snaps. Fix: `state = -1` on
  pinch end, and seed `position.z` from `orbitDistance_`.
- **`state` is a bare number with a comment for a type** — `0`, `1`, `-1` and
  a Chinese comment. This is exactly the shape that produced the bug above; an
  enum would have made the wrong value unwritable.
- **`Scene` is never released.** There is no `aboutToDisappear`, so the loaded
  scene, the camera and the render surface live for the process. Acceptable in
  a single-page demo, not in an app that opens models repeatedly.
- **`Scene.load`'s rejection only logs.** `sceneOpt` stays null and the page
  renders an empty `Column` with no message - a missing or malformed `.glb`
  looks identical to a slow load.
- **No `onActionCancel` on the pinch.** If the system cancels the gesture,
  `HandlePinchGestureActionEnd` never runs and `pinchDistance_` keeps its stale
  anchor into the next pinch.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `onTouch`, `TouchType.Down/Move/Up/Cancel`, `event.touches`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` - `PinchGesture`, `fingers`, `event.scale`, the action callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/05_graphics/js-apis-inner-scene-nodes.md` - `Camera`, `Node`, `position`, `rotation`, `postProcess`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-scene-nodes
- `documentation/harmonyos-references/05_graphics/js-apis-inner-scene-types.md` - `Quaternion` and `Vec3`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-scene-types
